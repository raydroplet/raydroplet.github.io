---
layout: post
title: "Project ABM"
date: 2026-03-05
---

> This is a write-up of a small perception engine I built in Rust. The goal was simple: give simulated entities a believable sense of the world around them.
>
> Getting there was less simple.

## State of the Art

<figure>
<img src="{{ 'assets/project-abm/cyberpunk.png' | relative_url }}" alt="Cyberpunk aesthetic image">
<figcaption> </figcaption>
</figure>

When you see a game with incredible graphics you're often amazed by how good it looks. This is no coincidence. It's much easier to sell a pretty picture and convince someone to try it out if it looks good on screen. But underneath all this magic there is often a carefully orchestrated mess of systems. Each one caring about its own job, while you, as the player, expects them to all work in harmony. 

You might have a physics engine with its set of rigid bodies, a rendering engine handling meshes, materials and shaders, an audio engine for spatial sound emitters, an AI managing navmeshes and behavior trees and so on. If you, for example, decide that an NPC in your world must also hear audio, you're in a lot of trouble, because an Audio engine may only care about playing sounds to your computer, not having them be perceived by the entities in your world.

Often a lot of wiring up is necessary to keep everything cohesive. That's a development difficulty and also time consuming.

<figure>
<img src="{{ 'assets/project-abm/skyrim.jpg' | relative_url }}" alt="Skyrim NPC with a bucket on their head">
<figcaption> </figcaption>
</figure>

### Trully alive worlds

The most authentic experiences, however, aren't locked to a staged environment. Instead, they place the player in a world that's actually alive. One that traditional scripting cannot replicate.

When you look at games like Rain World or Dwarf Fortress, the magic doesn't come from a level designer planning every single interaction. It comes from emergency. The entities operates reading the state of their surrounds and following certain rules, dynamically reacting to the enviroment without a hardcoded trigger.

Games like this are rare, because standard engines ~~are built for orchestrating~~ a scene, but to get a rich simulation, we need to build for emergence, which has as it foundation the concept of perception.

<figure>
<img src="{{ 'assets/project-abm/rain_world.webp' | relative_url }}" alt="Rain World gameplay">
<figcaption> </figcaption>
</figure>

## Designing a Perception Engine

- why?

### A simple model

<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/motion/wolf_chase.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>

Say I draw a circle, which represents the bounds of a player, and some trees scattered across the scene. As the player walks, they emit footsteps that exist for a short period of time. If there is something in range that perceives noise, like a wolf hiding in the forest, it could react to sounds and move accordingly. 

So the wolf now has a direction he may be looking at, defined by a vision cone. It turns in reaction to a certain noise, the footstep, sees the player, and starts walking in their direction, while avoiding the trees, considering they block its path.

Given this scenario, how do we represent all those shapes and behaviors in code?

### The naive Approach

<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/motion/naive_list.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>

The simplest approach is to create a signal struct, and throw everything into a m/assive list. At each scan, the entities which care about their surroundings check every other to see if there's any intersection between their shapes, in order to decide if it's a relevant interaction or not.

This, however, does not scale well as the simulation grows. Since as the number of entities increases, the required checks rise exponentially. Instead, we need some sort of spatial partitioning, so that there's only a need to query the signals in close proximity.

### The sparse grid

<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/motion/sparse_grid.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>

A better solution is to have what is called a sparse grid. By mapping x/y coordinates to a tile object, and having it be a key to a hashmap, with the values being an array of signals, we can store those based on their positions.

The way it works is that the world is divided in a grid, and only the populated tiles hold a list of their current signals, hence the name sparse. When an entity needs to query for close interactions, it just needs to check its surroundings. For any tile that is defined, it simply iterates the items stored there.

This solves the previous scaling issue, but it introduces another problem. Since the engine interprets signals as points when determining which tile they belong to, if any of them are bigger than the tile itself, we may run into a case where a far away signal that should be perceived is ignored, because the system only scans the tiles in close proximity.

### The multi level grid

<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/motion/multi_grid.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>

The fix here is straightforward. Just have bigger tiles. We go back to the tile struct and add another field, the level. Each level gets mapped to a specific tile size. Every time a new signal is added to a scene, besides its position, its size is also taken into consideration when deciding the tile it belongs to, by placing it in the lowest level where the cell is big enough to hold it.

The result is essentially being able to stack multiple grids on top of each other. The signal field then keeps track of the active levels, and on each scan, we iterate those, checking the surroundings. No matter how close or distant the signal may be, this guarantees that it will get perceived just fine.

## Blows and whistles

<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/motion/ecs.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>

With our perception problem solved, I shifted my energy into making this more of a usable engine. To move away from static circles and allow our entities more dynamic behavior, I decided to integrate an Entity Component System, using the hecs library.

First, we define our components. These are pure data structs representing the properties of our objects, such as Transform for positions, Velocity for movement, and the Emitter itself. The ECS allows us to /associate a unique identifier with a combination of those data components, and write isolated systems to manipulate them, like one for basic physics.

Finally, the last thing we'll be needing is rendering. We add yet another function, the render. Its purpose is to gather information about the current state of the system, and pack the relevant data into a FrameData object. This FrameData is then sent, as needed, to a render context, so that the renderer may interpret and draw it as it sees fit.

## A more realistic perception

<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/motion/occlusion.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>

Although the system works as intended, the previous scan was too permissive. I wanted some sort of mechanism that allows a signal to be occluded. To achieve this, I defined a special scan function. It works like this.

We first do our usual spatial scan, and sort the returned targets by distance, from closest to farthest. Then we define projection, shadow, and visibility bitmasks, to /assist in identifying which portions of the signals are actually seen or hidden. We start with the closest one, project its shape onto the outer perimeter, and map this range onto the projection mask. Next, we subtract the shadow mask from the projection, resulting in what is actually seen. In this case, nothing is blocked yet.

We repeat the process. The signals are blockers, so they get mapped to the shadow mask. On our fourth signal, something different happens. After subtracting the shadow mask from the projection, the blocked sections are filtered out, leaving the signal partially visible. The last signal is fully behind a shadow, so it gets hidden completely.

Although having a limited number of bits is not suited for rendering, where you need pixel-perfect ranges, this simplified model maps nicely to our goal of having more realistic perception and emergent behavior.

## The running system

<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/engine1.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>
<figcaption>A thousand signals colliding scanning their surroundings and colliding<figcaption>

<figure>
<video autoplay loop muted playsinline width="100%" style="border-radius: 8px;">
  <source src="{{ 'assets/project-abm/engine2.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>
<figcaption>The wolf example from the started implemented</figcaption>
</figure>
