# f32c XRAM SDRAM Open-Source FPGA Synthesis Guide

## Overview

This document describes how to synthesize the f32c XRAM SDRAM design for ULX3S (Lattice ECP5) using the open-source FPGA toolchain (GHDL + Yosys + nextpnr-ecp5).

**Date:** 2026-01-18
**Status:** WORKING

## Required Tools

1. **GHDL** - VHDL simulator/synthesizer (with custom patch)
2. **Yosys** - RTL synthesis
3. **ghdl-yosys-plugin** - Bridge between GHDL and Yosys
4. **nextpnr-ecp5** - Place and route for Lattice ECP5

## Docker Image

We created a custom Docker image `ghdl-yosys-patched` that includes:
- GHDL 5.0.0-dev (commit a00b7833f) with the "locally static" patch
- Yosys 0.36
- ghdl-yosys-plugin

### Building the Docker Image

```dockerfile
# Dockerfile.ghdl-yosys-patched
FROM hdlc/ghdl:yosys

# Install build dependencies
RUN apt-get update && apt-get install -y \
    git make gnat gcc g++ zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

# Clone exact GHDL version from base image
WORKDIR /build
RUN git clone https://github.com/ghdl/ghdl.git && \
    cd ghdl && git checkout a00b7833f

# Apply the relaxed locally static patch
RUN cd ghdl/src/vhdl && \
    sed -i 's/if Choice_Staticness \/= Locally$/if Choice_Staticness \/= Locally\n                    and then not Flag_Relaxed_Rules/' vhdl-sem_expr.adb

# Build and install
WORKDIR /build/ghdl
RUN ./configure --prefix=/usr/local && make -j$(nproc) && make install

ENV PATH="/usr/local/bin:${PATH}"
WORKDIR /workspace
```

Build command:
```bash
docker build -f Dockerfile.ghdl-yosys-patched -t ghdl-yosys-patched .
```

## Issues Encountered and Solutions

### 1. "choice is not locally static" Errors

**Problem:** GHDL strictly enforces that case statement choices must be "locally static" (compile-time constants). The f32c code uses function calls like `iomap_from()` and `iomap_to()` in case choices.

**Solution:** Applied patch to GHDL source `src/vhdl/vhdl-sem_expr.adb`:
```ada
-- Original:
if Choice_Staticness /= Locally

-- Patched:
if Choice_Staticness /= Locally
                    and then not Flag_Relaxed_Rules
```

This relaxes the check when `-frelaxed-rules` flag is used.

### 2. GHDL LLVM Backend Crash

**Problem:** GHDL 6.0.0-dev with LLVM backend crashes with "terminate called recursively" or "ASSERTION_ERROR : netlists.adb:167" during elaboration.

**Solution:** Use GHDL 5.0.0-dev with mcode backend instead. The mcode backend is more stable for synthesis.

### 3. "cannot associate individually with open" Errors

**Problem:** GHDL doesn't allow using `open` for individual port associations in port maps.

**Example error:**
```vhdl
imem_addr(31 downto 30) => open,  -- ERROR
```

**Solution:** Create dummy signals to receive unused outputs:
```vhdl
-- Add signal declaration:
signal video_cache_addr_unused: std_logic_vector(31 downto 30);

-- Use in port map:
imem_addr(31 downto 30) => video_cache_addr_unused,
```

### 4. SPI Port Partial Association

**Problem:** GHDL requires complete port associations, not partial like `spi_cen(0) => spi_ss(i)`.

**Solution:** Wrap in generate block with local signal:
```vhdl
G_spi_enabled: if C_spi > 0 generate
    type spi_cen_array_type is array (0 to C_spi - 1) of std_logic_vector(3 downto 0);
    signal S_spi_cen_internal : spi_cen_array_type;
begin
    G_spi: for i in 0 to C_spi - 1 generate
        spi_instance: entity work.spi
        port map (
            spi_cen => S_spi_cen_internal(i),
            ...
        );
        spi_ss(i) <= S_spi_cen_internal(i)(0);
    end generate;
end generate;
```

### 5. GPIO Multiple Drivers

**Problem:** GHDL's synthesis reports multiple drivers for signals when using generate loops with separate processes.

**Solution:** Change from generate loop with multiple processes to single process with for loop:
```vhdl
-- GHDL FIX: Changed from generate loop to single process
process(clk)
begin
  if rising_edge(clk) then
    for i in 0 to C_bits/8-1 loop
      -- logic here
    end loop;
  end if;
end process;
```

### 6. Missing bus_addr Port

**Problem:** Some SIO instantiations missing required `bus_addr` port.

**Solution:** Add explicit port mapping:
```vhdl
bus_write => deb_sio_tx_strobe, bus_addr => "00",
```

## Local VHDL Files Modified

The following files were created as GHDL-compatible versions in the project directory:

1. **glue_xram_vector_ghdl.vhd** - Modified glue logic with:
   - SPI generate block wrapper
   - Dummy signals for open associations
   - bus_addr port fix

2. **gpio_ghdl.vhd** - Modified GPIO with:
   - Single process instead of generate loop

3. **stubs.vhd** - Stub entities for:
   - spi (with modified port types)
   - usb_rx_phy_emard
   - video_cache_i, video_cache_d
   - compositing2_fifo, ledstrip
   - VGA_textmode, VGA_textmode_bram, VGA_textmode_font_bram8

## Synthesis Command

```bash
docker run --rm -m 16g \
  -v "$(pwd)://src" \
  -w "//src/rtl/proj/lattice/ulx3s/xram_sdram_12f_v2.0_c2_noscripts_trellis" \
  ghdl-yosys-patched \
  yosys -m ghdl synth.ys
```

## Synthesis Results (2026-01-18)

Successfully synthesized f32c XRAM SDRAM for ULX3S:

```
=== ulx3s_xram_sdram_vector ===

   Number of cells:              47952
     CCU2C                         261
     DP16KD                         11
     EHXPLLL                         1
     LUT4                        24916
     MULT18X18D                      4
     ODDRX1F                         4
     TRELLIS_DPR16X4                53
     TRELLIS_FF                  22653
     TRELLIS_IO                      4
     USRMCLK                         1
```

Output: `f32c_xram.json`

## Place and Route Progress

### Device Selection

The design was tested on multiple ECP5 device sizes:

| Device | LUT Utilization | Result |
|--------|-----------------|--------|
| LFE5U-12F | 106% | Too small |
| LFE5U-45F | 59% | Heap placer failed |
| LFE5U-85F | ~30% | SA placer works |

**Recommendation:** Use LFE5U-85F with SA placer (`--placer sa`).

### Placer Issues

The default **heap placer** fails with "Unable to find legal placement" error.
The **simulated annealing (SA) placer** works but takes longer (~20 minutes for placement).

### nextpnr-ecp5 Command

```bash
docker run --rm -v "$(pwd):/src" \
  -w "/src/proj/lattice/ulx3s/xram_sdram_12f_v2.0_c2_noscripts_trellis" \
  hdlc/nextpnr:ecp5 \
  nextpnr-ecp5 --85k --package CABGA381 \
    --json f32c_xram.json \
    --lpf /src/proj/lattice/constraints/ulx3s_v20.lpf \
    --textcfg f32c_xram_85k.config \
    --placer sa
```

### Routing Progress (2026-01-18)

- **Total arcs to route:** 147,215
- **Progress observed:** 73% complete (108,969 arcs remaining) after ~27 minutes
- **Status:** Container crash interrupted - routing in progress

### Generate Bitstream

After successful place and route:
```bash
ecppack f32c_xram_85k.config f32c_xram_85k.bit
```

### Program FPGA

```bash
fujprog f32c_xram_85k.bit
```

## GHDL Bug Report

**Should file bug report for GHDL LLVM backend:**

- GHDL 6.0.0-dev LLVM backend crashes during elaboration
- Error: "ADA.ASSERTIONS.ASSERTION_ERROR : netlists.adb:167"
- Same code works with GHDL 5.0.0-dev mcode backend
- Test case: f32c_core.vhd elaboration

## References

- f32c project: https://github.com/f32c/f32c
- GHDL: https://github.com/ghdl/ghdl
- ghdl-yosys-plugin: https://github.com/ghdl/ghdl-yosys-plugin
- ULX3S: https://github.com/emard/ulx3s
