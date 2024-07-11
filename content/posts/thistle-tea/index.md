+++
title = 'Building a WoW server in Elixir'
date = 2024-07-10
draft = false
tags = ['elixir', 'World of Warcraft', 'programming']
+++

[Thistle Tea](https://github.com/pikdum/thistle_tea) is my new World of Warcraft private server project.
You can log in, create a character, run around, and cast spells to kill mobs, with everything synchronized between players as expected for an MMO.
It was floating around in my head to build this for a while, since I have an incurable nostalgia for early WoW.
I first mentioned this on May 13th and didn't expect to get any further than login, character creation, and spawning into the map.
Here's a recount of the first month of development.

## Day 0

Before coding, I did some research and came up a plan.

* Code this in Elixir, using the actor model.
* MaNGOS has done the hard work on collecting all the data, so use it.
* Use Thousand Island as the socket server.
* Treat this project as an excuse to learn more Elixir.
* Reference existing projects and documentation rather than working from scratch.
* Speed along the happy path rather than try and handle every error.

## Day 1 - June 2nd

There are two parts that needed to be built: the authentication server and the game server.
Up first was authentication, since you can't do anything without logging in.

My plan was to build a MITM proxy between the client and a MaNGOS server to log all packets.
It wasn't as useful as expected, but it did help me internalize how the requests and responses worked.

The auth flow can be simplified as:

* Client sends a **CMD_AUTH_LOGON_CHALLENGE** packet
* Server sends back some data for the client to use in crypto calculations
* Client sends a **CMD_AUTH_LOGON_PROOF** packet with the client proof
* If the client proof matches what's expected, server sends over the server_proof
* Client is now authenticated

Packets have a header section that contains the opcode, or message type, and the size, followed by the payload.

It uses SRP6, which I hadn't heard of before this.
Seems like the idea is to avoid transmitting an unencrypted password, and instead both the client and server independently calculate something that only matches if they both know the correct password.

So basically, what I needed to do was:

* Listen for data over the socket
* Once data received, parse what message it is out of the header section
* Handle each one differently
* Send back the appropriate packet

This whole part is well documented, but I still ran into some issues with the cryptography.
Luckily, I found a blog post and an accompanying Elixir implementation, so I was able to substitute my broken cryptography with working cryptography.
Without that, I would've been stuck at this part for a very long time (maybe forever).
Still wasn't able to get login working on day 1, but I was close.

Links:

* https://wowdev.wiki/Login
* http://srp.stanford.edu/design.html
* https://shadowburn-project.org/2018/10/17/logging-in-with-vanilla.html
* https://gitlab.com/shadowburn/shadowburn

## Day 2 - June 3rd

I spent some time cleaning up the code and realised I had a logic error where I reversed some crypto bytes that weren't supposed to be.
Fixing that made auth work, finally getting a success with hardcoded credentials.

![](20240603_21h10m37s_grim.avif)

Next up was getting the realm list to work, by handling **CMD_REALM_LIST** and returning which game server to connect to.

![](20240603_21h26m33s_grim.avif)

This got me out of the tedious auth bits and I could get to building the game server.

Links:

* https://wowdev.wiki/CMD_REALM_LIST_Client
* https://wowdev.wiki/CMD_REALM_LIST_Server

## Day 3 - June 4th

The goal for today was to get spawned into the world.
But first more tedious auth bits.

The game server auth flow can be simplified as:

* When client first connects, server sends **SMSG_AUTH_CHALLENGE**, with a random seed
* Client sends back **CMSG_AUTH_SESSION**, with another client proof
* If client proof matches server proof, server sends a successful **SMSG_AUTH_RESPONSE**

This basically negotaties how to encrypt/decrypt future packet headers.
Luckily Shadowburn also had crypto code for this, so I was able to use it here.

After that, it's something like:

* Client sends message to server
* Server decrypts header, which contains type of message and message size
* Server handles message and sends back 0 or more messages to client

First is handling **CMSG_CHAR_CREATE** and **CMSG_CHAR_ENUM**:

![](20240604_16h06m17s_grim.avif)

Then I got side-tracked for a bit trying to get equipment to show up, since I had all the equipment display bytes just hardcoded to 0 before.

![](20240604_19h02m30s_grim.avif)

After that was handling **CMSG_PLAYER_LOGIN**.
I found an example minimal **SMSG_UPDATE_OBJECT** spawn packet, which was supposed to spawn me in Northshire Abbey.

That's probably the most important packet, since it does everything from:

* Spawning things into the world, like players, mobs, objects, etc.
* Updating their values, like health, position, etc.

It has a lot of different forms, can update multiple objects in a single packet, and has a compressed variant.

![](20240604_20h08m46s_grim.avif)

Whoops, had the coordinates a bit off.
After fixing that, I was in the human starting area as expected.
No player model yet, though.

![](20240604_20h18m34s_grim.avif)

Next up was adding more to that spawn packet to use the player race and proper starting area.
The starting areas were grabbed from a MaNGOS database that I converted over to SQLite.

![](20240604_20h59m35s_grim.avif)

Last for the night was to get logout working.

The implementation was something like:

* Queue a **:login_complete** message to send **SMSG_LOGOUT_COMPLETE** to the client
* Store a reference to that timer in state
* If received a cancel, cancel the timer

This was the first piece that really took advantage of Elixir's message passing.

The white chat box was weird, but it was nice being able to log in.

Links:

* https://wowdev.wiki/Opcodes
* https://wowdev.wiki/SMSG_UPDATE_OBJECT
* https://gtker.com/wow_messages/docs/smsg_update_object.html
* https://gtker.com/wow_messages/ir/implementing_world.html
* https://github.com/vdechef/mysql2sqlite

## Day 4 - June 5th

First up was reorganizing the code, since my game.ex GenServer was getting too large.

My strategy for that was:

* Split out related messages into separate files
  * auth.ex, character.ex, ping.ex, etc.
  * wrapped in the `__using__` macro
* Include those back into game.ex with `use`

It worked, but it messed with line numbers in error messages and made things harder to debug.

After that, I wanted to generate that spawn packet properly rather than hardcoding.
The largest piece of this was figuring out the update mask for the update fields.

Simplified, there are a ton of fields for objects, units, players, etc.
Before the fields in an update packet, there's a bit mask with bits set at offsets that correspond to the fields being sent.
Without that, the client wouldn't know what to do with the values.
Luckily it's all well documented, but it still took a while to implement.

Links:

* https://gtker.com/wow_messages/types/update-mask.html

## Day 5 - June 6th

Referencing MaNGOS, I added some more messages that the server sends to the client after a **CMSG_PLAYER_LOGIN**.
One of these, **SMSG_ACCOUNT_DATA_TIMES**, fixed the white chat box and keybinds being reset.
I also added **SMSG_COMPRESSED_UPDATE_OBJECT**, which compresses the update packet with `:zlib.compress/1`.

Movement would come up soon, so I started adding the handlers for those packets.

## Day 6 - June 7th

In the update packet, I still had the object guid hardcoded.
This is because it wants a packed guid and I needed to write some functions to handle that.
Rather than the entire guid, a packed guid is a byte mask followed by all non-zero bytes.
The byte mask has bits set that correspond to where the following bytes go in the unpacked guid.
This is for optimizing packet size, since a guid is always 8 bytes but a packed guid can be as small as 2 bytes.

This took a while, because the client was crashing when I changed the packed guid from `<<1, 4>>` to anything else.
After trying different things and wasting a lot of time, I realized that the guid was in two places in the packet and they needed to match.
A quick fix later and things were working as expected.

Links:

* https://gtker.com/wow_messages/types/packed-guid.html

## Day 7 - June 8th

It was about time to start implementing the actual MMO features, starting with seeing other players.
To test, I hardcoded another update packet after the player's with a different guid, to try and spawn something.

![](20240607_23h59m00s_grim.avif)

Then I used a Registry to keep track of logged in players and their spawn packets.
After entering the world, I would use `Registry.dispatch/3` to:

* spawn all logged in players for that player
* spawn that player for all other players
* both using **SMSG_UPDATE_OBJECT**

After that, I  added a similar dispatch when handling movement packets to broadcast movement to all other players.
This is where the choice of Elixir really started to shine, and I quickly had players able to see each other move around the screen.


![](20240608_01h02m37s_grim.avif)

I tested this approach with multiple windows open and it was very cool to see everything synchronized.

![](20240608_19h17m07s_grim.avif)

I added a handler for **CMSG_NAME_QUERY** to get names to stop showing up as Unknown and also despawned players with **SMSG_DESTROY_OBJECT** when logging out.

This is where I started noticing a bug: occasionally I wouldn't be able to decrypt a packet successfully, which would lead to all future attempts failing too, since there's a counter as part of the decryption function.
I couldn't figure out how to resolve it yet, though, or reliably reproduce.

Up next was working on chat.

Links:

* https://gtker.com/wow_messages/docs/movementinfo.html
* https://gtker.com/wow_messages/docs/msg_move_start_forward_client.html
* https://gtker.com/wow_messages/docs/msg_move_start_backward_server.html
* https://gtker.com/wow_messages/docs/cmsg_name_query.html
* https://gtker.com/wow_messages/docs/smsg_name_query_response.html
* https://gtker.com/wow_messages/docs/smsg_destroy_object.html

## Day 8 - June 9th

To get chat working, I handled **CMSG_MESSAGECHAT** and broadcasted **SMSG_MESSAGECHAT** to players, using `Registry.dispatch/3` here too.
I only focused on /say here and it's all players rather than nearby.
Something to fix later.

![](20240609_01h43m59s_grim.avif)

Related to that weird decryption bug, I handled the case where the server received more than one packet at once.
This might've helped a bit, but didn't completely resolve the issue.

Links:

* https://gtker.com/wow_messages/docs/cmsg_messagechat.html
* https://gtker.com/wow_messages/docs/smsg_messagechat.html

## Day 9 - June 10th

I still had authentication with a hardcoded username, password, and salt, so it was about time to fix that.
Rather than go with PostgreSQL or SQLite for the database, I decided to go with Mnesia, since one of my goals was to learn more about Elixir and its ecosystem.
I briefly tried plain :mnesia, but decided to use Memento for a cleaner interface.

So, I added models for Account and Character and refactored everything to use them.
The character object is kept in process state and only persisted to the database on logout or disconnect.
Saving on a **CMSG_PING** or just periodically could be a good idea too, eventually.
Right now data isn't persisted to disk, since I'm still iterating on the data model, but that should be straightforward to toggle later.

Links:

* https://www.erlang.org/doc/apps/mnesia/mnesia.html
* https://elixirschool.com/en/lessons/storage/mnesia
* https://github.com/sheharyarn/memento

## Day 10 - June 11th

Today was standardizing the logging, handling a bit more of chat, and handling an unencrypted **CMSG_PING**.
I was thinking that could be part of the intermittent decryption issues too, but looking back I don't think I've ever had my client send that unencrypted anyways.

## Day 11 - June 12th

I wanted equipment working so players weren't naked all the time, so I started on that.
Using the MaNGOS item_template table, I wired things up to set random equipment on character creation.
Then I added that to the response to **CMSG_CHAR_ENUM** so they would show up in the login screen.

![](20240612_00h50m55s_grim.avif)

Up next was getting it showing in game.

## Day 12 - June 13th

Took a bit to figure out the proper offsets for each piece of equipment in the update mask, but eventually got it working.

![](20240613_18h13m22s_grim.avif)

Since equipment is part of the update object packet, it just worked for other players.

![](20240613_18h41m43s_grim.avif)

## Day 13 - June 14th

I had player movement synchronizing between players properly so I wanted to get sitting working too.

![](20240614_00h39m20s_grim.avif)

Whoops.
Weird things happen when field offsets or sizes are incorrect when building that update mask.

![](20240614_00h50m30s_grim.avif)

After that, I wanted to play around a bit by randomizing equipment on every jump.
Here I learned that you need to send all fields in the update object packet, like health, or they get reset.
I was trying to just send the equipment changes but I'd die on every jump.

![](20240614_20h11m08s_grim.avif)

After making sure to send all fields, it was working as expected.

## Day 14 - June 15th

Took a break.

## Day 15 - June 16th

Today was refactoring and improvements.
I reworked things into proper modules, since it was getting hard to debug when all the line numbers were wrong.
Now game.ex called the appropriate module's `handle_packet/3` function, rather than combining everything with `use`.

I also reworked things so players were spawned with their current position instead of the initial position saved in the registry.
This included some changes to make building an update packet more straightforward.

### Day 16 - June 17th

Today was just playing around and no code changes.

![](20240617_01h25m24s_grim.avif)

Not sure why the model is messed up here, but it seems like it's something with my computer rather than anything server related.

## Day 17 - June 18th

The world was feeling a bit empty, so I wanted to spawn in mobs.
First was hardcoding an update packet that should spawn a mob and having it trigger on /say.

![](20240618_18h20m25s_grim.avif)

After that, I used the creature table of the MaNGOS database to get proper mobs spawning.
I used a GenServer for this so every mob would be a process and keep track of their own state.
Communication between mobs and players would happen through message passing.
First I hardcoded a few select ids in the starting area to load, and after that worked I loaded them all.

Rather than spawn all ~57k mobs for the player, I wired things up to only spawn things within a certain range.
This looked like:

* Store mob pids in a Registry, along with their x, y, z position
* Create a `within_range/2` function that takes in two x, y, z tuples
* On player login, dispatch on that MobRegistry, using `within_range/2` to only build spawn packets for mobs within range
* On player movement, do the same

It worked really well and I could run around and see the mobs.

![](20240618_20h02m55s_grim.avif)

Next up was optimization and despawning mobs that were now out of range.

## Day 18 - June 19th

For optimization, I didn't want to send duplicate spawn packets for mobs that were already spawned.
I also wanted to despawn mobs that were out of range.
To do this, I used ETS to track which guids were spawned.

In the dispatch, the logic was:

* if in_range and not spawned, spawn
* if not in_range and spawned, despawn
* otherwise, ignore

Despawn was done through the same **SMSG_DESTROY_OBJECT** packet that I used for player logout.

After getting that working, I ran around the world and explored for a bit.

![](20240619_00h23m11s_grim.avif)

Found a bug in Westfall.
Turns out I wasn't separating mobs by map, so Westfall had mobs from Silithus mixed in.
To fix, I reworked both the mob and player registries to use map as the key.

![](20240619_00h41m11s_grim.avif)

Having mobs standing in place was a bit boring and I wanted them to move around.
Turns out this is pretty complicated and I'll actually have to use the map files so mobs stay on the ground properly.
There are a few projects for this, all a bit difficult to include in an Elixir project.
I'm thinking RPC could work, not sure if it'll be performant enough or not though.

The standard update object packet can be used for mob movement here, but there might be some more specialized packets to look into later too.

Without the map data, I couldn't get the server movement to line up with what happened in the client.
So, I settled with getting mobs to spin at random speeds.

{{<video src="./2024-06-19 18-21-02-[00.00.799-00.02.699]-audio.webm">}}


That was a bit silly and used a lot of CPU, so I tweaked it to just randomly change orientation instead.

Links:

* https://gtker.com/wow_messages/docs/movementblock.html
* https://gtker.com/wow_messages/docs/compressedmove.html
* https://github.com/namreeb/namigator

## Day 19 - June 20th

Here I got mob names working by implementing **CMSG_CREATURE_QUERY**.
This crashed the client when querying mobs that didn't have a model, so I removed them from being loaded.
I also started loading in mob movement data and optimized the query a bit to speed up startup.

![](20240620_18h33m27s_grim.avif)

I finally got some people to help me test the networking later that day.
It didn't start very well.

![](20240620_22h20m18s_grim.avif)

Turns out I hadn't tested this locally since adding mobs and the player/mob spawn/despawns were conflicting with each other due to guid collisions.
Players were being constantly spawned in and out.

![](20240620_22h22m55s_grim.avif)

I did some emergency patching to make it so players are never despawned, even out of range.
I also turned off /say spawning boars since that was getting annoying.
That worked for now.

![](20240620_23h03m05s_grim.avif)

There were still some major issues.
My helper had 450 ms latency and would crash when running to areas with a lot of mobs.
I couldn't reproduce, though, with my 60 ms latency.

Links:

* https://gtker.com/wow_messages/docs/cmsg_creature_query.html
* https://gtker.com/wow_messages/docs/smsg_creature_query_response.html

## Day 20 - June 21

To reproduce that issue, I connected to my local server from my laptop on the same network.
On my laptop, I used `tc` to simulate a ton of latency and wired things up so equipment would change on any movement instead of just jump.
This sent a ton of packets when spinning and I was finally able to reproduce.

![](20240621_23h35m50s_grim.avif)

Turns out, the crashing issues were from the server not receiving an entire packet, but still trying to decrypt and handle it.
I was handling if the server got more than one packet, but not if the server got a partial packet.

Referencing Shadowburn, the fix for this was to let the packet data accumulate until there's enough to handle.
Seems to have fixed all of the network-related issues.

To fix the guid collision issue, I added a large offset to creature guids so they'll never conflict with players.

## Day 21 - June 22

Took a break.

## Day 22 - June 23

Worked on **CMSG_ITEM_NAME_QUERY** a bit, but there's still something wrong here.

![](20240623_17h34m30s_grim.avif)

Decided spells would be next, so I started that.
First was sending spells over with **SMSG_INITIAL_SPELLS** on login, using the initial spells in MaNGOS.
Everything was instant cast though, for some reason.

![](20240623_20h41m09s_grim.avif)

Turns out I needed to set **unit_mod_cast_speed** in the player update packet for cast times to show up properly in the client.

![](20240623_21h53m34s_grim.avif)

I started by handling **CMSG_CAST_SPELL**, which would send a successful **SMSG_CAST_RESULT** after the spell cast time, so other spells could be cast.
I also handled **CMSG_CANCEL_CAST**, to cancel that timer.
This looked a bit like the logout logic.

The starting animation for casting a spell would play, but no cast bar or anything further.

Links:

* https://gtker.com/wow_messages/docs/smsg_initial_spells.html
* https://gtker.com/wow_messages/docs/cmsg_cast_spell.html
* https://gtker.com/wow_messages/docs/cmsg_cancel_cast.html
* https://gtker.com/wow_messages/docs/smsg_cast_result.html

## Days 23 to 26 - June 24 to 27

Took a longer break.

## Day 27 - June 28

I was able to get a cast bar showing up by sending **SMSG_SPELL_START** after receiving the cast spell packet.

![](20240628_22h57m17s_grim.avif)

The projectile effect took a bit longer to figure out.
I needed to send a **SMSG_SPELL_GO** after the cast was complete, with the proper target guids.

![](20240628_23h58m06s_grim.avif)

Links:

* https://gtker.com/wow_messages/docs/smsg_spell_start.html
* https://gtker.com/wow_messages/docs/smsg_spell_go.html

## Day 28 - June 29

I got self-cast spells working by setting the target guid to the player's guid.

![](20240629_00h13m16s_grim.avif)

## Day 29 - June 30

Another break.

## Day 30 - July 1

Since I had spells somewhat working, next I had to clean up the implementation.
I dispatched the **SMSG_SPELL_START** and **SMSG_SPELL_GO** packets to nearby players and fixed spell cancelling, so now movement cancels spells as expected.

![](20240701_18h44m36s_grim.avif)

## Day 31 - July 2

I added levels to mobs, random from their minimum to maximum level, rather than hardcoding to 1.
Then I made spells do some hardcoded damage, so mobs could die.
Noticed that mobs would still change orientation when dead, so added a check to only move if alive.

![](20240702_20h41m22s_grim.avif)

That seemed like a good stopping point and was 1 month since I started writing code for the project.

# Future Plans

I plan on slowly working on this, adding more functionality as I go.
My goal isn't really a 1:1 Vanilla server, but more something that fits well with Elixir's capabilities.
I'd like to see how many players this approach can handle, and how it compares in performance to MaNGOS, eventually.

Some things on the list:

* proper mob + player stats
* proper damage calculations
* pvp
* quests
* mob movement + combat ai
* loot + inventory management
* more spells + effects

So still plenty more work to do. :)
