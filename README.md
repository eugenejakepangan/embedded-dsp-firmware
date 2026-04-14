# stm32h7-baremetal-dsp

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**About:** Bare-metal DSP signal chain on STM32H753ZI — FIR filter, ADC/DMA, CMSIS-DSP, no HAL — thesis foundation for FPGA acceleration

Bare-metal embedded systems portfolio on the STM32H753ZI — no HAL, no CubeMX, register-level only. DSP signal chain prototyped in Python and ported to C with CMSIS-DSP. Year 0–1 of a three-board heterogeneous system: STM32H7 acquisition layer → RPi4B Linux gateway → PYNQ-Z2 FPGA accelerator.

---

## Goal

Register-level firmware engineering — no HAL, no abstraction layers. Every peripheral configured directly from RM0433. DSP prototyped in Python, ported to C with CMSIS-DSP, verified with hardware measurements on the Analog Discovery 2.

P06 (DSP signal chain) is the thesis practical component — FIR filter designed in Oppenheim and Proakis, implemented in fixed-point C on Cortex-M7, benchmarked against Python test vectors.

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

### Year 0

| Project | Description | Status |
|---|---|---|
| P01 | Bare-metal LED driver — RCC + GPIO register-level | In progress |
| P02 | Interrupt-driven button — EXTI, NVIC, volatile flag, debounce | Planned |
| P03 | UART debug console — USART3, baud rate, ring buffer, printf redirect | Planned |
| P04 | ADC + DMA + Python plotter — 12-bit, pyserial, matplotlib live plot | Planned |

### Year 1

| Project | Description | Status |
|---|---|---|
| P05 | SPI sensor interface — register-level, CMake, Unity test, CI/CD | Planned |
| P06 | Complete DSP signal chain — ADC → FIR in C (CMSIS-DSP) → DAC, Python test vectors | Planned |
| P07 | FreeRTOS multi-task firmware — tasks, queues, semaphores, GDB task inspector | Planned |

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
├── P01-led/                  # Year 0
│   ├── src/
│   ├── include/
│   ├── CMakeLists.txt
│   └── README.md
├── P02-button-interrupt/     # Year 0
├── P03-uart-console/         # Year 0
├── P04-adc-dma/              # Year 0
├── P05-spi-sensor/           # Year 1
├── P06-dsp-signal-chain/     # Year 1
├── P07-freertos/             # Year 1
├── python/                   # DSP prototypes, test vectors, ADC plotter
└── cmake/                    # shared toolchain file
```

---

## Key References

- RM0433 Rev 8 — STM32H753 Reference Manual
- DS12117 Rev 10 — STM32H753xI Datasheet
- AN4841 Rev 2 — Digital Signal Processing for STM32 using CMSIS
- Cortex-M7 Technical Reference Manual r1p2
- PM0253 — STM32H7 Cortex-M7 Programming Manual

---

## Development Environment

Arch Linux — GDB + OpenOCD configured from project one.

---

## Roadmap

- **Year 0–1 (this repo)** — STM32H7 bare-metal + DSP, thesis completion.
- **Year 2** — [rpi4b-embedded-linux-yocto](https://github.com/eugenejakepangan/rpi4b-embedded-linux-yocto): RPi4B + Yocto, STM32 integration.
- **Year 3** — [pynq-z2-fpga-dsp-system](https://github.com/eugenejakepangan/pynq-z2-fpga-dsp-system): PYNQ-Z2 FPGA + full heterogeneous DSP system capstone.

---

## License

MIT — see [LICENSE](LICENSE)
