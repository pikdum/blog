+++
title = 'Building a World of Warcraft server in Elixir: 2024 Update'
date = 2024-12-23
draft = true
tags = ['elixir', 'World of Warcraft', 'programming', 'update']
+++

[Thistle Tea](https://github.com/pikdum/thistle_tea) is my World of Warcraft private server project, started in June 2024.
I wrote a [blog post](/thistle-tea/) back then journaling the first month of development.
Since then, I've been working on the project bit by bit and took a break for December.
This is a follow-up post to highlight progress since then, so I'd recommend skimming the original post first.

## Getting Started

![](./20241019_18h17m47s_grim.avif)

When I first shared the project, it was a bit hard to get running locally.
There were some manual steps to generate the `mangos0.sqlite` and `dbc.sqlite` databases, and not much documentation.
Things were also pretty cluttered, so I re-organized all the code.

Things are in a much better state now, with everything documented in the [README.md](https://github.com/pikdum/thistle_tea/blob/master/README.md).
I've scripted out generating the necessary databases, and the only tricky part now is sourcing a Vanilla 1.12.1 client.
The client is necessary to generate the `dbc.sqlite` and map files, which cannot be distributed with the project.

For contributors, I've created a [Discord channel](https://discord.gg/dSYsRXHDhb).
Feel free to join if you're interested in following the project or contributing.
Contributors especially welcome, it'd be a lot of fun (and motivating) to hack on this with others.

## Pathfinding

{{<video src="./recording_1734995598103-[00.07.150-00.11.920].webm">}}

I briefly tried to get this working during the first month, but couldn't get things working very well.
It was scrapped, replaced with mobs just randomly rotating.

I've since integrated with [namigator](https://github.com/namreeb/namigator), through [namigator-rs](https://github.com/gtker/namigator-rs) and [rustler](https://github.com/rusterlium/rustler).
I originally tried to use the C++ library directly, but wasn't having much luck - Rust was a lot easier to get working.
It's really neat to be able to Rust libraries pretty much directly in Elixir.

Namigator works by reading the game maps and doing proper pathfinding to account for obstacles and terrain.
For random wandering around a point, a new point can be found with `find_random_point_around_circle/3` and then a path with `find_path/3`.
Following that path is handled by the `:follow_path` message, which handles updating state and then queuing up another `:follow_path` message when the mob will be ready for the next point.

It's been working pretty well and is awesome to see mobs moving properly, but there are still some glitches to figure out.

## Mob Behavior

I couldn't think of a better name than behavior for this, but it describes what sort of AI a mob is currently following.

Right now there are three:
* **ThistleTea.WanderBehavior** - random wandering around a point
* **ThistleTea.FollowPathBehavior** - following waypoints
* **ThistleTea.AttackBehavior** - chasing a player during combat

Mobs either have wandering or following a path as their default behavior.
If wandering, they'll have a certain range to stay in from their initial part.
If following a path, they'll have some waypoints to follow.
The attack behavior is switched to when attacked, right now it's incredibly basic and only attempts to follow the player.

To keep CPU use low, mobs will idle until a player is within observation range and only then load a behavior.
Behaviors are implemented as GenServer processes too and interact with the main mob process by passing messages.
This way they can be easily started/stopped whenever.

## Combat

![](20240926_02h27m20s_grim.avif)

I've added basic support for auto attacks, only with the mainhand weapon, using the weapon's speed and damage data.
A lot of checks, like range or orientation, are still missing though.

After killing a mob, it will now respawn.
This is handled by setting a timer to kill the process, so it's restarted by the supervisor in the original state.

When in combat, mobs will now follow the player.
This is very rough right now and mobs will noticeably teleport a bit.
I plan to look into what packets other implementations are sending for this, there's probably something I'm missing to make this more smooth.

## Optimization

![](<Screenshot 2024-12-23 at 17-52-25 Phoenix LiveDashboard.avif>)

I needed some way to see how long packets were taking to handle, so I wired up `:telemetry` to get some timing information.
This showed that movement packets took a long time, since it was also handling spawning and despawning entities as the player moved around.
To optimize, I moved that into a period task instead that runs every second.

Another issue was storing every entity (~100k) in a regular list and looping the entire thing to check if it's in range.
I switched that to a spatial hashing implementation, which uses ETS and puts entities inside of cells.
Instead of checking every entity, now only nearby cells need to be checked.
This is significantly faster, but I lost the exact benchmarks.

I also looked into octrees instead of spatial hashing.
Octrees had faster query performance, but updating a position was worse.
The implementation was also more complex, especially with needing to rebalance the tree.
So to keep things straightforward, I went with spatial hashing instead.

Mob behavior idling was implemented to prevent wasting CPU cyclings on moving mobs around when there's nobody around to observe it.

## Web

![](<./Screenshot 2024-12-23 at 17-42-52 ThistleTea · Phoenix Framework.avif>)

I wanted this project to have a web aspect, so I wired up Phoenix.
I figured it'd be neat to have a map that shows real-time location of players, so I started with that.

Some technologies used:
* **gdal2tiles.py** - generate map tiles from image
* **OpenLayers** - render the map
* **Nx + Evision** - homography to transform game coordinates to map coordinates
* **LiveView** - the component
* **R2** - hosting the tiles

I originally started with Leaflet, but found OpenLayers to render more smoothly when panning around and similar.
The whole thing is wrapped in a LiveView that initializes the map and places players on it.
Player positions are updated every second.

I made a landing page with information on how to connect to the instance and embedded the map there.
The test site is visible at [https://thistle_tea.pikdum.dev](https://thistle_tea.pikdum.dev).
Feel free to connect and play around if you have a Vanilla 1.12 client handy.

I've also wired up Phoenix LiveDashboard to get some monitoring and telemetry visualizations.
Right now I only have timings on handling packets, but it'd be useful to add periodic tasks and more here later.

## Debug Commands

![](./20240906_00h52m42s_grim.avif)

To help test, I wired up some debug chat commands.
The whole list can be viewed with **.help**, but teleporting is my favorite.

Some neat places to go to:
* **GM Island** - .go xyz 16222.1 16252.1 12.5 1
* **Old Ironforge** - .go xyz -4841.46 -1062.67 501.74 0
* **Programmer Isle** - .go xyz 16303.2 16318.1 69.44 451
* **Emerald Dream** - .go xyz 3134.72 -3179.92 143.28 169

ChatGPT was useful to get even more interesting places to check out.

## NPC Gossip

![](./20241110_05h26m07s_grim.avif)

This is when you right click on a NPC and get a dialog window, with maybe some options.
An example is asking a city guard for directions.
I've also managed to get some quests showing up here, but I think these are all sort of delivery quests.

## Game Objects

![](./20240921_02h52m24s_grim.avif)

Previously I was only spawning mobs into the world, but now I'm also spawning game objects.
These are things like dynamic seasonal decorations, mailboxes, chairs, etc.
The implementation is a lot like mobs, with each game object getting a GenServer process.
There's still a lot of work here - like right now I'm just loading all game objects in the database and getting all seasonal decorations simultaneously.

## Future Work

My plan right now is to concentrate on breadth rather than depth.
I'd like a little bit of everything working, even if it's not complete or anywhere near perfect.
After that, then a lot of time can be spent refactoring and polishing the implementations.

If anybody's interested in helping out, there's the Discord channel and a bunch of GitHub issues.
