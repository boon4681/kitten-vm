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

The VM must support native desktop hosts and web browsers through WebAssembly.
Non-blocking I/O remains required on both targets.

but that is where im using it, not what the vm is limited to. its a general-purpose vm, the design tries to minimize pauses so if you write a game with it, the runtime itself should introduce as little lag as possible. game examples below are why i care about this, not features the vm needs to know about.

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

This vm must support Non-blocking I/O, including bounded submission and completion work. the runtime contract is below.

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

Kitten-VM, a shitty vm + codegen, Statically-type + hybrid system garbage-collector, coroutine-capable,
try to keep memory cleanup within a budget each tick. predictable as much as i can, but no hard realtime promise :).

The vm do [lifetime analysis](https://smallcultfollowing.com/babysteps/blog/2018/04/27/an-alias-based-formulation-of-the-borrow-checker/) on codegen but unlike rust this vm use lifetime analysis to reduce ref-c ops, the cycle detection run on the type graph to flag a potential cyclic (also on codegen).

The codegen takes IR and output VM instruction. its do somekind of acyclic assertions.

The vm round-robins across three kinds of pending work each tick:

- release one reference from a release job
- move the cycle collector one small step
- return one small memory block to the vm allocator

why one small step? cuz "one job" can be an array with 100000 objects inside it. calling that one job doesn't make it cheap.

so a job can take many ticks, the vm remember where it stopped and continue next time.

The vm has two configurable limits: how much work it can do, and how much time it can spend. check the deadline before each step, stop when either limit is reached.

can a step still go past the deadline? yes. we can't undo time already spent, and OS can interrupt us too. so we need small steps, and this is still soft realtime.

The vm scales the work limit between min and max as memory pressure goes up (default: start at 50% full, max at 85%). this doesn't give it permission to spend more time than the configured time budget. more on that below.

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
  most string die via a single `DROP` at scope end. still ref-counted and queued for freeing tho, just no cycle checking needed.
  
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

The vm must not stop everything (Stop-the-world) and walk the whole heap in one go. so its split release work and cycle checking into small steps, run some between program execution slices, then continue next tick.

a tick here is a vm scheduling turn. the host (whatever application is running the vm) can fit those turns into its own loop. the vm doesn't need to know what a frame is.

sounds simple until program code changes the thing we were checking.

#### who can touch the heap?

one vm thread owns the managed heap. fibers run on that thread too.

I/O can work on another thread, but its result needs to come back to the vm thread before it can create or change managed references. managed objects are thread-confined; native workers communicate through messages/results instead of sharing managed references. this is an ownership rule of the vm.

`strong_count` counts who owns the object. that includes object fields, arrays/maps, registers, call frames, suspended fibers and native handles registered with the vm.

weak reference doesn't own anything. collector bookkeeping doesn't own anything either.

codegen can remove retain/release if it can prove someone keeps the object alive for the whole borrow. across a function call? same rule. across a fiber yield? same rule. can't prove it? keep an owning reference.

native code also needs a registered handle if it wants to keep an object. otherwise vm can't know someone is still using it.

when replacing a reference, retain the new target first then release the old one. a move just transfers ownership. once an object is dead, no bringing it back.

also no user finalizers. imagine trying to free something and it runs another whole script, there goes our budget. external resources like files or native handles use explicit close/dispose, and cleanup inside gc must stay small.

#### freeing later doesn't mean still alive

count reaches 0 -> object is dead -> put it in the release queue once.

its memory is still there while cleanup is pending, but you can't use it. weak lookup must fail already, don't wait for the actual free.

why queue it? look at this:

```
A owns B
B owns C
C owns D
... keep going for 100000 objects

free A
now free B
now free C
...
there goes the frame (and maybe the stack)
```

so a release job remembers which reference slot it is on. one step clears one slot and decreases the target count. target reaches 0? queue that too, don't recursively free it right now.

target count is still non-zero and its type can cycle? put it in the cycle candidate queue. don't add the same candidate over and over.

once all slots are cleared, queue the storage for freeing. each slot gets released exactly once. arrays/maps need to remember their scan position too, even scanning a huge empty map can take time.

actual memory can only be reused after release work is done and any active batch pin on that object is removed (explained below). candidate queues use handles with a generation, so an old queue entry can't accidentally point at a new object using the same address.

the queues themselves cost memory too. reserve their bookkeeping space before allowing an operation that needs it. running out of memory cannot mean we just forget to release something.

#### the cycle-capable proof thing

codegen builds a graph of types using their **strong owning** references.

```
A owns B
B owns A
both can cycle

C owns A
nothing points back to C
C doesn't become cyclic just because A is
```

types in a strong connected component with multiple types, or a type with a self-edge, get `Type_Info.is_cyclic`.

this includes array elements, map keys/values, closure captures, fiber state and native ownership too. a field can hold multiple types? include all allowed targets. can't figure out the targets? assume it can cycle. weak references don't count here.

acyclic assertion needs to be checked by codegen, otherwise its just a programmer saying "trust me bro" memory management.

`is_cyclic` means it *can* cycle, not that every object of that type actually does but false positive is better than a false negative (in this case is memory-leak; trade with speed). 

keep the type component id too: cycle checks follow possible return paths inside that component. outgoing acyclic tails and other components are handled by normal release if their owner dies.

#### what if the program changes things while we check?

imagine collector checks A, pauses, program code gives A a new owner, then collector continues using the old numbers.

yeah, we can't free things based on old numbers.

the collector uses **local trial deletion with tracked internal edges**. no heap-wide snapshot, no saving every old pointer, and no holding unrelated objects until a cycle check finishes.

this is the intended for this vm. its correctness requirements below still need to be checked against a state model and the implementation; this design document doesn't hold a proof by itself.

#### only hold what we are checking

one batch is active at a time. its members have a batch id and a physical pin. a pin stops storage reuse, it doesn't add ownership or make a dead object alive. objects outside the batch keep using ordinary reference counting and deferred release.

batch discovery follows strong edges that can actually lead back into the same cyclic type component. edges to another type component or an acyclic type don't need cycle traversal: they can't lead back, otherwise codegen would have put them in the same component. those outgoing references get released normally if their owner dies. unknown types must use conservative components; this optimization depends on a sound type graph.

so if a cycle points at a huge acyclic asset tree, we don't need to walk or pin that whole tree just to prove the cycle is dead.

only discovered members are pinned. outgoing pointers to objects outside the batch are never kept in collector records across a yield. resolve a current slot while its owner is held, then enroll and pin an eligible target in that same small step. otherwise leave it outside the batch.

```
checking A <-> B
unrelated X reaches count 0
X's release job runs normally
X doesn't wait for A and B
```

#### candidates share the work

a candidate gets queued when a cycle-capable object loses an owning reference but stays non-zero. ownership removal during a move also counts, even if codegen combines the count updates. count 0 still goes straight to ordinary release.

each candidate has a change version, a queued bit and the version its last completed check covered. version ids cannot wrap into a still-used id.

start a round by detaching the current candidate queue in constant time. new requests go into the next queue. walk that finite round list incrementally, adding seeds to a shared batch. discovery uses the same visited/member records for every seed, so overlapping candidates don't start separate walks of the same graph. physically stale handles and dead candidates are discarded safely.

a live candidate whose version was checked doesn't get requeued just because it survived. no changes, no reason to keep checking the same thing. ownership removals during a check record a next-round request; consuming an older request must not clear that newer one. membership and next-round queue state are separate.

also record a retry request for a rescued member if it can cycle. this covers conservative survival from an ownership transfer. consume that request only in a later completed check. this can produce an extra check after activity, not an endless stream of checks on an unchanged live graph.

batch scratch and held storage have configured caps. discovery stops admitting members at a cap and leaves unadmitted candidates queued. references from excluded objects count as outside ownership. splitting a cycle this way may prevent collecting it, so report a batch-capacity obstruction and remember its change version instead of endlessly retrying it unchanged. increasing the allowance or a relevant ownership change permits a retry. the supported maximum collectible cycle is constrained by available batch storage; no hidden emergency full-heap collection.

#### discovery must finish even while the program runs

record an allocation cutoff and a finite frontier of existing slots for each enrolled object. new objects aren't admitted to this batch. new slots don't extend discovery's frontier. removed slots can be skipped, and a changed existing slot is read at its current value. discovery isn't trying to reconstruct an old graph.

reference containers use stable slot ids and bounded blocks while being scanned. removed slot ids cannot be reused within the active batch. retire removed blocks until the active member's cursors have passed them or the batch releases them. the live index can change, but collector cursors never point into an index that a resize can move. new blocks use a separate insertion path so they cannot keep extending an old scan frontier.

this retains some storage belonging to active members, not every old object in the heap. numeric arrays without managed references need no edge-tracking representation.

once discovery ends, **seal membership**. no more objects enter this batch. an edge from an excluded object is an outside reference, even if that object was created during collection.

#### temporary counts that follow the real graph

for a sealed batch, the number we care about is:

```
trial_count(object)
    = current strong_count(object)
    - number of currently recorded incoming edges from batch members
```

initialize each member's trial count from its real count, one member per step. a per-member ready bit tells the ref-c helper to apply subsequent count deltas to its trial count too. initialization and setting that bit happen in one non-yielding step. don't start recording internal edges until every member is ready.

then scan strong slots, one at a time. if source and target are both members, record that edge and subtract one from the target's trial count. a slot has a batch-tagged recorded bit so scanning it again can't subtract twice. no old target value is kept after the operation.

while edge recording is active, slot writes maintain that equation:

- removing a recorded internal edge adds one back to its old target's trial count and clears the record.
- real retain/release changes update the ready target's trial count by the same amount.
- installing a new internal edge records it and subtracts one from its new target's trial count, even if the scan hasn't reached that slot yet.
- slots created during the scan are handled by the write path, so the scan only needs to finish its fixed frontier. initializing a whole container still processes one owning slot at a time.

all parts of a single-slot replacement finish before a collector step can run. retain the new target before releasing the old target. transient intermediate trial counts inside that replacement aren't used to decide liveness.

a move is the same bookkeeping for its source and destination slots, with any count cancellation codegen proved safe. zero-count release jobs use these helpers too. deleting a container doesn't skip the per-slot rule.

when the edge scan finishes, every current internal edge is recorded, so trial counts are outside ownership counts. they must be non-negative. a negative count is an invariant failure, never evidence that an object is garbage.

#### protecting live objects while marking

before publishing a new owning reference to an active member, mark that target as rescued and queue survivor work once. this includes native handles, weak promotion and ownership moves. targets outside the batch need no rescue action.

rescues are recorded from enrollment onward, including discovery and count initialization. start survivor traversal only after edge recording finishes. positive trial counts and rescued members seed it. follow current internal edges and mark everything reachable as live.

each member is marked once. its scan has a fixed slot frontier, and writes that install edges rescue their targets directly. so inserting into a slot the scan already passed cannot hide a new live target, and repeated writes don't keep restarting the walk.

an outside count can increase after its member was checked. that ownership acquisition must rescue the member before publication, so commit doesn't depend on rescanning every trial count at the last instant. a removed outside owner may leave an unnecessary survivor; its removal schedules a later check.

#### commit and release

when discovery, count initialization, edge recording, root seeding and survivor work are all finished, check the work queues and flip the batch's committed flag in one non-yielding scheduler step. the heap is single-threaded, so a program write cannot race that check-and-flip.

unmarked members are now dead. effective death is the ordinary dead flag OR unmarked membership in a committed batch. retain, release, weak lookup and candidate lookup all use that rule. flipping the batch flag publishes death without looping over every member.

turn off trial-count/edge maintenance for the committed batch with that state change. then drain each newly dead member's current owning slots once, through normal release helpers. members already dead through ordinary ref-c belong to their existing release jobs; the batch doesn't create a second job for them. all slots cleared by either path have an explicit cleared state.

keep batch members pinned until the batch's dead-member edge drains are finished. then retire records and pins one member/block per step. persist ordinary death before removing dead membership. a member's memory also waits for its own ordinary release job to finish. the descriptor stays until no record points at it. cleanup must finish before another batch uses its storage.

#### what does an assignment pay for?

ordinary ref-c is still a cost. this design doesn't call it free. but it no longer saves old counts/slots or runs a rescue check for every old object in the heap.

| operation | collector work |
| --- | --- |
| plain numeric/value assignment with no managed references | none |
| proven borrow with an owner covering every use | no retain/release or ownership-publication barrier |
| owning update outside the active batch | ordinary retain/release and candidate rules, plus membership tests where not eliminated |
| real count change on a ready batch member | also apply the delta to its trial count |
| slot update on an active member during edge recording/marking | also maintain the recorded internal edge |
| new owner of an active member before commit | also set rescue state and enqueue once |

codegen can omit collector checks for values/types proved outside cycle batches, and for new objects proved outside the active batch. a check hoisted past a safepoint needs proof that batch state cannot change there. eliminating ref-c operations alone isn't proof that ownership-publication work can disappear.

use pre-reserved member records and intrusive queues for barrier work. barrier operations touch a fixed number of records; no traversal, allocation, hash-table growth or I/O on that path. per-slot recorded metadata is needed for reference-bearing layouts that can join a batch, not for arbitrary numeric data. its byte cost must appear in the layout specification.

deadline checks and round-robin dispatch also cost time. the work unit can be a small fixed-size chunk with a documented maximum, rather than paying a clock read for every pointer. choose that maximum from measured worst-step time; never treat an entire variable-sized object as one unit.

#### what needs to be proved when implementing this

these are acceptance requirements for the design:

- after a member is initialized, its trial-count equation holds at every scheduler boundary, across stores, moves, ordinary releases and native-handle changes.
- after recording, every current internal edge is counted exactly once, including newly inserted slots.
- positive outside counts and every newly published owner protect their reachable batch members before commit.
- sparse containers, repeated resize and new allocation cannot extend a scan frontier forever.
- unrelated objects reaching zero reclaim normally during a long cycle check.
- multiple queued candidates reaching the same graph share one traversal in a round; unchanged survivors stop generating work.
- ownership removal during a check survives queue deduplication and gets its later check.
- weak promotion versus commit, dead members with ordinary release jobs, batch-cap obstruction and scratch exhaustion cannot cause resurrection, double release or stale access.

check small graphs with an exhaustive state model that interleaves a mutation at every collector step, including multi-slot moves. measure program-side barrier time, total collection work, maximum step time, held bytes and repeat scans. the latency goal includes all of those, not just the scheduled gc timer.

#### what does "budget" actually mean?

one work unit is one reference slot, one member/queue record, or one small allocator block. a scheduler step can process at most a configured fixed number of those units before checking time again. growing queues, growing hash tables, tracking visited objects and abort cleanup need the same treatment. no hiding a huge loop inside "one step".

one object can use many blocks, so freeing all its blocks isn't one step either.

the vm allocator keeps reusable blocks in its own free lists. returning pages to the OS or doing a big native free can take time we don't control, so do those at explicit host maintenance points outside the normal cleanup time budget.

allocation, string hashing, large copies and native calls can still be slow too. fixing gc doesn't magically fix those.

configs:

- `heap_limit_bytes`: how much managed memory we allow. objects, container storage, allocator's retained blocks, queues and collector temporary storage all count.
- `maintenance_min_units` / `maintenance_max_units`: how many small steps per tick. min must be positive when maintenance runs, otherwise it can get no work forever.
- `maintenance_budget_us`: how much time cleanup gets each vm tick.
- `maintenance_reserve_bytes`: part of the heap limit kept for release/collector bookkeeping. ordinary allocation can't eat this part.

pressure = charged bytes / `heap_limit_bytes`.

dead but not freed yet? still counts. allocator kept the memory for reuse? still counts. reuse an already charged block? don't count it twice. so "full" here means capacity we hold, not just live objects.

round-robin after every step across release, cycle checking and allocator work. skip empty queues, remember whose turn is next across ticks. don't drain one whole queue before giving the others a turn.

#### what if memory still runs out?

more budget at 85% doesn't guarantee gc can catch up. allocation can still outrun cleanup, and the active batch holds its own members until their check and release work finish.

the rule is: allocation fails with a vm out-of-memory error if it can't fit inside the ordinary allowance. fail before exposing a half-created object. reporting that error must not need another managed allocation, for obvious reasons.

no surprise unlimited gc pause and no silently going past the heap limit. the host can explicitly give cleanup more time and retry at a safe operation boundary, or terminate the script.

collector runs out of trial scratch space? stop admitting members. before a batch can run, its member records must already provide the space needed for count tracking, marks, queues and cleanup. if an internal failure requires aborting before commit, disable trial tracking in one state change, then retire records and pins incrementally; real counts were never trial-decremented, so they need no rollback. after commit, finish the existing release jobs rather than aborting published death. required release/barrier bookkeeping cannot depend on a fallible allocation.

configure `cycle_batch_member_limit` and `cycle_batch_scratch_bytes` explicitly. a capacity-obstructed cycle is reported with the relevant limit; changing that limit re-enables its queued check. this trades collectible graph size against memory use without silently turning a budgeted collector into a blocking one.

so that's the tradeoff: keep the cleanup time budget and memory cap, allow allocation to fail. we can't promise unlimited successful allocations, limited memory and limited cleanup time all at once.

### Fibers

A coroutines. a `Fiber` carries its own register and call-frame stack, plus a `resumer` pointer so you can't resume something that's already someone else business.

fiber-to-fiber switching doesn't cost gc work budget, but it still counts as vm execution. otherwise two fibers can keep switching forever and cleanup never gets a turn.

so the vm scheduler gets control back after an instruction limit or at defined safepoints, and the host can regain control between execution slices. native calls need to cooperate too, vm can't keep execution responsive if a native function just blocks forever.

### Host bindings

Host binding API: native Odin and Wasm embeddings use explicit typed named
callbacks. Registration supplies argument/result Value kinds and a callback;
names resolve to numeric NATIVE IDs outside execution. Argument strings are
borrowed Odin strings. Return helpers copy Odin strings or retain borrowed
managed results, preserving VM ownership.

### Non-blocking I/O

this is a required part of the runtime. a tiny gc pause doesn't help if reading a file blocks the vm thread for 200ms.

submit an operation, get a request handle, then either keep running or await it from a fiber. await suspends that fiber, not the whole vm. a ready result may complete immediately, but completion never runs arbitrary user code recursively inside submission.

operations that can block despite looking like I/O helpers (file metadata, name resolution, some file APIs) run on a bounded native worker pool.

workers never touch managed objects. they return native completion records for the vm thread to adopt.

#### who owns the request and its bytes?

a pending request is registered as a vm root for its waiting fiber and any required managed handles. dropping the user's request reference doesn't free memory still in use by the OS/worker. the request stays registered until completion or acknowledged cancellation, then its roots are released on the vm thread through ordinary ref-c helpers.

workers use owned native buffers. reserve and charge buffer bytes before submission. managed data passed to a write is copied in bounded chunks or transferred through an explicitly supported buffer-ownership operation; don't hide a huge synchronous copy inside "async write". read results are adopted or converted in bounded chunks too. buffers being used by native work must not be concurrently mutated through a managed alias. while chunked write staging is pending, the source must be immutable or exclusively borrowed by the request until staging finishes. the runtime rejects conflicting access; it cannot copy changing bytes and pretend that was a consistent submission.

configure `io_request_limit`, `io_buffer_limit_bytes`, `io_waiter_limit`, `io_completion_max_units` and `io_completion_budget_us`. native request/buffer storage has its own charged allowance, reported alongside managed heap usage; it does not disappear from memory accounting just because gc cannot see it. reserve capacity before transferring an adopted result into the managed allowance, and release its native charge only when the transfer succeeds.

request count, outstanding buffer bytes and completion storage have limits. reserve one completion record per accepted request, so a full queue cannot lose a completion or strand ownership. a try-submit operation returns would-block/resource-limit when capacity is exhausted. an await-submit operation instead queues a fiber in a bounded FIFO waiter queue, rooted until admission or cancellation; exceeding the waiter limit returns resource-limit too. submission must not wait on the vm thread for a worker slot.

#### completion is work too

poll ready completions without waiting during an active scheduling turn. process a bounded number or a time-limited chunk, update request state, and put waiting fibers on the runnable queue. those fibers run under the ordinary execution budget. a burst of 10000 completed reads doesn't get to drain every callback before gc or other fibers have a turn.

the scheduler fairly alternates runnable execution, I/O completion handling and memory maintenance, each with its own work/time allowance. none of those queues must drain before another can run. the host may wait for events only when it chooses to idle; the vm's active poll/step operation never waits for I/O readiness.

cancellation is a request, not proof that native access stopped. completion/cancellation races resolve to one terminal result on the vm thread. keep buffers and roots until the native side confirms it is finished; late completions use generation-checked request ids and are consumed safely. report I/O errors through the request result, not by losing the fiber or unwinding a worker into managed code.

shutdown stops new submissions, requests cancellation, and drains acknowledgements incrementally. blocking native teardown belongs to an explicit host shutdown/wait operation, never an ordinary vm tick.

required checks: a slow read while other fibers progress, a completion flood, cancellation racing completion, a dropped request handle, worker/queue saturation, and large buffer conversion. measure submission and completion time too; calling an API asynchronous isn't enough.
