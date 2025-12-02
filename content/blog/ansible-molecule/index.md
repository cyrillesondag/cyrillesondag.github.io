---
title: "Ansible Molecule"
date: 2022-05-22T23:24:56+02:00
slug: ""
description: "Introduction à Molecule"
keywords: [ "ansible", "molecule", "tests" ]
draft: false
tags: [ "devops", "ansible" ]
math: false
toc: true
---

Molecule est un framework de test Ansible. Il permet en isolation de tester les roles (mais aussi les playbooks) Ansible.

C'est une réelle aide pour s'assurer de la non-régression mais également pour accélérer le processus de développement.

## Init
Donc d'abord pour commencer, nous allons créer un role vide et explorer la structure proposée par Molecule.

En explorant la commande ```init role``` on obtient le résultat suivant :

```shell
molecule init role --help
Usage: molecule init role [OPTIONS] ROLE_NAME

  Initialize a new role for use with Molecule, namespace is required outside
  collections, like acme.myrole.

Options:
  --dependency-name [galaxy]      Name of dependency to initialize. (galaxy)
  -d, --driver-name [delegated|openstack|podman]
                                  Name of driver to initialize. (delegated)
  --lint-name [yamllint]          Name of lint to initialize. (yamllint)
  --provisioner-name [ansible]    Name of provisioner to initialize. (ansible)
  --verifier-name [ansible|testinfra]
                                  Name of verifier to initialize. (ansible)
```

Ok donc créons notre nouveau rôle :
```shell
bash# molecule init role my_company.my_new_role -d podman
```

Cela aura pour effet de créer un nouveau role dans ```./my_new_role```
```shell
cd my_new_role/ && tree
.
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── molecule
│   └── default
│       ├── converge.yml
│       ├── molecule.yml
│       └── verify.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

Au-delà de la structure classique d'un rôle, nous allons surtout nous intéresser au répertoire ```molecule``` qui contient tous les éléments nécessaires aux tests.

Au premier niveau un répertoire ```default```, il s'agit du scenario par défaut, celui qui sera exécuté si aucune option n'est spécifiée (```-s <nom_du_scenario>``` ou le nom du scenario est celui du répertoire).

Ensuite à l'intérieur le plus important ```molecule.yml```.

## Molecule.yml

C'est le fichier qui est requis dans chaque scenario.
Il va servir à décrire et configurer : le driver, le(s) plateforme(s), le provisioner, le linter et le verifier

Dans notre exemple il ressemble à cela :
```yaml
---
dependency:
  name: galaxy
driver:
  name: podman
platforms:
  - name: instance
    image: quay.io/centos/centos:stream8
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
```

### ```dependency``` ([doc](https://molecule.readthedocs.io/en/latest/configuration.html#ansible-galaxy)) 

Molecule utilise Ansible galaxy comme gestionnaire de dépendances.
Cet objet va permettre de configurer ```ansible-galaxy``` pour installer les rôles et collections requises.

### ```driver``` ([doc](https://molecule.readthedocs.io/en/latest/configuration.html#driver))

Ici on spécifie le nom du pilote à utiliser pour accéder aux instances des plateformes sur lesquelles les tests seront exécutés.

Il existe différents types de drivers : podman, openstack, vagrant, docker, azure... qui correspondent chacun à une technologie pour créer des instances de tests (vm ou container).

Le driver ```delegated``` est quant à lui un peu particulier, car à la différence des autres il n'est pas chargé de créer / détruire les instances de test. 

On peut l'utiliser par exemple pour se connecter à une machine préexistante en renseignant seulement une clé ssh.

Les drivers ne sont pas présents par défaut, ils doivent être installés dans les librairies Python de la façon suivante (ici Docker) :
```shell
pip install 'molecule[docker]'
```

### ```platforms``` ([doc](https://molecule.readthedocs.io/en/latest/configuration.html#platforms))

C'est dans ce dictionnaire que nous allons retrouver la configuration de nos instances.
Ces configurations sont bien évidemment dépendantes du ```driver``` utilisé (bien qu'il puisse y avoir des similitudes).

Leurs noms doivent être uniques (ils serviront de hostname pour ansible), sans limitation de nombre.
On peut définir pour chaque instance un nom de groupe (et des children) pour générer l´inventaire de notre test.

Cette partie de configuration n'est pas forcément toujours très très bien documentée. 
Pour voir l'ensemble des options de configuration, il est souvent plus simple de se référer directement au code du driver (ou dans la section ['Common uses cases'](https://molecule.readthedocs.io/en/latest/examples.html)).
Par exemple pour Docker on peut trouver le détail [ici](https://github.com/ansible-community/molecule-docker/blob/main/src/molecule_docker/driver.py)


### ```provisionner``` ([doc](https://molecule.readthedocs.io/en/latest/configuration.html#provisioner))

Cet objet est là où est défini la configuration d'Ansible.
Il va être chargé à l'aide de playbook classique ansible de réaliser les actions suivantes :
 - ```create``` : création de l'ensemble des plateformes à l'aide du driver. Ce playbook n'est appelé qu'une fois durant la durée de vie des instances de test
 - ```prepare``` : utilisé par le tester pour appliquer les prérequis. Ce playbook n'est appelé qu'une fois durant la durée de vie des instances de test
 - ```converge``` : applique le role en cours de test sur les instances.
 - ```idempotence``` : re-applique le role et vérifie qu'aucun ```changed``` n'ait été déclenché
 - ```side_effect``` : utilisé pour des tests de haute disponibilité (pas activé par défaut)
 - ```verify``` : réalise les assertions sur les instances de tests (activé uniquement sur verifier = ansible)
 - ```cleanup``` (*) : utilisé pour nettoyer les ressources avant le destroy
 - ```destroy``` (*) : utilisé pour supprimer les plateformes de test.

Les playbooks des actions sont définis par défaut à la racine du scenario sous la forme ```./$action.yml```.

Il est également possible de définir des playbooks spécifiques par driver de la façon suivante ```./$driver/$action.yml```.

Enfin certains playbooks sont automatiquement définis par le driver lui-même.

Par exemple le driver docker va utiliser les arguments du dictionnaire des plateformes pour générer des Dockerfile via une template jinja2 et lancer la création des containers via un playbook Ansible.

Il est bien évidemment possible de définir des playbooks comme bon nous semble pour réaliser les actions que l'on souhaite.

Enfin, c'est aussi dans cet objet provisioner que l'on va pouvoir configurer ansible : niveau de log, variables, liens... etc

### ```verifier``` ([doc](https://molecule.readthedocs.io/en/latest/configuration.html#verifier))

Le verifier est chargé de valider le bon déroulement du rôle, en réalisant des assertions sur les plateformes de test.

Par défaut Ansible est utilisé, les tests s'écrivent comme n'importe quel playbook (via par exemple le module assert)

Historiquement [testinfra](https://testinfra.readthedocs.io/en/latest/) était privilégié pour cette tâche, dorénavant Ansible l'a remplacé.
À titre personnel, je trouve que la syntaxe Ansible n'est pas vraiment efficace pour réaliser ce genre de tâche tant c'est verbeux à écrire.
J'imagine que l'homogénéité a primé sur l'efficacité, car testinfra lui doit être écrit en python.

## Les scenarios

(je me demande s'il n'y a pas une confusion entre les termes : scénario répertoire où se trouve le fichier molecule.yml et les scénarios la liste des séquences des actions)

Pour la bonne compréhension de l'outil je remets ici la liste par défaut des actions réalisées par molecule :
```yaml
scenario:
  create_sequence:
    - dependency
    - create
    - prepare
  check_sequence:
    - dependency
    - cleanup
    - destroy
    - create
    - prepare
    - converge
    - check
    - destroy
  converge_sequence:
    - dependency
    - create
    - prepare
    - converge
  destroy_sequence:
    - dependency
    - cleanup
    - destroy
  test_sequence:
    - dependency
    - lint
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - side_effect
    - verify
    - cleanup
    - destroy
```

Ces séquences peuvent être appelées par les commandes suivantes :
```shell
molecule [create|check|converge|destroy|test]
```

NB: il peut être utile en phase de développement de travailler uniquement avec le ```converge```, ainsi les phases "couteuses" (create et prepare) ne sont appelées qu'une fois et on peut travailler plus rapidement sur notre propre environnement.

## Conclusion

Voilà qui conclut cet article, Molecule est un outil indispensable à Ansible. Il souffre néanmoins de quelques lacunes : documentation pas toujours très claire, pas de doc sur les drivers, la configuration du provisioner n'est pas simple à configurer dans des cas un peu complexes, et comme je l'ai dit plus le verifier ansible ne me semble pas idéal.

Je ne suis pas rentré beaucoup dans les détails mais tout est évidemment hautement paramétrable. 

Pour autant une fois le pas franchi et la configuration mise en place, c'est un réel plaisir à utiliser (merci le ```molecule login```)