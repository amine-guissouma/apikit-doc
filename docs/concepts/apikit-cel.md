# APIKit-CEL — Contract Error Layer

**APIKit-CEL (Contract Error Layer)** est une couche transverse de gestion des erreurs conçue pour rendre les défaillances applicatives **prévisibles, observables et cohérentes**, du backend jusqu'à l'expérience utilisateur frontend.

Son objectif est d'établir un contrat commun entre le serveur et le client pour **représenter, transporter, interpréter et présenter les erreurs** de manière uniforme.

L'idée fondamentale est simple :

> **Une erreur ne doit jamais rester silencieuse et chaque erreur doit avoir un comportement explicitement défini.**

APIKit-CEL structure ainsi le traitement des erreurs selon une chaîne cohérente :

**Exception → Contrat d'erreur → Transport → Interprétation côté client → Politique de gestion → Retour UI**

Cette approche permet de séparer la **gestion technique de l'erreur** de sa **présentation à l'utilisateur**, tout en adaptant le niveau d'information selon le contexte d'exécution.

## Problématique

Dans une application distribuée, une erreur peut traverser plusieurs couches :

```text
Backend
   │
   │ Exception / erreur métier / validation
   ▼
Réponse HTTP
   │
   ▼
Client
   │
   │ interprétation
   ▼
Interface utilisateur
```

Sans stratégie commune, certaines erreurs peuvent être mal interprétées, perdre leur contexte ou, pire, devenir silencieuses côté frontend.

L'utilisateur peut alors rencontrer une situation comme :

> « J'ai effectué une action, mais rien ne s'est passé. »

APIKit-CEL cherche à éliminer ce type de comportement en donnant à chaque erreur un **contrat, une stratégie de traitement et un comportement UI déterminé**.

## Gestion de la visibilité des erreurs

APIKit-CEL introduit également une notion de **visibilité des erreurs dépendante de l'environnement**.

Une même erreur ne doit pas nécessairement être présentée de la même manière en développement et en production.

```text
                    Même erreur
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
            DEV        QUALIF       PROD
             │           │           │
             ▼           ▼           ▼
          Détaillée    Contrôlée    Sécurisée
```

### Développement

Le développeur doit disposer du maximum d'informations utiles au diagnostic :

* exception ;
* message technique ;
* contexte ;
* informations de validation ;
* stack trace ;
* détails nécessaires à l'identification du problème.

Une erreur non gérée peut ainsi être directement présentée au développeur afin de rendre le problème immédiatement visible.

### Qualification

L'information peut être volontairement limitée à des éléments utiles au diagnostic et aux tests :

* code d'erreur ;
* message contrôlé ;
* contexte fonctionnel ;
* identifiant de corrélation ;
* informations nécessaires au suivi du problème.

### Production

L'utilisateur final ne doit pas recevoir de détails techniques ou d'informations internes à l'application.

Il reçoit une réponse adaptée à son expérience :

* message compréhensible ;
* notification ;
* alerte ;
* popup ;
* redirection ;
* comportement de récupération approprié.

Les informations techniques restent destinées aux mécanismes de **logging et d'observabilité**.

## Architecture

APIKit-CEL est conçu comme une architecture transverse séparant les responsabilités entre le serveur, le client et la présentation :

```text
                         APIKit-CEL
                             │
                  Contract Error Layer
                             │
             ┌───────────────┴───────────────┐
             │                               │
        Côté serveur                    Côté client
             │                               │
      Exception handling              Interprétation
      Validation                      Error policies
      Mapping                         Visibilité
      Error contracts                 Gestion client
             │                               │
             └───────────────┬───────────────┘
                             │
                      Expérience utilisateur
                             │
                  ┌──────────┼──────────┐
                  ▼          ▼          ▼
                Toast      Alert      Popup
```

L'architecture repose sur un **cœur indépendant des frameworks**, complété par des adaptateurs spécifiques aux technologies utilisées.

Cela permet notamment de séparer :

* la définition des contrats ;
* la logique de gestion des erreurs ;
* l'intégration avec le framework serveur ;
* l'interprétation côté client ;
* la présentation dans l'interface utilisateur.

## Principes fondamentaux

APIKit-CEL repose sur plusieurs principes :

* **Un contrat d'erreur commun** entre les différentes couches de l'application.
* **Une gestion centralisée** plutôt qu'une multiplication de traitements spécifiques dans chaque fonctionnalité.
* **Une visibilité adaptée à l'environnement**.
* **Aucune erreur importante laissée silencieuse**.
* **Une séparation entre diagnostic technique et expérience utilisateur**.
* **Des comportements UI configurables** selon le type et le contexte de l'erreur.
* **Une architecture extensible et indépendante des frameworks**.
* **Une stratégie cohérente de l'erreur jusqu'à son affichage final**.

## Flux global

L'objectif d'APIKit-CEL peut être résumé par le flux suivant :

```text
Exception
    │
    ▼
Classification / Mapping
    │
    ▼
Error Contract
    │
    ▼
Réponse API
    │
    ▼
Interprétation Client
    │
    ▼
Error Policy
    │
    ├── visibilité
    ├── comportement
    └── présentation
    │
    ▼
Interface utilisateur
```

À chaque étape, l'erreur conserve une représentation et un comportement prévisibles.

## Objectif

APIKit-CEL cherche finalement à transformer la gestion des erreurs en une **capacité applicative cohérente**, plutôt qu'en une succession de traitements dispersés dans le code.

Il répond à six questions essentielles :

1. **Qu'est-ce qui s'est passé ?**
2. **Comment l'erreur doit-elle être représentée ?**
3. **Comment doit-elle être transportée ?**
4. **Comment le client doit-il l'interpréter ?**
5. **Quel comportement doit adopter l'application ?**
6. **Quelles informations doivent être visibles dans l'environnement courant ?**

L'objectif final est de garantir qu'une erreur provenant du backend puisse être **correctement comprise, transportée, traitée et présentée jusqu'au frontend**, avec une expérience adaptée au développeur comme à l'utilisateur final.

**APIKit-CEL transforme ainsi la gestion des erreurs en un mécanisme contractuel, configurable et transverse, conçu pour éviter les erreurs silencieuses et améliorer la fiabilité ainsi que l'expérience de développement et d'utilisation des applications.**
