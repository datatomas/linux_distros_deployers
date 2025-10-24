Linux Distro Deployer

Tiny, no-nonsense scripts for writing bootable USB installers (Ubuntu, Arch, Debian) and a secure reinstall flow using LUKS2 encryption (delete old OS partition → recreate → encrypt → install Ubuntu).

⚠️ Danger: These commands overwrite disks. Triple-check device names (e.g. /dev/sda, /dev/nvme0n1) before running. You are responsible for your data.

Repository Layout
linux-distro-deployer/
├─ README.md
├─ .gitignore
└─ scripts/
   └─ write-os-usb.sh

Requirements

Linux host with bash, sudo, and lsblk

1× USB drive (will be erased)

ISO files (the script can fetch well-known defaults)

Quick Start (USB Installer)

Make the script executable

chmod +x scripts/write-os-usb.sh


Identify your USB device (double-check!)

lsblk -o NAME,SIZE,TYPE,MOUNTPOINT | grep -E 'sd.|nvme'
# or after plugging the USB: dmesg --follow   # watch the new device name appear


Run one of these (replace /dev/sdX as needed):

# Ubuntu 24.04.1 LTS (default)
sudo scripts/write-os-usb.sh /dev/sdX ubuntu

# Arch (latest)
sudo scripts/write-os-usb.sh /dev/sdX arch

# Debian 12.6 netinst (default)
sudo scripts/write-os-usb.sh /dev/sdX debian


Tip: If your USB shows as NVMe, it may look like /dev/nvme0n1 (use the disk, not a partition like nvme0n1p1).

(Optional) Verify ISO Checksums

If you manually download ISOs:

sha256sum /path/to/distro.iso
# Compare against the official checksum from the distro’s website

Secure Reinstall with LUKS2 (Ubuntu)

These steps recreate an encrypted root and then use the Ubuntu installer to install onto it.

Boot from any Ubuntu live USB and choose “Try Ubuntu”.

Delete old OS partition and create a new target partition (or reuse an empty one):

# Example: wipe signatures on the old root partition (be sure this is the right one!)
sudo umount -R /dev/sda2 2>/dev/null || true
sudo swapoff -a 2>/dev/null || true
sudo wipefs -a /dev/sda2
sudo partprobe /dev/sda && sudo udevadm settle


Encrypt with LUKS2 and format:

# Create LUKS2 container
sudo cryptsetup luksFormat --type luks2 /dev/sda2
# Open it as "cryptroot"
sudo cryptsetup open /dev/sda2 cryptroot

# Make filesystem inside the mapper device
sudo mkfs.ext4 -L rootfs /dev/mapper/cryptroot


(U)EFI system partition
Make sure you have an EFI System Partition (ESP), e.g. /dev/sda1 ~512 MB, FAT32 with the esp/boot flag:

# If needed, (re)create and format:
# sudo mkfs.vfat -F32 /dev/sda1


Run the Ubuntu installer and choose “Something else”:

Select /dev/mapper/cryptroot → Use as: Ext4 journaling file system → Mount point: /

Ensure the EFI partition /dev/sda1 is mounted at /boot/efi (do not format it if you want to keep other boot entries)

Device for bootloader installation: the disk (e.g. /dev/sda), not a partition

Continue installation. The installer will detect and configure the LUKS setup.

After reboot, you’ll be prompted for your LUKS passphrase during early boot.

Useful Snippets

Unmount / wipe signatures:

sudo umount -R /dev/sdX* 2>/dev/null || true
sudo swapoff -a 2>/dev/null || true
sudo wipefs -a /dev/sdX


Refresh kernel partition view:

sudo partprobe /dev/sdX
sudo udevadm settle


Check what’s holding a device open:

sudo fuser -mv /dev/sdX
# or
sudo lsof | grep sdX


“Soft” re-plug a USB root port (advanced, use with care):

# Find the bus ID: ls -l /sys/bus/usb/devices | grep -E '^[0-9]-'
# Example bus ID: 1-2
echo 1-2 | sudo tee /sys/bus/usb/drivers/usb/unbind
sleep 1
echo 1-2 | sudo tee /sys/bus/usb/drivers/usb/bind

Troubleshooting

USB won’t boot: Some firmware prefers GPT + EFI. Ensure your target machine is set to UEFI mode and the ISO supports it (Ubuntu/Arch/Debian do).

Write errors: Try a different port or cable; check dmesg for I/O errors.

Device busy: Use fuser/lsof (above), then unmount/stop services before writing.

Secure boot issues: If installing proprietary drivers, you may need to toggle Secure Boot or enroll a MOK.

Notes

The script uses standard Linux tools and will erase the target device. Read it before running.

LUKS2 defaults are sensible; you can tune --cipher, --hash, --pbkdf if you know what you’re doing.

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

