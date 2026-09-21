# SpotDeck — Économie & Marché

Ce document décrit toute l'économie du jeu : la monnaie **Spotz**, la valeur des
cartes, le recyclage, le marché entre joueurs et la boutique. C'est la référence
pour le code serveur. Rien de tout ça ne se calcule côté navigateur.

---

## 1. Principe fondateur : SENS UNIQUE

L'argent circule dans un seul sens :

```
Argent réel (€)  →  Spotz  →  Cartes / Packs
```

- On **peut** : acheter des Spotz (Stripe), vendre une carte contre des Spotz,
  acheter cartes et packs avec des Spotz.
- On ne **peut jamais** : reconvertir des Spotz en euros, ni les retirer, ni les
  transférer directement d'un joueur à l'autre.

Pourquoi c'est vital : dès qu'un joueur peut retirer du vrai argent, le jeu tombe
sous la réglementation des **jeux d'argent** et de la **transmission de fonds**.
Le sens unique garde SpotDeck dans la catégorie « monnaie de jeu », légale et
simple. Les Spotz n'ont aucune valeur monétaire réelle : ce sont des jetons de jeu.

---

## 2. La monnaie : les Spotz

- Nom : **Spotz** (« j'ai 450 Spotz », « cette carte vaut 200 Spotz »).
- Symbole possible dans l'UI : une petite gemme / lion doré (thème d'Arras).
- Stockés comme un entier dans le profil du joueur, **modifiable seulement par le
  serveur**.

---

## 3. Robinets & éviers (l'équilibre de l'économie)

Une économie tient si la monnaie créée est aussi détruite. Sinon, inflation : tout
le monde est riche et plus rien n'a de valeur.

**Robinets (création de Spotz)**
- Achat de Spotz via Stripe.
- Revente d'une carte (recyclage ou vente sur le marché).

**Éviers (destruction de Spotz)**
- Achat de packs en Spotz.
- **Commission** prélevée sur chaque vente du marché (elle disparaît).

**La règle d'or :** revendre une carte doit rapporter **moins** que ce qu'elle
coûte à obtenir dans un pack. Sinon les joueurs farment des packs, recyclent, et
génèrent des Spotz à l'infini. Le recyclage à ~40 % de la valeur garantit ça.

---

## 4. Valeur des cartes

Chaque carte a une **valeur de base** en Spotz, liée à sa rareté. Le **recyclage**
(vente instantanée au système) rend une fraction de cette valeur.

| Rareté | Valeur de base | Recyclage (~40 %) |
|--------|---------------:|------------------:|
| Commune | 10 | 4 |
| Peu commun | 25 | 10 |
| Rare | 75 | 30 |
| Épique | 200 | 80 |
| Légendaire | 600 | 240 |
| Mythique | 2000 | 800 |

Ces valeurs sont la **source de vérité**, définies côté serveur (voir
`spotz-config`). L'app ne fait que les afficher.

---

## 5. Vendre une carte : deux voies

### a) Recycler (vendre au système)
- Instantané. Le joueur reçoit la valeur de recyclage en Spotz.
- Idéal pour les doublons et les cartes communes.
- Robinet contrôlé : le système crée les Spotz, mais peu (40 %).

### b) Publier sur le Marché (vendre à un autre joueur)
- Le joueur fixe son prix (dans des bornes : entre le recyclage et un plafond,
  ex. 3× la valeur de base, pour éviter les prix absurdes).
- Un autre joueur paie en Spotz.
- Le vendeur reçoit **prix − commission** (commission ~15 %, détruite = évier).
- Réservé aux cartes rares et au-dessus (les communes ne vont qu'au recyclage,
  pour ne pas noyer le marché).

---

## 6. Le Marché — flux complet

```
Joueur A publie « Le Beffroi » (Légendaire) à 700 Spotz
        │
Joueur B veut l'acheter, il n'a que 300 Spotz
        │
   ┌──────────────────────────────────────┐
   │ Option 1 : vendre ses doublons        │ → gagne des Spotz
   │ Option 2 : acheter des Spotz (Stripe) │ → argent réel
   └──────────────────────────────────────┘
        │
Joueur B atteint 700 Spotz → il achète
        │
   Transaction ATOMIQUE (tout ou rien) :
     - débit 700 Spotz chez B
     - crédit 595 Spotz chez A (700 − 15 %)
     - 105 Spotz détruits (évier)
     - la carte passe de A à B
     - l'annonce est retirée du marché
```

**Atomicité obligatoire :** ces 5 opérations réussissent ensemble ou échouent
ensemble. Jamais « B a payé mais n'a pas reçu la carte » ni l'inverse. C'est une
transaction de base de données.

---

## 7. La Boutique (acheter des Spotz avec Stripe)

Paiement via **Stripe Checkout + Link** (comme wiki-masters), sur le web. Pas
d'Apple Pay ni de commission App Store puisque c'est un site.

| Pack | Spotz | Bonus | Prix |
|------|------:|------:|-----:|
| Poignée | 100 | — | 0,99 € |
| Bourse | 550 | +10 % | 4,99 € |
| Coffre | 1200 | +20 % | 9,99 € |
| Trésor | 2600 | +30 % | 19,99 € |

Les gros paliers donnent plus de bonus → incitation à acheter en volume.

**Sécurité paiement :** le crédit des Spotz se fait **après** confirmation du
paiement par Stripe (webhook signé), jamais sur un simple retour du navigateur.
Chaque paiement ne crédite qu'une fois (idempotence par ID de session Stripe).

---

## 8. Modèle de données (à ajouter à celui de SpotDeck)

```
config/spotz                         ← valeurs, lues par le serveur
  { rarityValues: {...}, recyclePct: 0.4, marketFeePct: 0.15,
    priceCapMultiplier: 3 }

users/{uid}
  { ..., spotz: 0 }                  ← solde, écrit SEULEMENT par le serveur

users/{uid}/collection/{instanceId}  ← une carte POSSÉDÉE (instance unique)
  { cardId, rarity, obtainedAt, listedOnMarket: false }
  # note : pour un marché entre joueurs, chaque carte est une INSTANCE unique
  # (avec son propre id), pas juste un compteur, pour pouvoir la transférer.

market/{listingId}                   ← une annonce en cours
  { sellerUid, instanceId, cardId, rarity, price,
    createdAt, status: 'active' }

transactions/{txId}                  ← journal (achats Stripe, ventes, recyclages)
  { type, uid, amountSpotz, ref, createdAt }

processed_stripe_sessions/{sessionId} ← anti-double-crédit des paiements
```

**Changement important vs la v1 :** avant, la collection comptait les cartes
(`count: 3`). Pour un marché, chaque carte devient une **instance unique**
transférable. Un joueur qui a 3 exemplaires du Beffroi a 3 instances distinctes,
dont il peut vendre une et garder deux.

---

## 9. Fonctions serveur à écrire

- `recycleCard(instanceId)` → détruit l'instance, crédite la valeur de recyclage.
- `listCard(instanceId, price)` → vérifie les bornes de prix, marque la carte en
  vente, crée l'annonce.
- `unlistCard(listingId)` → retire l'annonce.
- `buyListing(listingId)` → transaction atomique (section 6).
- `buySpotz(stripeSession)` → après webhook Stripe validé, crédite les Spotz.

Toutes vérifient que le joueur est bien connecté et propriétaire, et refusent si
le solde est insuffisant. Aucune ne fait confiance à un montant envoyé par le
client.

---

## 10. Règles anti-abus

- Prix du marché borné (entre recyclage et 3× la valeur de base) → pas de
  blanchiment via des ventes truquées entre deux comptes complices.
- Commune non vendable sur le marché → seulement recyclable.
- Un joueur ne peut pas acheter sa propre annonce.
- Journalisation de toutes les transactions → détection des comportements
  anormaux.
