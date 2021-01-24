---
weight: 4
title: "Coffre-Fort Numérique Distant"
date: 2020-09-01T00:00:00+00:00
draft: false
author: "Nyxott"
description: "Créer un coffre-fort numérique distant"
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Projects"]
categories: ["Projects"]

lightgallery: true
---

Cet article explique comment créer un coffre-fort numérique chiffré et accessible à distance.

<!--more-->

## Introduction

Lorsque nous avons besoin d'installer ou de réinstaller une machine Linux pour une raison x ou y (machine personnelle, serveur Web, etc.), nous devons recommencer, la plupart du temps, les mêmes procédures et effectuer les mêmes configurations.

Or, il arrive souvent d'écrire un fichier de configuration complet et propre, mais de finir par le perdre. Il faut alors tout recommencer et prendre plusieurs heures afin de refaire ce qui aurait pu être un simple copier/coller d'un fichier de configuration déjà fait (en effectuant peut être quelques légères modifications bien sûr).

L'idéal serait donc d'avoir une sorte de coffre-fort chiffré présent sur une machine chez soit et accessible depuis n'importe où de manière sécurisée. Ainsi, lorsqu'un fichier de configuration est finis, il serait possible de le placer dans ledit coffre puis de le récupérer n'importe quand.

## Projet

Nous allons donc voir comment mettre en place un tel coffre.

{{< admonition >}}
Même si les actions les plus importantes sont décrites et expliquées, un minimum de connaissances en Linux est nécessaire afin de comprendre ce post.
{{< /admonition >}}

### Prérequis

On utilise une image ISO *Debian Netinstall* disponible à l'adresse suivante : [https://www.debian.org/CD/netinst/](https://www.debian.org/CD/netinst/) (image de CD d'installation par le réseau (amd64))

On crée une clé USB bootable avec cette image ISO :
```shell
sudo fdisk -l (pour repérer la clé USB)  
sudo dd if=/chemin/du/fichier/debian.iso of=/dev/sd<X> bs=1M conv=fdatasync ("X" correspond à la lettre attribuée au volume de la clé USB)
```

### Partitionnement

Nous allons effectuer une installation manuelle afin de chiffrer et partitionner notre disque :
```shell
Manual
```

On sélectionne le disque sur lequel on souhaite installer notre OS (sda par exemple).

Nous avons le message `Create new empty partition table on this device` :
```shell
Yes
```

Nous sélectionnons alors la ligne indiquant `FREE SPACE`.

Nous avons le message `How to use this free space` :
```shell
Create a new partition
```

Par défaut la taille de la partition sera automatiquement mise au maximum nous allons la réduire pour la partition de /boot :
```shell
512M
Continue
```

Nous avons le message `Type for the new partition` :
```shell
Primary
```

Nous avons le message `Location for the new partition` :
```shell
Beginning
```

Nous arrivons aux paramètres de la partition :
```shell
Use as:          Ext4 journaling file system
Mount point:     /boot
Mount options:   defaults
Label:           BOOT
Reserved blocks: 5%
Typical usage:   standard
Bootable flag:   on

Done setting up the partition
```

Nous sélectionnons de nouveau la ligne indiquant `FREE SPACE`.

Nous avons le message `How to use this free space` :
```shell
Create a new partition
```

Par défaut la taille de la partition sera automatiquement mise au maximum nous n'y touchons donc pas :
```shell
Continue
```

Nous avons le message `Type for the new partition` :
```shell
Primary
```

Nous arrivons aux paramètres de la partition :
```shell
Use as:            physical volume for encryption
Encryption method: Device-mapper (dm-crypt)
Encryption:        aes
Key size:          256
IV algorithm:      xts-plain64
Encryption key:    Passphrase
Erase data:        no
Bootable flag      off

Done setting up the partition
```

Nous sélectionnons la ligne `Configure encrypted volumes`.

Nous avons le message `Write the changes to disk and configure encrypted volumes` :
```shell
Yes
```

Nous avons le message `Encryption configuration actions` :
```shell
Create encrypted volumes
```

Nous avons le message `Devices to encrypt` :
```shell
/dev/sdx2 ("x" représente la lettre du disque sélectionné au début (par exemple "a"))
Continue
```

Nous avons de nouveau le message `Encryption configuration actions` :
```shell
Finish
```

Nous choisissons une passphrase qui sera demandée pour déchiffrer le disque dur au démarrage de la machine.

Nous sélectionnons la ligne `Configure the Logical Volume Manager`.

Nous avons le message `Write the changes to disk and configure LVM` :
```shell
Yes
```

Nous avons le message `LVM configuration action` :
```shell
Create volume group
```

Nous entrons un nom (par exemple `VGCRYPT`).

Nous avons le message `Devices for the new volume group` :
```shell
/dev/mapper/sdx2_crypt ("x" représente la lettre du disque sélectionné au début (par exemple "a"))
Continue
```

{{< admonition >}}
Ici, j'utilise un disque dur de 10Go, mais pour la taille des partitions à venir, vous pouvez répartir l'espace comme bon vous semble.
{{< /admonition >}}

Nous avons le message `LVM Configuration details` :
```shell
Create logical volume
```

Nous avons le message `Volume group` :
```shell
VGCRYPT
```

Nous entrons un nom :
```shell
lv_swap
```

On définit une taille pour la partition `swap` :
```shell
2G
```

Nous avons de nouveau le message `LVM Configuration details` :
```shell
Create logical volume
```

Nous avons le message `Volume group` :
```shell
VGCRYPT
```

Nous entrons un nom :
```shell
lv_tmp
```

On définit une taille pour la partition `/tmp` :
```shell
1G
```

Nous avons de nouveau le message `LVM Configuration details` :
```shell
Create logical volume
```

Nous avons le message `Volume group` :
```shell
VGCRYPT
```

Nous entrons un nom :
```shell
lv_home
```

On définit une taille pour la partition `/home` :
```shell
3G
```

Nous avons de nouveau le message `LVM Configuration details` :
```shell
Create logical volume
```

Nous avons le message `Volume group` :
```shell
VGCRYPT
```

Nous entrons un nom :
```shell
lv_root
```

On définit une taille pour la partition `/` :
```shell
3.5G
```
{{< admonition >}}
Ici, il me reste 700Mb de disponible. Nous les utiliserons par la suite pour le coffre. Vous pouvez faire en sorte d'avoir plus ou moins d'espace disponible pour le coffre suivant vos préférences et surtout vos besoins.
{{< /admonition >}}

Nous avons de nouveau le message `LVM Configuration details` :
```shell
Finish
```

Nous sélectionnons la ligne en dessous de celle indiquant `LVM VG VGCRYPT, LV lv_swap`.

Nous arrivons aux paramètres de la partition :
```shell
Use as:	swap area

Done setting up the partition
```

Nous sélectionnons la ligne en dessous de celle indiquant `LVM VG VGCRYPT, LV lv_tmp`.

Nous arrivons aux paramètres de la partition :
```shell
Use as:          Ext4 journaling file system
Mount point:     /tmp
Mount options:   defaults
Label:           TMP
Reserved blocks: 5%
Typical usage:   standard

Done setting up the partition
```

Nous sélectionnons la ligne en dessous de celle indiquant `LVM VG VGCRYPT, LV lv_home`.

Nous arrivons aux paramètres de la partition :
```shell
Use as:        XFS journaling file system
Mount point:   /home
Mount options: defaults
Label:         HOME

Done setting up the partition
```

Nous sélectionnons la ligne en dessous de celle indiquant `LVM VG VGCRYPT, LV lv_root`.

Nous arrivons aux paramètres de la partition :
```shell
Use as:          Ext4 journaling file system
Mount point:     /
Mount options:   defaults
Label:           RACINE
Reserved blocks: 5%
Typical usage:   standard

Done setting up the partition
```

Nous sélectionnons la ligne indiquant `Finish paritioning and write changes to disk`.

Nous avons le message `Write the changes to disks` :
```shell
Yes
```

### Paquets à installer

Lorsque nous avons le message `Choose software to install`, nous installons `SSH server` et `standard system utilities`.

### Vérification partitionnement

A partir de là, nous pouvons nous connecter en `root`. Si nous faisons un `lsblk -f` nous devrions avoir la sortie suivante (ou quelque chose de similaire) :
```shell
sda
|-sda1                ext4 BOOT
|-sda2                crypto
  |-sda2_crypt        LVM2_m
    |-VGCRYPT-lv_tmp  ext4 TMP
    |-VGCRYPT-lv_swap swap
    |-VGCRYPT-lv_home xfs  HOME
    |-VGCRYPT-lv_root ext4 RACINE
```

### SSH

Nous devons désormais configurer le serveur SSH de manière à ce que nous puissions nous connecter dessus par la suite.

Nous souhaitons nous connecter à notre serveur uniquement par clef SSH et non pas par mot de passe.

Afin de mettre cela en place (en plus d'un léger hardening) il faut modifier le fichier `/etc/ssh/sshd_config` :
```shell
rm /etc/ssh/sshd_config
cd /etc/ssh/
wget https://raw.githubusercontent.com/Nyxott/CoffreDistantChiffre/master/sshd_config
systemctl restart sshd.service
```

Les contenus des fichiers `/etc/issue` et `/etc/issue.net` doivent également être modifiés :
```shell
rm /etc/issue*
cd /etc/
wget https://raw.githubusercontent.com/Nyxott/CoffreDistantChiffre/master/banner -O issue
cp issue issue.net
```

Le fichier ci-dessus sera affiché à un utilisateur avant qu'il se connecte.

On va supprimer le contenu du fichier `/etc/motd` :
```shell
rm /etc/motd
touch /etc/motd
```

On va supprimer le fichier `/etc/update-mot.d/10-uname` :
```shell
rm /etc/update-mot.d/10-uname
```

On va créer le fichier `/etc/update-mot.d/00-infosys` :
```shell
cd /etc/update-mot.d/
wget https://raw.githubusercontent.com/Nyxott/CoffreDistantChiffre/master/00-infosys
chmod +x /etc/update-mot.d/00-infosys
```

Ce script sera exécuté lorsque nous nous connecterons à la machine et nous permettra d'avoir rapidemment quelques informations utiles concernant cette dernière.

Sur le client (la machine avec laquelle nous souhaitons nous connecter au serveur en SSH), nous allons générer une clef SSH :
```shell
ssh-keygen -t ed25519 (il ne faut pas mettre de passphrase)
```

Nous devons ensuite copier le contenu du fichier `/home/<CLIENT_USER>/.ssh/id_ed25519.pub` de notre client dans le fichier `/etc/ssh/keys/<SERVER_USER>/authorized_keys` de notre serveur.

{{< admonition >}}
Le dossier `/etc/ssh/keys` n'existe pas par défaut et devra donc être créé (`mkdir /etc/ssh/keys`). Le dossier `<SERVER_USER>` correspond au nom d'utilisateur avec lequel on se connectera sur le serveur (`ssh <SERVER_USER>@IP`).
{{< /admonition >}}

### Vérification SSH

A partir de là, nous devrions être capable de nous connecter en SSH de notre client vers notre serveur.

Afin de vérifier cela, nous allons créer le fichier `/home/<CLIENT_USER>/.ssh/config` sur notre client :
```shell
Host <THE_NAME_YOU_WANT>
    HostName <SERVER_IP>
    User <USER_SERVER>
    Port <SERVER_SSH_PORT>
    IdentityFile <PATH_TO_PRIVATE_KEY>
```

Puis nous pouvons nous connecté :
```
ssh <THE_NAME_YOU_WANT>
```

Si tout fonctionne correctement nous devrions avoir le contenu du fichier `/etc/issue.net` qui s'affiche, suivi de la demande de saisie de la passphrase. Une fois la passphrase validée nous devrion voir le script contenu dans `/etc/update-mot.d/00-infosys` s'exécuter et nous devrions être connecté avec l'utilisateur présent sur le serveur.

### Dropbear

Nous avons un disque chiffré et nous pouvons nous connecter en SSH à notre serveur. Mais, si nous devons redémarrer le serveur a distance pour une raison x ou y, nous ne pouvons pas entrer le mot de passe demandé au démarrage afin de déchiffrer le disque dur. C'est pourquoi nous allons mettre en place l'outil Dropbear. Dropbear est un équivalent de OpenSSH, mais est beaucoup plus léger.

Nous installons donc Dropbear sur notre serveur :
```shell
apt install dropbear busybox
```

Dans le fichier `/etc/initramfs-tools/initramfs.conf` :
```shell
BUSYBOX=y (remplace "BUSYBOX=auto")
DROPBEAR=y (à ajouter)
```

Nous nous rendons dans le dossier `/etc/dropbear-initramfs/` :
```shell
cd /etc/dropbear-initramfs/
```

Nous effectuons les commandes suivantes :
```shell
/usr/lib/dropbear/dropbearconvert dropbear openssh dropbear_rsa_host_key id_rsa
dropbearkey -y -f dropbear_rsa_host_key | grep "^ssh-rsa" > id_rsa.pub
```

Sur le client nous créons une nouvelle clef SSH :
```shell
ssh-keygen -t rsa -b 4096
```

Nous copions ensuite le contenu du fichier `/home/<CLIENT_USER>/.ssh/id_rsa.pub` de notre client dans le fichier `/etc/dropbear-initramfs/authorized_keys` de notre serveur.

{{< admonition warning >}}
Attention, seul les toutes dernières version de Dropbear supportent des clefs ed25519 il est donc préférable pour le moment d'utiliser des clefs RSA.
{{< /admonition >}}

Dans le fichier `/etc/default/dropbear` :
```shell
NO_START=0 (remplace "NO_START=1")
DROPBEAR_PORT=lePortQueVousSouhaitezAutreQueLePortSSH (à ajouter)
```

Dans le fichier `/etc/dropbear-initramfs/config` :
```shell
DROPBEAR_OPTIONS="-p leMeMePortQueDansLeFichierPrecedent" (à décommenter et compléter)
```

Nous nous rendons dans le dossier `/etc/initramfs-tools/hooks/` :
```shell
cd /etc/initramfs-tools/hooks/
```

Nous effectuons les commandes suivantes :
```shell
wget https://gist.githubusercontent.com/gusennan/712d6e81f5cf9489bd9f/raw/fda73649d904ee0437fe3842227ad8ac8ca487d1/crypt_unlock.sh
chmod +x crypt_unlock.sh
update-initramfs -u
systemctl disable dropbear
```

Dans le fichier `/etc/default/grub` :
```shell
GRUB_CMDLINE_LINUX_DEFAULT="" (remplace GRUB_CMDLINE_LINUX_DEFAULT="quiet")
```

Nous effectuons la commande suivante :
```shell
update-grub
```

### Vérification Dropbear

A partir de là, nous devrions être capable de déchiffrer notre serveur depuis notre client.

Nous redémarrons le serveur et le boot se bloque avec le message `Begin: Starting dropbear ...`.

Nous ajoutons à notre fichier `/home/<CLIENT_USER>/.ssh/config` présents sur notre client, les lignes suivantes :
```shell
Host <THE_NAME_YOU_WANT>
    HostName <SERVER_IP>
    User root
    Port <SERVER_DROPBEAR_PORT>
    IdentityFile <PATH_TO_PRIVATE_KEY>
```

Sur le client nous effectuons les commandes suivantes :
```shell
ssh <THE_NAME_YOU_WANT>
unlock
```

Nous pouvons alors saisir le mot de passe permettant de déchiffrer le disque dur et le serveur démarre.

{{< admonition tip >}}
Si vous êtes directement sur le serveur et non pas en SSH vous pouvez taper sur la touche "Entrée" pour passer dropbear et directement entrer le mot de passe de déchiffrement.
{{< /admonition >}}

### Coffre

Nous pouvons désormais créer notre coffre chiffré via les commandes suivantes :
```shell
lvcreate -l 100%FREE -n lv_coffre_chiffre VGCRYPT (on crée "lv_coffre_chiffre" dans le volume VGCRYPT en utilisant tout l'espace disponible)
cryptsetup luksFormat /dev/mapper/VGCRYPT-lv_coffre_chiffre
cryptsetup luksOpen /dev/mapper/VGCRYPT-lv_coffre_chiffre coffre_chiffre
mkfs.xfs /dev/mapper/coffre_chiffre
```

Dans le fichier `/etc/fstab` nous ajoutons la ligne suivante :
```shell
/dev/mapper/coffre_chiffre	/home/<SERVER_USER>/COFFRE	xfs	defaults,noauto,user	0	2
```

On installe `sudo` :
```shell
apt install sudo
```

On ouvre le fichier `/etc/sudoers` :
```shell
visudo
```

On y ajoute les lignes suivantes :
```shell
<SERVER_USER>	ALL=(ALL) 		/usr/sbin/cryptsetup luksOpen /dev/VGCRYPT/lv_coffre_chiffre coffre_chiffre
<SERVER_USER>	ALL=(ALL) NOPASSWD:	/usr/sbin/cryptsetup luksClose /dev/mapper/coffre_chiffre
```

### Vérification coffre

En tant que simple utilisateur nous devons pouvoir ouvrir, monter, démonter et fermer le coffre dans notre répertoire courant.

Pour ouvrir et monter le coffre :
```shell
mkdir /home/<SERVER_USER>/COFFRE
sudo /usr/sbin/cryptsetup luksOpen /dev/VGCRYPT/lv_coffre_chiffre coffre_chiffre
mount /home/<SERVER_USER>/COFFRE
```

Le coffre étant monté via sudo, il a les droits de root. Nous devons donc indiquer, en tant que root, que le coffre appartient à notre utilisateur :
```shell
chown <SERVER_USER>:<SERVER_USER> /home/<SERVER_USER>/COFFRE
```

{{< admonition >}}
Il suffit de le faire la première fois seulement.
{{< /admonition >}}

Pour démonter et fermer le coffre :
```shell
umount /home/<SERVER_USER>/COFFRE
sudo /usr/sbin/cryptsetup luksClose /dev/mapper/coffre_chiffre
rm -r /home/<SERVER_USER>/COFFRE
```

### Automatisation coffre

Afin d'automatiser l'ouverture et le montage du coffre ainsi que son démontage et sa fermeture nous allons mettre en place deux scripts.

Nous pouvons créer ces scripts dans le répertoire courant de l'utilisateur du serveur :
```shell
cd
wget https://raw.githubusercontent.com/Nyxott/CoffreDistantChiffre/master/openVault.sh
wget https://raw.githubusercontent.com/Nyxott/CoffreDistantChiffre/master/closeVault.sh
chmod +x openVault.sh
chmod +x closeVault.sh
```

### SSHFS

Nous pouvons désormais installer SSHFS sur notre client afin de pouvoir monter le coffre dessus :
```shell
apt install sshfs
```

### Vérification SSHFS

Pour monter le coffre depuis notre client et sur ce dernier :
```shell
ssh -t <THE_NAME_YOU_HAD_PREVIOUSLY_INDICATE> '/home/<SERVER_USER>/openVault.sh'
mkdir /home/<CLIENT_USER>/COFFRE
sshfs -o port=<SERVER_SSH_PORT_YOU_HAD_PREVIOUSLY_INDICATE>,IdentityFile=/home/<CLIENT_USER>/.ssh/id_ed25519 <SERVER_USER>@<SERVER_IP>:/home/<SERVER_USER>/COFFRE /home/<CLIENT_USER>/COFFRE
```

Pour démonter le coffre sur notre client et depuis ce dernier :
```shell
fusermount -u /home/<CLIENT_USER>/COFFRE
rm -r /home/<CLIENT_USER>/COFFRE
ssh <THE_NAME_YOU_HAD_PREVIOUSLY_INDICATE> '/home/<SERVER_USER>/closeVault.sh'
```

### Automatisation SSHFS

Afin d'automatiser l'ouverture et le montage du coffre ainsi que son démontage et sa fermeture, le tout à distance, nous allons mettre en place deux scripts.

Nous pouvons créer ces scripts dans le répertoire courant de l'utilisateur du client :

Script de montage distant :
```shell
cd
wget https://raw.githubusercontent.com/Nyxott/CoffreDistantChiffre/master/distantOpenVault.sh
wget https://raw.githubusercontent.com/Nyxott/CoffreDistantChiffre/master/distantCloseVault.sh
chmod +x distant*
```

### Exposition sur Internet

Maintenant que tout fonctionne entre le client et le serveur, il faut configurer le NAT/PAT sur la box.

L'interface de configuration change suivant les box, mais le principe reste le même.

Il faut choisir un port pour Dropbear et un autre pour SSH. Ces ports peuvent être les mêmes que ceux déjà choisi ou être différent.

Ainsi, si nous choisissons le port 7000 pour SSH nous devrons ouvrir vers l'extérieur le port 7000 de notre box et l'associé à l'IP de notre serveur ainsi qu'au port SSH de celui-ci qui sera le port que vous aviez indiqué dans le fichier `/etc/ssh/sshd_config`.
De même, si nous choisissons le port 7001 pour Dropbear nous devrons ouvrir vers l'extérieur le port 7001 de notre box et l'associé à l'IP de notre serveur ainsi qu'au port Dropbear de celui-ci qui sera le port que vous aviez indiqué dans le fichier `/etc/default/dropbear`.

Comme indiqué précédemment, si dans le fichier `/etc/ssh/sshd_config` vous aviez indiqué le port 5555 et dans le fichier `/etc/default/dropbear` le port 6666, vous pouvez également ouvrir ces mêmes ports sur votre box vers l'extérieur.

Enfin, vous devrez simplement modifier le fichier `~/.ssh/config` sur votre client pour indiquer l'adresse IP de votre box au lieu de celle du serveur et modifier les ports pour indiquer ceux que vous avez ouvert sur votre serveur.

### Bonus

#### Augmenter la taille du coffre

Si a un certain moment vous n'avez plus de place dans votre coffre, vous pouvez passer root et l'agrandir en diminuant la taille de la swap :
```shell
swapoff -a
lvreduce -L -500M /dev/VGCRYPT/lv_swap (réduis la swap de 500Mb)
mkswap /dev/mapper/VGCRYPT-lv_swap
swapon -a

lvextend -L +500M /dev/VGCRYPT/lv_coffre_chiffre
/home/<SERVER_USER>/COFFRE
xfs_growfs /home/<SERVER_USER>/COFFRE
```

#### Vérification de l'augmenation de la taille du coffre

Nous pouvons effectuer la commande suivante pour constater la nouvelle taille du coffre :
```shell
df -h
```
