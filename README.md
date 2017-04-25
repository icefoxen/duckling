# 🦆duckling 🦆

An experiment in making a massively-parallel entity-component system for video games.  Right now it's a bit of an ugly duckling, but maybe someday it will grow into a beautiful swan...

...nah, sounds like too much work.

# What's cool about it?

In an entity-component system you generally organize your game state objects as a big table, more or less:

Name      | Position | Velocity | ...
----------------------------------------
obj1      | 1, 1     | 0, 1     | ...
obj2      | 1, 5     | 0, -1    | ...
building1 | 0, 0     | XXXXX    | ...
...       | ...      | ...      | ...

Each row is an entity, each column is a component, and each system is a function that takes a set of columns and Does Stuff.  In most of the multithreaded ECS's I've seen, namely `specs` and `constellation`, each system can run in its own thread, and it figures out which threads can run on its own based on the data dependencies between them.  The data dependency rules for multithreading are pretty simple, conceptually: Any number of threads can read a cell in the aboe table, XOR one thread can write to a cell.  Usually the way we handle this is is, if you try to write to a cell that is being read from or written to, or read from a cell that is being written to, that thread blocks until the data becomes available.  This seems a little over-simplistic but works pretty well in practice; instead of trying to figure out apriori what order things should be run in (which you usually can't dictate anyway), things mostly work themselves out and making a meh system better becomes a matter of controlling where locks happen.

Great, but there's a problem: It doesn't scale up.  You're usually only going to have a few really compute-intensive systems in a game, and often they'll need to do stuff with the same data.  And it means you have to break systems up into multiple pieces to try to make them more parallel.  For example, say a game has a physics system that uses up 50% of its run time, an AI system that uses up 25%, and everything else uses up the last 25%.  If you run it on a single-core system, ignoring data dependencies, 

In an FPS game this doesn't really matter much because the game is designed such that the game world only gets so big anyway.  There's never enough monsters on the screen that you need more than one core per system, ideally.  


...oh, and it's written in Rust.  But that's what all the cool kids are doing now anyway, right?

# How complete is it?

You can make entities with two components.  No more, no less.  And you can't remove entities.  So that should give you an idea.

# How fast is it?

