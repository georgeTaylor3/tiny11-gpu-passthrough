# NVIDIA USB Controller VFIO Binding

## Purpose

This document describes a passthrough issue involving the NVIDIA USB controller
at:

    0000:01:00.2

The GPU, audio, USB, and UCSI functions are all passed to Tiny11 together.

However, the NVIDIA USB controller did not always bind to `vfio-pci`
automatically during boot.

Instead, Linux sometimes claimed it with:

    xhci_hcd

A helper script and systemd service were added to make the binding deterministic.

## NVIDIA PCI Functions

The NVIDIA hardware exposes the following related functions:

    0000:01:00.0  NVIDIA GeForce GTX 1660 Ti
    0000:01:00.1  NVIDIA HD Audio
    0000:01:00.2  NVIDIA USB 3.1 Host Controller
    0000:01:00.3  NVIDIA USB Type-C UCSI Controller

Tiny11 passthrough mode expects all four devices to be controlled by:

    vfio-pci

The desired runtime state is:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

## Problem

The GPU, audio, and UCSI functions successfully bound to `vfio-pci`.

The USB controller at:

    0000:01:00.2

could instead bind to:

    xhci_hcd

This produced an incomplete passthrough state.

Conceptually:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> xhci_hcd
    01:00.3 -> vfio-pci

That was not the desired configuration.

The Tiny11 VM is intended to receive the complete set of NVIDIA PCI functions.

## Manual Binding Test

Before automating the fix, the USB controller was manually rebound.

The intended driver override was set to:

    vfio-pci

The device was then detached from:

    xhci_hcd

and attached to:

    vfio-pci

The equivalent sysfs operations were:

    echo vfio-pci > /sys/bus/pci/devices/0000:01:00.2/driver_override

    echo 0000:01:00.2 > /sys/bus/pci/drivers/xhci_hcd/unbind

    echo 0000:01:00.2 > /sys/bus/pci/drivers/vfio-pci/bind

After this manual procedure, the controller successfully reported:

    01:00.2 -> vfio-pci

This proved that the device itself could be passed through successfully.

## Automation

A helper script was created at:

    /usr/local/sbin/bind-nvidia-usb-vfio

The repository version is stored at:

    scripts/bind-nvidia-usb-vfio

The script performs the following logic:

    locate 0000:01:00.2
            |
            v
    load vfio-pci
            |
            v
    set driver_override = vfio-pci
            |
            v
    inspect current driver
            |
       +----+----+
       |         |
    vfio-pci   other driver
       |         |
       |         v
       |      unbind
       |         |
       |         v
       |      bind to
       |      vfio-pci
       |         |
       +----+----+
            |
            v
    verify vfio-pci

## Script Behavior

The helper uses the PCI address:

    0000:01:00.2

It first verifies that the device exists.

If the device is missing, the script exits safely.

The script then loads:

    vfio-pci

using `modprobe`.

Next, it writes:

    vfio-pci

to the device's:

    driver_override

attribute.

This tells the kernel that `vfio-pci` is the preferred driver for this
specific PCI device.

## Existing Driver Detection

The script checks the current driver symlink under:

    /sys/bus/pci/devices/0000:01:00.2/driver

If the device is already owned by:

    vfio-pci

the script exits successfully without making unnecessary changes.

If another driver is attached, the script determines its name and unbinds the
device from that driver.

For this system, the most common competing driver was:

    xhci_hcd

## Binding to vfio-pci

After the previous driver is removed, the script binds:

    0000:01:00.2

to:

    /sys/bus/pci/drivers/vfio-pci/

The script then verifies the resulting driver ownership.

The final required state is:

    01:00.2 -> vfio-pci

If verification fails, the script exits with an error.

## systemd Service

The helper is executed automatically by:

    nvidia-usb-vfio.service

The service is stored on the host at:

    /etc/systemd/system/nvidia-usb-vfio.service

The repository copy is stored at:

    systemd/nvidia-usb-vfio.service

The service runs the helper using:

    ExecStart=/usr/local/sbin/bind-nvidia-usb-vfio

It is configured as a one-shot service because the binding operation only needs
to occur once during system startup.

## Service Ordering

The service runs after kernel module loading and before libvirt VM startup.

Conceptually:

    system boot
        |
        v
    kernel modules
        |
        v
    NVIDIA USB VFIO binding
        |
        v
    libvirt
        |
        v
    Tiny11 may start

This reduces the chance that Tiny11 is launched before the NVIDIA USB
controller is correctly prepared.

## Mode Integration

The service is enabled only when the machine is configured for Tiny11
passthrough mode.

When running:

    sudo gpu-mode tiny11

the service is enabled for the next boot.

When running:

    sudo gpu-mode mint

the service is disabled.

This prevents the helper from interfering with normal Linux ownership of the
USB controller in Mint mode.

## Mint Mode Behavior

In normal Linux Mint mode, the USB controller is expected to use:

    xhci_hcd

This is correct.

The service is disabled, and the VFIO udev rule is also disabled.

Typical Mint-mode ownership is:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

## Tiny11 Mode Behavior

In Tiny11 mode, the desired ownership is:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

The helper and service specifically reinforce the binding of:

    01:00.2

because it was the function most likely to be claimed by a normal Linux driver.

## Verification

After booting into Tiny11 mode, the state can be checked with:

    sudo gpu-mode status

The expected result is:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

The Tiny11 startup safeguard independently checks the same driver state before
allowing the VM to start.

## Reliability Principle

The important lesson from this issue is that kernel command-line VFIO device
IDs alone do not always guarantee that every related PCI function will bind to
the desired driver.

This configuration therefore uses several layers:

    vfio-pci.ids kernel parameter
            |
            v
    udev driver override
            |
            v
    dedicated binding helper
            |
            v
    systemd startup service
            |
            v
    libvirt startup safeguard

Together these controls make the final device ownership predictable and easier
to verify.
