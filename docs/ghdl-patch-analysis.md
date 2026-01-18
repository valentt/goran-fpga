# GHDL Patch Analysis for f32c Compatibility

## Problem Summary

f32c VHDL code uses function calls in case statement choices:

```vhdl
case C_sio_break_detect_delay_ms is
  when iomap_from(ioaddr_sio, ioaddr_sio) to iomap_to(ioaddr_sio, ioaddr_sio) =>
    ...
end case;
```

GHDL reports error: **"choice is not locally static"**

Total errors in f32c XRAM SDRAM build: **43 errors across 4 files**
- glue_xram_vector.vhd: 27 errors
- VGA_textmode.vhd: 7 errors
- video_cache_i.vhd: 2 errors
- usb_rx_phy_emard.vhd: 4 errors (different type)

---

## VHDL Standard Context

### LRM Requirements
According to VHDL LRM (IEEE 1076), case statement choices must be **locally static**:
- Locally static = evaluable at analysis time using only constants, literals, and specific operators
- Function calls are NOT locally static (even if they would return constant values)

### Synthesis Reality
For **synthesis** (not simulation), the requirement is actually weaker:
- Choices only need to be **evaluable at elaboration/synthesis time**
- Pure functions with constant arguments produce constant results
- Synthesis tools like Yosys can handle these perfectly

---

## GHDL Code Analysis

### Error Generation Location
**File:** `src/vhdl/vhdl-sem_expr.adb`
**Lines:** 3091-3096

```ada
if Choice_Staticness /= Locally
  and then Is_Case_Stmt
then
   --  FIXME: explain why
   Error_Msg_Sem (+El, "choice is not locally static");
end if;
```

Key observations:
1. The `FIXME` comment suggests even the developer wanted to document the reasoning
2. `Is_Case_Stmt` distinguishes case statements from aggregates
3. No check for `Flag_Relaxed_Rules` here (unlike other places in the code)

### Existing Relaxed Rules Usage
`Flag_Relaxed_Rules` is already used in 30+ places to relax VHDL strictness:

```
src/vhdl/vhdl-sem_stmts.adb:1582:   elsif Flags.Flag_Relaxed_Rules then
src/vhdl/vhdl-sem_expr.adb:871:    elsif Flag_Relaxed_Rules then
src/vhdl/vhdl-sem_expr.adb:3627:   or else (Flag_Relaxed_Rules
src/vhdl/vhdl-sem_expr.adb:6010:   if Atype_Defined and then not Flag_Relaxed_Rules then
src/vhdl/vhdl-sem_expr.adb:6184:   if Flag_Relaxed_Rules then
```

The flag is set to `True` by default and enabled via `-frelaxed-rules` CLI option.

### Synthesis Path Analysis
**File:** `src/synth/elab-vhdl_types.adb`
**Function:** `Synth_Discrete_Range_Expression` (lines 54-87)

During synthesis:
1. Range limits are evaluated using `Synth_Expression_With_Basetype`
2. Values are checked with `Is_Static_Val` (VALUE staticness, not EXPRESSION staticness)
3. If values evaluate to constants, synthesis proceeds successfully

This means **the synthesis engine CAN handle non-locally-static choices** as long as they evaluate to constant values.

---

## Proposed Solution

### Option 1: Extend Flag_Relaxed_Rules (Recommended)

**Minimal change** - just 1 line added:

```ada
-- In src/vhdl/vhdl-sem_expr.adb, lines 3091-3096:

if Choice_Staticness /= Locally
  and then Is_Case_Stmt
  and then not Flags.Flag_Relaxed_Rules  -- ADD THIS LINE
then
   Error_Msg_Sem (+El, "choice is not locally static");
end if;
```

**Advantages:**
- Minimal code change
- Uses existing, well-tested flag infrastructure
- Already enabled in f32c build via `-frelaxed-rules`
- Consistent with how other relaxations are handled

**Risk:**
- Might allow some code that would fail during synthesis
- But synthesis will catch these with proper error messages

### Option 2: New Synthesis-Specific Flag

Add new flag: `Flag_Synthesis_Mode` or `Flag_Allow_Globally_Static_Choices`

```ada
-- In src/flags.ads:
Flag_Synthesis_Mode : Boolean := False;

-- In src/vhdl/vhdl-sem_expr.adb:
if Choice_Staticness /= Locally
  and then Is_Case_Stmt
  and then not Flags.Flag_Synthesis_Mode
then
   Error_Msg_Sem (+El, "choice is not locally static");
end if;
```

**Advantages:**
- Doesn't affect simulation-only usage
- More explicit about intent

**Disadvantages:**
- Requires adding new flag infrastructure
- More changes needed

### Option 3: Convert Error to Warning

```ada
if Choice_Staticness /= Locally
  and then Is_Case_Stmt
then
   if Flags.Flag_Relaxed_Rules then
      Warning_Msg_Sem (Warnid_Staticness, +El,
                       "choice is not locally static (relaxed for synthesis)");
   else
      Error_Msg_Sem (+El, "choice is not locally static");
   end if;
end if;
```

---

## Implementation Plan

### Phase 1: Verify Hypothesis
1. Create modified GHDL with Option 1 patch
2. Test f32c XRAM SDRAM build
3. Verify synthesis completes without semantic errors
4. Verify synthesized hardware is correct

### Phase 2: Upstream Contribution
1. Fork GHDL on GitHub
2. Create feature branch
3. Implement Option 1 (or Option 2 if GHDL maintainers prefer)
4. Add test cases for non-locally-static choices
5. Submit Pull Request to GHDL

### Phase 3: Alternative - ghdl-yosys-plugin
If GHDL maintainers reject the change, we could:
1. Modify ghdl-yosys-plugin to apply the patch
2. Or maintain a fork specifically for synthesis use

---

## Files to Modify

### Minimal Patch (Option 1)
```
src/vhdl/vhdl-sem_expr.adb  (1 line change)
```

### Full Patch (Option 2)
```
src/flags.ads               (add new flag)
src/options.adb             (add CLI option handling)
src/vhdl/vhdl-sem_expr.adb  (use new flag)
```

---

## Testing Strategy

### Unit Tests
- Add test VHDL file with non-locally-static case choices
- Verify analysis passes with `-frelaxed-rules`
- Verify synthesis produces correct netlist

### Integration Test
- Build f32c XRAM SDRAM version
- All 43 errors should be resolved
- Synthesis should complete successfully

---

## References

- GHDL Source: https://github.com/ghdl/ghdl
- VHDL LRM IEEE 1076-2008, Section 10.9 (Case Statement)
- VHDL LRM IEEE 1076-2008, Section 9.4 (Static expressions)
- f32c Project: https://github.com/f32c/f32c

---

*Analysis by: Hans Weber (FPGA Design Engineer)*
*Date: 2026-01-18*
