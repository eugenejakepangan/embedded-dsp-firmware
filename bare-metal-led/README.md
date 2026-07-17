# bare-metal-led

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../LICENSE)

**About:** LED driver on STM32H753ZI — RCC + GPIO register writes, no HAL. First project in the bare-metal DSP firmware portfolio.

Bare-metal firmware on the STM32H753ZI — no HAL, no CubeMX, register-level only. Every peripheral configured directly from RM0433. This project establishes the compile → flash → debug workflow before any DSP work begins.

> **Status: scaffolding.** Source files (`src/main.c`, `startup.s`, `linker.ld`, `Makefile`) are not yet written. `docs/design.md` and `docs/performance.md` are complete and ready to be filled with observed values as bring-up proceeds. Sections below describe the target build/flash/verify flow — not yet-confirmed results.

---

## Goal

Bring up the STM32H753ZI in pure bare-metal: prove GPIO control via direct register writes, establish the OpenOCD + GDB debug workflow, and verify every configuration step through GDB register inspection.

This project has no HAL dependency — no CubeMX-generated code, no abstraction layer between the code and the registers. CMSIS-Core and CMSIS-Device headers are used for register definitions (`RCC->`, `GPIOB->`, `DWT->`) — the same typed-register-access layer `docs/performance.md` uses via `CoreDebug->DEMCR` and `DWT->CTRL`. CMSIS is the register-naming layer; HAL is the abstraction on top of it that this project avoids.

---

## Hardware

| Component | Details |
|---|---|
| MCU | STM32H753ZI — Cortex-M7 @ 64 MHz (HSI default, no PLL) |
| Board | Nucleo-144 |
| LED | LD1 green — PB0 |
| Debugger | ST-Link onboard — SWD *(confirm V2 vs V3 against the physical board silkscreen or enumerated USB device string — not yet verified)* |
| Host | Arch Linux — arm-none-eabi-gcc, OpenOCD, GDB |

---

## How to Build

```bash
cd bare-metal-led
make
```

Target output: `build/firmware.elf` and `build/firmware.bin`. *(Not yet buildable — `Makefile` and `src/` do not exist yet; see status note above.)*

> Requires `arm-none-eabi-gcc` and `arm-none-eabi-binutils` installed.
> See `docs/toolchain-setup.md` at repo root for one-time setup.

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

LED LD1 (green, PB0) is expected to toggle at approximately 1 Hz once flashed — the blink rate is an uncalibrated busy-wait, not a measured deadline (see `docs/performance.md` §4 for the multiplier once PLL configuration is introduced).

---

## Verification

Registers to confirm once GDB bring-up is complete — full verification table and observed values live in `docs/design.md` §6:

```
(gdb) x/xw 0x580244E0
0x580244E0: [FILL]   // expected: 0x00000002 — RCC_AHB4ENR bit 1 (GPIOBEN) set

(gdb) x/xw 0x58020400
0x58020400: [FILL]   // expected: 0xFFFFFEBD — GPIOB_MODER PB0 field changed 11 → 01 (output)

(gdb) x/xw 0x58020414
0x58020414: [FILL]   // expected: 0x00000001 — GPIOB_ODR PB0 high (LED on)
```

Full register-level decisions and GDB verification in `docs/design.md`.

---

## Project Structure

Target structure once bring-up is complete:

```
bare-metal-led/
├── README.md
├── docs/
│   ├── design.md      # register decisions, BSRR vs ODR rationale, GDB verification
│   └── performance.md # build size, DWT methodology
├── src/
│   └── main.c
├── startup.s          # custom vector table and Reset_Handler
├── linker.ld          # FLASH + DTCM memory regions
└── Makefile
```

`docs/` is the only piece of this tree that currently exists.

---

## Key References

- RM0433 Rev 8 §11 — GPIO registers
- RM0433 Rev 8 §8.7.43 — RCC_AHB4ENR
- DS12117 Rev 10 — Table 12 (PB0 pin assignment)
- PM0253 — Cortex-M7 boot sequence

---

## Release

`v0.1.0` — see repo-level [CHANGELOG.md](../CHANGELOG.md)
