---
layout: post
title: "Project ABM"
date: 2026-03-05
---

## Introduction

Games are fake. The industry is obsessed with visuals, and to be fair, they do look incredible. But underneath that shiny surface, every other system—be it collision, animation, or AI—is often just there for convenience. It isn't an actual part of a living world; it acts mostly to support the illusion you're seeing.

Even when complex behavior looks real at first, it’s usually heavily scripted. That’s because the entities in traditional engines can’t actually perceive the world the way we do. The most authentic experiences aren't locked to a staged environment. Instead, they place the player in a world that's actually alive—one that traditional scripting cannot replicate.

Games that achieve this are rare because standard engines are built for orchestrating a scene. To get a rich simulation, we need to build for emergence.

But how do we do that?

## The Perception Problem

If we wish for smarter entities, we need a way to represent the world and build some systemic way for them to actually perceive it. 

Initially, I had envisioned a massive engine with many key pillars, but I needed to validate whether any of these ideas were actually feasible in the first place. I picked **perception** to start, and the plan I had seems deceptively simple at first glance.

Let's say we draw a circle representing the bounds of a player, with some trees scattered across the scene. As the player walks, they emit footsteps that exist for a short period. If there is a wolf hiding in the forest that perceives noise, it could react to those sounds and move accordingly. We can also give the wolf a vision cone. It turns in reaction to the noise, sees the player, and starts walking in their direction while avoiding the trees that block its path.

<video autoplay loop muted playsinline width="100%">
  <source src="assets/01-perception-basics.mp4" type="video/mp4">
</video>

This is a very small example, but it shows we can use simple shapes to represent a wide range of behaviors—vision, collisions, sound, and many others—while having the entities that care about them react accordingly. 

## The Signal Field & The Core Engine

With the plan laid out, we can start working on the implementation. 

I defined a simple `Engine` struct with a `tick()` function. This is the core loop of the simulation. Every time this runs, it steps the entire world forward by a fixed fraction of a second. The engine also holds the heart of the system, which I'll be calling the **Signal Field**.

<video autoplay loop muted playsinline width="100%">
  <source src="assets/02-engine-structs.mp4" type="video/mp4">
</video>

Considering the wolf and tree example, how do we represent all those overlapping shapes and intersections in code?

### The Naive Approach

The simplest approach is to create a signal struct and throw everything into a massive list. At each scan, the entities which care about their surroundings check every single other entity to see if there's any intersection between their shapes.

<video autoplay loop muted playsinline width="100%">
  <source src="assets/03-naive-scan.mp4" type="video/mp4">
</video>

This determines if an interaction is relevant, but it **does not scale well**. As the simulation grows and the number of entities increases, the required checks rise exponentially ($O(N^2)$). Instead, we need a form of spatial partitioning so that we only query signals in close proximity.

### The Grid Solution

A better solution is to use a **sparse grid**. By mapping `x, y` coordinates to a `TileKey` object and using it as a key in a `HashMap` (where the values are arrays of signals), we can store entities based on their physical positions.

<video autoplay loop muted playsinline width="100%">
  <source src="assets/04-sparse-grid.mp4" type="video/mp4">
</video>

The world is divided into a grid, and only the populated tiles hold a list of their current signals. When an entity needs to query for close interactions, it just checks its surrounding tiles and iterates through the items stored there.

### The Multi-Level Grid

This solves the initial scaling issue, but introduces a new problem. Since the engine interprets signals as points when determining which tile they belong to, what happens if a signal is bigger than the tile itself? We run the risk of ignoring a massive, far-away signal that *should* be perceived because our system only scans the immediate neighboring cells.

The fix is straightforward: we need bigger tiles. 

We go back to the tile struct and add another field: the `level`. Each level maps to a specific tile size (e.g., Level 0 = 1x1, Level 1 = 2x2, Level 2 = 4x4). 

<video autoplay loop muted playsinline width="100%">
  <source src="assets/05-multi-level-grid.mp4" type="video/mp4">
</video>

Every time a new signal is added to a scene, its size is taken into consideration. We place it in the lowest level where the cell is big enough to hold it. The result is essentially stacking multiple grids on top of each other. The signal field keeps track of the active levels, and on each scan, we iterate through them. This guarantees that massive or distant signals are always perceived correctly.

## The Missing Pieces

### The Entity Component System

With our perception problem solved, I shifted my energy into making this more of a usable engine. To move away from static circles and allow our entities more dynamic behavior, I integrated an **Entity Component System (ECS)** using the `hecs` crate.

<video autoplay loop muted playsinline width="100%">
  <source src="assets/06-ecs-components.mp4" type="video/mp4">
</video>

First, we define our components. These are pure data structs representing the properties of our objects: `Transform` for positions, `Velocity` for movement, and the `Emitter` itself. The ECS allows us to associate a unique identifier with a combination of those data components, and write isolated systems (like basic physics) to manipulate them.

### The Renderer Architecture

Finally, we need rendering. We add a `render()` function whose purpose is to gather information about the current state of the system and pack the relevant data into a `FrameData` object.

<video autoplay loop muted playsinline width="100%">
  <source src="assets/07-render-context.mp4" type="video/mp4">
</video>

This `FrameData` is safely sent across the boundary to a distinct presentation context, allowing the renderer to interpret and draw the simulation state without tangling the engine logic.

### Tooling & Debugging

With everything working, we finally have some visual output. I used the `egui` library to handle both the rendering and the tooling UI. 

I built three main windows: the monitor (tracking simulation data), the scene hierarchy (to select entities), and the inspector (to modify components on the fly). Right now, every moving dot on screen is independently scanning the multi-level grid, detecting nearby signals, and resolving its own collisions in real-time.

## Occlusion via Bitmasks

Although the system worked as intended, the initial spatial scan was too permissive. I wanted a mechanism that allows a signal to be occluded. To achieve this, I defined a specialized scan function.

We first do our usual spatial scan and sort the returned targets by distance (closest to farthest). Then, we define **projection**, **shadow**, and **visibility bitmasks** to identify which portions of the signals are actually seen or hidden.

<video autoplay loop muted playsinline width="100%">
  <source src="assets/08-bitmask-occlusion.mp4" type="video/mp4">
</video>

Here is how it evaluates:
1. We start with the closest signal, project its shape onto the outer perimeter, and map this range onto the **projection mask**. 
2. We subtract the **shadow mask** from the projection, resulting in what is actually seen. If nothing is blocking it, it is fully visible.
3. If the next signals are physical blockers, they get mapped directly to the shadow mask.
4. When we scan the next normal signal, we subtract the updated shadow mask from its projection. The blocked sections are filtered out automatically at the bit-level, leaving the signal only partially visible.
5. If a signal is fully behind a shadow, its resulting bitmask is empty, and it is hidden completely.

Although having a limited number of bits is not suited for high-end rendering (where you need pixel-perfect ranges), this simplified bitwise model maps perfectly to our goal of having fast, realistic perception and emergent behavior on the CPU side.

## Showcase

*(Implement the wolf example from the start to show the engine running live)*

I had to simplify some explanations here, but if you want to dive deeper into the code, the engine is fully open-source. You can find the repository on my GitHub.

***
