# Design

## Problem

[What this project solves in one or two sentences. Concrete. Measurable if possible.]

## Register-level implementation

### Clock configuration

| Register | Address | Write | Purpose |
|----------|---------|-------|---------|
| [e.g., RCC_AHB4ENR] | [e.g., 0x580244E0] | [`\|= (1 << N)`] | [e.g., Enable GPIOB clock] |

[Brief explanation of clock dependency — why this must come first.]

### Peripheral configuration

| Register | Address | Write | Purpose |
|----------|---------|-------|---------|
| [e.g., GPIOB_MODER] | [e.g., 0x58020400] | [bit pattern] | [purpose] |

[Additional register tables for EXTI, NVIC, DMA, ADC, USART, etc. as needed.]

### Interrupt / DMA configuration (if applicable)

[Same table format. Document EXTI, NVIC, DMA stream, DMAMUX, timer trigger
as needed. Omit this section entirely for projects without interrupts or DMA.]

## Key code

[The core code fragment — the one that demonstrates the central concept.
Keep it short (20–40 lines). Full source is in the repository.]

```c
// [annotated code excerpt]
```

## Design decisions

### [Decision title — e.g., "BSRR over ODR"]

**Choice:** [What you chose]

**Reason:** [Why — technical, concrete]

**Alternative considered:** [What you rejected and why]

[Repeat for each significant design decision. For P01–P02 this may be
one or two decisions. For P04–P06 this section grows naturally.]

## Memory layout

```
MEMORY {
    FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 2048K
    DTCM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
    SRAM   (rwx) : ORIGIN = 0x24000000, LENGTH = 512K
}
```

[Note which sections go where and why. If DMA buffers are placed in AXI SRAM,
document that here with the `__attribute__` and MPU configuration.]

## Startup and boot sequence

[Document what your startup.s does: vector table, initial SP, Reset_Handler,
.data copy, .bss zero, call to main(). Brief — the code is in the repo.]

## GDB verification

[Register inspection results that prove correct configuration.]

```
(gdb) x/xw 0x580244E0
0x580244e0: 0x00000002     // RCC_AHB4ENR: bit 1 set
```

[Include the most important register reads. Not exhaustive — just the ones
that confirm the design is correct.]

## Known limitations

- [Limitation 1 — honest about what this implementation does not handle]
- [Limitation 2]

## Failure modes (from P05 onward)

[What happens if the peripheral misbehaves, DMA overruns, ISR is delayed,
power is interrupted mid-operation? How would a production implementation
detect and handle these? Omit this section for P01–P04.]

## References

- [RM0433](https://www.st.com/resource/en/reference_manual/dm00314099.pdf) — Section X.Y (specific chapter referenced)
- [DS12117](https://www.st.com/resource/en/datasheet/stm32h753zi.pdf) — Table N (specific table referenced)
- [PM0253](https://www.st.com/resource/en/programming_manual/dm00237416.pdf)
- [Add project-specific references: AN4841, AN4838, etc.]
