# GPU Mode Switching

## Purpose

The NVIDIA GPU is shared between two possible owners:

- Linux Mint
- Tiny11 through VFIO

The system does not dynamically move the GPU between Linux and the VM while the
host is running.

Instead, the desired ownership mode is configured first and then activated by
rebooting.

The helper command used to manage this is:

    gpu-mode

The script is installed on the host at:

    /usr/local/sbin/gpu-mode

The repository copy is stored at:

    scripts/gpu-mode

## Available Modes

The script supports three primary commands:

    sudo gpu-mode mint
    sudo gpu-mode tiny11
    sudo gpu-mode status

The first two configure the next boot.

The `status` command reports both the current runtime state and the mode that is
configured for the next boot.

## Mint Mode

Mint mode returns the NVIDIA hardware to its normal Linux drivers.

The command is:

    sudo gpu-mode mint

The script prepares the next boot by removing the VFIO-specific NVIDIA device
IDs from the kernel command line.

The expected kernel command line includes:

    quiet splash intel_iommu=on iommu=pt

but does not include:

    vfio-pci.ids=10de:2191,10de:1aeb,10de:1aec,10de:1aed

The NVIDIA passthrough blacklist is disabled.

The VFIO udev rule is disabled.

The NVIDIA USB VFIO helper service is disabled.

If NVIDIA PRIME is available, the script configures:

    on-demand

mode.

The expected device ownership after reboot is approximately:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

The NVIDIA GPU is then available to Linux Mint.

## Tiny11 Mode

Tiny11 mode reserves the complete NVIDIA device set for VFIO.

The command is:

    sudo gpu-mode tiny11

The script adds the following device IDs to the kernel command line:

    vfio-pci.ids=10de:2191,10de:1aeb,10de:1aec,10de:1aed

The NVIDIA Linux modules are blacklisted for the next boot.

The VFIO udev rules are enabled.

The NVIDIA USB VFIO helper service is enabled.

The expected device ownership after reboot is:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

At this point, Linux Mint does not use the NVIDIA GPU.

The device is reserved for Tiny11.

## Why a Reboot Is Required

The script intentionally does not attempt to detach the NVIDIA GPU from one
driver and immediately attach it to another while the system is running.

The ownership transition is instead completed during boot.

The workflow is:

    sudo gpu-mode tiny11
            |
            v
    configuration updated
            |
            v
         reboot
            |
            v
    vfio-pci claims NVIDIA devices
            |
            v
    Tiny11 may start

Returning to Mint uses the reverse process:

    sudo gpu-mode mint
            |
            v
    configuration updated
            |
            v
         reboot
            |
            v
    native Linux drivers claim NVIDIA devices

## Why Live GPU Rebinding Was Avoided

Live GPU rebinding was considered but intentionally avoided.

This hardware demonstrated several behaviors that made runtime handoff less
attractive:

- D3cold power-state problems
- PCI reset sensitivity
- multiple NVIDIA PCI functions that must remain coordinated
- an NVIDIA USB controller that could bind to `xhci_hcd`

A reboot-based ownership model proved more predictable and reliable.

It also provides a clean single-owner state for each boot.

## VM Safety Check

Before changing modes, the script checks the Tiny11 VM state.

The VM must be:

    shut off

before a GPU ownership mode change is allowed.

This avoids changing the next-boot passthrough configuration while the VM is
still running.

## Configuration Marker

The script records the requested mode in:

    /etc/gpu-mode

The file contains either:

    mint

or:

    tiny11

This represents the mode configured for the next boot.

It is not treated as proof of the current runtime driver state.

## Current Mode vs Next-Boot Mode

The `status` command distinguishes between two concepts:

    Configured next-boot mode

and:

    Current active mode

The current active mode is determined from the actual driver attached to:

    0000:01:00.0

If the GPU is bound to:

    vfio-pci

the active mode is identified as:

    tiny11

If it is bound to:

    nvidia

the active mode is identified as:

    mint

## Pending Reboot Detection

If the current runtime mode and configured next-boot mode differ, the status
output reports that a mode change is pending.

For example:

    Configured next-boot mode: tiny11
    Current active mode:        mint

    NOTICE: A mode change is pending.
            Reboot to activate: tiny11

This makes it clear that changing configuration does not immediately move the
GPU between drivers.

## Runtime Driver Inspection

The status command reports the current driver for each NVIDIA PCI function:

    01:00.0
    01:00.1
    01:00.2
    01:00.3

A healthy Tiny11 mode should report:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

A healthy Mint mode should report approximately:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

## Additional Status Information

The status command also reports useful supporting information, including:

- NVIDIA GPU power state
- NVIDIA PRIME mode
- Tiny11 VM state
- current driver ownership of all four NVIDIA PCI functions

This makes the command useful as both an operational check and a troubleshooting
tool.

## Boot Configuration Changes

After changing mode, the script rebuilds the initramfs and GRUB configuration.

The equivalent maintenance operations include:

    update-initramfs -u -k all

and:

    update-grub

This ensures the next boot receives the intended VFIO and driver configuration.

## Tiny11 Mode Configuration Layers

Tiny11 mode uses several layers together:

    GRUB vfio-pci.ids
            |
            v
    NVIDIA module blacklist
            |
            v
    VFIO udev rules
            |
            v
    NVIDIA USB VFIO service
            |
            v
    reboot
            |
            v
    vfio-pci device ownership

This layered design is used because the NVIDIA hardware contains several PCI
functions with different normal Linux drivers.

## Mint Mode Configuration Layers

Mint mode reverses the passthrough-specific configuration:

    remove vfio-pci.ids
            |
            v
    disable NVIDIA blacklist
            |
            v
    disable VFIO udev rules
            |
            v
    disable NVIDIA USB VFIO service
            |
            v
    PRIME on-demand
            |
            v
    reboot
            |
            v
    native Linux driver ownership

## Startup Safeguard

The GPU mode script controls the intended ownership state for the next boot.

A separate libvirt startup hook independently checks the actual runtime state
before Tiny11 starts.

The hook verifies that all four NVIDIA PCI functions are currently bound to:

    vfio-pci

If they are not, Tiny11 startup is blocked.

This provides defense in depth:

    gpu-mode
       |
       v
    configure intended state
       |
       v
    reboot
       |
       v
    actual kernel driver state
       |
       v
    libvirt safety validation
       |
       v
    Tiny11 startup

## Daily Use

To switch from normal Mint use to Tiny11 GPU passthrough:

    sudo gpu-mode tiny11
    sudo reboot

After reboot:

    sudo gpu-mode status

Verify all four NVIDIA functions are using:

    vfio-pci

Then start Tiny11.

When finished, shut Tiny11 down completely.

Return the GPU to Mint with:

    sudo gpu-mode mint
    sudo reboot

After reboot, verify:

    sudo gpu-mode status

The NVIDIA GPU should once again be owned by the native Linux drivers.

## Design Principle

The central design rule is simple:

    one GPU
    one owner
    one boot

The NVIDIA GPU belongs either to Linux Mint or to VFIO/Tiny11 during a given
boot.

The script provides a repeatable and auditable way to move between those two
states.
