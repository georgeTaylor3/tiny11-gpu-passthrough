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

```text
01:00.0  NVIDIA GeForce GTX 1660 Ti
01:00.1  NVIDIA HD Audio
01:00.2  NVIDIA USB 3.1 Host Controller
01:00.3  NVIDIA USB Type-C UCSI Controller
