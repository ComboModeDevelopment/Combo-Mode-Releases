# Injecting PSA Events into fighter.pac (Global Action Modifications)

How to modify a shared fighter.pac action for every character at once by writing PSA event bytes directly into fighter.pac's loaded memory. Worked example: the "Global TurnRun Attack Cancel" ([TurnRunAttackCancel.asm](../modfolder/ComboMode+/Source/Community/Injects/MoveDef/TurnRunAttackCancel.asm)).

Companion doc to [css_custom_controls.md](css_custom_controls.md) — that one covers a UI + per-player system layered on top of tag data. This one covers direct fighter.pac PSA modification, which is simpler in concept but has its own set of traps.

---

## The problem this pattern solves

You want a moveset change that applies to every character (e.g. "make TurnRun cancelable into attacks"). The naive approach is to open every `Fit<Name>.pac` in PSAC and edit each character's Sub Action manually — but that's dozens of files, each edit is error-prone, and it doesn't propagate to fighters added later. The better approach: patch the shared logic in `fighter.pac`.

`fighter.pac` contains **Actions** (state-machine logic that decides which Sub Action runs and when to exit). Every character executes the same fighter.pac Actions, so one inject there affects everyone.

**Actions vs Sub Actions:**
- **Actions** live in `fighter.pac`, shared globally. Identified by hex ID (e.g. Action 0x08 = TurnRun, 0x12 = SquatWait).
- **Sub Actions** live in per-character `Fit<Name>.pac` files. Identified by hex ID too, but each character can put different content at the same Sub Action ID (Falcon's Sub Action 0x12 has different animation/timing/events than Fox's).

The Action calls a Sub Action via `Change Sub Action: Sub Action=<hex>`. All characters share the Sub Action ID → each character's own copy runs.

---

## Prerequisite knowledge

**Fighter.pac loads at `0x80F9FC20` in memory.** Confirmed by [ScrewAttack.asm:44](../modfolder/ComboMode+/Source/Community/Injects/MoveDef/ScrewAttack.asm#L44) (`0x80F9FC20 + 0xE84C`) and matched by every other MoveDef inject writing to `0x80FA____`–`0x80FC____`.

Runtime address for any byte in fighter.pac = `0x80F9FC20 + <file offset>`.

**PSAC dropdowns use hex numbering.** "Action 12" in PSAC = `0x12` = decimal 18 = SquatWait, NOT decimal 12. Same for Sub Action IDs.

**Common Action IDs** (from [Physics.asm:253-256](../modfolder/ComboMode+/Source/ProjectM/Modifier/Physics.asm#L253-L256) and [debug.txt:457-479](../modfolder/ComboMode+/Source/Extras/DebugMode/debug.txt#L457-L479)):

```
0x00  Wait          0x08  TurnRun       0x10  JumpSquat
0x01  Walk          0x09  TurnRunBrake  0x11  Jump
0x02  WalkBrake     0x0A  JumpSquat     0x12  SquatWait
0x03  Dash          0x0B  Jump          ...
0x04  Run           0x0C  JumpAerial
0x05  RunBrake      0x0D  Multijump
0x06  Turn1         0x0E  Fall
0x07  Turn2         0x0F  FallAerial
```

---

## The PSA event format

Every PSA event is stored as **2 words (8 bytes)**:

```
Word 1:  EE EE PP FF     Event ID  (EE=category, PP=param count, FF=flags)
Word 2:  arg-pointer     Points to argument list, OR value inline for some events
```

Argument lists are variable-length; each argument is 2 words:

```
Word 1:  TT TT TT TT     Type indicator (0=int, 1=scalar, 2=pointer, 3=bool, 4=variable, 5=requirement)
Word 2:  VV VV VV VV     Value
```

**Common event IDs** (confirmed from Hitstun.asm, Moonwalk.asm, Grabs.asm):

| Event ID | Meaning | Params |
|---|---|---|
| `0x00020000` | nop | 0 |
| `0x00070100` | Sub Routine (Call) | 1 (indirect — see gotcha below) |
| `0x00080000` | Return / End Of Script | 0 |
| `0x000A0100` | If Requirement | 1 |
| `0x000A0200` | If Requirement Value | 1 |
| `0x000A0400` | If Comparison | 4 |
| `0x000B0200` | And (compound condition) | 1 |
| `0x000F0000` | End If | 0 |
| `0x020A0100` | Allow Specific Interrupt | 1 (value: interrupt type) |
| `0x02040100` | External Subroutine | 1 |
| `0x02040400` | External Subroutine Call | 4 |
| `0x12000200` | Basic Variable Set | 2 |
| `0x12040100` | Basic Variable Decrement | 1 |
| `0x64000000` | Allow Interrupt (generic) | 0 |

---

## The critical gotcha: Sub Routine calls are INDIRECT

This is the trap that cost hours of debugging. `systemCmdFuncCall` at [0x8013B3CC](https://ghidra) reads the target address through TWO layers of indirection:

```c
// pseudocode from Ghidra decompilation:
getArg(current_event, &local_38, 0);        // local_38 = event's word 2 (arg pointer)
puVar2 = *(void **)(local_38 + 4);           // target = *(arg_ptr + 4)
param_1->field3_0xc = puVar2;                // jump to target
```

So a Sub Routine event's word 2 must point to an **arg descriptor**, not directly at the subroutine body:

```asm
# In fighter.pac:
word 0x00070100; word arg_descriptor_addr

# The arg descriptor (8 bytes, anywhere in memory):
word 2; word subroutine_target_addr
       # ^ type=2 (pointer), value=actual target
```

**Wrong** (what I did first — it built cleanly and the interpreter reached the event, but jumped to garbage):
```asm
word 0x00070100; word MY_SUBROUTINE     # ← MY_SUBROUTINE is NOT an arg descriptor
```

**Right**:
```asm
word 0x00070100; word MY_ARG_DESC
...
MY_ARG_DESC:
    word 2; word MY_SUBROUTINE
```

Verify by looking at Moonwalk.asm — its Sub Routine call at fighter.pac `$80FAC594` points at `PSA_Off+0x70`, which contains exactly `word 2; word PSA_Off+0x78` (the actual subroutine target).

---

## The other gotchas (short version)

**GCTRealMate syntax quirk.** Inside a `CODE @` block, `word $80543000` (bare `$`-prefixed literal) can cause the assembler to stall or produce broken output. `$` prefix works on the `CODE @` address itself and on `HOOK @`, but NOT inside `word` statements. Use an alias instead:

```asm
.alias PSA_Off = 0x80542F00
...
word 0x00070100; word PSA_Off+0x20    # works
# word 0x00070100; word $80542F20     # may stall the build
```

**Address collisions with other injects don't produce a clean error.** GCTRealMate silently stalls or drops the GCT if two `CODE @` blocks target the same address. Before picking a DOL free-space address, grep the source tree for `CODE @ \$<region>` and known-used PSA_Off aliases.

**"Free" doesn't mean free.** Many addresses in `0x8054xxxx` that no inject writes to are still populated by the game at runtime — e.g., `0x80543000` gets filled with 8 pointers to `0x8055Cxxx` (a game vtable). Always verify a candidate address in Dolphin's memory viewer during gameplay: if it's `0000 0000` across at least `0x40` bytes AFTER the match has started, it's safe. If it has any non-zero pattern, avoid it.

**Fighter.pac uses file-relative arg pointers on disk that get relocated to absolute addresses at load time.** In a hex editor, an arg pointer looks like `00 00 D8 5C` (file offset). At runtime, that same byte reads as `80 FA D5 7C` (absolute). Your inject writes to runtime memory, so use absolute addresses.

---

## The recipe: adding an event to a shared Action

Using the TurnRun Attack Cancel as the worked example.

### Step 1 — Find the Action's offset in PSAC

Open `fighter.pac` in PSAC. Events tab → Actions → pick the Action from the dropdown. The status bar shows an `Offset` field (e.g. `0xD9FC` for Action 0x08).

Runtime address for the action's first byte = `0x80F9FC20 + <offset>`.

### Step 2 — Find the exact byte position of the sacrificial event

Open `fighter.pac` in a hex editor. Jump to the offset from step 1. Scan forward for the byte pattern of the event you want to replace. For a `Basic Variable Set`, that's `12 00 02 00 XX XX XX XX`.

For TurnRun, the second `Basic Variable Set: RA-Basic[0] = IC-Basic[23001]` is at file offset `0xDABC` → runtime `0x80FAD6DC`. Note the next 4 bytes (`80 FA D5 FC` after relocation, or the file-offset form `00 00 D9 DC` on disk) — that's the arg pointer we'll need to preserve.

### Step 3 — Pick a DOL free-space address

Grep the codebase for existing `CODE @ \$8054` usages to see what's occupied. Common convention: `0x80540xxx`–`0x8054Cxxx` is the "safe" GCT region, but individual slots inside it are heavily contested and some are game-initialized.

Verify your candidate is empty in Dolphin memory viewer during gameplay. `0x80542F00` is a known-good hole (256 bytes of zeros) as of this writeup.

### Step 4 — Write the inject

Two `CODE @` blocks: one plants the subroutine in DOL free-space, one replaces the sacrificial event in fighter.pac with a Sub Routine call.

```asm
.alias PSA_Off = 0x80542F00

# Custom subroutine body + arg descriptors
CODE @ $80542F00
{
    # +0x00: Preserve the sacrificed event, reusing its vanilla arg pointer
    word 0x12000200; word 0x80FAD5FC   # Basic Variable Set → vanilla args

    # +0x08: The new event we're inserting
    word 0x020A0100; word PSA_Off+0x28 # Allow Specific Interrupt → arg at +0x28

    # +0x10: Return (must always end with this so control returns to the caller)
    word 0x00080000; word 0

    # +0x18: padding to align next section on 0x08
    word 0; word 0

    # +0x20: Arg descriptor for the Sub Routine call (indirect pattern!)
    word 2; word PSA_Off               # type=2 (pointer), value=subroutine body

    # +0x28: Arg descriptor for Allow Specific Interrupt
    word 0; word 0x04                  # type=0 (int), value=4 (Ground Attack)
}

# Replace the sacrificial event in fighter.pac with a Sub Routine call
CODE @ $80FAD6DC
{
    word 0x00070100; word PSA_Off+0x20 # Sub Routine, arg desc at PSA_Off+0x20
}
```

Save as a new `.asm` file under `Source/Community/Injects/MoveDef/` and add `.include MoveDef/<yourfile>.asm` to `Source/Community/Injects/MDEF.txt`.

### Step 5 — Build and verify in Dolphin

Run `.\GCTRealMate.exe .\Source\Community\Injects\MDEF.txt`. If it stalls without producing a GCT, you have one of the gotchas — check address collision, `$`-prefix in `word` statements, or Unicode in comments.

If it builds, load the GCT and boot into a match. In Dolphin's memory viewer:

- `0x80FAD6DC` should read: `00 07 01 00 80 54 2F 20`
- `0x80542F00` should read the full 48-byte layout from the inject

To verify the subroutine actually executes, set a **memory READ breakpoint** at `0x80542F00`. If it fires during the action, the Sub Routine call is being followed correctly.

---

## Debugging: what if the subroutine doesn't fire?

**`0x80FAD6DC` READ breakpoint fires but `0x80542F00` doesn't** → the Sub Routine indirect-arg-pointer trap. Check that your event's word-2 points at an arg descriptor with the shape `word 2; word target`, and that the descriptor's `+4` word holds the actual subroutine address.

**Neither breakpoint fires** → the Action doesn't reach that offset in practice. Either (a) an earlier event in the Action exits (e.g. Change Action to Fall), (b) fighter.pac gets reloaded and your inject was wiped, or (c) the Action ID is wrong and TurnRun runs from somewhere else. Set a code breakpoint on the C++ status handler (`ftStatusUniqProcess<Name>::initStatus` — look up in `RSBE01.map` or `ghidra_functions.txt`) to confirm the state entry.

**Both fire but no gameplay change** → the event you're using doesn't do what you expect in the current context. `Allow Specific Interrupt` works from action context AND subaction context (confirmed for TurnRun). Other events may only work in one or the other — check by comparing to a known-working per-fighter PSAC edit.

---

## Reference: `Allow Specific Interrupt` types

Value passed to `020A0100`. Determined empirically from PSAC — click the event, view the Value field:

| Value | Type |
|---|---|
| `0x04` | Ground Attack |
| (others — Air Special, Footstool, Wall Jump, Air Jump, etc. appear in [Hitstun.asm:151-158](../modfolder/ComboMode+/Source/Community/Injects/MoveDef/Hitstun.asm#L151-L158) with arg pointers you can hex-dump) |

To confirm any value: in PSAC, add an `Allow Specific Interrupt` event to any scratch subaction, pick the type from the dropdown, then click the event and read the raw hex from the properties view.

---

## Related resources

- [Moonwalk.asm](../modfolder/ComboMode+/Source/Community/Injects/MoveDef/Moonwalk.asm) — the canonical "PSA data planted in DOL + call replaces fighter.pac event" template
- [Hitstun.asm](../modfolder/ComboMode+/Source/Community/Injects/MoveDef/Hitstun.asm) — lots of `020A0100` (Allow Specific Interrupt) usage, good reference for arg encoding
- [Grabs.asm](../modfolder/ComboMode+/Source/Community/Injects/MoveDef/Grabs.asm) — clean Sub Routine call examples
- [Constants.asm](../modfolder/ComboMode+/Source/Community/Injects/fighter/Constants.asm) — one-word attribute tweaks inside fighter.pac
- Ghidra `interpretCmd/[acCmdInterpreter]` at `0x80139748` — the PSA event dispatcher itself. Decompile it to understand event dispatch.
- Ghidra `systemCmdFunc*` at `0x8013A914`–`0x8013C4XX` — one function per system command handler, listed in [ghidra_functions.txt](../../.claude/projects/c--Users-pvstv-Desktop-Combo-Mode-Plus/memory/ghidra_functions.txt) if you need to check what an event actually does.
