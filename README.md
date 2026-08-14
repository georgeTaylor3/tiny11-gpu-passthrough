# Tiny11 GPU Passthrough on Linux Mint

A documented implementation of NVIDIA GPU passthrough from Linux Mint to a
Tiny11 virtual machine using KVM/QEMU, libvirt, VFIO, and Intel IOMMU.

## Purpose

This project documents a working dual-mode GPU configuration on an Optimus-style
laptop.

The system can boot into one of two modes:

### Linux Mint mode

The NVIDIA GPU is controlled by the native Linux NVIDIA driver and remains
available to Linux through PRIME on-demand.

### Tiny11 passthrough mode

The NVIDIA GPU and its associated PCI functions are bound to `vfio-pci` during
boot and passed directly to the Tiny11 virtual machine.

Switching modes requires a reboot. This was intentionally chosen instead of
live GPU rebinding because it provides more predictable device ownership and
avoids reset and power-management issues encountered with this hardware.

## Hardware

- CPU: Intel Core i7-9750H
- Integrated GPU: Intel UHD Graphics 630
- Dedicated GPU: NVIDIA GeForce GTX 1660 Ti Mobile
- RAM: 32 GiB
- Host OS: Linux Mint
- Hypervisor: KVM/QEMU
- VM management: libvirt / virt-manager
- Guest OS: Tiny11

## Repository Layout

- `docs/` — implementation and troubleshooting documentation
- `scripts/` — GPU mode and VFIO helper scripts
- `systemd/` — systemd unit files
- `modprobe/` — kernel module configuration
- `udev/` — VFIO PCI binding rules
- `examples/` — sanitized configuration examples

## Security

Sensitive information, credentials, private keys, VM disk images, ISOs, and
other secrets are intentionally excluded from this repository.

See [SECURITY.md](SECURITY.md).

## Disclaimer

This configuration is hardware-specific. PCI addresses, device IDs, IOMMU
groups, driver behavior, firmware behavior, and power-management characteristics
can differ significantly between systems.

## Repository Structure

- `docs/` — Step-by-step documentation covering hardware, IOMMU, KVM/libvirt, GPU passthrough, D3cold troubleshooting, GPU mode switching, startup safeguards, USB redirection, troubleshooting, and daily usage.
- `scripts/` — Helper scripts for GPU mode switching, NVIDIA USB VFIO binding, and repository security scanning.
- `systemd/` — systemd unit used to ensure the NVIDIA USB controller is bound to `vfio-pci`.
- `udev/` — udev rules that assign the NVIDIA PCI functions to `vfio-pci` when passthrough mode is enabled.
- `modprobe/` — VFIO kernel module configuration, including the NVIDIA D3cold workaround.
- `libvirt/` — libvirt hook that prevents the Tiny11 VM from starting unless all required NVIDIA PCI functions are owned by `vfio-pci`.
- `examples/` — Sanitized libvirt XML examples showing the PCI passthrough configuration without exposing host-specific VM information.
- `security-scan-patterns.txt` — Public patterns used by the repository security scanner for human-reviewable findings.
- `security-scan-critical-patterns.txt` — Higher-risk patterns displayed as critical findings by the repository security scanner.
- `SECURITY.md` — Documents the repository's security scope and the types of sensitive information that must never be committed.
- `.gitignore` — Prevents VM disks, installation media, keys, credentials, logs, temporary files, and local security-scanner data from being committed.

## File Installation Map

The files in this repository are reference copies of the configuration used
for this project.

Files that configure the Linux host must be copied to their corresponding
system locations before they can take effect.

| Repository File | Destination on Linux Mint |
|---|---|
| `scripts/gpu-mode` | `/usr/local/sbin/gpu-mode` |
| `scripts/bind-nvidia-usb-vfio` | `/usr/local/sbin/bind-nvidia-usb-vfio` |
| `systemd/nvidia-usb-vfio.service` | `/etc/systemd/system/nvidia-usb-vfio.service` |
| `udev/99-vfio-pci.rules` | `/etc/udev/rules.d/99-vfio-pci.rules` |
| `modprobe/vfio-pci.conf` | `/etc/modprobe.d/vfio-pci.conf` |
| `libvirt/hooks/qemu.d/10-tiny11-gpu-safety` | `/etc/libvirt/hooks/qemu.d/10-tiny11-gpu-safety` |

### Repository-Only Files

The following files are not installed into system directories and should
remain in the cloned repository:

- `README.md`
- `SECURITY.md`
- `.gitignore`
- `docs/`
- `security-scan-patterns.txt`
- `security-scan-critical-patterns.txt`
- `scripts/security-scan`

The optional local file:

```text
security-scan-patterns.local
```

is intended for machine-specific or user-specific scan patterns and must
remain excluded from Git.

### Libvirt XML Example

The file:

```text
examples/tiny11-hostdev-snippets.xml
```

is a sanitized reference example.

It should not be copied directly into `/etc`.

The relevant `<hostdev>` entries should instead be adapted to the user's
virtual machine configuration using `virsh edit` or virt-manager.

### Install the Host Configuration Files

From the root of the cloned repository:

```bash
sudo install -m 755 \
    scripts/gpu-mode \
    /usr/local/sbin/gpu-mode

sudo install -m 755 \
    scripts/bind-nvidia-usb-vfio \
    /usr/local/sbin/bind-nvidia-usb-vfio

sudo install -m 644 \
    systemd/nvidia-usb-vfio.service \
    /etc/systemd/system/nvidia-usb-vfio.service

sudo install -m 644 \
    udev/99-vfio-pci.rules \
    /etc/udev/rules.d/99-vfio-pci.rules

sudo install -m 644 \
    modprobe/vfio-pci.conf \
    /etc/modprobe.d/vfio-pci.conf

sudo install -D -m 755 \
    libvirt/hooks/qemu.d/10-tiny11-gpu-safety \
    /etc/libvirt/hooks/qemu.d/10-tiny11-gpu-safety
```

### Additional Host Configuration

Copying the repository files is only part of the setup.

This project also requires host-specific configuration that cannot safely
be installed as a generic copy operation, including:

- enabling IOMMU
- configuring the kernel command line in `/etc/default/grub`
- loading the required VFIO modules through `/etc/initramfs-tools/modules`
- rebuilding the initramfs
- regenerating the GRUB configuration
- enabling the NVIDIA USB VFIO systemd service
- adapting PCI device IDs to the user's hardware
- configuring the virtual machine's PCI host devices
- installing the NVIDIA driver inside Windows
- verifying device ownership before starting the VM

These steps are documented in the files under `docs/`.

### Important

The PCI device IDs and addresses used in this repository are specific to the
hardware used for this project.

Do not blindly copy PCI IDs such as:

```text
10de:2191
10de:1aeb
10de:1aec
10de:1aed
```

or PCI addresses such as:

```text
01:00.0
01:00.1
01:00.2
01:00.3
```

onto another system.

Users should identify their own GPU and associated PCI functions before
modifying VFIO, udev, GRUB, systemd, or libvirt configuration.
