# Performance — bare-metal-led

**Project:** `bare-metal-led` · **Release:** `v0.1.0`
**Target:** STM32H753ZI (Cortex-M7)

> Methodology is written first and is complete. Numbers are `[FILL]`: they
> come from the toolchain (`size`) and the instrument (AD2) on the actual
> build. A calculated value placed in a Measured column is fabrication.

---

## 1. Methodology (written before the first measurement)

**Build size.** `arm-none-eabi-size build/bare-metal-led.elf` for the
`.text` / `.data` / `.bss` totals; `arm-none-eabi-size -A` for per-section
breakdown. Recorded at the tag, and before/after for any change expected to
move size.

**Stack high-water mark.** Paint the full stack region with `0xDEADBEEF` at
the start of `main()`, before any function calls. Run the worst-case path.
Inspect in GDB: `x/Nxw <stack_base>`, where N is derived from the explicit
reserved stack size defined in `linker.ld` (`N = stack_size_bytes / 4`) — not
from a subtraction between unrelated symbols such as `_estack − _sdata`
(`_sdata` is the start of initialized RAM data, not the bottom of the stack,
so that subtraction does not yield the stack size). Find the lowest address
where `0xDEADBEEF` is no longer present — that is the high-water mark. If
none remain, the stack overflowed.

**Timing.** DWT CYCCNT. Enable sequence — `TRCENA` must precede `CYCCNTENA`
(writing `CYCCNTENA` first produces a counter that does not increment on H7,
no error), and `CYCCNT` is reset to a known zero before the counter is
enabled:

```c
CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;  // trace enable first
DWT->CYCCNT       = 0;                            // reset before enable
DWT->CTRL        |= DWT_CTRL_CYCCNTENA_Msk;       // then counter enable
```

Enable before the measured block, not inside it. Time conversion:
`t_ns = (cycles × 1e9) / F_SYSCLK`.

```c
/* No-HAL project: SystemCoreClock is a CMSIS symbol maintained by
   SystemInit/SystemCoreClockUpdate — neither exists here. Use a constant.
   Dividing by an unmaintained SystemCoreClock either fails to link or
   silently uses the reset default (64 MHz), making every conversion wrong. */
#define F_SYSCLK 64000000u    /* HSI default for this project; update once
                                  PLL configuration is introduced (between
                                  uart-console and signal-acquisition) to the
                                  SYSCLK actually verified for your silicon
                                  revision — 480000000u on rev V, 400000000u
                                  on rev Y or an SMPS-supply board (see §4
                                  and docs/clock-tree.md) */
```

*For this project, see §5 — timing is N/A.*

**Power.** Analog Discovery 2 across the board's current path. AD2 resolves
run-mode current (mA range). Sleep-mode is below AD2 resolution and is not
present in this project (no `__WFI`); the methodology note is carried
forward for the first project that sleeps.

---

## 2. Build configuration

| Field | Value |
|---|---|
| Optimization | `[FILL — e.g., -O0]` |
| Compiler | `arm-none-eabi-gcc [FILL — --version output]` |
| Silicon revision | STM32H753ZI rev `[FILL — from DBGMCU_IDCODE at 0x5C001000]` |
| Build command | `[FILL — make / cmake invocation]` |
| I-cache | off (not enabled in this project) |
| D-cache | off (not enabled in this project) |
| FPU | disabled (not enabled in this project) |

---

## 3. Build size

Measured via `arm-none-eabi-size -A build/bare-metal-led.elf`.
Baseline for every later project.

| Section | Bytes | Notes |
|---|---|---|
| `.text` | `[FILL]` | Code + vector table + startup.s |
| `.data` | `[FILL]` | Initialized variables (flash → DTCM copy) |
| `.bss` | `[FILL]` | Zero-initialized variables in DTCM |
| **Total flash** | `[FILL]` | `.text` + `.data` load region |
| Stack (configured) | `[FILL]` | Explicit reserved size from `linker.ld` (e.g. `_Min_Stack_Size`), not a subtraction between unrelated symbols |
| Stack (high-water) | `[FILL]` | Sentinel method — see §1 |

---

## 4. Clock verification

`docs/clock-tree.md` is added once PLL configuration is introduced —
between `uart-console` and `signal-acquisition` — and does not yet exist for
this project. Current clock state is the reset default.

| Parameter | Expected | Observed (GDB) | Verified |
|---|---|---|---|
| SYSCLK source | HSI (SWS = `0b00`) | `(gdb) x/xw 0x58024410` — bits [5:3] | ☐ |
| PLL1 | not running (PLL1RDY = 0) | `(gdb) x/xw 0x58024400` — bit 25 | ☐ |

`F_SYSCLK` for this project = `64000000u`. All timing (busy-wait delay loop)
is relative to 64 MHz. Once PLL configuration is introduced, update to the
SYSCLK verified for your silicon revision (§2, DBGMCU_IDCODE):
`480000000u` and a **7.5× faster** blink rate on rev V (480 MHz), or
`400000000u` and a **6.25× faster** blink rate on rev Y or an SMPS-supply
board (400 MHz) — without a compensating loop adjustment. Confirm which
revision applies before assuming either multiplier.

---

## 5. Timing

**N/A — no time-critical code.**
The only time-dependent construct is the blink delay — an uncalibrated
busy-wait, not a deadline. Marking N/A rather than blank records that this
is a judgement, not an oversight. The DWT methodology in §1 is in place so
the first timed project (`signal-acquisition` onward) measures on day one.

---

## 6. Power

| Condition | Current | Method |
|---|---|---|
| Run, LED on | `[FILL] mA` | AD2 run-mode, 1 Ω shunt |
| Run, LED off | `[FILL] mA` | AD2 run-mode, 1 Ω shunt |

No sleep mode in this project; sleep-current methodology deferred to the
first `__WFI` project (the FreeRTOS project).

---

## 7. Regressions

None at `v0.1.0` — this is the baseline. Subsequent size or power changes
are appended here, noted not hidden.

| Date | Change | Metric | Before | After | Notes |
|---|---|---|---|---|---|
| — | baseline | `.text` | — | `[FILL]` | `v0.1.0` |

---

## 8. Reproducing

- **Build:** `[FILL — exact make command from repo root]`
- **Flash:** `openocd -f board/[FILL — verify this filename against your installed OpenOCD scripts before use, e.g. \`ls /usr/share/openocd/scripts/board/ | grep h7\`; do not assume st_nucleo_h743zi.cfg is present or correct for your version] -c "program build/bare-metal-led.elf verify reset exit"`
- **GDB attach:** `arm-none-eabi-gdb build/bare-metal-led.elf`, `target extended-remote :3333`, `monitor reset halt`
- **Build size:** `arm-none-eabi-size -A build/bare-metal-led.elf`
- **Stack high-water mark:** paint sentinel before any function call, run main loop for 10 s, inspect with `x/Nxw <stack_base>` where N = configured stack size in bytes / 4
- **Power:** AD2 W2 current measurement across 1 Ω shunt on 3.3 V rail; average over 5 s
