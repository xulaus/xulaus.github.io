# Efficient Fuzzy Searching

## Abstract

## Motivation

When I got this idea, AI powered text searching was all the rage. It seemed like every week there is a new vector database vying for attention on Hacker News.

My understanding of how these work is they combine two technologies. The k-d tree, a data structure allowing fast look up of nearest neighbours even when the positions have many dimensions, and Word Embeddings, a way of transforming text into positions in a high dimensional space where data is clustered according to meaning. Its fairly obvious how these two technologies are a marriage made in heaven for semantic search. You transform your search term into a Word Embedding, find the closest neighbors to that in your K-D tree and you are done.

However, sometimes when you are looking for things you aren't really looking for something which is talking about the same thing, but an exact phrase or misspellings of that phrase. For example, I might be looking for details of a book from the name of a book, or searching for a document you have written based on its title.

To solve this problem I think I have come up with a novel approach, although I think it might only be novel to me and I am just unable to find references to it any where else.

My idea is to steal a different technique from Machine Learning. Instead of using a word embedding, I'll transform the search term into a vector (math definition, not C++ dynamic array version) by counting pairs of characters, aka 2-grams. This will create a vector of 65536 dimensions.

The ML bit is that because we are using a 2-gram, there is some ordering information contained within the vector. E.G if we have a string `ABC` then 2-grams are `AB` and `BC` so just by looking at the 2 grams we know `C` is after `A` and never before.

This works well enough to create word embeddings, and Facebook did with [fastText](https://arxiv.org/abs/1607.04606) [^1], but building a search directly of this sub word information is new I think.

I think the similarity of the subword vectors will work as a very loose approximation of edit distance. If I use the k-d tree to filter a large dataset down and then calculate full edit distances on a much smaller subset, I am hoping I can get equivalent results much quicker.

## Implementation

The conversion of a string into an subword vector is straight forward.

```rust
fn as_vec(data: &str) -> Vec<f32> {
    fn to_idx(x: u8, y: u8) -> usize {
        ((x as usize) << 8) | (y as usize)
    }
    let data = data.as_bytes();
    let mut ret = vec![0.0f32; NUM_DIM];
    if data.is_empty() {
        return ret;
    }

    // Unwrap is safe as we have made sure it is non empty above
    let last = *data.last().unwrap();
    ret[to_idx(b' ', data[0])] = 1.0;
    ret[to_idx(last, b' ')] = 1.0;

    for pair in data.chunks_exact(2) {
        ret[to_idx(pair[0], pair[1])] += 1.0;
    }

    let sum = ret.iter().map(|x| x * x).sum::<f32>().sqrt();
    ret.iter_mut().for_each(|x: &mut f32| *x /= sum);
    ret
}
```

There are a couple of subtleties compared to the description above. We ignore whitespace at the begining and end, but also we pretend that there always is. This means that and exact match at the end of a word gives the same weight as an exact match in the middle.

Testing against a database is very easy. There is a package for rust called [`kdtree`](https://github.com/mrhooray/kdtree-rs), so we can just use someone else's code for the data structure. Reading the docs and porting to our vector representation gives the following (test data provided by George Orwell)

```rust
fn main() {
    let data = [
        "homage to catalonia".to_string(),
        "road to wigan pier".to_string(),
        "1984".to_string(),
        "down and out in paris and london".to_string(),
    ];

    let mut kdtree = KdTree::new(NUM_DIM);

    for item in data {
        kdtree.add(as_vec(&item), item).unwrap();
    }

    for (x, item) in kdtree
        .iter_nearest(&as_vec("catalonia"), &squared_euclidean)
        .unwrap()
        .take(4)
    {
        println!("\"{item}\" {x}");
    }
```

This will give the closest book names to our search string (`"catalonia"`) and the distance from the search. As the vector is normalised and strictly positive we can think of all points as lying on a half sphere, the means furthest away the two points can be is √2. The distance metric we are actually using is the square of this, so completely unrelated strings will have a distance of 2.

```
$ cargo run --quiet
"homage to catalonia" 0.769085
"down and out in paris and london" 1.825922
"1984" 1.9999998
"road to wigan pier" 2
```

1984 and Road to Wigan Pier are both within a rounding error of 2, Down and Out in Paris and London is slightly closer than the others as `London` and `Catalonia` share the `lon` substring, which I hadn't realised before running the test. Cool!

I have a habit of misspelling Catalonia though, so lets try again with `Catelonia`. This gives

```
"homage to catalonia" 1.015268
"down and out in paris and london" 1.825922
"1984" 1.9999998
"road to wigan pier" 2
```

### More data

Now we have something that seems to be working, lets shove more data at it. I grabbed a csv with 10,000 book titles in it, added some debug output and re-ran with that:

```
Reading titles from `data/books.csv`...
Done, took 0.019s
Building k-d tree...
Done, took 4.278s
Please enter a search term: catalonia
Found 20 results. Took 1.135s
```

This seemed pretty slow to me, particularly the lookup side. I have no intuition for how fast a k-d tree should build, but I would have thought the searching should be faster than parsing from the csv file as if we can iterate through every line in that amount of time surly the k-d tree isn't pulling its weight.

The key problem is that I have to loop over all 65k vector elements for each distance comparison, even though I know the majority of them are 0. My solution to this was to quickly whip up a sparse vector representation, storing only the non zero elements of our vectors along with the index that they would have had in a full representation.

Here is the struture I am using along with the lookup code for a specific index

```rust
#[derive(Default, PartialEq)]
pub struct SparseVec {
    pub len: usize,
    pub data: Vec<(usize, f32)>,
}

impl std::ops::Index<usize> for SparseVec {
    type Output = f32;

    fn index(&self, index: usize) -> &Self::Output {
        assert!(index < self.len);

        let x = self.data.partition_point(|&(x, _y)| x < index);
        if x >= self.data.len() || self.data[x].0 != index {
            &0.0
        } else {
            &self.data[x].1
        }
    }
}
```

I then excitedly went to use this and found out that the k-d tree library I was using required that the vector could be dereferenced to a `&[Float]`, requiring a dense representation.

After hacking in a `Vector` trait that allowed me to be properly generic in the tree between the dense and sparse representation, I had gained some performance.

```
Reading titles from `data/books.csv`...
Done, took 0.013s
Building k-d tree...
Done, took 2.413s
Please enter a search term: catalonia
Found 20 results. Took 0.848s
```

I suspect this is because initialy the code was passing in the distance function to the tree, so the compiler couldn't optimise well. I'd moved this onto the `Vector` trait as it made the types easier, but this single static dispach has improved the performance significantly. When small changes like that make such a big impact I am always impressed.

But if that was impressive then the sparse representation was shocking.

```
Reading titles from `data/books.csv`...
Done, took 0.015s
Building k-d tree...
Done, took 0.012s
Please enter a search term: catalonia
Found 20 results. Took 0.002s
```

I genuinely thought I had a bug until I double checked the scores against the dense version.

### Search Range

You may have noticed that I just hardcoded "find the closest 20" and stopped showing the actual results. To be frank, this is because they suck. The issue is with search strings much shorter than the target strings, the vector that represents target strings has non zero entries in many more dimensions. I did try some different encoding schemes, and settled for one hot for reasons described later, but it didn't have any appreciable effect on quality. I did start playing with the idea of changing the internals of the nearest search to ignore dimensions where the search vector was zero, but this seamed tedious so I didn't, plus I'm not sure that would then count as a distance mathmatically which might mess with the K-D tree in unpredictable ways.

This leaves us with the case where you are searching with a similar length string to the target. This makes searching a lot easier, as we can derive a distance that is acceptable to us, and find everything within that distance.

To do this I find Levenshtien distance divided by string length the most friendly metric. This can be easily understood as the percentage of characters different in each string. So we need a way of converting this distance to our distance metric.

There are three cases we need to care about, adding a character, removing a character, and changing a character. Really this is just two cases as adding a character is just the inverse of removing a character so the distances should be the same.

In general the euclidean distance between the two vectors is

<math>
  <msqrt>
    <msub><mo>&sum;</mo></mi>i</mi></msub> <mo>(</mo><msubsup><mi>x</mi><mi>i</mi><mn>2</mn></msubsup><mo>-</mo><msubsup><mi>y</mi><mi>i</mi><mn>2</mn></msubsup><mo>)</mo>
  </msqrt>
</math>

<math>
  <msqrt>
    <msub><mo>&sum;</mo></mi>i</mi></msub><msub><mi>c</mi><mi>i</mi></msub> <mo>(</mo><msub><mi>x</mi><mi>i</mi></msub><mo>-</mo><msub><mi>y</mi><mi>i</mi></msub><mo>)</mo>
  </msqrt>
</math>

As we are using a euclidien distance in the KD, and all the vectors are normalised, for a Levenshtien distance of 20% this would be

<math>
  <msqrt>
    <mn>20%</mn><mo>×</mo><mn>2</mn><mi>l</mi><msup><mi>x</mi><mn>2</mn></msup>
  </msqrt>
</math>

## Comparison

## Future Work

---

## Footnotes

[^1]: GPT may have everyone else's heart at the moment, but fastText remains my favourite embedding because it introduced me to this subword trick, which enables it to assign meaning to words it has never seen before. It may never seen the word asteroseismology but it knows astero and seismology and can work out its the study of star quakes. Thank you Facebook, very cool.
