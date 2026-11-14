---
title: "How to read these notes"
---

# How to read these notes

[Project page](./)

Notes on how the Metal Rage Online client talks to a server, written in November 2026. They come from my test implementation for the Taiwanese client (*鐵影特攻 Online*) and are grouped by topic.

**Conclusions only: no server code, no raw logs, no network addresses.**

- Every item says what it rests on:
  - **DLL**: I read the disassembly; the address is given.
  - **UnrealScript**: decompiled `.uc` from the same client; `file:line` is given.
  - **tested in-game**: a real client reacted to the packet as described.
  - **test log**: seen in my own test packet logs only, not checked on screen.
  - **guess** / **candidate**: inference without direct evidence. Check before building on it.
- Anything not listed is unknown to me. A gap does not mean "confirmed absent".
- Addresses are VAs in the **stock Taiwanese `ZNetwork.dll`** (sha256 `6b07758edf57adcce7c96adeccc015d7a30bb72ba67eb01ad8b84e3451cfea1b`, ImageBase `0x10700000`) unless prefixed `Engine.dll` (stock sha256 `fc51fe1240ee34111fc1a483e74a1b131d4b69f2b2a0940adbb4a860a138d24e`). Export-table addresses are thunks; the body is at the `jmp` target.
- Opcodes are full 8-digit hex. Header big-endian, body little-endian, offsets are from the start of the body. S→C names come from the client dispatcher; C→S names come from the builder or are mine by convention, as noted.
- "My implementation" means my own test implementation. Timings I give are what works for me, not limits I measured in the client, unless the text says so.
- Scenes: 5 is the room page, 6 is the battle. The game dispatcher only handles packets in scene 6, and the same opcode can mean something else in another scene.
