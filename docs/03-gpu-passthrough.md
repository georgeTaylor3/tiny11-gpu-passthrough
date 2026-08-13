# GPU Passthrough Architecture

## Purpose

This document explains how the NVIDIA GeForce GTX 1660 Ti Mobile is assigned
directly to the Tiny11 virtual machine.

The passthrough design uses:

- Intel IOMMU
- VFIO
- `vfio-pci`
- KVM/QEMU
- libvirt PCI host devices
- the native NVIDIA Windows driver

The goal is to allow Tiny11 to use the physical NVIDIA GPU directly rather than
relying only on an emulated or paravirtualized graphics adapter.

## Normal Virtual Graphics vs GPU Passthrough

A normal virtual machine typically uses a virtual graphics adapter.

Conceptually:

    Windows VM
        |
        v
    virtual GPU
        |
        v
    QEMU / hypervisor
        |
        v
    Linux host
        |
        v
    physical GPU

With PCI GPU passthrough, the design is different:

    Windows VM
        |
        v
    physical NVIDIA GPU

The physical device is exposed directly to the guest through VFIO and the
IOMMU.

This allows Windows to load the normal NVIDIA driver and detect the real GPU.

## VFIO

VFIO provides a secure framework for assigning physical devices to virtual
machines.

In normal Linux Mint mode, the NVIDIA GPU is controlled by the native Linux
driver:

    NVIDIA GPU
        |
        v
    nvidia
        |
        v
    Linux Mint

In Tiny11 passthrough mode, the GPU is instead claimed by:

    vfio-pci

The ownership path becomes:

    NVIDIA GPU
        |
        v
    vfio-pci
        |
        v
    QEMU / KVM
        |
        v
    Tiny11

The `vfio-pci` driver does not use the GPU for Linux graphics.

Its purpose is to hold the PCI device in a state where it can be safely mapped
into the virtual machine.

## IOMMU Role

PCI devices can perform Direct Memory Access (DMA).

Without an IOMMU, a passthrough device could potentially access host physical
memory outside the VM.

The IOMMU provides DMA isolation.

Conceptually:

    NVIDIA GPU
         |
         v
       IOMMU
         |
         +--> Tiny11 memory
         |
         X--> unrelated Linux host memory

The host enables IOMMU support with:

    intel_iommu=on iommu=pt

The actual IOMMU group layout should always be inspected before configuring PCI
passthrough.

## NVIDIA PCI Functions

The NVIDIA hardware exposes four PCI functions:

    0000:01:00.0  NVIDIA GeForce GTX 1660 Ti
    0000:01:00.1  NVIDIA HD Audio
    0000:01:00.2  NVIDIA USB 3.1 Host Controller
    0000:01:00.3  NVIDIA USB Type-C UCSI Controller

The corresponding vendor/device IDs are:

    10de:2191
    10de:1aeb
    10de:1aec
    10de:1aed

All four functions are reserved for VFIO in Tiny11 mode.

This avoids splitting ownership of related NVIDIA hardware between the Linux
host and Windows guest.

## Binding Devices to vfio-pci

Tiny11 mode adds the NVIDIA device IDs to the kernel command line using:

    vfio-pci.ids=10de:2191,10de:1aeb,10de:1aec,10de:1aed

The VFIO modules are also included in the initramfs so they are available early
during the boot process.

The required modules are:

    vfio
    vfio_iommu_type1
    vfio_pci

Early binding reduces the chance that normal Linux drivers will claim the
NVIDIA devices before VFIO.

## NVIDIA Driver Blacklisting

Tiny11 mode also prevents the native NVIDIA Linux modules from claiming the GPU.

The passthrough blacklist includes:

    nvidia
    nvidia_drm
    nvidia_modeset
    nvidia_uvm
    nvidiafb

This removes competition between the native NVIDIA driver and `vfio-pci`.

In Mint mode, the blacklist is disabled so Linux can use the NVIDIA GPU
normally.

## Expected Tiny11 Mode State

Before starting Tiny11, all four NVIDIA PCI functions should report:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

This can be checked with:

    sudo gpu-mode status

The Tiny11 startup safeguard also validates these bindings automatically before
libvirt allows the VM to start.

## libvirt PCI Assignment

The Tiny11 VM configuration contains PCI `hostdev` entries.

A simplified example is:

    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000'
                 bus='0x01'
                 slot='0x00'
                 function='0x0'/>
      </source>
    </hostdev>

The source address identifies the physical PCI device on the Linux host.

Equivalent entries are used for all four NVIDIA PCI functions.

The complete raw VM XML is intentionally not included in this repository.

Only sanitized examples relevant to passthrough are documented.

## Host vs Guest PCI Addresses

The physical device address on the host does not have to match the PCI address
seen by Windows.

For example:

    Host:
        0000:01:00.0

may appear at a different virtual PCI address inside Tiny11.

libvirt and QEMU construct the guest PCI topology independently of the host
topology.

Windows only needs to see a valid NVIDIA PCI device.

## Windows Driver

Once Tiny11 starts, Windows performs normal PCI device discovery.

The passed-through GPU is detected as the real NVIDIA GeForce GTX 1660 Ti.

The normal NVIDIA Windows driver can then attach to the device.

The resulting path is:

    Windows application
          |
          v
    NVIDIA Windows driver
          |
          v
    passed-through PCI device
          |
          v
    GTX 1660 Ti hardware

This allows Windows applications to use technologies such as:

- DirectX
- Vulkan
- OpenGL
- CUDA
- hardware video acceleration

without relying on an emulated GPU.

## Verification in Tiny11

After installing the NVIDIA Windows driver, the GPU was verified inside Tiny11
using:

    nvidia-smi

The command successfully detected the GTX 1660 Ti.

This confirmed that Windows was communicating with the physical GPU through
PCI passthrough.

## SPICE Is Still Retained

The Tiny11 VM also retains a SPICE virtual graphics device.

This is intentional.

The VM therefore has two graphics paths:

    Tiny11
      |
      +--> SPICE virtual display
      |
      +--> NVIDIA GTX 1660 Ti passthrough

SPICE provides a reliable management and recovery console even if the NVIDIA
driver or passthrough GPU becomes unavailable.

It also provides features such as:

- clipboard integration
- USB redirection
- convenient VM console access

## GPU Ownership After VM Shutdown

When Tiny11 shuts down, the NVIDIA devices remain bound to:

    vfio-pci

They do not automatically return to the Linux NVIDIA driver.

This is intentional.

To return the GPU to Linux Mint, the system is prepared for Mint mode:

    sudo gpu-mode mint
    sudo reboot

After reboot, the NVIDIA GPU is again claimed by the normal Linux drivers.

## Reboot-Based Ownership Model

The design intentionally avoids live GPU rebinding.

The complete ownership model is:

    Linux Mint mode
          |
          v
    NVIDIA Linux driver
          |
          v
    sudo gpu-mode tiny11
          |
          v
       reboot
          |
          v
       vfio-pci
          |
          v
    Tiny11 passthrough
          |
          v
    Tiny11 shutdown
          |
          v
       vfio-pci
          |
          v
    sudo gpu-mode mint
          |
          v
       reboot
          |
          v
    NVIDIA Linux driver

This approach proved more reliable for this hardware than dynamically moving
the GPU between Linux and VFIO while the system was running.

## Security and Reliability Principle

The passthrough design follows a single-owner model.

At any given boot, the NVIDIA GPU is intended to belong to exactly one side:

    Linux Mint

or:

    Tiny11 / VFIO

The GPU mode script configures that ownership before reboot.

The libvirt startup safeguard independently verifies the actual runtime driver
state before Tiny11 is allowed to start.

This provides both predictable device ownership and an additional safety check.
