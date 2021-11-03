---
title: "Débuter avec RxJava"
date: 2012-12-04T10:34:56+02:00
slug: ""
description: ""
keywords: [ "java" ]
draft: false
tags: [ "java" ]
math: false
toc: true
---

Pourquoi utiliser Rx ?
===================

Le Framework Rx prend une importance croissante dans le développement d'applications mobiles. Il apporte une très grande flexibilité dans la gestion des appels asynchrones et permet de répondre facilement aux problèmes de synchronisation des événements (le fameux [Callback Hell](https://www.quora.com/What-is-callback-hell)).

Néanmoins l'apprentissage peut être assez déroutant au départ si l'on ne comprend pas la philosophie sur laquelle est basée ce Framework. Une fois cette étape achevé, il devient très simple de répondre à des problématiques d’enchaînement d’événements complexes.

Pour démarrer d'un bon pied nous allons d'abord définir les principes fondamentaux.

Les concepts fondamentaux.
--------------

Rx est basé sur le pattern [Observer](https://en.wikipedia.org/wiki/Observer_pattern) en utilisant la terminologie **Subscriber** / **Observer** et  **Observable**  (un **Subject** en Rx recouvre une notion bien particulière).

### Les Observables

Le [reactive programming](https://en.wikipedia.org/wiki/Reactive_programming) voit les événements comme des flux qui se propagent dans l'application.

Ces flux peuvent avoir des sources multiples : une interaction avec un utilisateur (focus, sélection ...), un élément extérieur (ex: la réponse d'un serveur distant) ou interne (la fin d'un traitement) etc. 

Ils peuvent avoir une durée de vie déterminée (ex : une requête Http) ou non (ex le signal de perte de réseau). De la même manière ces flux peuvent ainsi emmètre un ou plusieurs événements à des intervalles différents.

En Rx un *Observable* sert à matérialiser ces flux, ils peuvent être écoutés, avoir leurs propres cycle de vie, être composés de différents flux…

### Les Subcribers

Est la notion la plus simple à appréhender. Il s'abonne à un *Observable* et il reçoit ses d'émissions, ses signaux d'erreurs ou fin de traitement.

### Les Operators

Les *Operators* servent à traiter et organiser les flux. Ils permettent de filtrer, combiner, transformer, parcourir, composer des flux…

De base une foultitude d'opérateurs sont fournis. Bien qu'on puisse en créer de nouveaux, en pratique les opérateurs par défaut sont généralement largement suffisants pour répondre à la majorité des situations. 

### Les Schedulers

Par défaut Rx n'est pas asynchrone, pour la simple et bonne raisons que le framework réserve cette décision au développeur de l'application. 

Les *Schedulers* servent à contrôler l’exécutions des traitements. Il est alors très simplement possible de décider sur quel Thread va s’exécuter un *Observable* et sur lequel le *Subscriber* va recevoir les réponses. 

RxJava en pratique.
--------------

Commençons d'abord par un exemple très simple pour illustrer les concepts vus précédemment.

### Hello World !

```java
Subscription subscription = Observable.just(example.dummyValue())
    .observeOn(Schedulers.newThread())
    .subscribe(new Observer<Boolean>() {
        @Override
        public void onCompleted() {
            System.out.println("dummyValue has Completed");
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("Error occured : " + getMessage());
        }

        @Override
        public void onNext(Boolean aBoolean) {
            System.out.println("dummyValue result : " + aBoolean);
        }
    });
}

public boolean dummyValue() {
    try {
        Thread.sleep(600);
        return true;
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
}
```

1. `Observable.just` crée un nouvel *Observable* avec une seule valeur de retour.
2. `.observeOn` définit que l'*Observable* s’exécute sur un nouveau Thread.
3. `.subscribe` s'abonne à l'observable.
4. Retourne un objet Subscription.

Il existe plusieurs méthodes génériques pour créer une *Observable* soit à partir de valeurs fixes, d'une collection, d'un `Future<?>` ou `Callback<?>` ou opérateurs. 
Il est aussi relativement facile de créer des *Observable* particuliers si l'on respecte son cycle de vie.

Comme vu précédemment par défaut un *Observable* s'exécute sur le même Thread que celui qui l'a instancié. En spécifiant `Schedulers.newThread()` on force l’exécution du traitement sur un nouveau Thread. 
Rx est fourni avec plusieurs schedulers prédéfinis tel que `io`, `computation`, … On a aussi la possibilité d'en créer afin d'optimiser la gestion des instances de Thread.

L'interface Observer est, elle aussi, relativement simple et lisible. Elle est composée de trois méthodes :
- `onNext`: appelé à chaque objet émis par l'observable.
- `onError`: lorsqu'une erreur s'est produite lors du traitement.
- `onComplete`: lorsque que l'observable a terminé d’émettre des éléments (Par contrat `onNext` ou `onError` ne seront plus jamais appelés).

L’objet Subscription - comme son nom l'indique - est la marque de la souscription d'un Observer vers un *Observable*.

Dans le cas présent, l'*Observable* est exécuté immédiatement avec une durée de vie fixée à un élément. 

On peut schématiser son cycle comme ceci :

> o ---> onNext(true) --> onComplete()

Au travers de cet exemple basique, il faut noter la gestion ultra simple des **threads** et des **erreurs**, qui est à mon avis le gros avantage de ce Framework.

### Débuter avec les Operators

Prenons pour exemple un Observable assez simple pour comprendre leurs fonctionnements.
```java
Observable<String> obs = Observable.just("a", "b", "c", "d", "e");
```

On aura donc 5 appels à `onNext(String s)` avant la méthode `onComplete()`

Le Subscriber affichera le résultat de `onNext` dans la console.

```java
@Override
public void onNext(String result) {
    System.out.println("Result : " + result);
}
```

Par défaut les résultats seront donc : 

>Result : a
>Result : b
>Result : c
>Result : d
>Result : e

#### Filter:
```java
.filter(new Func1<String, Boolean>() {
    @Override
    public Boolean call(String s) {
        return "a".equals(s) || "d".equals(s);
    }
})
```

Retourne :

>Result : a
>Result : d

#### Take:
`.take(2)` ou `.limit(2)`

Retourne les x premiers éléments :

>Result : a
>Result : b

#### TakeLast:
`.takeLast(2)`

Retourne les x derniers éléments :

>Result : d
>Result : e

#### Map:
```java
.map(new Func1<String, String>() {
    @Override
    public String call(String s) {
        return "Hello " + s + " !!";
    }
})
```
Applique pour chaque élément une transformation (il est également possible de modifier le type de retour) :

>Result : Hello a !!
>Result : Hello b !!
>Result : Hello c !!
>Result : Hello d !!
>Result : Hello e !!


Ces opérateurs sont bien évidement combinable entre eux.

### En conclusion

Voici une première approche des basiques du Framework. Nous verrons par la suite les avantages de son utilisation, mais aussi les problèmes spécifiques liés à son utilisation dans de prochains articles.
