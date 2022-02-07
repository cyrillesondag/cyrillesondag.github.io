---
title: "Systemd - partie 2"
date: 2022-02-06T23:24:56+02:00
slug: ""
description: "System de configuration"
keywords: [ "linux", "systemd", "init system" ]
draft: false
tags: [ "linux", "systemd" ]
math: false
toc: true
---

{{< figure src="Gustave_Caillebotte_-_Perissoires_sur_yerres.jpg" link="Gustave_Caillebotte_-_Perissoires_sur_yerres.jpg" caption="Gustave Caillebotte - Périssoires sur l'Yerres (1877), huile sur toile, 103 × 155 cm, Milwaukee (USA), Milwaukee Art Museum." >}}

\
Nous avons vu dans le précédent article quelques différences entre SysV init et systemd, ainsi que le mécanisme d'initialisation de l'espace utilisateur par le kernel.
Dans cet article, je vais aborder plus en détail sur le fonctionnement de systemd.

## Les Managers

L'objet _manager_ est central, il contient les valeurs par default des différentes propriétés du système et est responsable de la gestion des _units_ qui lui sont attachées. 
C'est avec lui que les "utilisateurs" vont communiquer.

Il est chargé de propager les actions appelées job (par exemple : start, stop, reload....) vers les _units_ et d'interagir avec les éléments tiers du système (watchdog, inotify, D-bus, cgroups....).

Il en existe sous 2 formes ayant chacune des portés différentes : 

La première (celui par défaut) appelé "system" est unique par machine et est normalement lancé par le process d'init avec les droits root. 

On peut vérifier que systemd est bien l'init _manager_ du système de la facon suivante :
```shell
# ls -ls /usr/sbin/init
/usr/sbin/init -> /lib/systemd/systemd
```
Ou plus simplement utiliser cette commande : 
```shell
systemctl
```

La seconde est "user", il est unique par utilisateur et hérite des mêmes droits que ce dernier.
Ce type d'instance s'avère particulièrement utile pour limiter les droits des services qui lui sont associés (rootless).

Pour connaitre l'état d'une instance utilisateur on peut utiliser la commande suivante :
```shell
systemctl --user
```
Bien spécifier l'argument ```--user``` devant toutes les commandes pour interagir avec l'instance de l'utilisateur.

### La gestion des Units

On l'a vu les _managers_ sont responsables de la gestion des _units_.

Une _unit_ chargée par un _manager_ elle est unique si son nom est unique (pour les templates, les alias, les fragments, le nom est résolu au runtime).

Les _units files_ sont lues au premier "référencement" (par exemple lors de leur activation, ou bien via de liens de dépendances...), transformées en _unit_ et mise en cache dans la mémoire du _manager_ qui leur est associé. 

Elles sont stockées à l'intérieur d'une HashMap sous la forme nom/valeur (d'où leur unicité).

Pour cette raison, il faut forcer le ramasse-miette (GC) ce qui a pour effet de sérialiser sur disque l'état des _units_ actives, vider le cache du manager et recharger la configuration des _units_ via leurs _unit file_ (ce n'est pas néanmoins pas toujours nécessaire, mais ça évite des erreurs).

Pour forcer le ramasse miette on utilise la commande suivante :
```shell
systemctl daemon-reload
```

### Arborescence et précédence

La definition des _unit files_ répondent a une précédence tres precise utilisé afin de faciliter l'administration du système et sa mise à jour.

Par ordre d'importance croissante (non exhaustif) en mode "--system" :
- ```/lib/systemd/system/*``` : installés par le gestionnaire de paquets (il est recommandé d'éviter de les modifier sous peine de pertes lors des mises à jour du paquet) 
- ```/usr/local/lib/systemd/system/*``` : gérés par l'administrateur
- ```/run/systemd/system/*``` : générées au runtime
- ```/etc/systemd/system/*``` : gérés par l'administrateur

Ainsi une _unit file_ définie dans le répertoire ```/etc/systemd/system/*``` sera prioritaire sur celle définie dans ```/lib/systemd/system/*``` 

En mode "--user" la hiérarchie est un peu différente :
- ```/lib/systemd/user/*``` : installés par le gestionnaire de paquets (ne doivent pas être modifiés sous peine de pertes lors des mises à jour) 
- ```/usr/local/lib/systemd/user/*``` : gérés par l'administrateur
- ```/run/systemd/user/*``` : générées au runtime
- ```/etc/systemd/user/*``` : gérés par l'administrateur
- ```~/.config/systemd/user/*``` : géré par l'utilisateur

Ça peut paraitre compliqué aux premiers abords, mais limite le périmètre des différents intervenants sur une machine (utilisateurs, administrateurs, packageurs...) pour qu'ils ne se marchent sur les pieds.

De plus des outils spécifiques sont fournis (on le verra plus loin) pour aider à la compréhension de ce mécanisme.

### Fragment Configuration

Les fragments configuration sont des fichiers qui permettent de modifier les propriétés d'une ou d'un groupe de _units_ sans avoir à la/les redéfinir entièrement.
Ces fichiers répondent aux mêmes ordres de précédence que nous venons d'aborder plus haut.

Pour surcharger, il est possible de définir des fichiers ```.conf``` qui seront lus par ordre alphabétique dans les formats suivants :
- Par type de _unit_ sous la forme ```<unit_type>.d/*.conf```. Ainsi un fragment dans un répertoire ```service.d/*.conf``` s'appliquera à tous les ```*.service```
- Une _unit_ spécifique sous la forme ```<unit_name>.d/*.conf```
- Ou grâce à une pseudo hiérarchie et l'utilisation du signe '-' dans le nom des _units_. Ainsi un fragment de configuration dans le répertoire ```foo-.d/*.conf``` va surcharger les configurations des _units_ ```foo-bar.service```, ```foo-foo.service``` et ```foo-bar-foo.service```...

Voyons maintenant comment appliquer cela et les outils fournis par systemd pour nous assister dans la mise en place.

Si l'on crée le fichier ```/etc/systemd/system/service1.service``` de la facon suivante :
```ini
[Service]
Type=oneshot
ExecStart=/bin/echo hello world
```

Puis le fragment de configuration dans le fichier suivant ```/etc/systemd/system/service1.service.d/00-override.conf``` :
```ini
[Service]
ExecStart=/bin/echo running this first
```

Puis le fichier suivant ```/etc/systemd/system/service.d/00-override.conf``` :
```ini
[Service]
ExecStart=/bin/echo top level exec pre start
```

On obtient en utilisant la commande ```systemctl cat``` le détail de la configuration de la _unit_ ainsi que ses différents fragments :

```shell
$ systemctl cat service1.service
# /etc/systemd/system/service1.service
[Service]
ExecPreStart=/bin/echo hello world

# /etc/systemd/system/service1.service.d/00-override.conf
[Service]
ExecPreStart=/bin/echo running this first

# /etc/systemd/system/service.d/00-override.conf
[Service]
ExecPreStart=/bin/echo top level exec pre start
```

Et produit le résultat suivant :
```
top level exec pre start
running this first
hello world
```

Il est intéressant de noter que les surcharges sont additives. 
Pour passer outre les précédentes définitions, il faut d'abord déclarer la propriété à vide avant de la surcharger avec la valeur attendue.

```ini
[Service]
ExecStart=
ExecStart=/bin/echo top level exec pre start
```

## Les Units

Les _units_ sont les éléments de bases de systemd.
Il existe 11 types différents de _unit_. 
Ils correspondent chacun à un type d'actions particulières.

Le type d'une _unit_ est déterminé par le suffix de son _unit file_ :
- *.service : Qui permet de définir un groupe de processus. C'est l'objet qu'on est le plus souvent amené à utiliser.
- *.socket : Pour faire de l'activation de services par sockets (IPC, network, unix socket...).
- *.device : Pour prendre en compte l'activation par périphériques via udev (dont le hot-plug).
- *.mount : Pour gérer le montage de partitions.
- *.automount : Qui permet le l'activation des _units_ .mount automatique lors du premier accès.
- *.swap : Configuration d'un point de montage swap
- *.target : Sont des point de synchronisation d'un ensemble de _units_ (comme les run-levels de SysVinit), mais également des points de déclenchements.
- *.path : Activation par observation de chemins
- *.timer : Déclencheur périodique (à la manière de cron)
- *.scope : Configuration d'un ensemble de processus externes
- *.slice : Qui permet de regrouper un ensemble de processus au sien d'un cgroup commun.

Une _unit_ est décrite par une _unit file_ qui est lu au premier référencement pour être transformé en _unit_.

L'objet _units_ implémentes les opérations de bases génériques, une spécialisation est faite via la structure ```UnitVtable``` qui va referencer les méthodes spécifiques à chaque type.

Voici un example non exhaustif des méthodes de ```UnitVtable``` :
```c
typedef struct UnitVTable {
        /* How much memory does an object of this unit type need */
        size_t object_size;

        /* The name of the configuration file section with the private settings of this unit */
        const char *private_section;

        /* Config file sections this unit type understands, separated
         * by NUL chars */
        const char *sections;

        /* This should reset all type-specific variables. This should
         * not allocate memory, and is called with zero-initialized
         * data. It should hence only initialize variables that need
         * to be set != 0. */
        void (*init)(Unit *u);

        /* This should free all type-specific variables. It should be
         * idempotent. */
        void (*done)(Unit *u);

        /* Actually load data from disk. This may fail, and should set
         * load_state to UNIT_LOADED, UNIT_MERGED or leave it at
         * UNIT_STUB if no configuration could be found. */
        int (*load)(Unit *u);

        void (*dump)(Unit *u, FILE *f, const char *prefix);

        int (*start)(Unit *u);
        int (*stop)(Unit *u);
        int (*reload)(Unit *u);

        int (*kill)(Unit *u, KillWho w, int signo, sd_bus_error *error);

        /* Boils down the more complex internal state of this unit to
         * a simpler one that the engine can understand */
        UnitActiveState (*active_state)(Unit *u);

        /* Return false when there is a reason to prevent this unit from being gc'ed
         * even though nothing references it and it isn't active in any way. */
        bool (*may_gc)(Unit *u);

        /* Return true when the unit is not controlled by the manager (e.g. extrinsic mounts). */
        bool (*is_extrinsic)(Unit *u);

        /* Returns the exit status to propagate in case of FailureAction=exit/SuccessAction=exit; usually returns the
         * exit code of the "main" process of the service or similar. */
        int (*exit_status)(Unit *u);
        .....
} UnitVTable;
```

On retrouve logiquement dans l'objet Unit l'ensemble des sous-types, qui permettra de propager les actions génériques vers leurs implementations. 
```c
const UnitVTable * const unit_vtable[_UNIT_TYPE_MAX] = {
        [UNIT_SERVICE] = &service_vtable,
        [UNIT_SOCKET] = &socket_vtable,
        [UNIT_TARGET] = &target_vtable,
        [UNIT_DEVICE] = &device_vtable,
        [UNIT_MOUNT] = &mount_vtable,
        [UNIT_AUTOMOUNT] = &automount_vtable,
        [UNIT_SWAP] = &swap_vtable,
        [UNIT_TIMER] = &timer_vtable,
        [UNIT_PATH] = &path_vtable,
        [UNIT_SLICE] = &slice_vtable,
        [UNIT_SCOPE] = &scope_vtable,
};
```

Un _unit_ a un cycle de vie, et donc un état. Les états de base sont les suivants :
- Active
- Reloading
- Inactive
- Failed
- Activating
- Deactivating
- Maintenance


Cette liste d'état peut être, elle aussi, enrichie pour répondre au cycle de vie spécifique d'un type d'_unit_.

Par exemple pour une _unit_ de type service on trouve cette liste d'états qui font tous la correspondance avec les états de base.
```c
static const UnitActiveState state_translation_table[_SERVICE_STATE_MAX] = {
        [SERVICE_DEAD] = UNIT_INACTIVE,
        [SERVICE_CONDITION] = UNIT_ACTIVATING,
        [SERVICE_START_PRE] = UNIT_ACTIVATING,
        [SERVICE_START] = UNIT_ACTIVATING,
        [SERVICE_START_POST] = UNIT_ACTIVATING,
        [SERVICE_RUNNING] = UNIT_ACTIVE,
        [SERVICE_EXITED] = UNIT_ACTIVE,
        [SERVICE_RELOAD] = UNIT_RELOADING,
        [SERVICE_STOP] = UNIT_DEACTIVATING,
        [SERVICE_STOP_WATCHDOG] = UNIT_DEACTIVATING,
        [SERVICE_STOP_SIGTERM] = UNIT_DEACTIVATING,
        [SERVICE_STOP_SIGKILL] = UNIT_DEACTIVATING,
        [SERVICE_STOP_POST] = UNIT_DEACTIVATING,
        [SERVICE_FINAL_WATCHDOG] = UNIT_DEACTIVATING,
        [SERVICE_FINAL_SIGTERM] = UNIT_DEACTIVATING,
        [SERVICE_FINAL_SIGKILL] = UNIT_DEACTIVATING,
        [SERVICE_FAILED] = UNIT_FAILED,
        [SERVICE_AUTO_RESTART] = UNIT_ACTIVATING,
        [SERVICE_CLEANING] = UNIT_MAINTENANCE,
};
```

## Job et Transaction

Tout changement d'état d'une _unit_ se fait au travers d'un Job.
Il existe plusieurs types de job :
- Start / Verify
- Stop
- Reload
- Restart
- Nop

On trouve également d'autres types de Job qui sont la combinaison des types précédents : 
par exemple "try-restart" se transformera à l'exécution en "start" ou "nop" si la _unit_ n'est pas activée ou en cours de rechargement.

Prenons l'exemple de la _unit file_ suivante :
```ini
[Unit]
Wants=multi-users.target
...
```

La propriété ```Wants=``` a pour effet de créer une dépendance entre cette _unit_ et la target _unit_ "multi-user.target".

NB : La plupart des relations peuvent être notée dans un sens ou dans l'autre (```Before=```/ ```After=```, ```Require=```/```RequiredBy=```...) 
On pourrait donc réaliser cette même dépendance a l'aide de la propriété ```WantedBy=``` dans la target et cela aurait le même effet.

Cette dépendance va avoir pour effet de propager l'activation de la target "multi-user.target" aux _unit_ ayant une relation de type ```Wants=``` avec elle.

Lors de cette activation une transaction qui va contenir le job d'activation de la target en elle meme plus autant de job que de dépendances.

Ainsi dans le cas d'une relation avec une _unit_ de type ```Wants=``` la transaction pourra est mise ne succès même si ce job est en échec. 
Ce qui n'aurait pas été le cas avec une relation plus "forte" (par exemple ```Require=```/```RequiredBy=```).

En pratique ces relations sont spécifiées par une enumeration de propriétés beaucoup plus fines les ```UnitDependencyAtom``` ou "atoms" (qui ne sont rien d'autre qu'un bitmask).

Ainsi la propriété ```Wants=``` est composée de la sorte :
```c
[UNIT_WANTS] = UNIT_ATOM_PULL_IN_START_IGNORED |
               UNIT_ATOM_RETROACTIVE_START_FAIL |
               UNIT_ATOM_ADD_STOP_WHEN_UNNEEDED_QUEUE |
               UNIT_ATOM_ADD_DEFAULT_TARGET_DEPENDENCY_QUEUE,
```

Cela lui permet de réagir aux propriétés suivantes qui peuvent être utilisées par different objects (transaction, units...). 
Sans rentrer dans les details on retrouve l'attribut "UNIT_ATOM_PULL_IN_START_IGNORED" le comportement décrit plus haut.

Il implémente un mécanisme de transaction connu sous le nom de Job qui s'assure de la transition d'une _unit_ d'un état A vers un état B (par exemple start / stop / restart...).

## Conclusion

Vous l'avez peut-être remarqué, mais systemd est concu comme un système orienté objet. 
On retrouve d'ailleurs beaucoup de principes de la POO (polymorphisme, sous-typage, redefinition...) dans les différents points que j'ai abordés

Voilà ainsi va se conclure cet article.
Dans le prochain, nous aborderons l'architecture de systemd.

