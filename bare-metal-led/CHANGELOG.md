# Changelog

All notable changes to this project are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Interrupt-driven button with EXTI and software debounce

## [0.1.0] — YYYY-MM-DD

Initial release.

### Added
- Bare-metal LED blink on PB0 (LD1 green) via RCC + GPIO register writes
- Custom startup.s with vector table and Reset_Handler
- Custom linker script (flash + DTCM memory regions)
- OpenOCD + GDB debug workflow
- Makefile build system with arm-none-eabi-gcc
