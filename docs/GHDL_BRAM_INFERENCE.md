# GHDL/Yosys BRAM Inference - Key Findings

**Date:** 2026-01-19
**Project:** f32c sdram_min migration to open-source toolchain

## Critical Discovery

GHDL/Yosys **NE MOŽE** inferirati memoriju iz **inlined** nizova deklariranih kao `signal` unutar velike arhitekture.

GHDL/Yosys **MOŽE** inferirati memoriju iz **zasebnih entiteta** koji koriste istu VHDL strukturu.

## Primjer - NE RADI (f32c_core.vhd)

```vhdl
-- Unutar velike arhitekture f32c_core
type reg_type is array(0 to 31) of std_logic_vector(31 downto 0);
signal M_R1, M_R2, M_RD: reg_type;  -- SINTETIZIRA SE KAO FF!

type bptrace_type is array(0 to 8191) of std_logic_vector(1 downto 0);
signal M_bptrace: bptrace_type;  -- SINTETIZIRA SE KAO FF!
```

**Rezultat:** 22,164 FF umjesto ~1,500 FF (13x više!)

## Primjer - RADI (bptrace.vhd, reg1w2r.vhd)

```vhdl
-- Zasebni entitet bptrace.vhd
entity bptrace is
    port (din, dout, rdaddr, wraddr, re, we, clk...);
end bptrace;

architecture Structure of bptrace is
    type bptrace_type is array(0 to 8191) of std_logic_vector(1 downto 0);
    signal M_bptrace: bptrace_type;  -- ISPRAVNO INFERIRA BRAM!
begin
    process(clk)
    begin
        if rising_edge(clk) then
            if we = '1' then M_bptrace(addr) <= din; end if;
            if re = '1' then dout <= M_bptrace(addr); end if;
        end if;
    end process;
end Structure;
```

**Rezultat:** 1,687 FF (ispravno!)

## Usporedba resursa

| Projekt | LUT4 | FF | BRAM | LUTRAM |
|---------|------|-----|------|--------|
| bram_trellis (pipeline.vhd + zasebni entiteti) | 2,771 | 1,687 | 33 | 32 |
| sdram_min v2 (f32c_core.vhd inlined) | 26,116 | 22,164 | 11 | 53 |
| Diamond (referenca) | 3,504 | 1,605 | 13 | - |

## Rješenje

1. **Koristi zasebne entitete za memorije** - `bptrace.vhd`, `reg1w2r.vhd`, `bram_true2p_1clk.vhd`
2. **Koristi `pipeline.vhd`** umjesto `f32c_core.vhd` (pipeline koristi zasebne entitete)
3. **Za cache koristi `cache_ghdl.vhd`** - verzija bez G_icache_2k bloka (GHDL ne podržava parcijalni port mapping s ne-statičkim indeksima)

## GHDL ograničenja

1. **Parcijalni port mapping** - indeksi moraju biti "locally static"
   ```vhdl
   -- NE RADI:
   addr_a(C_i_addr_bits - 2) => '0',

   -- RADI:
   signal temp_addr: std_logic_vector(...);
   temp_addr <= '0' & ...;
   addr_a => temp_addr,
   ```

2. **Memorijska inferencija** - bolje radi sa zasebnim entitetima nego s inline nizovima

## Async vs Sync Read

- **Async read** → samo LUTRAM ili FF (Yosys dokumentacija)
- **Sync read** → može BRAM

Register file (M_R1, M_R2, M_RD) ima async read = LUTRAM (OK, to je ispravno za CPU)
Branch predictor (M_bptrace) ima sync read = trebao bi biti BRAM

## GHDL Undriven Ports Issue

**KRITIČNO:** GHDL synthesis NE dozvoljava **undriven portove** na record tipovima!

### Problem
```vhdl
type sdram_port_array is array(0 to 15) of sdram_port_type;  -- 16 elemenata
signal sdram_bus: sdram_port_array;

-- Samo koristimo ports 0 i 1, portovi 2-15 su undriven
sdram_bus(0).addr_strobe <= ...;
sdram_bus(1).addr_strobe <= ...;
-- Ports 2-15 NISU assignani → GHDL ERROR: "driving constant bits"
```

### Error poruka
```
ERROR: Cell port sdram.mpbus is driving constant bits
```

### Rješenje
1. **Smanji array size** na točan broj portova koji se koriste
2. Ili **assignaj default vrijednosti** svim nekorištenim portovima

### Primjer fix-a
```vhdl
-- sdram_pack_ghdl.vhd
type sdram_port_array is array(0 to 1) of sdram_port_type;  -- Samo 2 porta!
```

## Reference

- [Yosys Memory Handling](https://yosyshq.readthedocs.io/projects/yosys/en/latest/using_yosys/synthesis/memory.html)
- [GHDL-Yosys-Plugin Issue #36](https://github.com/ghdl/ghdl-yosys-plugin/issues/36)
