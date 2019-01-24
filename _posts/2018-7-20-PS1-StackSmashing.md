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

# Attempting an overflow:
Now that you have the things you need, you are now ready to go ahead and edit start messing with savegames. Castlevania Chronicles comes with the Arranged/Original versions of the game. Both have separate profile name entities which makes it a dual win if you wanted exploit both versions of the game. If one doesn't work, there's a chance that the other will work.
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/NO$PSX-ScreenShot.JPG">
</figure>

I'll be looking over the 'Original' version of the game since it's more accessible than the arranged version. As mentioned before, I am able to compose a username string for my profile in the 'Original' version. Let's now look for the "AAAAAAAA" string in the memory card save.
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/HxD-ScreenShot.JPG">
</figure>

NO$PSX stores memory card saves in the 'MEMCARD' directory. Most saves will have the '.mcd' extension. Close NO$PSX and open the corresponding memory card save in HxD. Taking a look at the save, we were able to find our profile name. 

Since the PS1 doesn't really have any checksums, you can pretty much freely edit the save. Let's start with a large string and see what happens when we open it in NO$PSX. We'll simply narrow things down to letters to pinpont what's overflowing and where.
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/Initial-Overflow-ScreenShot.JPG">
</figure>

From the screenshot above, we'll run this memory save in NO$PSX. We'll see if it crashes and we'll take a look at the CPU registers to see that $ra/r31 is being overwritten with our long string. If you notice that the game just crashes and that the NO$PSX exception doesn't pop-up, you might want to pause the emulator itself and look at the CPU registers that way. If this still doesn't work, the game will have encountered a fatal crash and you might've overwritten too much. It's best to narrow things down 1 byte to another. Overflowing too much of the stack will just do random things like NOP(s). 

Looking at the screenshot below, we were given an exception error in NO$PSX!
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/NO$PSX-Overflow.JPG">
</figure>

Our profile name has overflown some of the stack registers and the main target, $ra/r31, has been overwritten as well with our FF(s). This game is vulnerable :D!

# Refining the Return Address:
Okay, so now we know that our game is vulnerable, let's refine our string to find the where the return address is being overwritten. This is a simple procedure since all you're doing is simplifiying the string. So what I did for this game is that I rewrote part of the string with 0x20 (the ' ' character for those of you that haven't memorised the ASCII encoding scheme) and focused on the FF(s). For simplicity, I'll rewrite those FF(s) to numeric hexidecimals. This should be enough to pinpoint where our return should be placed as shown below.
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/HxD-ScreenShot-2.JPG">
</figure>

It doesn't matter if you do either numeric or alphabetical hexidecimals, as long as you can easily pinpoint where the return address gets overwritten, you should be fine. 

Let's now run this in NO$PSX and see what we get. Also, remember that the CPU registers are 4 bytes long. As the game runs, our refined string should be read in the return address. From there, we have basically what we needed to overwrite. 


And what do you see... if you look at the register?
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/NO$PSX-Return-Address_FOUND.JPG">
</figure>

$ra/r31, it's being overwritten with 0x55554444. But you should also note that it's displayed as little-endian. Our original input was 0x44445555, so it's been reversed. Always keep that in mind. But moving on, we now know where our return address is being overwritten.

# Defining our Return Address:
Let's go back into [HxD](https://mh-nexus.de/en/hxd/) to trail our return address.
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/HxD-ScreenShot-3.JPG">
</figure>

So once again, we replaced the unnecessary bytes with 0x20 to keep things clean and rewrote the 4 bytes with 0x58 (translates to 'X'). We can apply this to a real MIPS exploit to jump to our payload and start running unsigned code. 

This won't be covered in this writeup since similar jump addresses can be explained. Plus, real code execution hasn't been achieved yet while I was doing this. But let's save the memory save and head right back into NO$PSX.
<figure>
  <img src="https://championleake.github.io/articles/assets/PS1/NO$PSX-Jump-Address.JPG">
</figure>

Once you trigger the overflow again, you should be greated with something similar you placed at the jump address. We have full control over the return address and can jump to anywhere in the savegame. We used our own bytes to overwrite the return-address. 

As mentioned before, this can be used to jump to our payload if we wanted to get demo code working. But now, we can call this simple exploit "CastleSploit" :D.

# Closing:
Hopefully this article/writeup gave you an overview on stack smashing games running on MIPS processors. Eventually, I would like to see this being used in action to actually execute some demo code. Hopefully I can get this to work on my own as well and update this guide if needed. Also, this can be applied to the PSP since its games are also coded for MIPS processors. You can find neat writeups on those somewhere. 

If you have any questions regarded to anything related to this article, you can contact me on Twitter: [@ChampionLeake79](https://twitter.com/ChampionLeake79)

# Credits:
I would like to thank all of the following:
* [Jhynjhiruu](https://github.com/Jhynjhiruu) - Your friendly neighbourhood Grammarly™. Big thanks for improving the grammar (note: there's only so much that can be done; you should have seen this before I started!).
* [@AcidSnakeDev](https://twitter.com/AcidSnakeDev), [@freakler94](https://twitter.com/freakler94), [@qwikrazor87](https://twitter.com/qwikrazor87) - Inspiriation on Sony PSP (PS1-mini hacking)
* [St4rkDev](htttps://twitter.com/St4rkDev) - Helpful friend in using integer overflows as a way to take over some registers
* Everyone else who contributed to giving me helpful advice
If you're not on this list and I forgot to add you, let me know.
