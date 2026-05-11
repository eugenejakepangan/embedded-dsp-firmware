# Performance — bare-metal-led

This file is intentionally sparse for bare-metal-led. The only measurable
quantity at this stage is build size. The DWT methodology is documented
now so it is established before it is needed.

---

## Methodology

### Cycle counting

Cortex-M7 DWT cycle counter. Enable once at startup:

```c
CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
DWT->CTRL       |= DWT_CTRL_CYCCNTENA_Msk;
DWT->CYCCNT      = 0;
```

Measure a block:

```c
uint32_t start  = DWT->CYCCNT;
/* ... code under measurement ... */
uint32_t cycles = DWT->CYCCNT - start;
/* time_us = cycles / (SystemCoreClock / 1000000) */
```

Convert cycles to time using `SystemCoreClock`. At 480 MHz:
1 cycle = 2.08 ns. 1 µs = 480 cycles.

### Current measurement

Analog Discovery 2 across a 1 Ω shunt resistor on the 3.3 V rail.
Voltage across shunt = current in mA. Not applicable for bare-metal-led.

### Build configuration

- Optimization: `-O0` (debug build — no optimization, full symbol information)
- Compiler: `arm-none-eabi-gcc` [fill in version: `arm-none-eabi-gcc --version`]
- Silicon revision: check via `DBGMCU->IDCODE` at `0xE00E1000`

---

## Build size

Measured via `arm-none-eabi-size build/firmware.elf`.

| Section | Size | Notes |
|---|---|---|
| `.text` | [X bytes] | Code + constants in flash |
| `.data` | [X bytes] | Initialized variables (flash → DTCM at boot) |
| `.bss` | [X bytes] | Zero-initialized variables in DTCM |
| **Total flash** | [X bytes] | `.text` + `.data` |
| **Total RAM** | [X bytes] | `.data` + `.bss` |

---

## Timing

Not applicable for bare-metal-led. First timing measurements appear in
`button-interrupt` (ISR execution time via DWT) and `uart-console`
(UART throughput via AD2).

---

## Power

Not applicable for bare-metal-led. First power measurements appear in
`freertos-dsp-power` (active vs WFI idle current via AD2).

---

## Reproducing

```bash
cd bare-metal-led
make
arm-none-eabi-size build/firmware.elf
```
