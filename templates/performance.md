# Performance

[This file is intentionally sparse for early projects. P01–P02 may have
only a build size table and a note. The file grows naturally from P03
onward as measurable quantities appear. Having the file present from the
start establishes the habit.]

## Methodology

**Cycle counting:** Cortex-M7 DWT cycle counter (`DWT->CYCCNT`), enabled via
`CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk; DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk`.
Counts converted to time via system clock frequency.

**Current measurement:** [e.g., Analog Discovery 2 across 1 Ω shunt on 3.3 V
rail. Omit if not measured for this project.]

**Build configuration:**
- Optimization: [`-O0` | `-O2` — state which and why]
- Compiler: `arm-none-eabi-gcc` [version]
- Silicon revision: [check via `DBGMCU_IDCODE`]

## Build size

| Section | Size | Notes |
|---------|------|-------|
| .text   | [X bytes] | Code + constants in flash |
| .data   | [X bytes] | Initialized variables (flash → DTCM at boot) |
| .bss    | [X bytes] | Zero-initialized variables in DTCM |
| **Total flash** | [X bytes] | |

Measured via `arm-none-eabi-size build/firmware.elf`.

## Timing (P02 onward)

| Parameter | Expected | Measured | Method |
|-----------|----------|----------|--------|
| [e.g., ISR execution time] | [calculated] | [measured] | DWT cycle count |
| [e.g., UART throughput] | [calculated from baud] | [measured] | Logic analyzer / AD2 |
| [e.g., ADC sample period] | [timer config] | [measured] | DWT / AD2 |
| [e.g., DMA transfer latency] | [calculated] | [measured] | DWT |
| [e.g., FIR cycles/sample] | [estimated] | [measured] | DWT |

[For P06 DSP chain: document filter execution time and confirm it meets
the real-time processing budget at your sample rate.]

## Power (P07 onward)

| Mode | Current | Notes |
|------|---------|-------|
| Run, active processing | [mA] | |
| Run, `__WFI()` in idle hook | [mA] | |
| Sleep | [µA] | |

Wake latency from `__WFI()` to first useful instruction: [cycles / ns].

## Reproducing

[How to reproduce these measurements. Script paths, UART baud rate for
benchmark output, instrument setup notes.]
