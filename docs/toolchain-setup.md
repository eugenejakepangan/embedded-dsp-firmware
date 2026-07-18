# Toolchain Setup

One-time setup for Arch Linux.

## Packages installed

```bash
yay -S arm-none-eabi-gcc openocd arm-none-eabi-binutils arm-none-eabi-gdb \
       arm-none-eabi-newlib stlink digilent.adept.runtime
```

**Digilent Waveforms** does not install cleanly through a plain `yay -S
digilent.waveforms` — that first attempt fails at the download step because
the `.deb` requires a Digilent account login `yay` can't automate.

**If stuck in `yay`'s cache after that failed attempt** — recovery path:

1. `yay -S digilent.waveforms` fails at the download step. Sign in at the
   Digilent site and download `digilent.waveforms_3.25.1_amd64.deb` manually.
2. Place the file in `~/.cache/yay/digilent.waveforms/`.
3. Edit the `PKGBUILD` with a text editor:
   - Remove `aarch64` from the `arch=('x86_64')` array.
   - Comment out or remove the `source_aarch64=(...)` and
     `sha256sums_aarch64=(...)` lines.
   - Leave the passing `sha256sums_x86_64` hash as-is.
4. Run `makepkg -si`. From this cache location, this only produces
   `digilent.waveforms-3.25.1-1.src.tar.gz` — a **source** package, not a
   binary one.
   > **Do not try `sudo pacman -U digilent.waveforms-3.25.1-1.src.tar.gz` —
   > it will not work.** `pacman -U` only installs binary packages
   > (`.pkg.tar.zst` / `.pkg.tar.xz`). A `.src.tar.gz` is the source package
   > `makepkg` produces for distribution, not something `pacman` can install
   > directly — it isn't a recognized installable format at all.
5. Extract it into a fresh build directory instead:
   ```bash
   mkdir -p ~/build/digilent.waveforms
   tar xfz digilent.waveforms-3.25.1-1.src.tar.gz -C ~/build/
   cd ~/build/digilent.waveforms/
   ```
6. Copy the downloaded `.deb` into that fresh directory, then run
   `makepkg -si` again from here. This second pass builds and installs the
   real binary package — Waveforms installs successfully at this point.

**Cleaner alternative, avoiding the two-pass rebuild entirely:** skip `yay`
and clone the AUR package directly, then do the same manual download and
edit — this starts from a clean directory instead of `yay`'s partially-failed
cache, and only needs one `makepkg -si` pass:

```bash
git clone https://aur.archlinux.org/digilent.waveforms.git ~/.cache/yay/digilent.waveforms
cd ~/.cache/yay/digilent.waveforms
```

Then the same steps 2–3 above (download the `.deb` into this directory, edit
the `PKGBUILD`), then:

```bash
makepkg -si
```

This produces `digilent.waveforms-3.25.1-1-x86_64.pkg.tar.zst` — a real
binary package — and installs it directly in one pass.

**Waveforms installs but won't launch — shared library fix.** The error:

```
waveforms: error while loading shared libraries: libdmgr.so.2: cannot open shared object file: No such file or directory
```

Fixed with two changes — registering the Digilent runtime path is the
complete fix on its own; the symlink is a narrower one-file patch that's
redundant once the path is registered, kept here because both were applied:

```bash
# Register the Digilent runtime path so the dynamic linker can find it
echo '/usr/lib/digilent/adept' | sudo tee /etc/ld.so.conf.d/digilent-adept.conf
sudo ldconfig

# Narrower one-file symlink — not needed once the path above is registered,
# but applied at the same time
sudo ln -s /usr/lib/digilent/adept/libdmgr.so.2 /usr/lib/libdmgr.so.2
```

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
- **Digilent Waveforms — first attempt via plain `yay -S` failed**, because
  `yay` can't automate the Digilent account login the `.deb` download
  requires. Building from `yay`'s cache after that (manual download +
  `PKGBUILD` edit) produces a `.src.tar.gz` source package on the first
  `makepkg -si` pass — **`pacman -U` cannot install this format**; it
  extracts into a fresh directory and needs a second `makepkg -si` there to
  actually build and install. Cloning the AUR package fresh
  (`git clone https://aur.archlinux.org/digilent.waveforms.git`) instead of
  starting from `yay`'s cache avoids the two-pass rebuild — same manual
  download and edit, one `makepkg -si` pass.
- **WaveForms installed but failed to launch** —
  `libdmgr.so.2: cannot open shared object file`. Fixed by registering
  `/usr/lib/digilent/adept` in `ld.so.conf.d` and running `ldconfig`.
