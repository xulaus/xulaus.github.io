# Ruby to WASM Compiler - Hello World
I’ve been interested in exploring compilers and WebAssembly (WASM) for a while now. I figured that the best way to really understand how compilers and WASM work is to build one.

This project won’t be a serious effort to create a production-ready compiler, so I'll be avoiding serious optimisations or hooking into LLVM. As I understand it, some WASM runtimes are capable of optimisation passes before running anyway.

### Prior Art
There are a few existing projects that run Ruby code in a bytecode context.

* The official Ruby interpreter can be compiled to WASM, as can some alternative interpreters such as Artichoke. These can then be used to read and run ruby code as normal.
* [mRuby](https://github.com/mruby/mruby): mRuby has a executable called `mrbc` which compiles Ruby to a non WASM bytecode which can then be executed using an interpreter.
* [RLang](https://github.com/ljulliar/rlang) (not that RLang): RLang is a subset of Ruby, and also a compiler for that subset that translates to WASM.

The aim would be closest to RLang, however RLang does provide extensions to ruby to allow for typing where as I will not.
## Hello World By Hand

The first step of learning any language is to write a Hello World application. To do so I'll be using the [`wasmtime`](https://wasmtime.dev/) runtime.

The first thing to note is that there isn't just one WASM format, there is a "human readable" version called WAT. For our hand written version its going to be convenient to use that format rather than a binary version.

On top of this, "pure" WASM has no ability to issue syscalls, so that means no `print`. This means we are going to have to use the WASI interface which provides useful functions for interacting with the host system.

I managed to find two "hello world"'s that I could <strike>steal</strike> be inspired by<sup>[1](https://github.com/danbev/learning-wasi/blob/6f4543fc1c6e9908b4fb6bebf64ff408ed9b7576/src/fd_write.wat) [2](https://blog.dkwr.de/development/wasi-load-fd-write/)</sup>. Both examples work as is. Just copying the code doesn't teach us anything though so I'm going to make some changes.

Looking at how the memory is lain out in both examples, neither starts using the memory at index 0. This seems odd to me. Also both dynamically add things to the memory at run time, which is also odd. The fact both do this is probably a sign I have no idea what I am doing but its also a good candidate for me change to make my own hello world.

After a while of struggling to understand the difference between `wasi_snapshot_preview1` and `wasi_unstable` (there isn't one for `fd_write`) I ended up with the following
```wast
(module
  (import "wasi_snapshot_preview1" "fd_write" (func $print (param i32 i32 i32 i32) (result i32)))
  (memory (export "memory") 1)
  (data (i32.const 0) "Hello World!\n\00\00\00\00\00\00\00\0D\00\00\00" )
  (func $main (export "_start")
    i32.const 1
    i32.const 16
    i32.const 1
    i32.const 100
    call $print
    drop
  )
)
```

This program has the hello world constant stored in memory and then some padding, and then two 32bit integers in little-endian order. These 2 integers make up a `ciovec` in the WASI spec, which defines an offset in the memory to start from and a length of string.

In a function called `_start` (so wasmtime knows this is our entry point) we then call print with the arguments:
 - `1` to mean stdout
 - `16` to mean at memory location 16 a list of `ciovec`'s starts
 - `1` to mean there is 1 `ciovec` to print,
 - `100` to mean use memory location 100 to return our result.

We then drop the return value.

Technically the second and third arguments are are actually one `ciovec_array`, but who's counting.

## Hello world, but this time with a compiler

Now we have a good understanding of the WASM program, the challenge is write a program that that can take the equivalent ruby and convert that into our WASM or something like it.

Generally speaking I find if I start trying to hard to design I get stuck in analysis paralysis, I'm going to something dumb with no design to it first to get something working and then iterate.

The plan is to use an off the shelf parser[<sup>3</sup>](https://github.com/lib-ruby-parser/lib-ruby-parser) search through the tree it creates, and look for a print statement. If we find anything else, throw an error.

Once we have our print statement, we'll grab its argument, check it is a list of constant strings and if it is, save that constant. Then we can construct a `ciovec` for each and save those as constants too.

In rust that logic looks like this:
```rust
fn recursive_parse(
    node: &lib_ruby_parser::Node,
    constants: &mut WasmConstants,
    current_function: &mut WasmFunction,
) -> Result<(), ()> {
  if let Send(send) == node  && send.method_name.as_str() == "print" {
      for arg in &send.args {
          if let Str(raw_s) = arg {
              let s: Vec<u8> =  raw_s.value.clone().into_raw();
              let offset = {
                  let var = constants.push_const(&s, 0);
                  constants.fetch(var).offset
              };

              let ciovec_data = {
                  let mut ciovec: [u8; 8] = [0; 8];
                  let (ciovec_offset, ciovec_len) = ciovec.split_at_mut(4);
                  ciovec_offset.copy_from_slice(&offset.to_le_bytes());
                  ciovec_len.copy_from_slice(&(s.len() as u32).to_le_bytes());
                  ciovec
              };

              let ciovec = constants.push_const(&ciovec_data, 4);
              current_function.body.push(WasmInstructions::InvokePrint { variable: ciovec })
          }
      }
  };

  Ok(())
}
```

The error handling and some other supporting code has been removed from the above version.

You probably have noticed the `Wasm*` types in the above. These are mostly just wrappers around small amounts of data that have know how to create WAT fragments.

The only interesting renderer is `WasmModule` which is just a list of functions at the moment. In the module however we also need to initialise our data. While we could just take every byte and use backslash escaping on it this would make the data block pretty unreadable, so we only escape this way if the byte of data is outside the ASCII printable range.
```rust
struct WasmModule {
    functions: Vec<WasmFunction>,
}
impl WasmModule {
    fn as_wat(&mut self, mut constants: WasmConstants) -> String {
        let mut child_wat = "\n".to_owned();
        for child in &self.functions {
            child_wat.push_str(&child.as_wat(&mut constants, 2));
        }

        const TABLE: &[char] = &[
            '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f',
        ];
        let mut data_as_str = String::with_capacity(constants.data.len() * 3);
        for ch in constants.data.iter() {
            if (0x20..0x7e).contains(ch) {
                data_as_str.push(*ch as char);
            } else {
                data_as_str.push('\\');
                data_as_str.push(TABLE[((ch & 0xF0) >> 4) as usize]);
                data_as_str.push(TABLE[(ch & 0xF) as usize]);
            }
        }
        format!(
            "(module\n  \
            (import \"wasi_snapshot_preview1\" \"fd_write\" (func $print (param i32 i32 i32 i32) (result i32)))\n  \
            (memory (export \"memory\") 1)\n  \
            (data (i32.const 0) \"{}\" ){})",
            data_as_str,
            child_wat
        )
    }
}
```
Once again there is some light editing here for clarity. The full unedited source for the compiler at this point is available [here](https://github.com/xulaus/ruby2wasm/blob/6719c89/src/main.rs).

### Compiling For The First Time
Now we have a simple compiler that can deal with strings being given directly to the print function. Given the test ruby file
```ruby
print "Hello\n"
```
compiling produces
```wast
(module
  (import "wasi_snapshot_preview1" "fd_write" (func $print (param i32 i32 i32 i32) (result i32)))
  (memory (export "memory") 1)
  (data (i32.const 0) "Hello World!\0a   \00\00\00\00\0d\00\00\00" )
  (func $_start (export "_start")
    i32.const 1
    i32.const 16
    i32.const 1
    i32.const 16
    call $print
    drop
  )
)
```
which is incredibly similar to our hand written function. If we then run these two files using the `time` utility
```
$ time ruby test.rb
Hello World!
ruby test.rb  0.08s user 0.03s system 96% cpu 0.108 total
$ time wasmtime run test.wat
Hello World!
wasmtime run hello.wat  0.01s user 0.01s system 87% cpu 0.015 total
```
😎
