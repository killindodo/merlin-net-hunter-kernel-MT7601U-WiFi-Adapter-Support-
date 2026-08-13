                                MT7601U WiFi Adapter Support on Kali NetHunter (Redmi Note 9 / merlin)

Enabling a MediaTek MT7601U USB WiFi dongle on a custom Kali NetHunter kernel for the Redmi Note 9, by finding, patching, and building the kernel with driver support compiled in.

TL;DR

The community NetHunter kernel for the Redmi Note 9 didn't ship with CONFIG_MT7601U enabled, so the dongle enumerated on USB but had no driver to bind to it. The fix was to clone the kernel source, enable the config option, build it with proton-clang, and flash the resulting kernel. The adapter now works in managed and monitor mode.

Environment
Component	Details
Device	Redmi Note 9 (codename merlin)
ROM	MIUI 12.5, Android 11
Kernel (before)	NetHunter kernel 4.14.186-g284095c
Kernel repo	amamarante92/nethunter-kernel-redminote9
Adapter	MT7601U USB WiFi dongle (148f:7601, "802.11 n WLAN")
Toolchain	kdrag0n/proton-clang
The Problem
modprobe mt7601u
modprobe: FATAL: Module mt7601u not found in directory /lib/modules/4.14.186-g284095c

The dongle was fully visible at the USB level (idVendor: 148f, idProduct: 7601), but no driver module existed for the running kernel — meaning the kernel build itself was compiled without MT7601U support. This wasn't a config toggle or firmware issue; the module simply didn't exist.

Diagnosis Path

Getting to that conclusion took a few dead ends, in order:

lsusb / dmesg — initially returned nothing useful; lsusb was empty, and dmesg only showed unrelated Android USB gadget-mode noise.
iw dev — only showed the phone's internal WiFi interfaces (ap0, p2p0, wlan1, wlan0), confirming the dongle wasn't bound to anything.
/sys/bus/usb/devices/ — showed the kernel did see USB devices, even though lsusb reported none. This pointed to lsusb/usbutils being broken, not a missing device.
/dev/bus/usb/001/ — had device nodes (001, 015, 016, 017, 018), confirming the dongle was enumerated at the kernel level.
Direct sysfs read (idVendor/idProduct/product per device) — identified the dongle as 148f:7601 on port 1-1.4.
Driver binding check (/sys/bus/usb/devices/1-1.4:1.0/driver) — no symlink existed, meaning no driver had ever attempted to claim the interface.
modprobe mt7601u — gave the definitive answer: the module doesn't exist for this kernel build.
Root Cause

The kernel's defconfig(s) never had CONFIG_MT7601U set — not even as a disabled entry. Even the fresh official prebuilt release from the kernel repo didn't include it. The driver source was present in the kernel tree (drivers/net/wireless/mediatek/mt7601u/mt7601u.h), it just was never being compiled in.

The Fix: Build a Custom Kernel
1. Clone the kernel source
bash
sudo apt update && sudo apt install -y git build-essential bc bison flex libssl-dev
git clone https://github.com/amamarante92/nethunter-kernel-redminote9.git -b nethunter
cd nethunter-kernel-redminote9
2. Get the toolchain (proton-clang)

git clone on proton-clang (~1.6GB+) kept failing partway through on an unstable connection. Downloading it as a resumable zip archive worked better:

bash
mkdir -p clang-13
cd clang-13
wget -c https://github.com/kdrag0n/proton-clang/archive/refs/heads/master.zip -O proton-clang.zip
unzip proton-clang.zip
mv proton-clang-master aaa
rm proton-clang.zip
cd ..

Gotcha: GitHub's zip-archive URL needs refs/heads/master.zip, not refs/master.zip — the latter silently 404s.

3. Find the correct defconfig and enable MT7601U

The build script (merlin.sh) uses nethunter_defconfig, not merlin_defconfig or xiaomi/merlin.config (both of which exist in the tree but aren't what's actually used — easy to edit the wrong file, which happened here first).

bash
find arch/arm64/configs -iname "*nethunter*"
grep -i "MT7601" arch/arm64/configs/nethunter_defconfig

If missing, add it as built-in:

bash
echo "CONFIG_MT7601U=y" >> arch/arm64/configs/nethunter_defconfig
grep -n "MT7601" arch/arm64/configs/nethunter_defconfig

Gotcha: if you use sed/echo to edit a file whose last line has no trailing newline, >> will append onto the end of that last line instead of starting a new one, corrupting the config. Always check with tail after editing.

4. Fix the build script's PATH order

merlin.sh puts the proton-clang bin/ directory first in PATH. This breaks the build, because a separate step compiles small host-side tools (like scripts/kconfig/conf) using the system's own gcc, and that step then picks up proton-clang's bundled ld instead of the system linker — which is too old to understand the RELR relocation format used by modern glibc:

unknown type [0x13] section '.relr.dyn'

Fix — append the clang path instead of prepending it, so the system tools are found first:

bash
sed -i 's|^export PATH=.*|export PATH="${PATH}:${PWD}/clang-13/aaa/bin"|' merlin.sh

Gotcha: editing shell scripts in nano on a phone/tablet keyboard can silently drop the leading # of the shebang line (#!/bin/bash → !/bin/bash), which breaks the script's interpreter. Always head -1 the script after editing to confirm.

5. Build
bash
chmod +x merlin.sh
./merlin.sh

A successful build ends with Image.gz-dtb being packed into an AnyKernel3 flashable zip:

bash
ls -la ~/nethunter-kernel-redminote9/AnyKernel3-master/*.zip
6. Flash
Flash via OrangeFox recovery (not generic TWRP — MIUI/merlin devices are picky about recovery compatibility).
Make sure the matching vbmeta patch is flashed too, or you'll bootloop.
Back up your current working kernel first.
Verification

After flashing and plugging in the dongle:

bash
uname -r                 # confirms the new kernel build
dmesg | tail -30         # watch driver init messages
iw dev                   # look for a new interface

Result — a new interface bound to the dongle:

phy#6
    Interface wlan2
        addr 00:e0:2d:84:4f:3f
        type managed

Monitor mode also works:

bash
airmon-ng start wlan2    # answer 'n' if it prompts about phy0 (that's the phone's internal WiFi — leave it alone)
iw dev wlan2 info        # confirm "type monitor"
airodump-ng wlan2        # confirm live packet capture

Known Quirks / Notes

A transient mt7601u ... Vendor request req:07 off:101c failed:-71 (EPROTO) can show up during init. In this case the driver retried and completed initialization successfully — it wasn't fatal. If it is persistent/fatal for you, suspect USB power delivery (try a powered OTG hub) or a marginal OTG cable/connector.
airmon-ng sometimes prompts about phy0 (the phone's internal WiFi chip) — always decline that (n) unless you actually mean to touch the internal adapter.
If you're on zsh rather than bash, patterns like .[!.]* break due to ! history expansion. Use .??* instead when moving hidden files.
iwconfig may fail to install due to a wireless-tools/systemd dependency conflict — not needed; iw covers the same functionality and is the modern replacement.

Credits

Kernel base: amamarante92/nethunter-kernel-redminote9

Toolchain: kdrag0n/proton-clang

Debugged and built by @killindodo
