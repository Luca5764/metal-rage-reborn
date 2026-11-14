---
title: "Battles & multiplayer"
---

# Battles & multiplayer

[Project page](./) · [How to read these notes](how-to-read.html)

Notes on battle start and in-battle packets, written in November 2026 from my test implementation for the Taiwanese client (*鐵影特攻 Online*). Conventions, evidence labels and the DLL versions the addresses refer to are in [How to read these notes](how-to-read.html).

**Conclusions only: no server code, no raw logs, no network addresses.**

---

## 1. Battle start

### 1.1 Sequence (P2P: the room host's client hosts the match)

| # | Dir | Opcode | Name | To | Notes |
|---|---|---|---|---|---|
| 1 | C→S | `0x00222103` | host presses Start / F5 (`Game_Start_CN` by convention) | — | the server checks first, see 1.3 |
| 2 | S→C | `0x00420111` | `Game_Wait_SN`, empty body | every member | clears battle data, switches to scene 6 |
| 3 | S→C | `0x00222112` | `Game_User_SN` | every member | player table: team, mech, level |
| 4 | S→C | `0x00222111` | `Game_Info_SN` | every member | map, mode, time limit (+0x13, minutes) |
| 5 | S→C | `0x00222102` | `Game_Ready_SN` | every member | |
| 6 | S→C | `0x00222104` | `Game_Start_SN`, status 0 | every member | a non-zero status refuses the start (1.3) |
| 7 | S→C | `0x00420113` | `Ready_Host_SQ`, 6 bytes: u16 0, u32 0 | **host only** | the host opens its listen server |
| 8 | C→S | `0x00420114` | `Ready_Host_CA` (name by convention) | — | 8-byte body: +0x00 u16 0, +0x02 u32 result (0 = listening, `0xFFFFFFFF` = no listen object), +0x06 u16 the port it listens on. Built at `0x107d90f0` (result at `0x107d918b`, port at `0x107d91a4`). |
| 9 | S→C | `0x00420115` | `Ready_Host_SN` | each guest | the host's address and port (1.2) |
| 10 | S→C | `0x00420116` | `Ready_Success_SN` | each guest right after its `0x00420115`; the host after its own CA | fires `NETWORK_GAME_START`: the client travels |
| 11 | C→S | `0x00420117` | `Battle_Success_CN` | — | map loaded |
| 12 | C→S | `0x00230151` | `BeginRound_CN` | — | from the host |
| 13 | S→C | `0x00230152` | `BeginRound_SN` | everyone in the battle | |
| 14 | S→C | `0x00230104` | `Respawn_SN` | everyone in the battle | +0x0A UserIndex: spawns that player's mech |

- Steps 2–7 go out on timers in my implementation: +0 / +60 / +150 / +300 / +300 / +450 ms, plus a second `Game_Info_SN` at +600 ms. That is what works for me, not a client requirement I have measured.
- **Guests wait on the host.** Nothing tells a guest where to connect until the host's client answers step 7. If that answer never comes, or the server drops it, every guest stays on the loading/room screen while the host loads alone.
- Steps 2, 3, 4, 7, 13 and 14 are confirmed by disassembly and in-game tests. Steps 5, 6, 9 and 10 have their names from the client dispatcher. Steps 1 and 8 are named by convention. Step 8's layout is from its builder in the DLL and matches what clients send.

### 1.2 `Ready_Host_SN 0x00420115`: what guests get
- Body: +0x00 u16 port (copy it from the host's `Ready_Host_CA` +0x06, don't assume 30907), +0x02 u8 0, +0x03 ASCII string + NUL. Size the body to fit the string.
- **Put only the host's IPv4 address in the string.** The client builds the travel URL itself, as `<string>:<port>/<its own map>?team=…`, and the field is short. Tested in-game: with `<ip>/Map_PC04` in the field, a guest's client log showed the string cut after `/M` and the map load failed; with the bare address it travelled normally. An even longer string that my implementation cut off itself crashed clients.
- The address must be one the guests can reach. A host that logs in through the loopback address or `localhost` gives guests an address that points at themselves.

### 1.3 When the start can't go ahead
- **Refuse before anything moves:** send the host `Game_Start_SN 0x00222104` with body u16 status 1 at +0x00 and a u32 message key at +0x02, and send nothing else. The handler (`0x107eccb0`) passes the key to `Event_Call`, and the room page shows the text for that key from `ZNetwork.twt` as a popup. I use:
  - `0x00A30223`: all participating users must be ready before the game can start;
  - `0x00A30222`: not enough players are ready, so the game cannot start (I use it when one team is empty in a mode that needs both).
- The client does not check readiness itself: the host can press Start while a guest is not ready. The original server must have refused it, since that text exists in the client.
- **Abort after the start began:** `Ready_Failed_SN 0x00420112`, empty body. The handler is in the game dispatcher (scene 6 only; export thunk `0x10702ff9` → body `0x107d5930`). DLL reading: on a player client (`GIsClient`) it fires `NETWORK_GAME_LEAVE` with `HOST_NOTING` and changes to scene 5 (the room); on a dedicated host it ends the dedicated session and changes to scene 1. It fits the cases where the host's `Ready_Host_CA` is too short or reports a listen failure, or where you have no usable host address: guests would go back to the room instead of sitting on a dead loading screen. **I have not sent it to real clients yet**, so treat this as DLL reading only. Because it is scene-6 only, it does nothing for a client that is still on the room page, for example after a refused `Game_Start_SN`.
- Whatever you do on a failed start, put the room back into its "not playing" state.

## 2. In battle

**Battle CNs arrive on the host's connection.** In a same-machine two-client run, `BeginRound_CN 0x00230151`, `ChangeSlot_CN 0x00230101` and `Respawn_CN 0x00230103` all arrived on the host's game-server connection, while the guest's connection carried only keepalives `0x00020083`. Use the UserIndex in the body to tell whose action it is. I have only checked those three CNs in that run; that every battle CN works this way is a hypothesis.

| Finding | Evidence |
|---|---|
| `Death_CN 0x00230123`, 11 bytes LE: +0x00 attacker u16, +0x02 victim u16, +0x04 DeathType u8 (1–4, collapsed from a 26-way jump table), +0x05 Critical u8, +0x06 KillType u8, +0x07 KillWeapon u32 (an ItemIndex). Built at `0x107d9a05` / `0x107d9a1b`. Environmental deaths have attacker 0. | DLL. The last three field names come from the `Death_SN` handler's pushes (`0x107db6a2`–`0x107db6be`), which line up with the log format string at `0x1082be78`. One KillWeapon value matched a real weapon ItemIndex. How the kill-feed icon uses KillType / KillWeapon is unknown. |
| `Death_SN 0x00230124`: victim at +0x0C. I answer every `Death_CN` by sending it (body 0x51 bytes) to everyone in the battle. | DLL, tested in-game |
| `ChangeSlot_CN 0x00230101` (UserIndex u16, slot u8 1..8) → `ChangeSlot_SN 0x00230102` (+0x0A UserIndex, +0x0C slot) → `Respawn_CN 0x00230103` → `Respawn_SN`. In PvE this is the mech pick at start and after death. | DLL for the fields read; tested in-game. The original SN's exact length and +0x06..+0x09 are unknown. |
| `Assist_CN 0x00230121`, 7 bytes: +0x02 UserIndex, +0x04 type, +0x06 value. Sent as the damaged mech's remaining health crosses 20 % bands: just before a death I saw values 80 → 60 → 40 → 20 → 0. `Assist_SN 0x00230122`: +0x0A / +0x0C two UserIndex values plus two values. **Answer with the full 25-byte body.** The success path of the handler (`0x107d5fa0`) reads up to +0x18, past a 16-byte zero ACK. | DLL for the format; the 20 % meaning from UnrealScript and DLL |
| `Timeout_CN 0x00230111` is sent only in scene 6, by the host, while the game runs (about once a second). After the host's countdown reaches zero it is resent every tick. Use the first one after zero as "time up". | DLL, test log |
| **`Timeout_SN 0x00230112` (in battle) sets the team scoreboard.** Body 0x26 bytes. +0x02 u32 **must be 0**, or the client skips the whole packet. Two 14-byte records, red at +0x0A and blue at +0x18, each: Team u16, Score u16, Round u8, Alive u8, Try u16, Goal u16, Exp u32 (handler `0x107d7820`; gates at `0x107d7884` / `0x107d788f`). Team key: red 0, blue 1. **Send it after `Death_SN`**; sent before, the score gets reset to 0. The same opcode means something else outside battle. | DLL (record layout static, from the push order after `Log_Set 0x10703f4e`; Ghidra's argument order is wrong there); tested in-game: host and guest both showed 007:003. |
| The team keys follow the `Game_Info_SN 0x00222111` you send, not a fixed value: `Game_Info_Team_Set 0x1071a420` stores body+0x04 / +0x06, and `Game_Play_Start 0x10734050` copies them to where `Game_Score_Get` compares. Sending 0 / 1 is correct. The 1 / 2 values (`ClientRedIndex` / `ClientBlueIndex`) belong to the single-player tutorial (`execTutorial_Open 0x107301f0`). | DLL |
| `Game_Info_SN 0x00222111` body+0x0F: `0x107d4f94 cmp word [eax+0x1f],2` → `0x107d4fe3 sete` = IsQuater ("half"). Send 0 for team deathmatch, or the title shows "first half". | DLL; UnrealScript `ZScoreBoardTeamDM.uc:96-147`; tested in-game |
| `InstantRespawn_CN 0x00230105` → `InstantRespawn_SN 0x00230106` (PvE "instant respawn" item): after death the player picks a slot (`ChangeSlot_CN`), the client sends `InstantRespawn_CN`, and answering with `InstantRespawn_SN` respawns the mech at once. | Tested in-game once. Not a stability proof. |
| PvE lives: `Game_User_SN` record +0x64 is the item count the client adds to its 3 base lives (`0x10734495`). Lives refill at every round start; that is the stock `ZModePve.ModeReset_BD`. | DLL, UnrealScript; tested in-game |
