# Simulateur Holding — CLAUDE.md

## Objectif
Simulateur standalone HTML/JS déployé sur Render (holding-04ri.onrender.com).
Repo GitHub : Benjaminhofman/holding — branche main — fichier : index.html

## Question centrale
"Combien la société doit-elle générer comme bénéfice avant IS pour que le dirigeant dispose de X € nets à investir ?"

## Architecture
- Fichier unique : index.html (HTML + CSS + JS inline)
- Aucune dépendance externe sauf Google Fonts
- Déploiement : Static Site sur Render (auto-deploy sur push GitHub)
- Pas de backend, tous les calculs en JS client

---

## Modèle financier

### Paramètres
- Cible : montant net à investir (saisi par l'utilisateur)
- Forme juridique : SAS/SASU ou SARL/EURL
- TMI : tranche marginale IR (0/11/30/41/45%)
- IS taux réduit : 15% (jusqu'à 42 500 €)
- IS taux normal : 25%
- PFU : PS 18,6% + IR flat tax 12,8% = 31,4% (2026)
- Charges patronales SAS : 45% du brut
- Charges salariales SAS : 22% du brut
- Cotisations TNS SARL rémunération : 45%
- Cotisations TNS SARL dividendes : 45% (fraction > seuil)
- Seuil TNS = 10% × (capital social + CCA)
- Quote-part mère-fille : 5%
- Frais holding/an, frais SCI/an, frais création, amortissement

### Fonction calcIS(b)
```
si b ≤ 42 500 → b × is_r
si b > 42 500 → 42 500 × is_r + (b - 42 500) × is_n
```
Résolution bénéfice nécessaire par point fixe (25 itérations) :
`B = dividende_brut + calcIS(B)` → convergence garantie car IS(B)/B < 1

---

## Scénario A — Salaire

### SAS (assimilé-salarié)
```
Brut = cible / (1 - TMI) / (1 - cs_sal)
Coût employeur = brut × (1 + cs_pat)
IS = 0 (tout versé en salaire → résultat fiscal = 0)
Bénéfice nécessaire = cout_emp
```

### SARL (gérant TNS)
```
Rém. nette = cible / (1 - TMI)
Coût total = rém_nette × (1 + cs_tns_r)
IS = 0 (tout versé en rémunération → résultat = 0)
Bénéfice nécessaire = cout_emp
```

Note : IS = 0 car on suppose que le dirigeant verse TOUT en salaire pour investir.
L'économie IS implicite = calcIS(cout_emp) mais n'est PAS retranchée du comparateur.

---

## Scénario B — Dividendes

### SAS
```
Dividende brut = cible / (1 - PFU)
Bénéfice nécessaire B : point fixe B = div_brut + calcIS(B)
Pas d'économie IS (dividende non déductible)
```

### SARL
```
Seuil TNS = 10% × (capital + CCA)
Fraction PFU (≤ seuil) : net = dp × (1 - PFU)
Fraction TNS (> seuil) : net = dtn × (1 - cs_tns_d) × (1 - ir_pfu)
  → PFU ET cotisations TNS s'appliquent (pas exclusif)
  → cotisations TNS déductibles IS

Si dtn = 0 (tout en PFU) : même calcul que SAS
Sinon :
  dt = dp + dtn (itération pour trouver dt)
  cotis = dtn × cs_tns_d
  base_IS = dt - cotis
  IS = calcIS(base_IS)
  beneB = dt + IS
  Économie IS = calcIS(dt) - calcIS(dt - cotis)
```

---

## Scénario C — Holding + SCI

```
Friction mère-fille = qp × is_n = 5% × 25% = 1,25%
Besoin avant mère-fille = (cible + frais_fixes) / (1 - friction_mf)
Bénéfice B : point fixe B = div_holding_need + calcIS(B)

Waterfall :
  Bénéfice nécessaire B
  − IS exploitation = calcIS(B)
  = Dividende holding
  − IS mère-fille = div × 5% × 25%
  = Disponible avant frais
  − Frais holding/an
  − Frais SCI/an
  − Amortissement création
  = Net disponible ✅
```

---

## Comparateur
Tous les scénarios affichent le **bénéfice avant IS nécessaire** :
- Salaire SAS : cout_emp (IS = 0)
- Salaire SARL : cout_emp (IS = 0)
- Dividendes SAS : beneB = B2
- Dividendes SARL : beneB = dt + calcIS(dt - cotis)
- Holding : B3

---

## Interface

### Cards
- 3 cards : Salaire / Dividendes directs / Holding + SCI
- Montant affiché = bénéfice avant IS nécessaire
- Bordure verte = meilleur (min), rouge = pire (max)
- Tooltip au hover (à droite pour s1/s2, en bas pour s3)

### Tooltips
- s1 SAS : calcul à l'envers TMI → salarial → patronal → IS = 0
- s1 SARL : calcul à l'envers TMI → cotis TNS → IS = 0
- s2 SAS : PFU → dividende brut → IS exploitation → bénéfice nécessaire
- s2 SARL : seuil TNS → fraction PFU/TNS → cotis → IS → bénéfice nécessaire
- s3 : waterfall holding complet

### Message marketing
```
Pour disposer de X € par an à investir dans l'immobilier :
→ votre société doit générer [s1] € en rémunération,
→ [s2] € en dividendes,
→ seulement [s3] € via une holding.
Soit une économie de [gain_div] € / an vs dividendes
et [gain_sal] € / an vs salaire.
```

### Tableau effort économique cumulé
- Slider horizon 1-20 ans
- Lignes : 1/2/3/5/10/15/20 ans + horizon choisi
- Colonnes : Salaire / Dividendes / Holding / Économie avec holding
- Couleur verte = meilleur, rouge = pire

### Waterfall Holding
Décomposition pas à pas : bénéfice → IS → dividende → IS mère-fille → frais → net

### Avantages patrimoniaux
Liste non chiffrée : nue-propriété, capitalisation, effet boule de neige,
centralisation, protection, titres de participation, transmission, réinvestissement, diversification

---

## Décisions techniques clés

### Pourquoi itération et non équation directe ?
L'équation `x = div + calcIS(x)` est résolvable analytiquement (x > 42 500 → 0,75x = div + 6 375 - 10 625) mais l'itération est plus robuste pour tous les cas (x ≤ 42 500, SARL avec cotis, etc.). 25 tours suffisent toujours.

### Pourquoi IS = 0 pour le salaire ?
On suppose que le dirigeant verse TOUT le disponible en salaire pour investir → résultat fiscal = 0 → IS = 0. C'est cohérent avec la question centrale : "combien générer pour obtenir X € nets ?"

### Pourquoi CCA dans le seuil TNS ?
Seuil TNS = 10% × (capital social + primes d'émission + comptes courants d'associés). Sans CCA, un capital de 10 000 € donne un seuil de 1 000 € → presque tout en TNS → coût explosif.

### PFU sur dividendes TNS SARL
Les cotisations TNS (45%) ET l'IR (12,8%) s'appliquent sur la fraction TNS :
`net = dtn × (1 - 45%) × (1 - 12,8%)`
Ce n'est pas exclusif — les deux prélèvements se cumulent.

---

## Workflow déploiement
1. Modifier index.html localement
2. `git add index.html && git commit -m "..." && git push`
3. Render redéploie automatiquement (Static Site, ~1-2 min)
4. Hard refresh Ctrl+Shift+R pour vider le cache navigateur

## Bugs résolus
- `note-s3` id manquant → null pointer dans setCard
- `window.addEventListener('load')` trop tardif → remplacé par `DOMContentLoaded`
- Barres utilisaient `s1` au lieu de `cout_emp` → incohérence visuelle
- Double PFU sur fraction TNS SARL → corrigé : `(1-cs_tns_d) × (1-ir_pfu)`
- `beneB` SARL utilisait `B2b` (itération fausse) → remplacé par `dt + calcIS(dt-cotis)`
- Économie IS fictive retranchée du salaire → supprimée, comparateur = cout_emp
- Tooltip `white-space:pre` cassait le HTML → remplacé par `normal` + `<br>`
