# Fedora 44 alongside Windows on an ASUS TUF Dash F15 (RTX 3060)

A verified plan for running Fedora Workstation 44 next to Windows on an ASUS TUF Dash F15, tuned so the RTX 3060 Laptop GPU stays asleep while you write code and wakes up only when you ask for it.

Target hardware: ASUS TUF Dash F15, Intel Core i5 (11300H in the 2021 FX516 series, or 12450H in the 2022 FX517 series), NVIDIA GeForce RTX 3060 Laptop GPU, NVMe SSD, 144Hz or 165Hz panel.

> This document supersedes an earlier draft that was written by a small model. That draft contained four errors that would have cost you a working install: it told you to disable Secure Boot, it told you to install and enable `supergfxd` (deprecated), it pointed at a dead COPR repository, and it gave a Claude Code install URL that does not exist. Every command below was checked against upstream docs in August 2026. Where something is model year dependent, the check command is given instead of a guess.

---

## What you get at the end

* A boot menu with Windows and Fedora, both booting with Secure Boot still enabled.
* Fedora on the iGPU for daily work, with the RTX 3060 in D3cold at roughly zero watts.
* NVIDIA drivers installed, signed, and available the moment you want CUDA, an external display, or a game.
* Fan and power control through the same firmware knobs Armoury Crate uses on Windows.
* A working dev environment including Claude Code.

Total time: about 90 minutes, most of it waiting on downloads.

---

## Read this before you touch anything

**Back up first.** Partition work is the one step in this document that can lose data. Copy anything irreplaceable off the machine.

**Write down your BitLocker recovery key.** Go to https://aka.ms/myrecoverykey while signed into the Microsoft account tied to this laptop and save the key somewhere off the machine. You need this even if everything goes right, because firmware changes can trigger a BitLocker recovery prompt on the next Windows boot.

**Two things that are easy to confuse.** Windows "Fast Startup" is a hibernation trick that leaves the NTFS filesystem in a dirty state. ASUS BIOS "Fast Boot" is a POST shortcut that makes it hard to catch the boot menu. You will turn off both, for different reasons.

**Do not disable Secure Boot.** The older advice to disable it is obsolete and actively harmful here: Windows 11 wants Secure Boot for its own health checks, and Fedora ships a Microsoft signed shim so it boots fine with Secure Boot on. The only thing Secure Boot blocks is the unsigned NVIDIA kernel module, and Phase 5 solves that properly by enrolling your own key. Leave Secure Boot enabled the whole way through.

---

## Phase 1: prepare Windows (15 minutes)

### 1.1 Suspend or turn off BitLocker

Modern ASUS laptops ship with BitLocker or Windows device encryption on. An encrypted partition cannot be safely resized from Linux, and the Fedora installer will refuse to reclaim space from it.

Open Settings, search for "Device encryption" or "BitLocker", and either turn it off (decryption takes a while on a full drive, but leaves you in the simplest state) or suspend protection before the partition work. Turning it off is the recommended path for a machine you plan to keep in this configuration.

Verify from an Administrator PowerShell:

```powershell
manage-bde -status
```

Every volume should read `Protection Off` and `Fully Decrypted` before you continue.

### 1.2 Turn off Fast Startup

Control Panel, Power Options, "Choose what the power buttons do", "Change settings that are currently unavailable", then untick "Turn on fast startup". Apply.

Leave hibernation off entirely if you want to be thorough:

```powershell
powercfg /h off
```

That also frees a chunk of disk equal to a good fraction of your RAM, and it removes the failure mode where Windows holds the filesystem hostage between boots.

### 1.3 Leave the GPU MUX in hybrid mode

The 2022 FX517 models have a hardware MUX switch. Open Armoury Crate, find the GPU mode setting, and set it to **Optimus** or **Eco / Standard**, not **Ultimate / dGPU only**. If the MUX is latched to the dGPU, Linux sees the panel wired to the NVIDIA card and the whole "keep the 3060 asleep" plan cannot work until you flip it back. The 2021 FX516 models have no MUX and need nothing here.

### 1.4 Shrink the Windows partition

Right click Start, Disk Management. Right click the `C:` volume, Shrink Volume, and free **100 GB minimum**. 150 GB to 200 GB is a much better number if you have the room: containers, model weights, and toolchains eat space quickly, and shrinking again later is far more annoying than doing it once now.

Leave the freed space as **Unallocated**. Do not create or format a partition in it. Fedora needs to see raw free space.

If Windows refuses to shrink as far as you want, the blockers are almost always immovable files. Turn off System Protection for `C:`, disable the pagefile temporarily, run `Disk Cleanup`, reboot, then try again. `defrag C: /X` (free space consolidation) sometimes helps too.

---

## Phase 2: write the installer (10 minutes)

You need a USB stick of 8 GB or larger. Its contents will be destroyed.

1. Download **Fedora Media Writer** from https://fedoraproject.org/workstation/download
2. Run it, choose **Fedora Workstation 44**, select your USB stick, and let it download and write. It verifies the checksum for you, which is why this is better than a manual `dd` or a random third party writer.

Fedora 44 ships GNOME 50 on a recent kernel. Kernel 6.19 or newer is what you want on this laptop: it carries the `asus-armoury` platform driver that exposes the firmware knobs (fan curves, panel overdrive, dGPU disable) that older kernels lacked. Fedora 44 satisfies that out of the box, which is why this guide pins to 44 rather than an older release.

---

## Phase 3: ASUS firmware settings (5 minutes)

Shut down fully (not a restart, and not with Fast Startup enabled). Power on and tap **F2** repeatedly to enter the UEFI utility. Press **F7** for Advanced Mode.

Set the following:

* **Boot, Fast Boot: Disabled.** This is the firmware POST shortcut, not the Windows setting. Turning it off gives you a reliable window to press ESC or F2.
* **Security, Secure Boot: leave Enabled.** Confirm it says Enabled and move on.
* **Boot, CSM / Legacy support: Disabled.** UEFI only. Windows is already installed in UEFI mode and Fedora will match it.
* Confirm the SATA/NVMe mode is **AHCI** and not Intel RST / Optane. If your machine is in RST mode, the Fedora installer will not see the SSD at all. Switching a live Windows install out of RST mode requires putting Windows into safe mode first, so check this now rather than discovering it mid install.

Press **F10** to save and exit.

---

## Phase 4: install Fedora (25 minutes)

1. With the USB inserted, power on and tap **ESC** to reach the ASUS boot menu. Pick the UEFI entry for your USB stick.
2. Choose **Start Fedora Workstation Live**. When the desktop loads, connect to WiFi, then launch **Install Fedora**.
3. On the installation method screen, pick the option to install alongside your existing system / use free space. The Fedora 44 web installer will target the unallocated space you created in Phase 1 and leave the Windows partitions untouched.
4. **Do not reformat the EFI System Partition.** The installer reuses the existing ESP and drops Fedora's bootloader beside Windows'. If any screen offers to reformat or erase the ESP, back out. That partition holds the Windows boot files.
5. On the review screen, read the summary line by line before confirming. It should show new partitions being created in free space, and it should not show the Windows partition being deleted or reformatted. This is the point of no return.
6. Let it install, then reboot and remove the USB.

You should land on GRUB with entries for Fedora and for Windows Boot Manager.

### If Windows is missing from the boot menu

Two fixes, in order of preference.

The firmware boot menu always works: tap ESC at power on and pick Windows Boot Manager directly. Nothing is broken, GRUB just did not enumerate it.

To add it to GRUB, install `os-prober` and let it scan:

```bash
sudo dnf install os-prober
echo 'GRUB_DISABLE_OS_PROBER=false' | sudo tee -a /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

Fedora ships with `GRUB_DISABLE_OS_PROBER=true` on purpose, so this is expected behaviour rather than a bug.

### Fix the clock disagreement now

Linux keeps the hardware clock in UTC, Windows assumes local time, so each one leaves the other several hours wrong. Fix it on the Windows side once, from an Administrator PowerShell:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```

Now both systems agree and neither needs to be corrected on every switch.

---

## Phase 5: NVIDIA driver with Secure Boot intact (20 minutes)

Order matters here. Enroll your signing key **before** the driver builds, so the very first module akmods compiles is already signed and trusted.

### 5.1 Update, then enable RPM Fusion

```bash
sudo dnf upgrade --refresh
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Reboot if the upgrade pulled a new kernel, so you build against the kernel you will actually run.

### 5.2 Create and enroll your Machine Owner Key

```bash
sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

`mokutil` asks for a one time password. Pick something you can retype at a firmware prompt in a minute, and remember it.

Reboot. The blue **MOK Manager** screen appears before the boot menu. Choose **Enroll MOK**, then **Continue**, then **Yes**, then enter that password. It reboots itself.

This screen appears exactly once. If you skip it or let it time out, the enrollment is discarded and you have to run the `mokutil --import` step again.

Confirm afterwards:

```bash
mokutil --list-enrolled | grep -i akmods
mokutil --sb-state
```

You want your akmods certificate listed and `SecureBoot enabled`.

### 5.3 Install the driver

```bash
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda
```

**Wait five minutes before rebooting.** The kernel module is compiled in the background after the package installs, and rebooting mid build leaves you with no working graphics stack on the next boot. Watch for it to finish:

```bash
modinfo -F version nvidia
```

Once that prints a version number instead of an error, reboot.

### 5.4 Verify

```bash
nvidia-smi
modinfo nvidia | grep -i sig
```

`nvidia-smi` should list the RTX 3060 Laptop GPU. `modinfo` should show a signature and signer. If `nvidia-smi` fails while Secure Boot is on, the module was built before the key was enrolled: run `sudo akmods --force --rebuild` and reboot.

---

## Phase 6: silence the dGPU (10 minutes)

This is the phase the original draft got most wrong, so here is what actually happens on this hardware in 2026.

`supergfxd` and `supergfxctl` are deprecated upstream and should not be installed. GPU control moved into `asusctl` for the hardware MUX, and into the new `cardwire` daemon for runtime switching without a reboot. `cardwire` is still labelled experimental, so the plan below leans on the mechanism that is both reliable and already built into the driver: NVIDIA runtime power management.

### 6.1 Install ASUS platform control from Terra

The `lukenukem/asus-linux` COPR is unmaintained. Terra is where these packages live now.

```bash
sudo dnf install --nogpgcheck \
  --repofrompath 'terra,https://repos.fyralabs.com/terra$releasever' terra-release
sudo dnf install asusctl asusctl-rog-gui
sudo systemctl enable --now asusd.service
```

Fedora defaults to `tuned-ppd`, which fights with what `asusctl` and GNOME expect. Swap it:

```bash
sudo dnf swap tuned-ppd power-profiles-daemon --allowerasing
sudo systemctl enable --now power-profiles-daemon.service
```

Do **not** also install TLP. TLP and `power-profiles-daemon` both want exclusive control of the same tunables, and running them together produces power behaviour that is worse than either alone and impossible to reason about.

Quick check:

```bash
asusctl profile -p          # current platform profile
asusctl profile -P Quiet    # Quiet, Balanced, or Performance
asusctl -s                  # supported features on this model
```

`asusctl-rog-gui` (ROG Control Center) gives you the same thing plus fan curve editing in a window.

### 6.2 Confirm the GPU is already asleep

Fedora enables NVIDIA dynamic power management by default on Turing and newer, and Ampere mobile parts drop to D3cold when idle. Check it rather than assuming:

```bash
GPU=$(lspci -Dn | awk '/10de:/{print $1; exit}')
cat /sys/bus/pci/devices/$GPU/power/runtime_status
cat /sys/bus/pci/devices/$GPU/power/control
```

`suspended` plus `auto` means the RTX 3060 is powered down right now. That is the quiet, cool, long battery state, and nothing further is required. `nvidia-smi` will wake the card just by running, so read the sysfs files instead when you want the truth.

If it reports `active` while idle, force the policy on explicitly:

```bash
sudo tee /etc/modprobe.d/nvidia-power.conf <<'EOF'
options nvidia NVreg_DynamicPowerManagement=0x02
EOF
sudo dracut --force
```

Reboot and recheck. `0x02` is the most aggressive setting: it lets the driver cut power to the card entirely after idle, not merely clock it down.

Also make sure the ASUS side is not pinning the card on:

```bash
cat /sys/devices/platform/asus-nb-wmi/gpu_mux_mode 2>/dev/null   # 0 = hybrid/Optimus, 1 = dGPU only
```

`0`, or the file not existing at all on the 2021 model, is what you want.

### 6.3 Optional: hard disable the dGPU

Runtime suspend gets you to roughly zero draw, so this is a preference rather than a fix. If you want the card gone from the PCI bus entirely on a given boot:

```bash
cat /sys/devices/platform/asus-nb-wmi/dgpu_disable
echo 1 | sudo tee /sys/devices/platform/asus-nb-wmi/dgpu_disable
```

Trade off: nothing CUDA, no external display over the dGPU outputs, and no games until you set it back to `0` and reboot. If you go this route, prefer driving it through `asusctl` / ROG Control Center so the setting survives cleanly rather than poking sysfs by hand.

Rebooting into Windows always restores full RTX 3060 behaviour regardless of what you set in Fedora. These knobs do not persist across the boot menu.

### 6.4 When you do want the GPU

Per application, on demand:

```bash
nvidia-offload() { __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia "$@"; }
nvidia-offload glxinfo | grep -i renderer
```

The card wakes for that process and goes back to sleep when it exits. Steam has the same thing built in as "Run with NVIDIA GPU" in a game's launch options.

---

## Phase 7: thermals, battery, and the rest of the hardware (10 minutes)

```bash
sudo dnf install thermald powertop
sudo systemctl enable --now thermald
```

`thermald` matters on both CPUs here. The 11300H is a 4 core Tiger Lake part that leans hard on turbo, and the 12450H is a hybrid Alder Lake part with performance and efficiency cores; in both cases you want the Intel thermal daemon managing the limits rather than the firmware alone.

Useful checks:

```bash
sudo powertop --auto-tune          # one shot, run once on battery
asusctl fan-curve -m Quiet -e true # enable the Quiet fan curve
sensors                            # lm_sensors, install if missing
upower -i /org/freedesktop/UPower/devices/battery_BAT0 | grep -E 'energy-rate|percentage'
```

`energy-rate` on an idle desktop with the dGPU suspended is the single number that tells you whether this whole configuration worked. Low single digit watts is the goal.

Battery longevity, since this laptop supports a charge limit:

```bash
asusctl -c 80    # stop charging at 80 percent
```

Worth setting if the machine mostly lives on a desk.

Keyboard backlight:

```bash
asusctl -k med   # off, low, med, high
```

---

## Phase 8: development environment (15 minutes)

```bash
sudo dnf install @development-tools git gh ripgrep fd-find fzf jq tmux neovim zsh
sudo dnf install podman podman-compose distrobox
```

Node via a version manager rather than the system package, so projects can disagree about versions:

```bash
curl -fsSL https://fnm.vercel.app/install | bash
```

Claude Code, with the correct URL:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

That installs a self contained binary to `~/.local/bin/claude` with data in `~/.local/share/claude`. No Node required, no root required, and it updates itself in the background. Confirm with `claude --version`, then run `claude` once to authenticate. Claude Code needs a Pro, Max, Team, or Enterprise plan, or a Console account with API billing.

Flatpak for desktop apps, which is how you get browsers and editors without waiting on Fedora packaging:

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

---

## Phase 9 (optional): make the boot menu look good

The default GRUB menu is plain text. It is fully themeable, and you can reduce it to exactly two entries, Fedora and Windows 11, with a 1080p background and real icons. See [BOOT_UI.md](BOOT_UI.md) for both routes: theming GRUB (recommended, leaves the signed boot chain alone) and replacing the picker with rEFInd (better looking, costs you a second MOK enrollment and manual updates forever).

---

## Final verification checklist

Run these once and you know the whole stack landed:

```bash
mokutil --sb-state                                   # SecureBoot enabled
nvidia-smi --query-gpu=name,driver_version --format=csv
cat /sys/bus/pci/devices/$(lspci -Dn | awk '/10de:/{print $1; exit}')/power/runtime_status
asusctl -s
systemctl is-active asusd power-profiles-daemon thermald
efibootmgr | grep -iE 'fedora|windows'
claude --version
```

Expected: Secure Boot enabled, the driver present, the GPU `suspended`, all three services `active`, both boot entries listed, and Claude Code on PATH.

---

## Troubleshooting

**Black screen after installing the NVIDIA driver.** You rebooted before the module finished compiling. At GRUB, press `e` on the Fedora entry, append `nomodeset` to the `linux` line, boot, then run `sudo akmods --force --rebuild`, wait for `modinfo -F version nvidia` to answer, and reboot normally.

**BitLocker recovery prompt on the next Windows boot.** Expected after firmware changes if you only suspended rather than disabled protection. Enter the recovery key you saved in the preflight step. This is why that step is first.

**Fedora installer does not see the SSD.** Almost always Intel RST / Optane mode in the firmware instead of AHCI. See Phase 3.

**Fans loud on idle in Fedora.** `asusd` is not running, or a Performance profile is latched. Check `systemctl status asusd` and `asusctl profile -p`, then set `asusctl profile -P Quiet` and enable the Quiet fan curve.

**No boot menu at all, straight into Windows.** ASUS firmware reordered itself, which it does after some Windows updates. Fix the order in the UEFI Boot tab or with `sudo efibootmgr -o <fedora_entry>,<windows_entry>`.

**`nvidia-smi` fails but `lspci` shows the card.** Signature problem. `modinfo nvidia | grep -i sig` will be empty. Rebuild after confirming the key is enrolled.

**GNOME session will not start after a driver update.** Fedora 44 GNOME 50 is Wayland only, there is no X.org session to fall back to. Boot the previous kernel from GRUB's advanced entries, which still has a working module, and rebuild from there.

---

## Sources

* [Fedora Workstation setup guide, asus-linux.org](https://asus-linux.org/guides/fedora-guide/) — Terra repository, `asusctl`, the `tuned-ppd` swap, and the note that `supergfxd` is superseded by `cardwire`
* [supergfxctl manual, asus-linux.org](https://asus-linux.org/manual/supergfxctl-manual/) — mode semantics and the deprecation status
* [supergfxctl repository](https://gitlab.com/asus-linux/supergfxctl/-/blob/main/README.md) — supported modes and MUX detection
* [asusctl on the Arch Wiki](https://wiki.archlinux.org/title/Asusctl) — `armoury` subcommand, profiles, `gpu_mux_mode`
* [ASUS Linux on the Arch Wiki](https://wiki.archlinux.org/title/ASUS_Linux) — `dgpu_disable`, `asus-nb-wmi` sysfs paths
* [Fedora 44 release announcement](https://fedoramagazine.org/announcing-fedora-linux-44/) — GNOME 50, Wayland only
* [Anaconda Web UI partitioning change proposal](https://fedoraproject.org/wiki/Changes/Anaconda_Web_UI_Partitioning) — free space and reclaim behaviour, the BitLocker constraint
* [Fedora 42 MOK enrollment guide](https://github.com/drgreenthumb93/Fedora42_MOK_enrollment) — `kmodgenca` and `mokutil` flow
* [RPM Fusion NVIDIA howto](https://rpmfusion.org/Howto/NVIDIA) — `akmod-nvidia`, build wait, CUDA package
* [Fedora quick docs: creating a live USB](https://docs.fedoraproject.org/en-US/quick-docs/creating-and-using-a-live-installation-image/) — Fedora Media Writer
* [ASUS TUF Dash F15 2022 tech specs](https://www.asus.com/us/laptops/for-gaming/tuf-gaming/asus-tuf-dash-f15-2022/techspec/) — MUX switch presence on FX517
* [LaptopMedia FX516 specs](https://laptopmedia.com/laptop-specs/asus-tuf-dash-f15-852/) and [FX517 specs](https://laptopmedia.com/ca/laptop-specs/asus-tuf-dash-f15-fx517-36/) — model year differences
* [Notebookcheck FX517ZM review](https://www.notebookcheck.net/Asus-TUF-Dash-15-FX517ZM-i5-12450H.693860.0.html) — RTX 3060 power limits
* [os-prober and GRUB on Fedora](https://itsfoss.com/grub-os-prober/) — restoring the Windows entry
