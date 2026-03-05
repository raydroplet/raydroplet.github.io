---
layout: post
title: "Project ABM"
date: 2026-03-05
---

> [!Warning] WIP. this is a draft. it is currently being written.

## Introduction

Games are fake, and the industry is obsessed with visuals.

> Cyberpunk screenshot with a footnote

They do look incredible, but underneath, every system, be it collision, animation, or AI, is there for convenience, not as an actual part of it, acting mostly to support the illusion you're seeing.

- explain how an game engine has different systems
- explain how they have difficult coordinating with each other since each one only cares about it's own state and simulation needs
- talk about all the wiring needed to keep everything cohesive and that's a development difficulty and time consuming

------
### Not so intelligent

> skyrim bucket (video/gif) scene with a footnote

Even when complex behavior looks real at first, it’s usually scripted. That’s because those entities can’t actually perceive the world the way we do.

- talk about how perception works in usual indie/AAA games
- talk about it's limitations

### Trully alive worlds

The most authentic experiences, however, aren't locked to a staged environment. Instead, they place the player in a world that's actually alive. One that traditional scripting cannot replicate.

- talk about games that actually simulate a world in a emergent way
- rain world, dwarf fortress are the two examples that may be talked here

Games like this are rare, because standard engines are built for orchestrating a scene, but to get a rich simulation, we need to build for emergence.

### A simple model for perception

- briefly explain that besides all the blows and whistles, backend simulations relly on simple logic and elements.
- explain the wolf model (it's already explained below)


Let's say I draw a circle, which represents the bounds of a player, and some trees scattered across the scene. As the player walks, they emit footsteps that exist for a short period of time. If there is something in range that perceives noise, like a wolf hiding in the forest, it could react to sounds and move accordingly. We can also have a vision cone for the wolf. So the wolf now has a direction he may be looking at. It turns in reaction to a certain noise, the footstep, sees the player, and starts walking in their direction, while avoiding the trees, considering they block its path.

- conclude saying we will be implementing this in rust

---

## Designing a Perception Engine

- introduce with something like "Considering the wolf and tree example, how to represent all those overlapping shapes and intersections in code?"

### The naive Approach
- to be elaborated

The simplest approach is to create a signal struct, and throw everything into a massive list. At each scan, the entities which care about their surroundings check every other to see if there's any intersection between their shapes, in order to decide if it's a relevant interaction or not. This, however, does not scale well as the simulation grows. Since as the number of entities increases, the required checks rise exponentially. Instead, we need some sort of spatial partitioning, so that there's only a need to query the signals in close proximity.

### The sparse grid
- to be elaborated

A better solution is to have what is called a sparse grid. By mapping X, Y coordinates to a tile object, and having it be a key to a hashmap, with the values being an array of signals, we can store those based on their positions. The way it works is that the world is divided in a grid, and only the populated tiles hold a list of their current signals, hence the name sparse. When an entity needs to query for close interactions, it just needs to check its surroundings. For any tile that is defined, it simply iterates the items stored there.

This solves the previous scaling issue, but it introduces another problem. Since the engine interprets signals as points when determining which tile they belong to, if any of them are bigger than the tile itself, we may run into a case where a far away signal that should be perceived is ignored, because the system only scans the tiles in close proximity.

### The multi level grid
- to be elaborated

The fix here is straightforward. Just have bigger tiles. We go back to the tile struct and add another field, the level. Each level gets mapped to a specific tile size. Every time a new signal is added to a scene, besides its position, its size is also taken into consideration when deciding the tile it belongs to, by placing it in the lowest level where the cell is big enough to hold it.

The result is essentially being able to stack multiple grids on top of each other. The signal field then keeps track of the active levels, and on each scan, we iterate those, checking the surroundings. No matter how close or distant the signal may be, this guarantees that it will get perceived just fine.

------
## Blows and whistles
- to be elaborated

With our perception problem solved, I shifted my energy into making this more of a usable engine. To move away from static circles and allow our entities more dynamic behavior, I decided to integrate an Entity Component System, using the hecs library. First, we define our components. These are pure data structs representing the properties of our objects, such as Transform for positions, Velocity for movement, and the Emitter itself. The ECS allows us to associate a unique identifier with a combination of those data components, and write isolated systems to manipulate them, like one for basic physics.

Finally, the last thing we'll be needing is rendering. We add yet another function, the render. Its purpose is to gather information about the current state of the system, and pack the relevant data into a FrameData object. This FrameData is then sent, as needed, to a render context, so that the renderer may interpret and draw it as it sees fit.

---
## A more realistic perception
- to be elaborated

Although the system works as intended, the previous scan was too permissive. I wanted some sort of mechanism that allows a signal to be occluded. To achieve this, I defined a special scan function. It works like this.

We first do our usual spatial scan, and sort the returned targets by distance, from closest to farthest. Then we define projection, shadow, and visibility bitmasks, to assist in identifying which portions of the signals are actually seen or hidden. We start with the closest one, project its shape onto the outer perimeter, and map this range onto the projection mask. Next, we subtract the shadow mask from the projection, resulting in what is actually seen. In this case, nothing is blocked yet.

We repeat the process. The signals are blockers, so they get mapped to the shadow mask. On our fourth signal, something different happens. After subtracting the shadow mask from the projection, the blocked sections are filtered out, leaving the signal partially visible. The last signal is fully behind a shadow, so it gets hidden completely.

Although having a limited number of bits is not suited for rendering, where you need pixel-perfect ranges, this simplified model maps nicely to our goal of having more realistic perception and emergent behavior.

----
## End Result
- show the 1000 bouncing balls
- show the wolf example
