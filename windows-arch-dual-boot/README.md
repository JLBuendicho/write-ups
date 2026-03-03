# Dual Boot Setup – Windows 11 + Arch Linux

## 1. Objective
The goal was to set up my machine to be able to run Windows11 and Arch Linux on the same disk. This is so that I can still have a "just works" OS, windows, to work on my school works, and at the same time have a Linux machine to build for learnings. 

---

## 2. System Specifications
- Laptop Model: Lenovo Ideapad Slim 3 14ABR8
- CPU: AMD Ryzen 7 7730U
- RAM: 16.0 GB
- Storage: 477 GB SSD (NVMe)
- Firmware: UEFI
- Disk Partition Table: GPT

---

## 3. Pre-Installation Planning
- Partitioning
    - Windows: 324 GB NTFS
    - Arch: 150 GB BTRFS
- Bootloader:
    - GRUB bootloader

---

## 4. Disk Partitioning Strategy
```
# NVMe SSD
[ EFI - FAT32 ] - /boot
[ Windows NTFS ]
[ Arch BTRFS ]
    |- @
    |- @home
    |- @snapshots
```

> I used btrfs for Arch for the benefit of snapshots, such that I do not have to worry about breaking my machine while playing around

---

## 5. Installation Process
For this set up I started with Windows, the default OS of my machine, and then installed Arch Linux as a second OS

### 5.1 Windows Setup
#### 5.1.1 Free up Space for shrinking later
> This is to make sure we have as much space as possible for when we start installing Arch
1. Disable hibernation (removes hibernation file)
    ``` cmd
    powercfg -h off
    ```
2. Disable page file (removes paging file)
    1. run sysdm.cpl
    2. Advanced Tab -> Performance -> Settings
    3. Advanced Tab -> Virtual Memory -> Change
    4. Uncheck "Automatically manage"
    5. Select C: -> Choose No paging file
    6. Apply and Set
    7. Reboot

3. Disable System Protection (removes protection files)
    1. run SystemPropertiesProtection
    2. Select C:\
    3. Click Configure
    4. Choose Disable system protection
    5. Click OK

4. Defragment Drive (this moves files to the front of the partition)

#### 5.1.2 Disable BitLocker
``` cmd
manage-bde -off C:
```
> This allows us to shrink the partition with a more powerful tool like MiniTool Partition Wizard

#### 5.1.3 Shrink C:\
1. Select C:\
2. Select Move/Resize
    1. Shrink the partition by the size you would like to allocate for arch
    > for this I used MiniTool Partition Wizard

#### 5.1.4 Disable Fast Startup
> This is so that shutdown in windows is a real shutdown. This prevents windows partially hibernating on shutdown. This is so that Linux does not write to the partially hibernating windows partition which would lead to possible file corruption

1. run control
2. Hardware and Sound -> Power Options
3. Click on Choose what the power button does
4. Click change settings that are currently unavailable
5. Uncheck Turn on fast startup
6. Save changes

### 5.2 Linux Setup
For this set up, to maximize learnings and for the fun of a hands on experience, I manually installed Arch linux, following the [Arch Wiki's installation guide](https://wiki.archlinux.org/title/Installation_guide)

#### 5.2.1 Booting into the live environment
To install linux, I flashed an ISO image into my flashdrive and used that as a bootable drive. I made sure to disable secure boot and then I put the flashdrive as first in the boot order to boot into the live environment.

#### 5.2.2 Connect to the internet
1. Unblock wlan from rfkill
    ``` bash
    rfkill unblock wlan
    ```

2. connect to WiFi with iwctl
    1. run interactive prompt
        ``` bash
        iwctl
        ```
    2. power WiFi device on
        ``` bash
        [iwd]# device name set-property Powered on
        ```
    3. scan for networks
        ``` bash
        [iwd]# station name scan
        ```
    4. connect to a network
        ``` bash
        [iwd]# station name connect SSID
        ```
    5. exit iwctl
        ``` bash
        [iwd]# exit
        ```

#### 5.2.3 Partition the disks
For this part the steps would be a bit different from the wiki since I'm dual booting, and also because I wanted to try playing around with btrfs, so I had some guidance from ChatGPT

1. Create a Linux partition in the same disk Windows is in
    1. cfdisk into the disk for an interactive interface
        ``` bash
        cfdisk /dev/nvme0nx
        ```
    2. Select the free space that you freed up for Linux
    3. Create new partition (use all that free space)
    4. Type: Linux filesystem
    5. Write changes
    
    that should create a new Linux FS partition in the disk
    ``` bash
    /dev/nvme0nxpy
    ```
2. Format the Linux partition as btrfs
    1. format with mkfs
        ``` bash
        mkfs.btrfs /dev/nvme0nxpy
        ```
3. Create btrfs subvolumes
    1. temporarily mount partition into /mnt
        > this gives us an interface to interact with the partition
        ```bash
        mount /dev/nvme0nxpy /mnt
        ```
    2. create subvolumes
        ``` bash
        btrfs subvolume create /mnt/@
        btrfs subvolume create /mnt/@home
        btrfs subvolume create /mnt/@snapshots
        ```
    3. unmount
        ``` bash
        umount /mnt
        ```

4. Mount with compression
    > we mount with compression because it allows the filesystem to compress the data written to the disk automatically, this helps us save disk space and improve performance because it is faster to read from compressed files

    1. mount root subvolume
        ``` bash
        mount -o noatime,compress=zstd,subvol=@ /dev/nvme0nxpy /mnt
        ```
        > we also use noatime to disable the update of atime everytime we read a file, this reduces unnecessary writes & improves performance
    2. create mount points
        ``` bash
        mkdir -p /mnt/{home,.snapshots,boot}
        ```
    3. mount /home
        ``` bash
        mount -o noatime,compress=zstd,subvol=@home /dev/nvme0nxpy /mnt/home
        ```
    4. mount /.snapshots
        ``` bash
        mount -o noatime,compress=zstd,subvol=@snapshots /dev/nvme0nxpy /mnt/.snapshots
        ```
5. Mount EFI Partition to /boot
    > this mounts the existing EFI partition to /boot, this is where we will install GRUB along with the existing Windows Boot Manager
    ``` bash
    mount /dev/nvme0nxpz /mnt/boot
    ```

#### 5.2.4 Install Essential Packages
For this I used the basic installation with the Linux kernel and firmware given in the [Arch Wiki](https://wiki.archlinux.org/title/Installation_guide#Install_essential_packages)
``` bash
pacstrap -K /mnt base linux linux-firmware
```

#### 5.2.5 Generate fstab
``` bash
genfstab -U /mnt >> /mnt/etc/fstab
```

#### 5.2.6 Enter chroot
``` bash
arch-chroot /mnt
```

#### 5.2.7 Install aditional packages
In this part I installed a few packages that I find important to go along with the base install
``` bash
pacman -S vim sudo grub efibootmgr
```
- vim for text editing
- sudo for user root perms
- grub and efibootmgr for boot

#### 5.2.8 Set local time
1. set time zone
    ``` bash
    ln -sf /usr/share/zoneinfo/Area/Location /etc/localtime
    ```
2. run hwclock to generate /etc/adjtime
    ``` bash
    hwclock --systohc
    ```

#### 5.2.9 Network configuration
This part assigns us an identifiable name, this comes in handy in a networked environment

1. create host name file
    ``` bash
    vim /etc/hostname
    ```
    ``` bash
    # /etc/hostname
    yourhostname
    ```
#### 5.2.10 initramfs
Because we are using btrfs create a new initramfs

1. Edit /etc/mkinitcpio.conf
    1. run `vim /etc/mkinitcpio.conf`
    2. change:
        ```bash
        HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
        ```
        to:
        ``` bash
        HOOKS=(base udev autodetect modconf block btrfs filesystems keyboard fsck)
        ```
2. Rebuild initramfs
    ```bash
    mkinitcpio -P
    ```

#### 5.2.11 Set root password
1. run `passwd`
2. enter password

#### 5.2.11 Set up GRUB
1. Install GRUB to EFI partition
    ``` bash
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    ```
2. Generate GRUB configuration
    ``` bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

#### 5.2.12 Reboot
1. exit chroot environment by typing `exit`
2. unmount all partitions
    ```bash
    umount -R /mnt
    ```
3. reboot by typing `reboot`

### 5.3 Bootloader Configuration
Because we are dual booting Windows + Arch Linux, we have a few more steps in configuring GRUB

1. install os-prober
    > this is so that GRUB detects windows
    ``` bash
    pacman -S os-prober
    ```
2. enable os-prober
    1. in `/etc/default/grub` uncomment or add:
    ``` bash
    GRUB_DISABLE_OS_PROBER=false
    ```
3. generate GRUB configuration
    ``` bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

### 5.4 Post install Windows Clean up
In this step we undo the steps from [5.1.1 Free up Space for shrinking later](#511-free-up-space-for-shrinking-later)


1. Enable hibernation
    ``` cmd
    powercfg -h on
    ```
2. Enable page files
    1. run sysdm.cpl
    2. Advanced Tab -> Performance -> Settings
    3. Advanced Tab -> Virtual Memory -> Change
    4. Select C: -> Choose System managed size
    5. check "Automatically manage"
    6. Apply and Set
    7. Reboot

3. Enable System Protection
    1. run SystemPropertiesProtection
    2. Select C:\
    3. Click Configure
    4. Choose Turn on system protection
    5. Click Apply and OK

4. Defragment Drive (this moves files to the front of the partition)

## 6. Troubleshooting & Issues Encountered
During this install, towards the end, I forgot to install and set up GRUB so I had to boot into the live environment again and do the following (with guidance from ChatGPT):

1. Mount the installed Arch System
    ``` bash
    # Mount root subvolume
    mount -o subvol=@ /dev/nvme0nxpy /mnt

    # Mount home
    mount -o subvol=@home /dev/nvme0nxpy /mnt/home

    # Mount EFI
    mount /dev/nvme0nxpz /mnt/boot
    ```
2. chroot into Arch
    ``` bash
    arch-chroot /mnt
    ```
3. Install GRUB
    ``` bash
    pacman -S grub efibootmgr
    ```
4. Install GRUB to EFI partition
    ``` bash
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    ```
5. install os-prober
    > this is so that GRUB detects windows
    ``` bash
    pacman -S os-prober
    ```
6. enable os-prober
    1. in `/etc/default/grub` uncomment or add:
    ``` bash
    GRUB_DISABLE_OS_PROBER=false
    ```
7. generate GRUB configuration
    ``` bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
8. exit chroot
9. reboot

## 7. Final Boot Flow Explanation
In this set up we have Windows and Arch Linux in one disk.

This is how the system boots:
1. UEFI firmware runs
2. UEFI reads GPT and finds the EFI System Partition
3. UEFI loads the configured GRUB EFI file
4. GRUB menu opens
5. We select which OS to load
6. OS loads

## 8. Lessons Learned
During the set up and install I learned how to do Dual booting and many other things.

During the setup for windows I encountered BitLocker which introduced me to Disk Encryption and how it could prevent us from resizing a partition.

During the disk partitioning with btrfs I encountered compression and atime, I learned about how data compression is good for storing files as it uses space more efficiently and allows for better performance when it comes to reading files, and I also learned about atime and how it is good to disable atime for file reads to reduce unnecessary writes

And after the set up and install I learned about how systems boot, how UEFI interacts with GPT and how GRUB gets loaded through a configured UEFI file to allow us to select the OS to load