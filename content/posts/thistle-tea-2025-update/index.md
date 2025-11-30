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

- before, we were using datastructures shaped like mangos's tables for easy interop
- but plan is to change that - mangos should only be used to 'bootstrap' the system, we don't need/want to use their datastructures themselves and should instead transform into a more idiomatic way for us to consume
- so instead of querying the mangos sqlite database on current state of things, we query the running thistle tea system directly
- still kinda wip, only mobs + game servers moved to this new model
- still figuring out a 'store' model

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

## world

- idea is to have more things live in world
- spatial hashing, pathfinding, cell activator and other systems
- example new system could be the season event system, switch up mobs/game objects/etc. based on active event
- right now everything's active
- querying nearby players, broadcasting packets to nearby players, etc. are in the public World interface

## network

- public interface for this has sending packets to pids

## organization!

idea is to kinda keep relevant stuff grouped together
and use the top level bits like World, Network, etc. as public interfaces to the internals
still a bit more to be done here, like getting rid of util, but it's in a much better spot

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
│   │   ├── cmsg_attackstop.ex
│   │   ├── cmsg_attackswing.ex
│   │   ├── cmsg_auth_session.ex
│   │   ├── cmsg_cancel_cast.ex
│   │   ├── cmsg_cast_spell.ex
│   │   ├── cmsg_char_create.ex
│   │   ├── cmsg_char_enum.ex
│   │   ├── cmsg_creature_query.ex
│   │   ├── cmsg_gameobject_query.ex
│   │   ├── cmsg_gossip_hello.ex
│   │   ├── cmsg_gossip_select_option.ex
│   │   ├── cmsg_item_name_query.ex
│   │   ├── cmsg_item_query_single.ex
│   │   ├── cmsg_join_channel.ex
│   │   ├── cmsg_leave_channel.ex
│   │   ├── cmsg_logout_cancel.ex
│   │   ├── cmsg_logout_request.ex
│   │   ├── cmsg_messagechat.ex
│   │   ├── cmsg_move_worldport_ack.ex
│   │   ├── cmsg_name_query.ex
│   │   ├── cmsg_npc_text_query.ex
│   │   ├── cmsg_ping.ex
│   │   ├── cmsg_player_login.ex
│   │   ├── cmsg_set_selection.ex
│   │   ├── cmsg_setsheathed.ex
│   │   ├── cmsg_standstatechange.ex
│   │   ├── cmsg_text_emote.ex
│   │   ├── cmsg_who.ex
│   │   ├── msg_move.ex
│   │   ├── smsg_account_data_times.ex
│   │   ├── smsg_attackstart.ex
│   │   ├── smsg_attackstop.ex
│   │   ├── smsg_auth_challenge.ex
│   │   ├── smsg_auth_response.ex
│   │   ├── smsg_bindpointupdate.ex
│   │   ├── smsg_cast_result.ex
│   │   ├── smsg_channel_notify.ex
│   │   ├── smsg_char_create.ex
│   │   ├── smsg_char_enum.ex
│   │   ├── smsg_chat_player_not_found.ex
│   │   ├── smsg_creature_query_response.ex
│   │   ├── smsg_destroy_object.ex
│   │   ├── smsg_emote.ex
│   │   ├── smsg_gameobject_query_response.ex
│   │   ├── smsg_gossip_message.ex
│   │   ├── smsg_initial_spells.ex
│   │   ├── smsg_item_name_query_response.ex
│   │   ├── smsg_item_query_single_response.ex
│   │   ├── smsg_login_settimespeed.ex
│   │   ├── smsg_login_verify_world.ex
│   │   ├── smsg_logout_cancel_ack.ex
│   │   ├── smsg_logout_complete.ex
│   │   ├── smsg_logout_response.ex
│   │   ├── smsg_messagechat.ex
│   │   ├── smsg_monster_move.ex
│   │   ├── smsg_name_query_response.ex
│   │   ├── smsg_new_world.ex
│   │   ├── smsg_npc_text_update.ex
│   │   ├── smsg_pong.ex
│   │   ├── smsg_set_rest_start.ex
│   │   ├── smsg_spell_failed_other.ex
│   │   ├── smsg_spell_failure.ex
│   │   ├── smsg_spell_go.ex
│   │   ├── smsg_spell_start.ex
│   │   ├── smsg_text_emote.ex
│   │   ├── smsg_transfer_pending.ex
│   │   ├── smsg_trigger_cinematic.ex
│   │   ├── smsg_tutorial_flags.ex
│   │   └── smsg_who.ex
│   ├── opcodes.ex
│   ├── packet.ex
│   ├── protocols.ex
│   ├── send.ex
│   ├── server.ex
│   └── update_object.ex
├── network.ex
├── utils
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

## misc

- this fixed some issues, due to previously having improper hex offsets for the update mask bits
- now batches update object packets, since they can be easily pattern matched on and the networking piece is cleaned up
  - currently up to 100 updates in a single packet, pretty cool

## community contributions

- better teleport (no longer requires logout) - poffdeluxe
- emote handler for /dance and others - poffdeluxe
- set rested state - jmmk
- set player `unit_faction_template` based on race - adamvietro

## things i didn't do

- code generation of wow_messages
- entity component system
- rewrite from scratch

## up next

things are in a much better state, but the plan is mostly to continue cleaning things up rather than new feature development
some things:

- get rid of util.ex
- clean up handlers more, they were mostly ported as-is, but should stop querying directly from mangos and instead somehow query state from the running system
  so need that
- rip out more from the game server processes (connection handler), some things could/should be separate processes, or at least have the logic elsewhere
- unit tests :^)
- need a 'store' abstraction - something for entities that aren't processes, like items
  but also maybe for storing anything else, maybe be a thistle tea in memory storage layer
  that we can then wire up persistence too
  not a fan of the current :mnesia based one
  needs more thought
- should players be a separate process from the connection handler? would handlers then just be a thin interface that sends messages to the player process?
- CellActivator, GameObjectSupervisor, MobSupervisor - these could be cleaned up, probably simplify to a dynamic supervisor and then despawn individual entities based on current position
- Combat - needs to be re-implemented in a way that can be consumed from both players + mobs
- more 'systems' - good first one is seasonal event system, others eventually could be things like battleground queues, etc.
