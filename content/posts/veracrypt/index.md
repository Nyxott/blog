---
weight: 4
title: "VeraCrypt"
date: 2021-01-15T00:00:00+00:00
draft: false
author: "Nyxott"
description: "Utiliser VeraCrypt afin d'interagir avec un volume chiffré"
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Miscellaneous"]
categories: ["Miscellaneous"]

lightgallery: true
---

Cet article explique comment utiliser VeraCrypt afin d'interagir avec un volume chiffré.

<!--more-->

## Introduction

[VeraCrypt](https://www.veracrypt.fr/en/Home.html) est un fork du projet abandonné [TrueCrypt](https://truecrypt.fr/).

Cet outil possède de nombreuses fonctionnalités dont celle permettant de créer des volumes chiffrés.

## Installation de VeraCrypt

Les fichiers d'installation de VeraCrypt sont disponibles sur [https://www.veracrypt.fr/en/Downloads.html](https://www.veracrypt.fr/en/Downloads.html).

Une fois téléchargé nous pouvons l'installer :
```shell
sudo dpkg -i <PATH_TO_VERACRYPT.deb>
```

## Utilisation simple

### Créer un volume chiffré

Nous pouvons créer un volume chiffré avec la commande suivante :
```shell
veracrypt -t -c --volume-type=normal --size=1G --encryption=aes-twofish-serpent --hash=whirlpool --filesystem=ext4 --pim=0 -k "" --random-source=/dev/urandom /home/nyxott/Documents/SecretDocuments
```

{{< admonition >}}
Les arguments qui ne sont pas fournis directement dans la ligne de commande seront demandés dans un environnement interactif.
{{< /admonition >}}

{{< admonition tip >}}
Si vous souhaitez utiliser un PIN et/ou une clef, vous pouvez retirer les arguments --pim et/ou -k afin de les renseigner de manière interactive et empêcher ainsi l'enregistrement de ces informations dans l'history de votre utilisateur.
{{< /admonition >}}

{{< admonition >}}
A noter que le choix de l'algorithme de chiffrement `AES(Twofish(Serpent))` ainsi que de l'algorithme de hachage `Whirlpool` est purement personnel. Le plus important sera la longueur et la complexité du mot de passe qui vous permettra d'accéder à votre volume.
{{< /admonition >}}

### Monter le volume chiffré

Nous pouvons monter le volume chiffré via la commande suivante :
```shell
veracrypt -t -k "" --pim=0 --protect-hidden=no /home/nyxott/Documents/SecretDocuments /mnt/SecretDocuments/
```

{{< admonition >}}
Assurez-vous d'avoir créé le dossier `/mnt/SecretDocuments` au préalable.
{{< /admonition >}}

Désormais, le volume est accessible en lecture et en écriture.

### Démonter le volume chiffré

Nous pouvons démonter le volume chiffré via la commande suivante :
```shell
veracrypt -d /mnt/SecretDocuments/
```

{{< admonition tip >}}
Si vous avez plusieurs volumes montés vous pouvez tous les démonter simultanément via la commande suivante :
```
veracrypt -d
```
{{< /admonition >}}

## Utilisation avancée

### Déni plausible

Si vous le désirez, il est possible de créer un volume chiffré puis, de recréer un second volume chiffré au sein du premier.

Cela peut sembler inutile, mais le fait est que VeraCrypt remplit le contenu des volumes chiffrés avec des données aléatoires. De cette manière, il est impossible de distinguer qu'un second volume chiffré est caché dans le premier puisqu'il se confondra avec ces données aléatoires. 

Cela permet de mettre en oeuvre [le déni plausible](https://fr.wikipedia.org/wiki/D%C3%A9ni_plausible_(cryptologie)).

Ainsi, il est possible de déchiffrer le premier volume pour accéder à des données que l'on fait croire importantes alors qu'elles n'ont aucune valeur, puis, de nier l'existence d'un second volume chiffré.

### Créer le volume chiffré

Nous pouvons créer un volume chiffré avec la commande suivante :
```shell
veracrypt -t -c --volume-type=normal --size=1G --encryption=aes-twofish-serpent --hash=whirlpool --filesystem=ext4 --pim=0 -k "" --random-source=/dev/urandom /home/nyxott/Documents/SecretDocuments
```

### Créer le volume chiffré caché

Nous pouvons créer un volume chiffré caché avec la commande suivante :
```shell
veracrypt -t -c --volume-type=hidden --size=950M --encryption=aes-twofish-serpent --hash=whirlpool --filesystem=ext4 --pim=0 -k "" --random-source=/dev/urandom /home/nyxott/Documents/SecretDocuments
```

{{< admonition warning >}}
Le trio mot de passe/PIN/clef doit obligatoirement différer entre le volume parent et le volume caché.

Si ce n'est pas le cas, VeraCrypt ne pourra pas distinguer le volume parent du volume caché et montera toujours le même par la suite.
{{< /admonition >}}

### Monter le volume chiffré

Nous pouvons monter le volume chiffré via la commande suivante :
```shell
veracrypt -t -k "" --pim=0 --protect-hidden=no /home/nyxott/Documents/SecretDocuments /mnt/SecretDocuments
```

{{< admonition >}}
Si vous entrez le trio mot de passe/PIN/clef faisant référence au volume parent, ce dernier sera monté.

A l'inverse si le trio d'informations renseigné est celui du volume caché, ce sera celui ci qui sera monté.
{{< /admonition >}}

### Démonter le volume chiffré

Nous pouvons démonter le volume chiffré via la commande suivante :
```shell
veracrypt -d /mnt/SecretDocuments/
```

## Bonus

### Lister les volumes chiffrés montés

Il est possible de lister les volumes chiffrés montés via la commande suivante :
```shell
veracrypt -l
```
