# embedded-dsp-firmware

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**About:** Bare-metal DSP firmware on STM32H753ZI — no HAL, register-level, CMSIS-DSP. Part of a three-board heterogeneous system: STM32H7 → RPi4B → PYNQ-Z2.

Bare-metal firmware on the STM32H753ZI — no HAL, no CubeMX, register-level only. DSP signal chain prototyped in Python and ported to C with CMSIS-DSP. Part of a three-board heterogeneous system: STM32H7 acquisition layer → RPi4B Linux gateway → PYNQ-Z2 FPGA accelerator.

---

## Goal

Register-level firmware engineering — no HAL, no abstraction layers. Every peripheral configured directly from RM0433. DSP prototyped in Python, ported to C with CMSIS-DSP, verified with hardware measurements on the Analog Discovery 2.

The `dsp-signal-chain/` project is the thesis practical component — FIR filter designed in Oppenheim and Proakis, implemented in fixed-point C on Cortex-M7, benchmarked against Python test vectors.

The focus is depth over breadth — understanding what the hardware is actually doing at the register level, not what a library does on top of it.

---

## Hardware

| Component | Details |
|---|---|
| MCU | STM32H753ZI — Cortex-M7 @ 480 MHz, 2MB Flash, 1MB RAM |
| Board | Nucleo-144 |
| Debugger | ST-Link V3 onboard — SWD |
| Host | Arch Linux — arm-none-eabi-gcc, OpenOCD, GDB |
| Instruments | Digilent Analog Discovery 2 |

---

## Toolchain

```
# Compiler and debug
arm-none-eabi-gcc     # compiler
arm-none-eabi-gdb     # debugger
openocd               # debug server (ST-Link SWD)
gnu make              # build system (bare-metal projects)
python + numpy/scipy  # DSP prototyping and test vector generation

# Testing and analysis
cmake                 # replaces Makefile from spi-sensor onward
unity                 # embedded C unit test framework
googletest            # C++ unit testing — dsp-signal-chain C++ layer onward
cppcheck              # static analysis (--addon=misra for dsp-signal-chain)
clang-tidy            # additional static analysis
clang-format          # code style enforcement
doxygen               # API documentation
github actions        # CI/CD — build, test, static analysis on every push
iar embedded workbench # free kickstart edition — build alongside GCC
tracealyzer           # FreeRTOS task visualization (Percepio free community edition)
```

---

## Projects

| Project | Description | Release | Status |
|---|---|---|---|
| `bare-metal-led/` | LED driver — RCC + GPIO register writes, custom startup.s + vector table | `v0.1.0` | In progress |
| `button-interrupt/` | Interrupt-driven button — EXTI, NVIC, volatile flag, AD2-measured debounce | `v0.2.0` | Planned |
| `uart-console/` | UART debug console — baud rate, printf redirect, ring buffer | `v0.3.0` | Planned |
| `signal-acquisition/` | ADC + DMA pipeline — 12-bit, circular DMA, MPU non-cacheable region, pyserial | `v0.4.0` | Planned |
| `spi-sensor/` | SPI sensor interface — CMake, Unity test, CI/CD, Doxygen | `v0.5.0` | Planned |
| `fdcan-diagnostic/` | FDCAN loopback — register-level, custom diagnostic frame format | `v0.6.0` | Planned |
| `dsp-signal-chain/` | DSP signal chain — ADC → FIR in C (CMSIS-DSP, Q1.15) → DAC, MISRA C | `v1.0.0` | Planned |
| `freertos-tasks/` | FreeRTOS — two tasks, queue, GDB plugin, stack high-water mark | `v1.1.0` | Planned |
| `freertos-dsp-power/` | FreeRTOS + DSP + power — FIR integrated, idle sleep, AD2 current, fault injection | `v1.2.0` | Planned |
| `dual-bank-bootloader/` | Dual-bank bootloader — CRC32, slot selection, rollback, UART firmware update | `v1.3.0` | Planned |

---

## How to Build

> Build instructions are documented per project inside each folder's `README.md`.
> Bare-metal projects use GNU Make. Later projects use CMake with a shared toolchain file.
> The CI badge will be added once the GitHub Actions workflow is running.

---

## How to Flash and Debug

```bash
# Terminal 1 — start OpenOCD
openocd -f interface/stlink.cfg -f target/stm32h7x.cfg

# Terminal 2 — connect GDB
arm-none-eabi-gdb build/firmware.elf
(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) continue
```

---

## Repository Structure

Each project follows the same layout:

```
project-name/
├── README.md          # what it does, how to build, how to verify
├── CHANGELOG.md       # version history per release
├── docs/
│   ├── design.md      # register decisions, architecture, key code, GDB verification
│   └── performance.md # measurements, build size, timing data
├── src/
│   └── main.c
├── startup.s          # custom startup file and vector table
├── linker.ld          # custom linker script
└── Makefile           # bare-metal projects / CMakeLists.txt later projects
```

Full repo layout:

```
embedded-dsp-firmware/
├── bare-metal-led/
├── button-interrupt/
├── uart-console/
├── signal-acquisition/
├── spi-sensor/
├── fdcan-diagnostic/
├── dsp-signal-chain/
├── freertos-tasks/
├── freertos-dsp-power/
├── dual-bank-bootloader/
├── python/                # DSP prototypes, test vectors, ADC plotter
├── cmake/                 # shared toolchain file
└── templates/             # README, design, performance, CHANGELOG templates
```

---

## Key References

- RM0433 Rev 8 — STM32H753 Reference Manual
- DS12117 Rev 10 — STM32H753xI Datasheet
- AN4841 Rev 2 — Digital Signal Processing for STM32 using CMSIS
- Cortex-M7 Technical Reference Manual r1p2
- PM0253 — STM32H7 Cortex-M7 Programming Manual

---

## Related Repositories

- [rpi4b-embedded-linux-yocto](https://github.com/eugenejakepangan/rpi4b-embedded-linux-yocto) — RPi4B + Yocto, embedded Linux gateway layer
- [pynq-z2-fpga-dsp-system](https://github.com/eugenejakepangan/pynq-z2-fpga-dsp-system) — PYNQ-Z2 FPGA, DSP acceleration and heterogeneous system capstone

---

## License

MIT — see [LICENSE](LICENSE)
