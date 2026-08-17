# A boot menu that does not look like 1998

Short answer: yes. The boot picker is fully themeable, and you can get it down to exactly two beautiful entries, **Fedora** and **Windows 11**, with a 1080p background, real icons, and your own fonts.

Two routes. Pick one.

* **Route A, theme GRUB.** Keeps the bootloader Fedora installed and supports. Nothing about your Secure Boot or NVIDIA setup changes. About 20 minutes. This is the recommended route and it looks very good.
* **Route B, replace the picker with rEFInd.** An icon based graphical picker with mouse support that auto detects every OS on the machine. It looks better than any GRUB theme, but it is an unsigned third party EFI binary, so with Secure Boot on you must enroll a second Machine Owner Key, and `dnf` will never update it for you. Choose this only if you want the look badly enough to own the maintenance.

Route C, `systemd-boot`, is worth naming only to rule it out: it is cleaner internally but has no theming engine at all. It would make the menu plainer, not prettier.

One thing that is true of both routes: the menu is drawn by firmware graphics on the iGPU, before any kernel or driver loads. Your RTX 3060 and NVIDIA configuration are irrelevant at this stage, so nothing here can break Phase 5 or Phase 6 of the main plan.

---

## Route A: theme GRUB

### A.1 Pick and install a theme

Good current options:

* [Elegant GRUB2 themes](https://github.com/vinceliuice/Elegant-grub2-themes) by vinceliuice. Photographic backgrounds, four layout styles, proper 1080p assets. The most polished set.
* [Catppuccin for GRUB](https://github.com/catppuccin/grub). Flat pastel, matches the palette if you already use Catppuccin elsewhere.
* [distro grub themes](https://github.com/AdisonCavani/distro-grub-themes). One theme per distro logo, clean and simple.

Elegant, installed for your 1080p panel:

```bash
git clone https://github.com/vinceliuice/Elegant-grub2-themes.git
cd Elegant-grub2-themes
sudo ./install.sh -t mountain -p float -c dark -s 1080p -b
```

The flags: `-t` picks the background (forest, mojave, mountain, wave), `-p` the panel style (window, float, sharp, blur), `-c` dark or light, `-s` the resolution, and `-b` installs into `/boot/grub2/themes` rather than `/boot/grub`. Fedora uses `grub2` paths, so `-b` matters here.

The installer writes the theme, points `GRUB_THEME` at it, and regenerates the config. Verify it landed:

```bash
ls /boot/grub2/themes/
grep THEME /etc/default/grub
```

If you would rather install a theme by hand, it is just a directory containing `theme.txt`:

```bash
sudo cp -r <theme-dir> /boot/grub2/themes/
sudo sed -i 's|^#\?GRUB_THEME=.*|GRUB_THEME="/boot/grub2/themes/<theme-dir>/theme.txt"|' /etc/default/grub
```

### A.2 Configure GRUB for graphics

Edit `/etc/default/grub`. The settings that matter:

```bash
GRUB_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu
GRUB_TERMINAL_OUTPUT=gfxterm
GRUB_GFXMODE=1920x1080x32
GRUB_GFXPAYLOAD_LINUX=keep
GRUB_THEME="/boot/grub2/themes/Elegant-mountain-float-dark/theme.txt"
GRUB_DISABLE_SUBMENU=true
```

The single most common reason a theme silently does not appear: `GRUB_TERMINAL_OUTPUT="console"` is still set. Console output forces text mode and the theme is ignored entirely. It must be `gfxterm`.

`GRUB_GFXPAYLOAD_LINUX=keep` holds the framebuffer resolution through kernel handoff, which removes the ugly mode flicker between the menu and the Plymouth splash.

Then regenerate:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

Fedora unified the config location, so `/boot/grub2/grub.cfg` is correct on UEFI too. The file under `/boot/efi/EFI/fedora/` is a stub that points at it. Do not write your config there.

### A.3 Get it down to exactly two entries

A stock Fedora menu shows three kernels plus a rescue entry plus Windows. That is not the two option picker you asked for.

Limit how many kernels are kept, so the menu stops growing:

```bash
echo 'installonly_limit=2' | sudo tee -a /etc/dnf/dnf.conf
```

Two is the right floor, not one. The previous kernel is your escape hatch when an `akmod-nvidia` rebuild fails, which is exactly the failure mode this laptop is prone to. `GRUB_DISABLE_SUBMENU=true` keeps the old kernel visible but flat rather than nested.

You can also delete the rescue entry from `/boot/loader/entries/*rescue*.conf`, but I would leave it. It costs you one line in the menu and it is the thing that saves you when a normal boot will not come up.

### A.4 Name the Windows entry properly, with the right icon

`os-prober` generates something like `Windows Boot Manager (on /dev/nvme0n1p1)`, and GRUB themes pick icons from the entry's `--class`, so an ugly auto generated title usually means a generic icon too. Write the entry yourself instead.

Find the EFI System Partition UUID:

```bash
lsblk -f | grep -i efi
sudo blkid -s UUID -o value /dev/nvme0n1p1   # use your actual ESP device
```

Add a clean entry:

```bash
sudo tee -a /etc/grub.d/40_custom <<'EOF'
menuentry 'Windows 11' --class windows --class os $menuentry_id_option 'windows' {
    insmod part_gpt
    insmod fat
    insmod chain
    search --no-floppy --fs-uuid --set=root REPLACE_WITH_ESP_UUID
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}
EOF
```

Replace `REPLACE_WITH_ESP_UUID` with the UUID you just read. Then turn `os-prober` back off so you do not end up with two Windows entries:

```bash
sudo sed -i 's/^GRUB_DISABLE_OS_PROBER=false/GRUB_DISABLE_OS_PROBER=true/' /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

`--class windows` is what makes the theme render the Windows icon rather than the fallback. Confirm the theme actually ships one:

```bash
ls /boot/grub2/themes/*/icons/ | grep -iE 'windows|fedora'
```

If `windows.png` is missing, drop a 32px or 64px PNG in with that exact name and it will be used.

### A.5 Pick which OS is the default

Boot into whatever you used last:

```bash
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT=saved/' /etc/default/grub
echo 'GRUB_SAVEDEFAULT=true' | sudo tee -a /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

Or pin Fedora as the default and give yourself a short window to choose Windows: set `GRUB_DEFAULT=0` and `GRUB_TIMEOUT=3`.

### A.6 Match the splash that follows

The menu is only half of what you see. After you pick Fedora, Plymouth draws the boot splash, and by default it shows the firmware logo (`bgrt`). Theme it to match:

```bash
plymouth-set-default-theme --list
sudo dnf install plymouth-theme-spinner
sudo plymouth-set-default-theme spinner -R
```

`-R` rebuilds the initramfs, which is required for the change to take effect. Preview without rebooting:

```bash
sudo plymouthd --tty=/dev/tty2 ; sudo plymouth --show-splash ; sleep 5 ; sudo plymouth --quit
```

### A.7 Test it

Reboot. If the theme does not show, in order of likelihood: `GRUB_TERMINAL_OUTPUT` is still `console`, the `GRUB_THEME` path has a typo, you regenerated to the wrong config path, or the theme's assets do not match the resolution in `GRUB_GFXMODE`.

Recovery if you land at a `grub>` prompt: the menu config is a text file on a filesystem GRUB can already read, so nothing is bricked. Boot from your Fedora USB in live mode, mount the install, and fix `/etc/default/grub`. This is a good reason to keep that USB stick in a drawer.

---

## Route B: rEFInd

What you get: a horizontal row of large OS icons, mouse and touchpad support, automatic detection of Fedora and Windows without any config, and themes that genuinely look designed. Themes worth looking at are [rEFInd Minimalistic](https://github.com/thecodermehedi/refind-minimalistic) and [Ursa Major](https://github.com/kgoettler/ursamajor-rEFInd).

What it costs you, stated plainly:

* rEFInd is not in the Fedora repositories. You install the RPM or zip from [rodsbooks.com](https://www.rodsbooks.com/refind/installing.html) and you update it by hand, forever.
* It is not signed by Microsoft, so with Secure Boot enabled you must install it through shim with locally generated keys and enroll a **second** MOK, separate from the akmods key from Phase 5.
* Upstream's own documentation says the install procedures work best with Secure Boot disabled. Disabling Secure Boot is not an acceptable trade here: Windows 11 wants it, and the whole akmods signing dance in the main plan exists specifically so you can keep it on.

If you accept that, install it after Phase 5 is finished and verified:

```bash
sudo rpm -Uvh refind-<version>.x86_64.rpm
sudo refind-install --shim /boot/efi/EFI/fedora/shimx64.efi --localkeys
```

That signs rEFInd and its drivers with a local key and installs them alongside shim. Reboot, and at the MOK Manager screen enroll `refind_local.crt` from the `keys` directory on the ESP, the same way you enrolled the akmods certificate.

Then add a theme:

```bash
sudo mkdir -p /boot/efi/EFI/refind/themes
sudo git clone https://github.com/thecodermehedi/refind-minimalistic.git \
  /boot/efi/EFI/refind/themes/refind-minimalistic
echo 'include themes/refind-minimalistic/theme.conf' | \
  sudo tee -a /boot/efi/EFI/refind/refind.conf
```

Useful `refind.conf` settings: `timeout 5`, `resolution 1920 1080`, `use_graphics_for linux,windows`, `dont_scan_volumes` to hide recovery partitions, and `scanfor manual,internal` once you have entries the way you want them.

Two failure modes to know about. If rEFInd will not launch with Secure Boot on, the usual cause is a stale extra shim binary left on the ESP from a previous attempt; remove the ones you are not using. And a Windows update can reassert `Windows Boot Manager` as the first firmware boot entry, which puts you back where you started; fix the order with `sudo efibootmgr -o <refind>,<windows>`.

---

## Which one I would actually do

Route A. A good GRUB theme at 1080p with two hand written entries and a matching Plymouth splash looks close enough to rEFInd that the difference is a preference, and it keeps the entire boot chain inside what Fedora signs, updates, and supports. rEFInd is the nicer picker in isolation and the worse choice on a machine where you have just spent an hour getting a signed NVIDIA module working under Secure Boot.

---

## Sources

* [Elegant GRUB2 themes](https://github.com/vinceliuice/Elegant-grub2-themes) — install flags, `-b` for `/boot/grub2`
* [Catppuccin for GRUB](https://github.com/catppuccin/grub) — theme install layout
* [Customizing GRUB](https://www.linuxtechmore.com/how-to-customize-grub) — `GRUB_THEME`, `gfxterm` requirement
* [Fedora discussion: editing vinceliuice GRUB themes](https://discussion.fedoraproject.org/t/how-to-edit-vinceliuices-grub-theme/85292) — Fedora paths and `grub2-mkconfig` target
* [rEFInd installation](https://www.rodsbooks.com/refind/installing.html) — RPM behaviour, `refind-install` flags
* [rEFInd and Secure Boot](https://www.rodsbooks.com/refind/secureboot.html) — shim plus `--localkeys` plus MOK enrollment
* [rEFInd on the Arch Wiki](https://wiki.archlinux.org/title/REFInd) — `refind.conf` options, theme includes
* [rEFInd Minimalistic theme](https://github.com/thecodermehedi/refind-minimalistic) and [Ursa Major theme](https://github.com/kgoettler/ursamajor-rEFInd) — theme install steps
* [Fedora change proposal: systemd-boot for laptops](https://fedoraproject.org/wiki/Changes/UseSystemdBootForDevicesLikeLaptops) — why GRUB is still the default on Fedora 44
* [os-prober and GRUB](https://itsfoss.com/grub-os-prober/) — Windows entry generation
