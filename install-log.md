## 2025-05-16 – Disk Partitioning & Encrypting

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
