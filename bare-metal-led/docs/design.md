# Design — bare-metal-led

**Project:** `bare-metal-led` · **Release:** `v0.1.0`
**Target:** STM32H753ZI (Cortex-M7), NUCLEO-H753ZI · bare-metal, no HAL, register-level only.

---

## 1. Purpose & scope

Drive the on-board user LED **LD1 (green, PB0)** by writing peripheral registers
directly — no CubeMX, no HAL, no LL. The scope is the minimum hardware path that
proves the toolchain, the custom `startup.s`, and the vector table: enable the
GPIOB clock, configure PB0 as a push-pull output, and toggle it.

---

## 2. System Dependency Map

**Full form.** Four Cortex-M7 core registers are considered and left at
reset in this project (CPACR, SCB_CCR, SCB_AIRCR, MPU) — shown as a sibling
branch rather than left undocumented, since none of the four are a default
a reader could simply assume. 

```
STM32H753ZI
│
├── Cortex-M7 core registers — considered, left at reset
│   ├── CPACR (FPU) — off
│   ├── SCB_CCR (I-cache / D-cache) — off
│   ├── SCB_AIRCR (PRIGROUP) — 0
│   └── MPU — not configured
│
└── Active dependency chain (required for this project)

    startup.s (Reset_Handler)
        │
        ▼
    Vector table (4 entries — SP, Reset, NMI, HardFault)
        │
        ▼
    RCC_AHB4ENR — GPIOBEN (bit 1)
        │   clock must be enabled — writes before this are silently discarded
        ▼
    GPIOB_MODER — PB0 = output (01)
        │   OTYPER / OSPEEDR / PUPDR left at reset — adequate for a DC LED
        ▼
    GPIOB_BSRR — set / clear PB0
        │
        ▼
    LED LD1 (PB0) — verified via GPIOB_ODR
```

---

## 3. Architecture rationale

- **No HAL / register-level.** The goal of this project is to make the silicon legible. A
  HAL call hides which register changed and why; writing the register directly
  makes every state transition observable in GDB, which is the premise of this
  repo. The cost — verbosity — is the point.
- **Custom `startup.s` + vector table.** Reset behaviour is owned, not inherited.
  The vector table at `0x08000000` supplies the initial stack pointer (`_estack`)
  and `Reset_Handler`; the handler copies `.data` (flash → DTCM), zeroes `.bss`,
  then branches to `main`. Owning this is what makes the linker-script work in
  later projects meaningful rather than magical.
- **Single port, single pin — no abstraction yet.** Abstraction is introduced
  only once there is a second instance to abstract over (`spi-sensor` onward).
  Introducing it here would be premature.

---

## 4. Register-level implementation

> **Ordering constraint:** see the System Dependency Map (§2) above —
> violating that order produces silent failures, not errors.

Register + address, bit field, value written, rationale, GDB verification.
Registers deliberately left at reset are documented as choices, not omissions.

---

### Cortex-M7 core configuration

M7 architectural registers — not STM32-specific. All are off or unconfigured
at reset. No Cortex-M7 core registers are changed in this project. Each is
documented as a deliberate non-change because the state is a dependency for
later RTOS integration. **PM0253 §4.**

#### FPU — CPACR

**Register:** `CPACR` at `0xE000ED88` — PM0253 §4.6
**Bit field:** bits [23:20] — CP10/CP11 coprocessor access
**Value written:** none — left at reset (FPU disabled, bits [23:20] = `0000`)
**Rationale:** No floating-point code in this project. FPU must be enabled in
`startup.s` before the C runtime executes when FP code is introduced (the
DSP signal-chain project). Three consequences of leaving it off: (1) any FP
instruction before enabling causes UsageFault (CFSR UFSR.NOCP), (2) compiler
using `-mfpu` without FPU on produces a fault at runtime, (3) FreeRTOS FPU
context save requires FPU to be on before the RTOS port initializes.
Recorded here so the later enable is not a surprise.
**GDB verification (confirms reset state is undisturbed):**
```
(gdb) x/xw 0xE000ED88
0xE000ED88: [FILL]    // expected: bits [23:20] = 0 (FPU off)
```

---

#### I-cache — SCB_CCR bit 17

**Register:** `SCB_CCR` at `0xE000ED14` — PM0253 §4.3.5
**Bit field:** bit 17 — IC
**Value written:** none — left at reset (I-cache off)
**Rationale:** No time-critical instruction paths in this project; timing is
N/A. I-cache will be considered deliberately when the DMA coherency question
in `signal-acquisition` forces a cache policy decision. Leaving it off here
avoids a warm-up-vs-cold measurement ambiguity in a project where timing is
N/A anyway.
**GDB verification:**
```
(gdb) x/xw 0xE000ED14
0xE000ED14: [FILL]    // expected: bit 17 = 0 (I-cache off)
```

---

#### D-cache — SCB_CCR bit 16

**Register:** `SCB_CCR` at `0xE000ED14` — PM0253 §4.3.5
**Bit field:** bit 16 — DC
**Value written:** none — left at reset (D-cache off)
**Rationale:** No DMA in this project. D-cache enabled without MPU and DMA
causes intermittent stale-data corruption (the defining bug class of the
`signal-acquisition` MPU work — AN4838, AN4839). Not enabling here is the
correct choice — there is nothing to gain and the coherency risk is not yet
a factor.
**GDB verification:** same register as I-cache — confirm bit 16 = 0.

---

**MPU:** Not configured in this project. No DMA, no stack-overflow protection
requirement. Introduced alongside D-cache + DMA in `signal-acquisition`.
Deliberate non-configuration.

---

### Clock configuration

> System clock (VOS0, PLL1, FLASH_ACR) lives in `docs/clock-tree.md`, added
> once PLL configuration is introduced — between `uart-console` and
> `signal-acquisition` — and does not yet exist for this project. For this
> project: SYSCLK = HSI 64 MHz (reset default, RCC_CFGR SWS = `0b00`). PLL
> configuration has not been done yet; all code in this project runs at
> 64 MHz.

#### GPIOB clock enable — RCC_AHB4ENR

**Register:** `RCC_AHB4ENR` at `0x580244E0` — RM0433 §8.7.43
**Bit field:** bit 1 — GPIOBEN
**Value written:** `|= (1u << 1)`
**Rationale:** GPIOB is unclocked at reset. Writes to GPIOB registers without
this bit set are accepted by the AHB4 bus but silently discarded by the
unpowered peripheral. Three consequences: (1) GPIOB_MODER write before this
is silently discarded, (2) GDB shows MODER unchanged at the reset value after
the write, (3) the LED does not respond regardless of code correctness. This
write must precede every GPIOB access.
**GDB verification:**
```
(gdb) x/xw 0x580244E0
0x580244E0: [FILL]    // expected: 0x00000002 (bit 1 set)
```

---

### NVIC priority grouping

#### Priority group — SCB_AIRCR

**Register:** `SCB_AIRCR` at `0xE000ED0C` — PM0253 §4.3.7
**Bit field:** bits [10:8] — PRIGROUP
**Value written:** none — left at reset (PRIGROUP = `000`, all 4 bits = preemption)
**Rationale:** No interrupts in this project. PRIGROUP matters once ISR
priorities are set, from `button-interrupt` onward. This is a dependency for
later RTOS integration: FreeRTOS requires `configKERNEL_INTERRUPT_PRIORITY`
to be encoded correctly relative to this grouping. The reset value of 0 is
the inherited assumption — documenting it here prevents that later phase
from guessing.
**GDB verification:**
```
(gdb) x/xw 0xE000ED0C
0xE000ED0C: [FILL]    // expected: PRIGROUP bits [10:8] = 000 (reset)
```

---

### GPIO configuration

#### PB0 output mode — GPIOB_MODER

**Register:** `GPIOB_MODER` at `0x58020400` — RM0433 §11.4.1
**Bit field:** bits [1:0] — MODER0 (PB0 mode)
**Value written:** `&= ~(0x3u << 0)` then `|= (0x1u << 0)` → `01` (general-purpose output)
**Rationale:** `GPIOB_MODER` resets to `0xFFFFFEBF` on H7 — not
`0x00000000`. PB0 bits [1:0] reset to `11` (analog). Always clear the full
2-bit field before setting — writing without clearing produces the wrong
mode if the reset field is non-zero (silent failure, no error). Mode
encoding: `00` = input, `01` = output, `10` = alternate function, `11` =
analog. `OTYPER` (push-pull), `OSPEEDR` (low speed), `PUPDR` (no pull) are
left at reset — adequate for a DC LED; recorded here as deliberate
non-changes, not omissions.
**GDB verification:**
```
(gdb) x/xw 0x58020400
0x58020400: [FILL]    // expected: 0xFFFFFEBD
                      // (reset 0xFFFFFEBF, PB0 field changed 11 → 01)
```

---

#### LED toggle — GPIOB_BSRR

**Register:** `GPIOB_BSRR` at `0x58020418` — RM0433 §11.4.7
**Bit field:** bit 0 BS0 — set PB0 high; bit 16 BR0 — reset PB0 low
**Value written:** `(1u << 0)` to set; `(1u << 16)` to clear
**Rationale:** See §5 (Design decisions) for the full BSRR vs. ODR argument.
BSRR is write-only — it cannot be read back and its written value does not
persist. Verification is therefore via `GPIOB_ODR` (offset `0x14`), which
reflects the current output latch state.
**GDB verification (via ODR, not BSRR — BSRR is write-only):**
```
(gdb) x/xw 0x58020414
0x58020414: [FILL]    // expected: 0x00000001 when PB0 high (LED on)
                      //           0x00000000 when PB0 low  (LED off)
```

---

## 5. Design decisions

### BSRR over ODR

**Claim:** `GPIOB_BSRR` for LED toggle instead of `GPIOB_ODR |=`. `ODR |=` is
a read-modify-write: read ODR, OR the bit, write back. If an interrupt writes
another PB pin between the read and the write, that change rides on the
stale read value and is silently lost. `BSRR` is atomic per bit: write `1`
to `BSx` sets pin x, `1` to `BRx` clears it, untouched bits are unaffected.
One store, no read, no race, interrupt-safe, single instruction. Three
derivations: (1) a same-port ISR corrupts `ODR |=` but not `BSRR`; (2)
`BSRR` compiles to a single `str` — visible in disassembly; (3) clearing via
`BR0` needs no separate mask. Choosing `BSRR` in this project establishes
the habit correct under concurrency before concurrency arrives in
`button-interrupt`.

**Boundary:** `BSRR` is write-only — it cannot be read for state inspection.
When the goal is reading back the current pin state (e.g., GDB debugging,
status check), ODR is the correct register. The atomicity argument does not
transfer to registers without a set/clear structure — any read-modify-write
on a normal configuration register still has the race.

**Alternative considered:** `GPIOB_ODR |= (1u << 0)`. Rejected: not
interrupt-safe; introduces a lost-update race that becomes observable in
`button-interrupt` (interrupt handling on the same port).

---

## 6. Wrong hypotheses

#### Assumed writing `GPIOB_MODER` alone would configure the output

Wrote `GPIOB_MODER` before enabling `GPIOBEN` in `RCC_AHB4ENR`. GDB showed
MODER **unchanged at its reset value (`0xFFFFFEBF`)** after the write — the
LED stayed dark. This reads as a GPIO bug; it is a **clock bug** — the write
hit an unpowered peripheral and was silently discarded (the mechanism stated
in §4). Fix: move the `RCC_AHB4ENR` write ahead of any GPIOB access.

*Note on the reset value: `GPIOB_MODER` resets to `0xFFFFFEBF`, not
`0x00000000`. If GDB shows `0x00000000` after a failed write, the peripheral
is not just underpowered — something else is wrong. The expected
"discarded write" symptom is the register holding `0xFFFFFEBF`.*

---

## 7. GDB verification summary

Every register in §4 appears here with its observed value. Tagging `v0.1.0`
is blocked until every row is filled and every checkbox is ticked. Values
below are what GDB actually showed — not the expected values restated; if
observed ≠ expected, the discrepancy belongs in this table with an
explanation.

| Register | Address | Expected | Observed (GDB) | Pass |
|---|---|---|---|---|
| `CPACR` | `0xE000ED88` | bits [23:20] = 0 (FPU off) | `[FILL]` | ☐ |
| `SCB_CCR` | `0xE000ED14` | bits 17+16 = 0 (cache off) | `[FILL]` | ☐ |
| `SCB_AIRCR` | `0xE000ED0C` | PRIGROUP bits [10:8] = 0 | `[FILL]` | ☐ |
| `RCC_AHB4ENR` | `0x580244E0` | `0x00000002` (GPIOBEN set) | `[FILL]` | ☐ |
| `GPIOB_MODER` | `0x58020400` | `0xFFFFFEBD` (PB0 = `01`) | `[FILL]` | ☐ |
| `GPIOB_ODR` | `0x58020414` | `0x00000001` when LED on | `[FILL]` | ☐ |

---

## 8. Memory layout

> Full memory region map, section placement, and linker decisions are in
> `docs/linker-script.md`, added between `signal-acquisition` and
> `spi-sensor` — it does not yet exist for this project. This project's
> `linker.ld` establishes the base layout; all later projects build on it.

This project's `linker.ld` base layout — no project-specific section overrides:

```
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 2048K
    DTCM  (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
    SRAM  (rwx) : ORIGIN = 0x24000000, LENGTH = 512K
}
```

`.text` and `.rodata` in FLASH. `.data` load in FLASH, run in DTCM (copied by
`startup.s`). `.bss` in DTCM (zeroed by `startup.s`). Stack at top of DTCM
(`_estack = ORIGIN(DTCM) + LENGTH(DTCM)`). No DMA buffers, no DTCM data
sections — those appear in `signal-acquisition` and `dsp-signal-chain`
respectively.

---

## 9. Startup and boot sequence

`startup.s` for this project is minimal — sufficient here, replaced once
`button-interrupt` writes the full vector table:

- **Vector table (4 entries):** initial SP (`_estack`), `Reset_Handler`,
  `NMI_Handler`, `HardFault_Handler`. Four entries are enough for this
  project — no peripheral interrupts. `button-interrupt` replaces this with
  the full table (16 system exceptions + external IRQs). External IRQ n
  sits at vector index 16 + n; a 4-entry table cannot reach any of them.
- **Reset_Handler:** copies `.data` from flash LMA to DTCM VMA (`_sidata` →
  `_sdata` to `_edata`), zeroes `.bss` (`_sbss` to `_ebss`), branches to `main`.
- **FPU:** not enabled in `startup.s` for this project. No FP code.
- **Caches:** not enabled. No cache management in this project.
- **VTOR:** `0x08000000` (reset default — vector table at flash origin). Not relocated.

---

## 10. Known limitations

- **4-entry vector table.** Cannot support any peripheral interrupt.
  `button-interrupt` writes the full table; alternatively, adopt the full
  table now — it is backward-compatible.
- **SYSCLK = HSI 64 MHz.** PLL configuration and `docs/clock-tree.md` are
  introduced between `uart-console` and `signal-acquisition`. All timing
  (busy-wait delay) in this project is relative to 64 MHz and will run
  **faster at 480 MHz (rev V) — 7.5×; or at 400 MHz (rev Y / SMPS supply) —
  6.25×** — once that happens. Confirm silicon revision (`docs/performance.md`
  §2, DBGMCU_IDCODE) before assuming which multiplier applies.
- **Uncalibrated busy-wait delay.** The blink period is frequency-dependent
  and unverified. Not a limitation for a proof-of-toolchain project; becomes
  one if any real timing is assumed.

---

## 11. Downstream assumptions

Decisions in this project that later projects — and eventually RTOS
integration — build on. Undocumented here becomes an undocumented
assumption downstream.

| Decision | Value / State | Later-phase implication |
|---|---|---|
| PRIGROUP | 0 (reset) | FreeRTOS `configKERNEL_INTERRUPT_PRIORITY` must be encoded under this grouping |
| FPU | disabled | RTOS port must enable FPU in startup before context-switching if FP code is added |
| D-cache | off | No cache maintenance needed until D-cache is explicitly enabled |
| MPU | not configured | RTOS MPU task-stack protection requires MPU setup before the FreeRTOS project |
| SysTick | not consumed | Free to use as RTOS tick source |
| VTOR | `0x08000000` | Flash default; must not be silently relocated without intent |

---

## References

- [RM0433](https://www.st.com/resource/en/reference_manual/dm00314099.pdf) — §8.7.43 (RCC_AHB4ENR), §11.4.1 (GPIOB_MODER), §11.4.7 (GPIOB_BSRR)
- [DS12117](https://www.st.com/resource/en/datasheet/stm32h753zi.pdf) — Table 12 (pin assignments, LD1 = PB0)
- [PM0253](https://www.st.com/resource/en/programming_manual/dm00237416.pdf) — §4.3.5 (SCB_CCR), §4.3.7 (SCB_AIRCR), §4.6 (CPACR / FPU)

---

*Fields marked `[FILL]` are empirical records, written during the session
from GDB and RM0433 output — not reconstructed afterward and not taken from
memory.*
