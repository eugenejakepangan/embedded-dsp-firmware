# Design

## Problem

Bring up the STM32H753ZI in pure bare-metal: prove GPIO control via direct register writes, establish the compile → flash → debug workflow, and verify every configuration step through GDB register inspection.

## Register-level implementation

### Clock configuration

| Register | Address | Write | Purpose |
|----------|---------|-------|---------|
| RCC_AHB4ENR | 0x580244E0 | `\|= (1 << 1)` | Enable GPIOB clock (GPIOBEN) |

This must be the first peripheral operation. GPIO registers do not respond to writes while their clock is gated — the write is silently ignored. No fault, no error. This is the most common bare-metal mistake on STM32.

### GPIO configuration

| Register | Address | Write | Purpose |
|----------|---------|-------|---------|
| GPIOB_MODER | 0x58020400 | `&= ~(1 << 1)`, then `\|= (1 << 0)` | PB0 mode = 01 (general purpose output) |
| GPIOB_BSRR | 0x58020418 | `= (1 << 0)` | Set PB0 high (LED on) |
| GPIOB_BSRR | 0x58020418 | `= (1 << 16)` | Reset PB0 low (LED off) |

OTYPER (push-pull) and OSPEEDR (low speed) are left at reset defaults. Correct and sufficient for LED toggling.

MODER uses 2 bits per pin: `00` = input (reset default), `01` = output, `10` = alternate function, `11` = analog. The clear-then-set discipline on MODER is intentional — even when reset value has the bit already clear, explicit clearing prevents bugs on ports where reset values differ.

## Key code

```c
#include <stdint.h>

#define RCC_BASE    0x58024400UL
#define GPIOB_BASE  0x58020400UL

#define RCC_AHB4ENR  (*(volatile uint32_t *)(RCC_BASE + 0xE0))
#define GPIOB_MODER  (*(volatile uint32_t *)(GPIOB_BASE + 0x00))
#define GPIOB_BSRR   (*(volatile uint32_t *)(GPIOB_BASE + 0x18))

int main(void)
{
    RCC_AHB4ENR |= (1 << 1);

    GPIOB_MODER &= ~(1 << 1);
    GPIOB_MODER |=  (1 << 0);

    while (1) {
        GPIOB_BSRR = (1 << 0);
        for (volatile int i = 0; i < 1000000; i++);
        GPIOB_BSRR = (1 << 16);
        for (volatile int i = 0; i < 1000000; i++);
    }
}
```

The `volatile` on the delay counter prevents the compiler from optimizing the loop away. Without it, the compiler removes the loop (no observable side effects) and the LED appears stuck.

## Design decisions

### BSRR over ODR for pin toggle

**Choice:** Use `GPIOB_BSRR` for all pin state changes.

**Reason:** BSRR is a single write — atomic, no read-modify-write. ODR requires read-modify-write (`ODR |= ...`), which is three CPU operations. If an ISR fires between the read and write and modifies a different pin on the same port, the main loop overwrites the ISR's change.

**In P01 this is academic** — no interrupts, one pin. In P02 (EXTI interrupt on PC13), it becomes a real race condition. Starting with BSRR from P01 means the correct habit is established before it matters.

### Busy-wait delay over hardware timer

**Choice:** `for (volatile int i ...)` loop for delay.

**Reason:** P01 is about GPIO register control, not timing accuracy. The delay is crude — not cycle-accurate, varies with optimization level, wastes CPU. Hardware timer replaces this in P04.

### No HAL, no CMSIS struct headers

**Choice:** Raw `#define` addresses with `volatile uint32_t *` casts.

**Reason:** Forces understanding of every address and every bit. CMSIS device headers (`GPIOB->BSRR` syntax) are a valid alternative — they map the same addresses via structs. Either approach is bare-metal. [Note: decide CMSIS headers vs raw defines before P02 and stay consistent.]

## Memory layout

```
MEMORY {
    FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 2048K
    DTCM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
    SRAM   (rwx) : ORIGIN = 0x24000000, LENGTH = 512K
}

SECTIONS {
    .text : { *(.isr_vector) *(.text*) *(.rodata*) } > FLASH
    .data : { *(.data*) } > DTCM AT> FLASH
    .bss  : { *(.bss*) *(COMMON) } > DTCM
}
```

Stack pointer initialized to top of DTCM (`0x20020000`). All data fits in DTCM for P01. AXI SRAM usage begins in P04 when DMA buffers require bus-accessible memory (DMA cannot access DTCM on STM32H7).

## Startup and boot sequence

Custom `startup.s` provides:

1. Vector table at `.isr_vector` — placed at flash base (`0x08000000`)
2. First vector table entry: initial stack pointer (`0x20020000`)
3. Second entry: `Reset_Handler` address
4. `Reset_Handler`: copies `.data` from flash (LMA) to DTCM (VMA), zeroes `.bss`, calls `main()`
5. Default handlers for all exceptions — infinite loops until implemented in later projects

The `.data` section has two addresses: it runs from DTCM at runtime but is stored in flash after `.text`. The startup code bridges this using linker-exported symbols (`_sidata`, `_sdata`, `_edata`, `_sbss`, `_ebss`).

## GDB verification

```
(gdb) x/xw 0x580244E0
0x580244e0: 0x00000002     // RCC_AHB4ENR: bit 1 set — GPIOB clock enabled

(gdb) p/x GPIOB->MODER
$1 = 0xXXXXXXX1            // Bits [1:0] = 01 — PB0 output

(gdb) p/x GPIOB->ODR
$2 = 0x00000001             // PB0 high — LED on

(gdb) p/x GPIOB->ODR
$3 = 0x00000000             // PB0 low — LED off
```

## Known limitations

- Delay timing is approximate and varies with clock frequency and optimization level
- No error handling — if clock enable fails (hardware fault), firmware hangs in the blink loop with no indication
- Single LED, single GPIO port — no multi-port considerations yet
- Default HSI clock (64 MHz) — no PLL configuration

## References

- [RM0433 Rev 8](https://www.st.com/resource/en/reference_manual/dm00314099.pdf) — Chapter 11 (GPIO), Section 8.7.40 (RCC_AHB4ENR)
- [DS12117 Rev 10](https://www.st.com/resource/en/datasheet/stm32h753zi.pdf) — Table 12 (pin assignments)
- [PM0253 Rev 5](https://www.st.com/resource/en/programming_manual/dm00237416.pdf) — Cortex-M7 boot sequence
