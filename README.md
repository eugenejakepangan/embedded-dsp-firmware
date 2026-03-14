# stm32h7-baremetal-dsp

Bare-metal embedded projects on the STM32H753ZI — no HAL, no CubeMX, no abstraction layers.
Every peripheral is configured by writing directly to hardware registers using RM0433.
DSP work is prototyped in Python (numpy/scipy) and ported to C with CMSIS-DSP.

This repository is Year 0–1 of a structured embedded systems roadmap targeting
professional embedded engineering in Sweden.

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

```bash
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/arm-none-eabi.cmake
make
```

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
└── cmake/            # toolchain file
```

---

## Key References

- RM0433 Rev 8 — STM32H753 Reference Manual
- DS12110 Rev 11 — STM32H753ZI Datasheet
- AN4841 Rev 2 — Digital Signal Processing for STM32 using CMSIS
- Cortex-M7 Technical Reference Manual r1p2
- PM0253 — STM32H7 Cortex-M7 Programming Manual

---

## License

MIT — see [LICENSE](LICENSE)
