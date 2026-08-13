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
