+++
title = 'Building a World of Warcraft server in Elixir: 2025 Update'
date = 2025-11-26
draft = false
tags = ['elixir', 'World of Warcraft', 'programming']
+++

This is a follow-up to these posts:

- [Building a World of Warcraft server in Elixir](/posts/thistle-tea/)
- [Building a World of Warcraft server in Elixir: 2024 Update](/posts/thistle-tea-2024-update/)

[Thistle Tea](https://github.com/pikdum/thistle_tea) is a World of Warcraft private server project that I've been working on for a while now. This is a quick update to highlight what's been going on in 2025.

![](./20251121_15h41m10s_grim.avif)

## New Year's Blues

The last big feature I worked on in 2024 was having mobs chase the player they were in combat with.
This surfaced some pain points in the code and I couldn't come up with an implementation I was happy with.
Movement splines weren't working as expected and the mob file was getting pretty cumbersome to work with in general.
Properly working movement splines would allow sending a sequence of movement points within a single packet, but I had to hack around this and do extra work server side instead.
This led me to take a step back and come up with ideas on how to clean things up.

Code was largely organized around a few large GenServers and it was pretty difficult to reason about individual parts of the system.
Network concerns were also mixed in with game logic, which wasn't very fun to work with.
Ideally I'd be able to handle game behavior at a different layer than packet layouts, but it was all intermingled.

After lots of thinking, experimentation, and procrastination, I adopted an incremental approach to clean up the existing code.

## Reworking Update Object Messages

The object update message is one of the more complicated parts of the networking logic.
This handles multiple types of updates, entities, and a ton of fields.
It uses a [bitmask](https://gtker.com/wow_messages/types/update-mask.html) to tell the client which fields it contains.

The previous implementation was pretty hacky and had a few bugs.
Fields were also added incrementally as they were needed, so it was incomplete.
This lives on the edge of things though, so it was a good first candidate for reworking.
To start with, I needed to add some structure to this.

The fields that this message works with can be grouped together into components:

- object
- item
- container
- unit
- player
- gameobject
- dynamicobject
- corpse

Entities are then combinations of these components, like:

- mob = object + unit
- player = object + unit + player

Thinking of things like that, I made structures for each component.
Using a small macro, byte offset and type information could be placed alongside the fields:

```elixir
defmodule ThistleTea.Game.Entity.Data.Component.Object do
  use ThistleTea.Game.Entity.UpdateMask,
    guid: {0x0000, 2, :guid},
    type: {0x0002, 1, :int},
    entry: {0x0003, 1, :int},
    scale_x: {0x0004, 1, :float}
end
```

And the update object message could look like this:

```elixir
defmodule ThistleTea.Game.Network.UpdateObject do
  defstruct [
    :update_type,
    :object_type,
    :movement_block,
    :object,
    :item,
    :container,
    :unit,
    :player,
    :game_object,
    :dynamic_object,
    :corpse
  ]
end
```

This ended up being a nice abstraction and now there's no difference in creating an update object message between mobs, players, or items.
It's also complete, with every field the client accepts set up in these components and ready for use.
With this message now using components, the next step was to make entities use them too.

## Entities

Following that pattern, entities could now look something like this:

```elixir
defmodule ThistleTea.Game.Entity.Data.Mob do
  defstruct object: %Object{},
            unit: %Unit{},
            movement_block: %MovementBlock{},
            internal: %Internal{}
end
```

Game logic was a bit of a paint point, with some bits written specifically for their entities.
Players could attack, but not receive attacks.
Mobs could receive attacks, but not attack.
Things like that.
But by unifying the data model, then the same implementation could work on either.

For example, by pattern matching on the individual components, this function works on both mobs and players:

```elixir
def take_damage(
      %{unit: %Unit{health: health} = unit, movement_block: %MovementBlock{movement_flags: movement_flags} = mb} =
        entity,
      damage
    ) do
  new_health = max(health - damage, 0)
  new_movement_flags = if new_health == 0, do: 0, else: movement_flags
  {:ok, %{entity | unit: %{unit | health: new_health}, movement_block: %{mb | movement_flags: new_movement_flags}}}
end
```

So now game objects, mobs, and players are all made up of the same components.
Logic is now moved outside of the GenServer modules and into re-usable pure components.
I took some inspiration from [Designing Elixir Systems with OTP](https://pragprog.com/titles/jgotp/designing-elixir-systems-with-otp/) for organizating things.
The goal is to have a nice functional core with a boundary layer made up of processes.

As part of this, I did remove some functionality, mostly around combat.
The idea is to re-implement that using the new abstractions, so combat will work consistently between all entities.

## On Mangos

Mangos is the main World of Warcraft private server implementation and Thistle Tea uses its database extensively for things like creatures, items, npc text, etc.
This worked really well, but I let some of the database model structure leak into the core code, which made things a bit annoying to work with.
Instead of using their database model directly, I've moved some of it to a boundary concern using 'loaders'.
These query from the database to get mobs and similar to spawn, but then convert to different structs that are easier to work with.

The idea is that the Mangos database can be used to 'bootstrap' Thistle Tea, but we should prefer working with our own data representations.
Additionally, the state of the system should be entirely separate from the Mangos database.
There's still a lot I need to think about there, but I basically want to make it so it's not as tightly coupled.

## Smarter Initialization

Previously, processes for every mob and game object were created on startup.
This took a few seconds and used about 1.4GB of memory.
Now these processes are managed dynamically based on where players are active.
When a cell is within range of a player, its processes are started.
When a cell is no longer within range, its processes are stopped.
This makes startup much quicker and brings initial memory use down to 92MB.

The implementation is mostly a GenServer that polls player positions every second and chooses to start or stop cells: `ThistleTea.Game.World.System.CellActivator`
This pattern seems to work pretty well and the idea is to build out more systems to cover other bits of functionality.
Think managing game events, battleground queues, battlegrounds in general, dynamic mob spawns, etc.

## Message Abstraction

Packets were still being constructed on the fly, so I wanted to build out a nicer and more standardized networking layer.
The idea was to use structs for messages, so there's less mental overhead and room for error when making them.
The end result is a higher level interface that looks more like this:

```elixir
%Message.SmsgDestroyObject{guid: "1234"}
|> Network.send_packet()
```

Implementing a server message looks like this:

```elixir
defmodule ThistleTea.Game.Network.Message.SmsgDestroyObject do
  use ThistleTea.Game.Network.ServerMessage, :SMSG_DESTROY_OBJECT

  defstruct [:guid]

  @impl ServerMessage
  def to_binary(%__MODULE__{guid: guid}) do
    <<guid::little-size(64)>>
  end
end
```

There's a small macro that helps wire things up and keep things consistent.

Client packets are similar, except they need to implement `from_binary/1` and `handle/2` instead:

```elixir
defmodule ThistleTea.Game.Network.Message.CmsgPing do
  use ThistleTea.Game.Network.ClientMessage, :CMSG_PING

  require Logger

  defstruct [:sequence_id, :latency]

  @impl ClientMessage
  def handle(%__MODULE__{sequence_id: sequence_id, latency: latency}, state) do
    Logger.info("CMSG_PING: #{latency}")

    Network.send_packet(%Message.SmsgPong{sequence_id: sequence_id})
    Map.put(state, :latency, latency)
  end

  @impl ClientMessage
  def from_binary(payload) do
    <<sequence_id::little-size(32), latency::little-size(32)>> = payload

    %__MODULE__{
      sequence_id: sequence_id,
      latency: latency
    }
  end
end
```

This wires things so the packet handling logic can be simplified a lot, something like:

```elixir
%Packet{
  opcode: @cmsg_foo,
  payload: <<>>,
  size: 0
}
|> Packet.to_message()
|> Message.handle(state)
```

All messages previously handled by the application have been migrated to this new consistent interface.
I've had good luck with having LLMs wire up the `from_binary/1` and `to_binary/1` functions from the packet spec, so I'll likely write some helper scripts to better automate that process.

As part of this, I was also able to figure out movement splines, so now a single message can move a mob to multiple points.
This simplifies movement handling by a lot and ends up looking smoother.
Turns out the last point needs to be first and then all the intermediate points follow that as packed offsets.

## Re-implementing Movement

As part of reworking things, I decided I wasn't going to do things from scratch.
I did end up scrapping the existing mob behavior setup, though.
It was a bit overcomplicated and used an unneccessary GenServer just to try to isolate state.

Now with abstractions cleaned up a bit and working movement splines, I reimplemented mob wandering and waypoint pathing.
This lives in ThistleTea.Game.Entity.Logic.Movement as a functional core now, with the GenServer being a thin wrapper around it:

```elixir
@impl GenServer
def handle_cast({:move_to, x, y, z}, state) do
  state = Movement.move_to(state, {x, y, z})
  {:noreply, state}
end

@impl GenServer
def handle_info(:wander, state) do
  state = Movement.wander(state)
  delay = Movement.wander_delay(state)
  Process.send_after(self(), :wander, delay)
  {:noreply, state}
rescue
  _ -> {:noreply, state}
end
```

Much easier to reason about.

I didn't add back the mob combat chasing behavior, since that's something to revisit when reworking combat to use the new abstractions.

## Magic Numbers

There were a lot of magic numbers littered across the various bits of the application.
Module attributes were used extensively for opcodes and other important bits, like `@smsg_foobar 0x123`.
It worked, but they had to be duplicated across all the modules that wanted to use them.

Instead of manually adding a bunch of opcode module attributes at the top of files, I created a helper macro to define them:

```elixir
use ThistleTea.Game.Network.Opcodes, [:SMSG_UPDATE_OBJECT, :SMSG_COMPRESSED_UPDATE_OBJECT]
```

This makes `@smsg_update_object` and `@smsg_compressed_update_object` available, but I don't need to remember or care about the actual opcode values.
There's also been some tweaks to use atoms in more places where it makes sense.

## Pattern matching with structs

Previously there were no structs in the project, everything was just raw maps.
Most things are now structs, which is nice.
When pattern matching on structs in function heads, the Elixir compiler can do some type checking that helps a lot with refactoring.

```elixir
def set_position(
      %{
        object: %Object{guid: guid},
        movement_block: %MovementBlock{position: {x, y, z, _o}},
        internal: %Internal{map: map}
      },
      table
    ) do
  SpatialHash.update(table, guid, self(), map, x, y, z)
end
```

I've been trying to do this a lot more frequently and it's a pattern I've really been liking.

## Cleaning up the Network Layer

Feel like I spoiled this a bit in a previous section, but I made some efforts to separate out a network layer.
Previously everything was intermingled, but I wanted to be able to focus on game logic at a higher level in places.

I originally did a [proof of concept](https://github.com/pikdum/thistle_tea/pull/8) for what handling packets could look like if building from scratch and came up with some decent ideas.
That implementation focused a lot on testability, with side effects represented as data and deferred until later.
I didn't use all of those ideas, but simplified it to something that was straightforward to retrofit.

The idea was to make the use of `ThousandIsland` for handling socket connections a boundary concern and come up with a cleaner abstraction than working with raw binary payloads.

The flow looks like:

- accumulate binary packets
- turn that into `%Packet{opcode: opcode, payload: payload, size: size}` structs
- turn those into the various message structs
- handle the messages

Previously, it was just:

- accumulate binary packets
- handle those

So the packet handler functions had to parse out what they cared about from the binary.
Now it's just all available in a struct to work with by default.
As part of this, I was able to simplify some things like there's no need for a second GenServer just to handle packet encryption anymore.
All relevant data for a connection is now on a nice `ThistleTea.Game.Network.Connection` struct.

## The World

ThistleTea.Game.World is a new namespace to help organize things a bit nicer.
Things like spatial hashing, pathfinding, loaders, and systems were moved in here.
Functions to query nearby players, broadcast packets, and start/stop entities are part of the public interface.

The loaders handle loading data from Mangos into Thistle Tea, transforming things into our representations.
Systems are another new abstraction, starting with cell activator and game event systems.
The idea behind systems is to make it more standardized to build things using higher level abstractions in a relatively isolated way.

There are currently systems for activating cells based on nearby players and changing the current game events, but future ones could handle:

- battleground queues
- battleground objectives
- auction house
- mail
- dynamic mob spawns
- gather spots

## Game Event System

TODO: add video showing changing

Mentioned above, there's now a system to change the active game events.
These are things like the current holidays or faire location.
We were previously spawning everything regardless, leading to things like overlapping halloween and christmas decorations.

It's a GenServer that keeps track of the current events and notifies subscribers of changed events:

```elixir
@impl GenServer
def handle_call(:get_events, _from, state) do
  {:reply, MapSet.to_list(state.events), state}
end

@impl GenServer
def handle_call({:set_events, new_events}, _from, %{events: old_events} = state) do
  notify(new_events, old_events)
  {:reply, :ok, %{state | events: new_events}}
end
```

If associated with a game event, mobs and game objects subscribe to a channel using Phoenix PubSub.
They can then decide to do things like change models or despawn themselves.
Starting an event also sends a message to the cell manager, which will spawn in things that weren't previously active.

The result is that events can now be changed on the fly and it'll handle adding and removing things properly.
This doesn't yet handle model changes, where a mobs is active all the time but should change appearance during events.
It also needs to be wired up with a scheduler, so that holiday events are started/stopped automatically.

This is probably the start of using PubSub for more things, too.

## Network

Networking has been moved to ThistleTea.Game.Network, where there's a nicer public interface for sending packets to processes.

## Organization

The idea is to keep relevant stuff grouped together to allow for some higher level abstractions.
Like a game system shouldn't be manually putting together binary packets and should instead be using message structs.
The top level bits like World, Network, etc. should sort of be public interfaces, with more deeply nested stuff being the internals.
This isn't enforced, but it's a pattern I'd like to continue exploring.

This is what things look like right now, under ThistleTea.Game:

```
game
├── entity
│   ├── data
│   │   ├── component
│   │   │   ├── container.ex
│   │   │   ├── corpse.ex
│   │   │   ├── dynamic_object.ex
│   │   │   ├── game_object.ex
│   │   │   ├── internal
│   │   │   │   ├── waypoint.ex
│   │   │   │   └── waypoint_route.ex
│   │   │   ├── internal.ex
│   │   │   ├── item.ex
│   │   │   ├── movement_block.ex
│   │   │   ├── object.ex
│   │   │   ├── player.ex
│   │   │   └── unit.ex
│   │   ├── game_object.ex
│   │   └── mob.ex
│   ├── logic
│   │   ├── core.ex
│   │   └── movement.ex
│   ├── server
│   │   ├── game_object.ex
│   │   └── mob.ex
│   └── update_mask.ex
├── entity.ex
├── math.ex
├── network
│   ├── binary_utils.ex
│   ├── connection
│   │   └── crypto.ex
│   ├── connection.ex
│   ├── message
│   │   ├── cmsg_*.ex
│   │   ├── msg_move.ex
│   │   ├── smsg_*.ex
│   ├── opcodes.ex
│   ├── packet.ex
│   ├── protocols.ex
│   ├── send.ex
│   ├── server.ex
│   └── update_object.ex
├── network.ex
├── world
│   ├── loader
│   │   ├── game_object.ex
│   │   └── mob.ex
│   ├── pathfinding.ex
│   ├── spatial_hash.ex
│   └── system
│       ├── cell_activator.ex
│       └── game_event.ex
└── world.ex
```

I feel like most things will be under message, since there will be one module per message with serialization and handling functions.
But this is the current pattern and hopefully it'll make more sense where things should be.
Definitely isn't the final organization though, the plan is to iterate on this and figure out what works and what doesn't.
There's also things missing, like I've mentioned a few times I need to figure out what Thistle Tea's 'store' or database looks like.
That'll be needed for entities like items that we don't need/want a process for each, internal lookup tables, etc.

But the overall goal of this refactoring work has been to separate out a functional core from the boundary layer.
Then the boundary layer can be changed as needs change.
Like maybe one process per entity actually doesn't scale, it'd be straightforward to swap that out to group by cell, zone, map, etc. instead.
Right now I'm using pids in some function calls where I should be using ids, but once that's changed the actual boundary layers should matter less and less to the game logic.

## Gains

Rewriting the object update bits fixed some issues, likely due to previously having improper hex offsets for some fields.
Like there was a weird issue where every time a player changed equipment the hover cursor would change, now resolved.

Since networking has been standardized with some higher level abstractions, it's been easier to build on top of it.
Object update packets support batching, but previously we were just doing things one at a time.
But batching was pretty straightforward to add now, so now if multiple object updates are queued they get batched into a single one automatically before sending to the client:

```elixir
def accumulate_updates(size, body) do
  receive do
    {:"$gen_cast",
     {:send_packet,
      %Packet{opcode: @smsg_update_object, payload: <<next_size::little-size(32), 0, next_body::binary>>}}}
    when size + next_size <= 100 ->
      accumulate_updates(size + next_size, body <> next_body)
  after
    0 -> %Packet{opcode: @smsg_update_object, payload: <<size::little-size(32), 0, body::binary>>}
  end
end

@impl GenServer
def handle_cast(
      {:send_packet, %Packet{opcode: @smsg_update_object, payload: <<size::little-size(32), 0, body::binary>>}},
      {socket, state}
    ) do
  packet = accumulate_updates(size, body)
  state = Network.Send.send_packet(packet, {socket, state})
  {:noreply, {socket, state}, socket.read_timeout}
end
```

It could also be possible to use this same pattern to optimize movement later, but that's a bit trickier.

## Community Contributions

We received some awesome community contributions this year!

- better teleport (no longer requires logout) - poffdeluxe
- emote handler for /dance and others - poffdeluxe
- set rested state - jmmk
- set player `unit_faction_template` based on race - adamvietro

## Things I Didn't Do

I looked into code generation from the wow_messages project to make a library that'd automatically be able to serialize/deserialize packets.
Didn't end up getting anything I was happy with, though.
I did find that LLMs are pretty good at working with the new Message format to implement that logic from the specs, though, so that's likely the way going forwards.
Still need to build some tooling to automate this further.

I also looked into fully rewriting this from scratch and actually did for bits of the networking layer.
But there's so much already working and I decided to refactor instead.
I think this is the right way, I want to build a codebase that can evolve and change nicely rather than one that needs to be frequently scrapped.

Using Entity Component System was another thing I tried a lot.
Couldn't really get anything I was happy with, though.
A lot of my proof-of-concepts relied on polling for systems, where sticking with the actor model makes things more reactive instead.
It also didn't feel like the best way to try and leverage OTP, so scrapped that idea.

I did steal some ideas from ECS, though, like building up our entities out of components.
Then functions can be written more generically to work on multiple types of entities without needing different implementations.
This helped a lot already with the object updates, but I'm hoping it helps a lot too when getting to reimplementing combat.

## Up Next

This are in a much better state, but there's still tons to do.

Some rough thoughts:

- clean up handlers, since these were mostly ported as-is
- stop querying from Mangos, move that all to the edge
- rip out more logic from the connection handler to make that leaner
- tests, especially around world systems
- 'store' abstraction, something for entities like items
- rip out :mnesia, figure something else out
- should players be a separate process from the connection handler?
- re-implement combat
- add more gameplay systems
- items and inventory management
- quests

## Contributing

Interested in the project?
Want to chat architecture?
Want to try implementing some features?

Hop in Thistle Tea's [Discord channel](https://discord.gg/dSYsRXHDhb).

These changes (hopefully) made the code much easier to work with and provide a bit of patterns for extending the system.
