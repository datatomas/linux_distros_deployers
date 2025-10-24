# Linux Distro Deployer

Tiny, no-nonsense scripts for writing **bootable USB installers** (Ubuntu, Arch, Debian) and for secure reinstalls using **LUKS2 encryption** (delete old OS partition → recreate → install Ubuntu).

> ⚠️ **Danger:** These commands **overwrite disks**. Double-check device names (e.g., `/dev/sda`, `/dev/nvme0n1`) before running anything. You are responsible for your data.

---

## Repository Layout

linux-distro-deployer/
├─ README.md
├─ .gitignore
└─ scripts/
└─ write-os-usb.sh

markdown


---

## Features

- **Bootable USB creation**: `scripts/write-os-usb.sh` writes a distro ISO to a USB drive (Ubuntu, Arch, Debian).
- **Secure reinstall guide**: step-by-step flow to remove an old OS partition, create a new one, encrypt it with **LUKS2**, and point the Ubuntu installer at it.
- **Useful snippets** for:
  - Unmounting / wiping partition signatures
  - Refreshing the kernel’s partition view
  - Checking which processes hold a device open
  - Soft re-plugging a USB root port
  - Creating / encrypting partitions (LUKS2)

---

## Requirements

- Linux with `bash`
- Tools: `wget`, `dd`, `lsblk`, `parted`, `wipefs`
- Optional: `lsof`, `cryptsetup`, `nvme-cli`

Install examples:

```bash
# Debian/Ubuntu
sudo apt-get update && sudo apt-get install -y wget parted util-linux lsof cryptsetup

# Arch
sudo pacman -S --needed wget parted lsof cryptsetup
Quick Start — Use the Existing USB Script
You already have a script to write ISOs to USB. Use either the repo copy or your own copy—you don’t need to modify its code here.

Option A: Use the repo script
bash

# From the repo root
chmod +x scripts/write-os-usb.sh
sudo ./scripts/write-os-usb.sh /dev/sdX ubuntu
Option B: Use your own saved script
bash

# Save it (if you haven't already)
nano ~/usb_iso_installer.sh   # paste your existing script, save

# Make it executable
chmod +x ~/usb_iso_installer.sh

# Run it (replace /dev/sdX with your USB device)
sudo ./usb_iso_installer.sh /dev/sdX ubuntu
The script accepts {ubuntu|arch|debian} and optionally a local ISO path or URL.
Example (local ISO):
sudo ./usb_iso_installer.sh /dev/sdX ubuntu ~/Downloads/ubuntu-24.04.1-desktop-amd64.iso

Identify your USB device first:

bash

lsblk -o NAME,SIZE,MODEL,TRAN
# Use the WHOLE DISK (e.g., /dev/sda), NOT a partition like /dev/sda1.
Script Reference — write-os-usb.sh (how to use)
Usage:

bash

sudo ./scripts/write-os-usb.sh /dev/sdX {ubuntu|arch|debian} [ISO_PATH_OR_URL]
Notes:

Pass the whole disk (e.g., /dev/sda), not /dev/sda1.

Defaults:

Ubuntu: UB_VER=24.04.1

Debian: DB_VER=12.6.0

If ISO_PATH_OR_URL is an http(s):// URL, the script downloads to /tmp then writes.

Examples:

bash

# Ubuntu (default version)
sudo ./scripts/write-os-usb.sh /dev/sda ubuntu

# Arch (latest)
sudo ./scripts/write-os-usb.sh /dev/sda arch

# Debian (netinst)
sudo ./scripts/write-os-usb.sh /dev/sda debian

# Use a local ISO explicitly
sudo ./scripts/write-os-usb.sh /dev/sda ubuntu ~/Downloads/ubuntu-24.04.1-desktop-amd64.iso
Secure Reinstall Cheat-Sheet (Delete Partition → LUKS2 → Install Ubuntu)
Purpose: Replace an existing OS partition with an encrypted Ubuntu root.
Flow: Identify → Delete old OS partition → Create new partition → LUKS2 encrypt → Make ext4 → Install Ubuntu (Something else).

0) Identify the right devices
bash

# Disks and transport (SATA/NVMe/USB)
lsblk -o NAME,SIZE,MODEL,TRAN

# Filesystem types and mount points
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT

# Choose your target disk (example)
DISK=/dev/nvme0n1

# Inspect layout
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT $DISK
1) Delete the old OS partition (example uses p3)
bash

# ⚠️ Replace '3' with your actual partition number
sudo parted $DISK --script rm 3

# Show the new FREE region (note Start/End in MiB)
sudo parted -s $DISK unit MiB print free
Example FREE range (yours will differ):
Start: 1625MiB → End: 154055MiB

2) Create a new partition in the free space
bash

# Replace with YOUR free-range bounds
START=1625MiB
END=154055MiB

sudo parted -s $DISK -a optimal mkpart primary ext4 $START $END
sudo parted -s $DISK name 3 ubuntu-root   # adjust number if different
sudo partprobe $DISK

# Confirm it appeared
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT $DISK
3) Encrypt the new partition with LUKS2
bash

PART=${DISK}p3   # adjust if number differs

# Initialize LUKS2 (you'll set a passphrase)
sudo cryptsetup luksFormat --type luks2 -s 512 -h sha256 -c aes-xts-plain64 $PART

# Open (map) it
sudo cryptsetup open $PART cryptroot
4) Create filesystem inside the encrypted mapper
bash

sudo mkfs.ext4 -L ubuntu-root /dev/mapper/cryptroot

# Optional quick check
sudo mkdir -p /mnt/cryptroot
sudo mount /dev/mapper/cryptroot /mnt/cryptroot
df -h /mnt/cryptroot
sudo umount /mnt/cryptroot

# Close until the installer uses it
sudo cryptsetup close cryptroot
5) Install Ubuntu (manual “Something else”)
Boot the Ubuntu USB you created.

Choose “Something else.”

Select your EFI System Partition (usually 100–500 MiB, FAT32):

Use as: EFI System Partition

Mount point: /boot/efi

Do not format

Select the new encrypted root:

Unlock it (enter LUKS passphrase)

Choose the mapped device (e.g., /dev/mapper/cryptroot)

Use as: Ext4

Mount point: /

Format: Yes

Proceed with the install. The installer writes /etc/crypttab and /etc/fstab for you.

Troubleshooting (“Device busy”, USB oddities)
bash

# Who has the device open?
sudo lsof /dev/sda /dev/sda?*            # or /dev/nvme0n1 /dev/nvme0n1p?

# If dd is stuck, send it Ctrl+C (replace PID):
ps -o pid,stat,cmd -p 45556,49939
sudo kill -INT 45556

# Soft re-plug a USB root port (adjust bus ID like "1-5")
echo 1-5 | sudo tee /sys/bus/usb/drivers/usb/unbind
sleep 2
echo 1-5 | sudo tee /sys/bus/usb/drivers/usb/bind

# Refresh partition table and re-check
sudo partprobe /dev/sda || sudo blockdev --rereadpt /dev/sda
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL /dev/sda
sudo blkid /dev/sda*
Post-Install Tips
BIOS/UEFI: Disable Fast Boot so you can reach boot menus.

Data volumes: Consider a separate encrypted /data or /home.

Add another LUKS key:

bash

sudo cryptsetup luksAddKey ${DISK}p3
