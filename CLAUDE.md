# CLAUDE.md - CVA6 RISC-V CPU Core

## Project Overview

CVA6 is a 6-stage, single-issue, in-order RISC-V CPU core (formerly "Ariane") maintained by the OpenHW Group. It implements the RISC-V ISA with I, M, A, C extensions (optionally F, D, V) and supports three privilege levels (M/S/U) for running Unix-like operating systems. The core is industrial-grade and used in production IC designs.

- **License:** Solderpad Hardware License v2.0 / Apache 2.0
- **ISA:** RV32/RV64, configurable via target
- **Pipeline:** Fetch -> Decode -> Issue -> Execute -> Memory -> Writeback/Commit

## Repository Structure

```
core/                       # CVA6 core RTL (SystemVerilog)
  cva6.sv                   # Top-level core module
  Flist.cva6                # Source file manifest for compilation
  include/                  # Config and package files (config_pkg, ariane_pkg, riscv_pkg)
    *_config_pkg.sv         # Per-target configuration packages
  frontend/                 # Fetch pipeline (BTB, BHT, RAS, instruction queue)
  cache_subsystem/          # L1 I$/D$ (WT, WB, or HPDCache variants)
    hpdcache/               # High-Performance Data Cache (submodule)
  cva6_mmu/                 # MMU with TLB and Page Table Walker
  cvxif_example/            # CV-X-IF coprocessor interface example
  cvfpu/                    # Floating-point unit (submodule)
  pmp/                      # Physical Memory Protection

corev_apu/                  # Application Processing Unit (core + peripherals)
  src/ariane.sv             # APU top-level wrapper
  tb/                       # Testbenches (ariane_testharness, ariane_tb.cpp)
  fpga/                     # Xilinx FPGA flow
  altera/                   # Intel/Altera FPGA flow
  bootrom/                  # Boot ROM
  clint/                    # Core Local Interrupt Controller
  rv_plic/                  # Platform-Level Interrupt Controller
  riscv-dbg/                # Debug module (submodule)
  register_interface/       # APB register interface (submodule)
  instr_tracing/            # Instruction tracing infrastructure

verif/                      # Verification infrastructure
  sim/                      # Simulation environment
    cva6.py                 # Main test orchestration script
    setup-env.sh            # Environment setup
  regress/                  # Regression test scripts
  tests/                    # Test cases and test lists (YAML)
  bsp/                      # Board Support Package for test programs
  tb/                       # Core-level testbench
  core-v-verif/             # Core-V verification framework (submodule)

docs/                       # Sphinx/RST documentation
vendor/                     # Third-party IP (pulp-platform AXI, common_cells, etc.)
common/                     # Shared utilities
config/                     # RISC-V config generation
ci/                         # CI scripts and test lists
pd/                         # Physical design / synthesis
spyglass/                   # Lint configuration
util/                       # Utility scripts (flist_flattener.py, toolchain-builder)
tutorials/                  # User tutorials (simulation, FPGA, ASIC)
```

## Build System

The primary build system is **GNU Make** (top-level `Makefile`). **Bender** (`Bender.yml`) is used as a dependency manifest.

### Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `RISCV` | **Required.** Path to RISC-V toolchain installation |
| `CVA6_REPO_DIR` | Repository root (auto-set if unset) |
| `target` | Hardware config (default: `cv64a6_imafdc_sv39`) |
| `TARGET_CFG` | Alias for target config (derived from `target`) |
| `HPDCACHE_DIR` | HPDCache directory (auto-set) |
| `SPIKE_TANDEM` | Enable Spike tandem verification when set to 1 |
| `DV_SIMULATORS` | Simulator selection for `cva6.py` |
| `DV_TARGET` | Target config for `cva6.py` |
| `NUM_JOBS` | Parallel build jobs (default: 1) |
| `BOARD` | FPGA board: `genesys2`, `kc705`, `vc707`, `nexys_video` |

### Hardware Configurations (targets)

Select with `make target=<config>`:

- **64-bit:** `cv64a6_imafdc_sv39` (default), `cv64a6_imafdc_sv39_wb`, `cv64a6_imafdc_sv39_hpdcache`, `cv64a6_imafdc_sv39_hpdcache_wb`, `cv64a6_imafdch_sv39`, `cv64a6_imafdcv_sv39`
- **32-bit:** `cv32a65x`, `cv32a60x`, `cv32a6_imac_sv0`, `cv32a6_imac_sv32`, `cv32a6_imafc_sv32`

Targets starting with `cv64` set `XLEN=64`; all others set `XLEN=32`. Each target has a corresponding `core/include/<target>_config_pkg.sv`.

### Main Make Targets

```bash
# Verilator (open-source, primary for CI)
make verilate target=<config>              # Compile RTL with Verilator
make sim-verilator elf_file=<path>         # Run simulation with Verilator

# QuestaSim
make build target=<config>                 # Compile RTL for QuestaSim
make sim elf_file=<path>                   # Run QuestaSim simulation

# VCS
make vcs_build target=<config>             # Compile with VCS
make vcs elf_file=<path>                   # Run VCS simulation

# Xcelium
make xrun_comp                             # Compile with Xcelium
make xrun_sim elf=<path>                   # Simulate with Xcelium

# FPGA
make fpga BOARD=genesys2                   # Xilinx FPGA bitstream
make altera                                # Intel/Altera FPGA bitstream

# Tests (QuestaSim)
make run-asm-tests                         # RISC-V assembly tests
make run-benchmarks                        # RISC-V benchmarks

# Tests (Verilator)
make run-asm-tests-verilator
make run-benchmarks-verilator

# Cleanup
make clean                                 # Remove all build artifacts
```

### Verification via cva6.py

The main test orchestration uses `verif/sim/cva6.py`:

```bash
# Setup environment first
source verif/sim/setup-env.sh
export DV_SIMULATORS=veri-testharness,spike
export DV_TARGET=cv32a65x

# Run regression scripts
bash verif/regress/smoke-tests-cv32a65x.sh
bash verif/regress/dv-riscv-arch-test.sh
bash verif/regress/cv64a6_imafdc_tests.sh
```

Available simulators for `DV_SIMULATORS`: `veri-testharness`, `vcs-testharness`, `vcs-uvm`, `spike`

Key regression scripts in `verif/regress/`:
- `smoke-tests-cv32a65x.sh`, `smoke-tests-cv64a6_imafdc_sv39.sh` - Quick smoke tests
- `dv-riscv-arch-test.sh` - Architecture compliance
- `cv32a6_tests.sh`, `cv64a6_imafdc_tests.sh` - Target-specific test suites
- `dhrystone.sh`, `coremark.sh` - Performance benchmarks

## CI/CD

### GitHub Actions (`.github/workflows/`)

- **`ci.yml`**: Runs on push/PR. Builds toolchain, Verilator, and Spike; runs riscv-arch-tests and target-specific tests for both 32-bit and 64-bit configs with Spike tandem verification.
- **`verible.yml`**: Checks Verible formatting on PRs for files in `core/`.
- **`bender-up-to-date.yml`**: Validates Bender manifest consistency.

### GitLab CI (`.gitlab-ci.yml`)

Stages: `setup` -> `light tests` -> `heavy tests` -> `backend tests` -> `find failures` -> `report`

Used for extended verification including VCS/UVM, gate-level simulation, FPGA boot, and coverage merging.

## Coding Conventions

### SystemVerilog Style

- **Style guide:** [lowRISC Style Guides](https://github.com/lowRISC/style-guides/)
- **Formatter:** Verible (`verible-verilog-format`)
  ```bash
  verible-verilog-format --inplace $(git ls-tree -r HEAD --name-only core | grep '\.sv$' | grep -v '^core/include/std_cache_pkg.sv$' | grep -v cvfpu)
  ```
- Formatting is enforced by CI on all `.sv` files in `core/` (excluding `std_cache_pkg.sv` and `cvfpu` submodule)

### Naming Conventions

- **Modules:** snake_case (e.g., `load_store_unit`, `branch_unit`)
- **Signals:** snake_case (e.g., `fetch_entry_valid`, `commit_ack`)
- **Types:** `_t` suffix (e.g., `scoreboard_entry_t`, `exception_t`, `cf_t`)
- **Parameters:** UPPER_CASE or CamelCase
- **Packages:** `_pkg` suffix (e.g., `ariane_pkg`, `riscv_pkg`, `config_pkg`)
- **Config packages:** `<target>_config_pkg` (e.g., `cv32a65x_config_pkg`)

### Git Commit Messages

- 50-character subject line limit, imperative mood ("Add feature" not "Added feature")
- Capitalize subject line, no trailing period
- Blank line between subject and body
- Wrap body at 72 characters
- Explain what and why, not how

### Languages Used

- **SystemVerilog:** Core RTL, testbenches (primary)
- **Python 3:** Build orchestration (`cva6.py`), test generation, utilities
- **C++ (C++17):** DPI interface, testbench drivers (`ariane_tb.cpp`)
- **VHDL:** Legacy UART and DPTI IP
- **Bash:** Regression and installation scripts

## Architecture Reference

### Pipeline Stages

1. **Frontend** (`core/frontend/`): Instruction fetch with BTB, BHT, RAS, instruction queue
2. **Decode** (`core/decoder.sv`, `core/compressed_decoder.sv`): Instruction decoding including C-extension decompression
3. **Issue** (`core/issue_stage.sv`): In-order issue with scoreboard tracking and RAW hazard detection
4. **Execute** (`core/ex_stage.sv`): ALU, branch unit, multiplier, serial divider, FPU, load/store unit
5. **Memory**: Cache subsystem (3 variants: Write-Through, Write-Back, HPDCache)
6. **Commit** (`core/commit_stage.sv`): In-order retirement, CSR updates

### Key Modules

| Module | File | Purpose |
|--------|------|---------|
| Core top | `core/cva6.sv` | Top-level core with full parameterization |
| APU top | `corev_apu/src/ariane.sv` | Core + peripherals wrapper |
| Decoder | `core/decoder.sv` | Main instruction decoder |
| CSR regfile | `core/csr_regfile.sv` | Control/Status Registers |
| MMU | `core/cva6_mmu/cva6_mmu.sv` | Memory Management Unit |
| Scoreboard | `core/scoreboard.sv` | Instruction tracking for issue/commit |
| I-Cache | `core/cache_subsystem/cva6_icache.sv` | Instruction cache |
| Testharness | `corev_apu/tb/ariane_testharness.sv` | Simulation test harness |

### Configuration System

Configuration uses a two-level system:
1. **`config_pkg.sv`**: Master `cva6_cfg_t` struct with 100+ parameters (XLEN, extensions, cache sizes, TLB entries, etc.)
2. **`<target>_config_pkg.sv`**: Variant-specific parameter sets that instantiate the master struct

The `build_config_pkg.sv` provides a `build_config()` function to combine configs with runtime overrides.

## Contribution Guidelines

Per CONTRIBUTING.md:

- New features must be **optional and disabled by default** (configurable via SV parameters)
- Contributions must **pass CI** with the feature both enabled and disabled
- Code coverage must **not be impacted** when the feature is disabled
- Each contribution needs its own **regression test** for CI integration
- RTL in `core/` must be **Verible-formatted**
- Contact the CVA6 team early for major contributions (info@openhwgroup.org)
- Commit to **2-year maintenance** for contributions
- Custom ISA extensions should use the **CV-X-IF coprocessor interface**, not modify the core

## File Manifest

- `core/Flist.cva6` - Core RTL source file list (used by all simulators via `-f` flag)
- `Bender.yml` - Bender dependency manifest with target-specific source selection
- `verilator_config.vlt` - Verilator lint/warning configuration

## Submodules

Key submodules (initialize with `git submodule update --init --recursive`):

- `core/cvfpu` - Floating-point unit
- `core/cache_subsystem/hpdcache` - High-Performance Data Cache
- `corev_apu/riscv-dbg` - Debug module
- `verif/core-v-verif` - Core-V verification framework
- `verif/sim/dv` - Google riscv-dv instruction generator
- `vendor/pulp-platform/*` - AXI, common cells, tech cells
