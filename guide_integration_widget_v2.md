# Guide d'intégration du Widget Simulateur — V2

## Introduction

Le **Widget Simulateur Idéalement** (V2) permet d'intégrer, directement sur votre site ou votre extranet, un **simulateur d'investissement immobilier complet** portant sur un lot précis de votre catalogue : financement, effort d'épargne, détail de l'acquisition et comparaison de fiscalités.

Il s'intègre via une balise `<iframe>` et s'adapte en partie à votre charte graphique.

Ce guide décrit **deux modes d'intégration** :

| Mode | Persistance | Authentification | Mise en place | Cas d'usage |
|---|---|---|---|---|
| **1. Intégration simple** | Aucune (sans état) | Aucune | Copier/coller d'une iframe | Afficher le simulateur sur une page lot |
| **2. Intégration avec persistance** | Simulations enregistrées | Token signé | Redirection d'un sous-domaine (CNAME) + activation | Espace client, sauvegarde et rechargement de simulations |

> Les deux modes partagent le même simulateur et les mêmes paramètres d'URL. Le mode 2 ajoute la capacité d'**enregistrer et recharger** des simulations pour un utilisateur.

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

## Mode 2 — Intégration avec persistance (redirection sous-domaine)

Ce mode ajoute la possibilité, pour un utilisateur, d'**enregistrer et recharger ses simulations**. Il nécessite deux éléments côté partenaire :

1. la **redirection d'un sous-domaine** vers notre plateforme (pour une session fiable) ;
2. l'**émission d'un token d'authentification** côté serveur.

Il est **activé par partenaire** par nos équipes.

### Pourquoi une redirection de sous-domaine ?

Dans une iframe, la conservation d'une session (donc la persistance des simulations d'un utilisateur) dépend du **contexte de cookie** du navigateur. Si le widget est servi depuis notre domaine (`app.idealement.fr`) et embarqué sur votre site (`www.mon-partenaire.fr`), le cookie est considéré comme **tiers** et bloqué par les navigateurs (Safari, Chrome).

La solution : servir le widget depuis un **sous-domaine de votre propre domaine**, redirigé (CNAME) vers notre infrastructure.

- Exemple : `simulateur.mon-partenaire.fr` → **CNAME** vers notre plateforme (Heroku).
- `simulateur.mon-partenaire.fr` et `www.mon-partenaire.fr` partageant le même domaine, l'iframe est **« same-site »** : le cookie de session devient **first-party** et n'est plus bloqué.

### Étapes de mise en place

1. **DNS (chez vous)** : créer un enregistrement **CNAME** `simulateur.mon-partenaire.fr` pointant vers la cible que nous vous communiquons.
2. **Certificat TLS (chez nous)** : une fois le CNAME en place, nous ajoutons le domaine et provisionnons automatiquement un **certificat HTTPS** (Let's Encrypt / ACM).
3. **Association & activation (chez nous)** : nous associons le sous-domaine à votre source et **activons la persistance**.

### Embarquer le widget avec authentification

Une fois le mode activé, l'iframe est servie depuis votre sous-domaine et reçoit un **token d'authentification** :

```html
<iframe
  src="https://simulateur.mon-partenaire.fr/widget/simulateur/mon_partenaire?source_id=63417_2503&token={TOKEN}"
  height="1000px"
  width="100%"
  style="background: white; border: none;">
</iframe>
```

### Comment obtenir le token (échange serveur-à-serveur)

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

Votre backend insère ensuite le `access_token` dans l'URL de l'iframe (paramètre `token`). À l'affichage, l'utilisateur est **automatiquement reconnu** (créé au premier passage, retrouvé aux suivants) et peut **enregistrer / recharger ses simulations**.

> `source_user_id` est **votre** identifiant d'utilisateur. Nous provisionnons de notre côté un utilisateur technique **rattaché à votre source**, sans email ni mot de passe, uniquement destiné au widget.

### Ce que ça débloque

- **Enregistrer une simulation** : une « photo » des critères d'investissement du moment associée au lot, rechargeable et modifiable.
- **Ouvrir une simulation** enregistrée : recharge les critères et le lot ; seules les données du lot (dont le prix) sont réactualisées.

---

## Sécurité — enjeux et engagements

Cette section synthétise nos engagements de sécurité, à destination de la DSI.

### 1. Isolation de l'iframe

Le widget s'exécute dans une **iframe cross-origin** : votre page et le widget sont **cloisonnés** par le navigateur (aucun accès réciproque au DOM ni aux données de l'autre contexte). Le widget ne lit ni ne modifie le contenu de votre page.

### 2. Cloisonnement strict du sous-domaine (mode 2)

Lorsque vous redirigez un sous-domaine vers notre plateforme, **seul le widget y répond**. Toutes les autres routes de notre application (back-office, administration, espaces internes) sont **inaccessibles** depuis ce sous-domaine (réponse 404). 

**Engagement** : le sous-domaine que vous exposez ne donne accès **qu'au simulateur**, jamais au reste de la plateforme. La surface d'exposition est réduite au strict nécessaire.

### 3. Authentification par token signé (mode 2)

- Le token est un **jeton signé cryptographiquement** (HS256). Toute tentative de **modification** du contenu (par ex. changer l'identifiant utilisateur) **invalide la signature** et le token est **rejeté** : un token ne peut pas être forgé ni altéré sans notre clé secrète.
- La **clé de signature reste exclusivement sur nos serveurs** : elle ne vous est jamais communiquée. L'émission des tokens se fait **serveur-à-serveur**, protégée par votre clé d'API (`client_secret`), qui **ne transite jamais par le navigateur**.
- Le token est **scopé à votre source** : un token émis pour votre source ne donne accès **qu'à vos données**. Il n'ouvre aucun accès aux données d'un autre partenaire.
- Le token est **à durée de vie courte** (quelques minutes), ce qui limite fortement l'impact d'une éventuelle interception.

### 4. Cloisonnement des données par utilisateur / source

- Les utilisateurs du widget sont **provisionnés par source**, **sans email ni mot de passe**, et **non connectables** aux espaces classiques de la plateforme : ce sont des comptes techniques strictement limités au widget.
- Les simulations enregistrées sont **rattachées à votre source** et à l'identifiant utilisateur que vous fournissez.

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
| **Authentification** | Aucune | Token signé (émis serveur-à-serveur) |
| **Persistance** | Aucune | Enregistrement / rechargement de simulations |
| **Prérequis partenaire** | Aucun (copier/coller iframe) | CNAME sous-domaine + émission du token |
| **Cookies / données perso** | Aucun | Cookie de session first-party ; identifiant externe uniquement |

---

## Contact

Pour la mise en place (association de source, activation de la persistance, redirection de sous-domaine, clé d'API) ou toute question technique, contactez notre équipe technique.
