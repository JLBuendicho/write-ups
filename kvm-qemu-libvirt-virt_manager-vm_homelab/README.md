# KVM + QEMU + libvirt + virt-manager VM Homelab Setup
Initial set up of my Virtual Machine Homelab to play around with network and server administration concepts.

---

## Verify that machine has virtualization support
I followed [ArchWiki's KVM Documentation](https://wiki.archlinux.org/title/KVM)

---

## Install packages
- quemu-base - Hypervisor
- libvirt - VM management service
- virt-manager - GUI
- dnsmasq - Virtual network DHCP/DNS
- edk2-ovmf - UEFI firmware for VMs

---

## Enable libvirt
Start and enable the daemon via systemctl:
``` bash
sudo systemctl enable --now libvirtd
```

Verify that it is running:
``` bash
systemctl status libvirtd
```

---

## Add my User to the libvirt and kvm groups
Append $USER to the groups via usermod:
``` bash
sudo usermod -aG libvirt, kvm $USER
```
> I rebooted the machine to ensure the new group memberships take effect


Use `groups` to confirm (you should see libvirt and kvm listed) 

---

## Create Windows Server 2022 VM
For the win2k22-dc machine I gave it these specs:
- Chipset: Q35
- Firmware: BIOS
- CPU: 3vCPU host-passthrough
- Memory: 4GB
- Disk: SATA (cache: none, discard: unmap) 
- CDROMs: Windows Server 2002, virtio-win
- NIC: e1000e NAT
- Display: VGA
- Video: VNC

---

## Setup networking and qemu guest agent for win2k22-dc

1. Change device name to DC01 via server manager -> local server
2. Configure static IPv4 address, gateway and DNS (DNS adress would be DC01's adress)
3. Install QEMU guest agent via virtio-win.iso -> virtio-win-guest-tools.exe
4. Shut down VM
5. Add Channel Hardware (name: org.qemu.guest_agent.0, device type: unix socket, target type: virtio)
6. Boot VM

---

## Install ADDS on win2k22-dc

1. Server Manager -> Add Roles and Features
2. Install ADDS, DHCP Server, File Services, and Print and Document Services

---

## Promote win2k22-dc to ADDS

1. Click Promote to ADDS
2. Add a new Forest
3. Configure domain name (I used homelab.test)
4. Configure DSRM password
5. Click install

---

## Create Windows 10 VM
For the win10 machine I gave it these specs:
- Chipset: Q35
- Firmware: BIOS
- CPU: 3vCPU host-passthrough
- Memory: 4GB
- Disk: SATA (cache: none, discard: unmap) 
- CDROMs: Windows 10, virtio-win
- NIC: e1000e NAT
- Display: VGA
- Video: VNC

---


## Setup networking and qemu guest agent for win10

1. Change device name to PC01 via Settings -> System -> About -> Rename this PC 
2. Configure DNS (DNS adress would be DC01's adress)
3. Install QEMU guest agent via virtio-win.iso -> virtio-win-guest-tools.exe
4. Shut down VM
5. Add Channel Hardware (name: org.qemu.guest_agent.0, device type: unix socket, target type: virtio)
6. Boot VM

---


## Create a domain user for win10 on win2k22-dc

1. Open Active Directory Users and Computers
2. Right click homelab.test -> new -> user
3. Give full name and logon name (i used PC01 for the logon name)
4. Set password

---


## Join the homelab.test domain via PC01

1. Settings -> System -> About -> Rename this PC (Advanced) 
2. Select domain and set it to homelab.test
3. Enter domain admin username and password
4. Reboot
5. Login as HOMELAB\PC01

---

## Errors Encountered
### Missing libvirtd dependency
When I ran `systemctl status libvrtd` I got this error:
``` bash
libvirt[12018]: Cannot find 'dmidecode' in path: No such file or directory
```
So I had to install the deepmdency and restart the libvirt daemon
``` bash
sudo pacman -S dmidecode
sudo systemctl restart libvirtd
```


This fixed the error