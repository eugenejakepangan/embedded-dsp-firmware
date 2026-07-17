## Objective

<!-- What does this project or change accomplish?
     One or two sentences. Match the projects table description in README.md. -->

**Assumes (upstream)**

<!-- What earlier module/PR's claim does this depend on still being true?
     e.g. "Assumes RCC PLL1 config from PR #3 — SYSCLK = 480 MHz."
     Write "None — no upstream dependency" if standalone. -->

## Why this design

<!-- Tradeoff rationale — why this approach over the alternatives?
     Examples:
     - Why DMA circular buffer over interrupt-per-sample for ADC?
     - Why Q1.15 over Q1.31 for the FIR coefficients?
     - Why polling over EXTI for this button debounce?
     - Why this clock frequency / prescaler combination?
     This is the field that separates a design decision from an implementation detail. -->

**Where does this analogy / pattern not hold?**

<!-- REQUIRED — one sentence stating the boundary condition where the
     design choice or the analogy behind it breaks down. For a Type III
     cross-domain isomorphism, this is the boundary where the isomorphism
     fails. State it as Claim / Boundary:
       Claim:    BSRR atomicity removes the ISR/main race on this pin.
       Boundary: does not hold for multi-bit fields — those still need a
                 read-modify-write guard.
     If you cannot name the boundary, the rationale is not yet a compression. -->

## Registers touched

<!-- List every peripheral register written or read, with the bit fields
     and values. If no new registers were touched, write "None — no new peripherals". -->

| Register | Address | Bits | Value | Purpose |
|---|---|---|---|---|
| | | | | |

## Test performed

<!-- State the prediction BEFORE running the test, then record what
     actually happened. A passing test with no prior prediction is
     confirmation, not falsification. -->

**Expected vs Observed**

| Expected | Observed |
|---|---|
| | |

**Verified on:** [ ] Hardware &nbsp;&nbsp; [ ] Host simulation / unit test &nbsp;&nbsp; [ ] Renode / QEMU

<!-- For register-layer modules (clock, cache, DMA, power), only "Hardware"
     counts as ground truth — a simulator validates against ST's documented
     behavior, not silicon. For pure logic modules, host sim is sufficient. -->

- [ ] GDB register inspection — values match expected (document in docs/design.md)
- [ ] Visual / functional verification — describe what was observed
- [ ] Build size recorded in docs/performance.md
- [ ] Timing measured (if applicable) — method and result in docs/performance.md
- [ ] docs/design.md updated — includes tradeoff rationale from "Why this design"
- [ ] Wrong hypotheses recorded in docs/design.md — not erased, not smoothed over
- [ ] Boundary condition stated — where the design choice / analogy does not hold
- [ ] Checked errata sheet (ES0392 or applicable) for this peripheral, if applicable
- [ ] Upstream assumption re-verified, if "Assumes" above is non-empty
- [ ] docs/performance.md updated
- [ ] CHANGELOG.md updated with release entry
- [ ] Commit messages describe register-level decisions, not just file changes
- [ ] Semver tag created with annotated message

## Notes

<!-- Anything unexpected encountered. Wrong hypotheses, GDB revelations,
     errata hit. This feeds the capture file (~/writings/capture/<project>.md) — copy anything worth keeping. -->
