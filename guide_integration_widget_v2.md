# Guide d'intégration du Widget Simulateur — V2

## Introduction

Le **Widget Simulateur Idéalement** (V2) permet d'intégrer, directement sur votre site ou votre extranet, un **simulateur d'investissement immobilier complet** portant sur un lot précis de votre catalogue : financement, effort d'épargne, détail de l'acquisition et comparaison de fiscalités.

Il s'intègre via une balise `<iframe>` et s'adapte en partie à votre charte graphique.

Ce guide décrit **deux modes d'intégration** :

| Mode | Persistance | Authentification | Mise en place | Cas d'usage |
|---|---|---|---|---|
| **1. Intégration simple** | Aucune (sans état) | Aucune | Copier/coller d'une iframe | Afficher le simulateur sur une page lot |
| **2. Intégration avec persistance** | Simulations enregistrées | Token signé dans l'URL de l'iframe | Sous-domaine délégué (CNAME) + émission du token côté serveur | Sauvegarde et rechargement de simulations |

> Les deux modes partagent le même simulateur et les mêmes paramètres d'URL. Le mode 2 est servi depuis un **sous-domaine de votre domaine** et ajoute un paramètre `token`, qui donne au visiteur la capacité d'**enregistrer et de rouvrir ses simulations** d'une visite à l'autre.

---

## Prérequis communs

Votre catalogue de biens doit avoir été importé dans notre plateforme via un flux de données (voir le *Guide pour les Générateurs de Flux*).

Vous aurez besoin, pour chaque lot :

| Paramètre | Description | Exemple |
|---|---|---|
| `program_id` | Identifiant du programme dans votre flux | `63417` |
| `listing_id` | Identifiant du lot dans votre flux | `2503` |

Le lot est désigné dans l'URL par le paramètre `source_id`, qui **concatène** le programme et le lot : `source_id={program_id}_{listing_id}` (ex. `63417_2503`).

---

## Mode 1 — Intégration simple, sans persistance (sans authentification)

C'est le mode le plus simple pour afficher le simulateur sur une page lot. **Aucune authentification, aucun cookie de suivi, aucune donnée personnelle** : chaque chargement est indépendant.

### Étape 1 — Copier le code iframe

```html
<iframe
  src="https://app.idealement.fr/widget/simulateur/{source}?source_id={program_id}_{listing_id}"
  height="1000px"
  width="100%"
  style="background-color: transparent; border: none;">
</iframe>
```

### Étape 2 — Renseigner les valeurs du lot

**Exemple concret :**

```html
<iframe
  src="https://app.idealement.fr/widget/simulateur/mon_partenaire?source_id=63417_2503"
  height="1000px"
  width="100%"
  style="background: white; border: none;">
</iframe>
```

Sur votre extranet, `source_id` doit être injecté **dynamiquement** en fonction du lot affiché.

### Étape 3 (optionnelle) — Pré-configurer le simulateur via l'URL

Vous pouvez initialiser les valeurs du simulateur en ajoutant des paramètres à l'URL (query string).

**Paramètres financiers**

| Paramètre | Type | Description | Défaut |
|---|---|---|---|
| `invest_contribution` | Entier | Apport initial (€) | `0` |
| `invest_loan_duration` | Entier | Durée du prêt (années) | `20` |
| `invest_loan_rate` | Décimal | Taux du prêt (%) | `4.0` |
| `invest_loan_insurance_rate` | Décimal | Taux d'assurance emprunteur (%) | `0.36` |
| `invest_guarantee_fee_rate` | Décimal | Taux des frais de garantie (%) | `1.35` |
| `possession_renting_rate` | Décimal | Frais de gestion locative (%) | `8` |

**Fiscalité**

| Paramètre | Type | Description | Défaut |
|---|---|---|---|
| `tax_program` | Chaîne | Fiscalité pré-sélectionnée (`pinel`, `no_dispositive_invest`, `no_dispositive_residence`, `lmnp_amortization`, `lmnp_non_gere`, `nue_propriete`, `lli_nu`, `lli_meuble`, `jeanbrun`, `deficit_foncier`, `ptz`, …) | Voir ci-dessous |

> **Comportement de la fiscalité par défaut** : si `tax_program` n'est pas fourni — ou s'il n'est **pas applicable au lot** — le widget s'initialise sur la **première fiscalité disponible du lot**. S'il est fourni **et** applicable au lot, c'est cette fiscalité qui est retenue.

**Paramètres PTZ**

| Paramètre | Type | Description | Défaut |
|---|---|---|---|
| `number_of_resident` | Entier | Nombre de résidents du foyer | `2` |
| `reference_revenue_y2` | Entier | Revenu fiscal de référence N-2 (€) | `50000` |

**Paramètres foyer / fiscalités investisseur (Jeanbrun, Déficit foncier, Nue-propriété…)**

| Paramètre | Type | Description | Défaut |
|---|---|---|---|
| `yearly_incomes` | Décimal | Salaires nets annuels du foyer (€) | `80000` |
| `number_of_parts` | Décimal | Nombre de parts fiscales | `3` |
| `marital_situation` | Chaîne | `celibataire` ou `marie` | `marie` |
| `yearly_rental_incomes` | Décimal | Revenus / déficits fonciers annuels (€) | `0` |
| `statut_jeanbrun` | Chaîne | `intermediaire`, `social`, `tres_social` | `intermediaire` |
| `apply_vat_5_5` | Booléen | Afficher les prix en TVA 5,5 % (`1`/`0`) | `0` |

> **Rétrocompatibilité** : l'alias `statut_bailleur_prive` reste accepté pour `statut_jeanbrun`.

**Exemple d'URL avec paramètres**

```
https://app.idealement.fr/widget/simulateur/mon_partenaire?source_id=63417_2503&tax_program=pinel&invest_contribution=20000&invest_loan_duration=25&invest_loan_rate=3.8
```

### Ce que garantit ce mode

- Le widget est **embarquable depuis n'importe quel domaine** (`X-Frame-Options: ALLOWALL`).
- **Aucune authentification, aucun état conservé** entre deux chargements.
- **Aucune donnée personnelle collectée**, aucun cookie de suivi.

---

## Mode 2 — Intégration avec persistance (sous-domaine + token signé)

Ce mode permet à un utilisateur **identifié chez vous** d'**enregistrer ses simulations** et de les **rouvrir** lors d'une visite ultérieure.

Il repose sur **deux prérequis**, à mettre en place dans cet ordre :

1. la **délégation d'un sous-domaine** de votre domaine (ex. `simulateur.mon-partenaire.fr`) vers notre infrastructure, depuis lequel l'iframe sera servie ;
2. l'**émission d'un token signé** par votre backend, injecté dans l'URL de l'iframe.

Le mode est **activé par partenaire** par nos équipes, en même temps que l'association du sous-domaine à votre source.

### Étape 1 — Déléguer un sous-domaine (CNAME)

**Pourquoi.** La persistance repose sur un **cookie de session**. Si l'iframe est servie depuis notre domaine (`app.idealement.fr`) et embarquée sur votre site (`www.mon-partenaire.fr`), ce cookie est un cookie **tiers** : Safari le bloque (ITP), et les autres navigateurs le restreignent progressivement. Le visiteur repart alors de zéro à chaque chargement — il n'y a rien à recharger d'une visite à l'autre.

En servant l'iframe depuis un **sous-domaine de votre propre domaine**, l'iframe devient **« same-site »** vis-à-vis de votre page : le cookie de session est **first-party**, il n'est plus bloqué, et la session du visiteur survit à ses navigations.

**Mise en place**, à planifier avec nos équipes :

1. **DNS (chez vous)** : créer un enregistrement **CNAME** `simulateur.mon-partenaire.fr` pointant vers la cible que nous vous communiquons.
2. **Certificat TLS (chez nous)** : une fois le CNAME en place, nous ajoutons le domaine à notre infrastructure et provisionnons automatiquement un **certificat HTTPS** (Let's Encrypt).
3. **Association & activation (chez nous)** : nous associons le hostname à votre source et **activons la persistance**.

> Cette brique d'infrastructure est **en cours de livraison**. Contactez-nous pour planifier la mise en place et obtenir la cible CNAME à déclarer.

### Étape 2 — Obtenir un token (échange serveur-à-serveur)

Le token est **émis par nos serveurs**, à la demande de **votre backend** (jamais depuis le navigateur), au moyen des identifiants d'API qui vous sont fournis :

```
POST https://app.idealement.fr/api/v1/auth/token
Content-Type: application/json

{
  "client_id": "mon_partenaire",        // nom de votre source
  "client_secret": "•••••••••••••",      // votre clé d'API (secrète)
  "source_user_id": "USR-12345"          // identifiant de VOTRE utilisateur
}
```

Réponse :

```json
{ "access_token": "<token signé>", "token_type": "Bearer", "expires_in": 600 }
```

`expires_in` vaut `600` : le token widget est valable **10 minutes**. Il doit donc être **émis au moment où vous rendez la page** contenant l'iframe, et surtout pas mis en cache ni réutilisé d'une page à l'autre. Sa seule fonction est d'**ouvrir la session** ; une fois celle-ci ouverte, son expiration est sans effet sur la navigation en cours.

> ⚠️ **Deux tokens différents sortent de ce même endpoint — ils ne sont pas interchangeables.**
>
> | Corps de la requête | Token émis | Validité | Où l'utiliser |
> |---|---|---|---|
> | `client_id` + `client_secret` | Token **API** | 24 h (`expires_in: 86400`) | Header `Authorization: Bearer` des endpoints `/saving_efforts/*` et `/renovation_tax_program_recommendations` |
> | idem **+ `source_user_id`** | Token **widget** | 10 min (`expires_in: 600`) | Paramètre `?token=` de l'iframe du widget, **uniquement** |
>
> Un token API placé dans l'URL de l'iframe est **rejeté** (le widget reste anonyme) ; un token widget placé en `Authorization: Bearer` est **rejeté** par l'API (401).

`source_user_id` est **votre** identifiant d'utilisateur, celui de votre extranet ou de votre CRM. Nous provisionnons de notre côté un utilisateur technique **rattaché à votre source**, sans email réel ni mot de passe, uniquement destiné au widget. Le couple (votre source, `source_user_id`) constitue son identité : le même `source_user_id` retrouve toujours le même utilisateur.

> **L'émission du token se fait toujours sur `https://app.idealement.fr`**, jamais sur votre sous-domaine : c'est un appel machine-à-machine depuis votre backend, et cet endpoint n'est pas exposé sur le sous-domaine, qui ne répond que pour le widget lui-même.

### Étape 3 — Embarquer l'iframe avec le token

L'iframe est servie depuis **votre sous-domaine**, et reçoit le token en paramètre d'URL :

```html
<iframe
  src="https://simulateur.mon-partenaire.fr/widget/simulateur/mon_partenaire?source_id=63417_2503&token={TOKEN}"
  height="1000px"
  width="100%"
  style="background: white; border: none;">
</iframe>
```

Tous les paramètres d'URL du mode 1 (`tax_program`, `invest_contribution`, `yearly_incomes`, …) restent utilisables ; `token` s'y ajoute simplement.

### Étape 4 — Ce qui se passe à l'affichage

1. Le token est vérifié, puis l'utilisateur technique est **créé** (premier passage) ou **retrouvé** (passages suivants).
2. La **session du widget** est ouverte.
3. Le paramètre `token` est **retiré de l'URL** par une redirection interne : l'iframe affiche ensuite `…/widget/simulateur/mon_partenaire?source_id=63417_2503`, sans le token.

Tant que la session dure, les rechargements de l'iframe n'ont pas besoin de token. Il reste néanmoins recommandé d'en **émettre un neuf à chaque rendu de page** côté partenaire : c'est ce qui garantit que l'utilisateur est bien reconnu, y compris après expiration du cookie de session.

### Ce que ça débloque

Une fois la persistance activée, le visiteur authentifié dispose d'un **bandeau de gestion des simulations** dans l'iframe, avec deux actions explicites :

- **Enregistrer une simulation** : une « photo » des critères d'investissement du moment, associée au lot affiché et nommée par le visiteur.
- **Ouvrir une simulation** enregistrée : les critères et le lot sont rechargés ; seules les données du lot (dont le prix) sont réactualisées.

L'enregistrement est **toujours explicite** : rien n'est sauvegardé tant que le visiteur ne clique pas sur « Enregistrer ». Les simulations sont cloisonnées par visiteur — un `source_user_id` ne voit que les siennes.

Les paramètres d'URL (`tax_program`, `invest_contribution`, …) s'appliquent à **chaque** chargement de l'iframe, exactement comme en mode 1 : ce sont eux qui définissent le point de départ de la simulation, l'enregistrement restant à la main du visiteur.

### Dégradation en cas de token absent ou invalide

Le widget ne renvoie **jamais d'erreur** à cause du token. Si le token est **absent**, **expiré**, **malformé**, **falsifié** ou **émis pour une autre source**, il est simplement ignoré : le simulateur s'affiche en **visiteur anonyme**, avec le comportement du mode 1 — sans bandeau de simulations, sans erreur, sans page blanche. Votre page reste donc fonctionnelle même si votre backend n'a pas pu émettre de token.

---

## Sécurité — enjeux et engagements

Cette section synthétise nos engagements de sécurité, à destination notamment de votre DSI.

### 1. Isolation de l'iframe

Le widget s'exécute dans une **iframe cross-origin** : votre page et le widget sont **cloisonnés** par le navigateur (aucun accès réciproque au DOM ni aux données de l'autre contexte). Le widget ne lit ni ne modifie le contenu de votre page.

### 2. Cloisonnement strict du sous-domaine (mode 2)

Lorsque vous déléguez un sous-domaine vers notre plateforme, **seul le widget y répond**. Toutes les autres routes de notre application — espace SAAS, administration, catalogue, pages de connexion — sont **inaccessibles** depuis ce sous-domaine (réponse 404). L'endpoint d'émission des tokens lui-même n'y est pas exposé : il reste réservé aux appels serveur-à-serveur sur `app.idealement.fr`.

Le sous-domaine est en outre **lié à votre source** : une URL forgée qui y demanderait le widget d'un autre partenaire répond 404.

**Engagement** : le sous-domaine que vous exposez ne donne accès **qu'au simulateur**, jamais au reste de la plateforme. La surface d'exposition est réduite au strict nécessaire.

### 3. Authentification par token signé (mode 2)

- Le token est un **jeton signé cryptographiquement** (HS256). Toute tentative de **modification** du contenu (par ex. changer l'identifiant utilisateur) **invalide la signature** et le token est **rejeté** : un token ne peut pas être forgé ni altéré sans notre clé secrète.
- La **clé de signature reste exclusivement sur nos serveurs** : elle ne vous est jamais communiquée. L'émission des tokens se fait **serveur-à-serveur**, protégée par votre clé d'API (`client_secret`), qui **ne transite jamais par le navigateur**.
- Le token est **scopé à votre source**, et ce périmètre est **imposé par nos serveurs** : il est déduit des identifiants d'API utilisés pour l'émettre, jamais d'un champ de la requête. Il n'est donc pas possible d'émettre un token portant sur la source d'un autre partenaire, même en tentant de le préciser dans le corps de l'appel.
- Le token est **à durée de vie courte** — **10 minutes** — ce qui limite fortement l'impact d'une éventuelle interception.
- **Hygiène du token** : après authentification, le token est **retiré de l'URL** (aucune persistance dans l'historique du navigateur) et **n'est jamais journalisé** côté serveur, ni dans les logs applicatifs, ni dans les rapports d'erreur.
- **Cloisonnement des jetons** : le token du widget n'ouvre **que** le widget. Il n'est accepté ni par l'API `saving_efforts`, ni par les espaces de connexion de la plateforme. Réciproquement, le token de l'API `saving_efforts` n'ouvre aucune session widget.
- **Dégradation sûre** : un token absent, expiré, altéré ou émis pour une autre source ne produit pas d'erreur — le widget est simplement rendu en visiteur anonyme.

### 4. Cloisonnement des données par utilisateur / source

- Les utilisateurs du widget sont **provisionnés par source**, **sans email réel ni mot de passe**, et **non connectables** aux espaces classiques de la plateforme (`/users/sign_in`, espace SAAS) : ce sont des comptes techniques strictement limités au widget.
- Les simulations enregistrées sont **rattachées à votre source** et à l'identifiant utilisateur que vous fournissez. La session ouverte par un token de votre source n'a aucune valeur sur le widget d'un autre partenaire.
- **Aucune contamination avec les comptes de la plateforme** : si le visiteur est par ailleurs connecté à notre espace SAAS dans le même navigateur, cette session est **ignorée** à l'intérieur de l'iframe. Un widget sans token reste anonyme, et les données du compte SAAS ne sont ni affichées ni modifiées depuis votre page.

### 5. Transport et confidentialité

- **HTTPS/TLS** de bout en bout ; certificats provisionnés et renouvelés automatiquement.
- **Mode 1** : aucune donnée personnelle collectée, aucun cookie de suivi.
- **Mode 2** : nous ne stockons que l'**identifiant externe** que vous nous transmettez (`source_user_id`) et les **paramètres de simulation** ; aucune donnée personnelle sensible n'est requise pour faire fonctionner le widget.

---

## Récapitulatif

| Élément | Mode 1 (sans persistance) | Mode 2 (avec persistance) |
|---|---|---|
| **URL de base** | `https://app.idealement.fr/widget/simulateur/{source}` | `https://simulateur.{partenaire}.fr/widget/simulateur/{source}` |
| **Désignation du lot** | `?source_id={program_id}_{listing_id}` | idem + `&token={TOKEN}` |
| **Authentification** | Aucune | Token signé, émis serveur-à-serveur sur `app.idealement.fr`, valable 10 min |
| **Persistance** | Aucune | Enregistrement et réouverture de simulations |
| **Prérequis partenaire** | Aucun (copier/coller iframe) | Sous-domaine délégué en CNAME + émission du token depuis votre backend |
| **Cookies / données perso** | Aucun | Cookie de session first-party ; identifiant externe (`source_user_id`) uniquement |

---

## Contact

Pour la mise en place (délégation du sous-domaine, association de source, activation de la persistance, clé d'API) ou toute question technique, contactez notre équipe technique.
