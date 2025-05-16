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