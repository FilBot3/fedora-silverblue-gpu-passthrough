# GPU PCI-Passthrough on Silverblue

## Overview

When you have hardware burning a hole in your desk, and you want to run a VM that can utilize physical hardware such as a Graphics Processing Unit, or GPU, this is how you set it up.

### References

Docs and Blogs on how-to

- [ArchLinux Wiki: PCI Passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [GitHub: BigOakley/gpu-passthrough](https://github.com/BigOakley/gpu-passthrough/blob/main/outline.md)
- [mrzk.io Blog: Fedora 32 and GPU Passthrough VFIO](https://mrzk.io/posts/fedora-32-and-gpu-passthrough-vfio/)
- [N00b security Blog: Easy GPU Passthrough using KVM on Fedora](https://n00bsecurityblog.wordpress.com/2019/11/03/easy-gpu-passthrough-using-kvm-on-fedora/)

Fedora Discussions

- [Fedora Discussions: GPU Pass-through on Silverblue](https://discussion.fedoraproject.org/t/gpu-pci-passthrough-with-silverblue-kinoite/45271)

## Hardware

You will need to have a _second monitor_ with this setup. You could try to setup _Looking Glass_ or _Sunshine+Moonlight_ if you wanted to.

### Primary GPU for Host

- [MSI Radeon RX 6700 XT MECH 2X 12G OC](https://us.msi.com/Graphics-Card/Radeon-RX-6700-XT-MECH-2X-12G-OC)

### Secondary GPU for VM

* [Zotac GTX 970 Dual Fan](https://www.zotac.com/us/product/graphics_card/gtx-970#spec) - 145w

## Preliminary Checks

> [!Info]
> Make sure you have all updates installed before doing any of this. Makes it go faster!

Run this grep command to find if your CPU has virtualization enabled.

```bash
sudo grep --color --regexp vmx --regexp svm /proc/cpuinfo
```

If there is no result, make sure to enable `VT-d` for Intel or `AMD-V` for AMD based motherboards. Consult your hardware's instructions on how to do that.

You may need to remove some Nvidia specific options if you had once used an Nvidia GPU.

```bash
sudo rpm-ostree kargs \
  --delete-if-present=rd.driver.blacklist=nouveau \
  --delete-if-present=modprobe.blacklist=nouveau \
  --delete-if-present=nvidia-drm.modeset=1 \
  --reboot
```

## Steps for Intel CPU

### Ensure CPU has IOMMU enabled

Run the following command.

```bash
dmesg | grep -i -e DMAR -e IOMMU
```

If this doesn't show, make sure to set:

```bash
sudo rpm-ostree kargs \
  --append-if-missing="intel_iommu=on" \
  --append-if-missing="iommu=pt" \
  --append-if-missing="rd.driver.pre=vfio_pci" \
  --reboot
```

as a Kernel parameter

## Steps for AMD CPU

### Ensure CPU has IOMMU enabled

Run the following command.

```bash
dmesg | grep -i -e DMAR -e IOMMU
```

This should already be enabled for AMD CPUs. Either way, you'll want to do this.

```bash
sudo rpm-ostree kargs \
  --append-if-missing="amd_iommu=on" \
  --append-if-missing="iommu=pt" \
  --append-if-missing="rd.driver.pre=vfio_pci" \
  --reboot
```

### Check PCI Bus Groups

```bash
#!/usr/bin/env bash

shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

It should spit out stuff that looks like this...

```bash
➜  ~ 
> bash ./02-check-pci-bus-groups.sh
IOMMU Group 0:
	00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 1:
	00:01.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
[...lots of output...]

IOMMU Group 32:
	0d:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 22 [Radeon RX 6700/6700 XT/6750 XT / 6800M] [1002:73df] (rev c1)
IOMMU Group 33:
	0d:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21/23 HDMI/DP Audio Controller [1002:ab28]

IOMMU Group 34:
	0e:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1)
	0e:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
[...more output...]
```

You can also get the device IDs using `lspci`. In this case, I'm looking for my NVIDIA card to pass-through.

```bash
➜  ~ 
> lspci -vnn | grep -i --regexp NVIDIA
0e:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1) (prog-if 00 [VGA controller])

0e:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)

➜  ~ 
> lspci -vnn | grep -i --regexp Radeon
0d:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 22 [Radeon RX 6700/6700 XT/6750 XT / 6800M] [1002:73df] (rev c1) (prog-if 00 [VGA controller])
```

Now we'll setup the Kernel Args for disabling the PCI Buses for the GPU.

```bash
sudo rpm-ostree kargs \
  --append-if-missing="vfio-pci.ids=10de:13c2,10de:0fbb" \
  --reboot
```

Then perform a `dracut` to make sure the _initramfs_ has the Kernel module loaded.

```bash
sudo rpm-ostree initramfs \
  --enable \
  --arg="--add-drivers" \
  --arg="vfio-pci" \
  --reboot
```

You should now see when you perform

```bash
sudo lspci -nnv
```

Should show something similar to...

```bash
➜  ~ 
> sudo lspci -vnn
[sudo] password for filbot:
[...lots of output...]

0e:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: ZOTAC International (MCO) Ltd. Device [19da:1366]
	Flags: bus master, fast devsel, latency 0, IRQ 255, IOMMU group 34
	Memory at f5000000 (32-bit, non-prefetchable) [size=16M]
	Memory at c0000000 (64-bit, prefetchable) [size=256M]
	Memory at d0000000 (64-bit, prefetchable) [size=32M]
	I/O ports at e000 [size=128]
	Expansion ROM at f6000000 [disabled] [size=512K]
	Capabilities: [60] Power Management version 3
	Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
	Capabilities: [78] Express Legacy Endpoint, MSI 00
	Capabilities: [100] Virtual Channel
	Capabilities: [250] Latency Tolerance Reporting
	Capabilities: [258] L1 PM Substates
	Capabilities: [128] Power Budgeting <?>
	Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
	Capabilities: [900] Secondary PCI Express
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau

0e:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
	Subsystem: ZOTAC International (MCO) Ltd. Device [19da:1366]
	Flags: bus master, fast devsel, latency 0, IRQ 131, IOMMU group 34
	Memory at f6080000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: [60] Power Management version 3
	Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
	Capabilities: [78] Express Endpoint, MSI 00
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel

[...lots of output...]
```

> [!Warning] If it doesn't say `vfio-pci`
> Then you're doing something wrong.

You can also check if the _vfio_ Kernel modules made it in the _initramfs_ by using:

```bash
➜  ~ 
> sudo lsinitrd /boot/ostree/fedora-af7516f20c0bee3df3aa77532024d9fb7c207d1b7246d1d4949b047d7ab916d9/initramfs-6.0.15-300.fc37.x86_64.img| grep -i --regexp vfio
Arguments:  --reproducible -v --add 'ostree' --tmpdir '/tmp/dracut' -f --no-hostonly --kver '6.0.15-300.fc37.x86_64' --reproducible -v --add 'ostree' --tmpdir '/tmp/dracut' -f --no-hostonly --add-drivers ' vfio-pci' --kver '6.0.15-300.fc37.x86_64'
drwxr-xr-x   3 root     root            0 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio
drwxr-xr-x   2 root     root            0 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/pci
-rw-r--r--   1 root     root        32140 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/pci/vfio-pci-core.ko.xz
-rw-r--r--   1 root     root         8596 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/pci/vfio-pci.ko.xz
-rw-r--r--   1 root     root        19884 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/vfio_iommu_type1.ko.xz
-rw-r--r--   1 root     root        14880 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/vfio.ko.xz
-rw-r--r--   1 root     root         3552 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/vfio_virqfd.ko.xz
```

The `/boot/ostree/fedora-???/initramfs-x.y.z-aaa.fc??.x86_64.img` depends on what image is live. You may have two or three depending on how many backup images you keep of your _ostree_ setup.

### Install bridge-utils

> [!Info] Optional Stuff
> This is entirely optional. You don't need it.

- [getlabsdone.com - How to connect kvm mv to host network](https://getlabsdone.com/how-to-connect-kvm-vm-to-host-network/)
- [getlabsdone.com - How to create a Bridge on Fedora](https://getlabsdone.com/how-to-create-a-bridge-interface-in-centos-redhat-linux/)

On Fedora, I installed _bridge-utils_.

```bash
sudo rpm-ostree install bridge-utils --reboot
```

Then follow the blogs listed. However, this will make your host present a different IP address and route traffic differently. I think my issue for my Windows VM was the disk size. See setting up the Virtual Machine further on.

## Setup the Virtual Machine

In this we're using `qemu-kvm` with `virt-manager` as a GUI to help us.

As of this writing, I'm using `qemu-kvm` version `7.0.0`. This should be sufficient.

```bash
➜  ~ 
> qemu-kvm --version
QEMU emulator version 7.0.0 (qemu-7.0.0-12.fc37)
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
```

### Download Windows 10 ISO

At the time of this writing I'm using _Windows 10_ and I am downloading it from this link. Verify it before clicking.

- [Microsoft Windows 10 ISO Download](https://www.microsoft.com/en-us/software-download/windows10ISO)

> [!Warning] Verify your ISO Download
> Remember to verify the ISO download you're performing!

### Use Virtual Machine Manager

This is going to use `virt-manager` or Virtual Machine Manager. Either way it's the GUI of the command-line application _virt-install_.

- Open `virt-manager`.
- Create a new Virtual Machine. Before installing, be sure to customize the configuration.
	- Provide at least 8GB of RAM.
	- Provide at least 4 CPUs.
	- Provide at least 500GB of Storage.
		- This is because Windows is stupid and huge.
- Ensure the following settings are set.
	- Overview -> Chipset: Q35
	- Overview -> Firmware: UEFI/OVMF
	- Add Hardware -> PCI Host Device -> NVIDIA GPU
	- Add Hardware -> PCI Host Device -> NVIDIA High Def Audio.
- Enable the SATA CD Drive.
- Change the boot order so that SATA CD Drive is first.
- Set CPUs -> Topology -> Sockets 1, Cores 4, Threads 2.
	- This makes 8 Virtual Processors.
- Apply the changes.
- Begin Installation

Setup Windows as you can do the best. I had to move my monitor, then adjust the position of screens, update a bunch and then I could finally move on.

### Modify the Domain XML

Domain is what `virsh` calls a Virtual Machine. We'll edit the XML for the VM. I use the Neovim Flatpak.

```bash
EDITOR=io.neovim.nvim virsh \
  --connect=qemu:///system edit Windows-Gaming
```

Here, we'll insert the few elements.

```xml
<domain type='kvm'>
  ...
  <features>
    <kvm>
      <hidden state='on' />
    </kvm>
    <hyperv>
      <vendor_id state='on' value='whatever' />
    </hyperv>
  </features>
  ...
</domain>
```

You'll need to have the VM powered off for these changes to take effect.

> [!Warning] Secure Boot UEFI
> I disabled this in the VM's Bios. I had to enable the boot menu, then press F12.

### Install virtio-win-guest-tools

This is from:

- [Fedorapeople.org - virt-win-guest-tools.exe](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win-guest-tools.exe)

This will need to be downloaded on the Windows PC. Then reboot.

### Install and configure NVIDIA drivers

Download the _official_ NVIDIA drivers.

### Install Steam

Download and install SteamSetup.exe from:

- [SteamPowered](https://steampowered.com)
