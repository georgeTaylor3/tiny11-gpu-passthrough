# KVM and Tiny11 VM Setup

## Purpose

This document describes the baseline Tiny11 virtual machine configuration used
before GPU passthrough was added.

The VM runs on Linux Mint using:

- KVM
- QEMU
- libvirt
- virt-manager

The goal was to create a stable Windows VM first, verify storage, networking,
SPICE integration, and guest drivers, and only then add PCI passthrough.

## Host Virtualization Support

Hardware virtualization was verified before creating the VM.

The host CPU supports Intel VT-x, and the KVM device is available at:

    /dev/kvm

The user account is a member of the required virtualization groups:

    kvm
    libvirt

This allows normal management of KVM/libvirt virtual machines without running
the entire desktop virtualization workflow as root.

## Virtual Machine

The virtual machine is named:

    tiny11

The VM uses a Q35 machine type and UEFI firmware.

Baseline resources:

- Memory: 12 GiB
- Virtual CPUs: 6
- Disk size: 150 GiB
- Disk format: qcow2
- Firmware: UEFI / OVMF
- Secure Boot: disabled

The virtual disk is stored under:

    /var/lib/libvirt/images/

The exact host filename is intentionally omitted from this documentation where
it is not required.

## Why Q35 Was Used

The Q35 machine type provides a more modern PCI Express-oriented virtual
platform than older i440fx-based machine types.

This is useful for PCI passthrough because the guest receives devices through a
PCIe-style topology that more closely matches modern hardware.

## UEFI Firmware

The VM uses OVMF UEFI firmware rather than legacy BIOS firmware.

UEFI was selected because it provides a modern guest boot environment and
works well with Windows 11-based guests and PCI passthrough configurations.

Secure Boot was not enabled for this VM.

## Tiny11 Installation

Tiny11 was installed from an ISO attached to the VM as virtual installation
media.

During installation, Windows did not initially see the VirtIO-backed system
disk.

The VirtIO driver ISO was therefore attached to the VM, and the storage driver
was loaded manually from the installer.

After loading the VirtIO storage driver, the virtual disk became visible and
Windows installation proceeded normally.

## VirtIO Storage

The VM uses VirtIO-based storage rather than emulated SATA or IDE storage.

VirtIO reduces virtualization overhead by using paravirtualized drivers designed
specifically for virtual machines.

Windows requires the corresponding VirtIO driver before it can access the disk
during installation.

## Networking

The Windows guest uses a VirtIO network adapter.

The NetKVM driver was installed inside Tiny11 from the VirtIO driver package.

After installation, Tiny11 successfully obtained network connectivity.

## SPICE Integration

SPICE is retained as the VM console and management display.

SPICE provides:

- a reliable virtual display
- clipboard integration
- USB redirection
- a fallback console if the passed-through GPU is unavailable

SPICE guest tools were installed inside Tiny11.

Bidirectional clipboard functionality was verified between Linux Mint and the
Windows guest.

## Why SPICE Was Kept After GPU Passthrough

The virtual SPICE display was intentionally retained after the physical NVIDIA
GPU was added to the VM.

This provides a recovery and management path independent of the passed-through
GPU.

The resulting design provides two graphics paths:

    Tiny11
      |
      +--> SPICE virtual display
      |
      +--> NVIDIA GTX 1660 Ti passthrough

The SPICE display remains useful for administration even when the physical GPU
is assigned directly to Windows.

## USB Redirection Capability

SPICE USB redirection is enabled in the VM.

USB devices remain owned by Linux Mint by default and can be redirected
individually to Tiny11 when needed.

This is documented in more detail in:

    docs/08-usb-redirection.md

## Initial Validation

Before configuring GPU passthrough, the following VM functions were verified:

- Tiny11 boots successfully
- VirtIO storage works
- VirtIO networking works
- Internet access works
- SPICE display works
- clipboard sharing works
- USB redirection works
- clean Windows shutdown works
- VM restart works

Establishing a stable baseline before PCI passthrough made later troubleshooting
significantly easier.

## Snapshot

A disk-only snapshot was created before GPU passthrough changes.

This provided a recovery point in case PCI passthrough configuration or Windows
driver installation caused problems.

The snapshot was created before the NVIDIA GPU was added to the guest.

## Design Principle

The VM was built and validated in stages:

    KVM / libvirt
          |
          v
    Tiny11 installation
          |
          v
    VirtIO storage
          |
          v
    VirtIO networking
          |
          v
    SPICE integration
          |
          v
    Baseline validation
          |
          v
    GPU passthrough

This staged approach reduced the number of variables involved during
troubleshooting.
