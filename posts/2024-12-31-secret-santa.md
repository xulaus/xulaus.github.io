# Secret Santa In Three Levels of difficulty

This time of year I always find myself thinking about secrets santa algorithms. While the sensible thing to do is to find a website and use whatever algorithm already provided, I cant help think about how those websites might operate. Particularly since I watched a numberphile video[^1] on the topic which presents a simple and beautiful algorithm for assigning secret santas. However the numberphile approach desent work for the secret santas I find myself in. Usually there are far more rules to them than just assigning random participants so I have spent a lot of time thinking about how these rules might be implemented, so here are 3 levels of difficulty to the secret santas with the algorithms to solve them.

---

## Level One


<div class="well">
  Given a list of secret santa participants, assign each person another person to buy a gift for. No person should be buying for themselves.
</div>
The video discussed in the introduction has possibly one of my favorite algorithm solutions to this. We can simple put every person in a list, shuffle that list, and then have each person buy for the next person on the list. Wrapping to the beginning when we reach the end. We can guarrentee any random order is valid because all people are allowed to buy for any other person, thus solving the problem. If you'd like further details I recommend the original video [^1]. Fairly nicely this results in a single chain of present buying.

---

## Level Two
If we assume the secret santa is for siblings and siblings in law then it can be assumed that couples will be buying presents for each other anyway, seperate to the secret santa. Therefore we need to change the problem a bit.

<div class="well">
  Given a list of secret santa participants, all of whom are couples, assign each person another person from a different couple to buy a gift for.
</div>

This is a fairly simple extension to the above. We can't use the single loop though but we can use _two_ loops. There is nothing in the problem statment that we need to have a single loop of presents so we can have 2 secret santas - randomly assign each couple to a different secret santa and then run the level one solution. That does feel like cheating though so...

<div class="well">
  Given a list of secret santa participants, all of whom are are a member of a <i>family</i>, assign each person another person from a different family to buy a gift for.
</div>

Maybe a couples children have grown old enough to participate, or maybe a couple broke up and now there is only one person participating - either way we now cant just segment blindly without proving to ourselves it is a valid thing to do.

---

## Level Three

<div class="well">
  Given a list of secret santa participants, all of whom are are a member of a family, assign each person another person from a different family to buy a gift for. No one should buy for the same person as the year before.
</div>



---
[^1]: https://www.youtube.com/watch?v=5kC5k5QBqcc
