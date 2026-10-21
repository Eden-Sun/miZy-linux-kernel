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
cd /workspace/build/artifacts
cp /workspace/kernel/mainline/ram-test.its .
mkimage -f ram-test.its mizy-6.12.113-ram-test.itb
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

`ram-test.its` combines the kernel, an initramfs bootstrap and the original-layout
DTB. The bootstrap reads the original squashfs from SPI without modifying it,
adds mainline files as a separate lower layer, and imports the saved miZy overlay
through miZy's preinit. Its serial console device nodes are explicitly included.

`ymodem-upload.py IMAGE` uploads to U-Boot at RAM address `0x48000000` and runs
`iminfo`. It does not boot or flash the image. It currently targets the macOS
serial path `/dev/cu.usbserial-1110` at 115200 baud. Close other serial clients
before using it.

Do not write SPI until the RAM test has verified boot, Wi-Fi, DHCP, DNS, proxy
requests and the reverse SSH tunnel. Keep the original bootloader and a verified
full SPI backup available for recovery.
