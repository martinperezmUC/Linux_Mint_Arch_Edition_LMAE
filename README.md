# Linux_Mint_Arch_Edition_LMAE
Install Arch Linux with Cinnamon DE and Mint shell style.

# Installation Guide: Arch Linux + BTRFS + Cinnamon

This is my personal guide to install **Arch Linux** from scratch on a UEFI system, using the compression-optimized **BTRFS** file system, the `systemd-boot` bootloader, and the **Cinnamon** desktop environment with a selection of essential applications. Follow at your own risk. The selected microcode and GPU drivers are compatible with AMD processors and graphics cards. Check the Arch Wiki for alternative packages for Intel and/or NVIDIA.

The `.bashrc` file included in the repository applies Mint's styles to the GNOME terminal. I recommend enabling the Tango color scheme (`gnome-terminal --preferences`).

---

## Phase 1: Pre-installation and Network Environment

Initial keyboard setup, internet connection, and time synchronization.

#### Set the keyboard to Spanish
`loadkeys es`

#### Connect to the Wi-Fi network
`iwctl`
>
`station wlan0 connect <SSID> `

#### Check connection
`ping -c 5 ping.archlinux.org`

#### Update the system clock
`timedatectl`

---

## Phase 2: BTRFS Partitioning and Subvolumes

### Suggested Partition Scheme (/dev/nvme0n1)
| Partition | Size | Filesystem type | Mounting point |
| :--- | :--- | :--- | :--- |
| /dev/nvme0n1p1 | 2 GiB | FAT32 (EFI System) | /boot |
| /dev/nvme0n1p2 | All remaining space | BTRFS | / (@, @home) |

Run `fdisk -l` to identify your disk, and then run `fdisk /dev/sdX#` to create the GPT partition table and the partitions.

### Formatting and Structure of Subvolumes

#### Format partitions
`mkfs.fat -F 32 /dev/nvme0n1p1`
>
`mkfs.btrfs /dev/nvme0n1p2`

#### Create BTRFS subvolumes for Timeshift support and organization
`mount /dev/nvme0n1p2 /mnt`
>
`btrfs subvolume create /mnt/@`
>
`btrfs subvolume create /mnt/@home`
>
`umount /mnt`

#### Build with ZSTD compression optimizations
`mount -o compress=zstd:3,subvol=@ /dev/nvme0n1p2 /mnt`
>
`mkdir -p /mnt/home`
>
`mount -o compress=zstd:3,subvol=@home /dev/nvme0n1p2 /mnt/home`

#### Mount EFI partition
`mkdir -p /mnt/boot`
>
`mount /dev/nvme0n1p1 /mnt/boot`

---

## Phase 3: Installing the Base System (Pacstrap)

Installation of system packages, audio tools (Pipewire), network management, microcode updates (AMD), and utilities.

`pacstrap -K /mnt base base-devel linux linux-firmware git btrfs-progs
timeshift nano networkmanager pipewire pipewire-alsa pipewire-pulse
pipewire-jack wireplumber reflector man sudo amd-ucode`

#### Generate FSTAB
`genfstab -U /mnt >> /mnt/etc/fstab`

---

## Phase 4: System Configuration (Chroot)

We log into the new system to configure the locale, users, and permissions.

`arch-chroot /mnt`

#### Time zone and clock
`ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime`
>
`hwclock --systohc`

#### Localization (Uncomment "es_ES.UTF-8 UTF-8" in the file)
`nano /etc/locale.gen`
>
`locale-gen`

#### Language and keyboard environment variables
`echo "LANG=es_ES.UTF-8" > /etc/locale.conf`
>
`echo "KEYMAP=es" > /etc/vconsole.conf`

#### Network (Hostname)
`echo "arch-pc" > /etc/hostname`
>
`echo "127.0.0.1   arch-pc" >> /etc/hosts`

#### User Settings
* Root password: `passwd`
>
* Create daily user: `useradd -mG wheel <new_user>`    
>
* New user password: `passwd <new_user>`               

#### SUDO privileges (Uncomment the line: %wheel ALL=(ALL:ALL) ALL)
`EDITOR=nano visudo`

---

## Step 5: Boot Manager (systemd-boot)

Installing the native systemd bootloader.

`bootctl install`

Edit the /boot/loader/loader.conf file:

```
default      arch.conf
timeout      5
console-mode max
editor       no
```

> [!TIP]
> Run `blkid` in another terminal or note down the UUID of the BTRFS partition (/dev/nvme0n1p2) for the next step.

Create the /boot/loader/entries/arch.conf configuration file:
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rootflags=subvol=@ rw
```

#### Generate initramfs and start essential services
`mkinitcpio -P`
>
`systemctl enable NetworkManager`

#### Exit and reboot
`exit`
>
`umount -R /mnt`
>
`reboot`

---

## Step 6: Post-Installation (Drivers and Paru AUR Helper)

#### Connect to Wi-Fi and sync the time
`nmcli d wifi connect <SSID> --ask`
>
`timedatectl set-ntp true`

#### Install PARU (AUR Helper)
`git clone https://aur.archlinux.org/paru.git`
>
`cd paru`
>
`makepkg -si`
>
`cd .. && rm -rf paru`

#### Configure the X11 keymap and enable 32-bit repositories
`sudo localectl set-x11-keymap es`

> [!IMPORTANT]
> Edit /etc/pacman.conf and uncomment [multilib] and the line below to support 32-bit applications (Steam, etc.). Then update: sudo pacman -Syu

#### Drivers GPU (AMD / Mesa)
`sudo pacman -S mesa vulkan-radeon libva-mesa-driver`
>
`sudo pacman -S lib32-mesa lib32-vulkan-radeon lib32-libva-mesa-driver`

---

## Step 7: Cinnamon Desktop Environment

Installation of the graphics server, the LightDM session manager, and the core Cinnamon tools.

#### Graphical server, LightDM, and Cinnamon base

`sudo pacman -S xorg-server lightdm lightdm-slick-greeter cinnamon xed
xviewer xreader system-config-printer blueman gnome-terminal
xdg-user-dirs xdg-desktop-portal-xapp xdg-desktop-portal-gtk xdg-desktop-portal`

#### Force user directories to Spanish
`LC_ALL=es_ES.UTF-8 xdg-user-dirs-update --force `

#### LightDM Login Configuration
Edit `/etc/lightdm/lightdm.conf`. Look for the `seat:*]` section and uncomment and modify the following line:
`greeter-session=lightdm-slick-greeter`

#### Enable services and add visual elements typical from Linux Mint / Audio
`systemctl enable lightdm`
>
`paru -S mint-artwork` (this may take a while to compile and install)
>
`sudo pacman -S ttf-dejavu ttf-liberation rtkit cups`
>
`systemctl enable bluetooth.service`
>
`systemctl enable cups`
>
`reboot`

---

## ⚠️ Reported BUG: xdg-desktop-portal (1.22.0-1)

> [!WARNING]
> There is a reported bug that prevents xdg-desktop-portal from starting correctly, which breaks dark mode detection in GTK (https://gitlab.archlinux.org/archlinux/packaging/packages/xdg-desktop-portal/-/work_items/4). Until an official fix is released, apply the following patch:

#### Modify the Systemd service
Edit `/usr/lib/systemd/user/xdg-desktop-portal.service`:
```diff
 [Unit]
 Description=Portal service
 PartOf=graphical-session.target
-Requisite=graphical-session.target
+Requires=dbus.service
+After=dbus.service
 After=graphical-session.target
```

---

## Phase 8: Software and Applications

#### LMDE Included Applications, emojis and oriental fonts (Linux Mint Debian Edition)
`sudo pacman -S fastfetch baobab gnome-calculator gnome-calendar firefox
noto-fonts noto-fonts-emoji noto-fonts-cjk ufw gufw drawing
gnome-disk-utility gnome-power-manager file-roller simple-scan
gnome-system-monitor gnome-screenshot seahorse nemo-fileroller ffmpegthumbnailer`

#### Enable Firewall (don't forget to activate it later)
`sudo systemctl enable --now ufw`

#### Other apps and frameworks of interest
`sudo pacman -S jdk-openjdk btop lact libreoffice-fresh lutris vlc vlc-plugins-all steam flatpak`
>
`paru -S visual-studio-code-bin`
>
`flatpak install flatseal`
