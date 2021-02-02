---
weight: 4
title: "Hugo"
date: 2021-01-01T00:00:00+00:00
draft: false
author: "Nyxott"
description: "Créer son blog avec Hugo et le mettre en ligne via GitHub Pages"
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Miscellaneous"]
categories: ["Miscellaneous"]

lightgallery: true
---

Cet article explique comment créer son blog avec Hugo et le publier via GitHub Pages.

<!--more-->

## Introduction

[Hugo](https://gohugo.io/) est un logiciel écrit en Go permettant de générer facilement des sites Web statiques. Ce site lui-même a été généré via Hugo.

La force d'Hugo réside dans les nombreux thèmes développés par la communauté. De plus, Hugo est parfait pour les blogs puisqu'il permet de se focaliser sur la rédaction des articles (en [Markdown](https://www.markdownguide.org/)) au lieu du design du site ou de son architecture à proprement parler.

Si vous voulez également créer votre blog et le publier gratuitement sur Internet via [GitHub Pages](https://pages.github.com/), la procédure à suivre est assez simple.

## Création des repositories sur GitHub

Tout d'abord, on crée sur GitHub un repository nommé `blog` et un autre nommé `<USERNAME>.github.io`.

Bien sûr, il faudra remplacer `<USERNAME>` par votre nom d'utilisateur GitHub.

{{< admonition >}}
Je pars du principe qu'aucun `README.md`, ni autre fichier, n'ai été créé au moment de l'initialisation du repository.
{{< /admonition >}}

## Installation d'Hugo

On installe Hugo :
```shell
sudo apt install hugo
```

## Création du blog

Nous pouvons alors générer localement la base de notre site Hugo :
```shell
hugo new site blog
```

{{< admonition tip >}}
Je conseille de supprimer les dossiers `archetypes` et `layouts` pour qu'ils ne priment pas sur ceux du thème que nous allons utiliser. Par la suite, vous pourrez les recréer si vous souhaitez ajouter de nouveaux archetypes/layouts ou override ceux déjà présents dans le thème choisi.
{{< /admonition >}}

## Installation du thème

Nous pouvons ensuite [sélectionner un thème](https://themes.gohugo.io/) parmi de nombreux disponibles.

Personnellement, j'ai choisi le thème `LoveIt`.

Une fois notre thème choisi, nous allons l'installer :
```shell
sudo apt install git
cd blog
git init
git remote add origin https://github.com/<USERNAME>/blog.git
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

Bien sûr, vous devrait adapter ces lignes en fonction du thème que vous aurez choisi.

Dans la documentation de LoveIt, il est indiqué que le contenu par défaut du fichier `config.toml`, qui est le fichier de configuration de notre site, doit être le suivant :
```toml
baseURL = "http://example.org/"
# [en, zh-cn, fr, ...] determines default content language
defaultContentLanguage = "en"
# language code
languageCode = "en"
title = "My New Hugo Site"

# Change the default theme to be use when building the site with Hugo
theme = "LoveIt"

[params]
  # LoveIt theme version
  version = "0.2.X"

[menu]
  [[menu.main]]
    identifier = "posts"
    # you can add extra information before the name (HTML format is supported), such as icons
    pre = ""
    # you can add extra information after the name (HTML format is supported), such as icons
    post = ""
    name = "Posts"
    url = "/posts/"
    # title will be shown when you hover on this menu link
    title = ""
    weight = 1
  [[menu.main]]
    identifier = "tags"
    pre = ""
    post = ""
    name = "Tags"
    url = "/tags/"
    title = ""
    weight = 2
  [[menu.main]]
    identifier = "categories"
    pre = ""
    post = ""
    name = "Categories"
    url = "/categories/"
    title = ""
    weight = 3

# Markup related configuration in Hugo
[markup]
  # Syntax Highlighting (https://gohugo.io/content-management/syntax-highlighting)
  [markup.highlight]
    # false is a necessary configuration (https://github.com/dillonzq/LoveIt/issues/158)
    noClasses = false
```

A vous de vous adapter au thème que vous avez choisi. Bien souvent, un fichier par défaut est proposé ou il suffit de copier celui présent dans le dossier `exampleSite` du thème en question.

{{< admonition warning >}}
La variable `baseURL` doit impérativement être modifiée pour contenir la valeur `https://<USERNAME>.github.io/` où `<USERNAME>` est à remplacer par votre nom d'utilisateur GitHub.
{{< /admonition >}}

## Liason entre les deux repositories

Le repository `blog` peut être assimilé au backend de notre site. En effet, c'est dans ce dossier que sont regroupés, entre autres, nos fichiers markdown, nos images/videos et quelques snippets.

Le dossier `public` du repository `blog` peut être assimilé au frontend de notre site. En effet, lorsque nous allons build le site, le theme choisi sera utilisé pour créer les fichiers HTML/CSS nécessaires et  nos fichiers markdown seront également transformés en fichiers HTML. L'ensemble de ces fichiers sera stocké au sein du dossier `public` en formant une arborescence particulière qui créera notre site statique. C'est pourquoi nous allons relier ce dossier au repository `<USERNAME>.github.io` :
```shell
git submodule add -b main https://github.com/<USERNAME>/<USERNAME>.github.io.git public
```

## Création de contenu

Dans un site Hugo, il y a le dossier `content`. Au sein de ce dossier, il y a d'autres dossiers qui représentent des catégories. Ces catégories constitueront généralement le menu du site. Les fichiers Markdown sont placés dans ces dossiers et sont ainsi triés par catégories.

Vous pouvez créer des dossiers/fichiers manuellement dans le dossier `content` ou bien utiliser la commande Hugo dédié à cela :
```shell
hugo new <DIRECTORY_NAME>/<FILE_NAME>.md
```

Une fois cette commande effectuée vous aurait, dans le dossier `content`, un dossier nommé `<DIRECTORY_NAME>` et dans ce dossier un fichier nommé `<FILE_NAME>.md`.

{{< admonition >}}
Suivant le thème sélectionner, il est possible que l'arborescence du dossier `content` change.
{{< /admonition >}}

Par défaut, le fichier contiendra plusieurs variables comprises entre deux `---` qui représentent l'en-tête du fichier.

La variable `draft`, si elle est à `true`, empêche le post d'être publié. Il faudra donc changer cette valeur à `false` une fois le post prêt à être mis en ligne.

A des fins de test, vous pouvez donc passer à `false` la variable `draft` afin d'avoir au moins un post de visible.

## Visualisation du blog en localhost

Avant de build le site, il est possible de vérifier localement le rendu de ce dernier :
```shell
hugo server --disableFastRender
```

Vous pouvez visualiser votre site en vous rendant à l'adresse [http://localhost:1313](http://localhost:1313).

## Build

Comme indiqué précédemment, build le site mettra à jour le site statique dans le dossier `public` :
```shell
hugo
```

## Déploiement

Une fois le site build et les dernières modifications prisent ainsi en compte, nous pouvons effectuer un commit pour chacun de nos repositories sur GitHub :
```shell
git add .
git commit -m "<COMMIT_MESSAGE>"
git push origin master
cd public
git add .
git commit -m "<COMMIT_MESSAGE>"
git push origin master
```

GitHub se charge du reste et vous pouvez vous rendre sur [https://<USERNAME>.github.io/](#) pour visualiser votre blog en ligne.

{{< admonition >}}
Il faut parfois attendre quelques minutes pour que les modifications soient visibles.
{{< /admonition >}}

## Déploiement automatique

Déployer son blog en ligne n'est pas long, mais très repétitif.

C'est pourquoi nous pouvons créer un script afin d'automatiser ce processus :
```bash
#!/bin/bash

NOW=$(date +"%d-%m-%Y %T")

echo -e "\e[1;35mBuild site\e[0m"
cd blog
hugo > /dev/null 2>&1

echo -e "\e[1;35mCommit to the blog repository...\e[0m"

git add .
git commit -m "Update : $NOW" > /dev/null 2>&1
git push origin master > /dev/null 2>&1

echo -e "\e[1;35mCommit to the github.io repository...\e[0m"

cd public
git add .
git commit -m "Update : $NOW" > /dev/null 2>&1
git push origin master > /dev/null 2>&1

echo -e "\e[1;35mSuccessfully committed\e[0m"
```

{{< admonition >}}
Ce script est à lancer dans le dossier parent au dossier `blog`.
{{< /admonition >}}
