---
title: "Using Godot as an Editor for OpenFight"
date: 2026-08-31
articles_tags: ["godot", "fighting", "open-source", "tools", "sdl"]
image: "/images/games/tecunhuman/openfight.jpg"
image_alt: "Screenshot of OpenFight"
summary: "How TecunHuman is using Godot as an editor, development environment, test harness, and runtime for OpenFight while keeping the original SDL game independent."
#external_url: "https://www.patreon.com/tecunhuman/posts/using-godot-as-168238881"
---

One of the problems I have wanted to solve with OpenFight for a while is making it easier to create content for the game.

[OpenFight](/games/tecunhuman/openfight/) is an older open-source fighting game project I originally built around SDL. The game works, but creating or replacing game assets can involve a lot of manual work. A fighting game character isn't just a sprite sheet: it has animations, moves, collision areas, hitboxes, timing information, and other configuration that all needs to work together.

This became particularly important because OpenFight still has some older graphics based on Street Fighter assets that I would eventually like to replace with assets that can be freely distributed with the project.

I could replace all of those assets manually, but that would be a fairly tedious process. More importantly, anyone wanting to create a new character would eventually run into the same problem.

What I really wanted was an editor.

Instead of building a custom game asset editor from scratch, I started looking at whether I could use Godot itself as the editor for OpenFight.

Godot already provides most of the pieces I would want from an authoring tool: an animation editor, visual scene editing, resource management, inspectors, scripting, collision shapes, and the ability to immediately run and test what I'm working on.

The idea is that I could eventually use Godot to create a character, configure its animations and moves visually, test it inside the game, and then generate the data that OpenFight needs.

But I didn't want that to mean rewriting OpenFight as a Godot game.

## Keeping OpenFight Independent

The existing SDL implementation still has value.

It is relatively lightweight, it already works, and keeping it around means that OpenFight doesn't have to depend on Godot in order to run.

So instead of porting the game to Godot, I started refactoring OpenFight so that the core functionality could live in a separate `libopenfight` library.

The goal is for the architecture to look something like this:

- `libopenfight` contains the reusable fighting game functionality.
- SDL provides the original standalone runtime.
- Godot provides another runtime as well as the development and asset-authoring environment.

This means Godot can become a powerful frontend and editor for OpenFight without replacing the original implementation.

Eventually, someone could potentially create a character using the Godot editor and then use that same content with either the Godot frontend or the standalone SDL version.

## Reusing What I Learned From Other Godot Libraries

Fortunately, I wasn't starting from zero.

Over the last few years, I have spent a lot of time integrating native libraries with Godot through projects like [godot-csound](/tools/tecunhuman/godot-csound/) and some of the other Godot libraries I've been working on.

Those projects required figuring out how to expose native C and C++ functionality to Godot, how to keep the underlying library separate from the Godot integration, and how to build and package that integration across different operating systems.

Most of that work has been related to audio, but the patterns aren't really specific to audio.

With godot-csound, for example, Csound remains its own independent library. Godot provides an integration layer around it, but Csound itself doesn't become dependent on Godot.

I was able to apply a similar idea to libopenfight.

Instead of wrapping an audio engine, this time I was exposing the functionality of a fighting game engine.

Because I had already solved many of those integration and build problems in previous projects, getting the initial OpenFight integration working was much easier than it would have been if this were my first native Godot library.

It's been interesting to see knowledge from the audio tooling carry over into something seemingly unrelated.

## Running OpenFight Through Godot

The initial implementation is now in place.

OpenFight currently runs through Godot on:

- Linux
- macOS
- Windows
- Web

The Web build is particularly useful because OpenFight can now run directly in a browser through Godot.

Since the integration is working across Godot's desktop and Web targets, adding Android and iOS also becomes possible. There will likely still be platform-specific work involved, but I no longer need to design an entirely separate mobile version of the engine.

At the same time, none of this removes the SDL version.

The same core `libopenfight` functionality can continue to be used without Godot.

## Godot as an OpenFight Asset Editor

Getting the game running inside Godot was really the foundation for the part I'm most interested in next: tooling.

The immediate goal is to start building a simple character editor.

Instead of manually editing all of the information associated with a character, I would like to use Godot's editor to visually define things such as:

- sprite animations
- collision areas
- hitboxes
- moves
- animation timing
- character properties

The character could then be tested immediately inside OpenFight without leaving the editor.

That would make replacing the existing placeholder graphics much more practical, but it would also provide a workflow for creating entirely new characters in the future.

Rather than solving the asset replacement problem once, I can build tooling that makes the same problem easier to solve every time.

Eventually, I could imagine a workflow where someone creates and tests an OpenFight character entirely through Godot and then exports the resulting character data.

Godot would essentially become an OpenFight SDK.

## Why Not Just Rewrite OpenFight in Godot?

Rewriting the whole project in Godot would probably be the more conventional approach.

But I don't think it would necessarily be the more interesting one.

There is already a working native engine. Throwing that away would mean losing some of the portability and independence that comes from having a small standalone library and SDL frontend.

By separating the engine from its frontends, I can use whichever environment makes the most sense.

Godot can provide the rich development environment.

SDL can provide a lightweight native runtime.

And libopenfight can remain independent from either one.

This also means the investment in building better Godot tooling doesn't necessarily lock the resulting game assets to Godot.

## Using Godot for Gameplay Testing

Another possibility that becomes available through the Godot integration is using scenes as repeatable gameplay tests.

Fighting games can be difficult to test manually because so much depends on timing. If I want to verify how two moves interact, I normally have to control both characters correctly and reproduce the same sequence of inputs each time.

Instead, I could create Godot scenes where the character inputs are scripted.

For example, one character could move forward, perform a specific attack, wait a known number of frames, and then perform another action. The opposing character could have its own predetermined sequence.

That would let me make changes to a character, run the scene again, and observe the same interaction under the same conditions.

It could be useful for testing things such as hit detection, move timing, blocking, recovery, spacing, and other gameplay mechanics without requiring me to reproduce the inputs perfectly by hand every time.

These scenes could also become useful regression tests.

If I fix a bug involving a particular interaction between two characters, I could keep a scene that reproduces that situation. Later changes to libopenfight could then be tested against the same scenario.

Because OpenFight is now integrated with Godot, I could also take advantage of testing frameworks and automation tools that already exist in the Godot ecosystem rather than building an entire testing environment specifically for OpenFight.

That starts to make Godot useful for more than just editing assets.

It can become a development environment where I create characters, test them interactively, reproduce gameplay situations, and eventually automate parts of the verification process while the actual fighting game logic continues to live in libopenfight.

## What's Next

This work is still very much in progress.

So far, the goal has been to establish the architecture and prove that libopenfight can successfully run through Godot across multiple platforms.

With Linux, macOS, Windows, and Web working, that initial foundation is now there.

The next step is to actually take advantage of the Godot editor.

I'll be starting with some simple character-authoring tools and using them to work toward replacing the older placeholder graphics in OpenFight.

From there, I can continue expanding the workflow and see how much of the process of creating a fighting game character can be moved into a visual editor.

I also want to explore how useful scripted scenes and automated tests can become for validating gameplay mechanics as the project evolves.

The interesting part for me is that Godot doesn't have to become the engine in order to provide value to the project.

It can be the editor, the development environment, the test harness, and one possible runtime while OpenFight remains OpenFight.

You can find the OpenFight source code on GitHub:

[OpenFight on GitHub](https://github.com/nonameentername/openfight)

You can also try the current Web build here:

[Play in the browser](https://nonameentername.github.io/openfight)

[Originally published on Patreon](https://www.patreon.com/tecunhuman/posts/using-godot-as-168238881)
