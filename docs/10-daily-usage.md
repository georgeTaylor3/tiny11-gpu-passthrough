# Daily Usage

## Purpose

This document provides the normal operating procedure for switching between
Linux Mint GPU use and Tiny11 GPU passthrough mode.

The system uses a reboot-based ownership model.

The NVIDIA GPU belongs to either:

    Linux Mint

or:

    Tiny11 through vfio-pci

during a given boot.

The GPU is not dynamically moved between the host and VM while the system is
running.

## Check Current State

At any time, inspect the GPU configuration with:

    sudo gpu-mode status

This reports:

- configured next-boot mode
- current active mode
- driver ownership for all NVIDIA PCI functions
- GPU power state
- PRIME mode
- Tiny11 VM state

## Normal Linux Mint Mode

In normal Mint mode, the NVIDIA hardware should use native Linux drivers.

Typical ownership is:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

The NVIDIA GPU is available to Linux Mint.

The host can verify NVIDIA driver access with:

    nvidia-smi

## Switching from Mint to Tiny11 Mode

Before switching, make sure Tiny11 is fully shut down:

    virsh domstate tiny11

Expected result:

    shut off

Prepare Tiny11 passthrough mode:

    sudo gpu-mode tiny11

The command configures the next boot.

It does not immediately move the GPU to VFIO.

Reboot:

    sudo reboot

## Verify Tiny11 Mode After Reboot

After the system returns, run:

    sudo gpu-mode status

The current active mode should report:

    tiny11

The NVIDIA PCI functions should report:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

The GPU power state should normally report:

    D0

The VFIO idle-D3 setting should remain enabled:

    cat /sys/module/vfio_pci/parameters/disable_idle_d3

Expected result:

    Y

## Start Tiny11

Start the VM with:

    virsh start tiny11

The libvirt startup safeguard automatically checks the actual driver ownership
before allowing the VM to start.

If all four NVIDIA PCI functions are bound to `vfio-pci`, Tiny11 starts
normally.

If the host is in the wrong mode, startup is blocked.

## Open the Tiny11 Console

Open Tiny11 using virt-manager or virt-viewer.

SPICE remains available as the VM management console even though the physical
NVIDIA GPU is also passed through.

The SPICE console provides:

- display access
- clipboard integration
- USB redirection
- fallback management access

## Verify the NVIDIA GPU in Windows

Inside Tiny11, the NVIDIA GPU can be checked with:

    nvidia-smi

Windows should detect:

    NVIDIA GeForce GTX 1660 Ti

The normal NVIDIA Windows driver should be active.

## USB Devices

Normal USB devices remain owned by Linux Mint by default.

To give a USB device temporarily to Tiny11:

    virt-viewer
      |
      v
    File
      |
      v
    USB device selection

Select only the device that Windows needs.

For example:

    YubiKey          -> Linux Mint
    USB flash drive  -> Tiny11

This allows the security key to remain available to Linux while storage is
temporarily redirected to Windows.

## Returning a USB Device to Mint

When finished using the USB device in Tiny11:

    virt-viewer
      |
      v
    File
      |
      v
    USB device selection

Deselect the device.

The device should then return to Linux Mint.

For USB storage, finish file operations in Windows before returning the device
to Linux.

## Shutting Down Tiny11

When finished with Windows, shut Tiny11 down normally from inside Windows.

Then verify from Mint:

    virsh domstate tiny11

Expected result:

    shut off

Do not switch GPU modes while Tiny11 is still running.

## Important: GPU Ownership After VM Shutdown

Shutting down Tiny11 does not automatically return the NVIDIA GPU to Linux.

After Tiny11 stops, the NVIDIA devices remain bound to:

    vfio-pci

until the system is configured for Mint mode and rebooted.

This is intentional.

## Returning to Mint Mode

After confirming Tiny11 is shut off:

    sudo gpu-mode mint

Then reboot:

    sudo reboot

## Verify Mint Mode

After reboot:

    sudo gpu-mode status

The current active mode should report:

    mint

Typical driver ownership should be:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

Verify the NVIDIA Linux driver with:

    nvidia-smi

## Complete Mint-to-Tiny11 Workflow

The normal transition is:

    Linux Mint mode
          |
          v
    verify Tiny11 is shut off
          |
          v
    sudo gpu-mode tiny11
          |
          v
       reboot
          |
          v
    sudo gpu-mode status
          |
          v
    verify all NVIDIA devices use vfio-pci
          |
          v
    virsh start tiny11
          |
          v
       use Windows

## Complete Tiny11-to-Mint Workflow

The normal return path is:

    use Tiny11
          |
          v
    shut down Windows
          |
          v
    virsh domstate tiny11
          |
          v
    verify "shut off"
          |
          v
    sudo gpu-mode mint
          |
          v
       reboot
          |
          v
    sudo gpu-mode status
          |
          v
    verify native Linux drivers
          |
          v
       use Mint

## If Tiny11 Startup Is Blocked

If starting Tiny11 produces an error from the GPU safety hook, do not try to
manually force the VM to start.

Check:

    sudo gpu-mode status

If the system is currently in Mint mode, prepare Tiny11 mode:

    sudo gpu-mode tiny11
    sudo reboot

Then verify the driver state again.

## If a Mode Change Is Pending

A status result such as:

    Configured next-boot mode: tiny11
    Current active mode:        mint

means the configuration has changed but the system has not rebooted yet.

Complete the transition with:

    sudo reboot

## Recommended Pre-Start Check

Before starting Tiny11, the preferred quick check is:

    sudo gpu-mode status

Confirm:

    Current active mode: tiny11

and:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

The libvirt hook provides an additional automatic safeguard if this check is
missed.

## Operational Principle

The daily workflow follows one rule:

    configure
        |
        v
      reboot
        |
        v
      verify
        |
        v
       use

For Tiny11:

    configure Tiny11 mode
        |
        v
      reboot
        |
        v
    verify VFIO
        |
        v
     start VM

For Mint:

    shut down VM
        |
        v
    configure Mint mode
        |
        v
      reboot
        |
        v
    verify native drivers
