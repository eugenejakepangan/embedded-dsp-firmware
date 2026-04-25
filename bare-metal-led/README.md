# bare-metal-led

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**About:** Bare-metal DSP signal chain on STM32H753ZI — FIR filter, ADC/DMA, CMSIS-DSP, no HAL — thesis foundation for FPGA acceleration

Bare-metal firmware on the STM32H753ZI — no HAL, no CubeMX, register-level only. DSP signal chain prototyped in Python and ported to C with CMSIS-DSP. Part of a three-board heterogeneous system: STM32H7 acquisition layer → RPi4B Linux gateway → PYNQ-Z2 FPGA accelerator.

---

## Goal

Register-level firmware engineering — no HAL, no abstraction layers. Every peripheral configured directly from RM0433. DSP prototyped in Python, ported to C with CMSIS-DSP, verified with hardware measurements on the Analog Discovery 2.

`dsp-signal-chain/` is the thesis practical component — FIR filter designed in Oppenheim and Proakis, implemented in fixed-point C on Cortex-M7, benchmarked against Python test vectors.

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

| Project | Description | Tags | Status |
|---|---|---|---|
| `bare-metal-led/` | LED driver — RCC + GPIO register-level, own startup.s + vector table | `v0.1-stlink-verify` → `v0.1-gdb-verify` → `v0.1` → `v0.1-gdb-integration` | In progress |
| `button-interrupt/` | Interrupt-driven button — EXTI, NVIC, volatile flag, debounce (AD2 measured) | `v0.2.1-poll` → `v0.2.2-exti` → `v0.2.3-debounce` | Planned |
| `uart-console/` | UART debug console — USART3, baud rate, printf redirect, ring buffer | `v1.0.1-tx-blocking` → `v1.0.2-printf` → `v1.0.3-rx-ringbuffer` | Planned |
| `signal-acquisition/` | ADC + DMA + Python plotter — 12-bit, circular DMA, MPU non-cacheable region, pyserial | `v1.0-adc-poll` → `v1.1-adc-dma` → `v1.2-adc-uart` → `v1.3-adc-plot` → `v1.4-mpu-dma` | Planned |
| `linker-script/` | Linker script session — DMA buffer relocation, `.dma_buffer` section, objdump verification | — | Planned |

| Project | Description | Tags | Status |
|---|---|---|---|
| `spi-sensor/` | SPI sensor interface — register-level, CMake, Unity test, CI/CD Stage 1, Doxygen | `feature/spi-sensor` | Planned |
| `fdcan-diagnostic/` | FDCAN loopback — register-level FDCAN, TX/RX verify, custom diagnostic frame format (ID/length/payload/checksum) | — | Planned |
| `dsp-signal-chain/` | Complete DSP signal chain — ADC → FIR in C (CMSIS-DSP, Q1.15) → DAC, Python test vectors, MISRA C (cppcheck) | `v2.0-dsp-chain` | Planned |
| `freertos-tasks/` | FreeRTOS basic — two tasks, queue, GDB FreeRTOS plugin, stack high-water mark, CI green | `v2.1-freertos-basic` | Planned |
| `freertos-dsp-power/` | FreeRTOS + DSP + power — `dsp-signal-chain/` FIR integrated, idle sleep (WFI), AD2 current measurement, fault injection | `v2.2-freertos-dsp` | Planned |

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

```
embedded-dsp-firmware/
├── bare-metal-led/
│   ├── src/
│   ├── include/
│   ├── CMakeLists.txt
│   └── README.md
├── button-interrupt/
├── uart-console/
├── signal-acquisition/
├── spi-sensor/
├── fdcan-diagnostic/
├── dsp-signal-chain/
├── freertos-tasks/
├── freertos-dsp-power/
├── python/                       # DSP prototypes, test vectors, ADC plotter
└── cmake/                        # shared toolchain file
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
