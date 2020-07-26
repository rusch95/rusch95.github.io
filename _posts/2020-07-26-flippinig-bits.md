---
layout: post
title:  "Flipping Bits for Fun and Profit"
date:   2020-07-26 12:35:25 -0400
categories: jekyll update
---

Today’s story is about how I hackily patched a _division-by-zero_ bug in the game _Brogue_ by using a few useful tools available on MacOS. 

I’m a sucker for well-made roguelikes. I cut my teeth on the venerable _NetHack_, had a long affair with the very 
focused _Dungeon Crawl: Stone Soup_[^1], and have flitted around with fresh takes on the genre like _Cataclysm: Dark Days Ahead_[^2].

My latest roguelike obsession has been _Brogue_.

![Screenshot of Brogue](https://user-images.githubusercontent.com/10816808/88485793-4d4c7600-cf3e-11ea-8c56-3599c806763a.png)

Brogue, whose development began in 2009, is fairly well renowned and influential for its 
[very streamlined game design](https://www.tigsource.com/2012/01/15/brogue/) and 
[accessible UI](https://www.rockpapershotgun.com/2015/01/23/have-you-played-brogue/). 
You may have even been exposed to Brogue’s core mechanical conceits through _Pixel Dungeon_,the oft-forked and remixed mobile app, such as I was.[^3]

While procrastinating working on performance reviews, I had been playing some Brogue. 
Picking up a magic Rapier of Confusion in the game, a quick weapon with powerful lunges, I enchanted it heavily with magic scrolls. 

Stepping onto a new level, I enhanced it one more time with a Scroll of Enchantment. 
I then went to stab a lonesome goblin, when my game suddenly crashed. I shrugged, as this is not too uncommon. 
Reloading the game again from my auto-save and appearing at the start of the level, I enchanted the rapier once again and tried to stab the goblin. 
Again, the game crashed.

Usually the game crashes for very inexplicable reasons, but this was a repeatable issue. 
I reloaded the game, enchanted the weapon, and then tried to look at the weapon’s stats. 
The game crashed.

This time instead of hand waving away the MacOS crashed program prompt, I clicked on the report issue button. 
Thanks to Brogue being open-source, the program was compiled with debug symbols, 
so I could see by the stack trace that crash was being caused by a 
division-by-zero error by the fp_sqrt function called by the function runicWeaponChance. 
By searching for _runicWeaponChance_ in the repo, 
I quickly found the line at fault [here](https://github.com/tsadok/brogue/blob/master/src/brogue/PowerTables.c#L418).

At first, I tried to do things the proper way. 
I downloaded the code from the official site and tried to compile it with Xcode. 
As many know, simply compiling code can often turn into a fun exercise in yak shaving. 
Apple appears to like to deprecate things very quickly with my version of Xcode 
refusing to touch the old Swift code that Brogue uses for its Mac UI. My heart pines 
for Windows more humane treatment of elderly code, as 
opposed to MacOS’s [Logan’s Run-esque](https://en.wikipedia.org/wiki/Logan%27s_Run_(film)) regime.  

I could have kept banging my head against Xcode, but it seemed like so much effort to change one 
line of code. At that point, I made my decision: even though there was probably a much better way 
of fixing this bug, I was just going to patch it out by hand. 

This was a nice exercise to explore the state of reverse engineering on MacOS, albeit with a non-obfuscated, source available, symbol lush binary. 

I first took a stab at the binary with [Hopper Diassembler](https://www.hopperapp.com/). 
The free-trial is fairly restrictive, but the labeling provided was very useful in 
hunting down the assembly code section corresponding to the buggy line of code.

I then used _otool_ to get a text disassembly dump via the command `otool -vtj Brogue > asmdump`.
_otool_ is a nifty CLI program for working with LLVM produced code and artifacts that I believe is on MacOS by default.
I’m a little more dextrous interacting with code via Vim, so it was easier to play around with the 
specific assembly sections. Additionally, this tool gave me the full hexdump of each line of assembly, 
which I couldn’t easily tease out of Hopper.

Via a combination of Hopper and otool, I anchored on that call to `fp_sqrt`, represented by `callq 0x1000209f0` 
in the `otool` dump. Scanning up, I found the telltale `cmpq` then `je` that usually represents a branch in x86-64 assembly. 

I struggled a bit finding ways of reversing the otool dump back into code, especially since I wanted to play 
with Brogue, instead of playing around with getting the correct linked header files. Thus, I fell back on trusty xxd. 

I dumped the binary to a structured hexdump with `xxd Brogue > hexdump`.
`xxd` is a very simple, fairly standard Unix program for creating readable hexdumps of binaries. 
I found particularly unique hexcode in the neighborhood of that branch in the otool dump `e8 7811` and jumped to it with the fun regex `e8.\?78.\?11` 
since I didn’t know how it would be split in the hexdump. I then found the branch leading to the code in the hexdump.

Looking back at the code and noticing that it was just a slight clamp on how powerful this Rapier of Confusion would be. 
I decided to short circuit the code. I changed the `JE` (jump if the previous cmp check was equal) to `JNO` (jump if not overflow). 
This jump was now was now always taken, skipping over the buggy code previously visited for adjust the proc chance for quick weapons. 
To do this change, I changed `0F 84` to `0F 81` I reversed the hexdump with `xxd -r hexdump > Brogue`.

I loaded back up the game. Sadly, the game handles saves via replay, which now diverged given that the proc chance was higher. 
Someone actually smart probably could have patched it to actually fix the bug and avoid the desync issue. 
Instead, I just grabbed the seed for that run and quickly played back to the autosave.

And there was no more crashing. I got back to the goblin and killed it. 
Soon after that, I had an unfortunate run-in with some eels and was likewise killed.

I went back to working on my performance review, having learned a few more tricks for 
hacking binaries on MacOS and a greater fear and respect for eels.  

[^1]: Or DCSS as the cool kids say

[^2]: Now you may say, “that’s a rouge-_lite_, good sir.” To which I respond: “Ok.” See  [http://www.roguebasin.com/index.php?title=Berlin_Interpretation](http://www.roguebasin.com/index.php?title=Berlin_Interpretation) for an ideological struggle similar Vim vs. Emacs or schism of the Early Christian Church. Roguelikes are _serious_ business. 

[^3]: All of these mentioned games happen to be free-range cage-free open source code
