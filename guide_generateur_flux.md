# Guide pour les Générateurs de Flux

> ### **Les noms de champs indiqués dans ce document sont donnés à titre d'exemple.**
>
> Nous construisons le système d'importation **en fonction du flux et de la documentation mis à disposition par le client**. Vous n'avez pas besoin d'adapter vos noms de champs à ceux présentés ici : c'est notre plateforme qui s'adapte à votre format. Ce document décrit les **informations nécessaires** (et non un format technique imposé) pour permettre une intégration réussie.

---

## Table des matières

- [Introduction](#introduction)
- [Formats acceptés](#formats-acceptés)
- [Structure des données : Programme et Lot](#structure-des-données--programme-et-lot)
- [Identifiants uniques](#identifiants-uniques)
- [Gestion du cycle de vie des biens](#gestion-du-cycle-de-vie-des-biens)
- [Champs communs à tous les lots](#champs-communs-à-tous-les-lots)
- [Champs par fiscalité](#champs-par-fiscalité)
- [Statuts des lots](#statuts-des-lots)
- [Types de biens acceptés](#types-de-biens-acceptés)
- [Format idéal recommandé (JSON)](#format-idéal-recommandé-json)

---

## Introduction

Ce document est destiné aux **générateurs de flux** souhaitant intégrer leurs données immobilières dans notre plateforme. Un flux est une source de données (fichier ou API) contenant la liste des programmes immobiliers et de leurs lots disponibles. Notre système importe ces données, crée et met à jour les biens automatiquement.

---

## Formats acceptés

Notre plateforme est capable d'ingérer des flux dans les formats suivants :

| Format | Description |
|--------|-------------|
| **JSON** | Format recommandé. Fichier ou endpoint API retournant du JSON. |
| **XML**  | Fichier XML ou endpoint retournant du XML. |
| **CSV**  | Fichier CSV avec séparateur virgule ou point-virgule. |

Le format **JSON via API REST** est le format privilégié car il offre la meilleure flexibilité et le mapping le plus fiable.

### Modes de récupération des données

Notre système est flexible sur la manière dont les données sont exposées. Plusieurs architectures sont possibles :

| Mode | Description |
|------|-------------|
| **Tout en un** | Un seul endpoint ou fichier contenant l'ensemble des programmes et de leurs lots. Simple à mettre en place, adapté aux catalogues de taille modérée. |
| **Programmes puis lots** | Un premier endpoint liste tous les programmes, puis un second endpoint retourne les lots d'un programme donné (ex : `/programs` puis `/programs/{id}/lots`). Recommandé pour les catalogues volumineux. |
| **Paginé** | Les résultats sont découpés en pages (ex : `/lots?page=1&per_page=100`). Adapté aux très gros volumes. |
| **Autre** | Toute autre organisation peut être envisagée. Nous nous adaptons à votre architecture existante. |

> **Limite de taille** : Nous préférons éviter de traiter des réponses dont la taille dépasse **4 Mo**. Si votre catalogue est volumineux, nous recommandons de privilégier une approche **paginée** ou **programme par programme** afin de garantir la fiabilité des imports.

---

## Structure des données : Programme et Lot

Notre modèle de données repose sur une relation **un-à-plusieurs** entre **Programmes** et **Lots** :

```
Programme (Résidence / Opération)
├── Lot 1 (Appartement T2, RDC)
├── Lot 2 (Appartement T3, 2ème étage)
├── Lot 3 (Maison T4)
└── ...
```

### Programme (= Résidence / Opération)

Un programme représente une **opération immobilière** dans son ensemble : une résidence, un immeuble, un ensemble de maisons. Il porte les informations communes à tous les lots qui le composent :

- Adresse, code postal, ville
- Coordonnées GPS
- Promoteur / constructeur
- Date de livraison
- Description commerciale
- Images du programme

### Lot (= Bien individuel)

Un lot est une **unité de vente** au sein d'un programme : un appartement, une maison. Il porte les informations spécifiques :

- Typologie (T1, T2, T3...)
- Surfaces (habitable, balcon, terrasse, jardin)
- Étage
- Prix
- Statut de commercialisation (libre, optionné, vendu)
- Données fiscales

**Un lot appartient toujours à un programme.** Chaque ligne du flux doit contenir à la fois les informations du lot ET l'identifiant du programme auquel il appartient.

---

## Identifiants uniques

Deux identifiants uniques sont **obligatoires** pour chaque entrée du flux :

### 1. Identifiant unique du lot

Chaque lot doit posséder un identifiant unique et **stable dans le temps**. Cet identifiant permet de :

- Retrouver le lot lors des mises à jour successives du flux
- Détecter les lots qui ont disparu du flux (considérés comme vendus)
- Éviter les doublons


### 2. Identifiant unique du programme

Chaque programme doit posséder un identifiant unique et **stable dans le temps**. Tous les lots d'un même programme doivent partager le même identifiant de programme.

> Exemple : `PRG-001`

**Important** : Ces identifiants ne doivent **jamais changer** d'un import à l'autre pour un même lot ou programme. Un changement d'identifiant entraînerait la création d'un doublon.

### Identifiant final sur notre plateforme

Une fois le catalogue intégré, l'identifiant unique de chaque lot sur notre plateforme est la **concaténation** du `program_id` et du `listing_id`, séparés par un underscore :

```
{program_id}_{listing_id}
```

> **Exemple** : Pour un lot `LOT-001` appartenant au programme `PRG-001`, l'identifiant sur notre plateforme sera `PRG-001_LOT-001`. C'est cet identifiant qui est utilisé pour accéder au lot via le **widget** ou via l'**API**.

---

## Gestion du cycle de vie des biens

### Biens disparus du flux = Biens vendus

Notre système compare le flux actuel avec les données précédemment importées. **Tout lot qui n'apparaît plus dans le flux est automatiquement considéré comme vendu.**

Concrètement :

1. Import N : les lots A, B, C sont présents dans le flux → A, B, C sont actifs
2. Import N+1 : seuls les lots A et C sont présents → **B est marqué comme vendu**

C'est pourquoi il est essentiel que **chaque flux contienne l'intégralité des lots disponibles** (non vendus) à un instant T. Un flux partiel entraînerait la mise en vente erronée de lots encore disponibles.

### Statuts explicites

Alternativement, vous pouvez renseigner un champ `status` pour chaque lot :

| Valeur     | Description |
|------------|-------------|
| `free`     | Lot disponible à la vente |
| `option`   | Lot en cours d'option |
| `sold`     | Lot vendu |

Si un lot est envoyé avec le statut `sold`, il sera désactivé de notre plateforme.

---

## Champs communs à tous les lots

### Champs obligatoires

| Champ | Type | Description |
|-------|------|-------------|
| `listing_id` | String | Identifiant unique du lot (ex: `LOT042`) |
| `program_id` | String | Identifiant unique du programme |
| `address` | String | Adresse du programme |
| `postcode` | String | Code postal (5 chiffres) |
| `city_name` | String | Nom de la ville |
| `builder` | String | Nom du promoteur / constructeur |
| `construction_type` | String | Type de bien (voir [types acceptés](#types-de-biens-acceptés)) |
| `number_of_rooms` | Integer | Nombre de pièces |
| `sqm_living` | Float | Surface habitable en m² |
| `price_with_vat` | Float | Prix TTC (TVA 20%) en euros |
| `status` | String | Statut : `free`, `option` ou `sold` |
| `tax_programs` | String / Array | Fiscalité(s) applicable(s) (voir section dédiée) |
| `sqm_balcony` | Float | Surface balcon en m² |
| `sqm_terrace` | Float | Surface terrasse en m² |
| `sqm_garden` | Float | Surface jardin en m² |
| `monthly_rental_price_estimated` | Float | Loyer mensuel estimé en euros |


### Champs facultatifs

| Champ | Type | Description |
|-------|------|-------------|
| `reference` | String | Référence commerciale du lot |
| `latitude` | Float | Latitude GPS du programme |
| `longitude` | Float | Longitude GPS du programme |
| `delivery_date` | Date | Date de livraison prévisionnelle (format `YYYY-MM-DD`) |
| `description` | String | Description commerciale du programme |
| `product_label` | String | Nom commercial du programme |
| `image_url` | String | URL de l'image principale du programme |
| `floor` | Integer | Étage du lot |
| `exposition` | String | Orientation (ex: `N`, `S`, `SE`, `NO`, `Nord`, `Sud-Est`) |
| `number_of_parkings` | Integer | Nombre de parkings inclus |

---

## Champs par fiscalité

Le champ `tax_programs` détermine les fiscalités applicables au lot. Voici les valeurs acceptées et les champs spécifiques à renseigner pour chaque fiscalité.


### LMNP Neuf Géré (résidences services)

**Valeur `tax_programs`** : `lmnp_amortization`

Pour les résidences de services neuves (étudiantes, seniors, tourisme, affaires).

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `price_with_vat` | Float | Oui | Prix TTC du lot |
| `price_without_vat` | Float | Oui | Prix HT du lot |
| `land_price_without_vat` | Float | Oui | Prix du foncier HT |
| `furniture_price_without_vat` | Float | Oui | Prix du mobilier HT |
| `managed_monthly_rental_price_without_vat` | Float | Oui | Loyer mensuel garanti HT (bail commercial) |
| `product_type` | String | Oui | Type de résidence (étudiante, seniors, tourisme, affaires) |
| `bailleur` | String | Oui | Nom du gestionnaire / exploitant |

---

### LMNP Non Géré

**Valeur `tax_programs`** : `lmnp_non_gere`

Pour les biens meublés non gérés par un exploitant.

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `price_with_vat` | Float | Oui | Prix TTC du lot |
| `furniture_price_with_vat` | Float | Oui | Prix du mobilier TTC |
| `furbished_monthly_rental_price_with_vat` | Float | Oui | Loyer meublé mensuel TTC estimé |

---

### LMNP Ancien Géré

**Valeur `tax_programs`** : `lmnp_ancien`

Pour les biens anciens en résidence gérée (marché secondaire). Nécessite un loyer géré > 0.

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `price_with_vat` | Float | Oui | Prix TTC du lot |
| `managed_monthly_rental_price_without_vat` | Float | **Oui** | Loyer mensuel garanti HT > 0 |
| `furniture_price_with_vat` | Float | Recommandé | Prix du mobilier TTC |
| `notary_fee` | Float | Recommandé | Frais de notaire (ancien = plus élevés) |
| `agency_costs` | Float | Recommandé | Frais d'agence |
| `refurbishment_price_with_vat` | Float | Recommandé | Prix des travaux de rénovation TTC |

---

### LMNP Ancien Non Géré

**Valeur `tax_programs`** : `lmnp_ancien_non_gere`

Pour les biens anciens meublés non gérés (sans loyer géré).

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `price_with_vat` | Float | Oui | Prix TTC du lot |
| `furniture_price_with_vat` | Float | Recommandé | Prix du mobilier TTC |
| `notary_fee` | Float | Recommandé | Frais de notaire |
| `agency_costs` | Float | Recommandé | Frais d'agence |
| `refurbishment_price_with_vat` | Float | Recommandé | Prix des travaux TTC |

---

### Nue Propriété

**Valeur `tax_programs`** : `nue_propriete`

Pour les biens en démembrement de propriété.

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `price_with_vat` | Float | Oui | Prix TTC pleine propriété |
| `nue_propriete_duration` | Integer | **Oui** | Durée du démembrement en années (doit être > 0) |
| `nue_propriete_rate` | Float | Recommandé | Taux de décote en nue propriété (ex: 0.60 pour 60%) |
| `nue_propriete_buying_price` | Float | Recommandé | Prix d'achat en nue propriété |
| `nue_propriete_usufructuary` | String | Recommandé | Nom de l'usufruitier |

> **Important** : Un lot en nue propriété sans durée de démembrement sera considéré comme inactif.

---

### LLI Nu (Logement Locatif Intermédiaire)

**Valeur `tax_programs`** : `lli`

Pour les biens éligibles au dispositif LLI en nu.

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `price_with_vat_10` | Float | **Oui** | Prix avec TVA réduite à 10% |

> **Important** : Un lot LLI sans `price_with_vat_10` sera rejeté à l'import.

---

### LLI Meublé

**Valeur `tax_programs`** : `lli`

Même dispositif que LLI Nu, mais avec un loyer meublé. La distinction se fait automatiquement si `furbished_monthly_rental_price_with_vat` > 0.

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `price_with_vat_10` | Float | **Oui** | Prix avec TVA réduite à 10% |
| `furniture_price_with_vat` | Float | Oui | Prix du mobilier TTC |
| `furbished_monthly_rental_price_with_vat` | Float | **Oui** | Loyer meublé mensuel TTC > 0 |

---

### Déficit Foncier

**Valeur `tax_programs`** : `deficit_foncier`

Pour les biens en déficit foncier (ancien avec travaux).

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `refurbishment_price_with_vat` | Float | Recommandé | Montant des travaux TTC |

> **Note** : Un lot en déficit foncier ne peut pas cumuler d'autres fiscalités (Pinel, LMNP, etc.).

---

### Hors Dispositif

**Valeur `tax_programs`** : `no_dispositive`

Pour les biens sans dispositif fiscal particulier (investissement classique ou résidence principale).

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `monthly_rental_price_estimated` | Float | Recommandé | Loyer mensuel estimé |

---

## Statuts des lots

| Valeur   | Description |
|----------|-------------|
| `free`   | Disponible à la vente |
| `option` | En cours d'option par un acquéreur |
| `sold`   | Vendu (sera désactivé de la plateforme) |

---

## Types de biens acceptés

Les types suivants sont reconnus et activés par notre plateforme :

| Valeur | Description |
|--------|-------------|
| `Appartement` | Appartement classique |
| `Maison` | Maison individuelle |
| `Villa` | Villa |
| `Duplex` | Duplex |
| `Maison de ville` | Maison de ville |
| `Chambre` | Chambre (résidence services) |
| `Chambre double` | Chambre double (résidence services) |
| `Immeuble` | Immeuble entier |
| `Collectif` | Logement collectif |

> Tout type non listé ci-dessus sera importé mais **désactivé** automatiquement.

---

## Format idéal recommandé (JSON)

Nous recommandons un flux JSON exposé via une **API REST** ou un **fichier statique hébergé**, contenant l'ensemble des lots disponibles. Voici le format idéal :

```json
{
  "programs": [
    {
      "program_id": "PRG-001",
      "label": "Résidence Les Jardins",
      "builder": "Promoteur Exemple",
      "address": "12 rue de la Paix",
      "postcode": "75002",
      "city_name": "Paris",
      "latitude": 48.8698,
      "longitude": 2.3311,
      "description": "Résidence haut de gamme au coeur de Paris...",
      "delivery_date": "2027-06-30",
      "image_url": "https://example.com/images/prg-001.jpg",
      "lots": [
        {
          "listing_id": "LOT-001",
          "reference": "A-101",
          "status": "free",
          "construction_type": "Appartement",
          "number_of_rooms": 3,
          "number_of_parkings": 1,
          "floor": 2,
          "exposition": "SE",
          "sqm_living": 65.4,
          "sqm_balcony": 8.5,
          "sqm_terrace": 0,
          "sqm_garden": 0,
          "price_with_vat": 450000,
          "tax_programs": ["pinel"],
          "monthly_rental_price_estimated": 1200,
          "yearly_estate_tax": 850,
          "yearly_coownership_fee": 1200,
          "extranet_url": "https://example.com/lots/A-101"
        },
        {
          "listing_id": "LOT-002",
          "reference": "B-201",
          "status": "free",
          "construction_type": "Appartement",
          "number_of_rooms": 2,
          "number_of_parkings": 1,
          "floor": 1,
          "exposition": "SO",
          "sqm_living": 42.3,
          "sqm_balcony": 5.0,
          "sqm_terrace": 0,
          "sqm_garden": 0,
          "price_with_vat": 280000,
          "tax_programs": ["lmnp"],
          "furniture_price_with_vat": 7000,
          "managed_monthly_rental_price_without_vat": 650,
          "monthly_rental_price_estimated": 800,
          "yearly_estate_tax": 600,
          "yearly_coownership_fee": 900
        },
        {
          "listing_id": "LOT-003",
          "reference": "C-001",
          "status": "option",
          "construction_type": "Maison",
          "number_of_rooms": 4,
          "number_of_parkings": 2,
          "floor": 0,
          "exposition": "S",
          "sqm_living": 95.0,
          "sqm_balcony": 0,
          "sqm_terrace": 20.0,
          "sqm_garden": 150.0,
          "price_with_vat": 380000,
          "tax_programs": ["nue_propriete"],
          "nue_propriete_duration": 15,
          "nue_propriete_rate": 0.62,
          "nue_propriete_buying_price": 235600,
          "nue_propriete_usufructuary": "CDC Habitat"
        }
      ]
    }
  ]
}
```

### Exemple avec fiscalité LLI

```json
{
  "listing_id": "PRG-002",
  "reference": "D-301",
  "status": "free",
  "construction_type": "Appartement",
  "number_of_rooms": 2,
  "number_of_parkings": 1,
  "floor": 3,
  "exposition": "N",
  "sqm_living": 45.0,
  "sqm_balcony": 6.0,
  "sqm_terrace": 0,
  "sqm_garden": 0,
  "price_with_vat": 260000,
  "price_with_vat_10": 238333,
  "tax_programs": ["lli"],
  "monthly_rental_price_estimated": 750
}
```

### Exemple avec fiscalité Déficit Foncier

```json
{
  "listing_id": "LOT-001",
  "reference": "E-101",
  "status": "free",
  "construction_type": "Appartement",
  "number_of_rooms": 3,
  "number_of_parkings": 0,
  "floor": 1,
  "exposition": "O",
  "sqm_living": 58.0,
  "sqm_balcony": 0,
  "sqm_terrace": 0,
  "sqm_garden": 0,
  "price_with_vat": 195000,
  "tax_programs": ["deficit_foncier"],
  "refurbishment_price_with_vat": 85000,
  "notary_fee": 15000,
  "agency_costs": 5000,
  "monthly_rental_price_estimated": 700
}
```

### Exemple LMNP Ancien Géré

```json
{
  "listing_id": "LOT-005",
  "reference": "F-102",
  "status": "free",
  "construction_type": "Chambre",
  "number_of_rooms": 1,
  "number_of_parkings": 0,
  "floor": 2,
  "exposition": "E",
  "sqm_living": 22.0,
  "sqm_balcony": 0,
  "sqm_terrace": 0,
  "sqm_garden": 0,
  "price_with_vat": 95000,
  "tax_programs": ["lmnp_ancien"],
  "managed_monthly_rental_price_without_vat": 420,
  "furniture_price_with_vat": 5500,
  "refurbishment_price_with_vat": 12000,
  "notary_fee": 7500,
  "agency_costs": 3000,
  "monthly_rental_price_estimated": 500
}
```

---

## Bonnes pratiques

1. **Exhaustivité du flux** : Le flux doit contenir **tous les lots disponibles** à chaque envoi. Un lot absent sera considéré comme vendu.

2. **Stabilité des identifiants** : Les `listing_id` et `program_id` doivent rester identiques tout au long de la vie du lot / programme.

3. **Fréquence de mise à jour** : Nous recommandons une mise à jour quotidienne du flux pour refléter les dernières disponibilités.

4. **Données numériques** : Les prix et surfaces doivent être des nombres (pas de symboles €, m², ni d'espaces). Le séparateur décimal accepté est le point (`.`) ou la virgule (`,`).

5. **Coordonnées GPS** : Si vous ne disposez pas des coordonnées GPS, nous les calculerons automatiquement à partir de l'adresse. Cependant, les fournir améliore la précision.

6. **Encodage** : UTF-8 est requis pour tous les formats.

7. **Fiscalités multiples** : Un lot peut avoir plusieurs fiscalités. Envoyez-les sous forme de tableau : `["pinel", "lli"]`. Notre système créera automatiquement les fiches d'investissement pour chaque fiscalité applicable.

---

## Récapitulatif des valeurs `tax_programs`

| Valeur | Fiscalité |
|--------|-----------|
| `pinel` | Pinel / Pinel+ |
| `lmnp` | LMNP Neuf Géré (résidences services) |
| `lmnp_non_gere` | LMNP Non Géré |
| `lmnp_ancien` | LMNP Ancien (géré si loyer garanti > 0, sinon non géré) |
| `nue_propriete` | Nue Propriété (démembrement) |
| `lli` | LLI - Logement Locatif Intermédiaire (nu ou meublé selon loyer meublé) |
| `deficit_foncier` | Déficit Foncier |
| `no_dispositive` | Hors Dispositif |

---

## Contact

Pour toute question technique sur l'intégration de votre flux, contactez notre équipe technique.
