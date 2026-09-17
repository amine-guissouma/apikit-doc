# UI

## Vue d'ensemble

La couche **UI** d'APIKit-CEL représente le dernier niveau du traitement d'une erreur.

Elle est responsable de transformer une décision produite par la couche cliente en un comportement visible dans l'interface utilisateur.

```text id="u8m3p1"
Server
   │
   ▼
Error Contract
   │
   ▼
Client
   │
   ▼
Error Policy
   │
   ▼
UI Policy / UI Service
   │
   ▼
React UI
```

L'UI ne doit pas avoir à comprendre l'origine technique de l'erreur.

Elle reçoit une décision et se concentre sur sa présentation.

---

# Responsabilité de la couche UI

La couche UI répond principalement à la question :

> **Comment présenter le résultat du traitement de l'erreur à l'utilisateur ?**

Elle peut notamment :

* afficher une notification ;
* ouvrir une popup ;
* afficher une alerte ;
* afficher une erreur directement dans un formulaire ;
* afficher une page d'erreur ;
* afficher un état de fallback ;
* ne rien afficher lorsque le comportement est volontairement silencieux.

L'UI ne doit donc pas devenir responsable de toute la logique de gestion des erreurs.

---

# Séparation Client / UI

Une distinction importante est faite entre la logique cliente et la présentation.

```text id="c5r9x2"
              Client Core
                  │
                  ▼
             Error Policy
                  │
                  ▼
              UI Decision
                  │
                  ▼
             React Layer
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
        Toast   Popup   Alert
```

Le client détermine **quoi faire**.

L'UI détermine **comment le présenter**.

---

# Architecture React

L'intégration React est fournie par `apikit-react`.

Elle permet de centraliser les mécanismes UI globaux autour d'un provider principal :

```text id="m7k2p8"
                  React Application
                         │
                         ▼
                  GlobalUiProvider
                         │
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
           Toast        Popup       Alert
```

Le `GlobalUiProvider` constitue le point d'entrée de l'infrastructure UI globale.

---

# GlobalUiProvider

Le `GlobalUiProvider` regroupe les différents providers nécessaires à l'interface.

Conceptuellement :

```text id="p4n8c1"
GlobalUiProvider
      │
      ├── GlobalToastProvider
      │
      ├── GlobalPopupProvider
      │
      └── GlobalAlertProvider
```

L'application n'a ainsi pas besoin de configurer indépendamment chaque mécanisme global.

Cette centralisation permet de fournir une infrastructure UI commune à l'ensemble de l'application.

---

# Pourquoi un Provider global ?

Les erreurs peuvent être détectées très loin dans l'arbre React.

Par exemple :

```text id="x6m3r8"
App
 │
 ├── Header
 │
 ├── Dashboard
 │    │
 │    └── UserList
 │          │
 │          └── UserService
 │
 └── Footer
```

Un composant ou un service situé profondément dans l'application peut avoir besoin de déclencher un feedback global.

Une infrastructure centralisée permet de ne pas faire remonter manuellement l'état jusqu'à `App`.

```text id="k8q2v5"
Deep Component
      │
      ▼
 UI Service
      │
      ▼
Global Provider
      │
      ▼
     UI
```

---

# Toast

Le Toast est adapté aux informations qui ne nécessitent pas d'interaction immédiate.

Exemples :

* opération échouée ;
* erreur réseau ;
* information temporaire ;
* résultat d'une action utilisateur.

Conceptuellement :

```text id="r5m9c3"
Error Policy
     │
     ▼
ToastService
     │
     ▼
GlobalToastProvider
     │
     ▼
Toast
```

Le Toast permet de fournir un feedback sans interrompre fortement l'utilisateur.

---

# Popup

Le Popup est adapté aux erreurs nécessitant davantage de contexte ou d'attention.

```text id="v2n7k4"
Error Policy
     │
     ▼
PopupService
     │
     ▼
GlobalPopupProvider
     │
     ▼
Popup
```

Le contenu peut être plus riche qu'une simple notification.

Cela peut notamment être utile en développement lorsqu'une erreur doit présenter des informations techniques supplémentaires.

---

# Alert

L'Alert est adaptée aux messages nécessitant une présentation plus explicite mais restant relativement simple.

```text id="j6p3x8"
Error Policy
     │
     ▼
AlertService
     │
     ▼
GlobalAlertProvider
     │
     ▼
Alert
```

Elle peut notamment être utilisée pour :

* avertissements ;
* erreurs importantes ;
* informations critiques ;
* confirmations ou messages nécessitant l'attention de l'utilisateur.

---

# UI Services

Les services UI permettent de découpler le déclenchement d'un comportement de son implémentation React.

Conceptuellement :

```text id="c9m4r7"
Application Logic
       │
       ▼
 UI Service
       │
       ▼
React Provider
       │
       ▼
 UI Component
```

Par exemple :

```text id="w5k8p2"
Error Policy
     │
     ▼
AlertService.error(...)
     │
     ▼
GlobalAlertProvider
     │
     ▼
MUI Alert / Dialog
```

L'appelant n'a donc pas besoin de gérer directement l'état du composant UI.

---

# Context React

Les providers utilisent le mécanisme de Context de React pour rendre les comportements UI disponibles dans l'arbre de composants.

Conceptuellement :

```text id="n3q7m1"
GlobalAlertProvider
       │
       ├── Context
       │
       └── Alert State
              │
              ▼
        React Components
```

Les composants consommateurs peuvent accéder au mécanisme UI sans connaître la manière dont l'état est stocké ou rendu.

---

# État UI

Les providers sont responsables de maintenir l'état nécessaire à leur mécanisme.

Par exemple, plusieurs Toast peuvent être actifs simultanément :

```text id="g8r2c5"
GlobalToastProvider
       │
       ▼
Toast State
       │
       ├── Toast A
       ├── Toast B
       └── Toast C
```

Le provider contrôle ensuite leur cycle d'affichage et leur disparition.

Cette gestion permet de centraliser le comportement plutôt que de le dupliquer dans chaque composant.

---

# UI et Error Policy

La couche UI ne doit pas déterminer seule la politique d'erreur.

Par exemple, elle ne devrait pas transformer arbitrairement :

```text id="m4x8q2"
AUTHENTICATION_REQUIRED
```

en Toast, Popup ou Redirect.

Cette décision appartient à la policy.

```text id="q7c3n9"
AUTHENTICATION_REQUIRED
          │
          ▼
     Error Policy
          │
          ▼
       Redirect
```

L'UI intervient uniquement lorsque la policy détermine qu'un comportement UI est nécessaire.

---

# Exemple de séparation

Un mauvais couplage serait :

```text id="z5r1p8"
React Component
      │
      ├── catch(error)
      ├── inspect error.message
      ├── decide what error means
      └── open popup
```

Cela conduit rapidement à une logique d'erreur dispersée.

L'architecture APIKit-CEL cherche plutôt à obtenir :

```text id="d8m4k6"
API Response
      │
      ▼
Client Core
      │
      ▼
Error Policy
      │
      ▼
UI Action
      │
      ▼
React
```

Le composant React reçoit une décision déjà interprétée.

---

# Exemple : erreur serveur

Supposons :

```text id="p3k7x2"
INTERNAL_ERROR
```

Le flux complet devient :

```text id="r9c4m8"
Server
  │
  ▼
Error Contract
  │
  ▼
Client
  │
  ▼
Error Policy
  │
  ▼
Visibility Policy
  │
  ▼
PopupService
  │
  ▼
GlobalPopupProvider
  │
  ▼
React UI
```

En développement, la popup peut présenter des informations techniques.

En production, elle peut présenter uniquement un message sécurisé.

---

# Exemple : erreur de validation

Une erreur de validation ne nécessite pas nécessairement de feedback global.

```text id="x6p2n7"
VALIDATION_ERROR
      │
      ▼
Client Policy
      │
      ▼
Form State
      │
      ▼
Field Error
```

Dans ce cas, l'UI peut afficher :

```text id="h4m8c1"
Email
┌──────────────────────────────┐
│ invalid@email                │
└──────────────────────────────┘
  Adresse email invalide
```

Le comportement UI est donc adapté à la nature de l'erreur.

---

# Exemple : erreur d'authentification

Une erreur d'authentification peut ne nécessiter aucun Toast.

```text id="v7q3k9"
AUTHENTICATION_REQUIRED
          │
          ▼
      Error Policy
          │
          ▼
   Authentication State
          │
          ▼
       Redirect
          │
          ▼
        Login
```

La couche UI participe alors à la navigation plutôt qu'à l'affichage d'un message.

---

# Visibilité et UI

La couche UI doit respecter la politique de visibilité.

```text id="n5r8c2"
Error Contract
      │
      ▼
Visibility Policy
      │
      ├── DEV
      │    └── technical details
      │
      ├── QUALIF
      │    └── controlled details
      │
      └── PROD
           └── safe message
                  │
                  ▼
                  UI
```

L'UI ne doit donc pas afficher automatiquement toutes les informations disponibles dans le contrat.

---

# UI et environnement

L'environnement peut influencer la quantité d'information présentée.

### DEV

```text id="c8m2p5"
┌─────────────────────────────┐
│ Internal Error              │
│                             │
│ DatabaseError               │
│                             │
│ Stack trace...              │
└─────────────────────────────┘
```

### PROD

```text id="r3k7n1"
┌─────────────────────────────┐
│ Une erreur est survenue.    │
│                             │
│ Référence : ERR-8F31A2      │
└─────────────────────────────┘
```

Le comportement fonctionnel peut rester identique alors que la présentation diffère.

---

# UI et Material UI

L'intégration actuelle de l'UI s'appuie sur les composants de **Material UI (MUI)**.

APIKit-CEL utilise notamment des primitives telles que :

* Dialog ;
* Alert ;
* Collapse ;
* Fade.

Le rôle d'APIKit-CEL n'est cependant pas de définir une identité visuelle particulière.

MUI constitue ici une implémentation UI.

Le principe architectural reste :

```text id="k4p9x3"
APIKit-CEL UI Abstraction
          │
          ▼
    UI Framework
          │
          ▼
        MUI
```

Cela permettrait à terme de remplacer ou compléter l'implémentation UI sans modifier les concepts du core.

---

# UI comme couche d'adaptation

La couche React peut être considérée comme un adapter entre la logique cliente et le framework d'interface.

```text id="m8c5r1"
             Client Core
                 │
                 ▼
          UI Abstraction
                 │
                 ▼
           apikit-react
                 │
                 ▼
               React
                 │
                 ▼
                MUI
```

Cette organisation limite la dépendance du core aux technologies de présentation.

---

# Cycle de vie

Les mécanismes UI globaux possèdent leur propre cycle de vie.

```text id="p2v7k4"
Provider Mount
      │
      ▼
Service Registration
      │
      ▼
UI Service Available
      │
      ▼
Application Usage
      │
      ▼
Provider Unmount
      │
      ▼
Service Cleanup
```

La gestion correcte du cycle de vie est importante lorsque l'application possède :

* plusieurs roots React ;
* des tests ;
* du rendu serveur ;
* des applications embarquées ;
* des montages/démontages dynamiques.

Les services globaux doivent éviter de conserver des références obsolètes vers des providers.

---

# Centralisation vs composants

L'infrastructure globale ne signifie pas que tous les messages doivent passer par un mécanisme global.

Il faut distinguer :

```text id="x3m8q6"
Global UI
    │
    ├── Toast
    ├── Popup
    └── Alert
```

et :

```text id="j7c2p9"
Local UI
    │
    ├── Form Error
    ├── Field Error
    └── Component State
```

Une erreur spécifique à un composant doit généralement rester locale.

Une erreur nécessitant un feedback global peut utiliser l'infrastructure APIKit-CEL.

---

# Choix du mécanisme UI

Le type d'erreur et son contexte peuvent guider le choix du mécanisme.

| Situation                    | Comportement possible |
| ---------------------------- | --------------------- |
| Validation de formulaire     | Erreurs de champs     |
| Information temporaire       | Toast                 |
| Erreur nécessitant attention | Alert                 |
| Diagnostic détaillé          | Popup                 |
| Authentification expirée     | Redirect              |
| Ressource indisponible       | Fallback / Alert      |
| Erreur globale critique      | Page / Popup          |

Le tableau représente des comportements possibles, et non une obligation.

La décision finale appartient à la policy de l'application.

---

# Flux UI complet

Le flux complet côté interface est :

```text id="q8m4r2"
                 Error Contract
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
                   UI Action
                       │
            ┌──────────┼──────────┐
            │          │          │
            ▼          ▼          ▼
          Toast      Popup      Alert
            │          │          │
            └──────────┼──────────┘
                       ▼
                  React Render
                       │
                       ▼
                     User
```

---

# Principes architecturaux

## 1. L'UI ne connaît pas les exceptions serveur

Elle travaille avec les informations interprétées par le client.

---

## 2. L'UI ne décide pas de la sémantique

Le composant ne doit pas avoir à comprendre ce que signifie chaque code d'erreur.

Cette responsabilité appartient à la policy.

---

## 3. Le core reste indépendant de React

Les mécanismes génériques doivent pouvoir être utilisés sans dépendre de React.

---

## 4. Les composants restent simples

Les composants UI doivent principalement se concentrer sur :

* le rendu ;
* l'interaction ;
* l'état de présentation.

La logique complexe de gestion des erreurs doit rester dans les couches précédentes.

---

## 5. Le comportement global est centralisé

Les mécanismes globaux comme Toast, Popup et Alert doivent être accessibles de manière cohérente dans toute l'application.

---

## 6. La visibilité est contrôlée

L'UI ne doit jamais exposer automatiquement les détails techniques reçus du serveur.

---

# Objectif

La couche UI d'APIKit-CEL transforme une décision de traitement d'erreur en une expérience utilisateur cohérente.

Elle permet de conserver une séparation claire :

```text id="w6p3k8"
Server
→ décrit l'erreur

Contract
→ standardise l'erreur

Client
→ interprète l'erreur

Policy
→ décide du comportement

UI
→ présente le résultat
```

L'objectif final est de permettre à l'application de fournir un feedback adapté sans disperser la logique d'erreur dans ses composants.

> **L'UI ne doit pas découvrir ce qu'est une erreur : elle doit recevoir une décision claire sur la manière dont cette erreur doit être présentée.**
