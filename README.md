# Kitten-vm

This is a design documentation for a stupid project that i choose as my "Final Project" (Catnip game engine)

> NOTE: if i just put word "**gc**", its mean garbage-collector or garbage-collected don't nit-picked me too much im lazy OKAY????

this vm is a simple garbage-collected and statically-typed vm

## Why gc?

Start with basic-vm knowledge, how do we know when a heap can be free?

in C we use free() - so its a programmer told signal?

nah, I can't do that cuz my goal is to have ready to go game-engine.
Imagine you writting a game, you are so f*cked if you forgot to free memory.

Then! if you don't wanna have programmer freeing memory them-self.

: Just have vm manage it you said.

Well....... how vm gonna know when to free memory?

Vm is not human that can see ohh!! that a memory that no one used, i should free that.

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

## Goal

Goal is simple, this vm gonna used as runtime for my game-engine im currently working on.

1. this vm need to be lightweight
2. this vm need to be not too slow (not fast, but work well in many senario)
3. this vm need to introduce minimal lag or avoid it if possible

"1." breakdown.

this is easy just write any vm and the goal is done.
how one can be big?

JAVASCRIPT? joke aside javascript is a good language btw.

"2." breakdown.

time to think how can vm be slow?
mostly i think it's because dynamic-type and garbage-collector (mostly garbage-collector),
but its not those two fault tho, because having those two things in the language or vm programmer just simply ignore things that bad for the performance so im cutting dynamic-type. garbage-collector maybe?

"3." breakdown.

this is the hard one, introduce minimal of lag or avoid it if possible.

first you need to think about how lag happen in video-game in case that we have unlimited of ram and vram
1. loading resource/assets (I/O)
2. a lot of piling memory being clean-up by gc
3. bad programmer code (this cannot be avoid, im not fixing this)
4. that's it or im just too dumb to come up with more

"3.1." breakdown

Have user use money to buy their new SSD or Have programmer write a better optimized game.

This vm should at least support Non-blocking I/O.

"3.2." breakdown

this is my focused problem, gc can be slow if you dealing with complete object, so how do we solve it?????

nature of game developer, in mind we always want to free things all the time. what do you mean by that free things all the time.

You might said: what does that even mean????

cuz game is a realtime media if you fired 10000 bullets there a 10000 things you need to update and that a lot and add up fast, make a game slow, sodeveloper tend to free things on there own but that a game object. not an object in the backend that player doesn't see like maybe a crafting recipe, if you spamming opening and close crafting recipe, you may crash a game cuz you used all of your ram and you didn't clear an old loaded recipe.

we can solve it by this.

we create set-of-rules to apply
- try to make as much as possible to make tracking memory cheap
- if we can't do it do mark and sweep

What memory tracking that's super cheap?
- Reference-counting

So we invent a hybrid system (that already existed 😥, [see more](https://www.baremetaldev.com/2021/10/26/gc-theory-reference-counting-explained/#Partial_Mark-Sweep_Algorithms_aka_trial_deletion)) that might have a cheap cost for freeing memory.

## What does this project contains?

- VM
- Codegen
- IR (intermediate-representation)

## Terminology

- Lifetime analysis refer to mechanism that used to detect when a pointer, reference or view type refers to an object that is no longer alive, a condition that leads to use-after-free bugs.
- Hybrid-GC refer to Reference-Count + Partial_Mark-Sweep_Algorithms (aka trial deletion) similar to system that got introduced by [Nim programming language](https://nim-lang.org/docs/mm.html)
- Fiber term for coroutine

## Specification

### Introduction

Kitten-VM, a shitty vm + codegen for my game-engine project, Statically-type + hybrid system garbage-collector, coroutine-capable, 
have predictable runtime.

The vm do [lifetime analysis](https://smallcultfollowing.com/babysteps/blog/2018/04/27/an-alias-based-formulation-of-the-borrow-checker/) on codegen but unlike rust this vm use lifetime analysis to reduce ref-c ops, the cycle detection run on the type graph to flag a potential cyclic (also on codegen).

The codegen takes IR and output VM instruction. its do somekind of acyclic assertions.

The vm have a predictable runtime by round-robins across three kinds of pending work each tick:
- draining a release job
- stepping the cycle collector one phase
- freeing one deferred-free buffer

It keeps doing that until the budget run out or a deadline passes.

The vm looks at how full it currently is and linearly scales the work budget between a min and max as pressure cross thresholds (default: ease in starting at 50% full, maxed out by 85% full).

### Types

#### Native support
: this have opcode supported by default

- Number: overflow supported
  
  ```
  i8, i16, i32, i64, i128
  u8, u16, u32, u64, u128
  f8, f16, f32, f64
  ```

- Boolean
- String
  
  All string ops produce new strings, which the ownership analysis then manages like allocation.
  most string die via a single `DROP` at scope end. No GC involved.
  
  **Short strings (below ~32 bytes)** is in a global hash table.
  
  EQ is pointer comparison and repeated so its cost nothing. Long strings are not interned cuz hashing huge string on allocation is a waste of time.
  
  **Constants detection** (refcount never reaches 0), so a string is
  a free borrow [(the Lobster string-constant rule)](https://aardappel.github.io/lobster/language_spec.html#operators-on-strings).

- Vector
  
  ```
  vec2, vec3
  ```

  same size array multiplication is also allow; error at runtime if the size is not the same

- Array
  
  Array is Dynamic by default; config mode - default or bloat (bloat mode make codegen tell user to specify every array allocation whether its gonna be static-size or dynamic)

- Map (Dict)

  HashMap

### Bytecode

Kitten-vm operands are variable-width similar to V8 bytecode

- Default: 8-bit operands
- Wide prefix byte: 16-bit operands
- ExtraWide prefix byte: 32-bit operands

### Memory management

The vm must not Stop-The-World and have cycle collector.

when a release drops an object count to something *non-zero* and the object is cycle-capable (instances flagged `is_cyclic`), it get buffered as a cycle candidate instead of ignore.


#### the cycle-capable proof thing

It done via type info and type-ref-graph.

Not every type needs to be cycle-checked, most game objects are tree not graphs, and running the cycle collector over stuff that can never actually cycle is wasted work. so `Type_Info.is_cyclic` is inferred at compile time, per type, from the type own field references.

### Fibers

coroutines, basically, cuz a game loop needs it. a `Fiber` carries its own register and call-frame stack, plus a `resumer` pointer so you can't resume something that's already someone else business.

NOTE: fiber-to-fiber switching itself doesn't cost work budget.