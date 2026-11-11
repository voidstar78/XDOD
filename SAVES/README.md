# XDOD native saves — schema 1

`ZSAVE` overwrites `XDOD.SAV` on device 8 in its current directory. `ZLOAD`
restores that file. An optional one-to-eight-digit argument selects XDOD-NUMBER.SAV; leading zeroes are preserved. Other filename arguments are rejected. The original unique
prefix parser also accepts `ZS` / `ZL`.

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
| 2,737 | 1 | Pending wizard branch |
| 2,738 | 2 | Game jiffy and second |
| 2,740 | 1 | Artifact-color mode: 0 mono, 1/2 RF phases, 3/4 RGB phases, 5/6 crisp red/blue phases |
| 2,741 | 1 | Reserved legacy stereo flag; playback is always centered in v0.32 |
| 2,742 | 1 | World-object handling enabled |
| 2,743 | 2 | Half-second task phase and pending ticks |

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
saved display, restores the sound/color preference and discards old queued
commands. Missing/bad saves and failed writes report `FILE ERROR` and resume the
current game. The input reader is bounded; it never loads an unchecked file over
live working memory.

## Original CoCo recordings

The cartridge does support a cassette filename: `PZTAPE.ASM:FILNAM` obtains the
next token, and `COMMON.ASM:LOAD12` compares eight header characters. Keeping a
fixed disk filename in XDOD is an intentional simplification.

Your [save archive](https://github.com/voidstar78/daggorath_saves) contains WAV
recordings verified on CoCo1 and CoCo3. There is a standard practical route to
CAS: [XRoar's tape rewrite mode](https://www.6809.org.uk/xroar/doc/xroar.shtml)
reads WAV input and writes a sanitized CAS stream through ROM tape-call
interception. Daggorath uses those ROM block calls, so it is a promising route;
this release has **not** converted or imported the recordings.

CAS conversion alone is insufficient for XDOD compatibility. The original writes
128-byte data blocks beginning at `DP.BEG` and restores a 6809 RAM image starting
at `$0200`. A host-side importer must extract/check those blocks, map original
RAM symbols, convert big-endian words and object links, and translate scheduler
queues into the port's timer representation. It can then emit this native format
and CRC. This keeps WAV/CAS decoding out of the X16 runtime and leaves the
recordings intact as preservation sources. Original-save import remains pending.

## Runtime layout and verification

The 1,258-byte helper is embedded in the losslessly compressed startup payload
and installed at `$8000`; its workspace is `$8F00`, and bounded staging begins
at `$9000`. Its temporary banked source bytes are subsequently reused for the
derived BANG/THUD traces. No helper file is required at runtime.

The r49 tests use real KERNAL sequential I/O and separate emulator processes.
They cover overwrite, an independently calculated CRC, all persistent bytes,
pointer rebasing between different link layouts, parser/redraw behavior, missing
files, bad versions, truncated/oversized data, corrupt checksums, invalid pointers,
cycles, invalid task cursors and failed writes. HostFS is tested; physical
SD-card/hardware operation has not yet been exercised here.
