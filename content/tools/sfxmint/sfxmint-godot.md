---
title: "SFXMint for Godot"
date: 2026-11-15
tool_tags: ["godot", "audio", "sound-effects", "plugins", "editor", "gdscript", "open-source"]
external_url: "https://github.com/sfxmint/sfxmint-godot"
summary: "An MIT-licensed Godot 4.6+ editor dock for searching, previewing and importing CC0 sound effects."
image: "/images/tools/sfxmint/sfxmint-godot.jpg"
image_alt: "Screenshot of the SFXMint dock in the Godot editor"
---

SFXMint for Godot is an open-source (MIT) editor plugin that adds a bottom dock to the Godot editor. You describe a sound, filter by category and duration, listen to results, and import MP3 files into `res://audio/sfxmint/` without leaving the editor.

Each import folder also gets a `sounds.json` manifest and a `SOUNDS.md` section recording the file's duration, SHA-256 and license, which is a practical pattern for keeping track of where game audio came from. The game itself plays local files and does not need the plugin, an account or a network connection.

Things to know before using it: the library sounds are AI-generated and CC0 1.0; searching and MP3 imports are free with daily limits (20 searches and 5 sounds a day without a key), and WAV imports need a paid download pack.

[Releases](https://github.com/sfxmint/sfxmint-godot/releases/latest)
