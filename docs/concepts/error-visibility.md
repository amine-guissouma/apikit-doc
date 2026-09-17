# Error Visibility

## Définition

**Error Visibility** désigne la manière dont une erreur applicative est rendue visible selon le contexte d'exécution et le profil de l'utilisateur.

Une erreur peut être correctement détectée et traitée par l'application sans pour autant être affichée de la même manière à tous les utilisateurs.

APIKit-CEL sépare donc la **gestion de l'erreur** de sa **visibilité** :

* la gestion détermine ce que l'application doit faire ;
* la visibilité détermine quelles informations doivent être exposées et à qui.

Cette distinction permet d'afficher des informations techniques utiles aux développeurs sans exposer de détails internes aux utilisateurs finaux.

---

## Pourquoi gérer la visibilité des erreurs ?

Une même erreur peut nécessiter des niveaux d'information différents selon l'environnement.

Par exemple, une erreur serveur non prévue peut être extrêmement utile à un développeur en environnement de développement :

```text
Unhandled exception
ValueError: invalid configuration

Stack trace:
...
```

Mais ces informations ne doivent généralement pas être exposées à un utilisateur en production.

L'objectif est donc de conserver une information suffisamment riche pour le diagnostic tout en contrôlant ce qui est présenté à l'extérieur de l'application.

---

## Visibilité selon l'environnement

APIKit-CEL permet d'associer un comportement de visibilité à l'environnement d'exécution.

### DEV

En développement, la priorité est le diagnostic.

Une erreur non prévue peut afficher :

* le code de l'erreur ;
* le message technique ;
* les détails de l'exception ;
* la stack trace ;
* les informations nécessaires au développement.

L'objectif est de rendre l'erreur immédiatement identifiable.

```text
Erreur interne

ValueError: invalid configuration

Trace:
  ...
```

Le développeur doit pouvoir comprendre rapidement **ce qui s'est produit et où**.

---

### QUALIF

En qualification, l'information doit rester utile au diagnostic tout en étant plus contrôlée.

Selon la configuration de l'application, on peut exposer :

* le code de l'erreur ;
* un message fonctionnel ;
* certaines informations de diagnostic ;
* une référence permettant de retrouver l'erreur dans les logs.

La stack trace complète ou les détails internes ne sont pas nécessairement exposés directement à l'utilisateur.

---

### PROD

En production, la priorité est la sécurité et l'expérience utilisateur.

Les informations techniques internes doivent rester côté serveur et ne doivent pas être exposées inutilement.

L'utilisateur peut recevoir par exemple :

```text
Une erreur inattendue s'est produite.

Référence : ERR-8F31A2
```

Pendant ce temps, les informations techniques peuvent rester disponibles dans les logs pour permettre l'investigation.

---

## Même erreur, visibilité différente

La visibilité ne modifie pas nécessairement l'erreur elle-même.

Une même erreur peut être représentée par un même contrat :

```text
INTERNAL_ERROR
```

mais produire des informations différentes selon l'environnement.

```text
                  INTERNAL_ERROR
                         │
              ┌──────────┴──────────┐
              │                     │
             DEV                   PROD
              │                     │
       détails techniques     message sécurisé
       stack trace             référence erreur
       diagnostic              aucune donnée interne
```

Le contrat reste stable tandis que la présentation et le niveau d'information peuvent varier.

---

## Visibilité ≠ gestion

Il est important de ne pas confondre ces deux responsabilités.

### Gestion de l'erreur

La gestion répond à la question :

> **Que doit faire l'application lorsqu'une erreur se produit ?**

Exemples :

* afficher une notification ;
* ouvrir une popup ;
* rediriger vers une page ;
* interrompre une opération ;
* effectuer un retry ;
* mettre à jour l'état de l'application ;
* journaliser l'erreur ;
* déclencher un fallback.

### Visibilité de l'erreur

La visibilité répond à la question :

> **Quelles informations doivent être présentées et sous quelle forme ?**

Exemples :

* message utilisateur ;
* détails techniques ;
* stack trace ;
* code d'erreur ;
* référence de diagnostic ;
* informations de validation.

Une erreur peut donc être **traitée sans être affichée**, ou être **affichée avec un niveau de détail différent selon l'environnement**.

---

## Visibilité et interface utilisateur

La visibilité ne doit pas dépendre directement d'un composant UI particulier.

Le backend ne doit pas avoir à savoir si le client utilise :

* un Toast ;
* une Alert ;
* une Popup ;
* une page d'erreur ;
* un composant React particulier.

Le serveur fournit les informations nécessaires dans le contrat d'erreur.

Le client applique ensuite une politique permettant de déterminer la présentation adaptée.

```text
Backend
   │
   │ Error Contract
   ▼
Client
   │
   │ Error Policy
   ▼
Visibility Policy
   │
   ├── Message
   ├── Details
   ├── Technical information
   └── Reference
          │
          ▼
         UI
```

Cette séparation permet au même backend d'être utilisé par différents clients sans introduire de dépendance à leur technologie d'interface.

---

## Informations techniques et informations utilisateur

APIKit-CEL distingue les informations destinées au diagnostic des informations destinées à l'utilisateur.

### Informations techniques

Elles peuvent inclure :

* exception d'origine ;
* stack trace ;
* paramètres techniques ;
* détails internes ;
* informations d'infrastructure ;
* contexte d'exécution.

Ces informations sont principalement destinées aux développeurs et aux systèmes de diagnostic.

### Informations utilisateur

Elles peuvent inclure :

* message compréhensible ;
* action à effectuer ;
* informations de validation ;
* référence d'erreur ;
* message fonctionnel.

Ces informations doivent être adaptées au contexte utilisateur.

---

## Principe de sécurité

Une règle importante d'APIKit-CEL est :

> **Les informations nécessaires au diagnostic ne doivent pas être automatiquement considérées comme des informations destinées à l'utilisateur.**

Une exception interne peut contenir des informations sensibles :

* chemins de fichiers ;
* noms de classes ;
* requêtes internes ;
* informations d'infrastructure ;
* détails de configuration ;
* données techniques ;
* informations permettant de comprendre l'architecture interne.

Ces informations peuvent être nécessaires au diagnostic mais doivent rester confinées au contexte technique approprié.

---

## Configuration de la visibilité

La visibilité doit être configurable plutôt que codée directement dans chaque endpoint ou composant.

Conceptuellement :

```text
Environment
     │
     ▼
Visibility Policy
     │
     ├── Error visibility
     ├── Technical details
     ├── Stack trace
     └── User message
```

Cela permet de centraliser les décisions de visibilité et d'éviter une logique du type :

```text
if development:
    ...
elif qualification:
    ...
else:
    ...
```

répliquée dans différentes parties de l'application.

---

## Visibilité et comportement UI

La visibilité peut également déterminer le comportement de présentation côté client.

Par exemple :

| Environnement | Informations                    | Présentation    |
| ------------- | ------------------------------- | --------------- |
| DEV           | Détails + stack trace           | Popup technique |
| QUALIF        | Message + diagnostic contrôlé   | Alert / Popup   |
| PROD          | Message utilisateur + référence | Toast / Alert   |

Cette configuration permet de conserver une stratégie cohérente tout au long de l'application.

---

## Pas d'erreur silencieuse

Le contrôle de la visibilité ne signifie pas que les erreurs doivent toujours être affichées.

Certaines erreurs peuvent être volontairement silencieuses pour l'utilisateur tout en étant :

* traitées par l'application ;
* journalisées ;
* observées ;
* transmises au système de monitoring ;
* associées à une référence de diagnostic.

L'objectif d'APIKit-CEL est donc plutôt :

> **Toute erreur applicative importante doit avoir un comportement explicitement défini et une stratégie de visibilité déterminée.**

Une erreur silencieuse par conception est différente d'une erreur ignorée par l'application.

---

## Architecture

La visibilité s'inscrit dans le flux global d'APIKit-CEL :

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
Transport
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
    ├── Technical information
    ├── User information
    └── Diagnostic reference
            │
            ▼
           UI
```

Le serveur reste responsable de la production d'un contrat cohérent.

Le client reste responsable de son interprétation et de sa présentation.

La politique de visibilité détermine ensuite quelles informations peuvent être exposées dans le contexte courant.

---

## Principes

APIKit-CEL repose sur plusieurs principes concernant la visibilité des erreurs :

1. **La visibilité dépend du contexte.**
2. **Le diagnostic technique et l'information utilisateur sont distincts.**
3. **Les informations internes ne doivent pas être exposées inutilement.**
4. **Le contrat d'erreur reste indépendant de l'interface utilisateur.**
5. **La politique de visibilité doit être centralisée et configurable.**
6. **Une erreur peut être traitée sans être affichée directement.**
7. **Une erreur ne doit pas devenir silencieuse par absence de stratégie.**
8. **Le niveau de détail peut évoluer sans modifier le contrat fondamental.**

---

## Objectif

L'objectif d'**Error Visibility** est de rendre l'expérience d'erreur cohérente sur l'ensemble de l'application :

```text
DEV
→ Comprendre rapidement l'erreur

QUALIF
→ Diagnostiquer tout en contrôlant l'exposition

PROD
→ Informer l'utilisateur sans exposer l'implémentation interne
```

APIKit-CEL transforme ainsi la visibilité des erreurs en une **préoccupation architecturale explicite**, plutôt qu'en une série de décisions dispersées dans les endpoints, les services et les composants UI.

> **Le système doit savoir qu'une erreur s'est produite, savoir comment la traiter et savoir quelles informations peuvent être exposées dans le contexte courant.**
