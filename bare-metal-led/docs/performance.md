# Performance

P01 is a GPIO bring-up project. No timing-critical behavior to measure. This file establishes the measurement habit; substantive data begins in P03 (UART throughput) and P04 (ADC sample rate).

## Build configuration

- Optimization: `-O0` — no optimization. GDB single-stepping is predictable.
- Compiler: `arm-none-eabi-gcc` [version — fill from `arm-none-eabi-gcc -v`]
- Clock: HSI 64 MHz (default, no PLL)

## Build size

| Section | Size | Notes |
|---------|------|-------|
| .text   | [fill] | Code + vector table + constants |
| .data   | [fill] | Initialized globals (copied from flash to DTCM at boot) |
| .bss    | [fill] | Zero-initialized globals |
| **Total flash** | [fill] | |

Measured via `arm-none-eabi-size build/firmware.elf`.
