---
weight: 4
title: "Installation d'Arch Linux"
date: 2020-11-01T00:00:00+00:00
draft: false
author: "Nyxott"
description: "Installer Arch Linux"
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Miscellaneous"]
categories: ["Miscellaneous"]

lightgallery: true
---

Cet article explique comment installer Arch Linux.

<!--more-->

##  Installation

### Prérequis

On utilise une image ISO disponible à l'adresse suivante : [http://mir.archlinux.fr/iso/latest/](http://mir.archlinux.fr/iso/latest/) (archlinux-YY.MM.DD-x86_64.iso)

On crée une clé USB bootable avec cette image ISO :
```shell
sudo fdisk -l (pour repérer la clé USB)
sudo dd if=/chemin/du/fichier/archlinux.iso of=/dev/sd<X> bs=1M status=progress oflag=sync ("<X>" correspond à la lettre attribuée au volume de la clé USB)
```

### Disposition du clavier

Une fois le boot effectué sur l'image ISO, nous obtenons un shell en tant que root.

Nous modifions le clavier afin qu'il soit en azerty :
```shell
loadkeys fr-pc (attention le clavier est en "qwerty" il faut donc taper "loqdkeys fr)pc")
```

### Connectivité du système

Avant de poursuivre l'installation nous devons vérifier si nous avons Internet, car l'installation va nécessiter de télécharger certaines choses :
```shell
ping -c 4 1.1.1.1
```

### Mise à jour de l'heure système

Afin que pacman puisse vérifier la validité des paquets téléchargés, il est nécessaire que l'heure soit correcte.

Nous allons donc préciser notre timezone :
```shell
timedatectl set-timezone Europe/Paris
timedatectl set-ntp true
```

### Partitionnement du disque

La méthode de partitionnement dépendra de si l'ordinateur utilise UEFI (GPT) ou BIOS (MBR). Ici, je ne montre que l'UEFI (GPT).

On crée la GPT :
```shell
parted /dev/sda -- mklabel gpt
```

On crée la partition EFI :
```shell
parted /dev/sda -- mkpart primary fat32 1MiB 101MiB
parted /dev/sda -- set 1 esp on
```

On crée la partition boot :
```shell
parted /dev/sda -- mkpart primary ext4 101MiB 361MiB
parted /dev/sda -- set 2 boot on
```

On définit la partition principale que nous allons chiffrer par la suite :
```shell
parted /dev/sda -- mkpart primary btrfs 361MiB 100%
```

### Chiffrement de la partition principale

On chiffre la partition principale :
```shell
cryptsetup -i 5000 -y luksFormat /dev/sda3
cryptsetup open /dev/sda3 archlinux-root
```

### Gestion par volumes logiques

On découpe notre partition principale en 2 sous-partitions.

Ici, "8G" corresponds à la taille que l'on souhaitera réserver à la swap (à adapter en fonction de la RAM) :
```shell
pvcreate /dev/mapper/archlinux-root
vgcreate VG_CRYPT /dev/mapper/archlinux-root
lvcreate --size 8G VG_CRYPT --name lv_swap
lvcreate -l +100%FREE VG_CRYPT --name lv_root
```

### Formatage des partitions

On formate la partition EFI :
```shell
mkfs.fat -n EFI -F32 /dev/sda1
```

On formate la partition boot :
```shell
mkfs.ext4 -L BOOT /dev/sda2
```

On formate la swap :
```shell
mkswap -L SWAP /dev/mapper/VG_CRYPT-lv_swap
```

On formate la partition principale :
```shell
mkfs.btrfs -L ARCHLINUXROOT /dev/mapper/VG_CRYPT-lv_root
```

### Montage des partitions

On monte la partition principale :
```shell
mount /dev/mapper/VG_CRYPT-lv_root /mnt
```

On monte la partition boot :
```shell
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
```

On monte la partition EFI :
```shell
mkdir /mnt/boot/EFI
mount /dev/sda1 /mnt/boot/EFI
```

On active la swap :
```shell
swapon /dev/mapper/VG_CRYPT-lv_swap
```

### Sélection des miroirs

On sélectionne les miroirs qui ont les meilleurs délais de réponses en fonction de notre zone géographique :
```shell
pacman -Sy
pacman -S reflector
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
reflector -c "FR" -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist
```

### Installation

On installe les paquets de base :
```shell
pacstrap /mnt base base-devel btrfs-progs linux linux-firmware lvm2 man-db man-pages networkmanager vim
```

### Génération du fichier fstab

Nous générons le fichier indiquant les points de montage de nos paritions :
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### Optimisation SSD

Pour les ordinateurs possédant un disque SSD, il est possible d'obtenir de meilleures performances en éditant le fichier `/mnt/etc/fstab` pour modifier le `relatime` en `noatime` sur la ligne concernant la partition `/dev/mapper/VG_CRYPT-lv_root LABEL=ARCHLINUXROOT`.

### arch-chroot

Nous entrons à présent dans le Arch Linux installé sur notre disque afin de finir quelques réglages : 
```shell
arch-chroot /mnt
```

### Paramètres réseaux

On ajoute un nom d'hôte à notre PC :
```shell
echo ArchLinux > /etc/hostname
```

On crée le fichier `/etc/hosts` :
```shell
touch /etc/hosts
```

On édite ce fichier afin d'y ajouter les lignes suivantes (`ArchLinux` corresponds au nom d'hôte attribué plus haut) :
```shell
127.0.0.1    localhost
::1          localhost
127.0.1.1    ArchLinux.localdomain ArchLinux
```

On active NetworkManager au démarrage :
```shell
systemctl enable NetworkManager.service
```

### Configuration de l'horloge

On configure l'horloge du système afin d'éviter des problèmes liés au protocol NTP lors des vérifications de paquets :
```shell
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

### Paramètres locaux
shell
Nous définissons le système en anglais :
```
sed -i "s/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g" /etc/locale.gen
locale-gen
echo LANG="en_US.UTF-8" > /etc/locale.conf
export LANG=en_US.UTF-8
```

Nous définissons le clavier en azerty :
```shell
echo KEYMAP=fr > /etc/vconsole.conf
```

### Mot de passe root

On définit un mot de passe pour l'utilisateur root :
```shell
passwd
```

### Bootloader

On installe GRUB ainsi qu'un paquet qui permettra de gérer l'UEFI :
```shell
pacman -S efibootmgr grub
```

On cherche l'UUID de `/dev/sda3` :
```shell
lsblk -f | grep "sda3"
```

On édite le fichier `/etc/default/grub` afin de remplacer la ligne `GRUB_CMDLINE_LINUX=""` par la ligne suivante (`<UUID>` doit être remplacé par l'UUID de votre partition `sda3`) :
```shell
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<UUID>:archlinux-root:allow-discards
```

On cherche l'UUID de la swap :
```shell
lsblk -f | grep "SWAP"
```

Toujours dans le fichier `/etc/default/grub` nous remplaçons la ligne `GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"` par la ligne suivante (`<UUID>` doit être remplacé par l'UUID de votre SWAP) :
```shell
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet resume=UUID=<UUID>"
```

Toujours dans le fichier `/etc/default/grub` nous décommentons la ligne :
```shell
"GRUB_ENABLE_CRYPTODISK=y"
```

On édite le fichier `/etc/mkinitcpio.conf` afin de remplacer la ligne `HOOKS=(base udev autodetect mdconf block filesystems keyboard fsck)` par la ligne suivante :
```shell
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 btrfs resume filesystems fsck)
```

On génère notre nouvelle image :
```shell
mkinitcpio -p linux
```

On installe GRUB :
```shell
grub-install --target=x86_64-efi --boot-directory=/boot --efi-directory=/boot/EFI --bootloader-id=GRUB --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

### Redémarrage

On sort du arch-chroot et on éteint la machine :
```shell
exit
umount /mnt/boot/EFI
umount /mnt/boot
umount /mnt
swapoff /dev/mapper/VG_CRYPT-lv_swap
cryptsetup close /dev/mapper/VG_CRYPT-lv_swap
cryptsetup close /dev/mapper/VG_CRYPT-lv_root
cryptsetup close /dev/mapper/archlinux-root
poweroff
```

Puis l'on redémarre la machine en prenant soin, au préalable, de retirer la clef USB contenant l'ISO.

## Post-installation

### Mise à jour

On commence avant tout par mettre à jour le système et la base de données de pacman :
```shell
pacman -Syu
```

### Utilisateur

On ajoute un nouvel utilisateur (ici, il faut remplacer `nyxott` par votre utilisateur) :
```shell
useradd -m -G users,wheel nyxott 
passwd nyxott
```

### Sudo

On modifie le fichier `/etc/sudoers` pour sauvegarder durant une heure le mot de passe à chaque fois qu'on le tape :
```shell
pacman -S vi
visudo
```

On ajoute la ligne suivante :
```shell
Defaults	timestamp_timeout=60
```

Puis, l'on décommente la ligne suivante :
```shell
%wheel ALL=(ALL) ALL
```

### Interface graphique

On installe les composants nécessaires pour notre interface graphique :
```shell
pacman -S i3-gaps lightdm lightdm-gtk-greeter xorg xorg-xinit xterm
```

On édite le fichier `/etc/X11/xinit/xinitrc`, on commente toutes les lignes à partir de `twm &`, puis, l'on ajoute la ligne suivante :
```shell
exec i3
```

On configure notre clavier pour lightdm :
```shell
sudo pacman -S wget
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/10-keyboard.conf -O /etc/X11/xorg.conf.d/10-keyboard.conf
```

On active lightdm au démarrage :
```shell
systemctl enable lightdm.service
```

On redémarre :
```shell
reboot
```

A partir de là, nous nous connecterons uniquement via notre utilisateur.

### Configuration de i3

On configure i3 :
```shell
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/i3.conf -O ~/.config/i3/config
```

On crée le répertoire permettant de stocker les captures d'écrans :
```shell
mkdir -p ~/Pictures/Screenshots
```

Dans le fichier `~/.config/i3/config` , il faut penser à modifier la ligne `bindsym Print exec flameshot full -c -p "/home/nyxott/Pictures/Screenshots/"` en remplaçant `nyxott` par votre utilisateur.

### AUR

On installe yay afin de pouvoir installer, par la suite, des paquets présents au sein de l'AUR :
```shell
sudo pacman -S git
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

### Installation du terminal

On installe le terminal kitty :
```shell
sudo pacman -S kitty
wget "https://raw.githubusercontent.com/dexpota/kitty-themes/master/themes/Monokai_Pro_(Filter_Spectrum).conf" -P ~/.config/kitty/kitty-themes/themes
ln -s ~/.config/kitty/kitty-themes/themes/Monokai_Pro_(Filter_Spectrum).conf ~/.config/kitty/theme.conf
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/kitty.conf -O ~/.config/kitty/kitty.conf
```

### Installation du shell

On installe le shell zsh :
```shell
sudo pacman -S zsh
chsh -s /usr/bin/zsh
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh
git clone https://github.com/zsh-users/zsh-autosuggestions.git ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting
```

On configure notre thème :
```shell
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/nyxott.zsh-theme -O ~/.oh-my-zsh/themes/nyxott.zsh-theme
```

On configure zsh :
```shell
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/zsh.conf -O ~/.zshrc
```

### Installation des utilitaires de son

On installe ce dont on a besoin pour gérer le son :
```shell
sudo pacman -S pavucontrol pulseaudio
```

### Installation des utilitaires de luminosité

On installe ce dont on a besoin pour gérer la luminosité :
```shell
sudo pacman -S xf86-video-intel xorg-xbacklight
```

### Installation des Nerd Fonts

On installe les Nerd Fonts :
```shell
yay -S nerd-fonts-complete
```

### Installation de Polybar

On installe polybar :
```shell
yay -S polybar
```

On crée le dossier `~/.config/polybar` :
```shell
mkdir ~/.config/polybar
```

On crée le fichier `~/.config/polybar/launch.sh` :
```shell
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/launch.sh -O ~/.config/polybar/launch.sh
chmod +x ~/.config/polybar/launch.sh
```

On configure polybar :
```shell
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/polybar.conf -O ~/.config/polybar/config
```

### Installation de Rofi

On installe rofi :
```shell
sudo pacman -S rofi
```

On crée le dossier `~/.config/rofi` :
```shell
mkdir ~/.config/rofi
```

On configure rofi :
```shell
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/rofi.conf -O ~/.config/rofi/config.rasi
```

### Installation de Dunst

On installe dunst :
```shell
sudo pacman -S dunst
```

On crée le répertoire `~/.config/dunst` :
```shell
mkdir ~/.config/dunst
```
On configure dunst :
```shell
wget https://raw.githubusercontent.com/Nyxott/ArchLinuxConfiguration/main/dunst.conf -O ~/.config/dunst/dunstrc
```

### Wallpapers

On crée un dossier qui contiendra nos wallpapers :
```shell
mkdir ~/.wallpapers/
```

On installe un navigateur pour pouvoir télécharger nos fonds d'écran :
```shell
yay -S brave-bin
```

L'image que l'on veut comme fond d'écran doit être enregistrée comme `~/.wallpapers/wallpaper.jpg`.  
L'image que l'on veut comme fond d'écran de verrouillage doit être enregistrée comme `~/.wallpapers/wallpaperLock.png`.

### Installation Betterlockscreen

On installe betterlock :
```shell
sudo pacman -S feh imagemagick
yay -S betterlockscreen
betterlockscreen -u ~/.wallpapers/wallpaperLock.png
```

### Installation du thème lightdm :

On installe le thème lightdm :
```shell
yay -S lightdm-webkit-theme-aether
```

### Installation du thème GRUB :

On installe le thème GRUB :
```shell
git clone https://github.com/vinceliuice/grub2-themes.git /tmp/grub2-themes.git
sudo /tmp/grub2-themes.git/install.sh -b -v
```

### Redémarrage

Nous pouvons désormais redémarrer la machine et constater que tout fonctionne correctement :
```shell
sudo reboot
```

### Paquets

On installe tous les paquets utiles :
```shell
sudo pacman -Sy
sudo pacman -S curl discord flameshot htop libreoffice-still unzip zip
yay -Sua
yay -S spotify visual-studio-code-bin
```
