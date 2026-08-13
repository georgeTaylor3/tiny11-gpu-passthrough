# USB Redirection

## Purpose

USB devices are not permanently assigned to the Tiny11 virtual machine.

Linux Mint owns normal USB devices by default.

When Tiny11 needs access to a USB device, that device can be redirected
temporarily through SPICE using `virt-viewer`.

This provides selective USB passthrough without requiring every USB device to
be dedicated to the Windows VM.

## Default Ownership Model

The normal state is:

    USB device
        |
        v
    Linux Mint

This includes devices such as:

- USB storage
- security keys
- phones
- USB adapters
- other removable peripherals

Tiny11 does not automatically take ownership of these devices.

## Why Permanent USB Assignment Was Avoided

A USB security key was initially configured as a permanent libvirt USB
`hostdev`.

That caused two operational problems:

- Tiny11 could depend on the device being physically present at startup
- the security key could be taken away from Linux Mint automatically

This was not the desired behavior.

The permanent USB `hostdev` entry was therefore removed from the Tiny11 VM.

USB devices are now redirected only when explicitly requested.

## SPICE USB Redirection

The VM retains a SPICE USB redirection device.

The relevant libvirt configuration includes a SPICE redirection device similar
to:

    <redirdev bus='usb' type='spicevmc'>
      ...
    </redirdev>

This allows USB devices attached to the Linux host to be redirected into the
running Windows guest through the SPICE connection.

## Redirecting a USB Device

While Tiny11 is running, open the VM console in `virt-viewer`.

Then use:

    File
      |
      v
    USB device selection

Select the device that should be redirected to Tiny11.

The ownership transition is:

    Linux Mint
        |
        | USB redirect
        v
    Tiny11

Only the selected USB device is redirected.

Other USB devices remain available to Linux Mint.

## Example: USB Storage

A USB storage device can be used for transferring files between the Linux host
and Tiny11.

The normal workflow is:

    connect USB storage device
            |
            v
    Linux Mint detects device
            |
            v
    start Tiny11
            |
            v
    virt-viewer
            |
            v
    File -> USB device selection
            |
            v
    select USB storage device
            |
            v
    Tiny11 detects device

While redirected, Linux Mint temporarily loses access to that USB device.

This is expected because the physical device should have only one active owner.

## Returning a Device to Linux Mint

When Tiny11 no longer needs the device, return to:

    virt-viewer
      |
      v
    File
      |
      v
    USB device selection

Deselect the device.

The ownership transition is then:

    Tiny11
        |
        | disconnect redirect
        v
    Linux Mint

The Linux host should detect the device again.

## YubiKey Behavior

The YubiKey remains owned by Linux Mint by default.

This allows it to continue being used on the host for:

- website authentication
- FIDO2
- WebAuthn
- passkeys
- supported multi-factor authentication

Tiny11 does not automatically claim the YubiKey.

If the YubiKey is ever needed inside Windows, it can be explicitly selected
through SPICE USB redirection.

While redirected to Tiny11, Linux Mint temporarily loses access to it.

When deselected, the YubiKey returns to Linux Mint.

## Why This Is Useful

This design allows different USB devices to be used independently.

For example:

    YubiKey        -> Linux Mint
    USB flash drive -> Tiny11

The YubiKey can remain available to a browser running on the Linux host while a
separate USB storage device is redirected to Windows.

This is more flexible than permanently assigning either device to the VM.

## VM Startup Behavior

Because normal USB devices are not configured as required libvirt `hostdev`
devices, Tiny11 does not depend on those devices being present during startup.

The VM can therefore start with:

    no YubiKey connected
    no USB storage connected
    no redirected USB devices

USB devices can be redirected later after Windows has already booted.

## Difference Between USB Redirection and PCI Passthrough

SPICE USB redirection should not be confused with the NVIDIA USB controller
that is passed through as a PCI device.

The NVIDIA device at:

    0000:01:00.2

is an entire PCI USB controller associated with the NVIDIA hardware.

It is passed to Tiny11 as part of the GPU passthrough configuration.

Normal USB devices connected to Linux-controlled USB ports use SPICE
redirection instead.

Conceptually:

    NVIDIA PCI USB controller
            |
            v
        vfio-pci
            |
            v
          Tiny11

while a normal host USB device uses:

    USB device
        |
        v
    Linux Mint
        |
        v
    SPICE redirect
        |
        v
    Tiny11

These are separate passthrough mechanisms.

## Single-Owner Principle

A physical USB device should normally have one active owner at a time.

When Linux Mint owns the device:

    Mint -> device

When the device is redirected:

    Tiny11 -> device

The device should not be actively accessed by both operating systems at the
same time.

This is especially important for USB storage because simultaneous filesystem
access could cause corruption.

## Operational Workflow

The standard USB workflow is:

    connect USB device
            |
            v
    Mint owns device
            |
            v
    decide whether Tiny11 needs it
            |
       +----+----+
       |         |
      no        yes
       |         |
       |         v
       |     redirect with
       |     virt-viewer
       |         |
       |         v
       |      Tiny11
       |         |
       |         v
       |     finish use
       |         |
       |         v
       |     deselect device
       |         |
       +----<----+
            |
            v
       Mint owns device

## Design Principle

USB devices remain with the host unless the user deliberately assigns them to
the guest.

This provides:

- predictable USB ownership
- no USB dependency during VM startup
- continued host access to security keys
- temporary USB storage passthrough
- easy return of devices to Linux Mint
- reduced risk of accidentally assigning the wrong device to Windows
