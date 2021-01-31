# QEMU/KVM Windows virtualization for gaming, fun and profit

## UNDER CONSTRUCTION

!!! warning "Attention"
    At the moment, this article is just a trash dump of my notes during the setup.
    Please, do not refer to it for now.

!!! warning "Disclaimer"
    Many game editors' TOS forbid to run their game in a VM. I haven't faced the problem personally, but anti-cheat systems may flag that your game is running in a VM. This may result in a permanent ban of your account. I decline any responsibility if it happens to you. Run games in a VM at your own risk.

---

## History

| Action | Date |
| ------ | ---- |
|Post date|2020, June 14th|
|Last update|2020, June 14th|

---

## Content

- Plan (where are we going, what do we expect from this)
- Prerequisites (hardware, distro, kernel, etc.)
- Setup
    - Packages installation
    - Mouse/kbd hooks, controler passthrough
    - Sound passthrough
    - GPU passthrough
    - CPU optimization
    - RAM optimization
- Sources (links, credit)

---

## Plan

### Why virtualizing Windows for gaming?

For fun! But also to be able to have an every-day computer running on Linux, without the pain to have to switch to Windows for an incompatible game.

The crucial part here will be to share hardware core components with the least possible logical intermediary. We can achieve that with **GPU passthrough** and **CPU pinning** (or even CPU reservation per core, that we will not cover here). RAM can also be entirely reserved, and optimize with **hugepages**.

Other components such as mouse, keyboard and sound will be shared with the VM indirectly (not pure passthrough). The VM will get direct access to all devices' sockets on your host, through virtio drivers. However, if the devices (like gaming keyboard and mouse) offer some features which depend on drivers not available on Linux, the only way to use these features will be to directly passthrough it. Meaning that it will not be available on the host anymore. This leads to either bad experience of plugging/unplugging devices to switch control, double everything, or invest into a USB switch.

---

### Why not running a Windows machine and virtualize Linux?

Because I want some GPU efficiency on the Linux host as well. GPU passthrough on Hyper-V depends on some BIOS parameters, which consists in authorizing the OS to take full control on the PCIe slot hosting the said GPU. My BIOS does not implement this feature, and KVM does not depend on this.

Also, my GPU threw an error 43 on the first try. It would probably have done the same on Hyper-V. After some reading on the internet, I think I would not have been able to correct this error, as KVM has a feature that hide to the OS that it is virtualized.

Also, I want my backups running with rsync and directly connected to the disks, not through a Microsoft passthrough that I have no idea how it works.

And because I love Linux too much ❤

---

### Where are we going?

The goal here will be to have some decent gaming experience, with Windows-only games running within Linux, using QEMU/KVM, DDA and some config tweaking!

When everything will be technically ready, we will run into some benchmarks and FPS tests on different games, showing performance differences between:

- Normal Windows 10 setup (not virtualized)
- Our QEMU/KVM Windows 10

---

## Prerequisites

Prerequisites (hardware, distro, kernel, etc.)

This list is not exhaustive! This is what I had when I did this, and I'm far from being able to test this setup on so much different hardware 😊 

I was able to test this setup using two different configs. First was involving an Intel CPU integrated graphic chipset and an Nvidia GTX 970. Second was involving the GTX 970 and another Nvidia 1050 Ti. I will show the config for both scenario, and the same logic can be applied for AMD hardware (I have not tested this, just refer to the sources at the bottom of this article).

---

### Software

| Component | Version |
| --------- | ------- |
|Disto|Linux Mint 19.3|
|Kernel|5.0.0-32|
|QEMU version|2.11.1|

---

### Hardware

- CPU: Intel i5 4690K (4 core 4 threads @3.9GHz) (support for both Intel VT-x and VT-d)
- Motherboard: MSI Z97 Gaming 5 (UEFI) (support for both Intel VT-x and VT-d)
- RAM: 16 GB DDR3

#### GPU Scenario 1

- GPU1: Intel HD Graphics
- GPU2: Nvidia GTX 970

#### GPU Scenario 2

- GPU1: Nvidia GTX970
- GPU2: Nvidia GTX 1050 Ti

---

## Setup

From here, I consider that you already have a Linux Mint 19.3 installed in EFI mode.

- Update your distro and upgrade if needed

```console
root@mint# apt-get update
root@mint# apt-get dist-upgrade -y
```

- Reboot, go into your BIOS and ensure that VT-x and VT-d are enabled (AMD-V and AMD-Vi for AMD CPU)
- Back to your host, check that your kernel was able to load virtualization modules and if your CPU is ready to act as an hypervisor

```console
root@mint# grep -e 'svm\|vmx' /proc/cpuinfo
```

- Install packages

```console
root@mint# apt-get install qemu-kvm qemu-utils libvirt-bin bridge-utils virt-manager ovmf \
                                               seabios hugepages cpu-checker dnsmasq ebtables gir1.2-spiceclientgtk-3.0
```

- Add your user to the libvirtd group

```console
root@mint# usermod -aG your_user libvirtd
```

- Test if your system is ready for virtualization

```console
root@mint# virsh -c qemu:///system list
```

Good, we are ready for next step!

---

### Mouse/kbd hooks, controler passthrough

```console
root@mint# usermod -aG input ndfeb
root@mint# usermod -aG kvm ndfeb
root@mint# usermod -aG libvirt ndfeb
root@mint# usermod -aG libvirt-qemu ndfeb
```

!!! note "`/etc/libvirt/qemu/win10.xml`"
    ```conf
    (...)
        <input type='mouse' bus='virtio'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x0e' function='0x0'/>
        </input>
        <input type='keyboard' bus='virtio'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x0f' function='0x0'/>
        </input>
        <input type='mouse' bus='ps2'/>
        <input type='keyboard' bus='ps2'/>
    (...)
    <qemu:commandline>
        <qemu:arg value='-cpu'/>
        <qemu:arg value='host,hv_time,kvm=off,hv_vendor_id=null'/>
        <qemu:arg value='-object'/>
        <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/usb-Logitech_Gaming_Mouse_G502_0F7538723832-event-if01'/>
        <qemu:arg value='-object'/>
        <qemu:arg value='input-linux,id=mouse2,evdev=/dev/input/by-id/usb-Logitech_Gaming_Mouse_G502_0F7538723832-event-mouse'/>
        <qemu:arg value='-object'/>
        <qemu:arg value='input-linux,id=mouse3,evdev=/dev/input/by-id/usb-Logitech_Gaming_Mouse_G502_0F7538723832-if01-event-kbd'/>
        <qemu:arg value='-object'/>
        <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/usb-Logitech_G413_Carbon_Mechanical_Gaming_Keyboard_1561395B3434-event-if01,grab_all=on,repeat=on'/>
        <qemu:arg value='-object'/>
        <qemu:arg value='input-linux,id=kbd2,evdev=/dev/input/by-id/usb-Logitech_G413_Carbon_Mechanical_Gaming_Keyboard_1561395B3434-event-kbd,grab_all=on,repeat=on'/>
        <qemu:arg value='-object'/>
        <qemu:arg value='input-linux,id=kbd3,evdev=/dev/input/by-id/usb-Logitech_G413_Carbon_Mechanical_Gaming_Keyboard_1561395B3434-if01-event-kbd,grab_all=on,repeat=on'/>
    </qemu:commandline>
    (...)
    ```

---

!!! note "`/etc/libvirt/qemu.conf`"
    ```conf
    user = "ndfeb"
    group = "kvm"
    (...)
    cgroup_device_acl = [
        "/dev/kvm",
        "/dev/null", "/dev/full", "/dev/zero",
        "/dev/random", "/dev/urandom",
        "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
        "/dev/rtc","/dev/hpet",
        "/dev/input/by-id/usb-Logitech_G413_Carbon_Mechanical_Gaming_Keyboard_1561395B3434-event-if01",
        "/dev/input/by-id/usb-Logitech_G413_Carbon_Mechanical_Gaming_Keyboard_1561395B3434-event-kbd",
        "/dev/input/by-id/usb-Logitech_G413_Carbon_Mechanical_Gaming_Keyboard_1561395B3434-if01-event-kbd",
        "/dev/input/by-id/usb-Logitech_Gaming_Mouse_G502_0F7538723832-event-if01",
        "/dev/input/by-id/usb-Logitech_Gaming_Mouse_G502_0F7538723832-event-mouse",
        "/dev/input/by-id/usb-Logitech_Gaming_Mouse_G502_0F7538723832-if01-event-kbd",
        "/dev/input/by-id/usb-Logitech_Gaming_Mouse_G502_0F7538723832-mouse"
    ]
    (...)
    ```

With this conf, keyboard and mouse can be switched from host to guest with ++lctrl+rctrl++

---

### Permission denied errors

If you get permission deny errors concerning audio sockets when you try to start your VM, you probably face a problem with SElinux or AppArmor. For AppArmor, add `/dev/input/* rw` to the file `/etc/apparmor.d/abstractions/libvirt-qemu` and restart apparmor:

```console
root@mint# cat /etc/apparmor.d/abstractions/libvirt-qemu
(...)
  /dev/input/* rw,

  /dev/net/tun rw,
  /dev/kvm rw,
  /dev/ptmx rw,
  /dev/kqemu rw,
(...)
root@mint# systemctl restart apparmor
```

---

From [passthroughpo.st](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/):

!!! quote "From [passthroughpo.st](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/)"
    You're probably running into an Apparmor (if you're on a Debian/Ubuntu based distro) or SELinux (if you're on a CentOS/Fedora based distro) problem. In case of AppArmor, /etc/apparmor.d/abstractions/libvirt-qemu contains the file to add additional permissions such as /dev/input and /dev/shm (needed for LookingGlass) which is parsed on VM startup. You'll want to add the line /dev/input/* rw, and then restart AppArmor (or reboot).

    In case of SELinux, you can refer to [this document](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-fixing_problems-allowing_access_audit2allow) on how to whitelist your input devices.

---

### Sound passthrough

!!! note "`01_old_qemu_conf_sound.txt` (`/mnt/etc/libvirt/qemu/win10.xml`)"
```xml
  <qemu:commandline>
    <qemu:arg value='-cpu'/>
    <qemu:arg value='host,hv_time,kvm=off,hv_vendor_id=null'/>
    <qemu:env name='QEMU_AUDIO_DRV' value='pa'/>
    <qemu:env name='QEMU_PA_SAMPLES' value='8192'/>
    <qemu:env name='QEMU_AUDIO_TIMER_PERIOD' value='99'/>
    <qemu:env name='QEMU_PA_SERVER' value='/run/user/1000/pulse/native'/>
  </qemu:commandline>
```

---


### GPU passthrough

to be continued...

---

### CPU optimization

---


### RAM optimization

---

## Sources (links, credit)

- [GPU passthrough](https://davidyat.es/2016/09/08/gpu-passthrough/)
- [mathiashueber.com](https://mathiashueber.com/)
  - [Windows VM with GPU passthrough (AMD)](https://mathiashueber.com/windows-virtual-machine-gpu-passthrough-ubuntu/)
  - [Fighting Nvidia GPU error 43](https://mathiashueber.com/fighting-error-43-nvidia-gpu-virtual-machine/)
  - [VM audio with PulseAudio](https://mathiashueber.com/virtual-machine-audio-setup-get-pulse-audio-working/)
- [About cifs-utils and SMB3](https://superuser.com/questions/1226973/how-to-force-linux-cifs-mount-to-default-to-smb3)
- [wiki.archlinux.org](https://wiki.archlinux.org/)
  - [PCI passthrough via OVMF](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)
  - [Passing VM audio to host via PulseAudio](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Passing_VM_audio_to_host_via_PulseAudio)
  - [Examples](https://wiki.archlinux.org/index.php/PulseAudio/Examples)
- [Configure samba to use SMB2 and/or SMB3](https://www.cyberciti.biz/faq/how-to-configure-samba-to-use-smbv2-and-disable-smbv1-on-linux-or-unix/)
- [Get rid of PulseAudio audio cracking](https://www.reddit.com/r/VFIO/comments/542bw1/ha_got_rid_of_the_pulse_audio_crackling/)
- [[PDF] About KVM](http://www.linux-kvm.org/images/2/24/01x02-Rik_van_Riel-KVM_realtime.pdf)

---

## Bookmarks used for setup:

- https://gitlab.com/NdFeB/cool-bash-aliases-and-bashrc
- https://stackoverflow.com/questions/1878974/redefine-tab-as-4-spaces
- https://davidyat.es/2016/09/08/gpu-passthrough/
- https://mathiashueber.com/windows-virtual-machine-gpu-passthrough-ubuntu/
- https://mathiashueber.com/virtual-machine-audio-setup-get-pulse-audio-working/
- https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Passing_VM_audio_to_host_via_PulseAudio
- https://www.reddit.com/r/VFIO/comments/542bw1/ha_got_rid_of_the_pulse_audio_crackling/
- https://mathiashueber.com/fighting-error-43-nvidia-gpu-virtual-machine/
- https://stackoverflow.com/questions/17327120/how-can-i-comment-a-single-line-in-xml
- http://www.linux-kvm.org/images/2/24/01x02-Rik_van_Riel-KVM_realtime.pdf
- https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF
- https://wiki.archlinux.org/index.php/PulseAudio/Examples

