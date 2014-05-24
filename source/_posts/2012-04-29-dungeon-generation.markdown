---
layout: post
title: 'Dungeon Fodder Update #4: Dungeon generation'
date: '2012-04-29T20:23:00-07:00'
comments: true
categories:
    - Dungeon Fodder
    - Unity3D
    - Game Development
    - Dev Diary
tumblr_url: http://jorgenpt.tumblr.com/post/22102329394/dungeon-generation
alias: [/post/22102329394, /post/22102329394/dungeon-generation]
---

This week I added some rudimentary dungeon generation code to [DungeonFodder](/blog/categories/dungeon-fodder/). You can see what it looks like [on YouTube](http://youtu.be/51Rvmtzo3gg).

{% youtube 51Rvmtzo3gg %}

**EDIT**: Eventually I expect to have a certain chance that a level will be a "special" level that's been made by hand instead of randomly generated. Same for rooms - some of them will be "special" rooms.

There're a lot of improvements that can be made to this approach - and so far it doesn't add much content to the rooms, just the infrastructure. Here's a quick explanation of how it currently works!

### Rooms

Rooms are created as regular 3D [Unity GameObjects](http://unity3d.com/support/documentation/ScriptReference/GameObject.html), complete with scripts and all. Then, the dungeon generation script takes each of these objects, converts their dimensions to a 2D tile-based system and determines where the room has openings for corridors.

Rooms are split into "large" and "small" rooms, so that the engine can produce many small rooms and a couple of large ones, depending on what feel the level should have.

### Placement

The script then chooses a random number of rooms of each size, based on a range for the level. For each room it wants to place, it picks a random rotation (n * 90°), and places it at a random location in the world. If the location is taken by another room, it starts searching for a free spot outwards from the initial random location.

Large rooms are placed before small rooms, to prevent the fragmentation from getting out of hand. A random "large" room is picked to be the "spawn" room.

### Monsters & loot

Each level has a configured range of "monsters per tile", like [0.1, 0.4]. For each room, it picks a random number in that range, and multiplies that number with the "square tileage" of the room. It skips the "spawn" room.

Currently, the spawn algorithm is very naïve. There's also no loot being spawned.

### Paths

Then, each "opening" in a room is connected by the shortest path to a neighboring room (using an implementation of the [A*](http://en.wikipedia.org/wiki/A-star) algorithm), but if a room has more than one "opening" then they are connected to different rooms. We tell our [A*](http://en.wikipedia.org/wiki/A-star) to count unpathed tiles as a bit more expensive than pathed tiles, so that paths will get reused.

Finally, each room is connected to the spawn room to ensure that we don't have any disjoint "sub-dungeons" that aren't connected to the rest of them. This time we configure our [A*](http://en.wikipedia.org/wiki/A-star) to count unpathed tiles as much more (currently 20x) expensive than pathed tiles, to encourage it to not create unneccessary paths.

### Questions? Comments?

Feel free to contact me on jorgenpt@gmail.com or [twitter](http://twitter.com/jorgenpt]. :-)
