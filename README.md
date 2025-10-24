# Linux Distro Deployer

Tiny, no-nonsense scripts for writing bootable USB installers (Ubuntu, Arch, Debian) and
handy one-liners for partitioning, wiping signatures, and troubleshooting busy devices.

> ⚠️ **Danger:** These commands overwrite disks. Double-check device names (e.g. `/dev/sda`)
> before running anything. You are responsible for your data.

## Features
- `write-os-usb.sh`: one command to download (or use a local ISO) and `dd` it to a USB
- Helpful snippets for:
  - Unmounting / wiping partition signatures
  - Refreshing the kernel’s partition view
  - Checking which processes hold a device open
  - Soft re-plugging a USB bus
  - Creating / encrypting partitions (LUKS2)

## Requirements
- Linux with `bash`
- `wget`, `dd`, `lsblk`, `parted`, `wipefs`
- Optional: `lsof`, `cryptsetup`, `nvme-cli`

Install missing bits (examples):
```bash
# Debian/Ubuntu
sudo apt-get update && sudo apt-get install -y wget parted util-linux lsof cryptsetup

# Arch
sudo pacman -S --needed wget parted lsof cryptsetup
#Usb Iso instaler
# 1) save it
nano ~/usb_iso_installer.sh   # paste the script, save

# 2) make it executable
chmod +x ~/usb_iso_installer.sh
chmod +x ./usb_iso_installer.sh   #if yo are ont he same fodler
# 3) Run ONE of these (replace /dev/sda if needed)
# Ubuntu 24.04.1 (default)
sudo ./usb_iso_installer.sh /dev/sda ubuntu
