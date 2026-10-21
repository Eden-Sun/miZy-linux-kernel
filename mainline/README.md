# Orange Pi Zero H2+ mainline bring-up

This is a **test candidate**, not a completed device upgrade. It keeps miZy's
userspace rather than installing Armbian. See the parent project's status record
for actual hardware test results.

## Inputs

- Linux 6.12.113 LTS, pinned by SHA-256 in `prepare.sh`.
- fifteenhex/xradio commit `3ce657ae533e2213f6fda2297da1598fa11d7a03`, also hash-checked.
- The XR819 interrupt/device-tree adjustment used by Armbian, bundled as a patch.
- Original miZy squashfs extracted locally from a verified SPI backup; it is not
  distributed in this repository and can contain device-specific information.

The tested build environment is Ubuntu 24.04 ARM64 with GCC 13.3's
`arm-linux-gnueabihf` cross compiler. `Containerfile` lists the dependencies. Keep
sources and the build directory on the container's native, case-sensitive Linux
filesystem; use the `/workspace` bind mount only for scripts and output.

## Build

From the parent miZy fork, mount that checkout at `/workspace` in the container.
The following commands are run **inside** the prepared container:

```sh
bash /workspace/kernel/mainline/prepare.sh
bash /workspace/kernel/mainline/build.sh
```

To package the existing miZy rootfs, first supply its squashfs payload at
`/workspace/build/downloads/mizy-original-rootfs.bin`. The SPI `initrd` partition
has a 64-byte uImage header before the squashfs; verify the image length/header
before extracting it. Then run:

```sh
unsquashfs -d /work/mizy-rootfs /workspace/build/downloads/mizy-original-rootfs.bin
bash /workspace/kernel/mainline/package-rootfs.sh
python3 /workspace/kernel/mainline/pack-ram.py /workspace/build/artifacts
```

These build commands do not flash the device. Outputs and local backups belong
under the ignored `build/` directory, never in git.

## Changes needed alongside the kernel

- Add XR819 and matching cfg80211/mac80211 modules; never reuse 3.4 modules.
- Enable SPI NOR, JFFS2, squashfs, RAM disks, TUN and legacy iptables support.
- Replace the old overlayfs mount syntax with modern `overlay` plus a workdir.
- Use device-tree power/interrupt setup instead of legacy Wi-Fi GPIO/register
  writes; keep the saved miZy Wi-Fi credentials and DHCP callback.
- Disable the legacy direct CPU/DRAM register initialization in mainline mode.
- Recompress the preserved miZy userspace with xz rather than lzo.

The first successful build produced a 3.64 MiB uImage and a 7.01 MiB uInitrd.
They fit a proposed 4 MiB kernel slot and the remaining initrd area while keeping
the user partition at its original address. The current `board-spi.dtsi` retains
**the original layout for read-only RAM testing**; it must not be used as the
permanent resized-layout DTB.

## RAM test

`pack-ram.py` combines legacy uImage, uInitrd and DTB at aligned addresses,
recorded in `ram-legacy.json`. The installed U-Boot does not support FIT images.
The bootstrap reads the original squashfs and saved user JFFS2 partition without
modifying SPI. It imports saved settings before mounting OverlayFS, with mainline
files as a separate lower layer. JFFS2 needs LZO compression and 64 KiB SPI erase
blocks to read the existing filesystem. The bootstrap includes its dynamic loader,
libgcc and serial device nodes.

`ymodem-upload.py IMAGE` uploads to U-Boot at RAM address `0x48000000` and runs
`iminfo`. It does not boot or flash the image. It currently targets the macOS
serial path `/dev/cu.usbserial-1110` at 115200 baud. Close other serial clients
before using it.

Do not write SPI until the RAM test has verified boot, Wi-Fi, DHCP, DNS, proxy
requests and the reverse SSH tunnel. Keep the original bootloader and a verified
full SPI backup available for recovery.

## Assemble a private SPI candidate

After the build, `assemble-spi.py BACKUP ARTIFACTS --wifi-mac DEVICE_MAC` creates a resized-layout DTB,
boot script, full candidate and payload files sized for the original MTD slots.
It checks that U-Boot and the saved user partition remain byte-identical to the
backup. The full candidate contains the backup's private configuration and must
remain outside git. Assembly alone is **not** evidence of hardware compatibility.

## Hardware results (2026-10-21)

Linux 6.12.113 booted on all four H2+ cores, entered miZy procd, read the original
saved JFFS2 overlay, and connected XR819 to WPA2 Wi-Fi with DHCP and DNS. An HTTPS
request through the reverse SSH relay returned CONNECT 200 and HTTPS 200.
Disable the old Wi-Fi hotplug restart hook and explicitly load the crypto modules:
the legacy modprobe does not resolve their algorithm aliases. The permanent DTB
provisions the saved MAC before interface registration and removes the ethernet1
alias so old U-Boot cannot overwrite it. A legacy squashfs initrd uses chroot for
its init handoff; only the cpio RAM bootstrap uses switch_root.

These RAM results do not by themselves establish successful permanent SPI boot.
