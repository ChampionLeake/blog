---
layout: post
title: PS1 Savegame Stack-Smashing
description: 
tags: [sample post]
image:
  background: triangular.png
---

# Introduction:
Over the years, there have been multiple vulnerabilities found for many PlayStation consoles and handhelds, like the PSP, PS Vita (and the PSTV, its miniaturised console counterpart), PS2, PS3, etc. Looking at how the original Sony PlayStation has been cracked (ASM ROMhacks, CD-Drive burning and POPS), there were no write-ups detailing the process of stack-smashing the savegames themselves on official retail discs. 

But today, I wanted to share how I found stack smash vulnerabilities in savegames on the original PlayStation. This guide will also teach you the basics of finding flaws in games that run on MIPS processors. Without further ado, let's begin :D

# Before we start:
You need know some things before you get started. Otherwise, you'll be going in blind doing this stuff.
* A crash isn't necessarily exploitable (as noted in my past write-up). A crash might be exploitable, depending on what's being overwritten or what's being messed with
* I suggest that you look up how stack-smashing works. To save time, [let me Google that for you](http://www.google.com/search?q=what+is+stack+smashing).
* Know that PS1 games are compiled to [MIPS machine language](https://en.wikipedia.org/wiki/MIPS_architecture). We'll mainly be focusing on getting control of the register $ra/r31, which is our return address. You can find more details on these registers [here](https://problemkaputt.de/psx-spx.htm#cpuregisters)!
* Most or almost all PS1 games do not use checksums, nor do they have any type of ASLR (as far as I know).
* Some overflows in games can turn into integer overflows, which would be a bit of a pain to deal with but they're still exploitable (integer overflows will not be covered in this write-up).

# Tools I used:
Now let's move on! You'll need the right tools to be able to get through this write-up if you're following along. Here are the tools I used:

* [no$psx](https://problemkaputt.de/psx.htm) - A really useful emulator for the original PlayStation with debugging capabilities.
* [HxD](https://mh-nexus.de/en/hxd/) - I use this hex editor a lot, and it has helpful analysis features that we'll use in this guide.
* PS1 games - Of course you're going to need a game to do some research - one that you obtained a legitimate backup of by dumping yourself, of course :P

# Picking a target:
For the sake of this article, we'll be looking for potential stack-smashing vulnerabilities. I usually start with games that have profile names or some type of username input like cars, clubs names, etc. It doesn't have to be strictly profile names, and there are some games that allow putting in your name if you achieve a high-score. Memory card names are another possible attack vector. Even if the string is short, you can still try. 

So, I'm going to be using the game "Castlevania Chronicles" (This game is really good and you should actually play it) since there's 2 ways to actually stack-smash this game. So remember what to do:
* Look for profiles that allow you to set a custom name.
* Look for games that allow you to set a custom name on achieving a high score.
* Don't just edit a section of the save if you don't know what it's used for. Doing so may corrupt the save itself.

It shouldn't be a challenge to pick a title to observe. If a game has user-inputs like names, you should be good to go.
