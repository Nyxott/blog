---
weight: 4
title: "L'assembleur"
date: 2020-05-01T00:00:00+00:00
draft: false
author: "Nyxott"
description: "Les bases du fonctionnement de la pile et du langage assembleur"
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Miscellaneous"]
categories: ["Miscellaneous"]

lightgallery: true
---

Cet article explique les bases du fonctionnement de la pile et du langage assembleur.

<!--more-->

## La base

Avant tout, il faut savoir que lorsqu'un programme est exécuté par nos ordinateurs, le code final sera toujours retranscrit en assembleur. Ainsi, un programme en C ou en Python, par exemple, finira par devenir de l'assembleur. La raison est que les processeurs ne peuvent interpréter que des instructions assembleurs (transformées en binaire bien sûr).

De plus, il existe différents types de processeurs et les uns ne peuvent pas interpréter les mêmes instructions que les autres. C'est pourquoi il existe une multitude de langage assembleur ([FASM](https://flatassembler.net/), [GASM](https://sourceware.org/binutils/docs/as/), [MASM](https://docs.microsoft.com/fr-fr/cpp/assembler/masm/microsoft-macro-assembler-reference?view=vs-2019), [NASM](https://www.nasm.us/), etc.). Ici, nous utiliserons le NASM.

{{< admonition >}}
Il est à noter qu'entre un code assembleur en 32 bits et en 64 bits, quelques changements ont lieu. Ici, seul l'assembleur 64 bits sera abordé.
{{< /admonition >}}

## Les registres

De la même manière que nous utilisons des variables pour programmer en C ou en Python, par exemple, nous utilisons les registres des processeurs afin d'y stocker des informations. En effet, les instructions assembleurs, que nous verrons plus loin, manipulent directement les regitres. De plus, ces derniers sont un peu plus rapides d'accès que les caches et surtout beaucoup plus rapides d'accès que la RAM et c'est pourquoi nous devons les utiliser au maximum.

Un registre a au moins trois tailles différentes. A savoir, 8 bits (1 octet/byte), 16 bits (2 octets/word) et 32 bits (4 octets/double word).  
Les architectures 64 bits ont une taille de registre supplémentaire de 64 bits (8 octets/quad word).

{{< admonition >}}
Je ne tiens pas compte des sous-ensembles tels que le x87 dédié au nombre à virgule flottante par exemple.
{{< /admonition >}}

Les registres sont imbriqués les uns dans les autres excepté les deux plus petits qui sont distincts.

Pour comprendre cela, nous pouvons représenter le registre `rax` (64 bits) contenant le registre `eax` (32 bits) contenant le registre `ax` (16 bits) contenant le registre `ah` (8 bits) et `al` (8bits) de la manière suivante :

```
                  rax
 → → → → → → → → → → → → → → → → → → → 
↑                                     ↓ 
↑                  eax                ↓
↑      → → → → → → → → → → → → → →    ↓
↑     ↑                            ↓  ↓
↑     ↑              ax            ↓  ↓
↑     ↑      → → → → → → → → → →   ↓  ↓
↑     ↑     ↑                   ↓  ↓  ↓
↑     ↑     ↑     ah     al     ↓  ↓  ↓
↑     ↑     ↑     → →    → →    ↓  ↓  ↓
↑     ↑     ↑    ↑    ↓ ↑    ↑  ↓  ↓  ↓
[ rax [ eax [ ax [ ah ] [ al ]  ]  ]  ]
```

Donc, si nous devons stocker une information ayant besoin de 8 bits nous pouvons et nous devons utiliser le registre `ah` ou `al`. Par convention, nous privilégierons les 8 bits dit les plus faibles. C'est-à-dire le registre `al`.

{{< admonition warning >}}
Ici, il est conseillé d'enregistrer notre information de 8 bits dans le registre `ah` ou `al` puisque ces derniers font justement une taille de 8 bits. Néanmoins, il est également possible de stocker notre information dans le registre `rax` qui lui fait 64 bits. Seulement, les 56 bits restant seront inutilisés et par conséquent seront remplis par des null bytes. Il est également à noter que l'inverse n'est pas vrai. A savoir que si l'on a besoin de stocker 64 bits il faudra forcément utiliser `rax`, dans cet exemple, puisque 64 bits d'informations ne peuvent pas être contenus dans seulement 8 bits (ni même 16 ou 32).
{{< /admonition >}}

Il est à noter ici que le fait que les registres soient imbriqués les uns dans les autres n'est pas sans conséquences. En effet, le contenu de `al`, par exemple, aura des repercussions sur `rax`, `eax` et `ax`, de même que le contenu de `eax`, par exemple, aura des répercussions sur `rax`, `ax`, `ah` et `al`.

Seul `ah` et `al` n'ont aucun impact l'un sur l'autre.

Voici la liste de tous les registres existants :

| Registre 64 bits                          | Registre 32 bits                       | Registre 16 bits             | Registre 8 bits (high)   | Registre 8 bits (low)          |
| :---------------------------------------: | :------------------------------------: | :--------------------------: | :----------------------: | :----------------------------: |
| rax<br>(Re-extented Accumulator eXtended) | eax<br>(Extended Accumulator eXtended) | ax<br>(Accumulator eXtended) | ah<br>(Accumulator High) | al<br>(Accumulator Low)        |
| rbx<br>(Re-extented Base eXtended)        | ebx<br>(Extended Base eXtended)        | bx<br>(Base eXtended)        | bh<br>(Base High)        | bl<br>(Base Low)               |
| rcx<br>(Re-extented Counter eXtended)     | ecx<br>(Extended Counter eXtended)     | cx<br>(Counter eXtended)     | ch<br>(Counter High)     | cl<br>(Counter Low)            |
| rdx<br>(Re-extented Data eXtended)        | edx<br>(Extended Data eXtended)        | dx<br>(Data eXtended)        | dh<br>(Data High)        | dl<br>(Data Low)               |
| rdi<br>(Re-extented Destination Index)    | edi<br>(Extended Destination Index)    | di<br>(Destination Index)    | ∅                        | dil<br>(Destination Index Low) |
| rsi<br>(Re-extented Source Index)         | esi<br>(Extended Source Index)         | si<br>(Source Index)         | ∅                        | sil<br>(Source Index Low)      |
| rbp<br>(Re-extented Base Pointer)         | ebp<br>(Extended Base Pointer)         | bp<br>(Base Pointer)         | ∅                        | bpl<br>(Base Pointer Low)      |
| rsp<br>(Re-extented Stack Pointer)        | esp<br>(Extended Stack Pointer)        | sp<br>(Stack Pointer)        | ∅                        | spl<br>(Stack Pointer Low)     |
| r8<br>(Register 8)                        | r8d<br>(Register 8 Double word)        | r8w<br>(Register 8 Word)     | ∅                        | r8<br>(Register 8 Byte)        |
| r9<br>(Register 9)                        | r9d<br>(Register 9 Double word)        | r9w<br>(Register 9 Word)     | ∅                        | r9<br>(Register 9 Byte)        |
| r10<br>(Register 10)                      | r10d<br>(Register 10 Double word)      | r10w<br>(Register 10 Word)   | ∅                        | r10b<br>(Register 10 Byte)     |
| r11<br>(Register 11)                      | r11d<br>(Register 11 Double word)      | r11w<br>(Register 11 Word)   | ∅                        | r11b<br>(Register 11 Byte)     |
| r12<br>(Register 12)                      | r12d<br>(Register 12 Double word)      | r12w<br>(Register 12 Word)   | ∅                        | r12b<br>(Register 12 Byte)     |
| r13<br>(Register 13)                      | r13d<br>(Register 13 Double word)      | r13w<br>(Register 13 Word)   | ∅                        | r13b<br>(Register 13 Byte)     |
| r14<br>(Register 14)                      | r14d<br>(Register 14 Double word)      | r14w<br>(Register 14 Word)   | ∅                        | r14b<br>(Register 14 Byte)     |
| r15<br>(Register 15)                      | r15d<br>(Register 15 Double word)      | r15w<br>(Register 15 Word)   | ∅                        | r15b<br>(Register 15 Byte)     |

{{< admonition warning >}}
Seuls les registres `rax`, `rbx`, `rcx` et `rdx` possèdent deux registres de 8 bits. Tous les autres n'en possède qu'un.
{{< /admonition >}}

Néanmoins, cela ne change pratiquement rien à notre schéma de `rax` si nous l'adaptons pour `rdi` par exemple :
```
               rdi
 → → → → → → → → → → → → → → → → 
↑                               ↓ 
↑                edi            ↓
↑      → → → → → → → → → → →    ↓
↑     ↑                      ↓  ↓
↑     ↑            di        ↓  ↓
↑     ↑      → → → → → → →   ↓  ↓
↑     ↑     ↑             ↓  ↓  ↓
↑     ↑     ↑      dil    ↓  ↓  ↓
↑     ↑     ↑     → → →   ↓  ↓  ↓
↑     ↑     ↑    ↑     ↓  ↓  ↓  ↓
[ rdi [ edi [ di [ dil ]  ]  ]  ]
```

Théoriquement chaque registre peut prendre n'importe quelle valeur. Néanmoins, plusieurs conventions existent dans le langage assembleur et il est conseillé de les respecter.

Voici donc les fonctions implicites de chaque registre :

| Registre | Fonctionnalité                                                                                                                        |
| :------: | :------------------------------------------------------------------------------------------------------------------------------------ |
| rax      | Utilisé pour les opérations arithmétiques ainsi que pour stocker le numéro des appels systèmes et la valeur de retour de ces derniers |
| rbx      | Utilisé comme pointeur de données                                                                                                     |
| rcx      | Utilisé comme compteur notamment dans des boucles (for/while/do while)                                                                |
| rdx      | Utilisé pour les opérations arithmétiques, les opérations d'entrée/sortie et pour stocker le troisième argument d'un appel système    |
| rdi      | Utilisé pour pointer vers l'adresse d'une zone mémoire à copier et pour stocker le premier argument d'un appel système                |
| rsi      | Utilisé pour pointer vers l'adresse de où copier une zone mémoire et pour stocker le deuxième argument d'un appel système             |
| rbp      | Utilisé pour pointer à la base de la pile                                                                                             |
| rsp      | Utilisé pour pointer au sommet de la pile                                                                                             |
| r8       | Utilisé pour tout et pour stocker le cinquième argument d'un appel système                                                            |
| r9       | Utilisé pour tout et pour stocker le sixième argument d'un appel système                                                              |
| r10      | Utilisé pour tout et pour stocker le quatrième argument d'un appel système                                                            |
| r11      | Utilisé pour tout                                                                                                                     |
| r12      | Utilisé pour tout                                                                                                                     |
| r13      | Utilisé pour tout                                                                                                                     |
| r14      | Utilisé pour tout                                                                                                                     |
| r15      | Utilisé pour tout                                                                                                                     |

Un autre registre existe et a une fonctionnalité toute particulière. Il s'agit du registre `rip` qui est utilisé pour pointer vers l'adresse de la prochaine instruction à exécuter. Ce registre est donc modifié implicitement par le processeur à chaque instruction exécutée.

| Registre 64 bits                         | Registre 32 bits                       | Registre 16 bits             | Registre 8 bits (high) | Registre 8 bits (low) |
| :--------------------------------------: | :------------------------------------: | :--------------------------: | :--------------------: | :-------------------: |
| rip<br>(Re-extented Instruction Pointer) | eip<br>(Extended Instruction eXtended) | ip<br>(Accumulator eXtended) | ∅                      | ∅                     |

| Registre | Fonctionnalité                                                             |
| :------: | :------------------------------------------------------------------------- |
| rip      | Utilisé pour pointer vers l'adresse de la prochaine instruction à exécuter |

## Les flags

Un registre un peu spécial nommé `RFLAGS` indique l'état des processeurs à un instant T. Il permet ainsi de connaître l'état résultant de la dernière instruction exécutée par le processeur. Ainsi, les flags sont définis de manière implicite par les processeurs même si certains peuvent être définis via des instructions bien précises.

| Registre 64 bits              | Registre 32 bits           | Registre 16 bits | Registre 8 bits (high) | Registre 8 bits (low) |
| :---------------------------: | :------------------------: | :--------------: | :--------------------: | :-------------------: |
| rflags<br>(Re-extented Flags) | eflags<br>(Extended flags) | flags<br>(flags) | ∅                      | ∅                     |

Voici la liste de tous les flags existants ainsi que leur fonctionnalité :

| Bit   | Flag                                         | Fonctionnalité                                                                                                                           |
| :---: | :------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------------------------- |
| 0     | CF<br>(Carry Flag)                           | Indique si la dernière opération a générée une retenue sur le bit de poids fort                                                          |
| 2     | PF<br>(Parity Flag)                          | Indique si la dernière opération a générée un nombre pair de bits à 1 sur l'octet de poids faible                                        |
| 4     | AF<br>(Adjust Flag)                          | Indique si la dernière opération a générée une retenue sur le troisième bit                                                              |
| 6     | ZF<br>(Zero Flag)                            | Indique si le résultat de la dernière opération est 0                                                                                    |
| 7     | SF<br>(Sign Flag)                            | Indique si le bit de poids fort est à 1 après la dernière opération effectuée                                                            |
| 8     | TF<br>(Trap Flag)                            | Indique si le débogage en mode pas à pas est activé                                                                                      |
| 9     | IF<br>(Interrupt Flag)                       | Indique si les processeurs peuvent répondre à toutes les requêtes d'interruptions ou seulement à celles qui ne peuvent pas être ignorées |
| 10    | DF<br>(Direction Flag)                       | Indique si les adresses des chaînes de caractères doivent être incrémentées ou décrémentées                                              |
| 11    | OF<br>(Overflow Flag)                        | Indique si le résultat de la dernière opération est trop grand pour être stocker dans l'opérande de destination                          |
| 12/13 | IOPL<br>(Input/Output Privilege Level field) | Indique le niveau de privilège en I/O de la tâche courante                                                                               |
| 14    | NT<br>(Nested task Flag)                     | Indique si la tâche courante résulte d'une tâche parente                                                                                 |
| 16    | RF<br>(Resume Flag)                          | Controle que le débogage en mode pas à pas n'intervienne qu'une seule fois par instruction                                               |
| 17    | VM<br>(Virtual-8086 mode Flag)               | Indique si le processeur est en mode 8086 ou en mode protégé                                                                             |
| 18    | AC<br>(Alignment Check Flag)                 | Indique si l'alignement des références mémoires doit être verifié                                                                        |
| 19    | VIF<br>(Virtual Interrupt Flag)              | Image virtuelle de IF                                                                                                                    |
| 20    | VIP<br>(Virtual Interrupt Pending flag)      | Indique si une interruption est en attente                                                                                               |
| 21    | ID<br>(Identification Flag)                  | Indique si la tâche en cours peut utiliser l'instruction CPUID                                                                           |

{{< admonition >}}
Les bits 1, 3, 5, 15 et de 22 à 63 ne sont pas définis, car ils sont réservés. Leur utilisation et fonctionnement sont inconnus.
{{< /admonition >}}

## La stack

La stack est une zone de mémoire. Au moment du lancement d'une tâche, le système d'exploitation définit la taille de la stack. Cette taille ne pourra pas varier durant toute l'exécution de la tâche. A la fin de cette dernière la stack est supprimée.

La stack permet de stocker plusieurs valeurs et notamment les petites données necéssaires à la bonne exécution des fonctions. Cette dernière est également utilisée pour savoir ou aller une fois que la fonction courante est terminée.

De plus, la stack à une structure de type Last In First Out (LIFO). Cela signifie que la dernière valeur ajoutée à la stack sera la première à être retirée :

```
             ajoute une valeur    retire une valeur
                    → → → ------------ → → →
ajoute une valeur  ↑      | valeur 3 |      ↓  retire une valeur
       → → → ------------ ------------ ------------ → → →
      ↑      | valeur 2 | | valeur 2 | | valeur 2 |      ↓
------------ ------------ ------------ ------------ ------------
| valeur 1 | | valeur 1 | | valeur 1 | | valeur 1 | | valeur 1 |
------------ ------------ ------------ ------------ ------------
```

Chaque case de mémoire de la stack a une adresse. Les adresses de la stack sont un peu particulières puisque les éléments sont d'abord sauvegarder dans les adresses les plus hautes de la stack puis les adresses diminuent au fur et à mesure.

Nous pouvons reprendre le schéma ci-dessus pour illustrer cela :

```
----------------------
| adresses           |                ajoute une valeur    retire une valeur
----------------------                       → → → ------------ → → →
| 0x0000000000000000 |   ajoute une valeur  ↑      | valeur 3 |      ↓  retire une valeur
----------------------          → → → ------------ ------------ ------------ → → →
| 0x7777777777777777 |         ↑      | valeur 2 | | valeur 2 | | valeur 2 |      ↓
----------------------   ------------ ------------ ------------ ------------ ------------
| 0xffffffffffffffff |   | valeur 1 | | valeur 1 | | valeur 1 | | valeur 1 | | valeur 1 |
----------------------   ------------ ------------ ------------ ------------ ------------
```

Bien sûr, pour l'exemple, les adresses sont fictives. En réalité, certaines zones sont réservées ou déjà utilisées par le kernel par exemple. Néanmoins, cela nous montre que l'on peut comparer la stack à un compte à rebours pour l'utilisation de ses adresses.

## Les instructions assembleur

Le langage assembleur consiste en un enchaînement d'instructions qui permettent notamment d'effectuer des opérations mathématiques ou logiques, des déplacements de contenu d'une case mémoire à une autre ou encore de vérifier l'état de la pile (rappelez-vous du registre `RFLAGS`).

Voici une liste de quelques instructions assembleur :

| Instruction    | Fonctionnalité                                                                                 |
| :------------: | :--------------------------------------------------------------------------------------------- |
| add rax, 0x07  | ajoute 7 à rax                                                                                 |
| sub rax, 0x07  | soustrait 7 à rax                                                                              |
| push rax       | met le contenu de rax au sommet de la stack                                                    |
| pop rax        | met le contenu du sommet de la stack dans rax                                                  |
| mov rax, 0x07  | met 7 dans rax                                                                                 |
| lea rax, [rbx] | met l'adresse de rbx dans rax                                                                  |
| inc rax        | incrémente rax                                                                                 |
| dec rax        | décrémente rax                                                                                 |
| xor rax, rbx   | effectue un xor entre rax et rbx et stocke le résultat dans rax                                |
| and rax, rbx   | effectue un ET logique entre rax et rbx et stocke le résultat dans rax                         |
| ror rax, 0x07  | effectue un décalage logique de bit de 7 vers la droite, sur le contenu de rax                 |
| rol rax, 0x07  | effectue un décalage logique de bit de 7 vers la gauche, sur le contenu de rax                 |
| cmp rax, rbx   | vérifie si rax et rbx sont égaux                                                               |
| jmp [rax]      | va à l'adresse contenue dans rax                                                               |
| call [rax]     | effectue l'instruction "push rip" suivi de l'instruction "jmp [rax]"                           |
| ret            | effectue l'instruction "pop rip"                                                               |
| je [rax]       | va à l'adresse contenue dans rax si les deux valeurs comparées précédemment sont égales        |
| jne [rax]      | va à l'adresse contenue dans rax si les deux valeurs comparées précédemment ne sont pas égales |
| jz [rax]       | va à l'adresse contenue dans rax si l'opération effectuée précédemment est égale à 0           |
| jnz [rax]      | va à l'adresse contenue dans rax si l'opération effectuée précédemment n'est pas égale à 0     |

{{< admonition >}}
Les instructions commençant par `j` sont dépendantes de l'instruction précédente excepté l'instruction `jmp`. Il existe énormément d'instructions de ce type, mais ici, seuls quelques exemples ont été donnés.
{{< /admonition >}}

## Les syscalls

Les syscalls sont des appels systèmes. A travers ces derniers, un programme peut demander au système d'exploitation d'effectuer une tâche.

Une liste des syscalls pour un Linux 64 bits est disponible sur [https://syscalls.w3challs.com/?arch=x86_64](https://syscalls.w3challs.com/?arch=x86_64).

{{< admonition tips>}}
Le mot clé `syscall` permet de lancer l'exécution d'un syscall une fois que tous les paramètres ont été indiqués dans les registres appropriés.
{{< /admonition >}}

## Les termes little-endian et big-endian

Les architectures des processeurs définissent si ce dernier utilise le little-endian ou le big-endian. Ces termes définissent l'ordre dans lequel les bits sont stockés au sein d'un octet. Ainsi, en little-endian les bits de poids faible sont stockés en premier tandis qu'en big-endian ce sont les bits de poids fort qui sont d'abord stockés.

De cette manière, la chaîne `example` s'écrira différemment entre une architecture en little-endian ou une architecture en big-endian :
```
Little endian : 0x656c706d617865 (elpmaxe)
Big endian : 0x6578616d706c65 (example)
```

La plupart des architectures sont de type little-endian.

## Les sections d'un programme assembleur

Un programme assembleur est découpé en plusieurs sections. La section `.text` contient le code, la section `.data` contient les données et enfin la section `.bss` contient les variables.

La différence entre la section `.data` et `.bss` est fine. La section `.bss` sera virtuellement remplis de 0. Ainsi, toutes les variables déclarées sans être initialisées doivent être dans la section `.bss` tandis que toutes les autres doivent être dans la section `.data`.

## Hello world!

Avec toutes ces informations, nous pouvons maintenant comprendre un programme `Hello world!` en assembleur :
```nasm
section .data
    helloMsg db "Hello, world!", 10 ;on déclare et on initialise notre chaîne de caractères suivi d'un retour à la ligne
    helloSize equ $-helloMsg        ;on déclare et on initialise dynamiquement la longueur de notre chaîne de caractères

section .text
    global _start                   ;on indique le point d'entrée du programme qui est "_start" par défaut

_start:
    mov rax, 0x01                   ;on indique à rax le numéro du syscall que l'on veut (ici, celui de "write")
    mov rdi, 0x01                   ;on indique à rdi le premier argument de "write" qui est le numéro du file descriptor (ici, STDOUT)
    mov rsi, helloMsg               ;on indique à rsi le deuxième argument de "write" qui est l'adresse de la chaîne à afficher
    mov rdx, helloSize              ;on indique à rdx le troisième argument de "write" qui est la taille de la chaîne à afficher
    syscall                         ;on exécute le syscall "write(1, helloMsg, helloSize)"
    mov rax, 0x3c                   ;on indique à rax le numéro du syscall que l'on veut (ici, celui de "exit")
    mov rdi, 0x00                   ;on indique à rdi le premier argument de "exit" qui est le code de retour
    syscall                         ;on exécute le syscall "exit(0)"
```

Afin d'exécuter ce programme, il faut mettre le code ci-dessus dans un fichier `.asm` puis, exécuter les commandes suivantes :
```shell
nasm -f elf64 <nomDuFichier>.asm
ld <nomDuFichier>.o -o <nomDuFichier>
./<nomDuFichier>
```

Le programme affiche simplement `Hello, world!` dans un terminal.

Merci à [GovanifY](https://govanify.com/) pour la relecture et la correction sur certaines parties techniques.
