# Client

## Vue d'ensemble

La partie client d'**APIKit-CEL** est responsable de l'interprétation des erreurs reçues du serveur et de la détermination du comportement approprié côté application.

Elle constitue la seconde moitié du flux d'erreur :

```text id="c8m4x2"
Server
   │
   │ Error Contract
   ▼
Client
   │
   ├── Interpretation
   ├── Error Policy
   └── Visibility
        │
        ▼
       UI
```

Le client ne doit pas avoir besoin de connaître les exceptions internes du serveur.

Il travaille à partir du contrat d'erreur défini par APIKit-CEL.

---

# Organisation

L'implémentation TypeScript est organisée en deux niveaux :

```text id="p7k3n9"
apikit-typescript
│
├── apikit-client-core
│
└── apikit-react
```

## `apikit-client-core`

Le core contient les mécanismes indépendants du framework :

* représentation des erreurs ;
* interprétation du contrat ;
* gestion des erreurs transport ;
* politiques de traitement ;
* normalisation des erreurs ;
* logique commune de traitement.

Le core doit pouvoir être utilisé indépendamment de React.

---

## `apikit-react`

`apikit-react` fournit l'intégration avec React.

Il prend notamment en charge les mécanismes nécessaires à la présentation des erreurs et notifications dans l'interface :

```text id="m5r8q1"
apikit-client-core
        │
        ▼
   apikit-react
        │
        ▼
   React Application
```

Cette séparation permet de conserver la logique métier et la logique d'erreur indépendantes du framework UI.

---

# Pipeline client

Le traitement d'une erreur côté client peut être représenté ainsi :

```text id="x6v2p8"
HTTP Response
      │
      ▼
Transport Handling
      │
      ▼
Error Interpretation
      │
      ▼
Error Normalization
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

Chaque étape possède une responsabilité spécifique.

---

# 1. Réception de la réponse

Le client reçoit une réponse HTTP provenant du serveur.

```text id="q4m9c2"
HTTP Request
     │
     ▼
Server
     │
     ▼
HTTP Response
     │
     ▼
Client
```

La réponse peut représenter :

* un succès ;
* une erreur applicative ;
* une erreur de validation ;
* une erreur d'authentification ;
* une erreur d'autorisation ;
* une erreur serveur ;
* une erreur réseau ou transport.

Le client doit être capable de distinguer ces différents cas.

---

# 2. Transport Handling

Toutes les erreurs reçues par le client ne proviennent pas nécessairement du contrat applicatif.

Certaines erreurs peuvent se produire avant même que le serveur ait pu produire un `Error Contract`.

Exemples :

```text id="j8c3v6"
Network unavailable
Connection timeout
DNS failure
Request cancelled
Malformed response
```

Le client doit donc distinguer :

```text id="r2k7m5"
                    Client Error
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
       Application Error      Transport Error
              │                     │
       Error Contract          Client Error
```

Cette distinction permet de traiter correctement les erreurs provenant du serveur et celles provenant du transport.

---

# 3. Error Interpretation

Lorsqu'un contrat d'erreur est reçu, le client doit l'interpréter.

```text id="f9m4x2"
HTTP Response
      │
      ▼
Error Contract
      │
      ▼
Client Error
```

L'interprétation permet notamment d'identifier :

* le code d'erreur ;
* le message ;
* les détails ;
* les informations de validation ;
* les métadonnées ;
* la référence de diagnostic.

Le client ne doit pas avoir besoin de connaître la classe d'exception originale.

---

# Error Code

Le **code d'erreur** constitue l'une des informations les plus importantes pour la logique cliente.

Le client doit privilégier un identifiant stable :

```text id="v3k8p1"
RESOURCE_NOT_FOUND
VALIDATION_ERROR
AUTHENTICATION_REQUIRED
RESOURCE_CONFLICT
INTERNAL_ERROR
```

plutôt qu'un message :

```text id="s5n2q7"
"User not found"
```

Le message est destiné à l'information humaine.

Le code est destiné à la logique applicative.

```text id="d7m4c9"
Error Code
    │
    ├── Application Logic
    └── Policy Selection

Message
    │
    └── User / Developer Information
```

---

# Error Normalization

Le client peut recevoir différentes formes d'erreurs selon leur origine.

L'objectif de la normalisation est de présenter une représentation commune au reste de l'application.

```text id="k6r1x8"
Server Error Contract ─────┐
                           │
Network Error ─────────────┼──▶ Client Error
                           │
Parsing Error ─────────────┘
```

Les composants consommateurs n'ont ainsi pas besoin de connaître les détails de chaque mécanisme de transport.

---

# Error Policy

Une fois l'erreur interprétée et normalisée, le client doit déterminer ce qu'il doit faire.

Cette responsabilité appartient à la **Error Policy**.

```text id="n8q3w5"
Client Error
     │
     ▼
Error Policy
     │
     ├── Notify
     ├── Redirect
     ├── Retry
     ├── Fallback
     ├── Update State
     └── Ignore intentionally
```

Le principe est de centraliser les décisions plutôt que de laisser chaque composant React implémenter sa propre logique.

---

# Exemples de policies

## Validation

```text id="c5m9r2"
VALIDATION_ERROR
       │
       ▼
Error Policy
       │
       ▼
Update form errors
```

L'erreur est associée aux champs concernés.

---

## Authentification

```text id="p4x7k1"
AUTHENTICATION_REQUIRED
       │
       ▼
Error Policy
       │
       ├── Clear authentication state
       └── Redirect to login
```

L'affichage d'un message peut être secondaire ou inutile.

---

## Ressource inexistante

```text id="v8n3m6"
RESOURCE_NOT_FOUND
       │
       ▼
Error Policy
       │
       ├── Update state
       └── Display notification
```

---

## Erreur réseau

```text id="j2k6q9"
NETWORK_ERROR
      │
      ▼
Error Policy
      │
      ├── Retry
      └── Notify user
```

---

## Erreur interne

```text id="r7m4p1"
INTERNAL_ERROR
      │
      ▼
Error Policy
      │
      ▼
Display controlled feedback
```

Le comportement de présentation dépend ensuite de la politique de visibilité.

---

# Visibility Policy

La **Visibility Policy** détermine quelles informations peuvent être exposées.

```text id="x4c8m2"
Client Error
     │
     ▼
Visibility Policy
     │
     ├── DEV
     ├── QUALIF
     └── PROD
```

Par exemple :

```text id="w6n1p9"
DEV
→ message + détails techniques

QUALIF
→ message + diagnostic contrôlé

PROD
→ message utilisateur + référence
```

Le client ne doit pas nécessairement afficher toutes les informations présentes dans le contrat.

---

# Gestion de la visibilité

La visibilité peut être utilisée pour contrôler la présentation d'une même erreur.

```text id="g3r8k5"
                  INTERNAL_ERROR
                        │
             ┌──────────┴──────────┐
             │                     │
            DEV                   PROD
             │                     │
             ▼                     ▼
      Technical details       Safe message
      Stack trace             Error reference
```

Cela permet de conserver une expérience adaptée à chaque environnement.

---

# UI Integration

La couche React reçoit les décisions produites par le core et les traduit en comportements UI.

```text id="m9c4x7"
                 Client Core
                     │
                     ▼
                Error Policy
                     │
                     ▼
                UI Behavior
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        Toast      Alert      Popup
```

Le core ne doit pas être directement dépendant de React.

React devient une implémentation de présentation.

---

# Global UI Infrastructure

`apikit-react` fournit une infrastructure permettant de centraliser certains comportements UI.

Le `GlobalUiProvider` regroupe notamment :

```text id="q8v2m6"
GlobalUiProvider
       │
       ├── GlobalToastProvider
       ├── GlobalPopupProvider
       └── GlobalAlertProvider
```

Cette architecture permet aux différentes parties de l'application de déclencher des comportements UI globaux sans devoir gérer directement leur état.

---

# Services UI

Les mécanismes UI peuvent être exposés au travers de services spécialisés.

Conceptuellement :

```text id="d5k9x1"
Error Policy
     │
     ├── ToastService
     ├── PopupService
     └── AlertService
```

La policy décide **qu'un feedback est nécessaire**.

Le service détermine ensuite **comment déclencher ce feedback**.

Cette séparation permet d'éviter de mélanger :

```text
Error logic
     +
UI state management
```

---

# Exemple de flux complet

Prenons une erreur serveur :

```text id="h7m3p8"
RESOURCE_NOT_FOUND
```

Le flux côté client devient :

```text id="a4c9x2"
HTTP Response
      │
      ▼
Error Contract
      │
      ▼
Client Interpretation
      │
      ▼
RESOURCE_NOT_FOUND
      │
      ▼
Error Policy
      │
      ├── Update application state
      └── Notify user
              │
              ▼
            Toast
```

La décision UI n'est donc pas prise directement par le serveur.

---

# Exemple : erreur de validation

Pour une erreur de validation :

```text id="k2n8v5"
VALIDATION_ERROR
      │
      ▼
Client Interpretation
      │
      ▼
Error Policy
      │
      ▼
Form State
      │
      ├── email → invalid
      └── password → invalid
```

Le résultat peut être directement intégré au formulaire sans afficher de notification globale.

---

# Exemple : erreur inattendue

Pour une erreur interne :

```text id="p6r1m9"
INTERNAL_ERROR
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
      ├── DEV  → technical popup
      └── PROD → safe alert
```

Le comportement reste cohérent tout en adaptant l'information exposée.

---

# Prévention des erreurs silencieuses

Le client joue un rôle essentiel dans l'objectif principal d'APIKit-CEL :

> **Éviter qu'une erreur reçue par l'application soit ignorée sans stratégie explicite.**

Un mauvais flux pourrait être :

```text id="x9c4m7"
Server Error
     │
     ▼
Client
     │
     ▼
Unhandled
     │
     ▼
Nothing
```

APIKit-CEL cherche plutôt à obtenir :

```text id="v3k8p6"
Server Error
     │
     ▼
Error Contract
     │
     ▼
Interpretation
     │
     ▼
Error Policy
     │
     ▼
Explicit Behavior
```

Le comportement peut être silencieux pour l'utilisateur si cela est volontaire.

Mais il doit être défini par la politique applicative.

---

# Séparation Core / React

L'architecture client suit le même principe que l'architecture serveur :

```text id="m4q8x1"
             apikit-client-core
                      │
                      │ contracts / policies
                      ▼
                 apikit-react
                      │
                      ▼
                React Application
```

Le core définit les concepts.

React fournit leur intégration dans l'interface.

Cette séparation permet notamment d'envisager d'autres intégrations :

```text id="r7c2n5"
apikit-client-core
       │
       ├── React
       ├── Vue
       ├── Angular
       └── Other client
```

---

# Responsabilités du client

| Responsabilité     | Description                                    |
| ------------------ | ---------------------------------------------- |
| Transport handling | Gérer les erreurs réseau et transport          |
| Interpretation     | Comprendre le Error Contract                   |
| Normalization      | Fournir une représentation commune côté client |
| Error Policy       | Déterminer le comportement applicatif          |
| Visibility         | Contrôler les informations exposées            |
| UI Integration     | Transmettre les décisions à l'interface        |
| State handling     | Adapter l'état de l'application                |
| User feedback      | Présenter un feedback lorsque nécessaire       |

---

# Principes architecturaux

## Le client ne dépend pas des exceptions serveur

Le client travaille avec le contrat et non avec les classes d'exception Python.

```text id="c6m9r4"
Python Exception
       X
       │
       │ not exposed as implementation dependency
       ▼
Error Contract
       │
       ▼
TypeScript Client
```

---

## Le code d'erreur pilote la logique

Les policies doivent privilégier des identifiants stables plutôt que des messages textuels.

```text id="q5n8x2"
Error Code
    ↓
Policy

Message
    ↓
Presentation
```

---

## L'UI ne décide pas de la sémantique de l'erreur

Un composant React ne devrait pas avoir à déterminer seul ce que signifie :

```text id="v8p3m6"
AUTHENTICATION_REQUIRED
```

La policy doit fournir cette décision.

L'UI se concentre sur la présentation.

---

## Le framework reste une couche d'intégration

React ne doit pas devenir une dépendance du cœur de gestion des erreurs.

```text id="j4r7k1"
Core
 │
 └── Framework Adapter
          │
          ▼
        React
```

---

# Flux complet client

La partie client peut finalement être résumée ainsi :

```text id="z6m2q8"
                    HTTP Response
                          │
                          ▼
                  Transport Handling
                          │
                          ▼
                  Error Interpretation
                          │
                          ▼
                  Error Normalization
                          │
                          ▼
                     Error Policy
                          │
                          ▼
                   Visibility Policy
                          │
                          ▼
                     UI Behavior
                          │
               ┌──────────┼──────────┐
               ▼          ▼          ▼
             Toast      Alert      Popup
```

---

# Relation avec le serveur

Le client complète le flux produit par le serveur :

```text id="n8c5v2"
┌──────────────────────── SERVER ────────────────────────┐
│                                                         │
│ Exception                                               │
│    ↓                                                    │
│ Error Mapping                                           │
│    ↓                                                    │
│ Error Contract                                          │
└──────────────────────────┬──────────────────────────────┘
                           │
                           │ HTTP
                           ▼
┌──────────────────────── CLIENT ────────────────────────┐
│                                                         │
│ Interpretation                                          │
│    ↓                                                    │
│ Error Policy                                            │
│    ↓                                                    │
│ Visibility Policy                                       │
│    ↓                                                    │
│ UI Behavior                                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Le contrat constitue ainsi la frontière entre les deux parties.

---

# Objectif

L'architecture client d'APIKit-CEL vise à garantir qu'une erreur reçue par l'application ne soit pas simplement rejetée ou ignorée.

Elle doit être :

* comprise ;
* normalisée ;
* associée à une politique ;
* traitée explicitement ;
* rendue visible lorsque nécessaire ;
* présentée de manière adaptée au contexte.

Le principe fondamental est :

> **Le client ne se contente pas de recevoir une erreur : il l'interprète, détermine son comportement et contrôle la manière dont elle est présentée à l'utilisateur.**
