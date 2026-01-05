# archlinux-installation-guide

## Installation

### Connect to internet with iwctl

```bash
iwctl
station wlan0 scan
station wlan0 connect <WIFIID>
station wlan0 show
exit
```

### Update the system clock

```bash
timedatectl
```

### Partition the disks

```bash
cfdisk -z /dev/nvme0n1
```
> choose GPT
>
> ```plaintext
> 1G - EFI System
> <rest>G - Linux filesystem
> ```

```bash
mkfs.fat -F 32 -n GRUB /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
mount /dev/nvme0n1p2 /mnt
```

### Setup pacman ParallelDownloads

```bash
nano /etc/pacman.conf
```

> uncomment
>
> ```plaintext
> Color
> ParallelDownloads = 4
> ```

### vconsole

In some cases, need to

```bash
nano /etc/vconsole.conf
```

> write
>
> ```plaintext
> # KEYMAP=us
> ```

### Install packages

```bash
pacstrap -K /mnt base base-devel \
    linux linux-headers linux-firmware \
    amd-ucode grub efibootmgr \
    zram-generator networkmanager \
    sudo neovim zsh
```

> fstab
>
> ```bash
> genfstab -U /mnt >> /mnt/etc/fstab
> ```


> chroot
>
> ```bash
> arch-chroot /mnt
> ```

### Timezone setup

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

### Localization setup

```bash
vim /etc/locale.gen
```

> uncomment
>
> ```plaintext
> en_US.UTF-8 UTF-8
> ru_RU.UTF-8 UTF-8
> ```

```bash
locale-gen
vim /etc/locale.conf
```

> write
>
> ```plaintext
> LANG=en_US.UTF-8
> ```

### Network configuration

```bash
vim /etc/hostname
```

> write
>
> ```plaintext
> {HOSTNAME}
> ```

```bash
systemctl enable NetworkManager
```

> initramfs
>
> ```bash
> mkinitcpio -P
> ```

### Setup root password

```bash
passwd
```

### Setup grub

```bash
vim /etc/default/grub
```

> change
>
> ```plaintext
> GRUB_TIMEOUT=0
> ```

```bash
mkdir /boot/efi
mount /dev/nvme0n1p1 /boot/efi
lsblk
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

### Exit

```bash
exit
umount -R /mnt
shutdown now
```

## After installation

### Connect to internet

```bash
nmtui
```

### Setup user

```bash
useradd -m -g users -G wheel -s /bin/zsh {USERNAME}
passwd {USERNAME}
EDITOR=vim visudo
```

> uncomment
>
> ```plaintext
> %wheel ALL=...
> %sudo ALL=...
> ```

### Setup pacman

```bash
vim /etc/pacman.conf
```

> uncomment
>
> ```plaintext
> [multilib]
> Include = ...
> ```

```bash
pacman -Syu
```

### Install base packages

```bash
pacman -S bluez bluez-utils \
    gvfs gvfs-mtp gvfs-gphoto2 \
    pipewire wireplumber pipewire-audio \
    pipewire-alsa pipewire-pulse pipewire-jack \
    nvidia nvidia-utils nvidia-prime nvidia-settings \
    ttf-jetbrains-mono noto-fonts noto-fonts-cjk \
    noto-fonts-emoji noto-fonts-extra oft-font-awesome \
    alacritty git eza bat tmux htop tree unzip unrar ripgrep openssh \
    systemd-resolvconf wireguard-tools
```

### Enable zram

```bash
vim /etc/systemd/zram-generator.conf
```

> write
>
> litteraly "ram / 2", not 8G
>
> ```plaintext
> [zram0]
> zram-size = ram / 2
> compression-algorithm = zstd
> ```

```bash
systemctl daemon-reload
systemctl start systemd-zram-setup@zram0.service
zramctl
```

### Enable base services

#### bluetooth

```bash
systemctl enable bluetooth
```

#### fstrim

```bash
systemctl enable fstrim
fstrim -v /
mkinitcpio -P
```

#### nvidia

```bash
vim /etc/default/grub
```

> write
>
> ```plaintext
> GRUB_CMDLINE_LINUX_DEFAULT="... nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
> ```

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

```bash
vim /etc/mkinitcpio.conf
```

> write
>
> ```plaintext
> MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
> remove kms from HOOKS=()
> ```

```bash
mkinitcpio -P
```

```bash
vim /etc/pacman.d/hooks/nvidia.hook
```

> write
>
> ```plaintext
> [Trigger]
> Operation=Install
> Operation=Upgrade
> Operation=Remove
> Type=Package
> Target=nvidia
> Target=linux
> 
> [Action]
> Depends=mkinitcpio
> When=PostTransaction
> NeedsTargets
> Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
> ```

```bash
systemctl reboot
```

### Hyprland

```bash
pacman -S hyprland hyprlock waybar wofi mako \
    xdg-desktop-portal-hyprland archlinux-xdg-menu \
    network-manager-applet brightnessctl blueman \
    wl-clipboard cliphist grim slurp \
    sddm qt5-wayland qt6-wayland kwallet dolphin ark
```

#### Enable SDDM

```bash
systemctl enable sddm
```

### GNOME

```bash
pacman -S gnome-shell nautilus evince file-roller gnome-keyring \
    gnome-control-center gdm gnome-tweaks gnome-shell-extensions gnome-themes-extra \
    gnome-shell-extension-appindicator papirus-icon-theme xdg-desktop-portal \
    xdg-desktop-portal-gnome
```

#### Enable GDM

```bash
systemctl enable gdm
```
