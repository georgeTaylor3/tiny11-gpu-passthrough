# Troubleshooting

## Purpose

This document records the major problems encountered while configuring NVIDIA
GPU passthrough to Tiny11 and the fixes that proved successful.

The goal is to provide a quick reference for diagnosing similar failures.

## Tiny11 Started While the Host Was in Mint Mode

### Symptom

Tiny11 was accidentally started while the NVIDIA GPU was still owned by the
normal Linux drivers.

Typical Mint-mode ownership looked like:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

### Cause

The VM could be launched manually without first verifying that the host was in
Tiny11 passthrough mode.

### Fix

A libvirt QEMU startup hook was added at:

    /etc/libvirt/hooks/qemu.d/10-tiny11-gpu-safety

Before Tiny11 starts, the hook verifies that all four NVIDIA PCI functions are
actually bound to:

    vfio-pci

If any device is owned by another driver, VM startup is blocked.

### Verification

Run:

    sudo gpu-mode status

A valid Tiny11 passthrough state should show:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

## Unknown PCI Header Type 127

### Symptom

Tiny11 failed to start with an error similar to:

    internal error: Unknown PCI header type '127' for device '0000:01:00.0'

PCI configuration-space inspection returned `ff` values.

### Cause

The NVIDIA GPU had entered the PCI:

    D3cold

power state.

In this state, VFIO could not reliably access the device's PCI configuration
space.

### Attempts That Did Not Work

The following approaches were tested but did not restore the GPU:

    power/control = on

    d3cold_allowed = 0

A PCI reset was also attempted but returned an I/O error.

The device exposed the reset methods:

    flr bus

### Working Fix

Configure `vfio-pci` with:

    options vfio-pci disable_idle_d3=1

in:

    /etc/modprobe.d/vfio-pci.conf

Then rebuild the initramfs:

    sudo update-initramfs -u -k all

The machine should then be fully powered off and cold-booted.

### Verification

Check:

    cat /sys/module/vfio_pci/parameters/disable_idle_d3

Expected result:

    Y

Check the GPU power state.

The working configuration keeps the GPU in:

    D0

while bound to VFIO.

## NVIDIA USB Controller Binds to xhci_hcd

### Symptom

Most NVIDIA PCI functions successfully bound to VFIO, but:

    0000:01:00.2

was controlled by:

    xhci_hcd

instead of:

    vfio-pci

Example:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> xhci_hcd
    01:00.3 -> vfio-pci

### Cause

The Linux xHCI driver claimed the NVIDIA USB controller before the desired
VFIO ownership was established.

### Manual Test

The device was successfully moved manually by setting:

    driver_override = vfio-pci

unbinding it from:

    xhci_hcd

and binding it to:

    vfio-pci

### Permanent Fix

A helper script was created at:

    /usr/local/sbin/bind-nvidia-usb-vfio

and executed automatically by:

    nvidia-usb-vfio.service

The helper:

- loads `vfio-pci`
- sets the device driver override
- detects the current driver
- unbinds the device if necessary
- binds it to `vfio-pci`
- verifies the final state

### Verification

Run:

    sudo gpu-mode status

Expected Tiny11-mode result:

    01:00.2 -> vfio-pci

## Tiny11 Does Not Start After Changing GPU Mode

### Symptom

The command:

    sudo gpu-mode tiny11

was run successfully, but Tiny11 still could not start immediately.

### Cause

`gpu-mode` configures the next boot.

It does not dynamically move the GPU between drivers while the system is
running.

### Fix

After changing mode:

    sudo gpu-mode tiny11

reboot:

    sudo reboot

Then verify:

    sudo gpu-mode status

The current active mode should report:

    tiny11

and all four NVIDIA PCI functions should report:

    vfio-pci

## gpu-mode Reports a Pending Mode Change

### Symptom

Status shows different values for:

    Configured next-boot mode

and:

    Current active mode

Example:

    Configured next-boot mode: tiny11
    Current active mode:        mint

### Cause

The GPU mode configuration has changed, but the machine has not yet rebooted.

### Fix

Reboot:

    sudo reboot

Then verify again:

    sudo gpu-mode status

## Tiny11 Cannot Start Because a USB Device Is Missing

### Symptom

An earlier Tiny11 configuration could fail to start with an error similar to:

    Did not find USB device 1050:0402

### Cause

A YubiKey had been configured as a required libvirt USB `hostdev`.

This meant the VM expected that physical USB device to be present during
startup.

### Fix

The permanent YubiKey USB `hostdev` entry was removed.

Normal USB devices are now redirected selectively through SPICE.

### Result

Tiny11 can start without requiring the YubiKey or other normal USB devices to
be connected.

## YubiKey Not Accessible in Yubico Authenticator

### Symptom

Yubico Authenticator detected the YubiKey but reported that it could not access
the device.

### Cause

Inside Tiny11, Yubico Authenticator required elevated privileges for the
operation being attempted.

### Fix

Run Yubico Authenticator using:

    Run as administrator

The redirected YubiKey then became accessible.

## Yubico Login for Windows Does Not Detect the Key

### Symptom

Yubico Login for Windows remained on:

    Please insert YubiKey to continue

even though Tiny11 could see the key.

### Cause

The available key exposed only:

    FIDO U2F
    FIDO2

and did not provide the OTP/challenge-response capability required by Yubico
Login for Windows.

### Resolution

The local Windows YubiKey login configuration was abandoned.

The key continues to be used for FIDO2, WebAuthn, passkeys, and website MFA.

No changes were made to the existing FIDO configuration.

## SPICE USB Device Does Not Appear in Windows

### Checks

First verify that the device is visible to Linux Mint.

Then open:

    virt-viewer
      |
      v
    File
      |
      v
    USB device selection

Select the desired device.

If the device was already redirected, deselect it and reconnect it.

### Important

Only one operating system should actively own a physical USB device at a time.

When redirected to Tiny11, Linux Mint temporarily loses access to the device.

## NVIDIA Driver Does Not Work Inside Tiny11

### Checks

Verify that Windows detects the NVIDIA GPU.

Then verify the NVIDIA Windows driver is installed.

Inside Tiny11, run:

    nvidia-smi

A successful result should identify the physical:

    NVIDIA GeForce GTX 1660 Ti

If the GPU is missing entirely, first inspect host-side VFIO ownership with:

    sudo gpu-mode status

## NVIDIA GPU Is Not Available to Linux Mint

### Symptom

After finishing with Tiny11, commands such as:

    nvidia-smi

do not work as expected on the Linux host.

### Cause

The host is still booted in Tiny11 mode, so the GPU remains bound to:

    vfio-pci

even if the VM itself has been shut down.

### Fix

Shut Tiny11 down completely.

Then run:

    sudo gpu-mode mint
    sudo reboot

After reboot:

    sudo gpu-mode status

The GPU should once again report:

    01:00.0 -> nvidia

## Useful Diagnostic Commands

Check overall GPU mode:

    sudo gpu-mode status

Check VM state:

    virsh domstate tiny11

Check PCI devices and drivers:

    lspci -nnk -s 01:00.0
    lspci -nnk -s 01:00.1
    lspci -nnk -s 01:00.2
    lspci -nnk -s 01:00.3

Check the VFIO D3 setting:

    cat /sys/module/vfio_pci/parameters/disable_idle_d3

Check GPU power state:

    cat /sys/bus/pci/devices/0000:01:00.0/power_state

Check the NVIDIA USB VFIO service:

    systemctl status nvidia-usb-vfio.service

Check the libvirt VM:

    virsh dumpxml tiny11

The full raw VM XML should not be published without first removing
host-specific or unnecessary identifiers.

## Troubleshooting Principle

When passthrough fails, verify the system from the host outward:

    requested GPU mode
            |
            v
    current kernel driver ownership
            |
            v
    GPU power state
            |
            v
    VFIO binding
            |
            v
    libvirt configuration
            |
            v
    Windows device detection
            |
            v
    Windows driver

This avoids troubleshooting the guest before confirming that the host has
successfully prepared the physical hardware.
