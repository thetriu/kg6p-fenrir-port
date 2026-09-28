# Fenrir device support: Tecno Spark 8T (kg6p)

This adds a [Fenrir](https://github.com/puuuuh/fenrir) `Device()` profile for the
**Tecno Spark 8T** (codename `kg6p`), a device not previously supported by
Fenrir or this fork.

## Status

- ✅ **Core exploit patch (`sec_get_vfy_policy`) — working, confirmed booting on real hardware.**
- ❌ **Boot-state spoofing (Orange State screen / `verifiedbootstate=orange`) — not solved.** See [Known limitations](#known-limitations) below.

## What this device profile does

The device ships an `lk.img` with a single executable sub-image (`lk`) plus a
device tree blob (`lk_main_dtb`) — no separate `bl2_ext`/`aee` split like some
other supported devices in this repo.

The patch targets the secure-boot policy check function, found via static
analysis (Ghidra, ARM:LE:32:v7 / Thumb):

- **Function:** unnamed, at `lk.img` offset `0x182D0` (payload offset `0x180D0`)
- **Behavior:** 4-instruction wrapper — calls a policy-reader function, extracts
  bit 1 of its return value via `ubfx`, returns it.
- **Confirmed via decompiled caller logic** (the `[SBC] img vfy(...)` verification
  path): a return value of `0` causes the caller to skip secure
  certificate/policy enforcement entirely and branch straight to plain
  image/hash verification.
- **Pattern uniqueness confirmed:** the 12-byte pattern occurs exactly once in
  both the extracted payload and the full `lk.img`.

```python
'sec_get_vfy_policy': PatchStage(
    'sec_get_vfy_policy',
    pattern='08 b5 ff f7 75 ff c0 f3 40 00 08 bd',
    replacement='00 20 70 47 00 bf 00 bf 00 bf 00 bf',
    match_mode=MatchMode.ALL,
    description="Don't enforce secure certificate/policy verification",
),
```

Replacement disassembles to `movs r0, #0; bx lr` padded with `nop`s to fill the
original 12-byte function exactly (verified by hand: Thumb `MOVS r0,#0` →
`0x2000` → bytes `00 20`; `BX LR` → `0x4770` → bytes `70 47`; `NOP` →
`0xBF00` → bytes `00 bf`).

## How it was found

1. Extracted `lk.img` from the stock ROM's `payload.bin`.
2. Loaded the `lk` sub-image in Ghidra (`ARM:LE:32:v7`, base address
   `0x48000000` per the `liblk`-reported partition metadata).
3. Searched for short functions with a "return-value → conditional branch"
   shape, called from paths referencing `[SBC]`/verification strings.
4. Identified the strongest candidate, then confirmed its semantics by reading
   the decompiled logic of its caller (the actual `img vfy` routine) rather
   than guessing from shape alone.
5. Verified pattern uniqueness (1 hit) before writing the patch.
6. Built and flash-tested on real hardware — device boots successfully.

## Known limitations

The device correctly displays the **"Orange State"** warning at boot and
reports `androidboot.verifiedbootstate=orange` — this is accurate, since the
state-spoofing patches (equivalent to `force_green_state` /
`spoof_seccfg_state` / etc. on other supported devices) have not been added
for this device.

Three separate, independent investigations failed to find a compiled
reference to the relevant strings in `lk.bin`, despite the strings
themselves being present:

- **Orange State display function** — strings at `0x90E0C`/`0x90E1C`
  ("Orange State" / "Your device has been unlocked and can't be trusted").
  Reference source (`orange_state_warning()`) was located upstream and used
  to guide the search; no matching compiled function with a live reference
  was found.
- **Boot-state cmdline strings** — the four complete literals
  `androidboot.verifiedbootstate={orange,yellow,red,green}` at
  `0x90F08`–`0x90F7C`. No executable reference found via Ghidra's reference
  manager, literal-pool dereferencing, or PC-relative load scanning.
- **Base-address hypothesis ruled out** — re-analyzed with the correct load
  base (`0x48000000`, from `liblk` partition metadata, instead of `0x0`);
  reference search still returned zero hits.
- **Jump-table hypothesis checked** — scanned for Thumb-2 `TBB`/`TBH`
  table-branch instructions (62 found); none resolve to these strings.

**Working hypothesis:** the actual `cmdline_append()`/boot-state-display
logic may live in the SoC's Preloader stage (a separate binary, not analyzed
here) rather than in `lk` itself, with these strings present in `lk.bin`
only as unreferenced remnants of a statically-linked vboot library. This
would require extracting and analyzing the Preloader separately, or dynamic
tracing on real hardware, to confirm — out of scope for this PR.

If anyone picks this up: don't re-run the same three searches above, they're
confirmed dead ends on this binary. Start from the Preloader-extraction
angle instead.

## Practical impact of the missing spoofing

The core exploit is fully functional (custom ROM flashing, root, full
partition access all work normally). The only real-world limitation is that
apps/services checking `verifiedbootstate` at runtime (Play Integrity,
banking apps, some DRM) will correctly see `orange` and may restrict
functionality accordingly.
