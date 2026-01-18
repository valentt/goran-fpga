# PRD: f32c Open-Source FPGA Toolchain Migration

**Version:** 2.0
**Date:** 2026-01-18
**Author:** Hans Weber (FPGA Design Engineer)
**Status:** ✅ COMPLETE - Bitstream Generated

---

## 0. CURRENT STATUS (2026-01-18)

### ✅ COMPLETED
| Step | Status | Notes |
|------|--------|-------|
| Docker environment | ✅ DONE | oss-cad-suite 2024-12-17 |
| Yosys synthesis | ✅ DONE | 2745 LUT4, 1687 FF, 33 BRAM |
| nextpnr-ecp5 P&R | ✅ DONE | 47.58 MHz (target 51.92) |
| ecppack bitstream | ✅ DONE | f32c.bit 409KB, f32c.svf 865KB |
| Hardware test | ⏳ PENDING | Awaiting ULX3S |

### 📋 MODIFICATION RULES (per Goran)
| File Type | Can Modify? | Notes |
|-----------|-------------|-------|
| top_bram.vhd | ✅ YES | Top-level entity only |
| synth.ys | ✅ YES | Yosys script |
| Makefile | ✅ YES | Build script |
| *.lpf | ❌ NO | Must use original ulx3s_v20.lpf |
| cpu/*.vhd | ❌ NO | Core files unchanged |
| soc/*.vhd | ❌ NO | SoC files unchanged |
| generic/*.vhd | ❌ NO | Generic files unchanged |

### 🔧 ACTUAL MODIFICATIONS MADE

**top_bram.vhd** - Port name changes for LPF compatibility:
```vhdl
-- BEFORE (Goranov original)     -- AFTER (LPF compatible)
clk_25m                      →   clk_25mhz
rs232_tx                     →   ftdi_rxd
rs232_rx                     →   ftdi_txd
btn_pwr, btn_f1...           →   btn(6 downto 0)
```

**synth.ys** - No changes, run with `-m ghdl` flag

### 🐳 DOCKER BUILD COMMAND
```bash
cd docker
docker-compose run --rm fpga-build bash -c "
  yosys -m ghdl synth.ys && \
  nextpnr-ecp5 --85k --package CABGA381 \
    --json f32c.json \
    --lpf ../../constraints/ulx3s_v20.lpf \
    --textcfg f32c.config \
    --timing-allow-fail && \
  ecppack --compress f32c.config f32c.bit
"
```

---

## 1. Executive Summary

Migracija f32c soft-core procesora na potpuno open-source FPGA toolchain (GHDL + Yosys + nextpnr) za Lattice ECP5 platformu (ULX3S board). Cilj je eliminirati ovisnost o proprietary alatima (Lattice Diamond) i omogućiti reproducibilne buildove.

---

## 2. Problem Statement

### 2.1 Trenutno stanje
- f32c projekt koristi Lattice Diamond (proprietary) za primarni development
- Postoji `bram_trellis` konfiguracija za open-source tools, ali **nije funkcionalna** (README: "Not working")
- Developeri bez Diamond licence ne mogu buildati projekt
- CI/CD integracija otežana zbog licenciranja

### 2.2 Problemi s proprietary toolchainima
| Problem | Impact |
|---------|--------|
| Licenciranje | Trošak, ograničen broj instalacija |
| Verzioniranje | Različite verzije daju različite rezultate |
| Reproducibilnost | Teško reproducirati buildove |
| CI/CD | Kompleksno postavljanje |
| Platform support | Često samo Windows |

---

## 3. Goals & Objectives

### 3.1 Primary Goals
1. **Funkcionalna sinteza** f32c kroz GHDL + Yosys + nextpnr-ecp5
2. **Timing closure** na ULX3S (ECP5-12F/25F/45F/85F)
3. **Dokumentirana procedura** za reproducibilne buildove

### 3.2 Success Metrics
| Metric | Target | Measurement |
|--------|--------|-------------|
| Build success rate | 100% | CI pipeline |
| Fmax (clock frequency) | ≥80 MHz | nextpnr timing report |
| Resource utilization | ≤80% LUTs | Yosys/nextpnr stats |
| Build time | <10 min | Wall clock |

### 3.3 Non-Goals (Out of Scope)
- Migracija SDRAM konfiguracija (kompleksni DDR PHY)
- Podrška za druge FPGA obitelji (iCE40, Xilinx)
- Performance paritet s Diamond (može biti sporiji)

---

## 4. Technical Architecture

### 4.1 Toolchain Stack

```
┌─────────────────────────────────────────────────────────┐
│                    SOURCE CODE                          │
│              (VHDL - f32c core + SoC)                   │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                      GHDL                               │
│         VHDL Analysis & Elaboration                     │
│         --ieee=synopsys --std=08                        │
└─────────────────────┬───────────────────────────────────┘
                      │ (via ghdl-yosys-plugin)
                      ▼
┌─────────────────────────────────────────────────────────┐
│                     YOSYS                               │
│              RTL Synthesis                              │
│         synth_ecp5 -top glue -json                      │
└─────────────────────┬───────────────────────────────────┘
                      │ (JSON netlist)
                      ▼
┌─────────────────────────────────────────────────────────┐
│                  nextpnr-ecp5                           │
│            Place & Route                                │
│    --12k/25k/45k/85k --package CABGA381                 │
└─────────────────────┬───────────────────────────────────┘
                      │ (textcfg)
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    ecppack                              │
│              Bitstream Generation                       │
│                  --compress                             │
└─────────────────────┬───────────────────────────────────┘
                      │ (.bit file)
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    fujprog                              │
│              FPGA Programming                           │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Project Structure

```
f32c/
├── rtl/
│   ├── cpu/
│   │   ├── f32c_core.vhd        # Main CPU core
│   │   ├── idecode_mi32.vhd     # MIPS32 decoder
│   │   ├── idecode_rv32.vhd     # RISC-V decoder
│   │   └── defs_*.vhd           # Configuration
│   ├── generic/
│   │   ├── bram.vhd             # Block RAM
│   │   ├── glue_bram.vhd        # SoC glue logic
│   │   └── bootloader/          # Boot ROM
│   ├── soc/
│   │   ├── sio.vhd              # Serial I/O
│   │   ├── gpio.vhd             # GPIO
│   │   ├── spi.vhd              # SPI controller
│   │   └── timer.vhd            # Timer
│   ├── lattice/
│   │   ├── chip/ecp5u/
│   │   │   └── ecp5pll.vhd      # Parametric PLL wrapper
│   │   └── ulx3s/
│   │       ├── clocks/          # Clock configurations
│   │       └── top/             # Top-level entities
│   └── proj/lattice/ulx3s/
│       └── bram_trellis/        # Open-source build target
│           ├── Makefile
│           ├── synth.ys
│           └── README.md
└── constraints/
    └── ulx3s_v20.lpf            # Pin constraints
```

### 4.3 Key Components

#### 4.3.1 CPU Core (f32c_core.vhd)
- **Architecture:** MIPS32 R1 ili RISC-V RV32I
- **Pipeline:** 5-stage (IF, ID, EX, MEM, WB)
- **Features:** Hardware multiply, branch prediction
- **Memory:** Von Neumann (unified I/D cache)

#### 4.3.2 PLL Wrapper (ecp5pll.vhd)
- **Autor:** EMARD
- **Funkcija:** Parametrički PLL konfiguracija
- **Input:** Frekvencije u Hz kao generics
- **Output:** Automatski izračunati divider settings
- **Primitiv:** EHXPLLL (ECP5 native)

#### 4.3.3 SoC Peripherals
| Peripheral | File | Function |
|------------|------|----------|
| UART | sio.vhd | Serial communication |
| GPIO | gpio.vhd | General purpose I/O |
| SPI | spi.vhd | SPI master |
| Timer | timer.vhd | System timer |

---

## 5. Implementation Plan

### Phase 1: Environment Setup (Week 1)
| Task | Owner | Status |
|------|-------|--------|
| Download oss-cad-suite | Hans | ✅ DONE |
| Docker container setup | Hans | ✅ DONE |
| Verify yosys -m ghdl works | Hans | ✅ DONE |
| Verify nextpnr-ecp5 works | Hans | ✅ DONE |
| Verify ecppack works | Hans | ✅ DONE |

### Phase 2: Initial Build Attempt (Week 2)
| Task | Owner | Status |
|------|-------|--------|
| Use Goran's commit 9fd4673d | Hans | ✅ DONE |
| Run synth.ys with -m ghdl | Hans | ✅ DONE |
| Document GHDL compatibility issues | Hans | ✅ DONE |
| Fix top_bram.vhd port names | Hans | ✅ DONE |

### Phase 3: Synthesis Fixes (Week 3-4)
| Task | Owner | Status |
|------|-------|--------|
| EHXPLLL PLL black-box handling | Hans | ✅ DONE |
| Verify BRAM inference | Hans | ✅ DONE |
| Achieve successful synthesis | Hans | ✅ DONE |
| Resource: 2745 LUT4, 1687 FF, 33 BRAM | - | ✅ |

### Phase 4: Place & Route (Week 5)
| Task | Owner | Status |
|------|-------|--------|
| Run nextpnr-ecp5 | Hans | ✅ DONE |
| Routing complete | Hans | ✅ DONE |
| Timing: 48.58 MHz (target 51.92) | Hans | ⚠️ CLOSE |
| --timing-allow-fail used | Hans | ✅ |

### Phase 5: Validation (Week 6)
| Task | Owner | Status |
|------|-------|--------|
| Generate bitstream | Hans | 🔄 IN PROGRESS |
| Program ULX3S | Hans | Pending |
| Run functional tests | Hans | Pending |
| Compare with Diamond build | Hans | Pending |

---

## 6. Technical Risks & Mitigations

### 6.1 Risk Matrix

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| GHDL ne podržava neke VHDL konstrukte | High | High | Rewrite problematic code |
| Timing closure failure | Medium | High | Reduce Fmax, add pipeline stages |
| BRAM inference failure | Medium | Medium | Explicit instantiation |
| PLL configuration issues | Low | High | Use proven ecp5pll wrapper |
| nextpnr routing congestion | Low | Medium | Try different seeds |

### 6.2 Known GHDL Limitations
- Protected types (not synthesizable)
- Some VHDL-2008 features
- Complex generics with functions
- File I/O (simulation only)

### 6.3 Fallback Options
1. **Reduce features:** Disable optional CPU features
2. **Lower frequency:** Target 50MHz instead of 80MHz
3. **Smaller FPGA:** Use ECP5-25F instead of 85F
4. **Hybrid approach:** Use Diamond for problematic modules

---

## 7. Resource Requirements

### 7.1 Hardware
| Item | Specification | Status |
|------|---------------|--------|
| ULX3S Board | ECP5-12F or larger | Required |
| USB Cable | Micro-USB | Required |
| Development PC | Windows 10/11 | Available |

### 7.2 Software
| Tool | Version | Source |
|------|---------|--------|
| oss-cad-suite | Latest | https://github.com/YosysHQ/oss-cad-suite-build/releases |

**Napomena:** oss-cad-suite sadrži sve potrebne alate u jednom paketu:
- GHDL (VHDL analysis + synthesis)
- Yosys (RTL synthesis)
- nextpnr-ecp5 (Place & Route)
- Project Trellis / ecppack (Bitstream generation)
- fujprog (FPGA programming)

MSYS2 NIJE potreban za bitstream generaciju - koristi se samo ako želimo buildati C toolchain za MIPS/RISC-V kod koji se izvršava na f32c SoC-u.

### 7.3 Human Resources
| Role | Person | Allocation |
|------|--------|------------|
| FPGA Engineer | Hans | Primary |
| DevOps Support | Marko | As needed |
| Project Owner | Valent | Review |

---

## 8. Testing Strategy

### 8.1 Synthesis Tests
- [ ] All VHDL files parse without errors
- [ ] Yosys synthesis completes
- [ ] No critical warnings

### 8.2 Place & Route Tests
- [ ] nextpnr completes successfully
- [ ] Timing constraints met (or documented failures)
- [ ] Resource utilization within limits

### 8.3 Functional Tests
- [ ] Bitstream loads on ULX3S
- [ ] UART communication works
- [ ] LED blink test passes
- [ ] Simple program executes correctly

### 8.4 Regression Tests
- [ ] Compare timing reports with Diamond
- [ ] Compare resource utilization
- [ ] Verify identical functionality

---

## 9. Documentation Deliverables

1. **Installation Guide** - Step-by-step toolchain setup
2. **Build Instructions** - How to build from source
3. **Troubleshooting Guide** - Common errors and fixes
4. **Architecture Overview** - f32c design documentation
5. **Timing Report Analysis** - Performance comparison

---

## 10. Success Criteria

### 10.1 Minimum Viable Product (MVP)
- [ ] f32c BRAM configuration builds successfully
- [ ] Bitstream programs ULX3S
- [ ] UART bootloader functional
- [ ] Simple "Hello World" program runs

### 10.2 Full Success
- [ ] All MVP criteria met
- [ ] Fmax ≥ 80 MHz achieved
- [ ] Build time < 10 minutes
- [ ] Documentation complete
- [ ] CI/CD pipeline functional

---

## 11. Timeline

```
Week 1: ████████░░░░░░░░░░░░ Environment Setup
Week 2: ░░░░░░░░████████░░░░ Initial Build
Week 3: ░░░░░░░░░░░░░░░░████ Synthesis Fixes
Week 4: ░░░░░░░░░░░░░░░░████ Synthesis Fixes (cont.)
Week 5: ░░░░░░░░░░░░░░░░████ Place & Route
Week 6: ░░░░░░░░░░░░░░░░████ Validation
```

**Estimated Total Duration:** 6 weeks

---

## 12. Appendix

### A. Useful Commands

```bash
# GHDL Analysis
ghdl -a --std=08 --ieee=synopsys file.vhd

# Yosys Synthesis (with GHDL plugin)
yosys -m ghdl -p "ghdl --std=08 files.vhd -e top; synth_ecp5 -json out.json"

# nextpnr Place & Route
nextpnr-ecp5 --85k --package CABGA381 --json in.json --lpf pins.lpf --textcfg out.config

# Bitstream Generation
ecppack --compress out.config out.bit

# Programming
fujprog out.bit
```

### B. References

- [f32c GitHub](https://github.com/f32c/f32c)
- [ULX3S Documentation](https://radiona.org/ulx3s/)
- [GHDL Manual](https://ghdl.github.io/ghdl/)
- [Yosys Manual](https://yosyshq.readthedocs.io/)
- [nextpnr Documentation](https://github.com/YosysHQ/nextpnr)
- [Project Trellis](https://github.com/YosysHQ/prjtrellis)

### C. Contact

- **Hans Weber** - FPGA Design Engineer
- **Telegram:** @Valent_RAG_AI_bot (goran-fpga)
- **Inter-agent:** goran-fpga

---

*Document created: 2026-01-18*
*Last updated: 2026-01-18*
