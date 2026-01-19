# f32c GHDL/Yosys Migration Status

**Date:** 2026-01-19
**Project:** f32c sdram_min for ULX3S
**Target:** LFE5U-12F (ECP5)

## Summary

Migracija f32c projekta s Lattice Diamond na open-source toolchain (GHDL + Yosys + nextpnr-ecp5).

## Key Discovery: BRAM Inference Problem

### Problem
GHDL/Yosys **NE MOŽE** inferirati memoriju iz **inlined** nizova u velikoj arhitekturi.

### Dokaz

| Verzija | LUT4 | FF | BRAM |
|---------|------|-----|------|
| f32c_core.vhd (inlined) | 26,116 | 22,164 | 11 |
| pipeline.vhd (separate entities) | 2,771 | 1,687 | 33 |
| Diamond referenca | 3,504 | 1,605 | 13 |

### Rješenje
Koristiti **pipeline.vhd** sa **zasebnim entitetima**:
- `bptrace.vhd` - branch predictor trace
- `reg1w2r.vhd` - register file
- `bram_true2p_1clk.vhd` - true dual port BRAM

## Created Files

### 1. cache_ghdl.vhd
**Path:** `projects/f32c-valent/rtl/cpu/cache_ghdl.vhd`

GHDL-kompatibilna verzija cache.vhd:
- Uklonjen `G_icache_2k` generate blok (GHDL ne podržava parcijalni port mapping s ne-statičkim indeksima)
- Koristi cache size >= 4KB

### 2. glue_sdram_min_ghdl_v4.vhd
**Path:** `projects/f32c-valent/rtl/proj/lattice/ulx3s/sdram_min_trellis_v2/glue_sdram_min_ghdl_v4.vhd`

GHDL-kompatibilna verzija glue_sdram_min.vhd:
- Dodana `debug_bus_out` signal za fix partial open port mapping
- Instancira `cache` entity (iz cache_ghdl.vhd)

### 3. synth_v4.ys
**Path:** `projects/f32c-valent/rtl/proj/lattice/ulx3s/sdram_min_trellis_v2/synth_v4.ys`

Synthesis script koji koristi:
- pipeline.vhd + bptrace.vhd + reg1w2r.vhd
- cache_ghdl.vhd
- glue_sdram_min_ghdl_v4.vhd

## GHDL Limitations

### 1. Parcijalni port mapping
```vhdl
-- NE RADI:
addr_a(C_i_addr_bits - 2) => '0',

-- RADI:
signal temp_addr: std_logic_vector(...);
temp_addr <= '0' & ...;
addr_a => temp_addr,
```

### 2. Partial open
```vhdl
-- NE RADI:
bus_out(8) => deb_sio_rx_done, bus_out(9) => open,

-- RADI:
bus_out => debug_bus_out,
...
deb_sio_rx_done <= debug_bus_out(8);
```

### 3. Memory inference
- Inlined arrays u velikoj arhitekturi → sintetizira se kao FF
- Zasebni entiteti → ispravno inferira BRAM/LUTRAM

## Current Status

**Problem:** Synthesis s flatten pasom javlja error:
```
ERROR: Cell port sdram.mpbus is driving constant bits
```

**Sljedeći koraci:**
1. Debugirati SDRAM controller compatibility
2. Možda potreban fix u sdram_mz.vhd

## Reference Files

### Working (bram_trellis)
- Uses: pipeline.vhd, bptrace.vhd, reg1w2r.vhd
- Result: 2,771 LUT4, 1,687 FF ✓

### Not Working (sdram_min v2)
- Uses: f32c_core.vhd (inlined)
- Result: 26,116 LUT4, 22,164 FF ✗

## Documentation

- [GHDL_BRAM_INFERENCE.md](GHDL_BRAM_INFERENCE.md) - Detaljni opis BRAM inference problema
