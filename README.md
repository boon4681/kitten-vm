# Kitten-vm

this is a design documentation for a stupid project that i choose as my "Final Project" (Catnip game engine)

> NOTE: if i just put word "**gc**", its mean garbage-collector or garbage-collected don't nit-picked me too much im lazy OKAY????

this vm is a simple garbage-collected and statically-typed vm

**why gc?**

basic-vm knowledge. how do we know when a heap can be free?

in C we use free() - so its a programmer told signal?

nah, cuz if you can ready to go or easy to use language/interface-for-programmer (code is like a interface to interact with computer so i use interface-for-programmer)

> you writting a game, you are so f*cked if you forgot to free memory

if you dont want to have programmer free memory everytime how do you gonna do it?

Just have vm manage it you said.

well....... how vm gonna know when to free? vm is not human that can see ohh!! that a memory that no one used, you should free that.

the simple approach is to just track who is using it by counting

```
A is used by B cnt +1
B is not used by anyone cnt =0
B is free
A is not used too cnt -1, cnt =0
A is free

simple math hahahaahaha.

but not that simple

A is used by B
B is used by A

you may hey how you can do that. boys

A and B is object in you lovely programming language it probably look like this

let A = {}
let B = {}

A.own = B
B.own = A

now it is

A is used by B cnt +1
B is used by A cnt +1
B is not used by anyone cnt = 1
A is not used by anyone cnt = 1

and that's it, no one is zero, memory cannot be free

but you may say, hey why we can't just free both cuz no one is using it?

haha how can you prove it
how can you prove that those two didn't have anyone using it?

Blimming-hell doesn't it

but we do have a solution just tracking who using it, we called that mark-and-sweep garbage-collector (the first one that counting things is reference-counting garbage-collector).
```

Having to much fun talking back to the main topic.

**Goal**

Goal is simple, this vm gonna used as runtime for my game-engine im currently working on.

1. this vm need to be lightweight
2. this vm need to be too slow (not fast, but do well)
3. this vm need to introduce minimal lag or avoid it if possible

"1." breakdown.

this is easy just write any vm and the goal is done.
how one can be big?

"2." breakdown.

time to think how can vm be slow?
mostly i think it's because dynamic-type and garbage-collector,
but its not those two fault tho, because having those two things in the language or vm programmer just simply ignore things that bad for the performance so im cutting dynamic-type. garbage-collector maybe?

"3." breakdown.

this is the hard one, introduce minimal of lag or avoid it.

first you need to think about how lag happen in video-game in case that we have unlimited of ram and vram
1. loading resource/assets (I/O)
2. a lot of piling memory being clean-up by gc
3. bad programmer code (this cannot be avoid, im not fixing this)
4. that's it, i jsut too dumb to come up with more

"3.1." breakdown

use money and buy a SSD or write a better optimized game

"3.2." breakdown

this is my focused problem, gc can be slow if you dealing with complete object, so how do we solve it?????

nature of game developer, in mind we always want to free things all the time. what do you mean by that free things all the time?? what does that even mean.

cuz can game is a realtime media if you fired 10000 bullets there a 10000 you need to update and that a lot and add up fast make a game slow so the game developer tend to free things on there own but that a game object. not an object in the backend that player doesn't see like maybe a crafting recipe, if you spamming opening and close crafting recipe, you may crash a game cuz you used all of your ram and you didn't clear an old loaded recipe.

we can solve it by this.

we make rules
- try to make as much as memory possible be on stack
- if we can't do it do mark and sweep

[see the source](https://www.baremetaldev.com/2021/10/26/gc-theory-reference-counting-explained/#Partial_Mark-Sweep_Algorithms_aka_trial_deletion)