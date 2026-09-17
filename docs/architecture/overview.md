# Architecture Overview

## Vue d'ensemble

**APIKit-CEL (Contract Error Layer)** est une couche transverse destinée à assurer une gestion cohérente des erreurs entre le serveur, le client et l'interface utilisateur.

Son objectif est de créer une chaîne de traitement explicite permettant à une erreur de traverser les différentes couches de l'application sans perdre son contexte ni devenir silencieuse.

L'architecture repose sur trois grandes parties :

```text id="a81f2c"
                    APIKit-CEL
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
       Server         Client           UI
          │             │             │
      Python         TypeScript       React
```

Ces parties communiquent au travers d'un **Error Contract** commun.

---

# Architecture globale

L'architecture complète peut être représentée ainsi :

```text id="v7x3q1"
┌─────────────────────────────────────────────────────────────┐
│                         APPLICATION                         │
│                                                             │
│   ┌──────────────┐       ┌──────────────┐       ┌────────┐ │
│   │    Server    │ HTTP  │    Client    │       │   UI   │ │
│   │              │──────▶│              │──────▶│        │ │
│   │ Python       │       │ TypeScript   │       │ React  │ │
│   │              │       │              │       │        │ │
│   └──────┬───────┘       └──────┬───────┘       └───┬────┘ │
│          │                      │                    │      │
│          │                      │                    │      │
│          ▼                      ▼                    ▼      │
│   Error Mapping          Error Policy         UI Behavior  │
│          │                      │                    │      │
│          └──────────┐           │           ┌────────┘      │
│                     ▼           ▼           ▼               │
│                  Error Contract / Visibility                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Le serveur produit l'information.

Le contrat la standardise.

Le transport la transmet.

Le client l'interprète.

La politique détermine le comportement.

L'interface présente le résultat lorsque cela est nécessaire.

---

# Les trois couches

## Server

La partie serveur est responsable de la gestion des exceptions et de leur transformation en erreurs applicatives.

Elle doit notamment :

* détecter les exceptions ;
* classifier les erreurs ;
* appliquer les règles de mapping ;
* construire le contrat d'erreur ;
* déterminer les informations transportables ;
* assurer la traçabilité ;
* retourner une réponse cohérente au client.

Dans l'écosystème APIKit, cette partie est portée par le repository :

```text id="k2j9q4"
apikit-python
```

avec notamment :

```text id="z7h4p2"
apikit-server-core
apikit-server-django
```

`apikit-server-core` contient les mécanismes indépendants du framework.

`apikit-server-django` fournit l'intégration avec Django.

Cette séparation permet de conserver le cœur du système indépendant de la technologie utilisée par l'application.

---

# Client

La partie client reçoit et interprète les erreurs produites par le serveur.

Elle est responsable de :

* reconnaître le contrat d'erreur ;
* normaliser les erreurs reçues ;
* déterminer leur comportement ;
* appliquer les politiques d'erreur ;
* gérer les erreurs réseau ou transport ;
* transmettre l'information aux couches UI lorsque nécessaire.

Elle est portée par :

```text id="r5p8n3"
apikit-typescript
```

avec notamment :

```text id="c6y1w8"
apikit-client-core
apikit-react
```

`apikit-client-core` contient la logique cliente indépendante du framework.

`apikit-react` fournit les mécanismes spécifiques à React.

---

# UI

La couche UI représente le dernier niveau de traitement visible de l'erreur.

Elle peut utiliser différents mécanismes selon le contexte :

* Toast ;
* Alert ;
* Popup ;
* formulaire ;
* écran d'erreur ;
* fallback ;
* redirection.

La couche React fournit notamment une infrastructure centralisée pour ces comportements.

Un exemple est le `GlobalUiProvider`, qui regroupe les différents mécanismes UI nécessaires à la présentation des erreurs et des notifications.

Conceptuellement :

```text id="x5n2s8"
                   React Application
                         │
                         ▼
                 GlobalUiProvider
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        Toast          Popup          Alert
```

Cette centralisation évite de disperser la gestion globale des erreurs dans les composants de l'application.

---

# Error Contract

Le point central de l'architecture est le **Error Contract**.

Il constitue la frontière entre le serveur et le client.

```text id="d4f8x1"
             SERVER
                │
                │
         ┌──────▼──────┐
         │    Error    │
         │   Contract  │
         └──────┬──────┘
                │
                │ HTTP
                ▼
             CLIENT
```

Le serveur n'a pas besoin de connaître l'implémentation du client.

Le client n'a pas besoin de connaître les exceptions internes du serveur.

Les deux couches partagent uniquement un contrat commun.

Cela permet notamment de découpler :

```text id="q9v2m6"
Python / Django
       │
       │
       ▼
 Error Contract
       │
       │
       ▼
TypeScript / React
```

---

# Error Flow

Le contrat s'inscrit dans un flux plus large.

```text id="m6z8r3"
Exception
    │
    ▼
Error Mapping
    │
    ▼
Error Contract
    │
    ▼
HTTP / Transport
    │
    ▼
Client Interpretation
    │
    ▼
Error Policy
    │
    ▼
Visibility Policy
    │
    ▼
UI Behavior
```

Chaque étape possède une responsabilité différente.

Cette séparation permet d'éviter qu'une erreur soit directement couplée à son mode d'affichage.

---

# Gestion et visibilité

Une distinction importante de l'architecture est la séparation entre **Error Handling** et **Error Visibility**.

```text id="e3k7p1"
                    Error
                      │
              ┌───────┴───────┐
              │               │
              ▼               ▼
        Error Handling    Error Visibility
              │               │
              ▼               ▼
       Que faire ?       Que montrer ?
```

### Error Handling

Détermine le comportement de l'application :

* retry ;
* redirect ;
* fallback ;
* changement d'état ;
* notification ;
* interruption d'une opération.

### Error Visibility

Détermine les informations exposées :

* message ;
* détails ;
* informations techniques ;
* stack trace ;
* référence de diagnostic.

Cette séparation est particulièrement importante entre les environnements.

```text id="n8c4w2"
DEV
→ diagnostic détaillé

QUALIF
→ diagnostic contrôlé

PROD
→ information utilisateur sécurisée
```

---

# Indépendance vis-à-vis des frameworks

APIKit-CEL cherche à limiter le couplage aux frameworks.

L'architecture distingue donc :

```text id="u2j5k9"
             Core
              │
      ┌───────┴───────┐
      ▼               ▼
 Framework          Framework
 Adapter             Adapter
```

Par exemple :

```text id="h7p3d6"
apikit-server-core
        │
        ▼
apikit-server-django
```

et :

```text id="s4m8q1"
apikit-client-core
        │
        ▼
apikit-react
```

Le cœur porte les concepts et les mécanismes génériques.

Les adapters portent les détails spécifiques aux frameworks.

---

# Organisation des repositories

L'écosystème est organisé autour d'un repository par langage, complété par un repository dédié à la documentation.

```text id="b5x9r2"
APIKit
│
├── apikit-python
│   │
│   ├── apikit-server-core
│   └── apikit-server-django
│
├── apikit-typescript
│   │
│   ├── apikit-client-core
│   └── apikit-react
│
└── apikit-doc
```

Cette organisation permet de séparer :

* l'implémentation Python ;
* l'implémentation TypeScript ;
* les intégrations framework ;
* la documentation et l'architecture.

---

# Relation entre les repositories

Les repositories d'implémentation ne sont pas indépendants conceptuellement.

Ils implémentent différentes parties du même modèle architectural.

```text id="c8m1v4"
                  APIKit-CEL
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
    apikit-python          apikit-typescript
          │                       │
          ▼                       ▼
       Server                  Client
          │                       │
          └───────────┬───────────┘
                      │
                      ▼
                Error Contract
                      │
                      ▼
                     UI
```

`apikit-doc` décrit cette architecture, ses concepts et ses règles d'utilisation.

---

# Principes architecturaux

L'architecture d'APIKit-CEL repose sur plusieurs principes.

## 1. Contrat avant implémentation

Le serveur et le client doivent communiquer au travers d'un contrat stable.

L'implémentation interne peut évoluer sans modifier inutilement la manière dont les erreurs sont consommées.

---

## 2. Séparation des responsabilités

Chaque couche doit avoir une responsabilité clairement définie.

```text id="r4h8v6"
Server
→ Produire

Contract
→ Standardiser

Transport
→ Transmettre

Client
→ Interpréter

Policy
→ Décider

Visibility
→ Contrôler

UI
→ Présenter
```

---

## 3. Pas de couplage serveur → UI

Le serveur ne doit pas savoir si l'erreur sera affichée dans :

```text id="e2q7n5"
Toast
Alert
Popup
Page
```

Il fournit un contrat.

Le client décide du comportement.

---

## 4. Pas de dépendance aux messages

Le client ne doit pas baser sa logique applicative sur le texte du message.

Le comportement doit être déterminé à partir d'informations structurées et stables, notamment le code d'erreur.

```text id="p6j3k9"
❌ "User not found"

        ↓

✔ RESOURCE_NOT_FOUND
```

Le message peut évoluer sans casser la logique cliente.

---

## 5. Visibilité contrôlée

Les informations techniques ne doivent pas être exposées automatiquement.

Le niveau de détail dépend du contexte d'exécution et de la politique définie.

---

## 6. Pas d'erreur silencieuse

Toute erreur importante doit aboutir à un comportement explicitement défini.

```text id="w1k8s3"
Error
 │
 ▼
Contract
 │
 ▼
Policy
 │
 ▼
Explicit Behavior
```

L'application doit éviter qu'une erreur soit simplement ignorée entre deux couches.

---

# Architecture et évolutivité

L'utilisation d'un cœur indépendant et d'adapters permet d'ajouter progressivement de nouvelles intégrations.

Par exemple :

```text id="f3m7q2"
apikit-server-core
       │
       ├── Django
       ├── FastAPI
       └── Flask
```

De même côté client :

```text id="j8p4x6"
apikit-client-core
       │
       ├── React
       ├── Vue
       └── Angular
```

Les intégrations peuvent évoluer indépendamment du cœur tant qu'elles respectent les contrats définis.

---

# Résumé

L'architecture APIKit-CEL peut être résumée par la chaîne suivante :

```text id="n5c2v8"
                    APIKit-CEL
                        │
                        ▼
                   Error Contract
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
        Server                       Client
          │                           │
     Error Mapping              Interpretation
          │                           │
          │                       Error Policy
          │                           │
          │                    Visibility Policy
          │                           │
          └─────────────┬─────────────┘
                        ▼
                        UI
```

L'objectif n'est pas simplement de standardiser les réponses d'erreur.

APIKit-CEL cherche à fournir une **architecture complète de gestion de l'expérience d'erreur**, depuis l'exception serveur jusqu'au comportement final de l'application.

> **Le serveur produit l'erreur, le contrat la standardise, le transport la transmet, le client l'interprète, la politique définit son comportement et l'interface présente le résultat selon le contexte.**
