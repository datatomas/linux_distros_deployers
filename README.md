Linux Distro Deployer — README
Generated on 2025-10-25 03:04
Overview
Tiny, no‑nonsense scripts for writing bootable USB installers (Ubuntu, Arch, Debian) and a secure reinstall flow using LUKS2 encryption. Includes lessons learned for networking on Gigabyte AORUS X870 AORUS ELITE WIFI7 ICE motherboards and general Linux install pitfalls.
⚠️ Danger: These commands overwrite disks. Triple‑check device names (e.g., /dev/sda, /dev/nvme0n1) before running. You are responsible for your data.
Repository Layout
linux-distro-deployer/
├─ README.md
├─ .gitignore
└─ scripts/
   └─ write-os-usb.sh
Requirements
    • Linux host with bash, sudo, wget, lsblk, parted, wipefs
    • Optional: lsof, cryptsetup, nvme-cli, ddrescue
    • 1× USB drive (WILL BE ERASED)
    • ISO files (script can fetch well‑known defaults)
Install Tools Quickly
# Debian/Ubuntu
sudo apt-get update && sudo apt-get install -y wget parted util-linux lsof cryptsetup ddrescue

# Arch
sudo pacman -S --needed wget parted lsof cryptsetup gnu-ddrescue
Quick Start (USB Installer)
    • Make the script executable: chmod +x scripts/write-os-usb.sh
    • Identify your USB device (WHOLE disk): lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
    • Run one of the presets (replace /dev/sdX or /dev/nvme0n1):
# Ubuntu 24.04.1 LTS (default)
sudo scripts/write-os-usb.sh /dev/sdX ubuntu

# Arch (latest)
sudo scripts/write-os-usb.sh /dev/sdX arch

# Debian 12 (netinst, default)
sudo scripts/write-os-usb.sh /dev/sdX debian
Tip: If your USB shows as NVMe, it may look like /dev/nvme0n1. Always use the disk, not a partition like nvme0n1p1.
Optional: Verify ISO Checksums
sha256sum /path/to/distro.iso
# Compare against the official checksum from the distro’s website.
Secure Reinstall with LUKS2 (Ubuntu)
These steps recreate an encrypted root and then use the Ubuntu installer. Boot a live USB and choose “Try Ubuntu”.
# 1) Delete old OS partition and wipe signatures (EXAMPLE uses /dev/sda2 — verify yours)
sudo umount -R /dev/sda2 2>/dev/null || true
sudo swapoff -a 2>/dev/null || true
sudo wipefs -a /dev/sda2
sudo partprobe /dev/sda && sudo udevadm settle

# 2) Create LUKS2 container and open it
sudo cryptsetup luksFormat --type luks2 /dev/sda2
sudo cryptsetup open /dev/sda2 cryptroot

# 3) Filesystem inside mapper
sudo mkfs.ext4 -L rootfs /dev/mapper/cryptroot

# 4) Ensure an EFI System Partition exists (e.g., /dev/sda1, ~512MiB, FAT32)
# If needed:
sudo mkfs.vfat -F32 /dev/sda1
Run the Ubuntu installer → “Something else”: mount /dev/mapper/cryptroot at '/', and mount the EFI partition (/dev/sda1) at /boot/efi (do not format EFI if you keep other boot entries). Choose the disk (e.g. /dev/sda) for the bootloader.
Disk Management Snippets
# Show current mounts & holders
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT,UUID
sudo fuser -mv /dev/sdX

# Unmount / wipe signatures
sudo umount -R /dev/sdX* 2>/dev/null || true
sudo swapoff -a 2>/dev/null || true
sudo wipefs -a /dev/sdX

# Refresh kernel view of partitions
sudo partprobe /dev/sdX
sudo udevadm settle

# Write an ISO to a USB (manual, when not using the script)
sudo dd if=~/Downloads/distro.iso of=/dev/sdX bs=4M status=progress oflag=sync
sync
Fast Path: Extend a Partition to the Right (No Move)
# Example: extend /dev/nvme0n1p4 to the end (does NOT move the start)
findmnt /dev/nvme0n1p4 && sudo umount /dev/nvme0n1p4
sudo parted /dev/nvme0n1 ---pretend-input-tty <<'EOF'
unit GiB
print
resizepart 4 100%
Yes
print
quit
EOF
sudo partprobe /dev/nvme0n1 && sudo udevadm settle
sudo e2fsck -f /dev/nvme0n1p4
sudo resize2fs /dev/nvme0n1p4
Left-Move (Start Earlier) — CLI Method
To make a partition start earlier (e.g., move /dev/nvme0n1p4 from 430 GiB to 250 GiB), you must move data. The reliable CLI approach is backup → recreate → restore:
DISK=/dev/nvme0n1
OLD=$DISKp4
BACKUP=/mnt/backup
MNT_OLD=/mnt/old
MNT_NEW=/mnt/new

# Backup (file-level)
sudo e2fsck -f $OLD
sudo mkdir -p $MNT_OLD $BACKUP
sudo mount -o ro $OLD $MNT_OLD
sudo rsync -aHAX --numeric-ids --info=progress2 $MNT_OLD/ $BACKUP/data/
sudo umount $MNT_OLD

# Recreate from 250 GiB → 100%
sudo parted -s $DISK rm 4
sudo parted -s $DISK unit GiB mkpart primary ext4 250 100%
sudo parted -s $DISK name 4 data
sudo partprobe $DISK

# Restore
sudo mkfs.ext4 -L data ${DISK}p4
sudo mkdir -p $MNT_NEW
sudo mount ${DISK}p4 $MNT_NEW
sudo rsync -aHAX --numeric-ids --info=progress2 $BACKUP/data/ $MNT_NEW/
sudo umount $MNT_NEW
sudo e2fsck -f ${DISK}p4
sudo resize2fs ${DISK}p4
Lessons Learned: Networking on Gigabyte AORUS X870 AORUS ELITE WIFI7 ICE
On some X870/AORUS ELITE WIFI7 ICE boards, Ethernet and Wi‑Fi may not work out‑of‑the‑box on older installers. Here are fixes that consistently bring networking up on Ubuntu 24.04/Arch (kernel 6.6+):
    • Prefer a current kernel + linux-firmware. On Ubuntu live media, run: sudo apt-get update && sudo apt-get install -y linux-firmware
    • Ethernet (2.5GbE, often Realtek RTL8125B/8125BG): if link is down or flaps with driver r8169, try r8168:
# Ubuntu/Debian:
sudo apt-get install -y r8168-dkms
sudo modprobe -r r8169 || true
sudo modprobe r8168

# Arch:
sudo pacman -S --needed r8168
sudo modprobe -r r8169 || true
sudo modprobe r8168
    • Wi‑Fi 7 module (often Intel BE200): requires recent iwlwifi firmware. Ensure package 'linux-firmware' is updated. On Arch: sudo pacman -Syu linux-firmware. On Ubuntu: sudo apt-get install --reinstall linux-firmware.
    • Check devices and bound drivers: lspci -nnk | grep -A3 -E 'Ethernet|Network'
    • Quick network bring‑up:
# Show NICs and state
ip -br a
# Bring Ethernet up (replace enpXsY)
sudo ip link set enp7s0 up
sudo dhclient -v enp7s0 || sudo NetworkManager

# NetworkManager CLI examples
nmcli device status
nmcli dev connect enp7s0
nmcli dev wifi rescan
nmcli dev wifi list
If the live installer lacks firmware and you cannot get online, copy /lib/firmware from a newer distro to a USB, then place it under /lib/firmware on the live session (temporary) and re‑load the module (e.g., modprobe r8169 or r8168, or modprobe iwlwifi).
Troubleshooting
    • USB won’t boot: use UEFI mode; most ISOs are hybrid GPT/EFI.
    • Write errors: try a different port/cable; check dmesg for I/O issues.
    • Device busy: use fuser/lsof, then unmount/stop services.
    • Secure Boot: if using proprietary drivers, disable Secure Boot or enroll a MOK.
# Who holds the device?
sudo fuser -mv /dev/sdX
sudo lsof | grep sdX

# Soft re-plug a USB root port (advanced)
echo 1-2 | sudo tee /sys/bus/usb/drivers/usb/unbind
sleep 1
echo 1-2 | sudo tee /sys/bus/usb/drivers/usb/bind
Appendix: Verify & Mount Data Volume
# After resizing:
sudo parted /dev/nvme0n1 print free
sudo e2fsck -f /dev/nvme0n1p4
sudo resize2fs /dev/nvme0n1p4

# Mount
sudo mkdir -p /data
sudo mount /dev/nvme0n1p4 /data
df -hT /data
# Optional: reduce ext4 reserved space on data volume
sudo tune2fs -m 1 /dev/nvme0n1p4
Arch/Linux philosophy: Keep it simple (but not simplistic). Understand what each command does before you run it.
