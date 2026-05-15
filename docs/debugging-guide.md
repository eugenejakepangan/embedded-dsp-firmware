# Debugging Guide

OpenOCD + GDB workflow for STM32H753ZI on Nucleo-144.
Expand each section as new techniques are used across projects.

---

## Starting the debug session

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

## Hard fault diagnosis — Cortex-M7

Hard faults are the most disorienting failure mode in bare-metal work
without a HAL. They produce no error message — the CPU halts and you
must read the fault status registers to understand what happened.

**Common causes on Cortex-M7 without HAL:**
- Stack overflow from undersized stack in linker script
- Unaligned memory access (Cortex-M7 strict alignment enabled by default)
- Bad function pointer in vector table (wrong address or missing handler)
- Write to read-only flash memory
- Executing from invalid memory address

**The fault register chain — read in this order:**

```
(gdb) monitor reset halt
(gdb) break HardFault_Handler
(gdb) continue
```

When the hard fault fires, inspect:

```
(gdb) x/x 0xE000ED28   # CFSR — Configurable Fault Status Register
(gdb) x/x 0xE000ED2C   # HFSR — Hard Fault Status Register
(gdb) x/x 0xE000ED34   # MMFAR — MemManage Fault Address Register
(gdb) x/x 0xE000ED38   # BFAR — BusFault Address Register
```

**CFSR breakdown (RM0433 §11.5):**

| Bits | Field | Meaning |
|---|---|---|
| [7:0] | MMFSR | MemManage fault status |
| [15:8] | BFSR | BusFault status |
| [31:16] | UFSR | UsageFault status |

Key UFSR bits:
- Bit 16 (UNDEFINSTR) — undefined instruction executed
- Bit 17 (INVSTATE) — invalid EPSR state — bad function pointer (Thumb bit not set)
- Bit 24 (UNALIGNED) — unaligned memory access

Key BFSR bits:
- Bit 8 (IBUSERR) — instruction fetch fault
- Bit 9 (PRECISERR) — precise data bus error — BFAR holds the address
- Bit 15 (BFARVALID) — BFAR holds a valid fault address

**Reading the stacked PC — where the fault occurred:**

```
(gdb) info registers
```

When a hard fault fires, the CPU stacks the pre-fault register state.
The stacked PC is the address of the instruction that caused the fault.

```
(gdb) x/8x $sp    # examine stack — stacked r0, r1, r2, r3, r12, lr, pc, xpsr
```

The stacked PC is at offset +24 bytes from the stack pointer at fault entry.

**Disassemble around the fault address:**

```
(gdb) disassemble 0xYOUR_PC_VALUE
```

Add this handler to catch and inspect faults during development:

```c
void HardFault_Handler(void)
{
    /* Set a breakpoint here in GDB: break HardFault_Handler */
    __asm volatile("bkpt #0");
    while (1);
}
```

---

## Inspecting peripheral registers

```
(gdb) x/x 0x58024400   # RCC base (check clock enables)
(gdb) x/x 0x58020000   # GPIOB base
(gdb) x/x 0xE0001000   # DWT base (cycle counter)
```

---

## DWT cycle counter — timing measurement

```c
/* Enable DWT cycle counter — do this once at startup */
CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
DWT->CTRL       |= DWT_CTRL_CYCCNTENA_Msk;
DWT->CYCCNT      = 0;

/* Measure a block */
uint32_t start = DWT->CYCCNT;
/* ... code to measure ... */
uint32_t cycles = DWT->CYCCNT - start;
/* time_us = cycles / (SystemCoreClock / 1000000) */
```

Read from GDB without modifying source:

```
(gdb) x/x 0xE0001004   # DWT->CYCCNT
```

---

## .gdbinit — useful startup commands

Place at repo root (already excluded by `.gitignore` if named `gdb.log`):

```
target extended-remote :3333
monitor reset halt
load
```

Invoke with:

```bash
arm-none-eabi-gdb -x .gdbinit build/firmware.elf
```

---

*Expand this file as new debugging techniques are used across projects.*
*Hard fault section should be read before v0.2.0 (button-interrupt) — EXTI*
*misconfiguration can produce faults that are non-obvious without this guide.*
