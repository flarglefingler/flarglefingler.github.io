---
layout: post
title: ZOMBI Decompilation
description: we decompilin' zombi
tags: zombi github zombiu
---

## Intro

ZombiU is a survival horror game released in 2012 as a launch title for the Wii U. The disaster that was the Wii U is something we can talk about in another blog post, but for now, I want to focus on ZombiU.

![ZombiU cover(../images/zombiucover.jpg)

It was... quite a departure from what Ubisoft was known for making back then.

Set in post-apocalyptic London in 2012, you assume control of a survivor caught in the middle of the "Blight", an infection that turns your average British citizen into... an even more British citizen. You must survive the drooling dead while following the instructions of "The Prepper", a mysterious man whose goal is to keep you alive for as long as possible.

One of ZombiU's flagship features was its **Rebirth system**. If you died, you wouldn't simply respawn at a checkpoint. Instead, you'd return as an entirely different survivor (well, the first three aren't random), complete with a randomized name, gender, age, and profession. None of these attributes really matter, but it's a nice touch.

You'd then have to backtrack to where you died and, depending on whether your previous survivor was infected, kill them and recover your belongings.

The game was generally well-received, particularly for its use of the Wii U GamePad. You could use it to manage your inventory, read notes, scan your surroundings, and perform various other actions. Doing so would leave your character vulnerable on the TV, forcing you to constantly divide your attention between two screens. On the other hand, the game was criticized for its repetitive melee combat and somewhat lackluster story, but I digress.

Anyways, in the year of our lord 2015, Ubisoft decided to bring the game to other platforms, with Straight Right handling the porting work. Considering that ZombiU had been exclusive to a console struggling against the Xbox One and PS4, this was a pretty reasonable decision.

Unfortunately, the ports were... questionable.

The GamePad mechanics were effectively gutted, with everything being condensed onto a single screen. The multiplayer mode was also removed, and a plethora of bugs were introduced. On the bright side, they added two additional melee weapons, compared to the single cricket bat available in the original Wii U version. So, uh, there's that.

While the Wii U version was praised for its unique mechanics, the ports (particularly the PC release) became notorious for their bugs, glitches, and the ever-wonderful Ubisoft launcher.

Ubisoft managed to fuck things up even further during the transition from Uplay to Ubisoft Connect. They eventually released a haphazard patch for the PC version that fixed some of the launcher-related headaches, meaning you could finally play the game without disconnecting from the internet.

Unfortunately, this update also broke the game's sound-cache system. The first time certain sound effects are triggered, they simply don't play. As a result, a good chunk of the game's audio can go missing during a playthrough.

Wonderful.

## Where That Brings Us Now

ZombiU has always fascinated me, particularly because of the technology behind it.

The game runs on Ubisoft's **LyN engine**, which evolved from their previous engine, Jade. Unlike LyN, Jade has some documentation and a fair bit of publicly available information, making it a useful starting point for understanding its successor.

A few months ago, I decided I wanted to attempt a decompilation of ZOMBI.

I wasn't going in completely blind. I'd already spent some time researching the game's file formats and had previously written a [Python-based archive manager](https://github.com/flarglefingler/ZOMBI-Manager) capable of opening and extracting the game's archives.

Eventually, I decided to take a closer look at the executable itself.

And, to my surprise, Ubisoft had left behind some pretty interesting stuff.

## Scratching the Surface

### The Engine and Game DLL

The game's executable is `ZOMBI.exe`. It's what you launch to actually play the game (duh).

What many people might not know, however, is that LyN works a little differently from what you'd expect. In some ways, its architecture reminds me of Valve's Source Engine.

With Source, games such as Half-Life 2 and Counter-Strike: Source can share the same executable, `hl2.exe`. The `-game` command-line argument tells the engine which game's directory and configuration to load.

LyN follows a somewhat similar principle, although its implementation is different.

**`ZOMBI.exe` is essentially the engine itself.**

Rather than containing all of the game's logic in one executable, LyN separates much of the game-specific code into another binary. In ZOMBI's case, the executable uses a command-line argument to identify the game DLL it needs to load.

That DLL is called `Rabbids.win32.f.dll`. On other platforms, such as the PS4, its equivalent is named `Rabbids.orbis.f.prx`.

So, why is a game about surviving a zombie apocalypse loading something called *Rabbids*?

Well, that's where things get interesting.

### Ubisoft's AI System

When developing games using Jade or LyN, Ubisoft would frequently fork and modify the engine for individual projects, adding whatever functionality they needed.

You can find evidence of this in various Ubisoft games, including assert strings and debugging information left behind in titles such as Just Dance 4, which shipped with debug symbols.

But the actual game-specific logic was handled somewhat differently.

Ubisoft used its own proprietary scripting system, referred to internally as **AI** (not to be confused with Artificial Intelligence). Parts of this technology have surfaced in games such as Rabbids Go Home.

Instead of interpreting all of this logic at runtime, Ubisoft could convert the AI code into C++ using an internal tool referred to as `AI2Cpp`.

The resulting C++ code would then be compiled into the game's DLL, which the engine loads at runtime.

In other words, a significant portion of the game's actual behavior lives inside `Rabbids.win32.f.dll`, while `ZOMBI.exe` handles much of the underlying engine functionality.

Ever noticed a file named `AiFunctionsMapping.txt` in the game's directory?

Well, it isn't just some random leftover.

The file contains mappings between identifiers and engine-exposed functions. These mappings are part of the interface that allows the game-specific code to communicate with the engine.

Understanding this relationship is incredibly useful when trying to reconstruct how the game works.

### Compiler Optimizations (or Lack Thereof)

There was another interesting discovery hiding inside the executable.

**Large portions of ZOMBI's executable appear to have been compiled with little to no optimization.**

For anyone unfamiliar with reverse engineering, this is a pretty big deal.

When decompiling a game, one of the best things you can possibly have is access to its debug symbols.

On Windows, these are commonly stored in `.pdb` files, which can contain function names, type information, source-file references, and other debugging metadata.

Having the original PDBs can turn an otherwise miserable reverse-engineering project into something considerably more manageable.

Unfortunately, ZOMBI didn't ship with them.

Ubisoft was smart enough to keep those to themselves.

But there's another factor that can make decompilation easier: **compiler optimizations**.

Normally, when compiling a program, the compiler can perform various transformations to make the resulting machine code faster or smaller. Functions might get inlined, loops might be rearranged, variables might disappear entirely, and the final assembly can end up looking almost nothing like the original source code.

For reverse engineering, this can be an absolute nightmare.

An unoptimized build, however, tends to preserve much more of the original program's structure. Functions are often easier to identify, control flow is more straightforward, and reconstructing the original logic becomes considerably less painful.

And in ZOMBI's case, much of the executable is surprisingly friendly in that regard.

That's not to say decompiling it is suddenly effortless. There's still a ton of work involved, especially without debug symbols.

But compared to a heavily optimized release build, I'll gladly take what Ubisoft gave us.

### A Shader Goldmine

Ubisoft also left behind something else that was incredibly useful.

To explain why, we need to talk briefly about 3D rendering.

Modern rendering pipelines make extensive use of **shaders**, small programs executed on the GPU that control various stages of rendering. They're responsible for things like transforming vertices, calculating lighting, sampling textures, and producing the final appearance of rendered surfaces.

Normally, games ship with shaders that have already been compiled into a format suitable for their target graphics API or hardware.

This avoids having to compile everything from scratch during gameplay, which can introduce loading delays or stuttering.

ZOMBI is no exception.

In the game's directory, you'll find a `Cache` folder containing precompiled shader data used by the renderer.

Nothing particularly unusual there.

However, Ubisoft made one rather glorious mistake.

**They also shipped the original, uncompiled shader source files.**

These files don't appear to be referenced by the executable or game DLL during normal operation. They're simply sitting there, alongside the compiled shader cache.

For someone trying to reconstruct the rendering pipeline, this is an absolute goldmine.

Instead of having to reverse engineer every shader from its compiled representation, we can examine the original source, understand the intended behavior, and adapt it to other graphics APIs.

It doesn't magically reverse engineer the entire renderer, but it eliminates a significant amount of guesswork.

Thanks, Ubisoft!

## Pulling the Trigger

With all of this information gathered, I finally decided to go ahead with the project.

Recently, there's been an absolutely monstrous increase in AI-assisted decompilation and recompilation projects.

I'm not going to open that particular can of worms here. I have my own opinions about the trend (and I'm not particularly fond of either approach, especially the latter), but I figured there was an opportunity to make use of these tools for something I actually cared about.

So, I decided to shill out a "few bucks" and bought Claude's highest subscription tier.

My original plan was fairly straightforward.

I'd handle the engine-related work myself, focusing primarily on `ZOMBI.exe`, while using Claude to help reconstruct the game-specific code inside `Rabbids.win32.f.dll`.

The reasoning was simple: unlike the executable, the game DLL was optimized, making its original logic considerably more difficult to recover.

Rather than spend an eternity manually untangling the entire thing, I wanted to see how much of the process could be accelerated while still keeping the resulting code understandable and maintainable.

And with that, the project officially began.

## Introducing... zombi-decomp

Introducing **zombi-decomp**, my attempt to reconstruct the latest Steam release of ZOMBI.

Progress so far has been surprisingly good.

Since I currently only have access to a macOS machine, one of the first priorities was getting the game's rendering code into a state where it could actually run on my hardware.

With Claude's assistance, I began converting the original shaders to **Metal Shading Language (MSL)**, allowing them to work with Apple's Metal graphics API.

I also wanted to avoid tying the project too closely to a single platform.

To make eventual Windows and Linux support less painful, I picked out a few cross-platform libraries early on:

- **SDL3** for window management and input.
- **FFmpeg**, using a slimmed-down configuration, for supported media playback, including Bink 1 content.
- **Metal** as the initial rendering backend for macOS, with room for other backends in the future.

The idea is to keep the core game and engine logic as platform-independent as reasonably possible.

### What Kind of Decompilation Is This?

Currently, I'd estimate that **roughly 50% of the game has been decompiled or reconstructed**.

I want to stress something here, though.

**This project is not intended to be a byte-for-byte decompilation.**

In traditional matching decompilation projects, the goal is often to reconstruct source code that, when compiled with the original toolchain, produces machine code identical (or as close as possible) to the original binary.

That's not what I'm aiming for.

Instead, my goal is a **functionally accurate reconstruction**.

As long as the resulting code behaves like the original game, I'm happy. I don't particularly care whether a function has the exact same structure, variable names, or assembly output as Ubisoft's original implementation.

What matters is preserving the behavior of the game while making its underlying code understandable, maintainable, and portable.

And, ideally, fixing some of the rather unfortunate issues Ubisoft left behind along the way.

### Some Screenshots

Here's the game running so far:

![ZOMBI main menu](../images/zdmenu.png)

![ZOMBI in-game screenshot](../images/zdingame.png)

There's still plenty of work left to do, but progress has been going nicely. Once decompilation is "finished", I plan to put all my current work in a branch simply titled "pc", and then start a new main branch called "enhanced" which I'll put a plethora of bug fixes, features, and more.
If everything continues at the current pace, I'm hoping to have the full game playable within the next month or so. But shit can change, so who knows 🤷. Stay tuned for more.
