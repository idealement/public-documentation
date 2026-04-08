# Guide d'intégration du Widget Effort d'Épargne

## Introduction

Le widget Idéalement permet d'afficher le **calcul de l'effort d'épargne** d'un lot immobilier directement sur votre extranet ou votre site web. Il est intégré via une balise `<iframe>` qui charge une page depuis `https://app.idealement.fr`.

Le widget affiche :

- Le **financement du bien** (apport, durée de prêt, taux)
- Le **calcul de l'effort d'épargne** personnalisable par l'utilisateur
- Le détail de l'investissement selon la fiscalité applicable
- La possibilité de **télécharger une étude personnalisée en PDF**

L'utilisateur peut modifier les paramètres financiers directement dans le widget (apport, durée, taux, fiscalité) et voir les résultats se recalculer en temps réel.

---

## Prérequis

Avant d'intégrer le widget, votre catalogue de biens doit avoir été importé dans notre plateforme via un flux de données (voir le [Guide pour les Générateurs de Flux](guide_generateur_flux.md)).

Vous aurez besoin de trois informations pour chaque lot :


| Paramètre    | Description                                        | Exemple          |
| ------------ | -------------------------------------------------- | ---------------- |
| `partner_id` | Identifiant de votre source, fourni par Idéalement | `mon_partenaire` |
| `program_id` | Identifiant du programme dans votre flux           | `PRG-001`        |
| `listing_id` | Identifiant du lot dans votre flux                 | `LOT-042`        |


> **Rappel** : Le `program_id` et le `listing_id` correspondent aux identifiants uniques que vous transmettez dans votre flux de données. Consultez la section [Identifiants uniques](guide_generateur_flux.md#identifiants-uniques) du guide des flux pour plus de détails.

---

## Intégration de base

### Étape 1 : Copier le code iframe

Insérez la balise `<iframe>` suivante dans le code HTML de votre page, à l'endroit où vous souhaitez afficher le widget :

```html
<iframe
  src="https://app.idealement.fr/widget/effort/{partner_id}/{program_id}/{listing_id}"
  height="1000px"
  width="100%"
  style="background: white; border: none;">
</iframe>
```

### Étape 2 : Remplacer les variables

Remplacez `{partner_id}`, `{program_id}` et `{listing_id}` par les valeurs réelles du lot à afficher.

**Exemple concret :**

```html
<iframe
  src="https://app.idealement.fr/widget/effort/mon_partenaire/PRG-001/LOT-042"
  height="1000px"
  width="100%"
  style="background: white; border: none;">
</iframe>
```

### Étape 3 : Intégration dynamique

Sur votre extranet, le `program_id` et le `listing_id` doivent être insérés **dynamiquement** en fonction du lot affiché sur la page. La plupart des extranets permettent d'injecter des variables dans le HTML du template de la page lot.

---

## Construction de l'URL

L'URL du widget suit le format suivant :

```
https://app.idealement.fr/widget/effort/{partner_id}/{program_id}/{listing_id}
```


| Segment      | Description                                               |
| ------------ | --------------------------------------------------------- |
| `partner_id` | Identifiant de votre source (fourni par Idéalement)       |
| `program_id` | Identifiant du programme tel que transmis dans votre flux |
| `listing_id` | Identifiant du lot tel que transmis dans votre flux       |


---

## Paramètres de personnalisation

Vous pouvez **pré-configurer** le widget en ajoutant des paramètres dans l'URL (query string). Cela permet d'initialiser les valeurs du formulaire de financement affichées à l'utilisateur.

### Paramètres financiers


| Paramètre                   | Type    | Description                       | Valeur par défaut |
| --------------------------- | ------- | --------------------------------- | ----------------- |
| `invest_contribution`       | Integer | Apport initial en euros           | `0`               |
| `invest_loan_duration`      | Integer | Durée du prêt en années           | `20`              |
| `invest_loan_rate`          | Float   | Taux du prêt (en %)               | `4.0`             |
| `invest_guarantee_fee_rate` | Float   | Taux des frais de garantie (en %) | `1.35`            |


### Paramètre de fiscalité


| Paramètre     | Type   | Description                                                                                                                                                                                                                                                                        | Valeur par défaut             |
| ------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| `tax_program` | String | Fiscalité à sélectionner par défaut. Valeurs possibles : `pinel`, `no_dispositive`, `no_dispositive_invest`, `no_dispositive_residence`, `lmnp_amortization`, `lmnp_non_gere`, `lmnp_ancien_gere`, `lmnp_ancien`, `nue_propriete`, `lli_nu`, `lli_meuble`, `jeanbrun`, `ptz` | Première fiscalité disponible |


> **Note** : Seules les fiscalités **applicables au lot** seront affichées dans le sélecteur. Si la fiscalité demandée en paramètre n'est pas applicable au lot, le widget utilisera la première fiscalité disponible.

### Paramètres PTZ (Prêt à Taux Zéro)


| Paramètre              | Type    | Description                             | Valeur par défaut |
| ---------------------- | ------- | --------------------------------------- | ----------------- |
| `number_of_resident`   | Integer | Nombre de résidents du foyer            | `2`               |
| `reference_revenue_y2` | Integer | Revenu fiscal de référence N-2 en euros | `50000`           |


### Paramètres Jeanbrun / Déficit Foncier / Nue Propriété


| Paramètre               | Type   | Description                                                                         | Valeur par défaut |
| ----------------------- | ------ | ----------------------------------------------------------------------------------- | ----------------- |
| `yearly_incomes`        | Float  | Salaires et assimilés annuels en euros                                              | 80000             |
| `number_of_parts`       | Float  | Nombre de parts fiscales                                                            | 3                 |
| `marital_situation`     | String | Situation familiale. Valeurs possibles : `celibataire`, `marie`, `union_libre`      | marie             |
| `yearly_rental_incomes` | Float  | Revenus / Déficits fonciers annuels en euros                                        | 0                 |
| `statut_jeanbrun` | String | Statut Jeanbrun. Valeurs possibles : `intermediaire`, `social`, `tres_social` | `intermediaire`   |


> **Note** : Ces paramètres sont utilisés pour le calcul de l'effort d'épargne en fiscalité **Jeanbrun**, **Déficit Foncier** et **Nue Propriété**. Si non renseignés, les valeurs par défaut du partenaire seront utilisées.

> **Rétrocompatibilité :** Les valeurs `bailleur_prive` (pour `tax_program`) et `statut_bailleur_prive` restent acceptées comme alias.

### Paramètres d'affichage


| Paramètre     | Type    | Description                                                | Valeur par défaut |
| ------------- | ------- | ---------------------------------------------------------- | ----------------- |
| `reduced_vat` | Boolean | Afficher les prix en TVA réduite (`1` ou `0`)              | `0`               |
| `for_renting` | Boolean | **Déprécié.** Utilisez `tax_program=no_dispositive_invest` à la place. Calculs Hors Dispositif avec mise en location (`1` ou `0`) | `1`               |

### Paramètres avancés (optionnels)

Ces paramètres sont acceptés par le widget et peuvent être passés en query string pour pré-configurer des cas plus avancés (selon le partenaire / la fiscalité et l'UI du widget).

#### Frais et assurance de prêt

| Paramètre                    | Type    | Description                               | Valeur par défaut |
| --------------------------- | ------- | ----------------------------------------- | ----------------- |
| `invest_loan_insurance_rate`| Float   | Taux d'assurance du prêt (en %)           | `0.36`            |
| `invest_filing_fees_amount` | Integer | Frais de dossier (en €)                   | `0`               |

#### Prix d'acquisition et frais de notaire

| Paramètre             | Type    | Description                                       | Valeur par défaut |
| -------------------- | ------- | ------------------------------------------------- | ----------------- |
| `custom_listing_price` | Integer | Prix du bien utilisé pour la simulation (en €)   | Prix du lot       |
| `notary_fees_offered`  | Boolean | Frais de notaire offerts (`1` ou `0`)            | `0`               |

#### Parkings (si le programme en propose)

Vous pouvez pré-sélectionner des parkings en passant une liste.

- Format recommandé : `selected_parkings[]=P1&selected_parkings[]=P2`
- Format alternatif (équivalent) : `profile[selected_parkings][]=P1&profile[selected_parkings][]=P2`

> **Note** : le widget recalculera le prix des parkings si `reduced_vat=1` et que le lot est éligible à une TVA réduite.

#### Prêts bonifiés (jusqu'à 4)

Certains widgets permettent d'ajouter des prêts complémentaires (prêts bonifiés). Chaque prêt \(i\) accepte :

| Paramètre                         | Type    | Description                 | Valeur par défaut |
| -------------------------------- | ------- | --------------------------- | ----------------- |
| `invest_bonified_loan_{i}_amount` | Integer | Montant du prêt (en €)      | `0`               |
| `invest_bonified_loan_{i}_duration` | Integer | Durée du prêt (en années) | `20`              |
| `invest_bonified_loan_{i}_rate`   | Float   | Taux du prêt (en %)         | `0`               |
| `invest_bonified_loan_{i}_offset` | Integer | Différé / décalage (mois)   | `0`               |


### Exemple d'URL avec paramètres

```
https://app.idealement.fr/widget/effort/mon_partenaire/PRG-001/LOT-042?tax_program=pinel&invest_loan_duration=20&invest_contribution=10000&invest_loan_rate=3.5
```

**Intégré dans l'iframe :**

```html
<iframe
  src="https://app.idealement.fr/widget/effort/mon_partenaire/PRG-001/LOT-042?tax_program=pinel&invest_loan_duration=20&invest_contribution=10000"
  height="1000px"
  width="100%"
  style="background: white; border: none;">
</iframe>
```

---

## Fonctionnalités du widget

### Calcul interactif

L'utilisateur peut modifier directement dans le widget :

- **L'apport** (montant de l'apport personnel)
- **La durée du prêt** (en années)
- **Le taux du prêt**
- **La fiscalité** (parmi celles applicables au lot)

À chaque modification, l'effort d'épargne et le détail de l'investissement se recalculent automatiquement.

### Téléchargement PDF (si inclu dans l'offre)

Le widget propose un bouton permettant à l'utilisateur de **télécharger une étude personnalisée en PDF**, tenant compte des paramètres financiers qu'il a saisis.

### Sélection de parkings (si inclu dans l'offre)

Si le programme propose des parkings optionnels, l'utilisateur pourra les sélectionner dans le widget. Le prix des parkings sera intégré au calcul de l'effort d'épargne.

---

## Personnalisation visuelle

Le widget est automatiquement adapté aux **couleurs et à l'identité visuelle** de chaque partenaire. Cette personnalisation est gérée par notre équipe lors de la mise en place de l'intégration.

Si vous souhaitez une personnalisation spécifique (couleurs, logo, typographie), contactez notre équipe technique.

---

## Recommandations techniques

### Dimensions

- **Largeur** : `100%` (le widget est responsive et s'adapte à la largeur de son conteneur)
- **Hauteur** : `1000px` recommandé (ajustable selon le contenu affiché)

### Compatibilité

- Le widget fonctionne sur tous les navigateurs modernes (Chrome, Firefox, Safari, Edge)
- Compatible mobile et tablette (responsive)
- L'en-tête `X-Frame-Options: ALLOWALL` est défini pour permettre l'intégration en iframe depuis n'importe quel domaine

### Sécurité

- Le widget ne nécessite aucune authentification
- Aucune donnée personnelle de l'utilisateur n'est collectée

---

## Récapitulatif


| Élément                   | Valeur                                                                                                                                                                                                                                                                                                |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **URL de base**           | `https://app.idealement.fr/widget/effort/{partner_id}/{program_id}/{listing_id}`                                                                                                                                                                                                                      |
| **Méthode d'intégration** | Balise `<iframe>`                                                                                                                                                                                                                                                                                     |
| **Hauteur recommandée**   | `1000px`                                                                                                                                                                                                                                                                                              |
| **Largeur recommandée**   | `100%`                                                                                                                                                                                                                                                                                                |
| **Paramètres optionnels** | `tax_program`, `invest_contribution`, `invest_loan_duration`, `invest_loan_rate`, `invest_guarantee_fee_rate`, `number_of_resident`, `reference_revenue_y2`, `yearly_incomes`, `number_of_parts`, `marital_situation`, `yearly_rental_incomes`, `statut_jeanbrun`, `reduced_vat`, `for_renting` |


---

## Contact

Pour toute question technique sur l'intégration du widget, contactez notre équipe technique.
