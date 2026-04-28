# 🧾 La Gestion de la TVA dans SAP S/4HANA — Version Expert

![SAP](https://img.shields.io/badge/SAP-S%2F4HANA-0FAAFF?style=flat-square&logo=sap&logoColor=white)
![Version](https://img.shields.io/badge/Version-Expert-blue?style=flat-square)
![Auteur](https://img.shields.io/badge/Auteur-Moez%20L'Agha-green?style=flat-square)
![MTC Computing](https://img.shields.io/badge/MTC%20Computing-Consultant%20SAP%20FI%2FCO-orange?style=flat-square)

> **Paramétrage · OB40 · Transport SM35 · Schéma de calcul · Debug /H · Flux MM/FI**
>
> Ce document est la version Expert du guide TVA SAP S/4HANA. Il répond à une question que les manuels esquivent systématiquement :
> **Pourquoi votre TVA est-elle à 0% en Production alors que votre ordre de transport est passé sans erreur dans la STMS ?**
> La réponse tient en un mot : **FTXP ne fonctionne pas comme le reste du Customizing.**

---

## 📋 Table des matières

1. [Fondamentaux de la TVA dans SAP](#i-fondamentaux-de-la-tva-dans-sap)
2. [Le Schéma de Calcul — La Mécanique Interne](#ii-le-schéma-de-calcul--la-mécanique-interne)
3. [Codes TVA — Transaction FTXP](#iii-codes-tva--transaction-ftxp)
4. [OB40 — La Clé de Voûte de la Comptabilisation TVA](#iv-ob40--la-clé-de-voûte-de-la-comptabilisation-tva)
5. [Debugging de la Détermination TVA — /H, GD-EDIT, SE37](#v-debugging-de-la-détermination-tva--h-gd-edit-se37)
6. [Le Transport des Codes TVA — Le Maillon Critique](#vi-le-transport-des-codes-tva--le-maillon-critique)
7. [Flux MM vs Flux FI — Deux Logiques Distinctes](#vii-flux-mm-vs-flux-fi--deux-logiques-distinctes)
8. [S/4HANA vs ECC — Ce Qui Change Réellement](#viii-s4hana-vs-ecc--ce-qui-change-réellement)
9. [Checklist Complète — Paramétrage TVA France](#ix-checklist-complète--paramétrage-tva-france)

---

## I. Fondamentaux de la TVA dans SAP

### 1. Les deux flux fiscaux — une règle absolue

La TVA dans SAP distingue systématiquement deux directions. Cette distinction est la base de tout paramétrage correct :

| Type SAP | Désignation | Flux | Clé de compte | Compte typique |
|---|---|---|---|---|
| **Input Tax** | TVA Déductible | Achats entrants (P2P) | VST | 4456xx |
| **Output Tax** | TVA Collectée | Ventes sortantes (O2C) | MWS | 4457xx |
| **Non-deductible** | TVA non récupérable | Achats spécifiques | NVV | 658xxx |

> [!CAUTION]
> **Règle absolue — un code TVA = un seul flux**
>
> Un code de type **A (Output)** ne renseigne QUE la ligne MWS. La ligne VST reste vide.
> Un code de type **V (Input)** ne renseigne QUE la ligne VST. La ligne MWS reste vide.
>
> Renseigner les deux lignes sur un même code provoque une **double comptabilisation silencieuse** que personne ne détecte avant la déclaration fiscale.

### 2. La déclaration TVA — transaction F.38

**TVA à payer = TVA Collectée (MWS) − TVA Déductible (VST)**

| Transaction | Environnement | Usage |
|---|---|---|
| **F.38** | ECC et S/4HANA | Report de taxes — génère la pièce de déclaration et solde les comptes |
| **Fiori Tax Déclaration** | S/4HANA uniquement | Vue consolidée temps réel via Universal Journal ACDOCA |

> [!TIP]
> **Démo IDES** : F.38 — simulation déclaration TVA mensuelle, visualisation comptes soldés.

---

## II. Le Schéma de Calcul — La Mécanique Interne

### 1. Les trois objets fondamentaux — transaction OBYZ

Tout le paramétrage de la condition de taxe repose sur trois objets interconnectés accessibles depuis **OBYZ** :

| Objet | Transaction | Rôle |
|---|---|---|
| **Condition Types** | OBQ1 | Définit la nature technique de la taxe (MWAS, VST…) et ses règles de calcul |
| **Access Sequences** | OBQ2 | Définit l'ordre dans lequel SAP recherche les enregistrements de condition |
| **Procedures** | OBQ3 | Liste ordonnée des types de conditions — c'est le schéma de calcul complet |

### 2. La procédure TAXF — structure complète (OBQ3)

La procédure **TAXF** (Sales Tax — France) contient 6 étapes ordonnées. L'ordre des steps détermine la base de calcul de chaque taxe.

| Step | Condition | Description | From | Clé compte | Compte affecté |
|---|---|---|---|---|---|
| **100** | BASB | Base Amount | — | — | Montant HT de référence |
| **110** | MWAS | Output Tax | 100 | MWS | 4457xx — TVA collectée |
| **120** | MWVS | Input Tax | 100 | VST | 4456xx — TVA déductible |
| **140** | MWVN | Non-deduct. Input Tax | 100 | NVV | 658xxx — TVA non récup. |
| **150** | NLXA | Acquisition Tax Cred | 100 | ESA | TVA intra-communautaire crédit |
| **160** | NLXV | Acquisition Tax Deb. | 150 | ESE | TVA intra-communautaire débit |

> **Lecture du schéma :** le champ `From` indique le step de référence pour la base de calcul. Step 110 calcule depuis Step 100 (Base Amount). Si une remise est appliquée sur la commande, BASB intègre cette remise — la TVA se calcule donc sur le montant net après remise, pas sur le brut.

### 3. Le Type de Condition — transaction OBQ1

| Paramètre OBQ1 | Valeur | Impact |
|---|---|---|
| **Condition Class** | D — Taxes | Identifie l'objet comme une condition fiscale |
| **Calculation Type** | A — Percentage | Calcul en pourcentage de la base |
| **Condition Category** | D — Tax | Détermine le comportement dans la pièce comptable |
| **Dat.Rec.Source** | Condition Technique (SAP S/4HANA) | Spécifique S/4HANA — active le moteur Universal Journal |

> [!NOTE]
> Le champ `Dat.Rec.Source = Condition Technique (SAP S/4HANA)` **n'existe pas en ECC**. Il bascule la taxe vers le moteur unifié de l'Universal Journal (ACDOCA). Sans ce paramètre, la taxe reste sur l'ancien moteur et ne bénéficie pas du reporting fiscal temps réel de S/4HANA.

---

## III. Codes TVA — Transaction FTXP

### 1. Structure d'un code TVA

| Champ FTXP | Exemple | Signification |
|---|---|---|
| **Country/Region Key** | FR | Pays d'application du code |
| **Tax Code** | I4 | Identifiant du code (2 caractères) |
| **Procedure** | TAXF | Schéma de calcul affecté au pays |
| **Tax Type** | V — Input Tax | V=Achat/Input, A=Vente/Output |
| **Tax Percent Rate** | 20.600 | Taux saisi sur la bonne ligne de condition uniquement |

> [!TIP]
> **Démo IDES** : FTXP — création code I4 (Input 20.6%), saisie du taux sur la ligne VST uniquement.

---

## IV. OB40 — La Clé de Voûte de la Comptabilisation TVA

### 1. Principe de fonctionnement

**OB40** est la transaction qui crée le lien entre la **clé de transaction fiscale** (VST, MWS…) et le **compte général (GL)** sur lequel SAP doit comptabiliser automatiquement la taxe.

Sans OB40, le code TVA peut être parfaitement paramétré dans FTXP — SAP générera quand même une **erreur FF709** au moment de la simulation de la pièce car il ne sait pas où écrire la taxe.

### 2. Les clés de transaction TVA et leurs comptes

| Clé transaction | Type TVA | Compte GL standard | Sens | Observation |
|---|---|---|---|---|
| **MWS** | Output Tax collectée | 4457xx | Crédit | TVA due à l'État — compte de passif |
| **VST** | Input Tax déductible | 4456xx | Débit | TVA récupérable — compte d'actif |
| **NVV** | Non-deductible Input | 658xxx | Débit | Passé en charge — pas de récupération |
| **ESA** | Acquisition Tax Cred | 4452xx | Débit | TVA intracommunautaire côté crédit |
| **ESE** | Acquisition Tax Deb. | 4453xx | Crédit | TVA intracommunautaire côté débit |

### 3. Comment lire et paramétrer OB40

**Chemin SPRO :** `Finance > Paramétrage de base > Taxe sur CA > Comptabilisation > Définir les comptes de taxes`

Le paramétrage se fait en **deux niveaux** :
- **Niveau plan de comptes** : on sélectionne le plan de comptes (ex : CAFR, INT)
- **Niveau clé de transaction** : pour chaque clé (VST, MWS…), on saisit le numéro de compte GL

```
Exemple concret OB40 pour la France :
Plan de comptes : CAFR
  Clé VST  →  Compte 445620  (TVA déductible sur autres biens et services)
  Clé MWS  →  Compte 445710  (TVA collectée)
  Clé NVV  →  Compte 658200  (Charges — TVA non récupérable)
```

> Chaque société qui utilise le plan CAFR hérite de ces affectations. Si deux sociétés ont des comptes TVA différents, il faut créer des entrées supplémentaires dans OB40 avec la clé de société en critère de sélection.

### 4. Erreur FF709 — diagnostic et résolution

| Symptôme | Cause probable | Vérification | Résolution |
|---|---|---|---|
| **FF709 à la simulation FB60** | Clé VST non affectée dans OB40 | OB40 — vérifier clé VST sur plan de comptes | OB40 — affecter compte 4456xx à VST |
| **FF709 sur code spécifique** | Nouveau code TVA créé sans maj OB40 | FTXP — vérifier la clé de transaction du code | OB40 — ajouter l'affectation pour ce code |
| **FF709 en MIRO seulement** | Compte GL de la ligne de charges non taxable | FS00 — indicateur taxe sur la fiche compte | FS00 — activer `Postings with tax allowed` |
| **FF709 sur société spécifique** | OB40 configuré sur le plan mais pas la société | OB40 — vérifier niveau société vs plan comptes | OB40 — ajouter entrée au niveau société |

---

## V. Debugging de la Détermination TVA — /H, GD-EDIT, SE37

### 1. Pourquoi debugger la TVA

Quand une taxe ne se calcule pas, ne s'affiche pas, ou produit un mauvais montant, les messages d'erreur SAP sont souvent trop génériques. Le debug permet de suivre le code ABAP pas à pas et de voir exactement où la détermination échoue.

### 2. Activer le debugger SAP — commande /H

| Étape | Action | Détail |
|---|---|---|
| **1** | Saisir `/H` dans la barre de commande | Taper /H puis Entrée — le message `Debugging activated` apparaît en bas |
| **2** | Exécuter l'action problématique | Ex : simuler la pièce FB60 avec le code TVA incriminé |
| **3** | Le debugger s'ouvre | SAP interrompt l'exécution et affiche le code ABAP en cours |
| **4** | Naviguer avec F5/F6/F7/F8 | F5=pas à pas, F6=pas par-dessus, F7=sortir routine, F8=continuer |
| **5** | Inspecter les variables | Onglet `Variables` — chercher les variables KSCHL, KOMV, TXKRS |

> [!NOTE]
> **Variables clés à surveiller en debug TVA :**
> ```
> KOMV-KSCHL  →  Code de la condition de taxe active (MWAS, VST…)
> KOMV-KWERT  →  Montant calculé de la taxe
> KOMV-KBETR  →  Taux de la condition
> TXKRS       →  Taux de TVA lu depuis la table T007A
> BUKRS       →  Société en cours — vérifie que OB40 est lu sur la bonne société
> MWSKZ       →  Code TVA passé dans la pièce (= ce que l'utilisateur a saisi)
> ```

### 3. Le mode GD-EDIT — debug en mode édition

| Méthode | Usage | Avantage |
|---|---|---|
| **/H depuis menu principal** | Debug au lancement de la prochaine transaction | Simple — utilisable depuis n'importe où |
| **GD-EDIT en paramètre** | Debug directement sur l'écran d'édition courant | Utile quand /H se déclenche trop tôt dans le flux |
| **/HS (soft break)** | Point d'arrêt conditionnel | Permet de ne debugger que sous certaines conditions |

### 4. SE37 — Debugger une Function Module directement

SE37 est le **Function Builder**. Il permet de tester et debugger les function modules SAP directement, sans passer par la transaction appelante.

| Function Module | Rôle TVA | Quand l'utiliser |
|---|---|---|
| **CALCULATE_TAX_FROM_NET_AMOUNT** | Calcule la TVA depuis un montant HT | Debug quand la TVA ne se calcule pas depuis FB60 |
| **CALCULATE_TAX_FROM_GROSSAMOUNT** | Extrait la base HT depuis un montant TTC | Debug de l'indicateur `Calculer montant net TVA` |
| **GET_TAX_ACCOUNT** | Détermine le compte GL depuis OB40 | Debug erreur FF709 — voir quel compte est retourné |
| **FI_TAX_DETERMINE** | Détermination globale de la taxe | Debug de bout en bout quand la cause est inconnue |

> [!TIP]
> **Procédure de debug SE37 sur `GET_TAX_ACCOUNT` :**
> 1. SE37 → saisir `GET_TAX_ACCOUNT` → Test/Execute (F8)
> 2. Renseigner : `MWSKZ = code TVA testé (ex: I4)`, `BUKRS = société`
> 3. Activer `/H` avant d'exécuter
> 4. Observer la variable **HKONT** en sortie — c'est le compte que SAP va utiliser
> 5. Si `HKONT` est vide → OB40 manquant pour cette combinaison clé/société
> 6. Si `HKONT` est incorrect → OB40 pointe vers le mauvais compte GL

### 5. SE24 — Debugger les classes ABAP liées à la TVA

SE24 est le **Class Builder**. En S/4HANA, certains calculs de taxe sont encapsulés dans des classes ABAP orientées objet.

| Classe / Méthode | Rôle | Point de debug utile |
|---|---|---|
| **CL_FI_TAX_CALCULATION** | Classe principale calcul taxe S/4HANA | Méthode CALCULATE — vérifie les paramètres d'entrée |
| **CL_FI_TAX_ITEM** | Gestion des lignes de taxe | Méthode CREATE_TAX_ITEMS — debug création des lignes |
| **CL_FINS_CLEARING_CONT** | Réconciliation FI/CO Universal Journal | Utile si taxe présente en FI mais absente en CO |

> **Workflow de debug recommandé :**
> 1. `/H` depuis FB60 ou MIRO pour identifier le point de blocage
> 2. SE37 sur `GET_TAX_ACCOUNT` pour vérifier OB40
> 3. SE37 sur `CALCULATE_TAX_FROM_NET_AMOUNT` pour vérifier FTXP et TAXF
> 4. SE24 sur `CL_FI_TAX_CALCULATION` si les étapes 2 et 3 ne révèlent rien
> 5. SE16N sur `T007A` pour vérifier les taux en base directement

---

## VI. Le Transport des Codes TVA — Le Maillon Critique

### 1. Pourquoi FTXP ne fonctionne pas comme le reste du Customizing

| Type de paramétrage | Mécanisme de transport | Outil de contrôle |
|---|---|---|
| **OBQ1, OBQ3, OBBG, OB40** | Ordre Customizing classique SE09/SE10 | SE10 — visualisation de l'ordre |
| **FTXP — taux TVA uniquement** | Export/Import spécifique + Batch Input SM35 | SM35 — vérification exécution |

> [!WARNING]
> **L'erreur n°1 des consultants débutants**
>
> Vous modifiez un taux dans FTXP. Vous transportez votre ordre SE09. La STMS indique OK. Vous allez vérifier en QA ou Production : **le taux est toujours à 0%**.
>
> **Raison :** FTXP ne stocke pas les taux dans un ordre de transport classique. L'ordre SE09 transporte la **structure** du code (ses propriétés) — **PAS le taux**. Le taux passe par le programme `RFTAXIMP` et une session Batch Input **SM35**.

### 2. Cycle complet Export — Import — SM35

#### Étape 1 — Export depuis le système source (FTXP)

`FTXP > menu Tax Code > Transport > Export`

SAP demande le code pays (FR). Il génère un ordre de type **A** (ex : `A4HK900726`). Cet ordre **n'est PAS** un ordre SE09 classique.

#### Étape 2 — Import dans le système cible (FTXP)

`FTXP > Tax Code > Transport > Import`

Saisir le numéro d'ordre et le code pays FR. Execute. SAP lance `RFTAXIMP` qui crée une session Batch Input dans SM35 — **il ne joue pas encore les taux**.

#### Étape 3 — Exécution de la session Batch Input (SM35)

`SM35 > sélectionner la session TXFR_Kxxxxxx > Process`

SAP rejoue FTXP en arrière-plan pour chaque code TVA. C'est **à ce moment** que les taux sont réellement écrits dans le système cible.

> [!IMPORTANT]
> **Vérification finale :**
> ```
> SM35 Analysis :  New = 0  /  With Errors = 0  /  To Process = 0
> ```
> Seulement quand ces trois compteurs sont à zéro, les taux sont effectifs en système.
> Un ordre SE09 `OK` dans la STMS sans cette vérification **ne garantit rien**.

---

## VII. Flux MM vs Flux FI — Deux Logiques Distinctes

### 1. La différence fondamentale

| Flux | Point d'entrée TVA | Transaction facture | Compte TVA mouvementé |
|---|---|---|---|
| **FI direct** | Saisie manuelle FB60/FB70 | FB60 / FB70 | À la saisie de la pièce |
| **MM (P2P)** | Commande d'achat ME21N | MIRO — contrôle facture | Uniquement à la MIRO, pas à la MIGO |

### 2. Le cycle P2P — rôle de la TVA à chaque étape

| Étape | Transaction | TVA présente ? | Détail |
|---|---|---|---|
| **Commande achat** | ME21N | Code saisi — pas d'écriture | Code TVA déterminé (souvent depuis la Fiche Info-Achat) |
| **Entrée marchandises** | MIGO | Aucune écriture TVA | Mouvement de stock uniquement — TVA en attente |
| **Contrôle facture** | MIRO | Comptabilisation TVA | Compte 4456xx mouvementé — c'est ici que la TVA entre en comptabilité |

> [!CAUTION]
> **Erreur classique flux MM — code TVA incorrect dans la commande**
>
> Si le code TVA est faux dans ME21N (ex : V0 au lieu de I4), la MIRO sera bloquée ou générera une TVA à 0% **même si FTXP est correctement paramétré**. La FTXP juste ne suffit pas. Le code doit être correct à la source — dans la commande.
>
> Vérification : `ME23N > onglet Facture > champ Tax Code sur chaque ligne de commande`

### 3. Indicateurs de saisie dans FB60 / FB70

| Indicateur | État | Comportement SAP |
|---|---|---|
| **Calculer TVA** | Coché | SAP calcule la taxe automatiquement à la simulation |
| **Calculer montant net TVA** | NON ACTIF | Montant saisi = HT. TVA ajoutée en dehors. Total = HT + TVA |
| **Calculer montant net TVA** | ACTIF | Montant saisi = TTC. SAP extrait la base HT. Total = TTC saisi |

---

## VIII. S/4HANA vs ECC — Ce Qui Change Réellement

| Point | SAP ECC 6.0 | SAP S/4HANA |
|---|---|---|
| **Table de stockage** | BKPF + BSEG | ACDOCA (Universal Journal) |
| **Réconciliation FI/CO** | Nécessaire — tables séparées | Disparaît — tout dans ACDOCA |
| **Reporting TVA** | SE16N BSEG — extraction manuelle | Fiori Tax Journal — vue temps réel |
| **Déclaration TVA** | F.38 uniquement | F.38 + Fiori App dédiée |
| **Type de condition OBQ1** | Pas de champ Dat.Rec.Source | Dat.Rec.Source = Condition Technique S/4HANA |
| **Transport FTXP** | RFTAXIMP + SM35 identique | RFTAXIMP + SM35 identique |
| **Debug TVA** | SE37 + /H sur FM classiques | SE37 + SE24 sur classes CL_FI_TAX_* |

---

## IX. Checklist Complète — Paramétrage TVA France

| # | Objet | Transaction | Vérification |
|---|---|---|---|
| ✅ 1 | Schéma de calcul | OBQ3 | TAXF existe avec 6 steps (100 à 160) correctement ordonnés |
| ✅ 2 | Types de condition | OBQ1 | MWAS, VST, MWVN présents avec Dat.Rec.Source S/4HANA |
| ✅ 3 | Affectation pays | OBBG | Code pays FR affecté au schéma TAXF |
| ✅ 4 | Codes TVA | FTXP | Taux renseignés sur la bonne ligne — une seule ligne active par code |
| ✅ 5 | Comptes TVA OB40 | OB40 | VST→4456xx / MWS→4457xx / NVV→658xxx affectés et actifs |
| ✅ 6 | Compte GL taxable | FS00 | Indicateur `Postings with tax allowed` activé sur comptes charges/produits |
| ✅ 7 | Transport taux | FTXP + SM35 | Export effectué, Import exécuté, SM35 Analysis : 0 erreur, 0 To Process |
| ✅ 8 | Flux MM | ME21N/ME23N | Code TVA correct sur toutes les lignes de commande actives |
| ✅ 9 | Debug vérification | SE37 | GET_TAX_ACCOUNT retourne un HKONT non vide pour chaque code TVA |

---

## 🏢 À propos

**Auteur :** Moez L'Agha — Consultant SAP FI/CO  
**Structure :** MTC Computing (SIREN 820 341 558)  
**Contact :** [LinkedIn](https://www.linkedin.com/in/) · [SAP Community](https://community.sap.com/)

> Ce document fait partie d'une série de référentiels SAP FICO publiés sous la marque MTC Computing. Reproduction à des fins pédagogiques autorisée avec mention de la source.
