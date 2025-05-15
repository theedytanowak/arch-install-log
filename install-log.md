## 2025-05-15 - Initial Setup: WiFi + SSH Access

### 1. Check available network devices

`ip link`


### 2. Connect to WiFi (so I can SSH from my Mac)

`iwctl`

`[iwd] device list`


Found wlan0 but it was powered off. Tried:

`[iwd] device wlan0 set-property Powered on`


Got: "Operation failed". Fixed it by:

`[iwd] adapter phy0 set-property Powered on`

`[iwd] device wlan0 set-property Powered on`


Then scanned for networks:

`[iwd] station wlan0 scan`

`[iwd] station wlan0 get-networks`

Networks weren’t showing up at first. Restarted iwd:

`systemctl restart iwd`


Eventually, networks appeared. Connected with:

`iwctl wlan0 connect my_SSID`


Confirmed connection:

`ping archlinux.org`

`iwctl station wlan0 show`

Status: Connected, IP assigned.

### 3. Set root password

`passwd`


### 4. SSH from MacBook

From Mac:

`ssh root@<ARCH_IP_ADDRESS>`


### Notes
- Restarting iwd helped with networks not showing.
- Powering on the adapter before the device matters.
- WiFi config is one of the most annoying parts of Arch setup. Got it working after trial & error.
