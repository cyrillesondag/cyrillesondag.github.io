---
title: "Systemd (1/x)"
date: 2021-10-17T10:34:56+02:00
slug: ""
description: "systemd - presentation et concepts"
keywords: ["linux", "systemd", "init system"]
draft: true
tags: ["linux", "systemd"]
math: false
toc: true
---

## Historique
Systemd est developpé par Lennard Poettering (la premiere release date de 2010 selon wikipeadia) comme etant pour etre le remplacant de l'historique SysV init.

Promu surtout par Redhat il a finit par simposer comme un standard de l'industrie dans les principales distributions Linux malgres les critiques nombreuses critiques que le projet a pu susciter (j'y reviendrai car les critiques sont interessantes).

Systemd est la contraction de system daemon.

## C'est quoi un init system

Sous Linux l'init system est le processus d'initialisation de l'espace utilisateur.

Apres le sequence de boot, le bootloader va fournir un kernel une archive compressée au format CPIO

Lors de la sequence de lancement le bootloader (type grub, lilo) ou uefi (je dois avouer ne pas connaitre la distinction entre les deux) va initialisé un espace memoire RAM dédie : soit un RAMFS, soit un TMPFS (la difference fondamentale entre les 2 est que le TMPFS peut etre serialisé alors que l'autre est uniquement volatile).
Puis il va alouer un dnode "/" et deux entrée "." et ¨.." qui pointent sur le répertoire.
Ensuite il va extraire d'une archive compressée de l'image initrd le systeme de fichier original le rootfs 

Après la sequence complete d'initialisation, le kernel va appeler l'executable ```/sbin/init``` avec l'utilisateur ```root```. 
Ce processus est chargé initialiser les différentes ressources materiels et logiciel disponibles pour l'utilisateur. Cela va de par exemple du : montage des disques, configuration des interfaces réseaux, terminaux, environement graphique... etc.

Comme c'est le premier executable appelé il porte donc logiquement le ```PID=1```.

Il est d'autant plus important que tous les processus existants sont issus de l'init via la méthode POSIX ```init()``` (et apparentés). 
Les processus enfants héritent de certains attributs de son parent (droits, utilisateur).

L'init est donc le seul processus à déroger a cette regle de parent / enfant et revet donc une importance particulière.

## Quoi de neuf ?

Pour bien comprendre ce qu'a apporté systemd il faut à mon avis le comparer a son remplacent SysV init (c'est d'ailleurs les 2 seuls systeme que je connaisse).

SysVinit est basé sur les run-levels qui sont au nombre de 5 (presque) :
- 0 - reservé - Arret (halt)
- 1 - Mode utilisateur unique avec les filesystems montés.
- 2..5 Mode multi-utilisateur, les significations varies suivant les distributions mais correspond a l'état opérationel de la machine.
- 6 - reservé - Redémarrage

Il faut d'ailleurs comprendre le mon SysV init comme étant "systeme 5 init" qui correspond aux 5 run-level de l'état d'un systeme.

Lors de l'init le system va passer d'un run-level à un autre de facon séquentiel jusqu'à arriver au level default.

En cas d'une erreur fatal du level X le level X+1 n'est pas appelé et l'initialisation est marquée en erreur.

Chacun des levels agit comme point de synchronisation entre les différentes étapes du lancement. Ainsi si les filesystem locaux sont montés au level 1 les participants du level 2 peuvent compter sur leur presence lors de leur lancement.

SysV init est basé sur un ensemble de scripts, qui doivent répondre à des conventions (certaines peuvent varier suivant les distributions) 

Prenons l'exemple du squelette du script de lancement d'un daemon classique : 
```bash
#!/bin/sh 
# 
# Demonstrate creating your own init scripts 
# chkconfig: 2345 91 64 
### BEGIN INIT INFO 
# Provides: Welcome 
# Required-Start: $local_fs $all 
# Required-Stop: 
# Default-Start: 2345 
# Default-Stop: 
# Short-Description: Display a welcome message 
# Description: Just display a message. Not much else. 
###END INIT INFO 

# Source function library. 
. /etc/rc.d/init.d/functions

broken_lock_file_for_centos=/var/lock/subsys/myservice

start()  { 
   touch "$broken_lock_file_for_centos" 
   echo "Welcome to Linux message by Sahil Suri" 
   sleep 2
 } 

case "$1" in  
      start) 
       start
        ;; 

      stop)
       rm -f  "$broken_lock_file_for_centos" 
       echo "System is shutting down"  
       sleep 2 
        ;; 

     status) 
       echo "Everything looks good"
        ;; 

         *)
       echo $"Usage: $0 {start|stop|status}" 
       exit 5 
esac 
exit $?
```
Il est conpososé de :
- la déclaration de l'interpréteur (ici shell)
- ensuite une série de commentaires qui n'en sont pas mais des metadonnés:
  - chkconfig qui permet de définir les levels d'execution et le niveau de priorité
  - LSB Headers qui contient des informations sur ordonnancement du service qui peut etre éventuellement utilisé
  - l'import des fonctions communes /etc/rc.d/init.d/functions
  - quelques fonctions
  - un switch case en fonction des arguments 'start' / 'stop' / 'status' 
  
PROS :
- Offre un nombre de fonctionnalités assez completes : ckconfig, lsb-headers.
- Une grande flexibilité : c'est un script shell on fait ceux qu'on veut avec.
- On remarque deja une volonté de standardisation des scripts (avec les fonctions).

CONS : 
- Tres verbeux.
- Beaucoup de conventions (start, stop, status.. etc).
- Compliqué à réaliser.
- Peut avoir de comportements différents selon le context d'appel du script

l'équivalent en systemd de ce service (bien pauvre) pourrait etre quelque chose comme ca :
```ini
[Unit]
Description=Just display a message. Not much else.

[Service] 
Type=simple
ExecStart=/usr/bin/echo "Welcome to Linux message by Sahil Suri"
ExecPreStop=/usr/bin/echo "System is shutting down" 

[Install]
Wants=multi-users.target
```

Les deux services sont simples (et assez peu réalistes) mais ils servent notre exemple.

Le premiere chose que l'on remarque est que le fichier systemd est un fichier de configuration et pas un script.
Il n'a donc pas d'utilité en dehors du contexte de systemd.

Deuxième chose est que les attributs de configuration sont normées, ils correspondent a une serie de propriete interpreté par le programme. Ce n'est plus nous qui realisons les actions, c'est systemd qui le fait a notre place, on se contente de lui decrire quoi faire.

Enfin certaines choses sont implicites:
- un service systemd est une instance unique on a donc pas besoin de lock comme dans le cas precedent.
- les états start / status / stop sont aussi implicite et correspondent au cycle de vie de tout les services (le status a souvent une signification differentes).
- la ou l'on devait specifier via les lsb-headers l'ordonancement (after $local_fs) fait aussi partie de la valeur par defaut d'un service.
- enfin le run-level trouve un equivalent dans l'attribut mnulti-users.target.