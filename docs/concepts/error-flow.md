# Error Flow

## Définition

**Error Flow** décrit le parcours complet d'une erreur applicative depuis son origine sur le serveur jusqu'à son traitement et sa présentation éventuelle dans l'interface utilisateur.

L'objectif d'APIKit-CEL est de rendre ce parcours explicite, prévisible et observable.

Une erreur ne doit pas simplement être générée par le backend puis transmise au frontend. Elle doit suivre un flux défini permettant de :

* détecter l'erreur ;
* la classifier ;
* la transformer en contrat ;
* la transporter ;
* l'interpréter côté client ;
* déterminer son comportement ;
* contrôler sa visibilité ;
* fournir un retour approprié à l'utilisateur ou au développeur.

---

## Flux global

Le flux principal d'APIKit-CEL peut être représenté ainsi :

```text
┌──────────────┐
│  Exception   │
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│ Error Mapping    │
│ / Classification │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  Error Contract  │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│    Transport     │
│   HTTP / API     │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Client           │
│ Interpretation   │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│   Error Policy   │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│    Visibility    │
│     Policy       │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│       UI         │
└──────────────────┘
```

Chaque étape possède une responsabilité distincte.

---

## 1. Exception

Le flux commence lorsqu'une erreur se produit.

Elle peut provenir :

* du code métier ;
* d'une validation ;
* d'une dépendance externe ;
* d'une base de données ;
* d'un problème réseau ;
* d'une configuration ;
* d'une erreur de programmation ;
* d'une exception inattendue.

Exemple :

```python
raise UserNotFound(...)
```

ou :

```python
raise ValueError("Invalid configuration")
```

À ce stade, l'erreur est encore une notion interne au serveur.

Elle ne doit pas être directement exposée au client.

---

## 2. Error Mapping

L'exception doit être interprétée par le serveur afin de déterminer comment elle doit être représentée dans le système d'APIKit-CEL.

Cette étape permet notamment de déterminer :

* la catégorie de l'erreur ;
* son code ;
* son niveau de gravité ;
* les informations exploitables ;
* les informations pouvant être exposées ;
* le statut HTTP approprié ;
* les éventuelles informations de diagnostic.

Conceptuellement :

```text
Exception
    │
    ▼
Classification
    │
    ├── Validation
    ├── Authentication
    ├── Authorization
    ├── Not Found
    ├── Conflict
    ├── Business Error
    └── Internal Error
```

L'objectif est de transformer une exception technique en une information applicative compréhensible par le reste du système.

---

## 3. Error Contract

Une fois l'erreur identifiée, elle est représentée sous la forme d'un **Error Contract**.

Le contrat constitue la frontière entre l'implémentation serveur et le client.

```text
Server exception
       │
       ▼
Error Contract
       │
       ▼
Client
```

Le client ne doit pas avoir besoin de connaître la classe Python qui a généré l'erreur.

Il doit pouvoir travailler avec une représentation stable et prévisible.

Par exemple, le client peut reconnaître :

```text
VALIDATION_ERROR
RESOURCE_NOT_FOUND
AUTHENTICATION_REQUIRED
RESOURCE_CONFLICT
INTERNAL_ERROR
```

plutôt que de dépendre d'un message d'exception arbitraire.

---

## 4. Transport

Le contrat d'erreur est ensuite transporté jusqu'au client.

Dans une architecture HTTP classique :

```text
Error Contract
      │
      ▼
HTTP Response
      │
      ▼
Network
      │
      ▼
Client
```

Le transport fournit notamment le contexte HTTP de l'erreur.

Il est important de distinguer :

* **HTTP status** : indique la nature du résultat au niveau du protocole ;
* **error code** : identifie l'erreur au niveau applicatif ;
* **message** : fournit une information lisible ;
* **details** : fournit des informations structurées complémentaires.

Le client ne doit donc pas utiliser uniquement le statut HTTP pour déterminer précisément le comportement applicatif.

---

## 5. Client Interpretation

Le client reçoit la réponse et interprète le contrat.

Cette étape transforme la représentation transportée en une erreur exploitable par l'application cliente.

```text
HTTP Response
     │
     ▼
Client Error
     │
     ├── code
     ├── message
     ├── details
     └── metadata
```

Le client peut alors reconnaître l'erreur sans dépendre directement de l'implémentation du serveur.

Par exemple :

```text
RESOURCE_NOT_FOUND
```

peut être interprété comme :

```text
Resource not found
        │
        ▼
Application behavior
        │
        ├── update state
        ├── redirect
        └── notify user
```

---

## 6. Error Policy

Une fois l'erreur interprétée, le client doit déterminer **quoi faire**.

Cette décision appartient à la **Error Policy**.

Une erreur ne signifie pas nécessairement qu'il faut afficher un message.

Selon son type, elle peut entraîner :

* une notification ;
* une popup ;
* une redirection ;
* un retry ;
* un fallback ;
* une mise à jour d'état ;
* une interruption de l'action courante ;
* une journalisation ;
* plusieurs actions simultanées.

Exemple :

```text
AUTHENTICATION_REQUIRED
        │
        ▼
Error Policy
        │
        ▼
Redirect → Login
```

Autre exemple :

```text
NETWORK_ERROR
      │
      ▼
Error Policy
      │
      ├── Retry
      └── Notify user
```

La politique permet donc de centraliser le comportement attendu plutôt que de laisser chaque composant décider individuellement quoi faire.

---

## 7. Visibility Policy

Une fois le comportement déterminé, la politique de visibilité définit quelles informations peuvent être présentées.

Cette étape dépend notamment de l'environnement.

```text
Error Policy
     │
     ▼
Visibility Policy
     │
     ├── DEV
     ├── QUALIF
     └── PROD
```

Une même erreur peut donc avoir un comportement fonctionnel identique tout en ayant une visibilité différente.

Exemple :

```text
INTERNAL_ERROR
      │
      ├── DEV
      │     └── détails + stack trace
      │
      ├── QUALIF
      │     └── diagnostic contrôlé
      │
      └── PROD
            └── message utilisateur + référence
```

La visibilité est donc une dimension indépendante du traitement de l'erreur.

---

## 8. UI Feedback

La dernière étape correspond à la présentation éventuelle de l'erreur dans l'interface.

Le client peut utiliser différents mécanismes :

* Toast ;
* Alert ;
* Popup ;
* formulaire ;
* page d'erreur ;
* écran de fallback ;
* redirection.

Le choix dépend de la politique applicative et non du serveur.

```text
Error
  │
  ▼
Policy
  │
  ▼
UI behavior
  │
  ├── Toast
  ├── Alert
  ├── Popup
  ├── Redirect
  └── Fallback
```

Le serveur ne doit donc pas connaître le composant UI utilisé par le client.

---

# Exemple complet

Prenons une erreur inattendue produite pendant le traitement d'une requête.

```text
DatabaseError
```

### Étape 1 — Exception

Le serveur rencontre une exception :

```text
DatabaseError
```

### Étape 2 — Mapping

L'exception est classifiée comme erreur interne :

```text
INTERNAL_ERROR
```

### Étape 3 — Contract

Elle est transformée en contrat d'erreur.

```text
code    = INTERNAL_ERROR
message = ...
details = ...
```

Les informations techniques sensibles restent contrôlées.

### Étape 4 — Transport

Le serveur retourne une réponse HTTP appropriée contenant le contrat.

```text
HTTP Response
    │
    └── Error Contract
```

### Étape 5 — Client

Le client interprète :

```text
INTERNAL_ERROR
```

### Étape 6 — Policy

La politique cliente détermine :

```text
Display error
```

### Étape 7 — Visibility

L'environnement détermine le niveau d'information.

```text
DEV  → technical details
PROD → safe user message
```

### Étape 8 — UI

L'interface affiche finalement :

```text
DEV:
┌─────────────────────────────┐
│ Internal Error              │
│ DatabaseError               │
│ Stack trace...              │
└─────────────────────────────┘
```

ou en production :

```text
┌─────────────────────────────┐
│ Une erreur est survenue.    │
│ Référence : ERR-8F31A2      │
└─────────────────────────────┘
```

---

# Exemple : erreur de validation

Le même flux peut être utilisé pour une erreur fonctionnelle.

```text
InvalidRequest
      │
      ▼
VALIDATION_ERROR
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
Form handling
```

Dans ce cas, l'interface peut associer les erreurs aux champs concernés :

```text
email
└── Invalid email address

password
└── Password is too short
```

Il n'est donc pas nécessaire d'afficher une popup globale.

Le type d'erreur détermine le comportement approprié.

---

# Exemple : authentification

Une erreur d'authentification peut suivre un autre flux :

```text
AUTHENTICATION_REQUIRED
          │
          ▼
     Client Policy
          │
          ▼
     Clear session
          │
          ▼
      Redirect
          │
          ▼
        Login
```

L'erreur est bien traitée, mais son traitement principal n'est pas nécessairement l'affichage d'un message.

Cela illustre pourquoi **Error Handling** et **Error Visibility** doivent rester séparés.

---

# Prévention des erreurs silencieuses

Un objectif central d'APIKit-CEL est d'éviter ce scénario :

```text
Server Error
     │
     ▼
HTTP 500
     │
     ▼
Client receives error
     │
     ▼
Nothing happens
```

Dans ce cas, l'erreur existe techniquement mais aucune stratégie applicative n'est définie.

L'utilisateur peut alors observer :

* un bouton qui ne semble rien faire ;
* une interface qui reste dans un état incohérent ;
* une requête qui échoue sans feedback ;
* un état frontend qui ne correspond plus au serveur.

APIKit-CEL cherche à transformer ce flux en :

```text
Server Error
     │
     ▼
Error Contract
     │
     ▼
Client Interpretation
     │
     ▼
Error Policy
     │
     ▼
Explicit Behavior
```

L'erreur peut être affichée, traitée, journalisée ou redirigée, mais son comportement doit être défini.

---

# Responsabilités par couche

| Couche            | Responsabilité                                |
| ----------------- | --------------------------------------------- |
| Server            | Détecter et classifier l'erreur               |
| Error Mapping     | Transformer l'exception en erreur applicative |
| Error Contract    | Standardiser la représentation                |
| Transport         | Transmettre le contrat                        |
| Client            | Interpréter le contrat                        |
| Error Policy      | Déterminer le comportement                    |
| Visibility Policy | Contrôler les informations exposées           |
| UI                | Présenter le résultat lorsque nécessaire      |

Cette séparation permet de limiter le couplage entre les différentes couches.

---

# Principe architectural

Le flux complet peut être résumé par :

```text
Exception
    ↓
Contract
    ↓
Transport
    ↓
Interpretation
    ↓
Policy
    ↓
Visibility
    ↓
UI
```

Chaque étape répond à une question différente :

```text
Exception
→ Qu'est-ce qui s'est produit ?

Contract
→ Comment représenter cette erreur ?

Transport
→ Comment la transmettre ?

Interpretation
→ Qu'est-ce que le client comprend ?

Policy
→ Que doit faire l'application ?

Visibility
→ Quelles informations peut-on exposer ?

UI
→ Comment informer l'utilisateur ?
```

---

# Objectif

L'objectif d'**Error Flow** est de garantir qu'une erreur applicative ne soit pas simplement transmise d'une couche à l'autre, mais qu'elle soit accompagnée d'une stratégie explicite jusqu'à son traitement final.

APIKit-CEL cherche ainsi à établir une chaîne cohérente :

> **L'erreur est détectée, classifiée, contractualisée, transportée, interprétée, traitée et rendue visible selon le contexte.**

Le principe fondamental est :

> **Aucune erreur importante ne doit traverser les couches de l'application sans comportement explicitement défini.**
