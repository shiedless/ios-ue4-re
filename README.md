<h1 align="center">ios-ue4-re</h1>

<p align="center">reverse engineering and building read-only overlays for Unreal Engine 4 games on iOS (arm64 / arm64e)</p>

<p align="center">
  <img src="https://img.shields.io/badge/engine-Unreal%20Engine%204-C7192E?style=for-the-badge" alt="engine">
  <img src="https://img.shields.io/badge/platform-iOS%20arm64%20%2F%20arm64e-000000?style=for-the-badge" alt="platform">
  <img src="https://img.shields.io/badge/scope-read--only%20%2F%20analysis-1f6feb?style=for-the-badge" alt="scope">
  <img src="https://img.shields.io/badge/license-MIT-2ea043?style=for-the-badge" alt="license">
</p>

---

Notes and a working example for reverse engineering and building overlays for
Unreal Engine 4 games on iOS. This is the stuff I kept re-explaining to people,
collected in one place, plus a small tweak that puts all of it together so you can
see it actually run instead of just reading about it.

> **Read-only, analysis-and-visualization oriented.** No memory writes into the
> game, no netcode tampering. If you want to learn how a UE4 iOS overlay is put
> together, start here.

---

## the example

[**Reveal**](https://github.com/shiedless/Reveal) is a ~500-line skeleton ESP. It
reads `GWorld`, walks the actors, pulls each character's bone pose straight out of
the skeletal mesh, projects the joints to screen and draws a figure. That is the
whole thing — deliberately tiny so the data path is legible end to end, and the
concrete version of every note below.

```mermaid
flowchart LR
    gw["GWorld"] --> actors["walk actors"]
    actors --> pose["bone pose<br/>from skeletal mesh"]
    pose --> proj["project joints<br/>to screen"]
    proj --> draw["draw the figure"]

    style gw fill:#C7192E,color:#fff
    style draw fill:#1f6feb,color:#fff
```

---

## the notes

Read them in this order if you're new — each one leans on the last.

```mermaid
flowchart TD
    n1["1 · ios-binary-re-notes<br/>decrypt · slice · load · read"]
    n2["2 · xor-string-deobf-notes<br/>break the string obfuscation"]
    n3["3 · arm64-ios-inline-hook-notes<br/>W^X · reloc · icache · PAC"]
    n4["4 · ue4-ios-gworld-gnames-notes<br/>the two globals + anti-tamper"]
    n5["5 · ue4-ios-fname-notes<br/>FName index -> string"]
    n6["6 · ue4-ios-processevent-notes<br/>the UFunction call funnel"]
    n7["7 · tencent-ace-anogs-notes<br/>ACE, and why naive attacks fail"]

    n1 --> n2 --> n4 --> n5 --> n6
    n3 -.->|needed to hook at all| n6
    n4 --> n7

    style n1 fill:#000,color:#fff
    style n4 fill:#C7192E,color:#fff
    style n7 fill:#222,color:#fff
```

| # | note | what it covers |
|---|------|----------------|
| 1 | [**ios-binary-re-notes**](https://github.com/shiedless/ios-binary-re-notes) | getting a decrypted binary, picking the right slice, loading it, and reading an iOS app at all — the groundwork |
| 2 | [**xor-string-deobf-notes**](https://github.com/shiedless/xor-string-deobf-notes) | breaking the XOR string obfuscation you hit the moment the app hides anything |
| 3 | [**arm64-ios-inline-hook-notes**](https://github.com/shiedless/arm64-ios-inline-hook-notes) | inline hooks by hand — W^X, instruction relocation, the icache, PAC — the four things that crash you |
| 4 | [**ue4-ios-gworld-gnames-notes**](https://github.com/shiedless/ue4-ios-gworld-gnames-notes) | finding the two globals everything hangs off, and the anti-tamper traps around them |
| 5 | [**ue4-ios-fname-notes**](https://github.com/shiedless/ue4-ios-fname-notes) | turning an FName index into a string, which you need to match anything by name |
| 6 | [**ue4-ios-processevent-notes**](https://github.com/shiedless/ue4-ios-processevent-notes) | finding `ProcessEvent`, the funnel every UFunction call goes through |
| 7 | [**tencent-ace-anogs-notes**](https://github.com/shiedless/tencent-ace-anogs-notes) | how Tencent's ACE anti-cheat is built and why the naive attacks on it fail — analysis, not a bypass |

---

## what you need

| | |
|---|---|
| **device** | a jailbroken iOS device (or a solid emulator/sim setup) to run and dump on |
| **build** | [Theos](https://theos.dev) for building tweaks |
| **static** | IDA, Ghidra or Binary Ninja |
| **background** | a rough grip on ARM64 assembly and C/C++ |

> None of the notes assume Windows-only tooling, which most UE RE guides quietly do.

---

## how the pieces fit

The short version of the whole workflow:

Get a decrypted binary (`ios-binary-re-notes`). Load it, skim the Objective-C
metadata and strings to map the surface. If the app hides its strings, break the XOR
(`xor-string-deobf-notes`). Find `GNames` and `GWorld`
(`ue4-ios-gworld-gnames-notes`), then you can resolve names (`ue4-ios-fname-notes`)
and walk the live scene. From there an overlay is just reading the world and drawing
it — which is what Reveal does. If you need to react to game events instead of just
reading state, hook `ProcessEvent` (`ue4-ios-processevent-notes`), and if you need to
hook code at all, do it without crashing (`arm64-ios-inline-hook-notes`). If the game
has ACE, know what you're walking into (`tencent-ace-anogs-notes`).

```mermaid
flowchart LR
    dec["decrypted binary"] --> map["map the surface<br/>objc metadata + strings"]
    map -->|hidden strings| xor["break XOR"]
    map --> glob["find GWorld / GNames"]
    xor --> glob
    glob --> name["resolve FName -> string"]
    name --> walk["walk the live scene"]
    walk --> overlay["read + draw<br/>(Reveal)"]
    walk -.->|react to events| pe["hook ProcessEvent"]
    pe -.->|hook safely| hook["inline hook: W^X · PAC"]

    style glob fill:#C7192E,color:#fff
    style overlay fill:#1f6feb,color:#fff
```

---

## a note on intent

This is for learning how these systems work and building read-only tools on top of
them. The anti-cheat writeup in particular stops at understanding on purpose. Do what
you want with the knowledge, but the repos themselves don't ship attacks.

---

## beyond ue4

Same approach, other engines and tools:

| repo | what it covers |
|------|----------------|
| [**unity-il2cpp-esp-tutorial**](https://github.com/shiedless/unity-il2cpp-esp-tutorial) | the unity il2cpp side — a native objc overlay from `dump.cs` to screen, plain uikit |
| [**roblox-ios-luau-vm-notes**](https://github.com/shiedless/roblox-ios-luau-vm-notes) | reversing roblox's luau runtime on iOS: functions, anchors, `lua_State` layout |
| [**ios-messiah-re**](https://github.com/shiedless/ios-messiah-re) | netease's messiah engine: an ecs with no `GWorld`, a python gameplay layer, telemetry anti-cheat |
| [**ida-pro-guide**](https://github.com/shiedless/ida-pro-guide) | getting around a binary in IDA — useful for every note above |

---

## questions and corrections

- **questions, ideas, "how did you find X"** → [Discussions](https://github.com/shiedless/ios-ue4-re/discussions)
- **something in a note is wrong or outdated** → [open a correction](https://github.com/shiedless/ios-ue4-re/issues/new?template=correction.yml) — there's a dropdown for which note

If the notes saved you time, a ⭐ on the repo helps other people find them.

---

## license

MIT across the board.

---

<p align="center">— shiedless</p>
