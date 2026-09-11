# Guide d'intégration — Recommandations de fiscalités de rénovation

## Introduction

L'API Idéalement permet de **recommander les dispositifs fiscaux de rénovation** adaptés au profil d'un investisseur (Malraux, Déficit foncier, Monument Historique), puis de **calculer l'effort d'épargne** des lots correspondants.

Le flux d'intégration se fait en **trois appels** :

1. **Authentification** — obtenir un token JWT
2. **Recommandation** — obtenir la (ou les) fiscalité(s) adaptée(s) et une fourchette de prix
3. **Effort d'épargne** — lister les lots de votre catalogue dans cette fiscalité, filtrés par la fourchette de prix

L'endpoint de recommandation **ne calcule aucun effort d'épargne**. Il applique uniquement l'arbre de décision (TMI + résultat foncier). C'est ensuite à vous d'appeler `/saving_efforts/by_tax_program` pour chaque fiscalité renvoyée.

---

## Prérequis

- Un couple `client_id` / `client_secret` fourni par Idéalement
- Votre catalogue de lots importé dans la plateforme
- Un client HTTP capable d'envoyer un header `Authorization: Bearer …`

**URL de base :** `https://app.idealement.fr/api/v1`

**Documentation OpenAPI :** [SwaggerHub Idealement 1.0.7](https://app.swaggerhub.com/apis/idealement/Idealement/1.0.7)

---

## Disponibilité des fiscalités

| `tax_program` | Recommandation | Effort d'épargne (`/saving_efforts/by_tax_program`) |
| --- | --- | --- |
| `malraux` | Disponible | **Disponible** |
| `deficit_foncier` | Disponible | **Disponible** |
| `monument_historique` | Renvoyé par l'arbre | **Semaine prochaine** — ignorer cet identifiant pour l'instant |

Lorsque la recommandation contient à la fois `monument_historique` et `malraux`, n'appelez `/saving_efforts/by_tax_program` que pour `malraux` jusqu'à la mise en production de Monument Historique.

---

## Étape 1 — Authentification

Échangez vos identifiants contre un `access_token` (durée de vie : 24 h).

```http
POST https://app.idealement.fr/api/v1/auth/token
Content-Type: application/json

{
  "client_id": "<votre_source>",
  "client_secret": "<votre_cle>"
}
```

**Réponse :**

```json
{
  "access_token": "<jwt>",
  "token_type": "Bearer",
  "expires_in": 86400
}
```

Passez ensuite le token sur **tous** les appels suivants :

```http
Authorization: Bearer <access_token>
```

> La clé `api_key` en query string reste tolérée sur demande explicite, mais le Bearer JWT est le mode recommandé.

---

## Étape 2 — Recommander les fiscalités

```http
GET https://app.idealement.fr/api/v1/renovation_tax_program_recommendations
Authorization: Bearer <access_token>
```

### Paramètres


| Paramètre | Type | Obligatoire | Description |
| --- | --- | --- | --- |
| `number_of_parts` | Number | Oui | Nombre de parts fiscales |
| `marital_situation` | String | Oui | `celibataire` ou `marie` |
| `yearly_incomes` | Number | Non | Salaires et assimilés annuels nets (avant abattement) |
| `yearly_rental_incomes` | Number | Non | Revenus fonciers ou déficits fonciers annuels. Défaut : `0` |
| `dividends_exceptional` | Number | Non | Dividendes exceptionnels avant abattement de 40 %. Défaut : `0` |


### Réponse

```json
{
  "tmi": 0.41,
  "income_tax": 30130.52,
  "recommendations": [
    {
      "tax_program": "malraux",
      "buying_price_min": 300000,
      "buying_price_max": null
    }
  ]
}
```


| Champ | Description |
| --- | --- |
| `tmi` | Taux marginal d'imposition (fraction : `0.3` = 30 %, `0.41` = 41 %) |
| `income_tax` | Impôt sur le revenu de l'année 1 (€), arrondi à 2 décimales |
| `recommendations` | Liste éventuellement vide (TMI < 30 %). Chaque entrée porte une fourchette de prix inclusive. `null` = borne non contrainte |


---

## Étape 3 — Effort d'épargne par fiscalité

Pour **chaque** entrée de `recommendations` dont le `tax_program` est disponible (aujourd'hui `malraux` et `deficit_foncier`), appelez :

```http
GET https://app.idealement.fr/api/v1/saving_efforts/by_tax_program
Authorization: Bearer <access_token>
```

Reprenez les paramètres du profil investisseur, et **reportez la fourchette de prix** :

- si `buying_price_min` n'est pas `null` → passez `buying_price_min`
- si `buying_price_max` n'est pas `null` → passez `buying_price_max`
- si une borne vaut `null`, **omettez** le paramètre correspondant

**Exemple** (TMI 41 %, Malraux, prix ≥ 300 000 €) :

```
https://app.idealement.fr/api/v1/saving_efforts/by_tax_program?tax_program=malraux&buying_price_min=300000&yearly_incomes=120000&number_of_parts=1&marital_situation=celibataire&yearly_rental_incomes=5000
```

Les paramètres financiers habituels (`invest_contribution`, `invest_loan_duration`, `invest_loan_rate`, etc.) restent disponibles. Voir la [documentation OpenAPI 1.0.7](https://app.swaggerhub.com/apis/idealement/Idealement/1.0.7).

---

## Les 5 sorties possibles

L'arbre de décision croise la **TMI** et le **résultat foncier** (`yearly_rental_incomes`). Seuil foncier : **10 000 €** (inclusif : `≤ 10 000` vs `> 10 000`). Pivot de prix : **300 000 €**.

Tous les exemples ci-dessous sont des `GET` à préfixer du header `Authorization: Bearer <access_token>`. Profil : 1 part, célibataire.

### 1. TMI < 30 % — aucune recommandation

Aucune fiscalité de rénovation n'est proposée.

```
https://app.idealement.fr/api/v1/renovation_tax_program_recommendations?number_of_parts=1&marital_situation=celibataire&yearly_incomes=10000&yearly_rental_incomes=5000
```

```json
{
  "tmi": 0.11,
  "income_tax": 264.0,
  "recommendations": []
}
```

### 2. TMI = 30 % et foncier ≤ 10 000 € — Malraux + Monument Historique, prix ≤ 300 000 €

```
https://app.idealement.fr/api/v1/renovation_tax_program_recommendations?number_of_parts=1&marital_situation=celibataire&yearly_incomes=50000&yearly_rental_incomes=5000
```

```json
{
  "tmi": 0.3,
  "income_tax": 8103.99,
  "recommendations": [
    { "tax_program": "monument_historique", "buying_price_min": null, "buying_price_max": 300000 },
    { "tax_program": "malraux", "buying_price_min": null, "buying_price_max": 300000 }
  ]
}
```

Enchaînement actuel : uniquement `tax_program=malraux&buying_price_max=300000`.

### 3. TMI = 30 % et foncier > 10 000 € — Déficit foncier, prix ≤ 300 000 €

```
https://app.idealement.fr/api/v1/renovation_tax_program_recommendations?number_of_parts=1&marital_situation=celibataire&yearly_incomes=50000&yearly_rental_incomes=15000
```

```json
{
  "tmi": 0.3,
  "income_tax": 11103.99,
  "recommendations": [
    { "tax_program": "deficit_foncier", "buying_price_min": null, "buying_price_max": 300000 }
  ]
}
```

Enchaînement : `tax_program=deficit_foncier&buying_price_max=300000`.

### 4. TMI ≥ 41 % et foncier ≤ 10 000 € — Malraux + Monument Historique, prix ≥ 300 000 €

```
https://app.idealement.fr/api/v1/renovation_tax_program_recommendations?number_of_parts=1&marital_situation=celibataire&yearly_incomes=120000&yearly_rental_incomes=5000
```

```json
{
  "tmi": 0.41,
  "income_tax": 30130.52,
  "recommendations": [
    { "tax_program": "monument_historique", "buying_price_min": 300000, "buying_price_max": null },
    { "tax_program": "malraux", "buying_price_min": 300000, "buying_price_max": null }
  ]
}
```

Enchaînement actuel : uniquement `tax_program=malraux&buying_price_min=300000`.

### 5. TMI ≥ 41 % et foncier > 10 000 € — Déficit foncier, prix ≥ 300 000 €

```
https://app.idealement.fr/api/v1/renovation_tax_program_recommendations?number_of_parts=1&marital_situation=celibataire&yearly_incomes=120000&yearly_rental_incomes=15000
```

```json
{
  "tmi": 0.41,
  "income_tax": 34230.52,
  "recommendations": [
    { "tax_program": "deficit_foncier", "buying_price_min": 300000, "buying_price_max": null }
  ]
}
```

Enchaînement : `tax_program=deficit_foncier&buying_price_min=300000`.

---

## Récapitulatif


| Élément | Valeur |
| --- | --- |
| **Auth** | `POST /api/v1/auth/token` |
| **Recommandation** | `GET /api/v1/renovation_tax_program_recommendations` |
| **Effort d'épargne** | `GET /api/v1/saving_efforts/by_tax_program` |
| **Fiscalités utilisables aujourd'hui** | `malraux`, `deficit_foncier` |
| **Fiscalité à ignorer pour l'instant** | `monument_historique` (semaine prochaine) |
| **Seuil foncier** | 10 000 € |
| **Pivot de prix** | 300 000 € |
| **OpenAPI** | [version 1.0.7](https://app.swaggerhub.com/apis/idealement/Idealement/1.0.7) |


---

## Contact

Pour toute question technique sur l'intégration, contactez notre équipe technique.
