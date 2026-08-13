# Hardware and IOMMU

## Host Hardware

This GPU passthrough configuration was implemented on the following system:

- CPU: Intel Core i7-9750H
- Integrated GPU: Intel UHD Graphics 630
- Dedicated GPU: NVIDIA GeForce GTX 1660 Ti Mobile
- Memory: 32 GiB
- Host OS: Linux Mint
- Hypervisor: KVM/QEMU
- VM management: libvirt / virt-manager
- Guest OS: Tiny11

The integrated Intel GPU remains available to the Linux host while the
dedicated NVIDIA GPU is assigned to the Windows VM.

## NVIDIA PCI Functions

The NVIDIA hardware exposes four PCI functions:

    01:00.0  NVIDIA GeForce GTX 1660 Ti
    01:00.1  NVIDIA HD Audio
    01:00.2  NVIDIA USB 3.1 Host Controller
    01:00.3  NVIDIA USB Type-C UCSI Controller

The corresponding vendor/device IDs are:

    10de:2191
    10de:1aeb
    10de:1aec
    10de:1aed

All four NVIDIA functions are assigned together when GPU passthrough is
enabled.

## IOMMU

Intel IOMMU support is enabled with the following kernel parameters:

    intel_iommu=on iommu=pt

The IOMMU provides DMA isolation between hardware assigned to a virtual
machine and memory owned by the Linux host.

## IOMMU Groups

PCI devices are organized into IOMMU groups.

Devices within the same IOMMU group may not always be safely isolated from
one another. Because of this, the PCI topology and IOMMU group layout were
inspected before configuring passthrough.

The NVIDIA device functions and their upstream PCIe topology were verified
before assigning the hardware to VFIO.

## Why the Integrated GPU Matters

This system uses an Optimus-style graphics configuration.

Linux Mint can continue using the Intel integrated GPU while the NVIDIA GPU
is reserved for Tiny11.

This allows the Linux desktop to remain usable while the dedicated GPU is
passed directly to the Windows virtual machine.

## Expected Driver Ownership

In normal Linux Mint mode, the NVIDIA-related devices are typically owned by
their native Linux drivers:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

In Tiny11 passthrough mode, all four NVIDIA functions are expected to be
owned by:

    vfio-pci

This separation ensures that Linux and the Windows VM do not attempt to use
the same PCI hardware at the same time.
