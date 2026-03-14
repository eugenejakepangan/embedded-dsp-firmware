# stm32h7-baremetal-dsp

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**About me:** Computer engineering student in the Philippines finishing a bare-metal DSP thesis
on STM32H7, building a 3-year STM32 → RPi4B → PYNQ-Z2 portfolio targeting remote embedded
engineering roles.

Bare-metal embedded projects on the STM32H753ZI — no HAL, no CubeMX, no abstraction layers.
Every peripheral is configured by writing directly to hardware registers using RM0433.
DSP work is prototyped in Python (numpy/scipy) and ported to C with CMSIS-DSP.

---

## Goal

This repository is Year 0–1 of a structured embedded systems roadmap:
finishing my STM32H7 bare-metal + DSP thesis, then using this work
to apply for remote embedded roles.

The focus is depth over breadth — understanding what the hardware is actually doing
at the register level, not what a library does on top of it.

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
arm-none-eabi-gcc     # compiler
arm-none-eabi-gdb     # debugger
openocd               # debug server (ST-Link SWD)
cmake                 # build system
python + numpy/scipy  # DSP prototyping and test vector generation
```

---

## Projects

| Project | Description | Status |
|---|---|---|
| P01 | Bare-metal LED driver — RCC + GPIO register-level | In progress |
| P02 | Interrupt-driven button — EXTI, NVIC | Planned |
| P03 | UART debug console — USART3, polling | Planned |
| P04 | ADC + DMA + Python plotter — 12-bit, pyserial pipeline | Planned |
| P05 | Timers + PWM — TIM register-level | Planned |
| P06 | FIR filter in C — CMSIS-DSP arm_fir_q15(), Q1.15 fixed-point | Planned |

---

## How to Build

> Build system will be added during P01. Each project will have its own
> CMakeLists.txt. A shared toolchain file will live at `cmake/arm-none-eabi.cmake`.
> The CI badge will be added once the GitHub Actions workflow is running.

---

## How to Flash and Debug

```bash
# Terminal 1 — start OpenOCD
openocd -f interface/stlink.cfg -f target/stm32h7x.cfg

# Terminal 2 — connect GDB
arm-none-eabi-gdb build/project.elf
(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) continue
```

---

## Repository Structure

```
stm32h7-baremetal-dsp/
├── P01-led/
│   ├── src/
│   ├── include/
│   ├── CMakeLists.txt
│   └── README.md
├── P02-button-interrupt/
├── P03-uart-console/
├── P04-adc-dma/
├── python/           # DSP prototypes, test vectors, ADC plotter
└── cmake/            # shared toolchain file
```

---

## Key References

- RM0433 Rev 8 — STM32H753 Reference Manual
- DS12110 Rev 11 — STM32H753ZI Datasheet
- AN4841 Rev 2 — Digital Signal Processing for STM32 using CMSIS
- Cortex-M7 Technical Reference Manual r1p2
- PM0253 — STM32H7 Cortex-M7 Programming Manual

---

## Roadmap

- **Year 0–1 (this repo)** — STM32H7 bare-metal + DSP, thesis completion.
- **Year 2** — [rpi4b-embedded-linux-yocto](https://github.com/eugenejakepangan/rpi4b-embedded-linux-yocto): RPi4B + Yocto, STM32 integration.
- **Year 3** — [pynq-z2-fpga-dsp-system](https://github.com/eugenejakepangan/pynq-z2-fpga-dsp-system): PYNQ-Z2 FPGA + full heterogeneous DSP system capstone.

---

## License

MIT — see [LICENSE](LICENSE)
