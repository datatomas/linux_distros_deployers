# Linux Distro Deployer

Tiny, no-nonsense scripts for writing **bootable USB installers** (Ubuntu, Arch, Debian) and for secure reinstalls using **LUKS2 encryption** (delete old OS partition → recreate → install Ubuntu).

 ⚠️ **Danger:** These commands **overwrite disks**. Double-check device names (e.g., `/dev/sda`, `/dev/nvme0n1`) before running anything. You are responsible for your data.



## Repository Layout

linux-distro-deployer/
├─ README.md
├─ .gitignore
└─ scripts/
└─ write-os-usb.sh

markdown




## Features

- **Bootable USB creation**: `scripts/write-os-usb.sh` writes a distro ISO to a USB drive (Ubuntu, Arch, Debian).
- **Secure reinstall guide**: step-by-step flow to remove an old OS partition, create a new one, encrypt it with **LUKS2**, and point the Ubuntu installer at it.
- **Useful snippets** for:
  - Unmounting / wiping partition signatures
  - Refreshing the kernel’s partition view
  - Checking which processes hold a device open
  - Soft re-plugging a USB root port
  - Creating / encrypting partitions (LUKS2)



## Requirements

- Linux with `bash`
- Tools: `wget`, `dd`, `lsblk`, `parted`, `wipefs`
- Optional: `lsof`, `cryptsetup`, `nvme-cli`

Install examples:

```bash
# ==============================================================================
# Linux Distro Deployer — ONE-FILE INSTRUCTIONS (ALL COMMENTED)
# Purpose:
#   - Use your existing USB writer script to make bootable installers
#   - Secure reinstall flow: delete old OS partition → create new → LUKS2 encrypt → install Ubuntu
# Notes:
#   - EVERYTHING here is comments. Remove the leading "# " to run a command.
#   - ⚠️ These operations can erase data. Triple-check device names.
# ==============================================================================


# 
# REQUIREMENTS (install tools)
# 

# Debian/Ubuntu
# sudo apt-get update && sudo apt-get install -y wget parted util-linux lsof cryptsetup

# Arch
# sudo pacman -S --needed wget parted lsof cryptsetup


# 
# QUICK START — USE YOUR EXISTING USB WRITER SCRIPT
# You already have a script that writes ISOs to a USB. You don't need to edit it.
# Pick either the repo copy or your own saved copy.
# 

# Option A: Use the repo script
# (From the repo root)
# chmod +x scripts/write-os-usb.sh
# sudo ./scripts/write-os-usb.sh /dev/sdX ubuntu
#   - Replace /dev/sdX with your USB device (e.g., /dev/sda). Use the WHOLE disk, not /dev/sda1.

# Option B: Use your own saved script
# nano ~/usb_iso_installer.sh            # paste your existing script, save
# chmod +x ~/usb_iso_installer.sh
# sudo ./usb_iso_installer.sh /dev/sdX ubuntu

# The script accepts {ubuntu|arch|debian} and an optional local ISO path or URL.
# Example (local ISO):
# sudo ./usb_iso_installer.sh /dev/sdX ubuntu ~/Downloads/ubuntu-24.04.1-desktop-amd64.iso

# Identify your USB device first (WHOLE disk, not a partition):
# lsblk -o NAME,SIZE,MODEL,TRAN


# 
# SCRIPT REFERENCE — write-os-usb.sh (HOW TO USE, NOT THE CODE)
# 

# Usage:
# sudo ./scripts/write-os-usb.sh /dev/sdX {ubuntu|arch|debian} [ISO_PATH_OR_URL]

# Notes:
# - Pass the WHOLE disk (e.g., /dev/sda), not /dev/sda1.
# - Defaults:
#     Ubuntu: UB_VER=24.04.1
#     Debian: DB_VER=12.6.0
# - If ISO_PATH_OR_URL is http(s)://, the script downloads to /tmp then writes.

# Examples:
# sudo ./scripts/write-os-usb.sh /dev/sda ubuntu
# sudo ./scripts/write-os-usb.sh /dev/sda arch
# sudo ./scripts/write-os-usb.sh /dev/sda debian
# sudo ./scripts/write-os-usb.sh /dev/sda ubuntu ~/Downloads/ubuntu-24.04.1-desktop-amd64.iso


# 
# SECURE REINSTALL CHEAT-SHEET
# Replace existing OS partition with encrypted Ubuntu root (LUKS2)
# Flow: Identify → Delete old OS partition → Create new partition → LUKS2 → mkfs → Install
# 

# 0) Identify the right devices
# lsblk -o NAME,SIZE,MODEL,TRAN
# lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT
# DISK=/dev/nvme0n1                          # example target disk
# lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT $DISK

# 1) Delete the old OS partition (example uses p3)  ⚠️ VERIFY NUMBER
# sudo parted $DISK --script rm 3
# sudo parted -s $DISK unit MiB print free     # note Start/End of FREE region

# Example FREE range (yours will differ):
# Start: 1625MiB
# End:   154055MiB

# 2) Create a new partition in the free space
# START=1625MiB                                # replace with YOUR Start
# END=154055MiB                                # replace with YOUR End
# sudo parted -s $DISK -a optimal mkpart primary ext4 $START $END
# sudo parted -s $DISK name 3 ubuntu-root      # adjust number if not 3
# sudo partprobe $DISK
# lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT $DISK

# 3) Encrypt the new partition with LUKS2
# PART=${DISK}p3                               # adjust if different
# sudo cryptsetup luksFormat --type luks2 -s 512 -h sha256 -c aes-xts-plain64 $PART
# sudo cryptsetup open $PART cryptroot

# 4) Create filesystem inside the encrypted mapper
# sudo mkfs.ext4 -L ubuntu-root /dev/mapper/cryptroot

# (Optional) Quick check
# sudo mkdir -p /mnt/cryptroot
# sudo mount /dev/mapper/cryptroot /mnt/cryptroot
# df -h /mnt/cryptroot
# sudo umount /mnt/cryptroot

# Close until the installer uses it
# sudo cryptsetup close cryptroot

# 5) Install Ubuntu (“Something else” method)
# - Boot from your Ubuntu USB installer.
# - Choose “Something else”.
# - EFI System Partition (existing, ~100–500 MiB, FAT32):
#     Use as: EFI System Partition
#     Mount point: /boot/efi
#     Do NOT format
# - Encrypted root:
#     Select the partition, click Unlock (enter LUKS passphrase)
#     Pick the MAPPED device (e.g., /dev/mapper/cryptroot)
#     Use as: Ext4
#     Mount point: /
#     Format: Yes
# - Proceed; the installer will write /etc/crypttab and /etc/fstab.

# Post-install tips:
# - Disable Fast Boot in BIOS/UEFI to easily reach boot menus.
# - Consider a separate encrypted /data or /home (repeat LUKS + mkfs).
# - Add another LUKS key:
#   sudo cryptsetup luksAddKey ${DISK}p3


# 
# TROUBLESHOOTING (“DEVICE BUSY”, USB ODDITIES)
# 

# Who has the device open?
# sudo lsof /dev/sda /dev/sda?*     # or /dev/nvme0n1 /dev/nvme0n1p?

# If dd is stuck, send Ctrl+C (replace PID):
# ps -o pid,stat,cmd -p 45556,49939
# sudo kill -INT 45556

# Soft re-plug a USB root port (adjust bus like "1-5"):
# echo 1-5 | sudo tee /sys/bus/usb/drivers/usb/unbind
# sleep 2
# echo 1-5 | sudo tee /sys/bus/usb/drivers/usb/bind

# Refresh partition table and re-check:
# sudo partprobe /dev/sda || sudo blockdev --rereadpt /dev/sda
# lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL /dev/sda
# sudo blkid /dev/sda*



# ==============================================================================
# END OF FILE
# ==============================================================================

