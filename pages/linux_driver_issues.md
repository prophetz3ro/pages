---
layout: post
title: Linux Drivers failed after upgrade
permalink: linux_drivers_failed
---
# Ubuntu Kernel / GPU Driver Notes

## Interrupted `apt upgrade`

If an upgrade was interrupted:

```bash
sudo dpkg --configure -a
sudo apt --fix-broken install
```

Then:

```bash
sudo update-initramfs -u -k all
sudo update-grub
```

If the new kernel does not work, boot an older one from:

```text
GRUB → Advanced options for Ubuntu
```

Check the running kernel:

```bash
uname -r
```

List installed kernel images:

```bash
ls -1 /boot/vmlinuz-*
```

---

## Initramfs

`initramfs` = a small temporary filesystem loaded into RAM before the real root filesystem `/`.

Boot flow:

```text
UEFI
  ↓
GRUB
  ↓
kernel
  ↓
initramfs
  ↓
root filesystem /
  ↓
systemd
```

It contains drivers and tools required early during boot.

Update it with:

```bash
sudo update-initramfs -u -k all
```

`initramfs` = **initial RAM filesystem**.

It is a small temporary filesystem loaded into RAM during early boot, before the real root filesystem `/` is mounted.

```text
UEFI
  ↓
GRUB
  ↓
kernel
  ↓
initramfs
  ↓
real root filesystem /
  ↓
systemd
```

### What it contains

Typically:

- storage drivers
- filesystem drivers
- NVMe/SATA modules
- LUKS tools
- LVM/device-mapper tools
- firmware
- early boot scripts
- other modules needed to reach `/`

### Why it exists

The kernel may need extra tools/modules before it can access the real system disk.

Example:

```text
encrypted root filesystem
        ↓
initramfs loads cryptsetup
        ↓
disk is unlocked
        ↓
real / can be mounted
```

### Where it is

Usually:

```text
/boot/initrd.img-<kernel-version>
```

Example:

```text
/boot/initrd.img-7.0.0-28-generic
```

Despite the name `initrd`, modern Ubuntu normally uses initramfs.

### How it is built

Ubuntu uses `initramfs-tools`.

Main command:

```bash
sudo update-initramfs -u -k all
```

It collects:

```text
kernel modules
+
firmware
+
boot tools/scripts
+
configuration
        ↓
compressed initramfs image
```

Kernel modules mainly come from:

```text
/lib/modules/<kernel-version>/
```

Each kernel normally has its own initramfs.

### Inspect it

```bash
lsinitramfs /boot/initrd.img-$(uname -r)
```

This shows the contents of the temporary early-boot environment.

### Mental model

```text
kernel
= Linux is already running

initramfs
= temporary "mini userspace" in RAM

real /
= normal Ubuntu userspace
```

So initramfs is not really **pre-Linux** — it is **pre-main-system Linux**.

---

## GRUB

```bash
sudo update-grub
```

Regenerates the GRUB configuration and detects available kernels and initramfs images.

It does **not** install a kernel.

---

# Drivers

A driver allows the kernel to communicate with hardware:

```text
application
   ↓
kernel
   ↓
driver
   ↓
hardware
```

Kernel drivers are often implemented as modules ending in `.ko`.

Example:

```text
nvidia.ko
```

Inspect a module:

```bash
modinfo nvidia
```

Show loaded modules:

```bash
lsmod
```

---

## In-tree vs Out-of-tree

### In-tree

The driver source is part of the Linux kernel source tree.

Examples:

```text
amdgpu
i915
xe
nouveau
```

It is built together with the kernel:

```text
kernel + driver → built together
```

### Out-of-tree

The driver is maintained outside the Linux kernel source tree.

Classic example:

```text
NVIDIA proprietary driver
```

A new kernel may change internal APIs, so an external driver needs a compatible module for that kernel.

---

# Kernel Modules

Each kernel has its own module directory:

```text
/lib/modules/<kernel-version>/
```

Example:

```text
/lib/modules/7.0.0-28-generic/
```

A module built for one kernel generally cannot simply be reused with another kernel.

---

# Linux Driver Commands

```bash
lspci -k
```

Shows hardware, the active driver, and possible modules.

```text
Kernel driver in use
= driver currently controlling the device

Kernel modules
= modules that could support the device
```

```bash
lsmod
```

Shows kernel modules currently loaded.

```bash
modinfo <module>
```

Shows module details such as path, version, dependencies and `vermagic`.

```bash
dpkg -S "$(modinfo -n <module>)"
```

Shows which Ubuntu/Debian package installed the module.

```bash
journalctl -k -b
```

Shows kernel logs from the current boot.

### Example

```bash
lspci -k
lsmod | grep iwlwifi
modinfo iwlwifi
dpkg -S "$(modinfo -n iwlwifi)"
journalctl -k -b | grep -i iwlwifi
```

### Module files

```text
.ko     = kernel module
.ko.zst = Zstandard-compressed kernel module
```

# Kernel Headers

Headers are needed to build external kernel modules locally:

```bash
sudo apt install linux-headers-$(uname -r)
```

Model:

```text
driver source
    +
kernel headers
    ↓
compiled .ko module
```

---

# DKMS

DKMS = **Dynamic Kernel Module Support**.

It can automatically build external kernel modules for newly installed kernels.

```text
driver source
    +
kernel headers
    ↓
DKMS
    ↓
nvidia.ko
```

Check status:

```bash
dkms status
```

Build registered modules:

```bash
sudo dkms autoinstall
```

Typical DKMS module location:

```text
/lib/modules/<kernel>/updates/dkms/nvidia.ko
```

If `dkms status` is empty, it does not automatically mean something is wrong. The system may use a precompiled Ubuntu module instead.

---

# Precompiled NVIDIA Modules

Ubuntu can provide a ready-built NVIDIA module for a specific kernel.

In my case:

```bash
modinfo nvidia
```

returned:

```text
/lib/modules/7.0.0-28-generic/kernel/nvidia-580/nvidia.ko
```

This means the module was already built for:

```text
7.0.0-28-generic
```

rather than compiled locally through DKMS.

Find which package installed it:

```bash
dpkg -S /lib/modules/7.0.0-28-generic/kernel/nvidia-580/nvidia.ko
```

---

# `ubuntu-drivers`

Show detected hardware and recommended drivers:

```bash
ubuntu-drivers devices
```

Install the recommended driver:

```bash
sudo ubuntu-drivers install
```

Conceptually:

```text
detect hardware
      ↓
choose recommended driver
      ↓
APT installs required packages
```

It may install either:

- a precompiled Ubuntu kernel module,
- or a DKMS-based driver.

---

# `ubuntu-drivers` vs DKMS

Main distinction:

```text
ubuntu-drivers
= chooses and installs the appropriate driver stack
```

```text
DKMS
= locally builds registered external kernel modules
```

So:

```text
ubuntu-drivers
      ↓
driver installation
      ├── precompiled module
      └── DKMS module
```

In my case, `ubuntu-drivers install` fixed the problem because Ubuntu installed the correct NVIDIA module for the new kernel.

---

# `apt update` vs `ubuntu-drivers`

```bash
sudo apt update
```

Only refreshes information about available packages.

```text
apt update
= "what packages are available?"
```

Whereas:

```bash
sudo ubuntu-drivers install
```

detects the hardware, selects the appropriate driver, and installs it through APT.

---

# GPU Troubleshooting After a Kernel Update

Check kernel:

```bash
uname -r
```

Check GPU and active driver:

```bash
lspci -k | grep -A4 -E 'VGA|3D|Display'
```

Check NVIDIA:

```bash
nvidia-smi
```

Check module:

```bash
modinfo nvidia
```

Check DKMS:

```bash
dkms status
```

If NVIDIA stops working after a kernel update:

```bash
sudo ubuntu-drivers install
sudo update-initramfs -u
sudo reboot
```

If using DKMS:

```bash
sudo apt install linux-headers-$(uname -r)
sudo dkms autoinstall
```

---

# Mental Model

```text
kernel
  ↓
kernel module / driver
  ↓
hardware
```

```text
in-tree
= driver shipped with the kernel source
```

```text
out-of-tree
= driver maintained separately
```

```text
DKMS
= locally rebuilds an external module for a kernel
```

```text
ubuntu-drivers
= chooses and installs the correct Ubuntu driver
```

```text
initramfs
= early-boot temporary filesystem
```

```text
GRUB
= bootloader that selects which kernel to start
```
