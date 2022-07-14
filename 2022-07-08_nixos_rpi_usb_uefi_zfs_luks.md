# Steps

With lots help from

* <https://mgdm.net/weblog/nixos-on-raspberry-pi-4/>
* <https://carjorvaz.com/posts/nixos-on-raspberry-pi-4-with-uefi-and-zfs/>

Get nixos generic image (<https://hydra.nixos.org/job/nixos/trunk-combined/nixos.sd_image.aarch64-linux>) and burn it do micro sd card, boot into nixos.

## Add nixos-hardware channel

```
sudo nix-channel --add https://github.com/NixOS/nixos-hardware/archive/master.tar.gz nixos-hardware
sudo nix-channel --update
```

cryptsetup

## Do partitioning

```
wipefs -a /dev/sda

parted -a optimal /dev/sda -- mklabel gpt
parted -a optimal /dev/sda -- mkpart ESP fat32 1MiB 513MiB
parted -a optimal /dev/sda -- set 1 esp on
parted -a optimal /dev/sda -- mkpart primary linux-swap 513MiB 8705MiB
parted -a optimal /dev/sda -- mkpart primary 8705MiB 100%

mkfs.fat -F 32 -n boot /dev/sda1

mkswap -L swap /dev/sda2
swapon /dev/sda2
```

### Cryptsetup

```
cryptsetup luksFormat /dev/sda3
cryptsetup luksOpen /dev/sda3 crypt-root
```

### ZFS

```
zpool create -O mountpoint=none -O atime=off -o ashift=12 -O acltype=posixacl -O xattr=sa -O compression=zstd -O dnodesize=auto -O normalization=formD zroot /dev/mapper/crypt-root
zfs create -o refreservation=1G -o mountpoint=none zroot/reserved
zfs create zroot/root

mount -t zfs -o zfsutil zroot/root /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

## UEFI

```
nix-shell -p wget unzip
cd /mnt/boot
wget https://github.com/pftf/RPi4/releases/download/v1.33/RPi4_UEFI_Firmware_v1.33.zip
unzip RPi4_UEFI_Firmware_v1.33.zip
rm README.md
rm RPi4_UEFI_Firmware_v1.33.zip
```

## Configuration

### Generate config

```
nixos-generate-config --root /mnt
```

### Add necessary statements

```
boot.supportedFilesystems = [ "zfs" ];
networking.hostId = "<8 random numbers>";
```

### Use latest kernel

```
boot.kernelPackages = pkgs.linuxPackages_latest;
```

### Add nixos-hardware for raspberry pi to imports

```
imports =
  [
    <nixos-hardware/raspberry-pi/4>
    ./hardware-configuration.nix
  ];
```

### Install nixos

```
nixos-install --root /mnt
```

### Add wheel to trusted users

```
nix.settings."trusted-users" = [ "@wheel" ];
```

Done