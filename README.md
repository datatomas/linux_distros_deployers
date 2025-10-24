# Linux Distro Deployer

Tiny, no-nonsense scripts for writing **bootable USB installers** (Ubuntu, Arch, Debian) and handy one-liners for partitioning, wiping signatures, and troubleshooting busy devices.

> ⚠️ **Danger:** These commands **overwrite disks**. Double-check device names (e.g. `/dev/sda`) before running anything. You are responsible for your data.



## Features

- `scripts/write-os-usb.sh`: one command to download (or use a local ISO) and `dd` it to a USB.
- Practical snippets for:
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


# Debian/Ubuntu
sudo apt-get update && sudo apt-get install -y wget parted util-linux lsof cryptsetup

# Arch
sudo pacman -S --needed wget parted lsof cryptsetup
Quick Start – USB ISO Installer
You can either use the repo script at scripts/write-os-usb.sh or save it as your own file ~/usb_iso_installer.sh.

Option A: Use the repo script
bash

# From the repo root
chmod +x scripts/write-os-usb.sh
sudo ./scripts/write-os-usb.sh /dev/sda ubuntu
Option B: Save it as ~/usb_iso_installer.sh
bash

# 1) save it
nano ~/usb_iso_installer.sh   # paste the script, save

# 2) make it executable
chmod +x ~/usb_iso_installer.sh
chmod +x ./usb_iso_installer.sh   # if you are in the same folder

# 3) Run ONE of these (replace /dev/sda if needed)
# Ubuntu 24.04.1 (default)
sudo ./usb_iso_installer.sh /dev/sda ubuntu
The script accepts {ubuntu|arch|debian} and an optional ISO path/URL.
Use a local ISO like:
sudo ./usb_iso_installer.sh /dev/sda ubuntu ~/Downloads/ubuntu-24.04.1-desktop-amd64.iso

Identify your USB device first:

bash

lsblk -o NAME,SIZE,MODEL,TRAN
# Note the WHOLE DISK, e.g. /dev/sda (not /dev/sda1)
Script Reference: write-os-usb.sh
Usage:

bash

sudo ./scripts/write-os-usb.sh /dev/sdX {ubuntu|arch|debian} [ISO_PATH_OR_URL]
Notes:

Pass the whole disk (e.g., /dev/sda), not a partition like /dev/sda1.

Defaults:

Ubuntu version via UB_VER env var (default 24.04.1)

Debian version via DB_VER env var (default 12.6.0)

If ISO_PATH_OR_URL is an http(s):// URL, the script fetches to /tmp before writing.

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
Troubleshooting a “Busy” Device
bash

# Who has the USB open?
sudo lsof /dev/sda /dev/sda?*

# Nudge a stuck dd (replace PIDs as needed)
ps -o pid,stat,cmd -p 45556,49939
sudo kill -INT 45556    # like Ctrl+C for dd

# Soft re-plug the USB root port (adjust bus like "1-5")
echo 1-5 | sudo tee /sys/bus/usb/drivers/usb/unbind
sleep 2
echo 1-5 | sudo tee /sys/bus/usb/drivers/usb/bind
Handy Disk Ops (Optional)
Clean up and write an ISO manually
bash

ISO=~/Downloads/ubuntu-24.04.1-desktop-amd64.iso
DEV=/dev/sda

sudo umount ${DEV}?* 2>/dev/null || true
sudo wipefs -a $DEV
sudo dd if="$ISO" of="$DEV" bs=4M status=progress conv=fsync
sudo partprobe $DEV
Inspect free space
bash

sudo parted -s $DEV unit MiB print free
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT $DEV
Example: Reclaim a Fedora partition and create a new Ubuntu root
bash

DISK=/dev/nvme0n1

# Double-check the disk layout
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT $DISK

# Delete the Fedora partition (e.g., p3) — VERIFY before running!
sudo parted $DISK --script rm 3

# Show free ranges to pick START/END (example numbers below)
sudo parted -s $DISK unit MiB print free

# Example free space:
# Start: 1625MiB  End: 154055MiB
sudo parted -s $DISK -a optimal mkpart primary ext4 1625MiB 154055MiB
sudo parted -s $DISK name 3 ubuntu-root
sudo partprobe $DISK

# (Optional) LUKS2 encryption
PART=${DISK}p3
sudo cryptsetup luksFormat --type luks2 -s 512 -h sha256 -c aes-xts-plain64 $PART
sudo cryptsetup open $PART cryptroot
sudo mkfs.ext4 -L ubuntu-root /dev/mapper/cryptroot
sudo cryptsetup close cryptroot
Installer Notes
In Ubuntu’s installer, choose “Something else” to select:

/boot/efi = your existing EFI partition (do not format)

/ = your new ext4 (or mapped LUKS) partition

For ML/dev workstations, plan 150–200 GB for / and keep datasets on /data.

Repository Layout
lua

linux-distro-deployer/
├─ README.md
├─ .gitignore
└─ scripts/
   └─ write-os-usb.sh







