+++
title = 'My worst (and most popular) project'
date = 2024-07-15
draft = false
tags = ['programming', 'Steam Deck', 'linux']
+++

[![](https://api.star-history.com/svg?repos=pikdum/steam-deck&type=Date)](https://github.com/pikdum/steam-deck)

After getting my Steam Deck in early 2023, I wanted to try installing a mod collection for Skyrim.
The recommendation at the time was to do it all on a Windows PC and then copy all the files over, but I didn't have a Windows PC.
Instead, I wanted to get [Vortex](https://github.com/Nexus-Mods/Vortex/) running on my Steam Deck and do all the modding on-device.

I put together some bash scripts with an installation process inspired by [CryoUtilities](https://github.com/CryoByte33/steam-deck-utilities) and was able to get things working.
After installing, it added a shortcut to the desktop to launch Vortex and a "Skyrim Post-Deploy" shortcut to run after deploying mods, to copy some files to the right places.
Originally it only supported Skyrim SE, but was later expanded to cover more games by adding more post-deploy shortcuts.

After I got the collection installed, I recorded a video with instructions and posted it to a Discord channel as an alternative install method, linking to the repository.
It got fairly popular and was even mentioned in a YouTube tutorial.
Things were working pretty well, until they weren't.

## What went wrong?

1. **I stopped actively using it, but was still trying to support it.**
Large collections take a very long time to install, it uses a ton of disk space, and just isn't a good experience.
It quickly lost its novelty and I switched back to just playing and modding on desktop using [vortex-linux](https://github.com/pikdum/vortex-linux) instead.
I don't use my Steam Deck much nowadays except for travel either.
2. **It was chasing a moving target.**
Vortex updates frequently broke things, requiring new Proton versions and .NET framework workarounds.
My favorite projects solve a problem and stay solved, but this kept getting worse the more I left it alone.
3. **If straying from the happy path even a little bit, things really sucked.**
Users would need to be experienced with modding on Windows and familiar with how Proton works on Linux to get things working.
Since Vortex uses a different Proton prefix as the games, things just didn't work as expected.
This was especially the case for games I didn't have that I couldn't add explicit support for.
4. **Bugs.**
Right now there's a bug that completely breaks it that I'm not able to reproduce nor fix.
My best guess is that I'm using Proton or the Steam runtime incorrectly somehow.
Huge bummer, since my goal for this was to make Vortex just work on Steam Deck, but it doesn't.

I wasn't able to get things up to my quality standards.
It was supposed to just work, but instead it's frustrating to use.
For now, I'm going to slap a notice in the README to help clarify expectations.

I'm hoping the [Nexus Mods App](https://github.com/Nexus-Mods/NexusMods.App) will completely deprecate this project once it supports Skyrim and other Bethesda games.
Until then, I'll keep this in maintenance mode and hopefully it still works for some people.
