# QEMU VFIO Setup and Trace Documentation

This guide walks you through enabling virtualization and VT-d/IOMMU in BIOS, configuring your kernel and init system, building QEMU from source with tracing support, and running QEMU with a VFIO PCIe device for capture and analysis.

This guide is strictly for educational reasons. 

My discord username: crump3tte

Shameless firmware plug: https://discord.gg/shillette

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Enable Virtualization and IOMMU in BIOS](#enable-virtualization-and-iommu-in-bios)
3. [Identify PCI Device IDs](#identify-pci-device-ids)
4. [Configure GRUB for IOMMU and VFIO](#configure-grub-for-iommu-and-vfio)
5. [Verify IOMMU Activation](#verify-iommu-activation)
6. [Install Dependencies and Clone QEMU](#install-dependencies-and-clone-qemu)
7. [Build and Install QEMU with Tracing](#build-and-install-qemu-with-tracing)
8. [Configure VFIO PCI Binding](#configure-vfio-pci-binding)
9. [Update Initramfs](#update-initramfs)
10. [List IOMMU Groups](#list-iommu-groups)
11. [Create a QCOW2 Disk Image](#create-a-qcow2-disk-image)
12. [Launch a Windows VM](#launch-a-windows-vm)
13. [Launch QEMU with VFIO and Tracing](#launch-qemu-with-vfio-and-tracing)

---

## Prerequisites

* A modern Linux distribution (e.g., Arch Linux, Manjaro).
* Root or sudo access.
* CPU and motherboard support for virtualization (VT-x/AMD-V) and IOMMU (VT-d/AMD IOMMU).
* Familiarity with shell commands, editing system files, and rebuilding initramfs.

---

## Enable Virtualization and IOMMU in BIOS

1. Reboot and enter your BIOS/UEFI setup.
2. Enable **Virtualization** (VT-x for Intel, SVM for AMD).
3. Enable **VT-d** (Intel) or **IOMMU** (AMD).
4. Save and exit.

---

## Identify PCI Device IDs

Before configuring GRUB, determine the vendor and device IDs of your target PCIe hardware. Run:

```bash
lspci -nnk
```

This lists all PCI devices with numeric IDs and kernel drivers. Locate the entry for your device, for example:

```text
05:00.0 VGA compatible controller [0300]: NVIDIA Corporation Device [10de:1b80] (rev a1)
```

Here, `10de:1b80` is the vendor\:device ID to use in subsequent steps (e.g., `vfio-pci.ids=10de:1b80`).

---

## Configure GRUB for IOMMU and VFIO

1. Open GRUB defaults:

   ```bash
   sudo nano /etc/default/grub
   ```

2. Find the `GRUB_CMDLINE_LINUX` line and append:

   * For Intel systems:

     ```text
     intel_iommu=on iommu=pt vfio-pci.ids=XXXX:XXXX loglevel=3
     ```
   * For AMD systems:

     ```text
     amd_iommu=on iommu=pt vfio-pci.ids=XXXX:XXXX loglevel=3
     ```

   Replace `XXXX:XXXX` with your device's vendor\:device ID (e.g., `10de:1b80`).

3. Regenerate GRUB config and reboot your machine:

   ```bash
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```

---

## Verify IOMMU Activation

Check that the kernel has enabled IOMMU:

```bash
sudo dmesg | grep -i IOMMU
```

Look for entries indicating IOMMU groups and DMA remapping.

---

## Install Dependencies and Clone QEMU

1. Install Git and required build tools:

   ```bash
   sudo pacman -Syu git base-devel pkg-config libglib libfdt libpixman
   ```

2. Clone the QEMU repository:

   ```bash
   git clone https://github.com/qemu/qemu.git
   cd qemu
   ```

---

## Build and Install QEMU with Tracing

1. Configure QEMU with tracing and desired modules:

   ```bash
   ./configure \
     --enable-trace-backends=log \
     --target-list=x86_64-softmmu \
     --prefix=/usr \
     --sysconfdir=/etc \
     --localstatedir=/var \
     --libexecdir=/usr/lib/qemu \
     --extra-ldflags="$LDFLAGS" \
     --smbd=/usr/bin/smbd \
     --enable-modules \
     --enable-sdl \
     --enable-vhost-user \
     --enable-slirp \
     --disable-werror \
     --audio-drv-list="pa,alsa,sdl"
   ```

2. Build using all CPU cores:

   ```bash
   make -j$(nproc)
   ```

3. Install QEMU:

   ```bash
   sudo make install
   ```

4. Verify installation:

   ```bash
   qemu-system-x86_64 --version
   qemu-system-x86_64 -d help
   ```

---

## Configure VFIO PCI Binding

1. Create or edit the VFIO modprobe file:

   ```bash
   sudo nano /etc/modprobe.d/vfio.conf
   ```

2. Add the binding option:

   ```text
   options vfio-pci ids=XXXX:XXXX
   ```

   Replace `XXXX:XXXX` with your device's vendor\:device ID.

---

## Update Initramfs

1. Edit mkinitcpio configuration:

   ```bash
   sudo nano /etc/mkinitcpio.conf
   ```

2. Add VFIO modules to the `MODULES=(...)` array:

   ```text
   MODULES=(... vfio_pci vfio vfio_iommu_type1)
   ```

3. Rebuild initramfs for the default kernel and reboot your machine:

   ```bash
   sudo mkinitcpio -p linux
   ```

---

## List IOMMU Groups

Use this script to list all IOMMU groups and their devices:

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "	$(lspci -nns ${d##*/})"
    done;
done;
```

Identify the group containing your target device.

---

## Create a QCOW2 Disk Image

```bash
qemu-img create -f qcow2 <vm-name>.img 30G
```

Replace `<vm-name>` with your desired name.

---

## Launch VM to install Windows

Use your created image and an ISO installer:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -hda <vm-name>.img \
  -cdrom "/path/to/windows.iso" \
  -m 16G \
  -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time \
  -smp $(nproc) \
  -usb -device usb-tablet
```

Once built, shutdown the VM and proceed to next step.

---

## Launch QEMU with VFIO and Tracing

After installing Windows, bind your PCIe device and run QEMU with tracing enabled:

```bash
sudo qemu-system-x86_64 \
  -enable-kvm \
  -usb -device usb-tablet \
  -m 16G \
  -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time \
  -smp $(nproc) \
  -device vfio-pci,host=0000:0X:00.0,x-no-mmap=on \
  -hda /home/<username>/<vm-name>.img \
  -D /home/<username>/mmio.log \
  -d 'trace:vfio*'
```

* `0000:0X:00.0`: Replace with your device's bus address.
* Output trace log to `mmio.log` for analysis.

---

## Cleanup

1. Power off the VM when done collecting registers.
2. Review `mmio.log` to inspect trace events and register accesses.

---
