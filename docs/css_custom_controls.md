# CSS Custom Controls — Adding New Options

How the per-nametag custom control system works, and a step-by-step guide for adding new options to the menu.

Uses the analog shield toggle as the worked example throughout. The entire CSS custom controls system, including the analog shield hooks that piggyback on it, lives in [CSSCustomControls.asm](../modfolder/ComboMode+/Source/LegacyTE/CSSCustomControls.asm) (analog shield section at the top of the file).

---

## Architecture overview

CSS Custom Controls is fundamentally a **UI for editing bytes in Brawl's save-file nametag data**. The tag data lives at `0x90172E20` (each tag is `0x124` bytes). Every configurable button mapping is stored as a specific byte offset within that tag block, and Brawl's own vanilla code reads those bytes at match start to apply control settings.

That means for most options — button remaps, tap jump, C-Stick — **there is no runtime code at all**. The CSS code writes to the tag; Brawl's native control-application pipeline reads it back automatically at match start.

**Custom features that Brawl doesn't natively understand** (like this analog shield toggle) need extra machinery: a game-side hook that reads the tag setting and gates the feature per-player.

### Three data tables

Three inline data blocks live in the GCT and are accessed indirectly via three fixed DOL pointer slots:

| DOL slot | Table | Purpose |
|---|---|---|
| `$8069FE7C` | `InputTable` | Per-controller-type metadata: slot counts, C-Stick/Tap Jump/Shake Smash/Analog Shield slot indices, and navigation button bitmasks |
| `$8069FE80` | `DataRefTable` | Runtime state: MiscData pointers, current tag pointers, per-port controller registration, and per-port analog shield flags |
| `$8069FE84` | `ButtonTable` | Menu index ↔ save-value ↔ tag-byte-offset lookup tables per controller type |

Every hook that touches these tables uses the same pattern:
```asm
lis rX, 0x806A
lwz rX, -0x180(rX)   # DataRefTable (or -0x184 for InputTable, -0x17C for ButtonTable)
```

### Page system

The tag menu uses a `page` byte at `0x60(r5)` (r5 = tag menu struct pointer):

| Page | Meaning |
|---|---|
| 0 | Tag rename (handled outside this file) |
| 1 | Normal tag page — home for a selected tag |
| 2 / 10-13 | Custom Input Settings — which action is mapped to each button (controller-specific: 10=GC, 11=CC, 12=Wii, 13=Nun) |
| 3 | Command picker — which action goes on the highlighted input |

Toggle options (Tap Jump, Shake Smash, Analog Shield) reuse Page 3 with only two entries (Off/On).

---

## Byte-in-tag storage

### Layout of a nametag block (0x124 bytes)

```
0x00-0x0B  6-character UTF-16 name
0x0C       Rumble
0x0D-0x0F  Free (currently: 0x0D bit 0x01 = Analog Shield)
0x10-0x13  Pin priority word — WRITTEN by StartPressP1 pin logic. DO NOT USE.
0x14-0x1F  GC controls  (0x1F bit 0x80 = Tap Jump, bit 0x70 = "controls complete")
0x20-0x27  Wiimote controls
0x28-0x33  Nunchuk controls  (0x33 bit 0x40 = Tap Jump, bit 0x80 = Shake Smash)
0x34-0x40  Classic controls  (0x40 bit 0x80 = Tap Jump)
```

### Gotcha: pin logic clobbers 0x10-0x13

`StartPressP1` (line ~442 of CSSCustomControls.asm) implements "pin this tag" via:
```asm
stwx r3, r4, r6   # writes a 4-byte word to tag[0x10..0x13]
```

Any bit you store in `0x10-0x13` will be wiped the moment the player presses Start on Page 1. Byte `0x10` was originally the natural home for a global toggle bit, but it must not be used. Byte `0x0D` is verified free and is the current home of the analog shield bit.

### Choosing a byte for a new option

- **Global toggle** (applies across controller types): use unused bits of `0x0D`, `0x0E`, or `0x0F`. Bit `0x01` of `0x0D` is taken by analog shield.
- **Controller-specific toggle**: use unused bits within that controller's section (e.g., bits `0x0F` of `0x1F` are free for GC-only flags).
- **Multi-value option** (not a bit-flag): dedicate a full byte within the appropriate controller section.

---

## InputTable layout

```
0x00: byte[4] input slot counts (-1) per controller — GC, CC, Wii, Nun
0x04: byte[4] C-Stick slot per controller
0x08: byte[4] Tap Jump slot per controller
0x0C: byte[4] Shake Smash slot per controller
0x10-0x1F: word[4] "Backward" button bitmasks
0x20-0x2F: word[4] "Forward" button bitmasks
0x30-0x3F: word[4] Y+Z combo bitmasks
0x40-0x4F: word[4] Y-alone bitmasks
0x50-0x5F: word[4] Z-alone bitmasks
0x60: byte[4] Analog Shield slot per controller  ← added after the bitmask rows
```

**Rule for adding new option-slot rows:** do NOT insert them between the existing byte rows and the bitmask rows — the bitmask offsets (`0x10, 0x20, 0x30, 0x40, 0x50`) are hardcoded in APressP2, TagPage, and elsewhere. Instead, append new rows AFTER `0x5F` (starting at `0x60`, `0x64`, etc.). Update the hook code to reference the new offset explicitly.

---

## ButtonTable layout

Each controller type has two 16-byte sub-sections inside a 32-byte block:

```
0x00-0x2F  Button: read-by-index, save-as, read-by-save (shared, non-controller-specific)
0x30-0x5F  C-Stick: same three tables
0x60-0x6F  Wiimote Button: two tables (no read-by-save)

Then, "Button save indexes" per controller type at offset 0x80 + type*0x20:
0x80-0x8F   GC:      button indexes (0x70) followed by tag byte offsets (0x80)
0xA0-0xAF   CC       same layout
0xC0-0xCF   Wii      same layout
0xE0-0xEF   Nunchuk  same layout
```

The APressP3 write path computes:
```asm
ButtonTable + 0x80 + (controller_type * 0x20) + option_index  →  tag byte offset
```

For GC option 12 (Analog Shield): `ButtonTable[0x8C]` must contain `0x0D`. For CC option 13: `ButtonTable[0xAD]` must contain `0x0D`.

---

## DataRefTable layout (as of analog shield addition)

```
0x00-0x13  word[5]  MiscData[140..180] pointers
0x14-0x1B  word[2]  Tag pointers (character array, tag location)
0x1C-0x1F  byte[4]  Controller slot values (which port owns each CSS slot; 0xFF = unregistered)
0x20-0x2F  word[4]  Controller object pointers per port
0x30-0x33  word[1]  Per-port Analog Shield enable byte [P1, P2, P3, P4]
0x34-0x37  word[1]  Scratch: port stashed by getPadInput hook for the game-time analog shield hook
```

Extend as needed for future features. `HOOK @ $806828C8` (CSS init) zeroes `0x30(r5)` — extend it if you add more state that needs zero-init.

---

## Runtime pipeline for non-vanilla toggles

The vanilla tag-to-controls pipeline handles button remaps, Tap Jump, and Shake Smash automatically. For a feature Brawl doesn't understand natively (like analog shield), the pipeline is:

1. **CSS write** (APressP3, `setting_XX` case): writes the bit into the tag byte, then immediately mirrors the current value into `DataRefTable+0x30+port` for the current CSS slot.
2. **Tag select** (`HOOK @ $806A0714`, `MuSelctChrList::close`): when a player finalizes a tag, re-reads the tag's byte and populates `DataRefTable+0x30+port`. This covers the case where a player picks an existing tag without editing it.
3. **Match start**: no explicit hook — `DataRefTable+0x30` persists in memory from CSS into gameplay.
4. **Per-frame game hook**: reads the port from `DataRefTable+0x34` and the enable bit from `DataRefTable+0x30+port`, gates the feature accordingly.

### Getting the port at input-time — hard-won lesson

The analog shield hook lives inside `setupPadStatusGc` at `$80048E18`. The port is NOT available there: `r4` is a pad-status struct with no port field, and `r5` is a 12-byte control settings buffer.

The port only exists inside `getPadInput` at `$8004a470` (`lbz r8, 0xb5(r4)`), which then tail-calls `setupPadStatusGc`.

**Approach that DID NOT work:** hooking `$80048bf8` (first instruction of `setupPadStatusGc`) to stash r8 before it gets clobbered internally. This broke `setupPadStatusGc` entirely — likely a GCTRealMate quirk when a hooked instruction is both a function entry point AND the target of a `b` (tail-call) from elsewhere.

**Approach that WORKED:** hook `$8004a4b4` inside `getPadInput`, one instruction before the tail-call. Original op `addi r6, r6, 0x15a` is preserved; r7 is free to use as scratch:

```asm
HOOK @ $8004a4b4
{
  lis r7, 0x806A
  lwz r7, -0x180(r7)     # DataRefTable
  stb r8, 0x34(r7)       # port 0-7
  addi r6, r6, 0x15a     # Original operation
}
```

**Rule of thumb:** avoid hooking the very first instruction of a function that is itself a tail-call target from another function. Hook the caller instead.

---

## Adding a new custom control — step-by-step

The steps below describe adding a new **on/off toggle** (like analog shield). For a multi-value option, skip the toggle-specific parts and follow the pattern used for button remaps.

### 1. Pick a tag byte and bit

Consult the tag layout above. For a global toggle, use unused bits of `0x0D`, `0x0E`, or `0x0F`. For a controller-specific toggle, use unused bits in that section. Do NOT use `0x10-0x13` (pin word).

### 2. Assign menu slots per controller

Decide which controllers get the option. In `InputTable` row 0 (line ~51), increment the slot count for each supporting controller type. Example — analog shield only makes sense for GC and CC:

```asm
byte[4] 12, 13, 9, 12   # was 11, 12, 9, 12
```

Then add a new byte row AFTER the bitmask rows (at offset `0x60`, `0x64`, etc.):

```asm
byte[4] 12, 13, -1, -1  # NewOption slot: GC=12, CC=13, N/A for Wii/Nun
```

Record the offset — you'll reference it in APressP2.

### 3. Update ButtonTable save-indexes

For each supporting controller, add your chosen tag byte offset in the corresponding save-index row. GC's row is around line 111, CC's around line 118. For GC option 12 (your new slot), that's position 4 in the second row (currently `0`):

```asm
byte[8] 0x1C, 0x1D, 0x1E, 0x1F, 0x??, 0, 0, 0   # C-Stick, Y, X, Tap Jump, NewOption
```

Position within the row = `slot - 8`.

### 4. APressP2: route your slot to OnOffSetting (toggles only)

In the `notTheCStick` chain, add a check that branches to `OnOffSetting` when the current option is your new slot. Insert just before the existing Tap Jump check:

```asm
addi r12, r17, 0x??      # your new row offset in InputTable
lbzx r0, r12, r14
cmpw r4, r0; beq- OnOffSetting
```

### 5. APressP3: write logic

Add a `setting_XX` case for your tag byte. The `_XX` refers to the bit mask you want (`_80`, `_40`, `_01`). If a case exists for your mask, reuse it; otherwise create a new one.

Add the dispatch check BEFORE the `cmpwi r30, 12` (Nunchuk slot-12 check), using **tag offset** as the discriminator:

```asm
cmpwi r19, 0x0D; beq setting_01   # your tag byte
```

The tag-offset check MUST come before the slot-12 check, otherwise GC/CC options at slot 12 will be misclassified as Nunchuk Tap Jump.

For a new bit mask, add a labeled block:

```asm
setting_01:
  andi. r12, r12, 0xFE     # clear the target bit
  andi. r6, r6, 0x01       # keep only the on/off value
  b TapShake               # or fall through if next in chain
```

Existing `setting_80` / `setting_40` handle those masks. `TapShake` combines the cleared byte with the setting: `or r6, r6, r12`.

### 6. APressP3: mirror to DataRefTable (only if you need a game-side hook)

After `stb r6, 0(r4)`, add the write to your DataRefTable byte:

```asm
cmpwi r19, 0x0D; bne- noMyOptionUpdate
lbz r6, 0(r4)            # re-read committed byte
andi. r6, r6, 0x01       # extract the bit
lbz r12, 0x57(r26)       # port byte from main pointer (0x31-0x34)
subi r12, r12, 0x31      # port 0-3
lis r11, 0x806A
lwz r11, -0x180(r11)
addi r11, r11, 0x30      # your chosen DataRefTable offset
stbx r6, r11, r12
noMyOptionUpdate:
```

Reserve a DataRefTable byte (add `word[1] 0` or extend an existing entry) and update `HOOK @ $806828C8` to zero it at CSS init.

### 7. Page 2 visuals (`HOOK @ $8069F684`)

For a toggle to display "Off"/"On" instead of a button-map name, add a case in the `loc_0x58` chain BEFORE the `cmpwi r0, 0x33; bne NormalMode`:

```asm
cmpwi r0, 0x0D; beq MyOption
```

And a handler that extracts your bit and jumps to `ToggleText`:

```asm
MyOption:
  andi. r29, r4, 0x01
  b ToggleText
```

### 8. Tag-select population (`HOOK @ $806A0714`)

When a player picks an existing tag (not just edits one), we need to load their setting into DataRefTable. Extend the existing tag-read block to read your byte too:

```asm
lbz r?, 0x0D(r6)    # your tag byte
andi. r?, r?, 0x01  # your bit
addi r?, r5, 0x??   # DataRefTable + your offset
stbx ..., ..., r4   # write for this port
```

### 9. Game-side hook (only for non-vanilla features)

If Brawl doesn't natively read your tag byte at match start, you need:

- A hook near the input pipeline that has access to the port. See the "getting the port at input-time" section — hook in `getPadInput` at `$8004a4b4` to stash the port to `DataRefTable+0x34` before `setupPadStatusGc` is called.
- Your feature's actual hook reads both the stashed port and the per-port enable byte, and skips its logic when disabled.

### 10. Message bins

Add the display strings to `sc_selcharacter.pac` MiscData:
- **MiscData[150]** (GC): message ID = your slot number for that controller
- **MiscData[160]** (CC): same
- **MiscData[170]** (Wii), **MiscData[180]** (Nun): as applicable
- **MiscData[140]** (values): "Off" and "On" are already IDs 10 and 11 — reuse for toggles.

---

## Testing checklist

For every new option:

1. Menu shows the option in Page 2 for the intended controllers.
2. Entering Page 3 shows the correct choices (Off/On for toggles, button-map list otherwise).
3. Selecting a value on Page 3 → returning to Page 2 shows the updated selection.
4. B out of Page 2, re-enter with Y → Page 2 still shows the saved value (tests tag byte persistence).
5. Press Start on Page 1 (pin tag) → re-enter → value still correct (tests that your byte isn't in `0x10-0x13`).
6. In a match, the feature actually behaves per your setting.
7. Toggling the value between matches on the same tag → new behavior takes effect in the next match.
8. Different players (P1-P4) with different settings → each gets their own behavior (tests the per-port DataRefTable pipeline).

---

## References

- Main file: [modfolder/ComboMode+/Source/LegacyTE/CSSCustomControls.asm](../modfolder/ComboMode+/Source/LegacyTE/CSSCustomControls.asm) — analog shield hooks live at the top, CSS custom controls machinery below
- Vanilla `setupPadStatusGc`: DOL address `$80048bf8`, size `0x374` — see [Source/Extras/DebugMode/RSBE01.map](../modfolder/ComboMode+/Source/Extras/DebugMode/RSBE01.map)
- Vanilla `getPadInput`: `$8004a468`, port read at `$8004a470` (`lbz r8, 0xb5(r4)`), tail-call at `$8004a4b8`
