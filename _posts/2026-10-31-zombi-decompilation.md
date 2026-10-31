---
layout: post
title: decompiling ZombiU
description: we decomp'ing babiessss
tags: zombi zombiu reverse-engineering
thumbnail: /images/zombiucover.jpg
thumbnail_alt: ZombiU cover art
---

ZombiU was developed by Ubisoft Montpellier and released as a Wii U launch title in 2012. You play as a survivor dropped into a zombie-infested London. You have to scavenge, loot, and survive while a mysterious voice known only as the Prepper guides you. More than once, you try to escape the whole hellhole for good.

One of the game's best mechanics was the Rebirth system. When you die, you don't just respawn, but rather you wake up as a different survivor and have to make your way back to where you died. Your old survivor could either just be laying there, or was an infected zombie, with all the gear you had on them, so you'd have to kill your previous self to get your stuff back.

ZombiU also made great use of the Wii U GamePad. It worked as your backpack and inventory, and you used it for things like the scanner and map. The game had the player looking away from the action on the TV down onto the GamePad, distracting you from anything that could sneak up.

The game is a real product of its time. Looking back, the graphics and atmosphere still hold up, but you can tell it was starting to show its age. 
In 2015, Ubisoft and Straight Right (an Australian porting company) got together and  brought the game to PC, PS4, and Xbox One. The ports have been negatively received however for all the horrible bugs that plague the game.

<figure class="article-figure"><img src="{{ site.baseurl }}/images/zombiucover.jpg" alt="ZombiU cover art"><figcaption>ZombiU launched with the Wii U in 2012.</figcaption></figure>

This isn't going to be a deep dive into ZombiU itself. Plenty of people have already made those videos. I want to get into the technical side and share some of the fun things I've learned over the past couple of years. For simplicity, I'll use ZombiU to refer to both versions unless said otherwise. (mainly because they are mostly the exact same)

## a little about the engine

ZombiU was made with Ubisoft's LyN engine. There isn't a ton of public documentation about LyN, but it powered ZombiU and a bunch of Just Dance games among other shit. LyN itself grew out of Ubisoft's previous 'Jade' engine. Jade has a lot more documentation, including material from a leak a few years ago along with a leaked build of Rayman 4 dating back to '05. LyN and Jade share a lot of the same basic file formats and ideas, so that was a nice help.

Also before it became a zombie game, ZombiU was a very different project about weird green gremlin dudes who vaguely resembled the Raving Rabbids. It was called 'Attack of the Killer Rabbids from Outer Space'. There is E3 footage of it, and you can see how some of the environments look a little like ZombiU concept art.

The game was showed off at E3 2011, but afterwards got a major rework. At some point, the team seems to have changed direction, brought on new people (such as Gabrielle Schrager and Antony Johnston), and turned it into the post-apocalyptic London game we got in 2012. The older version is why some of the code and files still refer to the game internally as 'Rabbids'.

## the game is split in two

Here is the first big thing that helped me understand the game. If you open the PC version, you'll see the usual executable, some DLLs, and a bunch of folders. ZOMBI has `ZOMBI.exe`, a cache folder containing compiled shaders, a data folder with the game assets, some dependencies, a support folder, and an AI native folder.

The executable is technically the engine, not the game-specific code. LyN builds the engine into the executable, while the game code is built as a separate library. When you launch `ZOMBI.exe`, it loads the game code library and the data folder, then starts the game.

<figure class="article-figure"><img src="{{ site.baseurl }}/images/zombipcfilehierachy.png" alt="The PC game folder and its main files"><figcaption>The PC folder contains the engine, game data, shader cache, and game code.</figcaption></figure>

That setup means Ubisoft could change game-specific logic without rebuilding the whole engine every time. It reminds me a bit of Source games, where `hl2.exe` provides the engine and loads a particular game's files and code. LyN has its own way of doing it, but the general idea is similar.

When `ZOMBI.exe` starts, it expects a command line. The game builds that command line itself, which I assume was some kind of Ubisoft security measure. It mostly ignores the command line you give it and constructs its own for some reason. The Japanese ZombiU demo disc includes the exact command line the game used, along with some other interesting leftovers.

<div class="article-gallery article-gallery--pair">
<figure><img src="{{ site.baseurl }}/images/japanesedemofiles.png" alt="Files left in the Japanese ZombiU demo"><figcaption>The Japanese demo disc has some especially useful development leftovers.</figcaption></figure>
<figure><img src="{{ site.baseurl }}/images/lyncommandlinecafe.png" alt="LyN Cafe command line file"><figcaption>The Wii U build includes a file with the launch arguments.</figcaption></figure>
</div>

The command line sets the project ID to `rabbids`, which was the game's codename. That's why the game code library is called Rabbids. It also points the engine at the data folder and sets things like VSync. The engine then takes those arguments and uses them to load the game.

## how the engine and game code talk

The engine and game code need a way to call into each other. ZOMBI ships a text file called `AIFunctionsMapping.txt`, and the engine uses it to connect the two sides.

The file maps IDs to engine functions. When the game code library loads, it builds a table of the functions it can call and matches them to those IDs. The engine passes that table to the library's startup function, which lets game code call back into the engine through the table.

I wrote a Ghidra script that reads `AIFunctionsMapping.txt` and uses those names to label the matching engine functions in the decompiled executable. It isn't the same as debug symbols, but it gets us a lot closer. The file even has comments explaining that developers should update it when they add engine functions.

<figure class="article-figure"><img src="{{ site.baseurl }}/images/AIFunctionsMapping.png" alt="AIFunctionsMapping.txt showing the engine AI function table"><figcaption>The file's header describes the function index mapping into LyN's AI ABI.</figcaption></figure>

<div class="article-gallery article-gallery--pair">
<figure><img src="{{ site.baseurl }}/images/rabbidsdll.png" alt="Rabbids game code library file"><figcaption>The game-specific code is loaded as a separate library.</figcaption></figure>
<figure><img src="{{ site.baseurl }}/images/ghidrascript.png" alt="Ghidra script that names engine functions from AIFunctionsMapping.txt"><figcaption>The Ghidra script uses the mapping file to label functions.</figcaption></figure>
</div>

## decompiling without source

Decompilation means looking at a compiled program's machine code and translating it back into something readable. The original ZombiU source code hasn't leaked. When developers compile source code, the compiler turns it into machine code and links the pieces into an executable or library. Compiler settings can optimize the result, which often makes reverse engineering harder.

A compiler can also produce a PDB file with debug symbols. Those symbols map machine code back to names like function names and structures. They aren't source code, but they're incredibly useful when trying to understand a program. ZOMBI didn't ship with its PDBs.

The Steam executable gave me another lucky break. Around 2020 or 2021, Ubisoft Connect replaced the old Ubisoft Game Launcher, and the change broke ZOMBI along with other Ubisoft games. Ubisoft eventually fixed it and pushed an update to Steam. That updated executable appears to have been compiled without optimizations. The machine code is still machine code, but without those optimizations it's more literal and easier to follow.

Just Dance 4 was a useful reference because it was also built on LyN and came out close to ZombiU. The Wii version released on October 9, 2012, about six weeks before ZombiU's North American launch on November 18. Its Wii ELF came with a `.sel` file containing symbol information. Think of the `.sel` as the Wii-side equivalent of a Windows PDB: it gave me function and symbol names that helped identify LyN systems. I used those names as a reference, not as a blueprint for ZombiU, since the games are still very different.

<figure class="article-figure"><img src="{{ site.baseurl }}/images/justdance4.png" alt="Just Dance 4"><figcaption>The Wii ELF's `.sel` file contains symbol information, similar in purpose to a Windows PDB.</figcaption></figure>

There are two broad approaches to decompilation. A functionally accurate decompilation recreates what the original program does, without requiring the new code to match line for line. A byte-for-byte decompilation aims to rebuild the exact same binary with the same compiler and settings.

I'm aiming for a functionally accurate reconstruction. The original game is an old 32-bit Windows game from 2012, and its renderer was built around DirectX 9. I've made some changes to get it running on modern hardware, but I keep comparing the results with the original to make sure the behavior stays faithful.

Ubisoft games using Jade or LyN weren't necessarily built on one universal engine shared by every team. Teams often copied the engine and developed a separate branch for their game. You can see evidence of that in debug logs and other information left in the game. The emulator logs include internal assertions tied to the ZombiU project, and opening the executable in Ghidra can reveal what compiler was used.

## navigation and UI: Pilot and FIRe

Two of the more interesting LyN systems are Pilot and FIRe. Pilot is the navigation middleware in ZOMBI's executable, version 3.18.01. The level data stores its navigation as `.nav` files. Those files contain triangle meshes split into layers, and Pilot uses them to find routes through a level. When a zombie needs to move somewhere, the gamecode sends a path request through LyN's navigation interface. Pilot locates the start and destination on the mesh, searches across triangles with A*, then turns that route into a smoother list of points for the zombie to follow.

<figure class="article-figure"><img src="{{ site.baseurl }}/images/navmeshatwork.png" alt="Debug view of Pilot navigation, with blue walkable triangles and a red path"><figcaption>Pilot's navigation in debug mode. Blue triangles mark walkable space, and the red line shows the path. Each upward jolt in the line marks a path recompute.</figcaption></figure>

The navigation isn't just one static mesh. The game also registers links for things that connect otherwise separate parts of a route, like doors, obstacles, climbs, and drops. Doors and other objects can enable or disable their links as the world changes, so a zombie's route can account for a door opening or a blockage. The data even has separate layers for different uses. The zombies use layer 0, while another layer carries sound propagation links. That's the broad shape of it: level-authored meshes and links feed into Pilot, and the game's AI asks LyN for paths through them.

FIRe is the other side of that same engine plumbing. It's Ubisoft's Flash player, built into the executable, and it runs Flash 8 movies using ActionScript 2. The HUD, menus, and GamePad screens are Flash movies rather than interface elements drawn directly by the gamecode. The files have their own names: `.feu` holds the movies, `.ffd` holds fonts, and `.fir` describes the movie players FIRe sets up. Inside the movies, ActionScript handles things like text, button behavior, and animation. The gamecode talks to them by sending named events with values, then the movie's scripts react to those events.

I could spend forever reverse engineering every part of FIRe, so I took the Ruffle source code, trimmed it down, and adapted it to the game's movies instead. That meant getting the movie timelines, text, fonts, and the bits of ActionScript ZOMBI actually uses working together. It was a big part of making the menus and HUD show up as they do in the original, and it gave me a much better look at how the game and engine communicate.

There are a lot of individual formats too. MOTION handles physics, COB handles collision, and TDT holds texture data. There are also formats for geometry, reflection data, and AI components. I'm still tracing what all of them do, but they give a good sense of how the engine is organized. The game's data archives use the `.bf` format, which stands for Big File. ZOMBI's PC archive extension is `.bfc`.

## the shaders were left behind

ZOMBI uses shaders to tell the graphics hardware how surfaces should look. Games often compile shaders for a player's specific GPU when the game starts. ZOMBI instead shipped with precompiled shaders in its cache folder, so players didn't need to compile them at launch.

The funny part is that the PC version also includes the raw shader files, even though the game doesn't load those files directly. They're just sitting in the game directory. The console releases don't include them. The only builds I've found with the raw shaders are the PC release and the Japanese demo.

The Japanese demo is even more interesting because it also includes the executable and batch files used to compile the shaders. They left the raw files and the tools used to build them in the demo.

<figure class="article-figure"><img src="{{ site.baseurl }}/images/rawshaders.png" alt="Raw shader files shipped with ZOMBI"><figcaption>The PC release includes raw shader files alongside its cache.</figcaption></figure>

Having the raw shaders is a huge help. I still had to trace how the renderer uses them, but I could use the shader code as a reference instead of trying to reconstruct all of that behavior from machine code. I rewrote them in a more platform-neutral form, and that's a big reason my version can look so close to the original.

## where the project is now

I'd say the decompilation is around 80 percent complete. There are still some mission-specific systems I haven't traced. Most missions work, but some don't. The Virucide sequence isn't implemented yet. That's the syringe you can use to kill a zombie while it's grabbing you.

The game builds and runs. Here are a few views from the current build:

<div class="article-gallery article-gallery--screens">
<figure><img src="{{ site.baseurl }}/images/zdmenu.png" alt="ZOMBI main menu running in the decompilation"><figcaption>Main menu</figcaption></figure>
<figure><img src="{{ site.baseurl }}/images/zdingame.png" alt="ZOMBI gameplay running in the decompilation"><figcaption>In game</figcaption></figure>
<figure><img src="{{ site.baseurl }}/images/zdoptions.png" alt="ZOMBI options screen running in the decompilation"><figcaption>Options</figcaption></figure>
</div>

You can play through the nightmare sequence, reach the safe house, and complete the first two missions. The game currently locks up in the doctor's scene where you're supposed to get the Virucide, because that part isn't working yet. Outside of that, a lot of the game already works.

Once the decompilation is finished, we could mod the game, add new modes, or even build weird multiplayer stuff if we wanted. Someone could port it to a web browser for no reason other than that it would be cool. This work is useful for preservation too. Other games built on LyN, including some Rabbids games, might benefit from the same approach. I've tried to keep the engine separate from ZombiU's game-specific code, so parts of it could potentially be reused.

I've also been using LLMs to help with some of the tedious parts. A lot of decompilation is careful, repetitive work, and the tools can speed that up. I'm fine with AI assistance when the person using it understands the work and checks the results. The problem is when people ask AI to build something huge, don't inspect what it produces, and ship code they don't understand.

That's part of why AI has such a bad reputation right now. People use it to make quick game recompilations, sometimes by gutting emulator code to get the graphics working. Those projects aren't necessarily fully native. The CPU code might run natively, while a lot of the GPU behavior is still handled through emulation. They can also violate software licenses along the way.

I'm not going to explain recompilation here. MattKC has a good [video about it](https://www.youtube.com/watch?v=lMGu6Ng_3yA).

I prefer decompilation because the result gives you code you can actually work with.

Anyway, the engine is mostly traced, and most of what's left is game code and some ugly systems I still need to clean up. There's a lot more to talk about, so expect more updates soon.

[check out the project on GitHub](https://github.com/flarglefingler/zombi-decomp) if you want to follow along.

bye.
