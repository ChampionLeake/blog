---
layout: post
title: Finding DSiWare Vulnerabilites
description: 
tags: [sample post]
image:
  background: ps_neutral.png
---

# Introduction:
I noticed that there were some questions about how some reverse-engineers or hackers find exploitable vulns in DSiWare applications. I also noticed that there weren't any write-ups, nor any documentation, on discovering DSiWare application flaws. 

But today, I wanted to explain how you could find vulnerabilities in DSiWare applications, as well as saving people time from not having to ask whether a game is vulnerable or not. Plus, I think this could be a good introduction to learn how to get into exploiting applications or games. Without further ado, let's get started :D

# Things to know:
To get things out of the way, you need know some things before you get started.
* A crash isn't necessarily exploitable. Of course, a crash might be exploitable, but if it doesn't overwrite parts of the stack registers, it's not likely to be exploitable - but that will be covered later.
* I'll only be covering stack smashing flaws since they're the easiest to start with and I'm familiar enough with these types of vulnerablities to explain what they are. I suggest that you look up how stack smashing works. There's an overview on this topic over at [CTurt's DS-exploit-finding guide](https://cturt.github.io/DS-exploit-finding.html).
* You must know that the DSi uses a ARM Processor. R0-R12 are generally based use for calculation and the last 3 (R13 (SP = Stack Pointer), R14 (LR = Link Register), and R15 (PC = Program Counter)) are unique. You can find more details on these registers [here](https://problemkaputt.de/gbatek.htm#armcpuregisterset)!

# Tools I used:
Now let's move on! You'll need the right tools to be able to get through this article. Here are the tools I used in this article:
* [NO$GBA](https://problemkaputt.de/gba.htm) - A really useful debugging emulator which can emulate your DSi NAND.
* [TWLTool](https://github.com/WinterMute/twltool) - Used to decrypt/encrypt your NAND dump. That way, you can edit the NAND with the save you're injecting for NO$GBA.
* [HiyaCFW (Optional)](https://dsi.cfw.guide/) - This makes it easy to access your savedata as SDNAND isn't encrypted. This is optional, though.
* [FWTOOL](https://davejmurphy.com/%CD%A1-%CD%9C%CA%96-%CD%A1/) - This is used to dump your NAND so you can use it in NO$GBA to debug/emulate your DSi. You'll need homebrew access to run it.
* [HxD](https://mh-nexus.de/en/hxd/) - I use this hex editor a lot, and it has helpful analysis features that we'll use in this guide.
* [VBinDiff](https://www.cjmweb.net/vbindiff/) - Used mostly to compare files against each other. This will help find the differences in a DSiWare savefile for checksums, different names or other garbage tbh :/
* [OSFMount](https://www.osforensics.com/tools/mount-disk-images.html) - This tool will be useful for mounting the public.sav file to access the 'real' savedata. Although the save is in the public.sav, it's useful to just mount the file and take what's in it to observe and then put it back.
* Money - You'll need it to buy these games you don't have. This is obviously not required if you're tesing games you already own.

# Picking a target:
For the sake of this article, I'll be looking for stack smashing vulnerabilities in DSiWare applications. I would start with games that have profile names or some type of username input. It doesn't have to be strictly profile names, and there are some games that allow putting in your name if you achieve a high-score - like Fieldrunners. Even if the string is only 3 bytes long, you can still try. 

So, I'm going to use a simple game called "WordSearcher" that I already looked at. It's not vulnerable, but it's useful for a really good example. So remember what to do:

* Look at the list of [examined DSiWare titles](http://dsibrew.org/wiki/DSiWare_VulnList) before you start. Don't waste your time if you're looking at finding new flaws.
* Look for profiles that allow you to set a custom name.
* Look for games that allow you to set a custom name on achieving a high score.
* Avoid games with multiple elements (some RPGs) unless you like the challenge for solvling checksums. Some may not have too many checksums but it's recommended to start simple.

It shouldn't be a challenge to pick a title to observe. I'll also be looking at Fieldrunners to show you what it looks like to find a useful flaw.

# Getting past Checksums:
Checksums play a huge role for certain games or applications. For those of you who do not know what checksums are, allow me to break it down for you. 

The word "Check" should ring a bell. The word "Sum" is like the total of some numbers or bytes. Maybe adding if you're a mathematician ALWAYS adding. So if you really put those 2 meanings together, you can tell they're checking something. But for what purpose? It's to ensure a file or a section in a file is not corrupted or tampered with. Without checksums, programs/applications would be extremly vulnerable to simple attacks. 

Editing a savefile with checksums without actually patching those checksum bytes will normally make the game delete or rewrite the save. So that's what we need to take care of before we start editing whatever we please. Let's create some saves and compare them so we can locate the checksum(s) for "WordSearcher". It's common for some DSiWare titles to use a familiar CRC algorithm which gives you a greater chance to patch checksums easily with [HxD](https://mh-nexus.de/en/hxd/) or other checksum calculators. 

So, I'm going to create 2 savefiles (two "public.sav"s). One will contain the username "AAAAAAAA", and the other will contain "AAAAAAAB". We'll eventually use [VBinDiff](https://www.cjmweb.net/vbindiff/) to compare the two files and look for differences. We first have to fetch the real save that's mounted within the "public.sav" file. This can be done with [OSFMount](https://www.osforensics.com/tools/mount-disk-images.html) or any program that can mount disk images.


You must have a DSi NAND dump (that you dumped yourself). [Wintermute's FWTOOL 2.0.0](https://davejmurphy.com/%CD%A1-%CD%9C%CA%96-%CD%A1/), homebrew application, should help you dump your NAND. The last things you'll need are your DSi's CID and ConsoleID, which you can obtain by looking at [Gadorach's downgrading guide](https://gbatemp.net/threads/dsi-downgrading-the-complete-guide.393682/%22). 

I assume that you already have the things you need. I'll quickly go over what you need to do when mounting your NAND to fetch the public.sav(s). 

Your NAND needs to be decrypted before any mounting should occur.
* Open up "OSFMount".
* Select "Mount New...".
* Click the "..." button to the right of "Image file".
* Find your DSi's NAND and select "Partition 0".
* Click "OK"
Your setup should look like the screenshot below.
<figure>
  <img src="https://championleake.github.io/articles/assets/OSFMount_SetUp.JPG">
</figure>

Now you should be able to open your mounted NAND and look for the correct titleid for your corresponding DSiWare title in "/title/0030004/TITLEID/data/". Copy the "public.sav" file to somewhere safe.

You can basically repeat the whole process. Create a new save with a different name, dump your NAND again, decrypt your NAND, mount your NAND, and find the "public.sav" again. I still suggest renaming those files to stay organized. After you're done you can still rename it back to its original name. Let's now move on to mounting our savefiles.
* Open up "OSFMount".
* Select "Mount New...".
* Click the "..." button to the right of "Image file".
* Find the "public.sav" (or the renamed savefile), I renamed the "public.sav"s to "AAAAAAAA.sav" and "AAAAAAAB.sav" for the purpose of this article.
* Uncheck the "Read-Only drive" box and check the "Mount as removeable media" option.
* Click "OK" and you should be ready to go.
Copy the file that's mounted from the savefile and place it somewhere safe. Repeat the whole process with the other savefile as well. 
<figure>
  <img src="https://championleake.github.io/articles/assets/OSFMount_ScreenShot.JPG">
</figure>
We can now move onto using VBinDiff to compare the two savefiles. 

Select the two savefiles and drag them to the VBinDiff program and it should give you this.
<figure>
  <img src="https://championleake.github.io/articles/assets/VBinDiff-ScreenShot.JPG">
  <img src="https://championleake.github.io/articles/assets/VBinDiff-ScreenShot2.JPG">
</figure>

We seem to have found the differences in these saves. By pressing Enter or manually scrolling, it'll find another difference in the saves. At offset 0x0, for both files, it seems we might be dealing with CRC32, considering that the first 4 bytes are different within the saves. CRC8/Checksum-8 would be 1 byte long, CRC16/Checksum-16 would be 2 bytes long, and of course CRC32/Checkum-32 would be 4 bytes long. Since we now know where the checksum is located, we're going to start investigating what and why those bytes are what they are. Let's open our saves in [HxD](https://mh-nexus.de/en/hxd/). 

I mentioned eariler that HxD has really useful features such as checksum analysis. This is where we're going to attempt to use those features to figure out if the checksum is using a classical-polynomial algorithm or using a custom-polynomial algorithm. I'll only be cover classical-polynomial, since custom checksum algorithms require dumping the game's code and using a disassembler (like IDA Pro) to reverse-engineer the code to find the CRC function. I don't know how to do something like that yet, but we'll be doing something else that I'd like to call [WemI0Sum](https://github.com/WemI0) (since he taught me this way of dealing with checksums). 

What is the 'WemI0Sum' method? It's a process that can help easily patch checksums (if the game uses a classical-polynomial algorithm). This method can also be used to determine if a game uses a classical-polynomial algorithm or not. The process consists of highlighting/selecting blocks/chunks in the save and using HxD's CRC analyzer. The 'Weml0Sum' method can be time-consuming because you're guessing which blocks to select to match the initial checksum in the save file. The 'WemI0Sum' method is good if you don't want to deal with dumping the game's code, so let's try this method with these savefiles we have. 

Open one of the savefiles in HxD if you haven't. I selected basically after the first 4-bytes. It's all about guessing which blocks match the current checksum. I went from 0x4 to the very end of the file. Finding the right blocks could take hours or days to find the perfect area where the checksum will be correct. But take a look at the screenshot of how things should look if you followed along:
<figure>
  <img src="https://championleake.github.io/articles/assets/HxD-ScreenShot.JPG">
</figure>

The 'WemI0Sum' method help us will determine if the game uses a classical-polynomial algorithm or not.
* Once you have the selected area, navigate to "Analysis".
* Select the "Checksums..." option.
* Be sure to choose the appropriate algorithm. In this case, the game uses CRC32/Checksum-32 so I select CRC32 first and pressed 'OK'.
<figure>
  <img src="https://championleake.github.io/articles/assets/HxD-ScreenShot2.JPG">
</figure>

Now what do you notice about the yellow highlighted areas in the screenshot above? Yes? No? Either way, THE RESULTS ARE THE SAME, BUT BACKWARDS. That's good! Most calculators will present the checksum in [little-endian](https://en.wikipedia.org/wiki/Endianness#Little-endian) format. 

Note how the bytes "C2 EC 62 15" in the save are backwards from the checksum "15 62 EC C2". To make sure this is correct we're going to take a look at the other file to make sure.
<figure>
  <img src="https://championleake.github.io/articles/assets/HxD-ScreenShot3.JPG">
</figure>

Once again, the bytes seem to be identical but in little-endian format. We can confirm that the game does use a common CRC32 polynomial algorithm. From there, you can basically use an open-source CRC patcher - just edit/add the offsets, and there you go, you have yourself a CRC patcher for your game. Now that we cracked the checksum of this game, we can move on to the next section. Keep in mind that not all games will be easy with this method due to reasons I explained earlier. You'll need to disassemble the game's code and look for the CRC function itself. That is slightly beyond the scope of this article.

# Attempting an overflow:
If you've made it this far, congratulations, you're finally ready to go ahead and edit the game's savefile to look for potential vulnerabilities. WordSearcher's profile name has a character limit that means names can only be 8 bytes long. Let's see what happens if we use an very long string for our profile. I like using "AA"s since I find that easier to look for in the stack and see if any were overwritten. 
<figure>
  <img src="https://championleake.github.io/articles/assets/HxD-ScreenShot4.JPG">
</figure>

Once you're done adding/editing the profile name with a long string, you can go through the 'WemI0Sum' process to correct the checksums or, if you already made a tool to patch it for you, use that. 

Rename the savefile back to its' original name (in that case, the game's original save name was 'game_user.bin'). You should copy and paste the overflowed savefile back into the 'public.sav' while OSFMount is open. Once things are done, you're free to "Dismount all & Exit" the program. (Click OK if there's a warning). 

You'll need to decrypt your NAND and inject the 'public.sav' to its' original directory. After that, you can dismount the NAND and re-encrypt it with TWLTool so you can run it in NO$GBA. You can find the tutorial for that [here](https://gbatemp.net/threads/dsi-downgrading-the-complete-guide.393682/) under the "Decrypting your NAND" spoiler. 

Once our NAND is re-encrypted, we can now try to emulate our NAND in NO$GBA. I won't go over how to set up the NAND for use in NO$GBA since it's a bit time-consumming. You can find instructions on the offical NO$GBA website. 

Open up NO$GBA and start the system menu by choosing any DS(i) ROM in the files menu. We now can test our overflowed save for 'WordSearcher'.
<figure>
  <img src="https://championleake.github.io/articles/assets/Inital-Overflow-ScreenShot.JPG">
</figure>

ON REAL HARDWARE, this part of the game would freeze and crash. Let's see if this happens in NO$GBA.
<figure>
  <img src="https://championleake.github.io/articles/assets/Inital-Overflow-ScreenShot2.JPG">
</figure>

If you remember from the beginning of this article, I mentioned that this game is not vulnerable. I'm going through this process to show you what would a useful and useless crash would look like in NO$GBA. In the emulator, it doesn't seem to have any error popups, overwritten registers in the highlighted regions, etc. But it's still not over. There are always more creative ways to look for flaws in a application. 

We know that the profile names are not vulnerable. The game uses plaintext crossword levels. This could be a potential flaw in the game if it's bounded. So let's try it! We'll first need to generate the crossword data in the save, which can be done by just starting a crossword level and then saving and quitting. We now have to repeat the process of dumping our NAND, decrypting it, fetching the save, mounting it, and then editing our save.
<figure>
  <img src="https://championleake.github.io/articles/assets/HxD-ScreenShot5.JPG">
</figure>

Instead of "AA"s, I changed things up and used "BB"s. We used a very large string once again to replace/overwrite one of the crossword levels. We can now finish things up with the 'Weml0Sum' method or use a working CRC patcher to correct the checksums. Now we just need to repeat injecting the save into the NAND. 

We now test our overflowed save once again in NO$GBA and there seems to be no errors nor any registers being overwritten. But at least there can be custom/non-existent strings in the crossword board itself! We can now conclude that this game doesn't seem to be vulnerable to stack smashing. :(
<figure>
  <img src="https://championleake.github.io/articles/assets/Inital-Overflow-ScreenShot3.JPG">
</figure>

Now let me show you what a useful crash would look like in NO$GBA with the game "Fieldrunners". You can tell if the game crashed in NO$GBA if an error popup shows and, in the highlighted area, some of the registers are overwritten with "AA"s. If R15 (the PC) is overwritten as well, you have yourself an exploitable application! 
<figure>
  <img src="https://championleake.github.io/articles/assets/Overflow-Example.JPG">
</figure>

# Closing:
So I hope this article/writeup gave you an overview on how to find vulnerabilities in DSiWare titles. Hopefully I can update this article to cover things like reverse-engineering the checksums by using IDA Pro and maybe more. If you ever have any questions, you can let me know on my [[Discord server](https://discord.gg/XRXjzY5) and I'll try my best to answer all of your questions. 

Hopefully I can get a writeup about porting NDS-HBMenu to exploitable games. Anyways, I'm suprised you made it this far considering how long this article is. I hope you learned something today at least.

# Credits:
I would like to thank all of the following:
* [Jhynjhiruu](https://github.com/Jhynjhiruu) - Your friend neighborhood grammarly. Big thanks for improving the grammar.
* [yellows8](https://github.com/yellows8) - for originally exploiting [Fieldrunners](https://github.com/yellows8/dsi/tree/master/exploits/fieldrunhax).
* [nocash](https://problemkaputt.de/) - for creating no$gba and the excellent documentation.
* [WinterMute](https://github.com/Wintermute) - for creating FWTOOL and TWLTool.
* [WemI0](https://github.com/WemI0) - for teaching me the 'WemI0Sum' method.
* [CTurt](https://github.com/CTurt) - for the inspiration of his DS-exploit-finding guide.
* [Jerbear64](https://github.com/jerbear64) & [emiyl](https://github.com/emiyl) - for the DSiCFW guide
If you're not on this list and I forgot to add you, let me know.
