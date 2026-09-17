# Error Contract

## Définition

Dans APIKit-CEL, un **Error Contract** définit la représentation commune d'une erreur entre les différentes couches d'une application.

Il permet au serveur et au client de partager une compréhension commune d'une erreur, indépendamment de l'implémentation interne du serveur ou du framework utilisé.

L'objectif est de ne pas transmettre simplement une exception ou un message arbitraire, mais de transformer l'erreur en une **information structurée, prévisible et exploitable par le client**.

```text
Exception
    │
    ▼
Error Mapping
    │
    ▼
Error Contract
    │
    ▼
HTTP Response
    │
    ▼
Client
```

Le contrat constitue ainsi le point de rencontre entre la gestion technique de l'erreur côté serveur et son interprétation côté client.

---

## Pourquoi un contrat d'erreur ?

Sans contrat commun, chaque endpoint ou chaque fonctionnalité peut retourner une structure différente :

```json
{
  "error": "User not found"
}
```

ou :

```json
{
  "message": "User does not exist",
  "status": 404
}
```

ou encore :

```json
{
  "errors": [
    {
      "field": "email",
      "message": "Invalid email"
    }
  ]
}
```

Cette diversité oblige le frontend à connaître les particularités de chaque endpoint.

Le résultat peut être :

* du traitement d'erreur dupliqué ;
* des comportements différents selon les fonctionnalités ;
* des erreurs mal interprétées ;
* des informations perdues lors du transport ;
* des erreurs qui ne produisent aucun retour utilisateur.

APIKit-CEL cherche à remplacer cette approche par un **contrat d'erreur commun**.

---

## Responsabilité du contrat

Un Error Contract doit permettre au client de répondre à plusieurs questions :

* Quelle erreur s'est produite ?
* Quelle est sa nature ?
* Peut-elle être affichée à l'utilisateur ?
* Existe-t-il des informations complémentaires ?
* Peut-elle être associée à un champ de formulaire ?
* Existe-t-il une action particulière à effectuer ?
* Comment l'erreur peut-elle être identifiée dans les logs ?

Le contrat ne doit cependant pas imposer au client la manière exacte de présenter l'erreur.

Il fournit **l'information nécessaire à la décision**, tandis que la politique de gestion détermine le comportement approprié.

```text
                    Error Contract
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
         Identité      Contexte     Détails
             │            │            │
             └────────────┼────────────┘
                          ▼
                    Error Policy
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
           Toast         Alert       Popup
```

---

## Structure conceptuelle

Le contrat d'erreur d'APIKit-CEL est conçu autour de plusieurs catégories d'informations.

```text
Error Contract
│
├── Identification
│   ├── code
│   └── correlation / reference
│
├── Description
│   ├── message
│   └── details
│
├── Classification
│   ├── error type
│   └── severity
│
└── Context
    ├── validation data
    └── additional metadata
```

La structure exacte peut évoluer selon les besoins du système, mais le principe reste le même :

> **Le contrat décrit l'erreur sans imposer sa présentation.**

---

## Error Code

Le **code d'erreur** permet d'identifier l'erreur de manière stable.

Exemple :

```text
USER_NOT_FOUND
AUTHENTICATION_REQUIRED
VALIDATION_ERROR
RESOURCE_CONFLICT
INTERNAL_ERROR
```

Le code est préférable à l'utilisation du message comme identifiant.

Le message peut changer :

```text
"Utilisateur introuvable"
```

puis :

```text
"Impossible de trouver cet utilisateur"
```

alors que le code reste :

```text
USER_NOT_FOUND
```

Le frontend peut donc baser son comportement sur le code plutôt que sur le texte.

```text
USER_NOT_FOUND
       │
       ▼
Error Policy
       │
       ▼
Message utilisateur
```

Cela permet également de traduire les messages sans modifier la logique de traitement.

---

## Message

Le contrat peut contenir un message associé à l'erreur.

Cependant, le message ne doit pas nécessairement être considéré comme une information directement destinée à l'utilisateur.

Il peut s'agir d'un message technique, fonctionnel ou contextuel selon le niveau de visibilité configuré.

La politique de visibilité détermine ensuite si cette information peut être exposée.

```text
Error Contract
      │
      ├── DEV
      │    └── message détaillé
      │
      ├── QUALIF
      │    └── message contrôlé
      │
      └── PROD
           └── message utilisateur
```

---

## Details

Les détails permettent de transporter des informations complémentaires sans modifier la structure principale du contrat.

Ils peuvent notamment contenir :

* des informations de validation ;
* des paramètres concernés ;
* des informations fonctionnelles ;
* des données utiles au diagnostic ;
* des métadonnées.

Exemple conceptuel :

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "details": {
    "fields": {
      "email": "Invalid email address",
      "password": "Password is too short"
    }
  }
}
```

Le client peut alors transformer ces informations en erreurs de formulaire plutôt qu'en simple notification globale.

---

## Erreurs de validation

Les erreurs de validation constituent un cas particulier important.

Une validation peut concerner :

* la requête ;
* un champ ;
* plusieurs champs ;
* une règle métier.

Le contrat doit permettre au client d'identifier précisément les éléments concernés.

```text
Validation Error
       │
       ├── email
       │     └── invalid format
       │
       └── password
             └── insufficient length
```

Cela permet une présentation adaptée :

```text
Email
└── Adresse e-mail invalide

Password
└── Le mot de passe doit contenir au moins 8 caractères
```

plutôt qu'une erreur générique affichée dans un popup.

---

## Correlation et traçabilité

Lorsqu'une erreur se produit en production, les informations affichées à l'utilisateur doivent rester limitées.

Cependant, le problème doit pouvoir être retrouvé côté système.

Un identifiant de corrélation ou de référence peut permettre d'établir un lien entre :

```text
Utilisateur
     │
     ▼
Réponse API
     │
     └── correlation ID
              │
              ▼
           Logs
              │
              ▼
       Diagnostic serveur
```

L'utilisateur peut ainsi recevoir :

```text
Une erreur est survenue.

Référence : 8F32A1
```

tandis que les informations techniques restent disponibles dans les logs.

---

## Séparation entre contrat et présentation

Un principe important d'APIKit-CEL est de ne pas mélanger le contrat d'erreur avec la présentation UI.

Le serveur ne devrait pas décider :

```text
"Cette erreur doit afficher un Toast React."
```

Il fournit plutôt :

```text
USER_NOT_FOUND
```

avec les informations nécessaires.

Le client applique ensuite une politique :

```text
USER_NOT_FOUND
      │
      ▼
Error Policy
      │
      ▼
Toast
```

Une autre application pourrait décider :

```text
USER_NOT_FOUND
      │
      ▼
Error Policy
      │
      ▼
Alert
```

Le contrat reste identique.

Cette séparation permet à APIKit-CEL de rester indépendant de la technologie d'interface utilisée.

---

## Contrat stable, implémentations variables

Le contrat constitue une frontière entre les différentes couches.

```text
┌──────────────────────┐
│      Backend         │
│                      │
│ Exception / Domain   │
└──────────┬───────────┘
           │
           ▼
     Error Contract
           │
           ▼
┌──────────────────────┐
│       Client         │
│                      │
│ Interpretation       │
└──────────┬───────────┘
           │
           ▼
      Error Policy
           │
           ▼
┌──────────────────────┐
│          UI          │
│                      │
│ Toast / Alert / ...  │
└──────────────────────┘
```

Le backend peut utiliser Python et Django, tandis que le client peut utiliser TypeScript et React.

Le contrat reste indépendant de ces technologies.

---

## Ce que le contrat ne définit pas

L'Error Contract ne doit pas définir directement :

* le composant UI utilisé ;
* le type de notification ;
* le style visuel ;
* le comportement exact de l'écran ;
* les détails internes du framework frontend ;
* les mécanismes internes de logging.

Ces responsabilités appartiennent aux couches supérieures.

```text
Error Contract
    │
    │ décrit l'erreur
    ▼
Error Policy
    │
    │ décide du comportement
    ▼
UI / Application
    │
    │ présente ou traite
    ▼
Utilisateur
```

---

## Objectif architectural

L'Error Contract permet ainsi de créer une frontière stable entre les systèmes.

```text
┌───────────────┐
│    Backend    │
└───────┬───────┘
        │
        │  Error Contract
        ▼
┌───────────────┐
│     Client    │
└───────┬───────┘
        │
        │  Error Policy
        ▼
┌───────────────┐
│      UI       │
└───────────────┘
```

Cette séparation permet :

* de réduire le couplage entre backend et frontend ;
* de standardiser les erreurs ;
* de simplifier leur interprétation ;
* d'éviter les traitements spécifiques par endpoint ;
* de conserver les informations importantes lors du transport ;
* de faire évoluer indépendamment les différentes couches.

## Principe

L'Error Contract peut finalement être résumé par un principe :

> **Le serveur décrit l'erreur, le contrat la transporte, le client l'interprète et la politique détermine son comportement.**

APIKit-CEL utilise ainsi le contrat d'erreur comme une **frontière commune entre les couches applicatives**, permettant de construire une gestion des erreurs cohérente tout en laissant chaque couche responsable de son propre comportement.
