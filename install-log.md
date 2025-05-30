## 2025-05-15 – Initial Setup: WiFi + SSH Access

### 1. Check available network devices

```bash
ip link
```

---

### 2. Connect to WiFi (so I can SSH from my Mac)

```bash
iwctl
[iwd] device list
```

Found `wlan0` but it was powered off.

Tried:
```bash
[iwd] device wlan0 set-property Powered on
```

Got: `"Operation failed"`

Fixed it by:
```bash
[iwd] adapter phy0 set-property Powered on
[iwd] device wlan0 set-property Powered on
```

Scanned and connected:
```bash
[iwd] station wlan0 scan
[iwd] station wlan0 get-networks
systemctl restart iwd
[iwd] station wlan0 connect my_SSID
ping archlinux.org
[iwd] station wlan0 show
```

→ **Status: Connected, IP assigned**

---

### 3. Set root password

```bash
passwd
```

---

### 4. SSH from MacBook

From Mac:
```bash
ssh root@<ARCH_IP_ADDRESS>
```

---

### Notes
- Restarting `iwd` helped with networks not showing.
- Powering on the adapter *before* the device matters.
- WiFi config is one of the most annoying parts of Arch setup. Got it working after trial & error.

---
## 2025-05-16 – Full System Setup: Partitioning, Encryption, Base Install, Networking, mkinitcpio

### 1. Check the current disk partitioning

```bash
lsblk
fdisk -l
```

Confirmed I need to work on `/dev/sda` (not the USB stick with the same size).

Started partitioning:
```bash
fdisk /dev/sda
```

I created two partitions:
- `/dev/sda1` for EFI boot (1G)
- `/dev/sda2` for everything else, using LVM

Final result:
```
Device       Start       End   Sectors   Size Type
/dev/sda1     2048   2099199   2097152     1G EFI System
/dev/sda2  2099200 234440703 232341504 110.8G Linux LVM
```

---

### 2. Encrypting /dev/sda2 (LVM on LUKS)

```bash
cryptsetup luksFormat /dev/sda2
cryptsetup open /dev/sda2 cryptlvm
pvcreate /dev/mapper/cryptlvm
vgcreate archvg /dev/mapper/cryptlvm
```

Confirmed the container is open:
```bash
ls /dev/mapper
# shows: control  cryptlvm
```

Created logical volumes:
```bash
lvcreate -L 4G -n swap archvg
lvcreate -L 40G -n root archvg
lvcreate -L 66.5G -n home archvg
```

Formatted and mounted:
```bash
mkfs.ext4 /dev/archvg/root
mkfs.ext4 /dev/archvg/home
mkswap /dev/archvg/swap

mount /dev/archvg/root /mnt
mount --mkdir /dev/archvg/home /mnt/home
swapon /dev/archvg/swap
```

---

### 3. Preparing the boot partition

```bash
mkfs.fat -F32 /dev/sda1
mount --mkdir /dev/sda1 /mnt/boot
```

(I initially mounted the wrong partition: `/dev/sdb1`, then corrected it after verifying with `lsblk -f`.)

---

### 4. Continuing with Arch installation

Installing essential packages:

```bash
pacstrap -K /mnt base linux linux-firmware
```

Generated an fstab file for automatic mounting:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Changed root to the new system's:

```bash
arch-chroot /mnt
```

Set the timezone and updated the locale:

```bash
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/^#pl_PL.UTF-8 UTF-8/pl_PL.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
```

Realized I actually need `vim`:

```bash
pacman -Syu vim
```

Temporarily lost SSH when switching networks on my MacBook, but the Arch install kept running. Reconnected via SSH and resumed work:

```bash
ssh root@<ARCH_IP>
arch-chroot /mnt
pacman -Syu which man-db man-pages texinfo intel-ucode
```

Set Polish keymap for bilingual writing:

```bash
echo "KEYMAP=pl" > /etc/vconsole.conf
```

---

### 5. Networking

Enabled systemd networking:

```bash
systemctl enable systemd-networkd.service
systemctl enable systemd-resolved.service
```

Configured wireless network:

```bash
vim /etc/systemd/network/25-wireless.network
```

Pasted:

```ini
[Match]
Name=wlan0

[Link]
RequiredForOnline=routable

[Network]
DHCP=yes
IgnoreCarrierLoss=3s
```

Installed iwd to handle wireless authentication:

```bash
pacman -Syu iwd
systemctl enable iwd.service
```

Set the hostname:

```bash
echo "archlinux" > /etc/hostname
```

---

### 6. Configure mkinitcpio

`mkinitcpio` builds the initramfs – a tiny pre-boot Linux environment. It's essential for mounting the encrypted disk and LVM volumes.

Edited `/etc/mkinitcpio.conf` and updated the hooks:

```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```

Regenerated the initramfs:

```bash
mkinitcpio -P
```

This initially returned an error due to a missing binary (`pdata_tools`).

Fixed it by installing `lvm2` (I had skipped it earlier):

```bash
pacman -S lvm2
```

Installing lvm2 pulled in the missing pdata_tools binary, resolving the mkinitcpio errors. It also triggered an automatic initramfs rebuild via pacman's post-install hooks. No errors anymore.
---

### Notes

- I chose LUKS + LVM because I want to learn common enterprise-level setups (and suffer less when managing multiple volumes).
- First time I understood how LVM layers work: physical volume → volume group → logical volumes.
- `cryptsetup open /dev/sda1` was a mistake I almost made – glad I double-checked the partition (`sda2` is for encryption).
- `mkinitcpio` is required to bootstrap the kernel, unlock encryption, activate LVM, and mount the root filesystem.
---

## 2025-05-17 Bootloader & Failed reboot


### 1. Installing systemd-boot - easy-to-configure UEFI boot manager

SSH-ed back into the Arch system and chrooted into the installed environment.
Initial install attempt:

`bootctl install`

Then I noticed an important note on the Arch Wiki:

```text
This section is out of date!
Reason: When running in a PID namespace (as arch-chroot does), bootctl no longer creates the UEFI boot entry in NVRAM (since systemd v257).
See: https://github.com/systemd/systemd/issues/36174
```

To fix this:

1. **Exited the chroot**, then ran:
   ```bash
   bootctl --esp-path=/mnt/boot install
   ```
   This correctly wrote the EFI files and registered the boot entry on the actual ESP.

2. **Chrooted back in**, and re-ran:
   ```bash
   bootctl install
   ```
   This ensures all config files (`loader.conf`, random seed, etc.) are created inside the real system.

systemd-boot is now correctly installed and registered with UEFI.
---

### 2. Loader configuration

Created `/boot/loader/entries/arch.conf`:

```ini
title   Arch Linux                       # Label shown in the boot menu
linux   /vmlinuz-linux                   # Kernel image path (relative to /boot)
initrd  /intel-ucode.img                 # Intel CPU microcode (must come before initramfs)
initrd  /initramfs-linux.img             # Main initramfs image
options cryptdevice=UUID=4d70747b-44a1-4c6a-94c6-ef83f3b5f3cc:cryptlvm root=/dev/archvg/root rw
```

Explanation of the `options` line:

- `cryptdevice=UUID=...:cryptlvm`: tells the initramfs to unlock the encrypted LUKS device at boot using the given UUID and map it to `/dev/mapper/cryptlvm`
- `root=/dev/archvg/root`: specifies the logical volume to mount as the root filesystem
- `rw`: mounts the root filesystem as read-write
---

### 3. Bootloader loader configuration

Edited `/boot/loader/loader.conf` to define the default boot entry and timeout:

```ini
default arch              # Match the filename: /boot/loader/entries/arch.conf
timeout 3                 # Show boot menu for 3 seconds
editor no                 # Disable manual editing at boot (recommended)
```

This ensures systemd-boot loads the correct entry automatically and consistently.
---

### 4. User creation and sudo setup

Created a regular user account:

```bash
useradd edyta
```

Realized I forgot `-m` (create home) and `-G wheel` (sudo group), so I manually fixed it:

```bash
mkdir -p /home/edyta
chown edyta:edyta /home/edyta
usermod -aG wheel edyta
```

Set the user password:

```bash
passwd edyta
```

Then configured sudo by editing the sudoers file using `visudo`:

```bash
visudo
```

Uncommented the following line to enable sudo for users in the `wheel` group:

```text
%wheel ALL=(ALL:ALL) ALL
```
---
### 5. Root password setup

Checked the root account status:

```bash
passwd -S root
```

Got:
```text
root L 2010-09-19 -1 -1 -1 -1
```

Realized the password was never set inside the chrooted system (only in the live environment). Fixed it:

```bash
passwd
```

Confirmed it's now active:

```bash
passwd -S root
# root P 2025-05-17 -1 -1 -1 -1
```
---
### 6. First reboot and boot failure

Exited the chroot and prepared for reboot:

```bash
exit
umount -R /mnt
swapoff -a
reboot
```

Removed the USB stick as the machine restarted.

systemd-boot loaded and displayed the "Arch Linux" entry. Selected it.

🛑 **Boot failed — system dropped into emergency shell.**

No LUKS password prompt appeared. After a long delay, the following error was shown:

```text
Timed out waiting for device /dev/archvg/root.
Dependency failed for Initrd Root Device.
Dependency failed for /sysroot.
Dependency failed for Initrd Root File System.
```

System entered emergency mode. Access was denied because the root account was locked.

---

### 7. Extensive Troubleshooting (unsuccessful)

Many components of the setup were confirmed to be correct, including bootloader installation, UUID usage, and initramfs content. Despite this, the root volume could not be mounted because decryption never occurred.

Booted back into the live ISO and attempted multiple rounds of troubleshooting:

- ✅ Verified the `UUID` used in `arch.conf` matched `/dev/sda2`
- ✅ Confirmed the `cryptdevice` mapping and `root=/dev/archvg/root` were correct
- ✅ Inspected `/etc/mkinitcpio.conf` — confirmed presence of critical hooks:
  ```bash
  HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
  ```
- ✅ Rebuilt initramfs multiple times using:
  ```bash
  mkinitcpio -P
  ```
- ✅ Re-ran both:
  ```bash
  bootctl --esp-path=/mnt/boot install
  bootctl install
  ```
- ✅ Checked that `/boot/initramfs-linux.img` and `/boot/loader/entries/arch.conf` existed and were properly linked
- ✅ Used `lsinitcpio` to confirm presence of `sd-encrypt` in the initramfs

Despite all efforts, the system still failed to prompt for LUKS decryption and dropped into emergency mode due to inability to mount the root volume.

---

### 8. Lessons Learned & Next Attempt Checklist

This installation attempt was ultimately unsuccessful, but produced valuable insight. Future attempts will apply the following rigorously:

**Partitioning:**
- `/dev/sda1` → EFI (FAT32, 1G)
- `/dev/sda2` → LUKS container for LVM

**Encryption and LVM setup:**
```bash
cryptsetup luksFormat /dev/sda2
cryptsetup open /dev/sda2 cryptlvm
pvcreate /dev/mapper/cryptlvm
vgcreate archvg /dev/mapper/cryptlvm
lvcreate -L 4G -n swap archvg
lvcreate -L 40G -n root archvg
lvcreate -l 100%FREE -n home archvg
```

**mkinitcpio config:**
```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
mkinitcpio -P
```

**systemd-boot install (must be done in this exact sequence):**
```bash
bootctl --esp-path=/mnt/boot install   # From live ISO
arch-chroot /mnt
bootctl install                         # Inside the chroot
```

**Correct `/boot/loader/entries/arch.conf`:**
```ini
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options cryptdevice=UUID=<UUID-of-sda2>:cryptlvm root=/dev/archvg/root rw
```

Use `blkid /dev/sda2` to replace `<UUID>`.

**Sanity checks before reboot:**
```bash
ls /boot/initramfs-linux.img
ls /boot/loader/entries/arch.conf
lsinitcpio /boot/initramfs-linux.img | grep sd-encrypt
```

If `sd-encrypt` is missing from the image, LUKS unlock will never be triggered.

---

Final result: The system was set up correctly in many ways, but boot failure confirmed a critical issue in the decryption pipeline. Will reattempt with lessons fully integrated.
---

## 2025-05-18 Installation attempt #2

### 1. Prep-work

- Verified BIOS settings (UEFI, Secure Boot disabled)
- Downloaded latest Arch Linux ISO
- Created a bootable USB using balenaEtcher on the MacBook
- Booted into the live environment
- Powered on the network adapter and wlan0 device:

```bash
iwctl
[iwd] adapter phy0 set-property Powered on
[iwd] device wlan0 set-property Powered on
[iwd] station wlan0 scan
[iwd] station wlan0 connect my_SSID
```

Got: "Operation failed"
Fixed it:

```bash
exit
systemctl restart iwd
iwctl station wlan0 connect my_SSID
```

- entered passphrase
- verified network connectivity

`ping archlinux.org`

- set root password:

`passwd`

- confirmed IP address of the device

`iwctl station wlan0 show`
---

### 2. SSH from MacBook

- deleted outdated entries from known_hosts:

`vim .ssh/known_hosts`

- connected via SSH:

`ssh root@192.168.x.x`

---

### 3. Update system clock

- synced system clock:

`timedaectl`
---

### 4. Partitioning & LVM setup

- verified correct disk:

```bash
lsblk
fdisk -l
```

- partitioned disk:

`fdisk /dev/sda`

- created:

```text
/dev/sda1 - EFI System (1G)
/dev/sda2 - Linux LVM (rest of disk)
```

- Encrypted LVM partition:

```bash
cryptsetup luksFormat /dev/sda2
cryptsetup open /dev/sda2 cryptlvm
```

---

### 5. Logical Volumes

- initialized LVM:

```bash
pvcreate /dev/mapper/cryptlvm
vgcreate archvg /dev/mapper/cryptlvm
```

- created LVs (left 256MB for e2scrub):

```bash
lvcreate -L 4G archvg -n swap
lvcreate -L 40G archvg -n root
lvcreate -l 100%FREE archvg -n home
lvreduce -L -256M archvg/home
```

- formatted partitions:

```bash
mkfs.ext4 /dev/archvg/root
mkfs.ext4 /dev/archvg/home
mkswap /dev/archvg/swap
mkfs.fat -F32 /dev/sda1
```

- mounted partitions:

```bash
mount /dev/archvg/root /mnt
mount --mkdir /dev/sda1 /mnt/boot
swapon /dev/archvg/swap
```

- fixed mount layering issue (EFI was mounted before root):

```bash
umount /mnt
umount /dev/sda1
mount /dev/archvg/root /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```
---

### 6. Base System Installation

- Installed packages:

```bash
pacstrap -K /mnt base base-devel linux linux-firmware git btrfs-progs grub efibootmgr grub-btrfs inotify-tools timeshift vim networkmanager pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector zsh zsh-completions zsh-autosuggestions openssh man sudo man-pages man-db texinfo lvm2 which
```
(Reminder: add iwd in the future)
---

### 7. Configure the System

- generated fstab:

`genfstab -U /mnt >> /mnt/etc/fstab`

- entered new system:

`arch-chroot /mnt`

- set timezone and clocks:

```bash
ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
hwclock --systohc
```

- edited locale:

```bash
vim /etc/locale.gen  # uncommented en_US.UTF-8 UTF-8
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```
---

### 8. Network config

- set hostname:

`echo "edyta-arch" > /etc/hostname`

- enabled services:

```bash
systemctl enable systemd-networkd.service
systemctl enable systemd-resolved.service
```
- configured wireless networking:

```ini
# /etc/systemd/network/25-wireless.network
[Match]
Name=wlan0

[Network]
DHCP=yes
IgnoreCarrierLoss=3s
```

- installed iwd:

```bash
pacman -Syu iwd
systemctl enable iwd.service
```
---

### 9. mkinitcpio

- edited hooks:

```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```

- rebuilt initramfs:

`mkinitcpio -P`
---

### 10. Bootloader

- installed systemd-boot:

```bash
bootctl --esp-path=/mnt/boot install
arch-chroot /mnt
bootctl install
```

- found UUID of encrypted device:

`blkid /dev/sda2`

- created /boot/loader/entries/arch.conf:

```ini
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options rd.luks.name=<UUID>=cryptlvm root=/dev/archvg/root rw
```

### 11. User & Sudo

- Created user and added to wheel group:

```bash
useradd edyta
usermod -aG wheel edyta
```

- enabled sudo:

```bash
visudo
# uncommented `%wheel ALL=(ALL:ALL) ALL`
```
- set passwords:

```bash
passwd edyta
passwd
```

### Results:

System booted successfully into encrypted Arch install.

Time check: 03:05 AM.