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

![](./melli.avif)

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

## The Plan

I've been doing some reading and [Designing Elixir Systems with OTP](https://pragprog.com/titles/jgotp/designing-elixir-systems-with-otp/) was a very helpful book.
That gave me the idea to more clearly separate the functional core and boundary layer of the application.

The functional core will be concerned with data and logic; the boundary layer will handle the process orchestration.
This means building out structs, separating logic out of GenServers, and building an interface that doesn't care what the processes are doing under the hood.
The functional core can be stable and well tested with unit tests, but the boundary layer can be tweaked more easily as needs change.

Like maybe if one process per entity doesn't actually scale, then the boundary layer can be tweaked to group entities together by cell or map or whatever else instead.

The goal is to make it easier to reason about and develop Thistle Tea going forward, giving it a much more solid foundation.

## Building Entities Out Of Components

First, some definitions.\
An _entity_ is something like a player, mob, or item that uses the **SMSG_UPDATE_OBJECT** message.\
An entity is made up of a combination of _components_, individual groups of related fields.

Components:

- Object
- Item
- Container
- Unit
- Player
- Game Object
- Dynamic Object
- Corpse

Entities:

- Mob = Object + Unit
- Player = Object + Unit + Player

For example, this is what the mob struct now looks like:

```elixir
defmodule ThistleTea.Game.Entity.Data.Mob do
  defstruct object: %Object{},
            unit: %Unit{},
            movement_block: %MovementBlock{},
            internal: %Internal{}
end
```

A pain point with writing combat was bits written specifically for their entities.
Player could attack, but not receive attacks.
Mobs could receive attacks, but not attack.
Players could cast spells, but mobs couldn't.
Things like that.

By unifying the data model, now implementations can be more generic and work across all entities:

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
Logic is moved outside of the large GenServer modules and into reusable pure components.
Some features were lost as part of this, like combat, to be reimplemented later using cleaner abstractions.

## Message Abstraction

When sending packets to the client, most were still being constructed on the fly and I wanted to clean that up.
The idea is to use structs for messages, so there's less mental overhead and room for error when making them.
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

Client packets are similar, except they need to implement `from_binary/1` and `handle/2` instead:

```elixir
defmodule ThistleTea.Game.Network.Message.CmsgPing do
  use ThistleTea.Game.Network.ClientMessage, :CMSG_PING

  defstruct [:sequence_id, :latency]

  @impl ClientMessage
  def handle(%__MODULE__{sequence_id: sequence_id, latency: latency}, state) do
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

This allows simplifying the connection handler a lot, to roughly this:

```elixir
%Packet{
  opcode: @cmsg_foo,
  payload: <<>>,
  size: 0
}
|> Packet.to_message()
|> Message.handle(state)
```

Where previously it was passing raw binary around.

All messages previously handled by the application have been migrated to this new consistent interface.

As part of this, I was also able to figure out movement splines, so now a single message can move a mob to multiple points.
This simplifies movement handling by a lot and ends up looking smoother.
Turns out the last point needs to be first and then all the intermediate points follow that as packed offsets.

## Reworking Update Object Messages

The object update message is one of the more complicated parts of the networking logic.
This handles multiple types of updates, entities, and a ton of fields.
It uses a [bitmask](https://gtker.com/wow_messages/types/update-mask.html) to tell the client which fields it contains.

The previous implementation was pretty hacky and had a few bugs.
Fields were also added incrementally as they were needed, so it was incomplete.

I rewrote a lot of this implementation, following the new data models.
Since entities are now made up of components and this message deals with components, byte offset and other type information are placed directly alongside the component fields:

```elixir
defmodule ThistleTea.Game.Entity.Data.Component.Object do
  use ThistleTea.Game.Entity.UpdateMask,
    guid: {0x0000, 2, :guid},
    type: {0x0002, 1, :int},
    entry: {0x0003, 1, :int},
    scale_x: {0x0004, 1, :float}
end
```

This allows the update object message struct to look like this:

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

## The World

**ThistleTea.Game.World** is a new namespace to help organize things a bit nicer.
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

### Game Event System

{{<video src="./faire.mp4">}}

Mentioned above, there's now a system to change the active game events.
These are things like the current holidays or faire location.
Previously everything was being started regardless, leading to things like overlapping Hallow's End and Winter's Veil decorations.

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
Starting an event also sends a message to the cell activator, which will spawn in things that weren't previously active.

The result is that events can now be changed on the fly and it'll handle adding and removing things properly.
This doesn't yet handle model changes, where a mob is active all the time but should change appearance during events.
It also needs to be wired up with a scheduler, so that holiday events are started/stopped automatically.

This is probably the start of using PubSub for more things, too.

### Cell Activator System

Previously, processes for every mob and game object were created on startup.
This took a few seconds and used about 1.4GB of memory.
Now these processes are managed dynamically based on where players are active.
When a cell is within range of a player, its processes are started.
When a cell is no longer within range, its processes are stopped.
This makes startup much quicker and brings initial memory use down to 92MB.

### On Mangos

[Mangos](https://www.getmangos.eu/) is the main World of Warcraft private server implementation and Thistle Tea uses its database extensively for things like creatures, items, dialogue text, etc.
This works really well, but I let some of the database model structure leak into the core code, which made things a bit annoying to work with.
Instead of using their database model directly, I've moved some of it to a boundary concern using 'loaders'.
These query from the database to get mobs and other entities to spawn, but then convert to different structs that are easier to work with.

The idea is that the Mangos database can be used to 'bootstrap' Thistle Tea, but we should prefer working with our own data representations.
Additionally, the state of the system should be entirely separate from the Mangos database.
There's still a lot I need to think about there, but I basically want to make it so it's not as tightly coupled.

## Re-implementing Movement

As part of this effort, I wanted to keep as much of the existing code functional as feasible.
I did end up scrapping the existing mob behavior setup, though.
It was a bit overcomplicated and used an unnecessary GenServer just to try to isolate state.

Now with abstractions cleaned up a bit and working movement splines, I reimplemented mob wandering and waypoint pathing.
This lives in **ThistleTea.Game.Entity.Logic.Movement** as a functional core now, with the GenServer being a thin wrapper around it:

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

Much easier to reason about and the underlying logic will work for any entities with movement.

I didn't add back the mob combat chasing behavior, since that's something to revisit when reworking combat to use the new abstractions.

## Magic Numbers

There were a lot of magic numbers littered across the various bits of the application.
Module attributes were used extensively for opcodes and other important bits, like `@smsg_foobar 0x123`.
It worked, but they had to be duplicated across all the modules that wanted to use them.

Instead of manually adding a bunch of opcode module attributes at the top of files, now there's a helper macro to define them:

```elixir
use ThistleTea.Game.Network.Opcodes, [:SMSG_UPDATE_OBJECT, :SMSG_COMPRESSED_UPDATE_OBJECT]
```

This makes `@smsg_update_object` and `@smsg_compressed_update_object` available, but I don't need to remember or care about the actual opcode values.
There's also been some tweaks to use atoms in more places where it makes sense.

## Gains

Rewriting the object update bits fixed some issues, due to previously having incorrect hex offsets for some fields.
For example, there was a weird issue where the hover cursor changed every time a player changed equipment.
This now no longer happens.

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
- set player **unit_faction_template** based on race - adamvietro

## Things I Didn't Do

I feel like I spent more time thinking about doing things than actually doing things this year.

### Code Generation of Messages

There's an awesome project called [wow_messages](https://github.com/gtker/wow_messages) that has message definitions for every packet.
These can be used to automatically generate libraries, but I decided not to go down that path.
Looked into it for a while, but I couldn't get things working nicely.
Instead, I went with the message abstraction described above.
I found that LLMs are pretty decent at reading these message definitions and generating what we need, so I'm planning on building some automation around that.

### Rewriting From Scratch

I also looked into fully rewriting this from scratch and actually did for bits of the networking layer.
But there's so much already working and I decided to refactor instead.
I think this is the right way, I want to build a codebase that can evolve and change nicely rather than one that needs to be frequently scrapped.

### Entity Component System

ECS makes a lot of sense and I put together some basic proof of concepts using it, but couldn't get this in a spot I liked either.
A lot of my attempts relied on polling for systems, where sticking with the actor model makes things more reactive instead.
The centralized state for components also made things scale poorly once you got to large numbers of components.
It also didn't feel like the best way to try and leverage OTP, so scrapped that idea.

I did steal some ideas from ECS, though, like building up entities out of components.
Then functions can be written more generically to work on multiple types of entities without needing different implementations.
This helped a lot already with the object updates, but I'm hoping it helps a lot too when getting to reimplementing combat.

## Up Next

Things are in a much better state, but there's still tons to do.

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

Interested in the project?\
Want to chat architecture?\
Want to try implementing some features?

Hop in Thistle Tea's [Discord channel](https://discord.gg/dSYsRXHDhb).

These changes (hopefully) make the code much easier to work with and provide consistent patterns for extending the system.
