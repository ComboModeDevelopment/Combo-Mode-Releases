# ft_falcomelee REL Fix — Laser + Phantasm Articles

## Overview

The `ft_falcomelee` REL is a Falco-based clone (kind `0x5D`, "Falco (Melee)"). Its `Section[8]` is currently a verbatim copy of `ft_meleefox`'s `Section[8]`, which targets Fox internals throughout. This causes:

- **Both laser AND phantasm articles appear simultaneously** when using the laser move
- **Laser hitbox does not work**
- **Phantasm crashes the game** on activation

### Root Cause

Without working `1D8A` patches, Falco `Section[1]` retains `li r4, 0x8` (Falco's native kind) when calling article getter functions. The getter validates `r4 == 0x8` and passes, returning Falco's base module data — not the clone's. Articles spawn with wrong module associations (positions off, hitboxes broken). Phantasm crashes because its constructor receives wrong module data for the clone's skeleton.

The D8A getter replacements in the current Section[8] target Fox's getter functions (`loc_148380`, etc.) which Falco's `Section[1]` never calls — they have no effect.

---

## Key Values

| Item | MeleeFox (existing) | FalcoMelee (target) |
|------|---------------------|---------------------|
| Clone kind | `0x55` | `0x5D` |
| Native kind in REL | `0x6` (Fox) | `0x8` (Falco) |
| Gun getter (DOL module 27, sec 1) | `loc_148380` | `loc_148524` |
| Bullet getter | `loc_148414` | `loc_14858C` |
| Illusion/Phantasm getter | `loc_1484A8` | `loc_148620` |
| Landmaster/4th getter | `loc_1484BC` | `loc_1486B4` |
| Gun module field offset | `0x9C` | `0xAC` ⚠️ verify |
| Bullet module field offset | `0xA0` | `0xB0` ⚠️ verify |
| Illusion module field offset | `0xA4` | `0xB4` ⚠️ verify |
| Landmaster field offset | `0xA8` | `0xB8` ⚠️ verify |
| getFinalResId fallback kind | `li r4, 0x6` | `li r4, 0x8` |
| REL module reference | `"ft_fox"` | `"ft_falco"` |

**Getter addresses** from `sora_melee.map` (or `RSBE01.map`):
- `getWeaponData52` = `0x148524` — Falco gun (FtFalcoGun)
- `getWeaponData53` = `0x14858C` — Falco bullet
- `getWeaponData54` = `0x148620` — Falco phantasm (wnFalcoIllusion)
- `getWeaponData55` = `0x1486B4` — Falco Landmaster/4th weapon

**Field offsets** are inferred as sequential after Fox (`0x9C`→`0xAC`, `0xA0`→`0xB0`, etc.). **Verify with Dolphin before editing** (see Step 1).

**Falco native kind** = `0x8` — confirmed by multiple `li r4, 0x8` instances in `rels/ft_falcomelee/Section[1].asm`.

**Falco fighter map** found at:
`C:/Users/pvstv/Documents/Smash_Work/Brawl_Work/Tools/Reltools/maps/ft_falco.map`

---

## Current Section[8] State (ft_falcomelee — what's wrong)

The Section[8] at `rels/ft_falcomelee/Section[8].asm` currently contains:

| Entry | Current (wrong) | Should be |
|-------|-----------------|-----------|
| Header kind | `0x00000055` | `0x0000005D` |
| D8A 1 target | `loc_148380` (Fox gun getter) | `loc_148524` (Falco gun getter) |
| D8A 1 dest | `"ft_fox", 8, "loc_A0"` | `"ft_falco", 8, "loc_A0"` |
| D8A 2 target | `loc_148414` (Fox bullet getter) | `loc_14858C` (Falco bullet getter) |
| D8A 2 dest | `"ft_fox", 8, "loc_C0"` | `"ft_falco", 8, "loc_C0"` |
| D8A 3 target | `loc_1484A8` (Fox illusion getter) | `loc_148620` (Falco illusion getter) |
| D8A 3 dest | `"ft_fox", 8, "loc_E0"` | `"ft_falco", 8, "loc_E0"` |
| D8A 4 dest | `"ft_fox", 8, "loc_130"` | `"ft_falco", 8, "loc_130"` |
| D8A 5 dest | `"ft_fox", 8, "loc_160"` | `"ft_falco", 8, "loc_160"` |
| D8A 6 target | `loc_1484BC` (Fox Landmaster getter) | `loc_1486B4` (Falco 4th getter) |
| D8A 6 dest | `"ft_fox", 8, "loc_180"` | `"ft_falco", 8, "loc_180"` |
| 1D8A ref | `"ft_fox", 8, "loc_100"` | `"ft_falco", 8, "loc_100"` |
| loc_100 entries | 9× `"ft_fox", 1, "loc_???+2"` | 9× `"ft_falco", 1, "loc_???+2"` (Falco Section[1] sites) |
| loc_A0 `lwz` opcode | `0x8063009C` (field 0x9C) | `0x806300AC` (field 0xAC) |
| loc_C0 `lwz` opcode | `0x806300A0` (field 0xA0) | `0x806300B0` (field 0xB0) |
| loc_E0 `lwz` opcode | `0x806300A4` (field 0xA4) | `0x806300B4` (field 0xB4) |
| loc_180 `lwz` opcode | `0x806300A8` (field 0xA8) | `0x806300B8` (field 0xB8) |
| loc_130 fallback | `0x38800006` (`li r4, 0x6`) | `0x38800008` (`li r4, 0x8`) |

---

## Candidate 1D8A Sites in Falco Section[1]

Agent scan of `rels/ft_falcomelee/Section[1].asm` for `li r#, 0x8`:

| Address (section-relative) | Register | Context | Likely kind site? |
|----------------------------|----------|---------|-------------------|
| `0x00000944` | r6 | function init block | possible |
| `0x000011BC` | r4 | param, then `li r5, 0x2` | **yes** |
| `0x000011D8` | r4 | param, then `li r5, 0x1` | **yes** |
| `0x000011F4` | r4 | param, then `li r5, 0x0` | **yes** |
| `0x0000126C` | r6 | soAnimCmdInterpreter param | possible |
| `0x00008104` | r0 | extsh sequence | no |
| `0x0000954C` | r0 | store to `0x14(r1)` | **likely** (article ctor range) |
| `0x00009B08` | r0 | store to `0x8(r1)` | **likely** (article ctor range) |
| `0x0000A77C` | r4 | param with `li r5,0`, `li r6,1`, `li r7,-1` | **yes** |
| `0x0000DD3C` | r3 | return value (`getMediateNum`) | no |
| `0x0000EA14` | r0 | extsh sequence | no |

**For the 1D8A patch entry**, the target addresses are each site address `+ 2` (to hit the immediate field of the `li` instruction). For example, site at `0x11BC` → patch target `"ft_falco", 1, "loc_11BC+2"`.

MeleeFox had 9 sites. Falco likely has the same count. **Confirm with Dolphin** (Step 2 in plan).

---

## Step-by-Step Fix Plan

### Step 1 — Verify getter field offsets (Dolphin)

Set BP at `loc_148524` (Falco gun getter runtime address). Fire the Falco clone's laser.
Step through and read the `lwz r3, 0x??(r3)` opcode — the offset is the gun field.
Repeat for bullet (`loc_14858C`), illusion (`loc_148620`), Landmaster (`loc_1486B4`).
Expected: `0xAC`, `0xB0`, `0xB4`, `0xB8`.

### Step 2 — Find 1D8A sites (Dolphin)

Set BPs at all four Falco getter addresses. When each fires, note the Section[1] offset of the `li r4, 0x8` that immediately precedes the `bl`. Those offsets `+2` are the patch targets for loc_100.

### Step 3 — Edit `rels/ft_falcomelee/Section[8].asm`

Apply all changes from the table above:
1. Header: `0x55` → `0x5D`
2. D8A entries 1–3: swap getter target addresses to Falco's
3. D8A entries 1–3: swap `"ft_fox"` → `"ft_falco"` in dest refs
4. D8A entries 4–5: swap `"ft_fox"` → `"ft_falco"` in dest refs (target stays same)
5. D8A entry 6: swap getter target to Falco's, swap `"ft_fox"` → `"ft_falco"`
6. 1D8A entry: swap `"ft_fox"` → `"ft_falco"`
7. loc_100: replace all 9 address entries with confirmed Falco Section[1] sites
8. loc_A0/C0/E0/180: update `lwz r3, 0x??(r3)` opcode bytes for new field offsets
9. loc_130: `0x38800006` → `0x38800008` (getFinalResId fallback)

### Step 4 — Verify ft_falcomelee.json

Confirm:
- `"Name": "ft_falco"` (not meleefox)
- `"ModuleID": 108`
- Section[8] `ExpandSize` ≥ current content size (MeleeFox used 172)

### Step 5 — Build and test

Build `ft_falcomelee.rel` via BrawlEX REL tooling using `rels/ft_falcomelee/ft_falcomelee.json`.
Copy to `modfolder/ComboMode+/rsbe/pf/module/ft_falcomelee.rel`.
Run `build.bat` and sync via VSDSync.

Test:
- Laser: gun at correct hand position, bullet travels with ef_custom bank `0x14F` GFX, hitbox works
- Phantasm: no crash

### Step 6 — Handle phantasm crash if it persists

Enable MMU in Dolphin. Trigger phantasm crash. Note `SRR0` (crash instruction) and `LR`.
Map `SRR0` to a function using `ft_falco.map`:
- Convert: `runtime_crash_addr - ft_falco_load_base = section_offset`
- Find in `ft_falco.map` which function contains that offset

If the crash is in a sub-constructor that receives a null module pointer (same pattern as `wnFoxLandMasterShell::__ct` crash for MeleeFox), add a hook in `CloneEngine.asm`:

```asm
; Phantasm sub-ctor null skip for FalcoMelee clone
HOOK @ $<address_of_bl_to_crashing_subctor>
{
    lwz r0, 0xC8(<self_reg>)   ; founderKind of article self
    cmpwi r0, 0x5D             ; FalcoMelee?
    beq- end%                  ; skip sub-ctor for clone
    stwu r1, -0x10(r1)
    mflr r0
    stw r0, 0x14(r1)
    bctrl
    lwz r0, 0x14(r1)
    mtlr r0
    addi r1, r1, 0x10
end%:
}
```

Self register (the `r##` for `0xC8` offset) needs to be confirmed from Dolphin register dump at the crash site.

---

## Reference: MeleeFox Section[8] Structure (Working Template)

Source: `rels/ft_meleefox/Section[8].asm`

```
loc_0:    .4byte 0x00000055     ; clone kind
          (padding CCCCs)

D8A 1:    source = (27, 1, loc_148380)     ; Fox gun getter
          dest   = (ft_fox, 8, loc_A0)
          size   = 0x18, type = 0xD8A

D8A 2:    source = (27, 1, loc_148414)     ; Fox bullet getter
          dest   = (ft_fox, 8, loc_C0)
          size   = 0x18, type = 0xD8A

D8A 3:    source = (27, 1, loc_1484A8)     ; Fox illusion getter
          dest   = (ft_fox, 8, loc_E0)
          size   = 0x18, type = 0xD8A

D8A 4:    source = (27, 1, loc_85E06C)     ; getFinalResId
          dest   = (ft_fox, 8, loc_130)
          size   = 0x24, type = 0xD8A

D8A 5:    source = (27, 1, loc_3117B4)     ; fighter kind bypass
          dest   = (ft_fox, 8, loc_160)
          size   = 0x10, type = 0xD8A

D8A 6:    source = (27, 1, loc_1484BC)     ; Fox Landmaster getter
          dest   = (ft_fox, 8, loc_180)
          size   = 0x18, type = 0xD8A

1D8A:     source = (null)
          dest   = (ft_fox, 8, loc_100)
          count  = 9, type = 0x1D8A

loc_A0:   lwz r3,0(r3) / mulli r4,r4,12 / lwzx r3,r3,r4 / lwz r3,0x9C(r3) / blr
loc_C0:   same but field 0xA0
loc_E0:   same but field 0xA4
loc_100:  9× ft_fox Section[1] addresses (+2) for li r#,0x6 patch sites
loc_130:  getFinalResId replacement code (ends with li r4,0x6 fallback)
loc_160:  fighter kind bypass (Wolf check: cmpwi r3,0x2C)
loc_180:  same as loc_A0 but field 0xA8 (Landmaster)
```

---

## Related Files

| File | Purpose |
|------|---------|
| `rels/ft_falcomelee/Section[8].asm` | **Primary edit target** |
| `rels/ft_falcomelee/Section[1].asm` | Falco base rel code — find `li r#, 0x8` sites |
| `rels/ft_falcomelee/ft_falcomelee.json` | REL build config |
| `rels/ft_meleefox/Section[8].asm` | Working reference template |
| `modfolder/ComboMode+/Source/ProjectM/CloneEngine.asm` | DOL-level hooks (phantasm crash fix goes here) |
| `modfolder/ComboMode+/rsbe/pf/module/ft_falcomelee.rel` | Output binary |
| Maps: `RSBE01.map`, `ft_falco.map` | For crash address lookup |

## GFX Fix (already in place)

The blaster bullet GFX redirect for FalcoMelee is already wired in `CloneEngine.asm`:
```asm
HOOK @ $80A2E138
{
    lwz r0, 0xC8(r25)
    %GFXFixOri(0x55, 0x014A, 0xA)   ; MeleeFox → ef_custom13
    %GFXFixOri(0x5D, 0x014F, 0xA)   ; MeleeFalco → ef_custom18 (bank 0x14F, local 0xA)
end:
    mtctr r12
}
```
This hook is at `activate/[wnFoxBlasterBullet]` + offset, and fires for Fox's blaster bullet only. Falco's blaster (wnFalcoBlasterBullet) may need a **separate hook** at the equivalent offset in Falco's blaster activate function — check if FalcoMelee's laser bullet uses this same Fox path or a different one.
