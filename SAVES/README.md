# XDOD native saves — schema 1

`ZSAVE` overwrites `XDOD.SAV` on device 8 in its current directory. `ZLOAD`
restores that file. An optional one-to-eight-digit argument selects
XDOD-NUMBER.SAV. Leading zeroes are preserved. An optional `F` after the
slot, or by itself for the default file, skips the tape sound and its delay.
The original unique prefix parser also accepts `ZS` / `ZL`.

On emulator r49 HostFS, the current directory is determined by the emulator's
working directory or its `-fsroot` / `-startin` options. The project's Run workflow
starts the emulator in the project directory. On an SD card, use the current
directory on device 8. See the [official emulator HostFS documentation](https://github.com/X16Community/x16-emulator/blob/master/README.md#host-filesystem-interface).

The file is **2,745 bytes**, with no PRG load-address prefix. All words and
pointers are little endian. It contains logical game state, not an emulator/CPU
snapshot: no return stack, IRQ vectors, KERNAL buffers, framebuffer, audio sample
banks or open channels are restored.

## Header

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | 4 | ASCII `XDOD` |
| 4 | 1 | Schema version: 1 |
| 5 | 1 | Reserved, zero |
| 6 | 2 | Payload length: 2,729 |
| 8 | 2 | Object-table address in the saving build |
| 10 | 4 | Reserved, zero |
| 14 | 2 | CRC16/CCITT-FALSE, low byte first |

CRC parameters: polynomial `$1021`, initial value `$FFFF`, no reflection or final
XOR. Calculate it over the entire file with bytes 14 and 15 treated as zero.
`tests/SaveReference.cs` is an independent host implementation.

## Payload

Offsets below are absolute file offsets. `src/save_format.inc` defines the same
schema and `save_ranges` in `src/save.inc` supplies the current runtime addresses.

| Offset | Bytes | Contents |
| --- | --- | --- |
| 16 | 15 | Bag pointer, two hand pointers, active torch pointer, weight, row, column, display mode, text polarity, object count |
| 31 | 1,008 | All 72 original 14-byte object records |
| 1,039 | 544 | All 32 original 17-byte creature records |
| 1,583 | 1,024 | Current maze cell bytes |
| 2,607 | 3 | Gameplay RNG seed |
| 2,610 | 1 | Level, zero based |
| 2,611 | 1 | Player direction |
| 2,612 | 2 | Damage/exertion |
| 2,614 | 1 | Scroll map flags |
| 2,615 | 9 | Power, heartbeat rate/countdown, recovery countdown/armed/pending, fainted/dead flags |
| 2,624 | 1 | Heartbeat phase |
| 2,625 | 2 | Torch burn task phase |
| 2,627 | 49 | Creature task state, cursor, regeneration countdown and 32 per-creature delays |
| 2,676 | 60 | Five-level creature population matrix |
| 2,736 | 1 | Combat enabled |
| 2,737 | 1 | Wizard transition state |
| 2,738 | 2 | Game jiffy and second |
| 2,740 | 1 | Artifact-color mode: 0 mono, 1/2 RF phases, 3/4 RGB phases, 5/6 crisp red/blue phases |
| 2,741 | 1 | Reserved legacy stereo flag. v0.35 writes zero and ignores 0 or 1 on load. Playback is mono |
| 2,742 | 1 | World-object handling enabled |
| 2,743 | 2 | Half-second task phase and pending ticks |

## Cassette sound

On a successful operation without `F`, XDOD v0.38 and later play 128 bytes of
`$55` as a leader, then generate tones from file offset 2,607 (`S_RNG`). At
the end of the file, the byte stream wraps to offset 1,583 (`S_MAZE`). Bits
are emitted least-significant first. Zero and one use one cycle at about
1,200 Hz and 2,400 Hz respectively, for about three seconds total. These
offsets are playback rules and require no additional file fields.

## Object pointers and compatibility

The four inventory pointers, each object's link word at record offset 0, and
each creature's object pointer at record offset 8 are absolute addresses relative
to the saved header's object-table base. Zero means null. On load, XDOD validates
that non-null pointers refer to aligned, allocated object records, then rebases
them to the current object's address. It also rejects cyclic object links.

Changing code/link addresses alone does not require a new save format. Changing
the field order, record layout, persistent state requirements or meanings does
require a new schema or an explicit migration. Unsupported versions, incorrect
lengths, CRC errors and invalid records/pointers are rejected before any live
game state is replaced.

The game clock/heartbeat/creature timers pause during I/O. A load redraws the
saved display, restores the color preference and discards old queued
commands. Missing/bad saves and failed writes report `FILE ERROR` and resume the
current game. The input reader is bounded; it never loads an unchecked file over
live working memory.



## Inventory record at offset 16

| Absolute offset | Size | Field |
| --- | ---: | --- |
| 16 | 2 | Backpack head |
| 18 | 2 | Left hand |
| 20 | 2 | Right hand |
| 22 | 2 | Lit torch object |
| 24 | 2 | Carried weight |
| 26 | 1 | Row, 0–31 |
| 27 | 1 | Column, 0–31 |
| 28 | 1 | Display: 0 dungeon, 1 inventory, 2 scroll map |
| 29 | 1 | Renderer polarity scratch. A different renderer can derive its own |
| 30 | 1 | Allocated object records, 1–72 |

## Object records

Record i starts at 31 + 14 × i. Record offsets in the next tables are relative to the record.

| Offset | Size | Field |
| --- | ---: | --- |
| 0 | 2 | Next object link |
| 2 | 1 | Row |
| 3 | 1 | Column |
| 4 | 1 | Level index, 0–5. Index 5 is outside the playable levels |
| 5 | 1 | Ownership: 0 floor, 1 player, 255 creature-owned or awaiting assignment |
| 6 | 3 | Class-specific values described below |
| 9 | 1 | Object type, 0–24 |
| 10 | 1 | Class: 0 flask, 1 ring, 2 scroll, 3 shield, 4 sword, 5 torch |
| 11 | 1 | Reveal value. Zero means revealed. REVEAL compares power with 25 times this value |
| 12 | 1 | Magic offense factor |
| 13 | 1 | Physical offense factor |

Type IDs in order from 0 through 24:

| IDs | Names in order |
| --- | --- |
| 0–4 | Supreme ring, Joule ring, Elvish sword, Mithril shield, Seer scroll |
| 5–9 | Thews flask, Rime ring, Vision scroll, Abye flask, Hale flask |
| 10–14 | Solar torch, Bronze shield, Vulcan ring, Iron sword, Lunar torch |
| 15–19 | Pine torch, Leather shield, Wooden sword, Final ring, Energy ring |
| 20–24 | Ice ring, Fire ring, Gold ring, Empty flask, Dead torch |

For torches, offsets 6–8 hold remaining burn minutes, regular light and magical light. For shields, offsets 6–7 hold magic and physical damage factors. Combat factors use 128 as unity. For rings, offsets 6–7 hold remaining charges where applicable and the type ID unlocked by INCANT. A zero incantation value means no pending transformation. Preserve all three bytes when round-tripping. Unrevealed items can carry generic stats. Restore the stored stats rather than regenerating them from type alone.

Backpack membership follows its head and links. Hands are single pointers. The torch pointer may refer to a backpack member. Each live creature has its own loot chain. Use these roots to determine reachable inventory. Some historical saves retain player-owned slots which the level-three transition removed from the backpack. Reconstructing inventory by ownership alone would restore confiscated items.

## Creature records

Record i starts at 1039 + 17 × i.

| Offset | Size | Field |
| --- | ---: | --- |
| 0 | 2 | Power |
| 2–5 | 4 | Magic offense, magic defense, physical offense, physical defense |
| 6–7 | 2 | Movement delay and attack delay in tenths of a second |
| 8 | 2 | Held-object list head |
| 10 | 2 | Accumulated damage |
| 12 | 1 | Occupied flag, 0 inactive or 255 live |
| 13 | 1 | Creature type, 0–11 |
| 14 | 1 | Facing, 0 north, 1 east, 2 south, 3 west |
| 15–16 | 2 | Row and column |

Types 0–11 in order are spider, viper, stone giant 1, blob, knight 1, stone giant 2, scorpion, knight 2, wraith, balrog, wizard 0 (level-three wizard), wizard 1 (final wizard). The population matrix at 2676 uses five consecutive rows of 12 counts in this same order. The current creature records describe only the active level. Future visits rebuild the population from the matrix.

## Maze, timers and state

The maze is 32 rows of 32 bytes at offset 1583. Index a cell as row × 32 + column. Byte 255 represents solid space. Otherwise the north, east, south and west edges occupy bit pairs 1:0, 3:2, 5:4 and 7:6. Edge codes 0, 1, 2 and 3 are passage, ordinary door, secret door and wall. The same facing numbering applies to the player. Vertical features come from the game's per-level tables, separate from the maze bytes. The three seed bytes at 2607 are in low-to-high shift-register order.

| Absolute offset | Field |
| --- | --- |
| 2614 | Map flag, 0 vision or 1 seer |
| 2615–2616 | Power word |
| 2617–2623 | Heartbeat interval, heartbeat countdown, recovery countdown, recovery armed flag, pending health ticks, fainted flag, dead flag |
| 2624 | Heartbeat phase, 0 or 1 |
| 2625–2626 | Half-second ticks toward next torch minute (0–119), then legacy burn-task byte |
| 2627–2630 | Creature task enabled, ready flag, redraw tick count, jiffy phase (0–5) |
| 2631–2641 | Creature worker scratch: index, type, remaining, direction, scan row, scan column, scan count, preference, tries, dirty, refresh |
| 2642–2643 | Round-robin cursor (0–31), minutes to regeneration (0 means ready) |
| 2644–2675 | Per-slot delays in tenths of a second. Zero is due for a live creature |
| 2737 | Wizard state: 0 normal, 10 pending level-three transition, 11 pending final-wizard transition, 12 ring riddle, 13 victory |
| 2738–2739 | Jiffy within second and second within minute, each 0–59 |
| 2743–2744 | Jiffy within half-second (0–29), then pending half-second ticks |

A C64 port can map timers to its own scheduler while retaining the bytes for round-trip export. XDOD uses 60 jiffies per game second and six jiffies per creature tick. Preserve the frozen ring-riddle state at 12, including one surviving object, cleared torch pointer and fixed regular/magical light of 7/19. Victory at 13 normally has no save command entry point.

## Interchange procedure

1. Read exactly 2745 bytes, check the header and verify the CRC before changing live state. Use a Commodore SEQ file or an equivalent byte stream.
2. Decode a nonzero pointer as slot = (pointer − header object base) / 14. Require a nonnegative exact multiple of 14 and a slot below object count. Use slot indices or rebase to the destination game's objects. Validate pointers even in the serialized inactive slots. XDOD checks all 72 links and 32 creature loot pointers and checks allocated object chains for cycles.
3. Map object/creature IDs and rebuild reachable chains. Restore the saved maze and RNG seed together. Generating the maze again can advance RNG state and replace saved cell bytes.
4. Restore the current level, position, damage, power, lit torch, population matrix and applicable timers. Adapt presentation flags to the destination renderer. Accept legacy stereo values 0 and 1 and write 0 on export.
5. On export, encode null as zero, choose a nonzero object base such as $4000, and encode slot i as base + 14 × i. Write the new header and CRC after serializing all payload bytes. Zero unused records when producing new files.

The existing CoCo conversion bundle contains 28 schema-1 examples, their manifest and a save index. It uses canonical base $4000 and translates cassette RAM and task queues offline. XDOD's on-device loader reads the resulting SEQ bytes directly. Save 28 is a useful endgame test because it resumes just before the final wizard fight.
