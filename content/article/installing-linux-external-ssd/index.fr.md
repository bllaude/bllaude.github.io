---
title: "Installation de Linux sur un SSD externe"
date: 2022-08-03T18:03:48+08:00
draft: false

categories: ['Tutoriel']
tags: ['Linux', 'Tutoriel']
toc: false
author: ""
---
Je souhaitais installer Linux sur un SSD portable et le rendre amorçable depuis mon ordinateur portable, ce qui nécessitait la prise en charge de l'UEFI. J'ai mis au point ma propre méthode, car je n'ai pas trouvé de meilleure solution en ligne.

<!--more-->

Je vais tout d'abord aborder la différence entre le BIOS, l'EFI et l'UEFI. Il s'agit dans tous les cas de méthodes permettant de démarrer un système d'exploitation. Le BIOS est le mode de démarrage traditionnel que l'on trouve sur les ordinateurs plus anciens. Les ordinateurs plus récents, tels que les MacBook, utilisent l'UEFI (qui succède à l'EFI), ce qui nécessite quelques étapes supplémentaires par rapport au BIOS lors de l'installation du système d'exploitation.

Pour qu'une installation Linux UEFI soit amorçable, le disque doit disposer d'une partition d'amorçage EFI, c'est-à-dire une partition formatée en FAT32 d'au moins 200 Mo. Pour effectuer une installation UEFI, le disque d'installation doit également être en mode UEFI (sinon, il ne permettra pas d'installer une version UEFI de Linux).

## Approche
La plupart des méthodes que j'ai trouvées consistaient à démarrer le système à l'aide d'un Live CD et à installer le système d'exploitation sur le disque dur externe connecté. Cependant, j'ai constaté que cela écrasait le chargeur d'amorçage du disque interne de l'ordinateur ; après l'installation, il fallait donc réparer ce disque pour que la machine puisse à nouveau démarrer.

Ma solution consiste à créer une machine virtuelle (à l'aide de VirtualBox) pour démarrer le Live CD Linux, puis à installer le système d'exploitation sur le disque dur externe au sein de cette machine virtuelle. Cette approche n'aura aucune incidence sur le disque interne (la machine virtuelle étant entièrement isolée) et devrait fonctionner avec la plupart des systèmes d'exploitation Linux.

## Étapes
1. Installez [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
2. Installez le *Oracle VM VirtualBox Extension Pack*. Il est nécessaire pour connecter des périphériques `USB 2.0` et `USB 3.0` à la machine virtuelle.
3. Si votre système d’exploitation hôte est Linux, assurez-vous que votre utilisateur fait partie du groupe `vboxusers` à l’aide de la commande `sudo usermod -aG vboxusers $USER`, puis redémarrez pour que les modifications soient prises en compte.
4. Téléchargez la distribution Linux de votre choix. Assurez-vous qu’il s’agit d’une version 64 bits pour la prise en charge de l’UEFI.
5. Créez une nouvelle machine virtuelle dans VirtualBox
6. N’ajoutez pas de disque dur virtuel, car nous allons monter le disque dur externe dans la machine virtuelle pour qu’il serve de disque.
7. Accédez aux paramètres de la machine virtuelle et activez la prise en charge EFI (sous « Système > Activer EFI »).
8. Sous « USB », sélectionnez le contrôleur USB ; essayez d’abord « USB 3.0 » et, si cela ne fonctionne pas, essayez « USB 2.0 ».
9. Démarrez la machine virtuelle ; lorsque vous y êtes invité, indiquez l’image ISO à utiliser,
10. Montez le disque dur USB en cliquant sur l’icône USB en bas de la fenêtre de VirtualBox et en sélectionnant le périphérique USB (si vous ne voyez pas le périphérique, essayez de changer de contrôleur USB dans les options, voir l’étape 8).
11. Lancez le programme d’installation comme d’habitude.
12. Lors du partitionnement du disque, assurez-vous d’avoir au moins 3 partitions : « EFI Boot Partition » (FAT32, au moins 200 Mo), / (point de montage racine, EXT4), Swap (dont la taille doit être proche de la capacité de la mémoire vive de votre ordinateur).
13. Procédez à l’installation comme d’habitude.
14. Éteignez la machine virtuelle et l’ordinateur, puis essayez de démarrer à partir du disque dur externe sur l’ordinateur.

## Résultat
- Système amorçable avec prise en charge UEFI, fonctionnant sur différents
- N'apporte aucune modification au disque dur interne de l'ordinateur lors de l'installation
- Testé avec Ubuntu