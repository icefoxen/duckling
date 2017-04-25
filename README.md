# 🦆duckling 🦆

An experiment in making a massively-parallel entity-component system for video games.  Right now it's a bit of an ugly duckling, but maybe someday it will grow into a beautiful swan...

...nah, sounds like too much work.

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

# How complete is it?

You can make entities with two components.  No more, no less.  And you can't remove entities.  So that should give you an idea.

# How fast is it?

'Cause that's the real point, right?