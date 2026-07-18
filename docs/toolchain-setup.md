# Toolchain Setup

One-time setup for Arch Linux.

## Packages installed

```bash
yay -S arm-none-eabi-gcc openocd arm-none-eabi-binutils arm-none-eabi-gdb \
       arm-none-eabi-newlib stlink digilent.adept.runtime
```

**Digilent Waveforms** does not install cleanly through `yay` — it needs manual
intervention:

1. `yay -S digilent.waveforms` fails at the download step. Sign in at the
   Digilent site and download `digilent.waveforms_3.25.1_amd64.deb` manually.
2. Place the file in `~/.cache/yay/digilent.waveforms/`.
3. Edit the `PKGBUILD` with a text editor:
   - Remove `aarch64` from the `arch=('x86_64')` array.
   - Comment out or remove the `source_aarch64=(...)` and
     `sha256sums_aarch64=(...)` lines.
   - Leave the passing `sha256sums_x86_64` hash as-is.
4. Run `makepkg -si`. This produces `digilent.waveforms-3.25.1-1.src.tar.gz`.
5. Extract and inspect:
   ```bash
   mkdir -p ~/build/digilent.waveforms
   tar xfz digilent.waveforms-3.25.1-1.src.tar.gz -C ~/build/
   cd ~/build/digilent.waveforms/
   ```
6. Copy the downloaded `.deb` into that directory, then run `makepkg -si`
   again. Enter the password when prompted — Waveforms installs successfully
   on this second pass.

## Versions confirmed

```bash
$ arm-none-eabi-gcc --version
arm-none-eabi-gcc (Arch Repository) 16.1.0

$ openocd --version
Open On-Chip Debugger 0.12.0-01004-g9ea7f3d64-dirty (2025-11-12-08:18)

$ arm-none-eabi-gdb --version
GNU gdb (GDB) 17.2
```

## ST-Link / OpenOCD connectivity check

Board plugged into the Arch host via USB:

```bash
$ lsusb | grep -i stm
Bus 005 Device 006: ID 0483:374e STMicroelectronics STLINK-V3
```

Product ID `374E` — confirms **ST-Link V3** on this board (OpenOCD's own
detection below confirms this a second way).

**udev rules.** Check whether the rule files already exist:

```bash
ls /etc/udev/rules.d/ | egrep -i "openocd|stlink"
```

If nothing is listed, copy them from where the `stlink` package installs them:

```bash
sudo cp /usr/lib/udev/rules.d/60-openocd.rules /etc/udev/rules.d/
sudo cp /usr/lib/udev/rules.d/49-stlinkv3.rules /etc/udev/rules.d/
```

If no OpenOCD rule is available at all, create one manually:

```bash
sudo vim /etc/udev/rules.d/99-openocd.rules
```

```
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374E", MODE="0666", GROUP="plugdev"
```

Then reload:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

**Confirm OpenOCD config files are present:**

```bash
ls /usr/share/openocd/scripts/interface/ | grep stlink
ls /usr/share/openocd/scripts/target/ | grep stm32h7
```

**Group membership** — needed for serial port access later, once UART work
begins:

```bash
groups $USER
sudo usermod -aG uucp $USER
sudo usermod -aG dialout $USER
```

**First successful connection.** Terminal 1:

```bash
$ openocd -f board/st_nucleo_h743zi.cfg
Open On-Chip Debugger 0.12.0-01004-g9ea7f3d64-dirty (2025-11-12-08:18)
Licensed under GNU GPL v2

Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : clock speed 1800 kHz
Info : STLINK V3J17M10 (API v3) VID:PID 0483:374E
Info : Target voltage: 3.284881
Info : [stm32h7x.cpu0] Cortex-M7 r1p1 processor detected
Info : [stm32h7x.cpu0] target has 8 breakpoints, 4 watchpoints
Info : starting gdb server for stm32h7x.cpu0 on 3333
Info : Listening on port 3333 for gdb connections
[stm32h7x.cpu0] halted due to breakpoint, current mode: Thread
xPSR: 0x61000000 pc: 0x080003d8 msp: 0x2001fff8
```

`board/st_nucleo_h743zi.cfg` is confirmed correct — this is a working
connection, not just a file-existence check.

> **Two different "revision" numbers appear here — don't conflate them.**
> `Cortex-M7 r1p1` is the **ARM core** revision (the processor IP block itself).
> The **STM32 silicon revision** (rev V vs rev Y — the one that gates whether
> 480 MHz is reachable) is a separate axis, read from `DBGMCU_IDCODE` at
> `0x5C001000` — not from this OpenOCD banner. Confirm silicon revision
> separately; see `docs/performance.md` §2.

The 8 hardware breakpoints and 4 watchpoints reported here are physical
registers in the Cortex-M7 debug unit, not a software limit.

Terminal 2:

```bash
$ arm-none-eabi-gdb firmware.elf
(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) break main
(gdb) continue
```

## GPG signing setup

```bash
# Generate the key (RSA 4096, no expiry — see public_repository.md's
# GPG hardening note for the revocation certificate this requires)
gpg --full-generate-key

# Find the key ID
gpg --list-secret-keys --keyid-format LONG

# Export public key (for GitHub)
gpg --armor --export your_email@email.com > public-key.asc

# Export secret key (offline backup — never in the repo, never on this machine alone)
gpg --export-secret-keys --armor YOUR_KEY_ID > private-key.asc

# Configure Git
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# Verify signing works
git commit --allow-empty -m "test: verify GPG signing on laptop"
git log --show-signature -1
```

> **Revocation certificate.** Generated and confirmed stored on external HDD,
> not committed to the repo. Secret-key backup stored alongside it. GitHub's
> GPG key matches the local key used throughout — no key change, no action
> needed on GitHub.

## Gotchas hit during setup

- **OpenOCD udev rules weren't where expected.** The rule files are not in
  `/etc/udev/rules.d/` by default on this system — they ship with the
  `stlink` package under `/usr/lib/udev/rules.d/` and have to be copied over
  manually.
- **Digilent Waveforms can't be installed automatically via `yay`.** It
  requires manually downloading the `.deb` from the Digilent site and editing
  the AUR package's `PKGBUILD` to drop the `aarch64` architecture entries
  before `makepkg -si` succeeds.
