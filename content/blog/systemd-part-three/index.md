---
title: "Systemd - partie 2"
date: 2021-11-20T10:34:56+02:00
slug: ""
description: "System de configuration"
keywords: [ "linux", "systemd", "init system" ]
draft: true
tags: [ "linux", "systemd" ]
math: false
toc: true
---

{{< figure src="Gustave_Caillebotte_-_Perissoires_sur_yerres.jpg" link="Gustave_Caillebotte_-_Perissoires_sur_yerres.jpg" caption="Gustave Caillebotte - Périssoires sur l'Yerres (1877), huile sur toile, 103 × 155 cm, Milwaukee (USA), Milwaukee Art Museum." >}}

\
Nous avons vu dans le précédent article quelques differences entre SysV init et systemd, ainsi que le mécanisme d'initialisation de l'espace utilisateur par le kernel.

Dans cet article, je vais aborder plus en détail l'architecture et le fonctionnement interne de systemd.

## Architecture 

Le cœur de systemd est basé sur UDev et DBus qui permettent de mettre en place une approche évènementielle, les CGroup pour l'encapsulation des processus et le contrôle des ressources et EBPF pour les métriques.


{{< figure src="Systemd-components.png" link="Systemd-components.png" caption="Architecture de systemd" >}}

Il se base sur plusieurs principes forts que je vais tenter de détailler 

### 1 Retarder l'initialisation

Pouvoir lancer les services uniquement quand ils sont nécessaires et éviter d'engorger le démarrage du système.

Il ne faut garder à l'esprit qu'il a été pensée pour répondre a plusieurs cas d'utilisation assez différents, pour citer les principaux :
- les serveurs
- les stations de travail
- les mobiles

Si le temps de démarrage n'est (souvent) pas crucial pour un serveur ce n'est pas le cas pour une station de travail ou pour un mobile.  

Pour arriver a cela plusieurs types de _units_ sont chargées uniquement de l'activation des services lors de leur première utilisation.
C'est le cas pour l'activation par socket, connexion réseau, timer ou encore par accès aux fichiers (inotify)

Cela permet de ne pas démarrer un service trop tôt, où lorsqu'il n'est pas nécessaire.

Pour réaliser cela systemd utilise plusieurs méthodes d'activation :
#### Socket.

Le principe est assez simple et est largement inspiré de inetd.
Des buffers de sockets sont alloués automatiquement au démarrage pour chaque service. 
Ils se remplissent jusqu'a une certaine taille au dela de laquelle l'écriture est impossible, dans ce cas le client se met alors en attente et l'appel devient bloquant (c'est la meme chose quand il attend une réponse).
Au premier message le service consommateur est démarré en parallèle, les sockets lui sont transmis en paramètres. En cas d'échec, il peut être relancé sans perte de messages.

Ainsi contrairement a inetd, plusieurs types de socket sont supportés : UNIX, INET, named pipes, netlink...

Attention le passage de socket n'est pas un comportement "natif" sous Linux, il faut donc que le service consommateur soit en mesure de traiter les sockets passés en paramètre (c'est heureusement le cas de nombreux serveurs web entre autres).

Enfin grâce à ce comportement, non seulement on peut activer un service uniquement au premier appel (et éviter d'avoir à le démarrer trop tôt), 
mais en plus peut également briser les chaines de dépendances et paralléliser l'activation de services.

Par exemple pour un service qui nécessite syslog, ils peuvent être lancés en parallèle sans attendre l'initialisation complète de ce dernier. Les messages syslog seront simplement mis en attente dans le buffer avant d'être dépilé.


#### UDev
UDev est un daemon qui permet d'exposer dans l'espace utilisateur les périphériques (il permet notamment de gérer le hotplug). 

C'est un "simple" daemon qui permet d'écouter les évènements publiés par le kernel (netlink), de lire les informations du sysfs et de déclencher les règles spécifiques écrites par un administrateur.

Ce projet a acquis une telle importance qu'il fait maintenant partie à part entière de systemd.

Prenons un exemple concret de comment utiliser udev avec systemd:

Tout d'abord nous pouvons écouter les évenement publiés par le kernel et udev ainsi :
```shell
udevadm monitor
```
Qui donne ce résultat lors de l'activation de bluetooth sur mon portable
```
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

KERNEL[88503.920800] change   /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/bluetooth/hci0/rfkill0 (rfkill)
UDEV  [88503.926695] change   /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/bluetooth/hci0/rfkill0 (rfkill)
KERNEL[88521.147695] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/bluetooth/hci0/hci0:256 (bluetooth)
UDEV  [88521.151331] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/bluetooth/hci0/hci0:256 (bluetooth)
KERNEL[88522.280203] add      /devices/virtual/input/input24 (input)
KERNEL[88522.280341] add      /devices/virtual/input/input24/event18 (input)
UDEV  [88522.282180] add      /devices/virtual/input/input24 (input)
UDEV  [88522.319399] add      /devices/virtual/input/input24/event18 (input)
```
La première partie concerne la mise en route du récepteur bluetooth, la seconde la connexion du casque audio.
On voit donc apparaitre un nouveau device sous l'arborescence ```/devices/virtual/input/input24```.

Ensuite on cherche à écrite une règle udev qui va s'activer uniquement pour ce périphérique, pour cela il nous faut donc chercher ses attributs discriminants que l'on peut voir à l'aide de :

```shell
udevadm info -a /sys/devices/virtual/input/input24
```
NB: les chemins UDev sont relatifs aux repertoires ```/sys``` ou ```/dev/```


Je serai bien incapable d'expliquer la signification de tous les attributs pour contre on peut en reconnaitre quelque uns, par exemple le nom et son adresse :  
```
  KERNEL=="input24"
  SUBSYSTEM=="input"
  DRIVER==""
  ....
  ATTR{inhibited}=="0"
  ATTR{name}=="LG-TONE-FP9 (AVRCP)"
  ATTR{phys}=="0c:dd:24:30:06:00"
  ATTR{power/async}=="disabled"
  ATTR{power/control}=="auto"
  ....
```

Ensuite avec ses informations on est en mesure d'écrire une règle appropriée pour ce materiel.
```
SUBSYSTEM=="input",  ATTR{name}=="LG-TONE-FP9 (AVRCP)", ATTR{phys}=="0c:dd:24:30:06:00", TAG+="systemd", ENV{SYSTEMD_ALIAS}+="/sys/devices/virtual/input/lg_tone_fp9" 
```

Les premiers éléments servent à sélectionner les attributs dans les événements UDev qui vont déclencher la règle UDev (le system, l'attribut mom et son adresse).
La partie importante pour l'intégration avec systemd est ```TAG+="systemd```, sans cet attribut le périphérique ne sera pas interprété comme une unit "device".

C'est en soit la seule chose à faire faire fonctionner cette règle avec systemd.

J'ai ajouté l'attribut ```SYSTEMD_ALIAS=``` car le chemin du périphérique n'est pas prévisible (```/sys/devices/virtual/input/input<X>```), grâce à lui une unit device 
apparaitra à chaque fois sous le nom ```sys-devices-virtual-input-lg_tone_fp9.device```, on est donc maintenant en mesure de d'écrire des services qui vont s'activer lors de la présence de ce device.
```ini
[Unit]
Wants=sys-devices-virtual-input-lg_tone_fp9.device

[Service]
ExecStart=/bin/echo headphone connected
RemainAfterExit=true
```

Il existe également la possibilité d'activer un service spécifique directement à partir d'une règle UDev

#### D-Bus
D-Bus est un nouvel framework d'IPC (inter-process communication) qui permet de communiquer de facon standardisé entre different services.
Il permet - entre autre - de faire :
- du RPC (Remote Procedure Call) en offrant une serialisation standardisé.
- de sécuriser les communications via un mechanism d'autorisation.
- de l'activation a la demande via un système de nommage.
- de la découverte de service
- ...

Les services D-Bus adoptent la convention de notation des packages Java

Systemd est complètement intégré avec D-Bus (l'architecture des deux produits est par ailleurs assez similaire). 

Il est possible de d'interagir avec systemd uniquement via les API D-BUS.

Pour voir les services exposés par B-BUS il suffit de taper la commande :
```shell
~ busctl list --acquired
NAME                                 PID PROCESS         USER             CONNECTION UNIT                             SESSION DESCRIPTION
...
org.freedesktop.systemd1               1 systemd         root             :1.1       init.scope                       -       -
```

On peut également voir l'ensemble des propriétés d'un service 
```shell
~ busctl introspect org.freedesktop.systemd1 /org/freedesktop/systemd1
```

Puis pour lister les propriétés, méthodes d'un service (systemd ici) :
```shell
~ busctl introspect org.freedesktop.systemd1 /org/freedesktop/systemd1
```
Cela donne accès à l'ensemble des objects accessibles via D-BUS (c'est aussi un bon moyen pour connaitre les valeurs des propriétés par défaut)

#### Inotify

Inotify est un mécanisme qui permet d'observer les actions sur un système de fichier. 

Il permet de s'abonner a un repertoire ou fichier et de recevoir les types d'évènements suivants :
- IN_ACCESS : le fichier est accédé en lecture ;
- IN_MODIFY : le fichier est modifié ;
- IN_ATTRIB : les attributs du fichier sont modifiés ;
- IN_OPEN : le fichier est ouvert ;
- IN_DELETE : un fichier a été supprimé dans le répertoire surveillé ;
- IN_CREATE : un fichier a été créé dans le répertoire surveillé.
- ...

Cette fonction est utilisé en interne pour prendre en compte les units de type path.


- l'utilisation de buffers (unix ou net) créées en avance par le daemon et qui va les lire. Ainsi une dépendance qui utilisait ces sockets est en mesure de publier dans ses sockets dans attendre et sera mise en attente s'il attend une réponse de ce service.
- Lancer l'initialisation des services le plus tard possible en se basant sur une approche événementielle avec le recours notamment a DBus, mais aussi automount, l'activation par socket...


### Paralléliser au maximum

Systemd se base sur un model de dependence assez complexe et cherche à minimiser au maximum les points de synchronisations.

Cela permet de construire un arbre de dependence et de pouvoir démarrer un maximum de service en parallèle en ayant à attendre uniquement les services qui sont nécessaires.

Ce qui était difficilement réalisable en script shell, et rendu possible grâce notamment à l'utilisation de C et a la mise en place des _units_ qui permettent de mieux décrire l'ordonnancement des _units_ et leurs dépendances.

À noter que lorsque systemd active un certain nombre de _unit_ - et si aucun ordre n'est spécifié - tout se fait en parallèle. Cela permet d'accélérer le démarrage sans attendre le démarrage d'un service qui n'est pas strictement nécessaires.

Pour arriver a cela un graphique de dépendances est construit et un ordonnancement souvent implicite est mise en place suivant different critères pour garantir la cohérence de l'ordre de démarrage (c'est d'ailleurs souvent la partie la plus complexe à mettre en œuvre a mon avis).

### 3 L'utilisation massive des cgroups

En pratique, il s'avère compliqué de terminer correctement un service ayant un nombre de processus forkés important. 

Un processus est rattaché à une session. Par convention le ```SID``` est égale au ```PID```  du premier membre de la session le "session leader". La session peut être un terminal, une connexion ssh... ou autre.

Dans une session on trouve un certain nombre de groupes de processus. Le ```PGID``` est égale au ```PID``` du premier membre du groupe le "process group leader" (vous voyez la logique). 
Au sein d'une session un seul groupe peut être au premier plan.

Un processus PID a (presque) toujours un parent et hérite de certain de ses attributs (UID, GID, SID, PGID...). 
Il est lui-même composé de plusieurs threads.

Si en theories cela semble simple en pratique, c'est assez compliqué pour différentes raisons (liberté implementation, différences entre les systèmes UNIX, manque de syscall...) qui ont faites qu'il est difficile de suivre l'ensemble des processus issus apres un fork. 

Par exemple l'une des techniques utilisés pour lancer un processus en daemon est la technique dites du "double-fork" qui consiste à effectuer les opérations suivantes : 
- Un premier ```fork()``` pour créer un processus enfant de facon classique.
- Puis va appeler ```setsid()``` pour se détacher de la session courant.
- Et enfin effectuer un nouveau ```fork()``` pour que ce changement soit pris en compte. Le processus nouvellement créé est ainsi rendu orphelin et ne peut plus interagir avec la session d'origine (et vis versa).

Pour les init manager, le problème est qu'ils n'ont connaissance que du PID issu du premier fork et perdent donc la trace des autres processus forkés.

Présent depuis longtemps dans le kernel linux et à la base de tous les systèmes d'isolation (containers par exemple) les CGroups sont prévus pour régler ce type de problème. 
En effet, à la difference d'autres propriétés un processus ne peut pas s'échapper d'un cgroup (à moins de créé un nouveau sous cgroup qui reste néanmoins le descendant du parent).

Systemd a donc décidé de lancer tous les services dans son propre CGroup, ce la permet de résoudre une bonne fois pour toute le problème d'échappement des processus.



SystemD (pour System Daemon) offre de nombreuses améliorations par rapport a l'historique SysV init :
- la parallélisation du lancement
- un meilleur ordonnancement 
- le 'lazy loading' des services
- standardisation des elements de configurations
- meilleure isolation des composants
- facilite le packaging
- ....



## Socket

Unit file suffixée par ".socket". Permet l'activation par socket IPC ou Network.



### Active et passive target

Une target est dite pull lorsqu'elle est référencée par un service via la directive ```Wants=``` ou ```Required=```

Elle peut être dite passive lorsque le service (provider) est exécutée avant via la directive ```Before=``` ou active si le(s) services (consumers) sont exécutés après via la directive ```After=```.


 Il existe une entre les types de targets.


 Un target active est appelée (pull) par un service fournisseur (provider). Ces target ne sont optionnelles (valide uniquement en cas d'initialisation du provider) a moins qu'elles soient inclus dans le cycle de boot.

Une target est passive va appeler des services (consumers) lors de son déclenchement.

 Par exemple :

network-pre.target : est une target passive
network.target: est une target active


## FileSystem

Systemd permet de prendre en charge le montage des différents filesystems.
Historiquement la description des points de montage se situait dans le fichier ```/etc/fstab```.


### Discovrable Pratitions

On peut noter le cote incongru de la situation ou la description des filesystems se trouve-t-elle meme sur un fichier. Ce paradoxe est résolu grâce à l'utilisation du paramètre kernel ```root=``` qui specifie la partition principale ou se situe le fichier de description.

Avec l'utilisation des GTP (GUID Partition Table) contenu sur le disque ou l'image disque systemd est capable de découvrir et de monter automatiquement different type de partitions automatiquement en l'absence de tout paramètre kernel et fstab.

Voir [documentation officielle](https://systemd.io/DISCOVERABLE_PARTITIONS/).



### Mount units

Au démarrage systemd va convertir tous les points de montage des partitions en unit file ```.mount``` 

On peut voir le résultat avec la commande :
```bash
$ systemctl list-units -t mount
```
Ce qui donne :
```shell
  UNIT                          LOAD   ACTIVE SUB     DESCRIPTION
  -.mount                       loaded active mounted Root Mount
  boot-efi.mount                loaded active mounted /boot/efi
  recovery.mount                loaded active mounted /recovery
  ...
```

Les mount unit répondent a une règle spéciale de nommage. 
La racine est toujours ```-.mout```, et leur nom doit correspondre au point de montage en respectant un nom bien précis.

Pour connaitre le nom a donner a ce type de fichier on peut utiliser l'utilitaire suivant
```shell
$ systemd-escape --suffix=mount --path /boot/efi
```

### Montage a la demande

Les units de type ```.mount``` peuvent être activée à la demande, lors du premier accès au repertoire.

Pour cela elles doivent être accompagnées de d'une unit portant le meme nom suffixé paar ```.automount```

Il est possible d'utiliser l'utilitaire systemd-mounts pour générer automatiquement des units ```.mount``` et ```.automount``` a la volée.

