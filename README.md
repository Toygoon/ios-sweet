# ios-sweet

**Apple's iOS 16 running on a POCO X3 NFC.**

This is a fork of [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet), which ports Apple's open-source XNU kernel (xnu-8796, the iOS 16 generation) to the Xiaomi Redmi Note 10 Pro (`sweet`). This fork runs the same kernel on the **POCO X3 NFC** (`surya`), which shares the Qualcomm SM7150 / Snapdragon 732G platform but has a different touch controller and panel. On top of the kernel runs the **unmodified iOS 16.7.16 (20H392) userland of the iPhone X**: launchd, the system daemons, backboardd, SpringBoard and the stock apps.

As of **2026-10-24** the POCO X3 NFC boots to the **iOS home screen** straight from its `boot` partition, with working touch. Everything below is written for the POCO X3 NFC; the Redmi Note 10 Pro path is in the upstream README. You can unlock it, open Settings, pull down Control Center and the notification center, and use the touchscreen and the hardware buttons. Rendering runs entirely on the CPU; there is no GPU driver.

> Experimental bring-up, not a usable phone OS. Many things are missing or stubbed (see below), and Apple's security model is mostly switched off. It installs into the `userdata` partition and so **erases Android's user data**.

## What works

| Area | State |
| --- | --- |
| Boot | Stock Xiaomi bootloader → our shim (flashed to `boot`, or `fastboot boot` for tests) → XNU |
| CPU | 6 of the 8 cores (the six Cortex-A55s) via PSCI, 16K pages, GICv3 interrupts, timers |
| Memory | About 3.95 GiB of the upper DDR bank |
| Storage | Own UFS host driver; the iOS root is a journaled HFS+ volume in `userdata`, read/write |
| iOS userland | Real iPhone X 16.7.16 root filesystem and shared cache; launchd and the system daemons run |
| Display | Kernel IOSurface + IOMobileFramebuffer (`AppleCLCD`) services; CoreAnimation renders in software; frames are copied into the bootloader's framebuffer. Presented to iOS as an iPhone X panel (1125×2436) |
| GUI | backboardd, SpringBoard, lock screen, home screen, Control Center, notification center, Settings |
| Touch | Novatek NT36672C TDDI over SPI (GENI SE0). The kernel downloads the touch firmware into the IC at every boot (as Android does) and delivers HID digitizer events |
| Buttons | Power, volume up/down (PMIC through SPMI) as HID consumer keys; lock/wake works |
| Battery | IOPMPowerSource service (static 100 %, charging icon; real fuel-gauge readout prepared) |
| Device identity | iPhone X (`iPhone10,3` / `D22AP`): no Home button, gesture UI |
| Activation | Native "Hactivation" path of mobileactivationd, chosen through device-tree security status. No Apple servers involved |

## What does not work (yet)

- **Navigation:** you cannot leave an app (the home gesture and AssistiveTouch Home do nothing yet).
- **No Wi-Fi, Bluetooth, cellular, audio, camera, sensors, GPS or SEP / Face ID.** Daemons for them start and fail or keep relaunching.
- **No GPU.** Everything is drawn by the CPU at roughly 20–30 fps. The two Cortex-A76 cores are not used yet.
- **Few apps.** The image has only what is on the system volume. There is no App Store, no Apple ID, and some Settings pages are empty.
- **Slow boot:** a few minutes on the Apple logo on the first boot of a fresh root (LaunchServices seeds its app database), faster afterwards.
- **Security is off:** permissive code signing (AMFI) and sandbox policies, a stub AppleKeyStore, no data protection. Do not put anything personal on it.

## How it boots

```
ABL (stock Xiaomi bootloader, unlocked)
  -> boot partition = shim-boot.img          (or fastboot boot shim-boot.img for tests)
     -> shim: own MMU setup, reads the live FDT, loads the payload from the ramdisk
        -> XNU Mach-O + generated Apple Device Tree (ADT) + boot-args
           -> XNU -> UFS driver -> HFS+ root in the userdata partition
              -> dyld + shared cache -> launchd -> daemons -> backboardd + SpringBoard
```

There is no serial port (the debug UART is wired as SPI on this board). The kernel draws its console into the framebuffer that ABL leaves at `0x9c000000`, and keeps a persistent 4 MiB log ring at the end of `userdata`. You read that log from TWRP with `tools/sweet-log-pull.sh`, or live over USB with `tools/sweet-usbnet.sh` (the kernel streams its console as UDP over a CDC-ECM link), so you don't have to film the screen.

## How to do it yourself

You need a POCO X3 NFC (`surya`, M2007J20CG, SM7150) with an **unlocked bootloader**, a TWRP image for surya (used only with `fastboot boot`, never flashed), a Linux PC, and a GitHub account for the kernel build. This is the exact path used on a global V14.0.2.0.SJGMIXM phone with the Huaxing NT36672C panel; the Tianma panel variant needs its own touch firmware file (see step 0). **Everything on `userdata` is lost, and once you flash the shim to `boot` Android no longer boots** (you keep a backup to put it back). Bootloaders, recovery, modem, persist and vendor partitions are never touched.

### 0. Back up the phone

Boot TWRP temporarily (`fastboot boot recovery/recovery.img`) and make a full backup, then also pull raw copies of at least `persist`, `modemst1`, `modemst2`, `fsg`, `fsc`, `boot`, `dtbo` and `vbmeta` (`adb exec-out "dd if=/dev/block/by-name/NAME bs=1M" > backup/NAME.img`). They contain per-device calibration data. Keep them off git; `.gitignore` already excludes `backup/`, `dtb/` and `recovery/`.

While in TWRP, copy the touch firmware out of the vendor partition. The IC has no flash of its own; Android (and our kernel) download it at every boot:

```sh
adb shell mount /vendor
adb pull /vendor/firmware/novatek_ts_huaxing_fw.bin research/surya-fw/   # Huaxing panel; Tianma panels use novatek_ts_tianma_fw.bin
```

(Which panel you have: in Android, `getprop ro.boot.lcmtype` or the kernel cmdline says `dsi_nt36672c_huaxing_...` or `..._tianma_...`.)

The shim needs the stock DTB. Unpack your phone's stock `boot.img`:

```sh
python3 tools/unpack_bootv2.py backup/boot.img dtb/boot_unpacked   # -> dtb/boot_unpacked/dtb
```

### 1. Build the kernel

The kernel builds on GitHub's macOS runners with [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build). The workflow fetches xnu-8796, applies `xnu/patches/` in order, and also stages the pinned open-source libpthread, HFS and BearSSL sources it needs. Fork the repo (public repos get free macOS minutes) and run:

```sh
gh workflow run xnu.yml -R YOU/ios-sweet
gh run download RUN_ID -R YOU/ios-sweet -n xnu-kernel -D xnu/out
```

To check a patch locally before spending a CI run, `tools/xnu-cc.py` replays the compile commands of a previous CI log for single files.

### 2. Build the iOS root image

Download the iPhone X (iPhone10,3) iOS 16.7.16 (20H392) IPSW from Apple. Then, on Linux, with `apfs-fuse`, `hfsprogs` and sudo:

```sh
# OS, SystemOS and AppOS disk images from the IPSW (see BuildManifest.plist for their names)
tools/mkiosroot.sh OS.dmg SystemOS.dmg AppOS.dmg ios16-root.img 16G
sudo fsck.hfsplus -fy ios16-root.img
sudo mount -t hfsplus -o loop,force ios16-root.img /mnt/iosroot
sudo python3 tools/sweet-root-tweaks.py /mnt/iosroot
sudo umount /mnt/iosroot
img2simg ios16-root.img ios16-root.simg
```

`mkiosroot.sh` copies the OS volume and both cryptexes into one case-sensitive journaled HFS+ volume. It adds the dyld cache links, the plain system keybag and the log directories the daemons expect. `sweet-root-tweaks.py` applies the bring-up changes: it skips Setup, seeds the activation state, removes the baseband daemon, disables the watchdogs that kill slow apps and the crash-looping media paths, and so on. The script's header explains each change.

The device tree builder copies `/product` from the real iPhone X device tree, which MobileGestalt needs. Extract it once, without downloading the whole IPSW:

```sh
python3 tools/remotezip.py IPSW_URL DeviceTree.d22ap.im4p research/ipsw-x/dt
ipsw img4 im4p extract research/ipsw-x/dt/Firmware/all_flash/DeviceTree.d22ap.im4p -o research/ipsw-x/d22ap.adt
```

### 3. Build the boot image

Needs `aarch64-linux-gnu-gcc` and `mkbootimg`.

```sh
python3 tools/mkadt.py shim/out/sweet.adt --highmem --touch-fw research/surya-fw/novatek_ts_huaxing_fw.bin
python3 tools/mkpayload.py xnu/out/kernel.development.sweet shim/out/sweet.adt shim/out/payload.bin \
    --cmdline 'debug=0xc serial=3 cpus=6 keepsyms=1 -enable_kprintf_spam -noprogress rd=ufsroot0 unentitled_ios_sim_launch=1 ignition_level=1 wdt_test=no_user_checkin ufsrw=1 rootrw=1 launchd_unsecure_cache=1 wdt=-1 sweetfb_noconsole=1 sweetfb_hz=30 sweetusb=2 sweettouch=0 sweetnvt=1 sweetnvtdelay=10 sweetsample=-'
shim/build.sh ../shim/out/payload.bin      # -> shim/out/shim-boot.img
```

What the less obvious boot-args do:

| Boot-arg | Effect |
| --- | --- |
| `rd=ufsroot0 ufsrw=1 rootrw=1` | Root on the `userdata` partition, writable |
| `cpus=6` | Start the six A55 cores; `cpus=1` is the safe fallback |
| `launchd_unsecure_cache=1` | launchd accepts its job cache without AMFI |
| `wdt=-1 wdt_test=no_user_checkin` | No hardware watchdog |
| `sweetfb_noconsole=1` | Stop drawing kernel text over the GUI. It still goes to the persistent log; panics are drawn |
| `sweetfb_hz=30` | Panel refresh rate of the CPU copy |
| `sweetnvt=1` | POCO X3 NFC touch: the Novatek NT36672C SPI driver (firmware from the device tree, `mkadt.py --touch-fw`). `sweetnvtdelay=S` start delay, `sweetnvtdiv=N` SPI clock = 19.2 MHz / N (default 3), `sweetnvtnofw=1` probe only |
| `sweettouch=0`, `sweetkeys=0` | Disable the Redmi Note 10 Pro Goodix touch driver (always on surya) or the button driver |
| `sweetusb=2` | USB device mode: CDC-ECM link to the PC (192.168.7.2) that streams the kernel console (`tools/sweet-usbnet.sh`) |
| `sweetsample=-` | Turn off the periodic user-stack sampler (default samples SpringBoard and backboardd every 2 minutes) |

### 4. Flash the root and boot

```sh
fastboot -S 512M flash userdata ios16-root.simg   # ~5-7 minutes; do not unplug
# Xiaomi's fastboot hangs after a large flash: re-enter fastboot with Power + Volume down
fastboot boot shim/out/shim-boot.img              # temporary test boot
```

The first boot takes a few minutes on the Apple logo (lsd seeds the LaunchServices database), then the lock screen appears. Swipe up to unlock. To get back to fastboot, hold Power + Volume down.

Once it boots the way you want, make it permanent so the phone starts iOS on its own:

```sh
fastboot flash boot shim/out/shim-boot.img       # Android no longer boots; restore with: fastboot flash boot backup/boot.img
fastboot reboot
```

Keep using `fastboot boot` for test kernels; flash `boot` again when a build is good. Never let the stock Android boot with the iOS root in `userdata`: MIUI may reformat it.

### 5. Read the kernel log

```sh
fastboot boot recovery/recovery.img        # TWRP, temporary
tools/sweet-log-pull.sh logs/boot.txt      # 4 MiB console ring from the end of userdata
adb reboot bootloader
```

With `sweetusb=2` the running kernel also streams its console over USB: run `tools/sweet-usbnet.sh logs/live.txt` on the PC before booting. If NetworkManager keeps taking the interface, mark it unmanaged (`nmcli dev set usb0 managed no`) and keep `192.168.7.1/24` on it.

The log contains crash reports with backtraces (`CRASH name ...`), IOKit traces of backboardd and SpringBoard, and the drivers' `SWEET ...` lines. `tools/sweet-crash.py` symbolicates them against the shared cache.

### QEMU

`tools/qemu-sweet.py` boots the same kernel on QEMU `virt` with a test RAM-disk root. It is useful for kernel-only changes; see `NOTES.md`.

## Repository layout

| Path | What |
| --- | --- |
| `shim/` | Bare-metal boot shim (C + asm) packed as an Android boot.img v2 |
| `xnu/patches/` | The XNU patch series (applied in order by CI), plus staging scripts for libpthread, HFS and BearSSL |
| `.github/workflows/xnu.yml` | Kernel CI build |
| `tools/mkadt.py` | Generates the Apple Device Tree from sweet's MMIO layout and the iPhone X identity |
| `tools/mkpayload.py` | Packs kernel + ADT + boot-args into the shim payload |
| `tools/mkiosroot.sh` | Builds the HFS+ iOS root from the IPSW images |
| `tools/sweet-root-tweaks.py` | Bring-up changes to the root (Setup skip, activation, daemons) |
| `tools/sweet-log-pull.sh`, `sweet-log.py` | Read the persistent kernel log from recovery |
| `tools/sweet-crash.py` | Symbolicate console crash reports |
| `tools/xnu-cc.py` | Local single-file compile check from a CI log |
| `tools/qemu-sweet.py` | QEMU test boots with framebuffer text decoding |
| `tools/adt.py`, `remotezip.py`, `unpack_bootv2.py`, `sparsediff.py` | Helpers |
| `NOTES.md` | Detailed engineering log: every boot, crash and decision, plus hardware facts |

Not in git: partition backups, live device trees, recovery images, Apple firmware and root images, and downloaded or generated build trees. They contain device serials and calibration data, or Apple's copyrighted files.

## Kernel patches

| # | What |
| --- | --- |
| 0001–0009 | Toolchain compatibility, SWEET machine config, non-PAC arm64, stubs for Apple-internal pieces |
| 0010–0018 | Early framebuffer console, physmap fix, PRNG without corecrypto, device-tree platform expert, timer PPI, boot CPU driver |
| 0019–0020 | Apple open-source libpthread kernel provider; BearSSL AES |
| 0021–0027 | RAM-disk root, mockfs, SHA-256, minimal AMFI, Apple open-source HFS+, MD5 |
| 0028–0034 | UFS host driver and root device, read/write root, DMA above 4 GiB |
| 0035–0045 | Entitlements, bindfs, AppleKeyStore stub, IOWatchdog, console crash reports |
| 0046 | IOSurface (IOCoreSurfaceRoot) and IOMobileFramebuffer (AppleCLCD) services |
| 0047–0055 | Diagnostics: persistent log, /dev/null tee, IOKit trace, heap strings and user stack sampler on crash; permissive Sandbox policy; HFS rename-swap; AppleKeyStore selectors |
| 0056–0057 | Throttled panel copy; no external identity resolver |
| 0058 | HID event services. Apple's IOHIDFamily-1915.140.3 user client and event queue, matching the iOS 16.7 userland |
| 0059 | iPhone X identity (`hw.product` / `hw.target` from the device tree) |
| 0060 | Power and volume keys over SPMI; touch over GENI I²C |
| 0061 | Present the display as 1125×2436 |
| 0062 | RPMh regulator vote for the touch rail; battery power source |
| 0063 | GENI FIFO packing fix and the Goodix GT9896 protocol |
| 0064 | Grant PRIV_WORK_INTERVAL (UIKit's CoreAnimation work interval) |
| 0065 | SMP: secondary A55 cores via PSCI CPU_ON |
| 0066–0067 | Fault-region crash info; console mute |
| 0068 | IOSurface lookup semantics for already-mapped surfaces |
| 0069 | Batched persistent-log flushes |
| 0070–0081 | Upstream: pid_hibernate, edge touches, display power, USB device mode + CDC-ECM live log, networking traces |
| 0082 | **POCO X3 NFC:** Novatek NT36672C touch over GENI SPI with host firmware download and HW CRC; RPMh vote for its VDDIO (PM6150 L18) |

## Key hardware facts

- RAM: payloads use the upper bank, `0x100000000` + `0xfcc00000` (about 3.95 GiB), above Qualcomm's protected carve-outs.
- GICv3 with security extensions: the timer goes through Group 1 IRQ (virtual timer INTID 19), not FIQ as on Apple silicon.
- PSCI through `smc`, 19.2 MHz timer, ARMv8.2 without pointer authentication. MPIDR uses Aff1 for the core number.
- Panel (surya): NT36672C Huaxing 1080×2400 DSI video mode, continuous splash from ABL, no refresh kick needed.
- Touch (surya): Novatek NT36672C TDDI over SPI mode 0 on QUP wrapper 0 SE0 (`0x880000`, pins gpio49–52 `qup00`), reset gpio8, interrupt gpio9 (polled), VDDIO PM6150 L18 through RPMh; the panel rails (LAB/IBB) are left on by ABL. Trim ID `0c 00 00 72 66 03`, NT36675 memory map, firmware `novatek_ts_huaxing_fw.bin` (ILM + DLM + info partitions, HW CRC).
- Touch (sweet, upstream): Goodix GT9896 at I²C `0x5d` on QUP SE7 (`0xa84000`), powered by PM6150L LDO11 through RPMh.
- Buttons: power and volume down on the PM6150 PON block, volume up on PM6150L GPIO2 (SPMI SID 4). Same on both phones.
- `userdata` is `sda16` on surya (`sda34` on sweet); the kernel finds it by GPT name, only the manual `adb` commands care.
- The device tree must carry MAC addresses (`/chosen mac-address-*`, `/arm-io/wlan` and `bluetooth` `local-mac-address`): libMobileGestalt otherwise waits forever for an `IONetworkController` and lsd never finishes seeding the app database, so SpringBoard stays on the Apple logo. `tools/mkadt.py` emits locally administered addresses.

More in [`NOTES.md`](NOTES.md).

## Credits

The port itself is [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet). Built on Apple's open-source releases (xnu, IOHIDFamily, libpthread, hfs), [BearSSL](https://bearssl.org), [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build), [ipsw](https://github.com/blacktop/ipsw), Qualcomm/Xiaomi's published Linux kernel sources for the hardware details (the Novatek `nt36xxx_spi` driver in Xiaomi's `surya-q-oss` tree for the touch protocol), [OrangeFox Recovery](https://orangefox.download) and [TWRP](https://twrp.me). iOS itself is Apple's and is not distributed here.
