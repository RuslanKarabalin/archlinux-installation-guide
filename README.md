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
mkdir -p /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
```

### Setup pacman on live USB

```bash
nano /etc/pacman.conf
```

> uncomment
>
> ```plaintext
> Color
> ParallelDownloads = 5
> ```

### vconsole

In some cases, need to

```bash
nano /mnt/etc/vconsole.conf
```

> write
>
> ```plaintext
> # KEYMAP=us
> ```

### Install packages

```bash
pacstrap -K /mnt base \
    base-devel linux \
    linux-headers \
    linux-firmware \
    amd-ucode \
    grub efibootmgr \
    zram-generator \
    networkmanager \
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
ln -sf /usr/share/zoneinfo/<REGION>/<CITY> /etc/localtime
hwclock --systohc
timedatectl set-ntp true
timedatectl status
```

### Localization setup

```bash
nvim /etc/locale.gen
```

> uncomment
>
> ```plaintext
> en_US.UTF-8 UTF-8
> ...
> ru_RU.UTF-8 UTF-8
> ```

```bash
locale-gen
nvim /etc/locale.conf
```

> write
>
> ```plaintext
> LANG=en_US.UTF-8
> ```

### Network configuration

```bash
nvim /etc/hostname
```

> write
>
> ```plaintext
> <HOSTNAME>
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
nvim /etc/default/grub
```

> change
>
> ```plaintext
> GRUB_TIMEOUT=0
> ```

```bash
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

Login as `root`

### Connect to internet

```bash
nmtui
```

### Setup user

```bash
useradd -m -g users -G wheel -s /bin/zsh <USERNAME>
passwd <USERNAME>
EDITOR=nvim visudo
```

> uncomment
>
> ```plaintext
> %wheel ALL=(ALL:ALL) ALL
> ```

### Setup pacman in installed system

```bash
nvim /etc/pacman.conf
```

> uncomment
>
> ```plaintext
> Color
> ParallelDownloads = 5
> ...
> [multilib]
> Include = ...
> ```

```bash
pacman -Syu
```

### Install base packages

```bash
pacman -S bluez bluez-utils \
    gvfs gvfs-mtp \
    gvfs-gphoto2 \
    pipewire wireplumber \
    pipewire-audio pipewire-alsa \
    pipewire-pulse pipewire-jack \
    ttf-jetbrains-mono \
    noto-fonts noto-fonts-cjk \
    noto-fonts-emoji \
    noto-fonts-extra \
    otf-font-awesome \
    alacritty tmux git eza \
    bat htop tree ripgrep \
    unzip unrar openssh \
    systemd-resolvconf \
    wireguard-tools
```

### Enable zram

```bash
nvim /etc/systemd/zram-generator.conf
```

> write
>
> literally "ram / 2", not 8G
>
> ```plaintext
> [zram0]
> zram-size = ram / 2
> compression-algorithm = zstd
> ```

```bash
systemctl daemon-reload
systemctl enable --now systemd-zram-setup@zram0.service
zramctl
```

### Enable base services

#### resolvconf

```bash
systemctl enable systemd-resolved
```

#### bluetooth

```bash
systemctl enable bluetooth
```

#### fstrim

```bash
systemctl enable fstrim.timer
fstrim -v /
```

### Desktop Environment

#### Hyprland

```bash
pacman -S hyprland \
    hyprlock hypridle \
    waybar wofi mako \
    xdg-desktop-portal-hyprland \
    polkit-kde-agent \
    archlinux-xdg-menu xdg-user-dirs \
    network-manager-applet \
    brightnessctl \
    blueman wl-clipboard cliphist \
    grim slurp swappy \
    hyprpaper \
    pavucontrol playerctl \
    imv mpv \
    zathura zathura-pdf-mupdf \
    firefox \
    thunar thunar-volman \
    thunar-archive-plugin \
    sddm \
    qt5-wayland qt6-wayland \
    kwallet dolphin ark \
    gtk3 gtk4 \
    papirus-icon-theme
```

##### Enable SDDM

```bash
systemctl enable sddm
```

#### GNOME

```bash
pacman -S gdm gnome-shell \
    nautilus evince \
    file-roller gnome-keyring \
    gnome-control-center \
    gnome-tweaks \
    gnome-themes-extra \
    papirus-icon-theme \
    gnome-shell-extensions \
    gnome-shell-extension-appindicator \
    xdg-desktop-portal \
    xdg-desktop-portal-gnome
```

##### Enable GDM

```bash
systemctl enable gdm
```

## Extra

### yay

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

### docker

```bash
sudo pacman -S docker \
    docker-compose \
    lazydocker
```

```bash
sudo systemctl enable docker.socket
```

```bash
sudo usermod -aG docker <USERNAME>
```

```bash
newgrp docker
docker run hello-world
```

### nvidia

```bash
sudo pacman -S nvidia-dkms \
    nvidia-utils \
    nvidia-prime \
    nvidia-settings
```

```bash
sudo nvim /etc/default/grub
```

> write
>
> ```plaintext
> GRUB_CMDLINE_LINUX_DEFAULT="... nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
> ```

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

```bash
sudo nvim /etc/mkinitcpio.conf
```

> write
>
> ```plaintext
> MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
> remove kms from HOOKS=()
> ```

```bash
sudo mkinitcpio -P
```

```bash
sudo mkdir -p /etc/pacman.d/hooks/
sudo nvim /etc/pacman.d/hooks/nvidia.hook
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
sudo reboot now
```
