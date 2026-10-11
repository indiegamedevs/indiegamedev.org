---
title: "Minecraft Java mods: check the installed JAR, not only the dev client"
date: 2026-10-12
summary: "A developer checklist for separating build, loader, resource and packaged-JAR failures before sharing a Minecraft Java mod."
tutorial_studios: ["universal-modder-field-guide"]
tutorial_tags: ["minecraft", "java", "modding", "testing", "troubleshooting"]
---

A mod that appears in an IDE's development client has passed one useful check. Before distributing it, also test the artifact another player will install. This checklist helps identify the first failing stage, instead of changing several dependencies at once.

It is intended for Minecraft **Java Edition**. Use the documentation for the exact game and loader version being tested. NeoForge 1.21.1, Forge and Fabric are distinct targets; a loader or file-extension change does not make their mod artifacts interchangeable.

## 1. Record the environment

Write down the operating system, Minecraft version, loader and loader version, launcher instance, required dependencies, and output JAR name. Check both the build tool's Java runtime and the launcher's runtime.

For a NeoForge **1.21.1** workspace, the [versioned official setup guide](https://docs.neoforged.net/docs/1.21.1/gettingstarted/) specifies a 64-bit JDK/JVM 21. This is a version-specific requirement, not a recommendation to upgrade every older modpack to that runtime.

From the development workspace, capture the selected JVM and first build failure:

```sh
./gradlew --version
./gradlew build --stacktrace
```

On Windows PowerShell, use `.\gradlew.bat` in place of `./gradlew`.

## 2. Identify the failing stage

| Observation | First check |
|---|---|
| Build fails before Minecraft starts | First compiler, dependency-download or Gradle error. |
| Loader refuses to start | Exact missing dependency, incompatible version or duplicate mod ID from the loader message. |
| Menu opens but the feature is absent | Whether the mod is loaded, then registration and the intended feature test. |
| Feature exists but looks wrong | Model/texture references, resource namespace and file case. |
| Dev client succeeds but installed mod fails | Contents and identity of the packaged JAR, plus the launcher instance's environment. |

Keep tests in a separate instance and new world. Start with the matching loader, then the new mod and its required dependencies, so unrelated mods do not obscure the first failure.

## 3. Inspect the shipped artifact

Select the main mod JAR under `build/libs/`. A sources or Javadoc archive is not the playable mod artifact. Open the chosen JAR with an archive viewer and check that it contains compiled code, loader metadata and the assets/data expected by the mod.

Copy that exact file into the selected instance's `mods/` folder. Restart the instance and repeat the feature checks. The official NeoForge guide distinguishes `build` output from the `runClient` development environment; neither should be substituted for the other in a release record.

## 4. Preserve useful evidence

Read the selected instance's `logs/latest.log`, or the development run's own log, and a crash report when one was written. A launcher failure can occur before a Minecraft log exists. [Fabric's crash-report guide](https://docs.fabricmc.net/players/troubleshooting/crash-reports) explains where reports live and why the selected game directory matters with third-party launchers.

Record the first relevant error and its cause chain, the last change, and the smallest reproduction. Remove private paths, account identifiers and secrets before sharing logs publicly. Change one thing, then reproduce the same test.

## Further reading and scope

[The expanded Minecraft mod-loading guide](https://universalmodder.dev/guides/minecraft-mod-not-loading/) adds Java-error examples, a versioned item/recipe/resource checklist and a structured troubleshooting prompt.

Affiliation: this contribution comes from the maintainer of **Universal Modder Field Guide**, an independent guide publisher. This tutorial uses AI-assisted writing and source review. It is not official Minecraft or loader documentation and is not a report of a performed compilation or Minecraft play-test. Mark an unperformed check as unverified rather than treating documentation review as runtime evidence.
