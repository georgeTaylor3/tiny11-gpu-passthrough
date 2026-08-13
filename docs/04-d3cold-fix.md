# NVIDIA D3cold Passthrough Fix

## Problem

Initial GPU passthrough attempts failed with an error similar to:

    internal error: Unknown PCI header type '127' for device '0000:01:00.0'

The NVIDIA GPU had entered the PCI `D3cold` power state.

Its PCI configuration space became unreadable, and inspection with `lspci`
returned `ff` values.

## Troubleshooting Performed

Several approaches were tested before finding the working solution.

### Attempt 1: Force the device power policy to "on"

The PCI power control setting was changed from its default behavior to:

    on

using the device's:

    power/control

interface.

This did not wake the GPU.

### Attempt 2: Disable D3cold for the device

The device's:

    d3cold_allowed

setting was changed in an attempt to prevent the GPU from remaining in
`D3cold`.

This also did not successfully restore access to the GPU.

### Attempt 3: PCI Device Reset

A PCI reset was attempted.

The reset operation returned an I/O error.

The device reported the following available reset methods:

    flr bus

Neither reset method resolved the inaccessible PCI configuration space.

## Working Solution

The successful fix was to configure `vfio-pci` to prevent the device from
entering the problematic idle D3 state.

The following option was added to:

    /etc/modprobe.d/vfio-pci.conf

Configuration:

    options vfio-pci disable_idle_d3=1

The initramfs was then rebuilt.

Afterward, the system was fully powered off and cold-booted rather than simply
restarted.

## Verification

After the change, the following kernel parameter:

    /sys/module/vfio_pci/parameters/disable_idle_d3

reported:

    Y

The NVIDIA GPU also remained in the PCI power state:

    D0

while bound to `vfio-pci`.

The PCI configuration space was once again readable, and Tiny11 GPU
passthrough started successfully.

## Why This Setting Remains Enabled

The `disable_idle_d3=1` setting remains permanently configured.

It matters when the NVIDIA devices are bound to `vfio-pci`, which occurs in
Tiny11 passthrough mode.

In normal Linux Mint mode, the NVIDIA GPU is instead controlled by the native
Linux NVIDIA driver, so the VFIO idle-D3 behavior is not involved.

## Result

Before the fix:

    GPU -> D3cold
    PCI configuration space -> unreadable
    libvirt startup -> failure

After the fix:

    GPU -> D0
    PCI configuration space -> readable
    vfio-pci -> owns NVIDIA devices
    Tiny11 GPU passthrough -> successful
