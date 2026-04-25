# [project-name]

[One or two sentences. What this firmware does. What hardware it runs on. No adjectives.]

Built and verified on STM32H753ZI Nucleo-144 (Cortex-M7, 480 MHz). No HAL. Register-level.

## Status

`v0.X.0` — [Current state in one line. What works. What's experimental.]

## Features

- [Feature 1 — concrete, not aspirational]
- [Feature 2]
- [Feature 3]

## Hardware

- **Required:** STM32H753ZI Nucleo-144
- **Debug probe:** Onboard ST-Link V3 (SWD)
- **Optional:** [e.g., Analog Discovery 2 for signal verification]

## Build

```bash
git clone https://github.com/eugenejakepangan/stm32h7-baremetal-dsp.git
cd stm32h7-baremetal-dsp/[project-dir]
make
```

**Toolchain:** `arm-none-eabi-gcc`, GNU Make, OpenOCD — Arch Linux

**Optimization:** [`-O0` for learning/debug | `-O2` for verified builds]

## Flash

```bash
openocd -f board/st_nucleo_h743zi.cfg \
        -c "program build/firmware.elf verify reset exit"
```

## Verify

[How to confirm it works. GDB commands, visual observation, instrument capture —
whatever applies to this project.]

```
(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) break main
(gdb) continue
```

## Documentation

- [`docs/design.md`](docs/design.md) — register-level decisions, architecture, trade-offs
- [`docs/performance.md`](docs/performance.md) — measurements, timing, power
- [`CHANGELOG.md`](CHANGELOG.md) — release history

## References

- [RM0433](https://www.st.com/resource/en/reference_manual/dm00314099.pdf) — STM32H753ZI reference manual
- [DS12117](https://www.st.com/resource/en/datasheet/stm32h753zi.pdf) — STM32H753ZI datasheet
- [PM0253](https://www.st.com/resource/en/programming_manual/dm00237416.pdf) — Cortex-M7 programming manual
- [Add project-specific references]

## License

MIT
