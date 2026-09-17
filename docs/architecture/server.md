# Server

## Vue d'ensemble

La partie serveur d'**APIKit-CEL** est responsable de la détection, de la classification et de la transformation des erreurs applicatives en un contrat exploitable par les clients.

L'architecture serveur sépare les mécanismes génériques de l'intégration avec un framework particulier.

```text id="s8k2m4"
                 APIKit-CEL Server
                        │
                        ▼
                apikit-server-core
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
        API processing       Error handling
              │                   │
              └─────────┬─────────┘
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

---

# Organisation

L'implémentation Python est organisée en deux niveaux :

```text id="p4n7c1"
apikit-python
│
├── apikit-server-core
│
└── apikit-server-django
```

## `apikit-server-core`

Le core contient les mécanismes indépendants du framework :

* configuration ;
* contrats ;
* validation ;
* traitement des requêtes ;
* validation des réponses ;
* mapping des exceptions ;
* observation des erreurs ;
* génération des réponses API ;
* métadonnées des vues.

Le core constitue le cœur architectural d'APIKit-CEL côté serveur.

---

## `apikit-server-django`

L'adapter Django fournit l'intégration entre le core et Django.

Il prend notamment en charge les particularités du framework :

* récupération des données de requête ;
* accès au body HTTP ;
* récupération des paramètres ;
* construction des réponses ;
* intégration avec les serializers lorsque nécessaire ;
* adaptation des objets Django vers les contrats APIKit-CEL.

Le principe est :

```text id="c6w9r2"
             APIKit Server
                  │
                  ▼
          Framework-independent
                  │
                  ▼
        apikit-server-core
                  │
                  ▼
         Framework adapter
                  │
                  ▼
               Django
```

Le core ne doit pas dépendre directement de Django.

---

# Pipeline serveur

Le traitement principal d'une route peut être représenté par une chaîne :

```text id="m3f8q1"
HTTP Request
     │
     ▼
Method Validation
     │
     ▼
Request Validation
     │
     ▼
View Metadata
     │
     ▼
Application View
     │
     ▼
Response Validation
     │
     ▼
API Response
```

Les exceptions produites pendant ce traitement sont interceptées par le mécanisme de gestion des erreurs.

```text id="v7k2p5"
                  Request
                     │
                     ▼
              API Route Pipeline
                     │
          ┌──────────┴──────────┐
          │                     │
       Success                 Error
          │                     │
          ▼                     ▼
    API Response          Exception Handling
                                │
                                ▼
                          Error Contract
                                │
                                ▼
                          API Response
```

---

# API Route

Le décorateur `api_route` constitue le point d'entrée principal du pipeline serveur.

Conceptuellement :

```python
@api_route(...)
def view(...):
    ...
```

Il centralise plusieurs responsabilités autour d'une même route.

Le pipeline peut être résumé ainsi :

```text id="x2n6q8"
api_route
    │
    ├── require_methods
    │
    ├── validate_request
    │
    ├── ViewMeta
    │
    ├── validate_api_response
    │
    └── handle_api_exceptions
```

L'intérêt est d'éviter que chaque vue réimplémente individuellement les mêmes mécanismes.

---

# Validation de la requête

Une requête entrante peut être validée avant l'exécution de la logique applicative.

```text id="h5r9c2"
HTTP Request
     │
     ▼
Request Adapter
     │
     ▼
Request Contract
     │
     ▼
Validator
     │
     ▼
Validated Request Data
```

La validation permet notamment de vérifier :

* la présence des données attendues ;
* leur structure ;
* leur type ;
* les contraintes métier ou applicatives définies par le validator.

Une erreur de validation doit être transformée en erreur applicative structurée plutôt que de remonter sous la forme d'une exception arbitraire.

---

# Request Contract

Le `RequestContract` définit l'interface attendue pour accéder aux informations de la requête.

Le core ne doit donc pas dépendre directement d'une API spécifique à Django.

Conceptuellement :

```text id="k7v3p9"
                 Request
                    │
                    ▼
            RequestContract
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
     Django                   Other
     Adapter                  Adapter
```

Cette abstraction permet de conserver la logique de traitement indépendante du framework.

---

# Validation de la réponse

Le serveur peut également valider les réponses produites par les vues.

```text id="r8c1m5"
Application View
      │
      ▼
Response Validation
      │
      ▼
API Response
```

Cette validation permet de détecter les réponses qui ne respectent pas le format attendu avant qu'elles soient retournées au client.

Le principe est important pour APIKit-CEL :

> **Le contrat ne concerne pas uniquement les erreurs ; il participe à la standardisation des échanges entre le serveur et le client.**

---

# Response Contract

Le `ResponseContract` fournit une abstraction permettant au core de construire ou manipuler une réponse sans dépendre directement du framework HTTP.

Conceptuellement :

```text id="f6m2q8"
                 APIKit Core
                     │
                     ▼
             ResponseContract
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
       Django                Other
       Response              Adapter
```

L'adapter transforme ensuite cette abstraction en réponse native du framework.

---

# Gestion des exceptions

La gestion des exceptions est une partie centrale du serveur APIKit-CEL.

Le principe est de ne pas laisser chaque vue décider individuellement comment transformer une exception en réponse API.

```text id="q3h8n1"
Exception
    │
    ▼
Exception Handler
    │
    ▼
Exception Mapping
    │
    ▼
Error Data
    │
    ▼
API Error Response
```

Le système distingue notamment :

* les erreurs applicatives connues ;
* les erreurs de validation ;
* les erreurs de configuration ou de traitement ;
* les exceptions inattendues.

---

# Exception Mapping

Le mapping permet d'associer une exception à une représentation applicative.

```text id="w9k4p2"
Exception
    │
    ▼
Exception Registry / Mapping
    │
    ▼
Error Classification
    │
    ▼
Error Data
```

L'objectif est d'éviter que le client doive connaître les détails des exceptions Python.

Par exemple :

```text id="n4c7x1"
UserNotFound
     │
     ▼
RESOURCE_NOT_FOUND
```

ou :

```text id="z5m8q3"
InvalidRequest
     │
     ▼
VALIDATION_ERROR
```

Le client travaille ainsi avec une représentation applicative stable.

---

# Error Data

`ErrorData` constitue une représentation structurée de l'erreur.

Elle permet de séparer :

```text id="a7p2m6"
Exception interne
      │
      ▼
ErrorData
      │
      ▼
Error Contract
```

Cette séparation est importante car l'exception Python ne doit pas être considérée comme le contrat public de l'API.

Le contrat public doit rester stable même si l'implémentation interne change.

---

# Exception Observer

La gestion d'une erreur ne doit pas empêcher son observation.

APIKit-CEL fournit donc un mécanisme d'observation des exceptions.

```text id="u8f3k7"
                  Exception
                      │
                      ▼
              Exception Handling
                      │
              ┌───────┴────────┐
              │                │
              ▼                ▼
          Error Mapping     Observers
              │                │
              ▼                ├── Logging
        Error Contract         ├── Monitoring
                               └── Diagnostics
```

Le système d'observation permet notamment de connecter la gestion des erreurs à des mécanismes de logging ou de diagnostic sans coupler directement le cœur à ces systèmes.

---

# Observer Pattern

Le mécanisme d'observation repose sur une séparation entre :

* l'événement ;
* le dispatcher ;
* les observers.

Conceptuellement :

```text id="d1r6k9"
Exception Event
      │
      ▼
Dispatcher
      │
      ├── Logger Observer
      ├── Monitoring Observer
      └── Custom Observer
```

Cette approche permet d'ajouter de nouveaux comportements sans modifier le mécanisme principal de traitement des exceptions.

---

# Exception Events

Les événements d'exception permettent de transporter le contexte nécessaire à l'observation.

Ils peuvent représenter différentes situations du cycle de traitement d'une erreur.

```text id="e5q8w2"
Application
    │
    ▼
Exception
    │
    ▼
Exception Event
    │
    ├── classification
    ├── context
    ├── metadata
    └── exception information
```

L'événement est destiné à l'observation et au diagnostic.

Il ne doit pas être confondu avec le **Error Contract**, qui est destiné à l'échange avec le client.

---

# Séparation Error Event / Error Contract

Cette distinction est importante :

```text id="p7n4c8"
                    Exception
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
       Exception Event        Error Contract
             │                     │
             ▼                     ▼
        Observability           API Client
```

### Exception Event

Destiné aux systèmes internes :

* logs ;
* monitoring ;
* diagnostics ;
* observabilité.

### Error Contract

Destiné au consommateur de l'API :

* code ;
* message ;
* détails ;
* informations fonctionnelles ;
* référence d'erreur.

Cela permet de conserver des informations techniques côté serveur sans nécessairement les exposer au client.

---

# Configuration

Le serveur fournit une configuration centralisée permettant notamment de définir :

* la visibilité des erreurs ;
* les observers ;
* les adapters de requête ;
* les adapters de réponse ;
* le mapping des exceptions ;
* le logging.

Conceptuellement :

```text id="c9m5x2"
              Configuration
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Visibility    Adapters    Exception
                             Handling
       │            │            │
       └────────────┴────────────┘
                    │
                    ▼
             Server Pipeline
```

Cette configuration permet de centraliser les règles transverses plutôt que de les répéter dans chaque route.

---

# Environnement et visibilité

La configuration serveur participe également à la stratégie de visibilité définie par APIKit-CEL.

```text id="k4r8v1"
Environment
    │
    ├── DEV
    ├── QUALIF
    └── PROD
         │
         ▼
Error Visibility
```

Le serveur peut ainsi conserver une représentation riche de l'erreur tout en contrôlant les informations exposées dans la réponse.

```text id="y2m7q5"
Internal Exception
      │
      ▼
Error Processing
      │
      ├───────────────┐
      ▼               ▼
Diagnostic        Public Error
information        Contract
      │               │
      ▼               ▼
   Server           Client
```

---

# Framework Adapter

Le principe d'adapter permet d'isoler les particularités du framework.

```text id="f8q3m6"
                  Core
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
       Contract          Contract
          │                 │
          ▼                 ▼
       Django             Other
       Adapter             Adapter
```

L'adapter Django est responsable de la traduction entre les abstractions APIKit-CEL et les objets Django.

Le core ne doit pas contenir de logique spécifique à Django lorsqu'une abstraction permet de l'éviter.

---

# Flux complet serveur

Le fonctionnement serveur peut finalement être résumé ainsi :

```text id="s6k1p4"
                    HTTP Request
                         │
                         ▼
                  Request Adapter
                         │
                         ▼
                  Method Validation
                         │
                         ▼
                  Request Validation
                         │
                         ▼
                    Application
                         │
              ┌──────────┴──────────┐
              │                     │
           Success                 Error
              │                     │
              ▼                     ▼
      Response Validation     Exception Handler
              │                     │
              │                     ▼
              │               Exception Mapping
              │                     │
              │              ┌──────┴──────┐
              │              │             │
              │              ▼             ▼
              │         Exception       Error Data
              │           Event             │
              │              │               │
              │              ▼               ▼
              │          Observers     Error Contract
              │                              │
              └──────────────┬───────────────┘
                             ▼
                       HTTP Response
                             │
                             ▼
                           Client
```

---

# Responsabilités du serveur

La partie serveur d'APIKit-CEL peut être résumée par les responsabilités suivantes :

| Responsabilité        | Description                                          |
| --------------------- | ---------------------------------------------------- |
| Request handling      | Accéder et normaliser les données entrantes          |
| Validation            | Vérifier les requêtes et éventuellement les réponses |
| Exception handling    | Intercepter et traiter les exceptions                |
| Error mapping         | Transformer les exceptions en erreurs applicatives   |
| Error Contract        | Produire une représentation stable pour le client    |
| Observability         | Publier les événements nécessaires au diagnostic     |
| Visibility            | Contrôler les informations exposées                  |
| Framework integration | Adapter APIKit-CEL au framework utilisé              |

---

# Principes architecturaux

## Core indépendant du framework

Le cœur doit rester indépendant de Django ou de tout autre framework.

## Adapters explicites

Les dépendances spécifiques à un framework doivent être isolées dans des adapters.

## Contrat stable

Les exceptions internes ne doivent pas devenir le contrat public de l'API.

## Gestion centralisée

Les règles communes de traitement des erreurs doivent être centralisées plutôt que dupliquées dans les vues.

## Observabilité séparée

Le diagnostic interne doit être découplé de la réponse envoyée au client.

## Visibilité contrôlée

Les informations techniques doivent être exposées selon une politique définie et adaptée à l'environnement.

---

# Relation avec le client

Le serveur constitue la première moitié du flux APIKit-CEL.

```text id="q6v9b3"
┌──────────────────────┐
│        SERVER        │
│                      │
│ Exception            │
│      ↓               │
│ Mapping              │
│      ↓               │
│ Error Contract       │
└──────────┬───────────┘
           │
           │ HTTP
           ▼
┌──────────────────────┐
│        CLIENT        │
│                      │
│ Interpretation       │
│      ↓               │
│ Error Policy         │
│      ↓               │
│ Visibility           │
│      ↓               │
│ UI                   │
└──────────────────────┘
```

Le serveur n'a donc pas pour responsabilité de définir le comportement final de l'interface.

Il fournit un contrat suffisamment structuré pour que le client puisse prendre cette décision.

---

# Objectif

L'architecture serveur d'APIKit-CEL vise à transformer les exceptions internes en informations applicatives **standardisées, observables et contrôlées**.

Le serveur doit être capable de répondre à trois questions :

```text id="r3m8x5"
1. Qu'est-ce qui s'est produit ?
2. Comment cette erreur doit-elle être représentée ?
3. Quelles informations peuvent être transmises au client ?
```

Le comportement final appartient ensuite au client.

> **Le serveur ne dicte pas l'expérience utilisateur : il fournit au client un contrat d'erreur fiable, contextualisé et exploitable.**
