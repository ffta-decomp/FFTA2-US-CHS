# FFTA2 USA Rev 0 DraStic graphics compatibility patch

[Download the r1 BPS patch](ffta2-usa-drastic-bg3-wrap-r1.bps?raw=true)

This optional patch is for the original North American release of *Final Fantasy Tactics A2: Grimoire of the Rift* (A6FE, revision 0). Back up your in-game save and start a fresh emulator session. The patch preserves the original language options. **Compatibility on the original DraStic build remains unverified.**

## Apply the patch offline

1. Save the [standalone HTML patcher](bps-patcher.html?raw=true) as `bps-patcher.html` and open it in your browser. If GitHub displays the source, use **Save link as** to save the file.
2. Drop your clean USA Rev 0 `.nds` file and the downloaded `.bps` file into the patcher, or select them with the file pickers.
3. Choose **Apply patch**, then **Download ROM**. The patcher verifies the source, patch, and output before enabling the download. It works offline and leaves the input ROM intact.

| File | SHA-256 |
| --- | --- |
| Original USA Rev 0 ROM | `3b5d469aae5d351de31a89cf2bf592c98cb5b68cb4efc30b34426a119fee2372` |
| r1 BPS | `9c3957d39b7dbd94ee1100841ae203e184fa1950076d3b3819a426969abaf70d` |
| Patched USA Rev 0 ROM | `960e946f01f34d905db907a8db33aec33112bbad0bb33a88728c888942b7a2b9` |

## Reproduction and observations

1. Start a new game with the unmodified USA Rev 0 ROM. Play through the prologue into the first forest battle against Klesta. Watch the arrival sequence before issuing a battle command. A battle suspend save can skip part of that sequence.
2. Repeat the route with r1 and the same inputs. In a replay using the instrumented melonDS 0.9.5 software renderer, the original entered the relevant BG3 clipping path during the prologue and Klesta's arrival. Both runs reached the first-battle dialogue. The patched run completed the same **24,346 frames** with zero matching clipping records.
3. For a separate spell test, compare Thunder casts that hit one, two, and three units. Controlled replays on a Plus build entered the same clipping path for all three target counts. This included a three-target case in which every target survived and double speed was disabled. Applying the same two instruction changes preserved the measured damage.

A separate Blue Mage test on Plus successfully learned the skill and returned to the command menu. The tested learning case and its miss control produced zero matching clipping records. Other skills, camera angles, and battle states remain open.

These comparisons reproduce the graphics condition in a diagnostic core. A matching clipping record counts a rendering observation. Original DraStic crash reproduction remains pending.

## Rendering behavior and workaround

Some battle and cutscene effects share an 8-bit indexed affine bitmap on the main display engine's BG3 layer. When a transform puts a screen pixel outside a bitmap with wrapping disabled, native DS rendering treats that pixel as transparent. Earlier reports point to this clipping path in DraStic.

r1 changes two ARM9 instructions in the shared bitmap setup. The 128×128 and 256×256 indexed branches enable wrapping so that out-of-range coordinates map to the opposite edge. The direct-color branch, effect assets, battle rules, and save format stay as in the original.

Some frames can show extra flashes or repeated edges where the original would show transparency. This was visible during Fire and Klesta's arrival. The long-term emulator fix is to treat out-of-range pixels in non-wrapping bitmaps as transparent before fetching the pixel.

## Diagnostic details and reproduction provenance

### Environments and what was observed

| Item | Identity and scope |
| --- | --- |
| Public crash report | European FFTA2; Android DraStic **r2.6.0.4a build 109**, ARM32 native `SIGSEGV` in the [third-party dump](https://www.scribd.com/document/838489813/crash-dump). The supplied investigation attributes the instruction to `libdrastic.so`; the dump alone does not establish the exact library identity, ROM SHA, triggering animation or library SHA/build ID. |
| USA A/B experiment | The exact USA Rev 0 and r1 hashes above; an instrumented software-renderer build based on [melonDS 0.9.5](https://github.com/melonDS-emu/melonDS/releases/tag/0.9.5), upstream tag commit [`430de6b2702bb93faa8c2004aff3fbd084db4a1e`](https://github.com/melonDS-emu/melonDS/tree/430de6b2702bb93faa8c2004aff3fbd084db4a1e). This is a modified standalone diagnostic build, not the official binary or a libretro core. |
| Instrumented library | Local `libquest_oracle.dylib`, SHA-256 `5e98ae5457678ba1ea82a46986f14bba48c230ba4d16748323639bc2cdda1ea5`; its `GPU2D.cpp` and `GPU2D_Soft.cpp` contain instrumentation. The original DraStic library was **not** used for the USA A/B replay. |
| Other platforms | The supplied investigation describes ARM32/ARM64 isolated sampling experiments and about 300 enemy attacks without a crash on a Mac; binary identities and equivalent inputs are not fully recorded there. Do not interpret those observations as a measured DraStic failure rate. |

The third-party EU dump reports `ldrb r0, [r6, r0, lsl #0]`, `r6 = 0x8FF9F000`, `r0 = 0xFFFFFFAA` (−86), and fault address `0x8FF9EFAA`. The arithmetic matches. A 128-byte bitmap row sampled at `(42, -1)` would also yield `-1 * 128 + 42 = -86`; this is an **illustrative reconstruction**, not a recorded coordinate or proof that the USA replay has the same crash path. The original native library and its SHA were not independently available for this comparison.

### Repeatable USA graphics observation

1. Use fresh in-game save data and the same software-renderer settings for both ROMs. Do not start from a first-battle suspend state: it can omit the arrival sequence. Record ROM and emulator hashes, platform/ABI, JIT and renderer settings, frameskip, speed, and save identity.
2. Instrument the indexed affine-bitmap sampling path to record main engine BG3, display mode and `BG3CNT`, transformed coordinates, bitmap size, and whether wrap is enabled. Count an out-of-range no-wrap sample as a **clip observation**, not a crash. Do not perform an unsafe host memory read to model the fault.
3. Replay the same inputs from cold start through Klesta's arrival and the first battle dialogue. The original and r1 each ran **24,346 frames**. The original had **7,765 frames** containing a match and **23,254 records**, grouped by frame, engine, layer and out-of-range axis; r1 had **0** matching records. This counter is not a count of pixels or crashes. The optional synthetic-fault mode was disabled.
4. Example from the original run: at frame 16,409, main engine 0 / BG3, `DISPCNT=0x00011B1D`, `BG3CNT=0x0081`, a 128×128 bitmap had an out-of-range sample at `(128, 0)`. The corresponding setup log showed the patched value `0x2081`, enabling wrap. These exact log values are one observation, not a guarantee of the same frame on different configurations.
5. Separately exercise Fire and Thunder with one, two and three targets; retain the same target positions and compare the resulting HP/MP and screenshots. The Thunder tests used a separately identified Plus build with the same two instruction edits; they are not cold-start runs of the standalone USA r1 ROM. The first reported Fire sample differed by 17,211 pixels: wrap can visibly repeat edges or add flashes.

The test requires instrumentation: the public BPS does not include the diagnostic library, replay script, game ROM, BIOS, or saves. The 31-run coverage listed above is a bounded diagnostic result, not a complete gameplay or real-DraStic acceptance test.

### Two-instruction ROM workaround

For the exact USA Rev 0 input only, ARM9 is loaded from ROM file offset `0x4000` to `0x02000000`. The r1 patch changes these two 32-bit ARM instructions; bytes below are in ROM little-endian order:

| Path | ARM9 address | ROM offset | Original bytes | r1 bytes |
| --- | --- | --- | --- | --- |
| 128×128 indexed | `0x020E7F70` | `0x000EBF70` | `80 00 80 E3` | `82 0D 80 E3` |
| 256×256 indexed | `0x020E7F98` | `0x000EBF98` | `80 00 80 E3` | `82 0D 80 E3` |

```asm
; At each site, OR the existing r0 value; neither instruction updates condition flags.
; Before: orr r0, r0, #0x0080
; After:  orr r0, r0, #0x2080  ; also sets BGxCNT bit 13 (wrap)
```

The resulting indexed 128×128 `BG3CNT` example is `0x0081 → 0x2081`; instruction-level tests also gave `0x4081 → 0x6081` for the 256×256 branch. A direct-color control remained `0x0085`. The 256×256 branch was checked at instruction level; matching 256×256 effect assets were not found in the scanned archive. Addresses and original bytes are version-specific; do not apply them to the European, localized or modified ROMs by assumption.

The emulator-side behavior to check is conceptually:

```text
if wrap is disabled and (x < 0 or x >= width or y < 0 or y >= height):
    skip this background pixel  # transparent; do not read bitmap memory
else:
    sample the in-bounds or wrapped bitmap coordinate
```

This is a description of the expected boundary behavior, not recovered DraStic source. The r1 ROM workaround instead enables wrapping, so its repeated-edge visual difference is expected.

### References

- [melonDS 0.9.5 indexed bitmap software renderer, pinned upstream source](https://github.com/melonDS-emu/melonDS/blob/430de6b2702bb93faa8c2004aff3fbd084db4a1e/src/GPU2D_Soft.cpp) — reference for the guarded 2D rendering path; our diagnostic build is instrumented.
- [GBATEK: Nintendo DS video](https://mgba-emu.github.io/gbatek/#ds-video) — BG2/BG3 area-overflow and bitmap behavior.
- [Public European DraStic crash dump](https://www.scribd.com/document/838489813/crash-dump) — independent third-party report, with the version and register values above.
- [Community reports of FFTA2 DraStic crashes](https://www.reddit.com/r/finalfantasytactics/comments/16mdebm/solution_ffta2_crashing_on_drastic/) — reported symptoms and frameskip observations, not a controlled reproduction.
- [SeedlessDS-Core GPU source](https://github.com/SapphireRhodonite/SeedlessDS-Core/blob/mainSub/src/gpu/gpu.c) — a separate open-source reconstruction for further comparison; not official DraStic source and not the binary used in this experiment.
- [nds-bootstrap issue #1683](https://github.com/DS-Homebrew/nds-bootstrap/issues/1683) — a separate flashcard/loader freeze report; a common cause with this emulator-side hypothesis has not been established.

## Test coverage

The structural scan covered all **421 recognized EFX containers** in the USA Rev 0 archive index. It found **192** candidates with 128×128 indexed bitmaps, **167** of them associated with registered magic effects. Runtime coordinates depend on the animation, camera, and targets.

The 31 diagnostic runs covered cold prologue replays, Fire, four Klesta animation records, Thunder with several target counts, and eight additional effect spot checks. The 256×256 setup branch passed instruction-level checks. All matching bitmaps found in the archive were 128×128. Full gameplay and original DraStic validation remain open.

## Appendix — Original reproduction materials

| Supplied material | How it informed the tests |
| --- | --- |
| *FFTA2 ROM Crash Investigation*, revision 2, 2026-11-19 | The report analyzed a publicly reported European-version DraStic crash and proposed indexed bitmap clipping as a candidate mechanism. Player reports mentioned multi-target Thunder, bird attacks, and summons. The report also recorded roughly 300 enemy attacks on a Mac without a crash, with incomplete equivalence to the reported setups. These observations motivated checking both the arrival sequence and spell animations. The European report and the USA Rev 0 replay provide separate evidence. |
| First-battle suspend save | The supplied save cold-loaded into the Klesta battle. The observed continuation produced zero matching clipping records. The clean new-game replay was used to capture the earlier arrival sequence. |
| Ordinary save after the first quest | The second supplied save records one completed quest. It was inspected as a progress reference. The Klesta arrival and Thunder results above came from their own qualified inputs. |

For visual reports, capture both DS screens side by side in landscape. Use the same layout for the original and patched ROMs, and record the lead-in, the effect, and what happens afterward.
