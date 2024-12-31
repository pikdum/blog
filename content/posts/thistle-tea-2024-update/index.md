+++
title = 'Building a World of Warcraft server in Elixir: 2024 Update'
date = 2024-12-23
draft = false
tags = ['elixir', 'World of Warcraft', 'programming', 'update']
+++

[Thistle Tea](https://github.com/pikdum/thistle_tea) is my World of Warcraft private server project, started in June 2024.
I wrote a [blog post](/posts/thistle-tea/) back then journaling the first month of development and have since been working on the project bit by bit.
This is a follow-up to highlight progress since then, so I'd recommend skimming the original post first.

## Pathfinding

{{<video src="./recording_1734995598103-[00.07.150-00.11.920].webm">}}

I briefly tried to get mobs walking around during the first month, but couldn't get it working very well.
Since I didn't have map data handy, I couldn't account for terrain changes or buildings.
Didn't want to spend too much time on it, so I settled for randomly changing mob orientation to at least have something.

I've since integrated with [namigator](https://github.com/namreeb/namigator), through [namigator-rs](https://github.com/gtker/namigator-rs) and [rustler](https://github.com/rusterlium/rustler).
With `mix build_maps`, namigator parses the game map files to generate navigation meshes and similar.
These are then used at runtime for pathfinding, exposed through the **ThistleTea.Pathfinding** module.
Pathfinding is a huge project by itself, so it's great to have this library and not need to build something from scratch.

The Rust bindings were straightforward to wire up, since Rustler handles the heavy lifting of building a NIF.
I previously tried writing a NIF using the C++ library directly, but didn't have any luck.
Integrating code written in different languages together is really neat and I haven't done similar before.
Was surprised by how easy Rustler made it.

For getting a mob to randomly wander around, the process looks something like:
1. use **find_random_point_around_circle/3** to find a new point
2. use **find_path/3** to find a path to that new point
3. send a **:follow_path** message to the mob
4. **:follow_path** updates state and queues another **:follow_path**, until the entire path is traversed

Works great most of the time, but there are still some bugs to figure out.

## Mob Behavior

This describes what sort of AI routine a mob is currently following:
* **ThistleTea.WanderBehavior** - random wandering around a point
* **ThistleTea.FollowPathBehavior** - following waypoints
* **ThistleTea.AttackBehavior** - chasing a player during combat

The default behavior for a mob is either wander, follow path, or none.
If wandering, they'll stay within a specified range of their initial point.
If following a path, they'll have a list of waypoints to visit in a loop.
Mobs switch to the attack behavior when attacked and start chasing the player.

To keep CPU usage low, mobs idle until a player is within observation range and only then load a behavior.
Once all players are out of range, mobs unload their behavior and go back to idling.
Behaviors are implemented as GenServer processes and interact with the main mob process by passing messages, so they can easily be started/stopped/swapped whenever.

## Combat

![](20240926_02h27m20s_grim.avif)

I've added basic support for auto attacks, only with the mainhand weapon, using the weapon's speed and damage data.
A lot of checks, like range or orientation, are still missing though.

After killing a mob, it will now respawn.
This is handled by setting a timer to kill the mob process, so it's restarted by the supervisor in the original state.

When in combat, mobs will now follow the player.
This is very rough right now and mobs will noticeably teleport a bit.
I plan to look into what packets other server implementations are sending for this, there's probably something I'm missing to make this more smooth.

## Optimization

![](<Screenshot 2024-12-23 at 17-52-25 Phoenix LiveDashboard.avif>)

I needed some way to see how long packets were taking to handle, so I wired up **:telemetry** to get timing information.
This showed that movement packets took significantly longer than others, since spawning and despawning entities was being handled as the player moved around.
To get that expensive work out of a packet handler function, I moved it into a periodic task instead that currently runs every second.

Another issue was storing every entity (~100k) in a regular list and looping the entire thing to check if it's in range.
I switched that to a spatial hashing implementation, which puts entities inside of cells and uses [ETS](https://www.erlang.org/doc/apps/stdlib/ets.html) under the hood.
Instead of checking every entity, now only nearby cells need to be checked.
This is significantly faster, but I lost the exact benchmarks.

I also looked into using an [octree](https://en.wikipedia.org/wiki/Octree) instead of spatial hashing.
Octrees had faster query performance, but updating a position was worse.
The implementation was also more complex, especially with needing to rebalance the tree.
So to keep things straightforward, I went with spatial hashing instead.

Mob behavior idling helps with both CPU and memory use, since it allows us to load navigation meshes on demand and only simulate movement for nearby mobs.

## Web

![](<./Screenshot 2024-12-23 at 17-42-52 ThistleTea · Phoenix Framework.avif>)

I wanted this project to have a web aspect, so I wired up [Phoenix](https://phoenixframework.org).
I figured it'd be neat to have a map that shows real-time location of players, so I started with that.

Some technologies used:
* **gdal2tiles.py** - generate map tiles from image
* **OpenLayers** - render the map
* **Nx + Evision** - homography to transform game coordinates to map coordinates
* **LiveView** - the component
* **R2** - hosting the tiles

I originally started with [Leaflet](https://leafletjs.com), but found [OpenLayers](https://openlayers.org) to render more smoothly when panning around and similar.
The whole thing is wrapped in a LiveView that initializes the map and places players on it.
Player positions are updated every second.

For a proper landing page, I added some information on how to connect and embedded the map.
The test site is visible at [https://thistle_tea.pikdum.dev](https://thistle_tea.pikdum.dev).
Feel free to connect and play around if you have a Vanilla 1.12 client handy.

I've also wired up [Phoenix LiveDashboard](https://github.com/phoenixframework/phoenix_live_dashboard) to get some monitoring and telemetry visualizations.
Right now there's only timings on handling packets, but it'd be useful to add periodic task telemetry and more here later.

## Debug Commands

![](./20241231_16h02m19s_grim.avif)

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
I was hoping to get proper quests showing up, but that's likely something different than gossip.

## Game Objects

![](./20240921_02h52m24s_grim.avif)

Previously I was only spawning mobs into the world, but now I'm also spawning game objects.
These are things like dynamic seasonal decorations, mailboxes, chairs, bonfires, etc.
The implementation is a lot like mobs, with each game object getting a GenServer process.
There's still a lot of work here, like properly handling seasonal decorations so they all aren't active at once.

## Contributing

![](./20241019_18h17m47s_grim.avif)

When I first shared the project, it was a bit hard to get running locally.
There were some manual steps to generate the **mangos0.sqlite** and **dbc.sqlite** databases and not much documentation.
Things were also pretty cluttered, so I had to go through and organize all the code.

Things are in a much better state now, with everything documented in the [README.md](https://github.com/pikdum/thistle_tea/blob/master/README.md).
I've scripted out generating the necessary databases and the only tricky part now is sourcing a Vanilla 1.12.1 client.
The client is necessary to generate the **dbc.sqlite** and map files, but cannot be distributed with the project.

For contributors, I've created a [Discord channel](https://discord.gg/dSYsRXHDhb).
Feel free to join if you're interested in following the project or helping out.
Contributors especially welcome, it'd be a lot of fun (and motivating) to hack on this with others.

## Future Work

My plan right now is to concentrate on breadth rather than depth.
I'd like a little bit of everything working, even if it's not complete or anywhere near perfect.
After that, a lot of time can be spent refactoring and polishing the implementations.

Some items on my short list:
* quests
* loot + inventory management
* mobs attacking players

If anybody's interested in helping out, there's the [Discord channel](https://discord.gg/dSYsRXHDhb) and a [bunch of GitHub issues](https://github.com/pikdum/thistle_tea/issues).
