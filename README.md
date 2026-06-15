# CachyOS Kernel Debian Builder

This repository contains a GitHub Actions workflow for building the latest
CachyOS Linux kernel as installable Debian packages.

The workflow is intended for Debian-based desktop systems where you want to test
or run a CachyOS-flavored kernel without manually maintaining the full build
toolchain on your machine.

## What It Builds

The workflow fetches the newest kernel metadata from the official
[CachyOS/linux-cachyos](https://github.com/CachyOS/linux-cachyos) packaging
repository, downloads the matching CachyOS kernel source tarball, applies a
Debian-oriented desktop configuration pass, compiles the kernel, and packages the
result with the kernel's upstream Debian packaging target.

Expected output includes standard `.deb` packages such as:

- `linux-image-*.deb`
- `linux-headers-*.deb`
- related generated kernel packages produced by `make bindeb-pkg`
- `BUILD-MANIFEST.txt` with source/version details and SHA256 checksums

## Workflow Trigger

The workflow is manual-only.

It does not run on pushes, pull requests, tags, or a schedule. A repository
maintainer must explicitly start it from the GitHub Actions tab with the
`workflow_dispatch` trigger.

## Manual Usage

1. Open the repository on GitHub.
2. Go to **Actions**.
3. Select **Build CachyOS Kernel Debian Packages**.
4. Click **Run workflow**.
5. Choose the desired inputs:
   - `runner`: GitHub-hosted or Blacksmith runner size.
   - `kernel_variant`: CachyOS package variant to build.
   - `cpu_scheduler`: scheduler/config flavor.
   - `run_qemu_smoke_test`: whether to boot-test the built kernel in QEMU.
6. Wait for the build to finish.
7. Download the generated artifacts from the workflow run.

Kernel builds are large and slow. A full run can take several hours and uses a
significant amount of GitHub-hosted runner disk space.

## Available Build Inputs

### `runner`

Supported values:

- `blacksmith-8vcpu-ubuntu-2404`
- `blacksmith-16vcpu-ubuntu-2404`
- `blacksmith-32vcpu-ubuntu-2404`
- `ubuntu-24.04`

The default is `blacksmith-16vcpu-ubuntu-2404`, which is a good fit for kernel
compilation because it provides substantially more CPU, memory, and disk space
than the default GitHub-hosted Ubuntu runner.

Blacksmith runners require the Blacksmith GitHub integration to be installed and
enabled for the repository's organization. If Blacksmith is not configured, use
the `ubuntu-24.04` fallback runner.

### `kernel_variant`

Supported values:

- `linux-cachyos`
- `linux-cachyos-bore`
- `linux-cachyos-eevdf`
- `linux-cachyos-lts`

The default is `linux-cachyos`.

### `cpu_scheduler`

Supported values:

- `cachyos`
- `bore`
- `eevdf`

The default is `cachyos`.

### `run_qemu_smoke_test`

When enabled, the workflow extracts the generated kernel image package, creates a
minimal BusyBox initramfs, and boots the kernel with QEMU. The test passes only
if the guest reaches the init process and prints a success marker on the serial
console. The workflow enables the 8250 serial console path in the kernel config
so this direct QEMU boot test can report reliably.

The default is enabled for manual runs.

## Desktop-Oriented Kernel Configuration

The workflow starts from the CachyOS kernel configuration and then ensures common
Debian desktop requirements are available. The configuration pass includes
support for:

- initramfs booting
- loadable kernel modules
- EFI systems
- dynamic preemption and high-resolution timers
- schedutil CPU frequency governor
- common cgroups, BPF, and pressure-stall interfaces
- Intel, AMD, Nouveau, and VirtIO graphics modules
- common USB, HID, audio, webcam, Bluetooth, and Wi-Fi modules
- common filesystems such as ext4, Btrfs, XFS, F2FS, exFAT, NTFS3, NFS, CIFS,
  OverlayFS, and SquashFS
- KVM, VirtIO, and VFIO modules
- WireGuard and common networking features

The build uses a generic x86-64 target so the resulting packages are more
portable across Debian-based machines.

## Validation

The workflow performs package validation before uploading artifacts:

- verifies that `.deb` files were produced
- verifies that image and header packages exist
- inspects packages with `dpkg-deb`
- runs `lintian --fail-on error` with a narrow allowlist for expected
  kernel-header helper binaries
- checks for a packaged `vmlinuz-*`
- checks for packaged kernel modules
- checks for installed header files
- optionally performs a minimal QEMU boot smoke test

## Release Assets

Artifacts are always uploaded to the workflow run.

If a maintainer manually runs the workflow against a tag ref, the workflow can
also publish the generated `.deb` packages and `BUILD-MANIFEST.txt` as GitHub
Release assets.

## Installing Packages

Download the generated `.deb` files from the workflow artifacts, then install the
image and headers on a Debian-based system:

```bash
sudo apt install ./linux-image-*.deb ./linux-headers-*.deb
sudo update-grub
sudo reboot
```

After rebooting, confirm the running kernel:

```bash
uname -a
```

Keep a known-good distribution kernel installed so you can select it from the
bootloader if the custom kernel does not work on your hardware.

## Important Notes

- This project repackages CachyOS kernel sources for Debian-based systems; it is
  not an official CachyOS project.
- CachyOS kernel sources and packaging metadata come from upstream CachyOS
  repositories at build time.
- Kernel compatibility depends on your hardware, firmware, bootloader, Secure
  Boot configuration, DKMS modules, and distribution release.
- Secure Boot users may need to sign the generated kernel or disable Secure Boot,
  depending on local policy.
- Proprietary or out-of-tree modules, such as NVIDIA DKMS modules, are not built
  by this workflow.

## Repository Layout

```text
.github/workflows/build-cachyos-kernel.yml
README.md
```

The workflow is self-contained and fetches the CachyOS kernel sources during the
GitHub Actions run.
