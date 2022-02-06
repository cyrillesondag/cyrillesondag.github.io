---
title: "Systemd - partie 1"
date: 2021-10-17T10:34:56+02:00
slug: ""
description: "Init system de initV a systemd"
keywords: [ "linux", "systemd", "init system" ]
draft: false
tags: [ "linux", "systemd" ]
math: false
toc: true
---


{{< figure src="Jean-François_Millet_-_Gleaners_-_Google_Art_Project_2.jpg" link="Jean-François_Millet_-_Gleaners_-_Google_Art_Project_2.jpg" caption="Jean-Francois Millet - Les Glaneuses (1857), huile sur toile, 83,5 × 110 cm, Paris, musée d'Orsay." >}}

Je vais m'attacher dans cette partie introductive à expliciter la séquence de boot sous Linux pour m'arrêter au lancement du processus d'init. 
Puis faire la distinction entre systemd et SysV init sans rentrer dans les détails d'implémentation, puis revenir rapidement sur le débat autour de l'adoption du systemd.

Les détails techniques et l'utilisation de systemd en lui-même feront les sujets d'articles suivants.

## Historique
Systemd a été développé à l'initiative de Lennard Poettering (la première version date de 2010 selon Wikipedia) comme étant le remplaçant - entre autres - de l'historique SysV init.

Promu par Redhat, il a fini par s'imposer comme un standard de l'industrie dans les distributions majeures Linux, malgré les critiques encore nombreuses.

Systemd est la contraction de system daemon.

## Du boot à l'init

Avant de parler de l'init il me semble important d'expliquer les étapes précédentes pour bien comprendre le contexte et le rôle de ce processus.

Donc lors du démarrage, la séquence est la suivante : 

Le bootloader va sélectionner et lancer l'image du kernel au format bzImage (pour boot executable image), elle est située sur la partition de boot généralement sous le nom ```vmlinux.**```.

Cette image est compilée et généralement assez minimale, c'est-à-dire qu'elle contient peu d'éléments statiques dans le but d'occuper un minimum d'espace disque et de faciliter son chargement et sa distribution.
Grâce à cela une même image peut être utilisée par exemple aussi bien sur un téléphone portable que sur un serveur dernière génération (pourvu qu'ils utilisent la même architecture). 

Pour réaliser le démarrage, il faut pouvoir accéder aux fichiers de configuration qui peuvent se trouver sur différents types de support : disque, partition, réseaux...
Pour y accéder il est donc nécessaire de charger les différents modules / drivers correspondants.

Pour ce faire il existe principalement deux méthodes :
- Soit pré-compiler dans l'image du kernel l'ensemble des drivers requis (mais sacrifier les avantages vus plus haut).
- Soit passer par une image temporaire contenant ce qui est nécessaire au démarrage du système.

Parmi la multitude de tâches d'initialisations (vm, console, horloge...) que le kernel va réaliser, on va s'intéresser à quelques-unes en particulier.

#### Instanciation du rootfs.

Le rootfs est un filesystem un peu spécial, c'est la base de tous les futurs filesysteme, il est stocké en mémoire et est présent dès les premières étapes du démarrage du kernel, ne peut être démonté bien qu'il soit rarement utilisé après la phase d'init.

Il existe sous deux types :
- RAMFS qui est un simple "page cache" en mémoire dont aucun élément ne peut être persisté. 
- TMPFS qui à l'inverse de RAMFS permet de limiter l'espace utilisé en mémoire et l'écriture sur la SWAP.

Le choix de l'un ou l'autre se fait à la compilation du kernel.

#### La décompression de l'initrd

L'image temporaire du rootFS évoquée plus haut s'appelle l'initrd (initial ram disk) en hommage à l'ancien fonctionnement (< 2.6) qui émulait un périphérique disque. 
C'est une simple archive compressée d'un filesystem au format cpio (un gz "amélioré").
Elle a l'avantage d'être plus facilement manipulable que l'image kernel, car non compilée. 
Il n'est donc pas nécessaire d'avoir un gcc ou des headers installés pour la générer ou la modifier.
Elle est d'ailleurs régulièrement mise à jour au cours de la vie du système (souvent par les packages manager).

> on peut facilement lister le contenu grace a la commande ```lsinitramfs``` ou la mettre à jour via ```update-initramfs```

Cette archive est aussi stockée dans la partition de boot et passée en paramètre au kernel via le paramètre ```initrd=```, qui va l'extraire dans le rootFS.

Enfin - et c'est là que process d'init à proprement parler commence - le kernel va appeler l'exécutable ```/init``` (ou l'exécutable spécifié via ```rdinit=```) sur le rootFS.

L'initrd a pour but d'inclure les différents drivers et de monter le root device spécifié par le paramètre ```root=``` dans le dossier ```/root```.
Ensuite, il va supprimer les autres fichiers du rootFS puis appeler la fonction ```pivot_root``` pour transformer le repertoire ```/root``` en ```/``` (à la manière d'un chroot).
Bien souvent (c'est le cas de systemd) ce process va re-exécuter une autre phase d'init pour prendre en charge la suite de l'initialisation du système après le montage du root.

#### Le montage "/root"

En cas d'absence de l'initrd le kernel va appeler la méthode ```prepare_namespace```(qui rempli le même contract que l'exécutable ```/init```) c'est-a-dire tenter le monter le périphérique spécifié par le paramètre ```root=``` dans le répertoire ```/root```, puis faire basculer ce répertoire à la racine.

Enfin il va appeler l'un des processus d'init suivant (```/sbin/init```, ```/etc/init```, ```/bin/init```, ```/bin/sh```)


> _main.c_ 'kernel_init()'
```c
if (ramdisk_execute_command) {
    ret = run_init_process(ramdisk_execute_command);
    if (!ret)
        return 0;
    pr_err("Failed to execute %s (error %d)\n",
           ramdisk_execute_command, ret);
}

/*
 * We try each of these until one succeeds.
 *
 * The Bourne shell can be used instead of init if we are
 * trying to recover a really broken machine.
 */
if (execute_command) {
    ret = run_init_process(execute_command);
    if (!ret)
        return 0;
    panic("Requested init %s failed (error %d).",
          execute_command, ret);
}
if (!try_to_run_init_process("/sbin/init") ||
    !try_to_run_init_process("/etc/init") ||
    !try_to_run_init_process("/bin/init") ||
    !try_to_run_init_process("/bin/sh"))
    return 0;

panic("No working init found.  Try passing init= option to kernel. "
      "See Linux Documentation/admin-guide/init.rst for guidance.");
```


## L'init
Comme c'est le premier processus lancé par le kernel, il porte donc logiquement le numéro ```PID=1```.

Ce processus est un peu différent des autres :
- Ne peut être tué.
- N'a pas de parent ```PPID=```.
- Est l'ancêtre de tous les autres processus.
- Est chargé d'initialiser tout un nombre de ressources (disque, terminaux, réseau, interface graphique…).

> Sous linux les processus sont tous issus de la methode ```fork()``` ou dérivés.
> 
> Tous les processus enfants héritent d'un certain nombre d'attributs de leur parent : user, session, cgroup, mémoire partagé...
> 
> L'init a donc un rôle primordial dans la vie du système.


## SysV init 

Pour bien comprendre ce que systemd apporte il faut le comparer à son prédécesseur SysV init (j'ai choisi initV uniquement parce que c'était le plus répandu et que je l'ai un peu utilisé).

SysV init est basé sur les run-levels qui sont au nombre de 5 (presque) :
- 0 - réservé - Arrêt (halt)
- 1 - Mode utilisateur unique avec les filesystems montés.
- 2..5 Mode multi-utilisateur, les significations varies suivant les distributions, mais correspond à l'état opérationnel de la machine.
- 6 - réservé - Redémarrage

Il faut d'ailleurs comprendre le nom SysV init comme étant "systeme 5 init" qui correspond aux 5 run-level de l'état d'un système.

Lors de l'init le system va passer d'un run-level à l'autre jusqu'à arriver au ***default level*** (qui peut varier suivant les distributions...) qui correspond au mode nominal de fonctionnement (par exemple une interface graphique pour un desktop ou un terminal pour un serveur).

En cas d'erreur du level X le level X+1 n'est pas appelé et l'initialisation est marquée en échec.

Chacun des levels agit comme un point de synchronisation, c'est-a-dire qu'il va déclencher un nombre d'actions à une étape précise du démarrage du système. 

> Par exemple si un daemon "A" nécessite une interface réseau ou filesystem locaux, il a tout intérêt à se déclencher à un run-level supérieur à 1.

## Scripts SysV init

SysV init est basé sur un ensemble de scripts, qui doivent répondre à des conventions, certaines peuvent varier suivant les distributions Linux utilisées.

Prenons l'exemple très simple d'un script de lancement d'un daemon :

```bash
#!/bin/sh #(1)
# 
# Demonstrate creating your own init scripts 
# chkconfig: 2345 91 64 (2) 
### BEGIN INIT INFO (3)
# Provides: Welcome 
# Required-Start: $local_fs $all 
# Required-Stop: 
# Default-Start: 2345 
# Default-Stop: 
# Short-Description: Display a welcome message 
# Description: Just display a message. Not much else. 
###END INIT INFO 

# Source function library. 
. /etc/rc.d/init.d/functions #(4)

lock_file=/var/lock/subsys/myservice

start()  { 
   touch "$lock_file" 
   echo "Starting service" 
   sleep 2
 } 

case "$1" in  #(5)
      start) 
       start
        ;; 

      stop)
       rm -f  "$lock_file" 
       echo "Service is shutting down"  
       sleep 2 
        ;; 

     status)
       if [[ -f "$lock_file" ]]; then
         echo "Service looks good"
       else
         echo "Service not started !!"
       fi
        ;; 

         *)
       echo $"Usage: $0 {start|stop|status}" 
       exit 5 
esac 
exit $? #(6)
```
Il est composé :
- (1) D'une déclaration de l'interpréteur (ici shell)
- Ensuite une série de commentaires qui n'en sont pas :
  - (2) chkconfig qui permet de définir les levels d'exécution et le niveau de priorité
  - (3) LSB Headers qui contient des informations sur ordonnancement du service qui peut être éventuellement utilisé
- (4) L'import des fonctions communes /etc/rc.d/init.d/functions
- (5) Un switch case en fonction des arguments 'start' / 'stop' / 'status' 
- (6) Un code de retour
  
***PROS*** :
- Offre un nombre de fonctionnalités assez completes : ckconfig, lsb-headers.
- Une grande flexibilité : c'est un script shell on fait 'ce qu'on veut'.
- Un début de standardisation (avec les fonctions, lsb headers, checkconfig).

***CONS*** : 
- Très verbeux, répétitif et difficilement extensible.
- Repose uniquement sur des conventions (start, stop, status...).
- Compliqué à réaliser.
- Peut avoir de comportements différents selon le context d'appel du script.

D'un point de vue utilisateur, on remarque bien que ce n'est pas standardisé (avec ses avantages et ses inconvénients) et en pratique demande pas mal d'expertise en script pour réaliser des choses assez semblables (gérer les sorties standards, changer d'utilisateur, lancer un daemon, execution unique...) 

La ou SysV init se résume à sequencer l'exécution de scripts (en schématisant), systemd se base sur de la description de configuration.

## Service unit systemd

Ces configurations sont écrites au format ini et sont appelées "unit file" ou "unit" (mais ce n'est pas exact, car ça désigne les objects manipulés par systemd et non les fichiers).

Il existe plusieurs types d'unit files qui répondent à différents cas d'utilisations (service, montage...), chacune est suffixée par son type. 

L'équivalent systemd du script ci-dessus est une unit file de type "service" que l'on pourrait adapter de la sorte :
```ini
[Unit]
Description=Service

[Service] 
Type=simple
ExecStart=/usr/bin/sleep 2
ExecPreStop=/usr/bin/sleep 2 

[Install]
Wants=multi-users.target
```

La première chose que l'on remarque est que le fichier systemd n'est pas un script, il n'a donc pas d'utilité en dehors du contexte de systemd.

Deuxième chose, les attributs de configuration sont normées, ils correspondent à une série de propriétés interprétées par le programme.
Ce n'est plus nous qui réalisons les actions, c'est systemd qui le fait à notre place.

Le script de base est extrêmement simple, et n'a pas vraiment de sens (encore moins avec systemd) mais met bien en evidence toutes les choses qui sont inclus de base dans systemd, et la difference entre les deux approches. 

L'avantage ici d'être dans un "framework" est qu'il prend nativement en charge toutes les tâches usuelles sans avoir à les redéfinir nous même.

- Un service est un unique, pour un meme nom (au sein d'une instance systemd). Nous n'avons donc pas besoin de lock de synchronisation (il est possible de le variabiliser via le templating).
- Les états start / status / stop / restart sont implicites et commun à tous les services. Il faut noter que le status a souvent une signification un peu différente des scripts initV qui réalisent souvent des tâches de vérification assez complexes. Sous systemd le status ne reporte généralement uniquement l'état du ```PID``` déterminé comme principal.
- Là où l'on devait spécifier via les lsb-headers l'ordonnancement (after $local_fs) fait ici partie de la valeur par défaut (cf ```defaultdependencies```) qui doit répondre à la majorité des cas d'utilisations des daemons de haut niveau.
- Enfin on retrouve un équivalent des run-level avec l'attribut ```Target=multi-users.target``` qui sert ici également de point de synchronisation lors du démarrage.


## Le mal aimé

On a beaucoup reproché a systemd d'être envahissant, et de briser le principe "un programme doit faire UNE chose et la faire bien".

Oui systemd est envahissant, mais principalement par ce que c'est un init system, une des pièces principales des systèmes UNIX. Il est de ce fait chargé de réaliser une multitude d'actions très différentes (monter les fs, mettre en place les interfaces réseaux...).
Ce qui était "caché" sous des scripts est ici rendu apparent.

Les défenseurs rétorquent souvent que systemd est modulaire - ce qui est vrai - mais cela ne me semble pas être un argument valable, les composants peuvent faire partis de different projets/module (journald, networkd, nspawn...) ils font tous partis d'un ensemble coherent et ont tous vocation à fonctionner ensemble.

À mon avis la principale résistance est que systemd a imposé ses standards (en termes de nomenclature, arborescence, packaging et autres) aux seins des distributions. 

J'y vois également la réaction à la professionnalisation inévitable de l'écosystème Linux, avec la disparition progressive de l'esprit pionnier qui perdure dans les communautés de passionnés organisées autour des distributions / projets OSS.
(Il faut aussi bien avouer que la communauté Linux ADORE les dramas et trouver des occasions de s'écharper).

Systemd n'est certainement pas idéal, mais il est "constant". 
Il a indéniablement apporté beaucoup de bénéfices : facilitation du packaging, une meilleure portabilité, une qualité accrue et plus homogène, une facilité d'accès aux fonctions avancés du kernel...

Bref, il ne s'agit pas de défendre systemd mais bien de mettre en avant les bénéfices qu'il apporte.
