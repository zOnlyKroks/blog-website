---
layout: post
title: "Why Lunar Client and PvP Clients Are a Bad Idea"
date: 2026-03-06
tags: [minecraft, security, gaming]
---

If you have spent any time in the Minecraft PvP scene, you have almost certainly come across Lunar Client, Badlion Client, or one of the many other clients that promise better FPS, cosmetics, and a competitive edge. They are slick, they are popular, and millions of players use them without a second thought. This post is about why that is a mistake.

## What Are PvP Clients?

PvP clients are third-party Minecraft launchers that bundle mods into a polished package: keystroke displays, armor HUDs, FPS boosters, mini-maps. The pitch is simple, download one thing and get everything, no fiddling with Forge or Fabric required.

## You Cannot See What They Do

The biggest issue is that these clients are closed source. That means the code they run on your machine is not publicly readable. You have no way of knowing what they actually do beyond what the company tells you.

This matters more than most people realise. Every time you launch Lunar Client you are running software from a company you have no real relationship with, and you have no way to verify it does only what they claim. You are taking their word for it, every single session.

Compare that to Fabric mods, where every mod is typically open source. Anyone can read the code, which means the community constantly scrutinises it. If a mod did something suspicious, someone would notice.

## Telemetry and Data Collection

Closed-source clients collect data about you, and the exact scope of that collection is not something you can verify. Their privacy policies exist, but you cannot confirm the software actually behaves as described, because you cannot read it.

## The "But It Is Whitelisted on Servers" Argument

A common defence is that major servers like Hypixel explicitly support Lunar Client. If big servers trust it, why shouldn't you?

Server whitelisting only means one thing: the server has verified the client does not give players an unfair in-game advantage. That is all. Servers do not review what the software does to your computer, and they have no reason to. That is your problem, not theirs.

## When It Breaks, You Are on Your Own

This is something most players do not think about until they are already stuck.

When Minecraft crashes or behaves weirdly on a normal Fabric setup, the crash report is readable. You can search the error online, post it in a community Discord, or ask a mod developer for help. Someone can usually figure out what went wrong.

With a PvP client, the crash report contains chunks of code with scrambled, unreadable names that belong to the client itself. Nobody outside the company can tell you what those mean. The client developers typically will not help either. You end up with a broken game and no way to diagnose it.

## Mods Break in Ways You Cannot Fix

PvP clients modify the same parts of Minecraft that mods do. The problem is they do it without any coordination with the broader modding community, and without being transparent about what they are touching.

The result is that mods that work perfectly fine on their own can break completely when combined with a PvP client. Not because the mod is bad, but because the client quietly changed something the mod depended on. You will not get a useful error message explaining this. You will just get a crash, or worse, things silently not working as expected.

In the open modding ecosystem this kind of conflict can actually be fixed, because everyone can see what everyone else is doing. With a closed-source client there is no fixing it. You can only wait and hope the client pushes an update, which may or may not ever come.

## What to Use Instead

Everything PvP clients offer exists as individual open-source Fabric mods. Sodium[^1] alone will give you better FPS than any PvP client's built-in optimisations. Iris handles shaders, Lithium handles further performance improvements, and there are HUD mods for every piece of information you could want on screen. A comprehensive list of alternatives covering performance, visuals, and quality of life is maintained at [optifine.alternatives.lambdaurora.dev](https://optifine.alternatives.lambdaurora.dev).

If you want a ready-made starting point rather than assembling mods yourself, [Fabulously Optimized](https://modrinth.com/modpack/fabulously-optimized) is a well-maintained, battle-tested Fabric modpack that covers performance and quality-of-life essentials out of the box. It will not replicate every PvP-specific feature a dedicated client offers, but for most players it covers the things that actually matter.

It takes a bit more setup than downloading one installer, but the result is a setup you actually understand and control.

## The Short Version

The core problem with PvP clients is not any single feature they have or lack. It is that you are running software you cannot inspect, that modifies your game in ways nobody outside the company can see, and when something goes wrong you have nowhere to turn. The open modding ecosystem is built on transparency. PvP clients opt out of that entirely.

---

[^1]: Sodium source code: [https://github.com/CaffeineMC/sodium](https://github.com/CaffeineMC/sodium)
