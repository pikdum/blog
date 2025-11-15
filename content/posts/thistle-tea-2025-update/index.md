+++
title = 'Building a World of Warcraft server in Elixir: 2025 Update'
date = 2025-11-14
draft = false
tags = ['elixir', 'World of Warcraft', 'programming']
+++

This is a follow-up to these posts:

- [Building a World of Warcraft server in Elixir](/posts/thistle-tea/)
- [Building a World of Warcraft server in Elixir: 2024 Update](/posts/thistle-tea-2024-update/)

[Thistle Tea](https://github.com/pikdum/thistle_tea) is a World of Warcraft private server project that I've been working on for a while now. This is a quick update to highlight what's been going on in 2025.

## New Year's Blues

- at start of year, things bad
- code organized around genservers
- huge genservers
- genservers where i didn't need genservers
- packets manually constructed ad-hoc
- long startup + high initial memory use from starting all mobs
- bad organization
- constructing update object messages was a pain
- ended up doing an incremental approach of the existing code

## Reworking Update Object Messages

- this is complicated
- https://gtker.com/wow_messages/types/update-mask.html
- entities: player, mob, item, game object, etc.
- entities are made up of some combination of these components:
  - object
  - item
  - container
  - unit
  - player
  - gameobject
  - dynamicobject
  - corpse
- the plan:
  - make struct for each component
    - use a DSL to specify include offset, size, type directly alongside the fields
  - make update_object struct that contains components
  - give it a nice interface for turning that into an https://gtker.com/wow_messages/docs/smsg_update_object.html packet
  - clean up the logic
- benefits:
  - everything's there now, no need to add more fields as we add functionality
  - clean abstraction
  - all entities can use this interface the same
- next steps:
  - after this was done, next obvious step was to make entities use these components directly, too

## Entities

- build out entity abstraction
- end goal is that players, mobs, etc. would be 'entities' made up of 'components'
- that way we could write code that more or less uses the components directly, so same code could work for player + mob + game object, etc.
- this would help solve the pain point that a lot of things are only half implemented right now - like players can autoattack, but mobs cannot, since they're different enough, same with casting spells, etc.
- idea was to make a nicer foundation so things only need to be implemented once
- and also split out logic from genservers - taking inspiration from https://pragprog.com/titles/jgotp/designing-elixir-systems-with-otp/ - logic should be nice functional code working on structs, that way unit tests are easier to write and it's easier to reason about, genservers are thin wrappers for that + state
- the plan:
  - make new GameObject struct made up of object + gameobject + movement block + internal components
  - move implementation over to that
  - new genserver
  - do the same for mobs
- lost some functionality as part of this - mobs no longer do combat-related things, combat will be re-implemented using this new entity approach so it will automatically work for players + mobs

## On Mangos

- before, we were using datastructures shaped like mangos's tables for easy interop
- but plan is to change that - mangos should only be used to 'bootstrap' the system, we don't need/want to use their datastructures themselves and should instead transform into a more idiomatic way for us to consume
- so instead of querying the mangos sqlite database on current state of things, we query the running thistle tea system directly
- still kinda wip, only mobs + game servers moved to this new model
- still figuring out a 'store' model

## Smarter Initialization

- previously, started processes for every mob + game object on startup - like 100k
- now, poll for which cells players are active in and then dynamically spawns/despawns cells
- still needs a bit of work, but went from 1.4GB memory use on startup down to 192MB
- (some edge cases with mob waypoint navigation probably, needs a slightly cleaner implementation)

## Message Abstraction

- rather than ad-hoc binary construction for packets, idea is to give each client and server message a struct
- so don't need to care about more than just the fields
- uses protocols + some macros + behaviours to wire things up
- ClientMessage - implements handle/2 and from_binary/1
  - handle takes in message struct + player state and 'handles' the received message
    - this replaces and simplifies the old packet handler
  - from_binary turns a binary packet into the struct
- ServerMessage - implements to_binary/1 to turn struct into binary packet
- all messages sent from the app now have these implemented and can be worked with nicely from structs
- LLMs were useful to migrate over handlers + the parsing logic, will plan on wiring up a script for easily scaffolding out more messages
- as part of this, was able to better figure out movement to properly use splines, so monster movement can be simpler + smoother

## Re-implementing Movement

- didn't want to scrap things, but did decide to scrap the mob behavior bit needed to re-implement wandering + pathfinding movements
- thanks to the better abstractions, this is much cleaner
- most logic is in the entity core, and the mob genserver is a really thin wrapper around that + some timers
- works pretty well, still a bit glitchy but think those might need to be solved in namigator
- still work to be done with re-implementing the 'attacking' chase behavior mobs do in combat

## Magic Numbers

- previously used module attributes for this, like `@smsg_foobar 0x123`
- scattered across + duplicated in every module that wanted them
- made macro to help clean this up:

```elixir
use ThistleTea.Game.Network.Opcodes, [:SMSG_UPDATE_OBJECT, :SMSG_COMPRESSED_UPDATE_OBJECT]
```

- this makes `@smsg_update_object` and `@smsg_compressed_update_object` available, and don't need to care/worry about the actual opcode numbers
- other tweaks to use atoms in more places too

## pattern matching is nice

- basically get the benefits of types
- show some examples of the new style i've been using

## clean up the network layer

- or more accurately, build it, since everything is intermingled
- bit cheating, actually did a poc of what i'd build this like if building from scratch: https://github.com/pikdum/thistle_tea/pull/86
- way before a lot of this
- brought the ideas from that into the main codebase
- accumulate binary until we can parse packets out of it, then turn packets into messages, then handle messages, basically
- packet |> Packet.to_message() |> Message.handle()
- Connections struct + functions to store crypto + other state, remove need for a separate crypto genserver
- idea is to keep the 'connection handler' more focused on the network stuff, and just delegate to Message.handle()
- still work to be done to clean up player, right now state is still part of this, could be separated into other genservers to look more like mob/game objects

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
├── network
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
│   └── util.ex
├── world
│   ├── cell_activator.ex
│   ├── mangos
│   │   ├── game_object_supervisor.ex
│   │   └── mob_supervisor.ex
│   ├── pathfinding.ex
│   └── spatial_hash.ex
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
