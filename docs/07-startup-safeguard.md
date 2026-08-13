# Tiny11 GPU Startup Safeguard

## Purpose

Tiny11 can be started from multiple libvirt clients, including:

    virt-manager
    virsh

Starting Tiny11 while Linux Mint still owns the NVIDIA GPU is an unsafe and
undesired state.

To prevent accidental startup in the wrong GPU mode, a libvirt QEMU hook is
used as a safety control.

The hook blocks Tiny11 from starting unless all required NVIDIA PCI functions
are currently bound to:

    vfio-pci

## Hook Location

The hook is installed on the host at:

    /etc/libvirt/hooks/qemu.d/10-tiny11-gpu-safety

The repository copy is stored at:

    libvirt/hooks/qemu.d/10-tiny11-gpu-safety

## Why the Safeguard Was Added

The safeguard was added after testing showed that Tiny11 could still be
launched manually while the host was running in normal Mint GPU mode.

That created the possibility of attempting to start the VM while the NVIDIA
hardware was still owned by native Linux drivers.

The goal was to make accidental startup fail safely rather than relying only
on operator memory.

## libvirt Hook Timing

The hook runs during the QEMU/libvirt:

    prepare begin

stage.

This occurs before QEMU is allowed to fully start the VM.

Conceptually:

    request to start Tiny11
            |
            v
    libvirt prepare begin
            |
            v
    GPU safety hook
            |
       +----+----+
       |         |
     PASS       FAIL
       |         |
       v         v
    continue    abort
       |
       v
    QEMU starts
       |
       v
    Tiny11 starts

A non-zero exit code from the hook causes libvirt to abort startup.

## Scope

The hook only applies to the VM named:

    tiny11

Other virtual machines are ignored.

The script also ignores unrelated libvirt hook operations.

The validation is performed only when:

    VM name = tiny11
    operation = prepare
    sub-operation = begin

## Required NVIDIA Devices

The hook validates the four NVIDIA PCI functions used by Tiny11:

    0000:01:00.0
    0000:01:00.1
    0000:01:00.2
    0000:01:00.3

These represent:

    01:00.0  NVIDIA GeForce GTX 1660 Ti
    01:00.1  NVIDIA HD Audio
    01:00.2  NVIDIA USB 3.1 Host Controller
    01:00.3  NVIDIA USB Type-C UCSI Controller

All four devices must be controlled by:

    vfio-pci

before Tiny11 is allowed to start.

## Runtime Validation

The hook checks the actual Linux sysfs driver link for each PCI device.

For example:

    /sys/bus/pci/devices/0000:01:00.0/driver

This symlink identifies the kernel driver currently controlling the device.

Possible examples include:

    vfio-pci
    nvidia
    snd_hda_intel
    xhci_hcd
    nvidia-gpu

The script resolves the driver symlink and compares the resulting driver name
to:

    vfio-pci

If any required device is using another driver, validation fails.

## Device Existence Check

Before checking driver ownership, the hook verifies that each expected PCI
device exists.

For example:

    /sys/bus/pci/devices/0000:01:00.0

If an expected device is missing, the hook treats that as an error.

This helps detect unexpected conditions such as:

- changed PCI addressing
- disabled hardware
- firmware changes
- device enumeration problems

## Driver Binding Check

The script also verifies that each PCI device has a bound kernel driver.

A device with no driver is not accepted as a valid passthrough state.

The expected condition is specifically:

    driver = vfio-pci

This avoids allowing Tiny11 to start when a device is merely unbound.

## Why Actual Driver State Is Used

The GPU mode script stores the intended next-boot configuration in:

    /etc/gpu-mode

That file can contain:

    mint

or:

    tiny11

However, this file only represents intended configuration.

It does not prove that the current running kernel successfully bound every
device to the expected driver.

For example, the configuration marker could say:

    tiny11

while the running system unexpectedly has:

    01:00.2 -> xhci_hcd

Because of this, the safety hook does not rely only on `/etc/gpu-mode`.

It checks the actual runtime driver state instead.

## Expected Passing State

Tiny11 startup is allowed only when the hook sees:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

If all four devices match, the hook exits successfully and libvirt continues
starting the VM.

## Expected Failure State

In normal Mint mode, the hardware typically appears as:

    01:00.0 -> nvidia
    01:00.1 -> snd_hda_intel
    01:00.2 -> xhci_hcd
    01:00.3 -> nvidia-gpu

If Tiny11 is started in this state, the hook reports the mismatched drivers and
returns a failure status.

libvirt then blocks VM startup.

## Failure Message

When validation fails, the hook instructs the operator to prepare Tiny11 mode
with:

    sudo gpu-mode tiny11
    sudo reboot

After reboot, the system should be verified with:

    sudo gpu-mode status

Tiny11 should only be started after all four NVIDIA devices report:

    vfio-pci

## Manual Hook Test

The hook can be tested manually using the same arguments that libvirt supplies:

    sudo /etc/libvirt/hooks/qemu.d/10-tiny11-gpu-safety tiny11 prepare begin

A successful result returns no error and exits with:

    0

The exit status can be checked with:

    echo $?

A failed validation returns:

    1

and prints the reason startup would be blocked.

## Testing in Tiny11 Mode

When the host is correctly configured for Tiny11 passthrough:

    sudo gpu-mode status

should report:

    01:00.0 -> vfio-pci
    01:00.1 -> vfio-pci
    01:00.2 -> vfio-pci
    01:00.3 -> vfio-pci

Starting the VM with:

    virsh start tiny11

should succeed.

## Testing in Mint Mode

After returning to Mint mode:

    sudo gpu-mode mint
    sudo reboot

the NVIDIA hardware should return to the native Linux drivers.

Attempting:

    virsh start tiny11

should then fail safely.

The VM should remain shut off.

## Defense in Depth

The GPU passthrough design uses two separate controls.

The first control is:

    gpu-mode

which configures the intended ownership state for the next boot.

The second control is:

    libvirt startup hook

which verifies the actual runtime state immediately before Tiny11 starts.

The relationship is:

    gpu-mode tiny11
            |
            v
    configure next boot
            |
            v
         reboot
            |
            v
    kernel binds devices
            |
            v
    libvirt startup request
            |
            v
    safety hook verifies
    actual driver ownership
            |
       +----+----+
       |         |
     valid     invalid
       |         |
       v         v
    Tiny11     startup
    starts     blocked

This provides a defense-in-depth approach to GPU ownership.

## Security and Reliability Benefit

The safeguard converts an operator mistake into a controlled failure.

Without the hook:

    accidental Tiny11 start
            |
            v
    passthrough attempted
    in unexpected host state

With the hook:

    accidental Tiny11 start
            |
            v
    runtime state checked
            |
            v
    unsafe state detected
            |
            v
    VM startup blocked

This reduces the risk of device conflicts and makes the passthrough workflow
more predictable.
