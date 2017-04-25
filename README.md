# 🦆duckling 🦆

An experiment in making a massively-parallel entity-component system for video games.  Right now it's a bit of an ugly duckling, but maybe someday it will grow into a beautiful swan...

...nah, sounds like too much work.

# How complete is it?

You can make entities with two components.  No more, no less.  And you can't remove entities.  So that should give you an idea.

# What's cool about it?

Okay, buckle your seatbelts for a rant...

In an entity-component system you generally organize your game state objects as a big table, more or less:

Name      | Position | Velocity | ...
--------- | -------- | -------- | ----
obj1      | 1, 1     | 0, 1     | ...
obj2      | 1, 5     | 0, -1    | ...
building1 | 0, 0     | XXXXX    | ...
...       | ...      | ...      | ...

Each row is an entity, each column is a component, and each system is a function that takes a set of columns and Does Stuff.  In most of the multithreaded ECS's I've seen, namely `specs` and `constellation`, each system can run in its own thread, and it figures out which threads can run on its own based on the data dependencies between them.  The data dependency rules for multithreading are pretty simple, conceptually: Any number of threads can read a cell in the aboe table, XOR one thread can write to a cell.  Usually the way we handle this is is, if you try to write to a cell that is being read from or written to, or read from a cell that is being written to, that thread blocks until the data becomes available.  This seems a little over-simplistic but works pretty well in practice; instead of trying to figure out apriori what order things should be run in (which you usually can't dictate anyway), things mostly work themselves out and making a meh system better becomes a matter of controlling where locks happen.

Great, but there's a problem: It doesn't scale up.  You're usually only going to have a few really compute-intensive systems in a game, and often they'll need to do stuff with the same data.  And it means you have to break systems up into multiple pieces to try to make them more parallel.  For example, say a game has a physics system that uses up 50% of its run time, an AI system that uses up 25%, and everything else uses up the last 25%.  If you run it on a single-core system, ignoring data dependencies, then everything just takes up that fraction of your CPU.  If you run it on a dual-core system, that divides out, on average, to one core running the physics and one core running everything else.  On a quad-core system though, you have one core running physics, one core AI (and running idle half the time), one core everything else (and also half idle), and the last one completely unused.  And on an eight-core system this gets even worse; you end up with six cores unused, on average, which is a massive amount of wasted capacity that should be going into making a game more awesome.

In an FPS game this doesn't really matter much because the game is designed such that the game world only gets so big anyway.  There's never enough monsters on the screen that you need more than one core per system, ideally.  That means that in this model where systems are the basic unit of threading, it's usually better to optimize a system to run really well in one thread than to try to split it up.  But I like playing simulation games like Stellaris and Dwarf Fortress, where your game simulation can become more or less arbitrarily complicated.  So being able to throw multithreaded computing power and have the game grow, goldfish-like, to fill consume it all is a big win.

It is my belief that if we don't find better ways of exploiting multithreading by default rather than as a hacked-in special case, the CPU manufacturers will never bother giving us more cores.  And that's terrible.  So the goal of this is to experiment with, instead multithreading on systems (sets of columns), you multithread on entities (rows).  That way you can make your simulation scale up to however many cores you have as long as you have more entities than cores.  But making >8 entities seems a lot eaiser than having >8 systems.

In a perfect world we'd be able to parallelize over systems AND entities, that is, both rows and columns in the above table could run in parallel.  Then you could REALLY run it on a supercomputer.  But, one thing at a time.

...oh, and it's written in Rust.  But all new software is written in Rust, right?

# Where do I find out more about this?

Everything I've come up with here comes from:

 * John Carmack: https://www.youtube.com/watch?v=1PhArSujR_A&feature=youtu.be&t=17m34s
 * refD: http://michaelshaw.io/game_talk/game.html#/

I am certainly not an expert.  I really need to find more information about this sort of thing.

# So how do you make this work?

More or less how you'd expect.  The key point from the sources above is "the further you go with this, the more the game starts looking like a distributed system".  Well, that's fine, if it works for supercomputers then it will probably work for Quake as well, right?

At its heart, a system is a function that modifies an entity: `fn(&mut Entity)`.  Running a system just means applying it to the whole list of entities.  If a system doesn't apply to whatever type of entity it's given, as defined by which components that entity has, then it skips it.

Now, shared mutation is awful to begin with, and double-awful for multithreading, so we make the system a pure function: `fn(Entity) -> Entity`.

Now this allocates memory, which is something to avoid when possible in a game, especially for something that's happening a bajillion times per second.  So we double-buffer our game worlds, so we have an old arena and a new arena, and just swap them each frame.  Huzzah, no memory allocation.

Well entities can modify themselves, but they need to be able to modify each other as well.  My canonical mental example is opening a treasure chest in an RPG: You check whether the chest is accessible (read-only), then set its state to "opened" (modify other entity), and add whatever was in it to the player's inventory (modify own entity).  But you can't modify other entities because other threads might be messing with them, so you batch your modification up into an event and send it to the other entity.  So your system is now `fn(Entity) -> (Entity, Events)`  But of course the entity has to receive events as well, so the function signture is `fn(Entity, Events) -> (Entity, Events)`.

Very nice, right?  All mutation is made explicit, and all mutation happens "between" frames (or at least between one system ending and the next beginning) in a basically atomic but also basically instantanious fashion; it's single-threaded, but it's just a buffer swap.

The problem here is that dispatching events is a pain in the butt; if you return a new `Vec<Event>` or whatever you're allocating memory, which you want to avoid, and either way you end up with a big collection fo events to go through to route the messages to the destination entities.  So instead of that, each entity gets its own event queue, and it's wrapped up in an object that gets passed to each entity, so your system is now `fn(Entity, Events, &mut EventQueue)`.  The event queue is also double-buffered, so no allocation has to happen.  The only part that can't be parallelized is adding an event to an entity's queue: that requires a write lock.  If two entities try to send a message to the same target entity at the same time, one will block until the other is done.  On the up side you're just copying your event into an array, so it's not exactly slow, but this is a source of blocking.  On the up side, it is literally the *only* source of blocking.

# How fast is it?

'Cause that's the real point, right?  The benchmark is a slightly-hacked-up version of the
[ecs_bench](https://github.com/lschmierer/ecs_bench) benchmark, which like all benchmarks is awful but is what everyone looks at.
It's even more awful because I'm comparing apples to oranges across different machines, so really these are some of the least
meaningful numbers ever.  But still:

Graphs:

![pos-vel benchmark](https://github.com/icefoxen/duckling/blob/master/figures/pos_vel.png)

![parallel update benchmark](https://github.com/icefoxen/duckling/blob/master/figures/parallel-update.png)

Raw numbers:

## Dual-core server (VM):

 Library         | pos_vel build                 | pos_vel update                 | parallel build                 | parallel update
 --------------- |:-----------------------------:|:------------------------------:|:------------------------------:|:--------------------------------:
 [calx-ecs]      | 331,844 ns/iter (+/- 60,653)      | 17,982 ns/iter (+/- 882)      | 459,280 ns/iter (+/- 47,495)      | 75,199 ns/iter (+/- 1,960)
 [constellation] | 264,420 ns/iter (+/- 26,055) | 7,311 ns/iter (+/- 1,877) | 485,141 ns/iter (+/- 15,568) | 111,954 ns/iter (+/- 69,501)
 [ecs]           | 1,523,169 ns/iter (+/- 532,686)           | 319,704 ns/iter (+/- 150,149)           | 1,499,052 ns/iter (+/- 101,118)           | 3,815,809 ns/iter (+/- 663,224)
 [froggy]        | 958,573 ns/iter (+/- 99,809)        | 18,004 ns/iter (+/- 1,491)        | 2,247,556 ns/iter (+/- 208,079) | 84,085 ns/iter (+/- 29,542)
 [recs]          | 16,421,689 ns/iter (+/- 2,972,704)          | 6,848,831 ns/iter (+/- 3,795,579)          | 21,636,351 ns/iter (+/- 3,082,950)          | 13,946,178 ns/iter (+/- 5,881,502)
 [specs]         | 364,951 ns/iter (+/- 184,508)         | 51,987 ns/iter (+/- 14,097)         | 492,265 ns/iter (+/- 170,958) | 86,700 ns/iter (+/- 31,416)
 [trex]          | 979,435 ns/iter (+/- 77,867)          | 229,093 ns/iter (+/- 51,886)          | 1,909,282 ns/iter (+/- 445,376) | 443,308 ns/iter (+/- 126,689)
 [duckling]          | 459,280 ns/iter (+/- 47,495)          | 75,199 ns/iter (+/- 1,960)          | 3,448,372 ns/iter (+/- 546,962)          | 1,146,826 ns/iter (+/- 381,791)


## Quad-core desktop (i7 4790K):

 Library         | pos_vel build                 | pos_vel update                 | parallel build                 | parallel update
 --------------- |:-----------------------------:|:------------------------------:|:------------------------------:|:--------------------------------:
 [calx-ecs]      | 195,331 ns/iter (+/- 15,037)      | 11,875 ns/iter (+/- 337)      | 279,369 ns/iter (+/- 11,323)      | 51,052 ns/iter (+/- 1,782)
 [constellation] | 172,863 ns/iter (+/- 6,352) | 4,808 ns/iter (+/- 223) | 309,432 ns/iter (+/- 10,112) | 100,042 ns/iter (+/- 15,305)
 [ecs]           | 919,894 ns/iter (+/- 49,128)           | 193,387 ns/iter (+/- 8,319)           | 911,655 ns/iter (+/- 59,144) | 2,228,651 ns/iter (+/- 104,434)
 [froggy]        | 665,024 ns/iter (+/- 19,636)        | 10,130 ns/iter (+/- 385)        | 1,355,981 ns/iter (+/- 61,574) | 39,833 ns/iter (+/- 3,246)
 [recs]          | 8,663,385 ns/iter (+/- 672,072)          | 2,327,478 ns/iter (+/- 567,831)          | 10,788,256 ns/iter (+/- 1,107,606)          | 5,773,911 ns/iter (+/- 1,765,957)
 [specs]         | 204,633 ns/iter (+/- 13,123)         | 26,648 ns/iter (+/- 1,111)         | 286,225 ns/iter (+/- 24,567) | 42,372 ns/iter (+/- 6,943)
 [trex]          | 613,310 ns/iter (+/- 36,629)          | 142,329 ns/iter (+/- 6,242)          | 1,164,912 ns/iter (+/- 137,753) | 300,398 ns/iter (+/- 15,304)
 [duckling]          | 1,908,954 ns/iter (+/- 162,028)          | 174,088 ns/iter (+/- 111,111)          | 1,950,064 ns/iter (+/- 138,703)          | 297,987 ns/iter (+/- 126,111)


## Eight-core desktop (R7 1700):

 Library         | pos_vel build                 | pos_vel update                 | parallel build                 | parallel update
 --------------- |:-----------------------------:|:------------------------------:|:------------------------------:|:--------------------------------:
 [calx-ecs]      | 176,725 ns/iter (+/- 20,154)      | 12,245 ns/iter (+/- 935)      | 241,929 ns/iter (+/- 4,726)      | 71,024 ns/iter (+/- 3,547)
 [constellation] | 144,436 ns/iter (+/- 20,872) | 4,833 ns/iter (+/- 444) | 304,474 ns/iter (+/- 26,561) | 142,104 ns/iter (+/- 10,091)
 [ecs]           | 1,098,158 ns/iter (+/- 83,806)           | 264,841 ns/iter (+/- 30,045)           | 1,116,987 ns/iter (+/- 64,762)           | 3,226,200 ns/iter (+/- 337,694)
 [froggy]        | 585,574 ns/iter (+/- 55,544)        | 7,583 ns/iter (+/- 608)        | 1,394,921 ns/iter (+/- 61,279)        | 95,177 ns/iter (+/- 26,246)
 [recs]          | 9,255,650 ns/iter (+/- 319,178)          | 2,879,176 ns/iter (+/- 157,361)          | 11,734,252 ns/iter (+/- 508,273)          | 6,960,598 ns/iter (+/- 420,790)
 [specs]         | 302,820 ns/iter (+/- 110,025)         | 69,680 ns/iter (+/- 40,156)         | 312,214 ns/iter (+/- 131,336) | 80,826 ns/iter (+/- 698)
 [trex]          | 634,846 ns/iter (+/- 36,926)          | 146,467 ns/iter (+/- 11,642)          | 1,112,914 ns/iter (+/- 59,634) | 312,112 ns/iter (+/- 50,536)
 [duckling]          | 1,891,190 ns/iter (+/- 133,853)          | 190,331 ns/iter (+/- 11,072)          | 1,959,849 ns/iter (+/- 126,833)          | 329,353 ns/iter (+/- 32,939)

## 48-core compute server (Opteron 6176):


 Library         | pos_vel build                 | pos_vel update                 | parallel build                 | parallel update
 --------------- |:-----------------------------:|:------------------------------:|:------------------------------:|:--------------------------------:
 [calx-ecs]      | 500,183 ns/iter (+/- 9,473)      | 37,525 ns/iter (+/- 119)      | 907,678 ns/iter (+/- 27,388)      | 202,474 ns/iter (+/- 1,270)
 [constellation] | 434,830 ns/iter (+/- 343,822) | 14,626 ns/iter (+/- 35) | 861,489 ns/iter (+/- 1,963) | 1,576,463 ns/iter (+/- 509,304)
 [ecs]           | 2,280,898 ns/iter (+/- 42,373)           | 521,050 ns/iter (+/- 19,091)           | 2,376,688 ns/iter (+/- 37,621)           | 6,268,735 ns/iter (+/- 195,149)
 [froggy]        | 1,317,524 ns/iter (+/- 3,016)        | 29,822 ns/iter (+/- 63)        | 4,057,576 ns/iter (+/- 37,515) | 267,346 ns/iter (+/- 74,781)
 [recs]          | 21,453,312 ns/iter (+/- 3,977,101)          | 5,875,710 ns/iter (+/- 387,937)          | 27,582,075 ns/iter (+/- 283,694)          | 12,104,573 ns/iter (+/- 320,076)
 [specs]         | 741,133 ns/iter (+/- 298,069)         | 251,997 ns/iter (+/- 16,617)         | 986,633 ns/iter (+/- 28,471) | 365,614 ns/iter (+/- 28,959)
 [trex]          | 2,161,095 ns/iter (+/- 29,113)          | 555,833 ns/iter (+/- 10,045)          | 3,648,676 ns/iter (+/- 530,292)          | 1,007,904 ns/iter (+/- 2,624)
 [duckling]          | 907,678 ns/iter (+/- 27,388)          | 202,474 ns/iter (+/- 1,270)          | 907,678 ns/iter (+/- 27,388)          | 202,474 ns/iter (+/- 1,270)

# Notes for future work

 * Profile!
 * It would be nice to be able to make it so you don't bother copying entities into a new buffer when they're not being changed.
 * The easy way to do that would be to make each column in the table double-buffered independently with a dirty flag associated
   with it.  This brings some benefits: One, running a system checks the dirty flag of the components it needs and can raise an
   error (or swap buffers, or just keep going, or whatever???) if its "current" state is out of date.  Two, doing a flip buffers
   operation can be cheaper because we don't need to flip non-dirty buffers.  And THREE, it might become possible to run systems
   in parallel as well.  However, the flip-buffers operation is not the expensive and weird part, the deliver-messages operation
   is.  So figure out how that's going to work.
