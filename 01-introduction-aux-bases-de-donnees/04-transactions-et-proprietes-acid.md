🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.4. Le concept de transaction et les propriétés ACID

## Introduction

Imaginez que vous êtes en train de faire un virement bancaire de 100 € de votre compte vers celui de votre ami. Cette opération simple implique en réalité **deux étapes** :

1. Retirer 100 € de votre compte  
2. Ajouter 100 € au compte de votre ami

Que se passerait-il si, entre ces deux étapes, le système informatique de la banque tombe en panne ? Vous perdriez 100 € qui disparaîtraient dans la nature ! Votre ami ne les recevrait pas, et vous ne les auriez plus. **Catastrophique.**

C'est exactement pour éviter ce type de problème que le concept de **transaction** existe. Dans cette section, nous allons découvrir comment les SGBDR comme PostgreSQL garantissent que vos données restent **cohérentes et fiables**, même en cas de problème.

---

## Qu'est-ce qu'une transaction ?

### Définition simple

Une **transaction** est un **ensemble d'opérations** qui doivent être exécutées de manière **indivisible** : soit **toutes les opérations réussissent**, soit **aucune n'est appliquée**.

En d'autres termes : **tout ou rien** (*all or nothing*).

### Analogie : Le saut en parachute

Pensez à un saut en parachute :

```
Étape 1 : Sauter de l'avion ✅
Étape 2 : Ouvrir le parachute ✅

Résultat : Atterrissage réussi ! 🎉
```

Maintenant, imaginez que le parachute ne s'ouvre pas :

```
Étape 1 : Sauter de l'avion ✅
Étape 2 : Ouvrir le parachute ❌

Résultat : PROBLÈME !
```

Dans une transaction, si l'Étape 2 échoue, le système **annule** automatiquement l'Étape 1. C'est comme si vous n'aviez jamais sauté de l'avion. Vous restez en sécurité à l'intérieur !

### Les commandes de base d'une transaction

En SQL, une transaction se déclare avec trois commandes principales :

```sql
BEGIN;  -- Début de la transaction

    -- Vos opérations ici
    UPDATE compte SET solde = solde - 100 WHERE id = 1;  -- Retrait
    UPDATE compte SET solde = solde + 100 WHERE id = 2;  -- Dépôt

COMMIT;  -- Valider la transaction (tout est OK)
```

Si un problème survient :

```sql
BEGIN;

    UPDATE compte SET solde = solde - 100 WHERE id = 1;
    -- ERREUR : Le compte 2 n'existe pas !
    UPDATE compte SET solde = solde + 100 WHERE id = 999;

ROLLBACK;  -- Annuler TOUTE la transaction
-- Le solde du compte 1 est revenu à son état initial
```

### Exemple concret : Le virement bancaire

Reprenons notre exemple de virement :

**Situation initiale** :
```
Compte Alice (id=1) : 500 €  
Compte Bob (id=2)   : 200 €  
```

**Transaction : Virement de 100 € d'Alice vers Bob**

```sql
BEGIN;

    -- Étape 1 : Débiter Alice
    UPDATE comptes SET solde = solde - 100 WHERE id = 1;
    -- Solde Alice : 400 €

    -- Étape 2 : Créditer Bob
    UPDATE comptes SET solde = solde + 100 WHERE id = 2;
    -- Solde Bob : 300 €

COMMIT;  -- Tout s'est bien passé, on valide !
```

**Résultat final** :
```
Compte Alice (id=1) : 400 €  ✅  
Compte Bob (id=2)   : 300 €  ✅  
Total : 700 € (conservé)  
```

**Scénario avec problème** :

```sql
BEGIN;

    -- Étape 1 : Débiter Alice
    UPDATE comptes SET solde = solde - 100 WHERE id = 1;
    -- Solde Alice : 400 €

    -- PANNE DE COURANT ! 💥

-- Quand le système redémarre, PostgreSQL détecte que la transaction
-- n'a pas été validée (pas de COMMIT) et effectue un ROLLBACK automatique

-- Résultat : Le solde d'Alice est revenu à 500 € ✅
```

**Sans transaction** (système catastrophique) :
```
Après Étape 1 : Alice = 400 €, Bob = 200 €
💥 PANNE
Après redémarrage : Alice = 400 €, Bob = 200 €
→ 100 € ont disparu ! ❌
```

**Avec transaction** (système fiable) :
```
Après Étape 1 : Alice = 400 € (changement temporaire, non validé)
💥 PANNE
Après redémarrage : Alice = 500 €, Bob = 200 €
→ ROLLBACK automatique, rien n'est perdu ! ✅
```

---

## Les propriétés ACID

Les transactions dans les SGBDR respectent quatre propriétés fondamentales, résumées par l'acronyme **ACID** :

- **A**tomicité  
- **C**ohérence  
- **I**solation  
- **D**urabilité

Ces propriétés garantissent que vos données restent **fiables et cohérentes**, même dans les situations les plus difficiles (pannes, accès simultanés, erreurs).

Explorons chacune de ces propriétés en détail.

---

## A - Atomicité

### Définition

L'**atomicité** garantit qu'une transaction est **indivisible** : soit **toutes** les opérations sont exécutées, soit **aucune**.

**"Atomique"** vient du grec *atomos* = "qui ne peut être divisé".

### Principe : Tout ou rien

```
Transaction réussie :
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Opération│ --> │ Opération│ --> │ Opération│
│    1     │  ✅ │    2     │  ✅ │    3     │  ✅
└──────────┘     └──────────┘     └──────────┘
                                       │
                                       ▼
                                    COMMIT
                              Tout est appliqué ✅

Transaction échouée :
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Opération│ --> │ Opération│ --> │ Opération│
│    1     │  ✅ │    2     │  ✅ │    3     │  ❌
└──────────┘     └──────────┘     └──────────┘
                                       │
                                       ▼
                                   ROLLBACK
                         TOUT est annulé, même 1 et 2 ✅
```

### Exemple : Réservation de billets d'avion

Vous réservez un vol Paris → New York avec escale à Londres :

```sql
BEGIN;

    -- 1. Réserver le siège Paris → Londres
    INSERT INTO reservations (vol, siege, passager)
    VALUES ('AF001', '12A', 'Jean Dupont');

    -- 2. Réserver le siège Londres → New York
    INSERT INTO reservations (vol, siege, passager)
    VALUES ('AF002', '15C', 'Jean Dupont');

    -- 3. Débiter le compte du client
    UPDATE comptes SET solde = solde - 850 WHERE client = 'Jean Dupont';

COMMIT;
```

**Scénario d'échec** :

```sql
BEGIN;

    INSERT INTO reservations (vol, siege, passager)
    VALUES ('AF001', '12A', 'Jean Dupont');  ✅

    INSERT INTO reservations (vol, siege, passager)
    VALUES ('AF002', '15C', 'Jean Dupont');  ✅

    -- Oups, le client n'a pas assez d'argent !
    UPDATE comptes SET solde = solde - 850 WHERE client = 'Jean Dupont';  ❌
    -- ERREUR : Solde insuffisant

ROLLBACK;  -- Automatique en cas d'erreur
```

**Résultat** : Les deux réservations de sièges sont **annulées automatiquement**. Le client ne se retrouve pas avec un vol Paris → Londres sans vol de correspondance !

### Pourquoi c'est crucial ?

Sans atomicité, vous pourriez vous retrouver dans des situations incohérentes :

- ❌ Argent débité mais produit non livré  
- ❌ Siège réservé mais paiement non effectué  
- ❌ Commande enregistrée mais stock non décrémenté

Avec atomicité :
- ✅ Soit tout fonctionne, soit rien ne change  
- ✅ Pas d'état intermédiaire corrompu

---

## C - Cohérence

### Définition

La **cohérence** garantit que la base de données passe toujours d'un **état valide** à un autre **état valide**, en respectant toutes les **règles d'intégrité** définies.

### Principe : Respect des règles

La base de données a des **contraintes** (règles) qui doivent toujours être respectées :

```sql
-- Exemples de contraintes :

-- Un solde ne peut pas être négatif
CHECK (solde >= 0)

-- Un email doit être unique
UNIQUE (email)

-- Un âge doit être positif et raisonnable
CHECK (age >= 0 AND age <= 150)

-- Une commande doit être liée à un client existant
FOREIGN KEY (client_id) REFERENCES clients(id)
```

**PostgreSQL refuse** toute transaction qui violerait ces règles.

### Exemple : Contrainte de solde positif

```sql
CREATE TABLE comptes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    solde NUMERIC(10,2) CHECK (solde >= 0)  -- Contrainte !
);

INSERT INTO comptes (nom, solde) VALUES ('Alice', 500);  
INSERT INTO comptes (nom, solde) VALUES ('Bob', 200);  
```

**Tentative de transaction violant la contrainte** :

```sql
BEGIN;

    -- Alice essaie de retirer 600 € alors qu'elle n'a que 500 €
    UPDATE comptes SET solde = solde - 600 WHERE nom = 'Alice';
    -- ERREUR : La contrainte CHECK (solde >= 0) est violée !
    -- Le nouveau solde serait -100 €, ce qui est interdit

ROLLBACK;  -- Transaction automatiquement annulée
```

**Résultat** :
```
Solde Alice : 500 €  ✅ (inchangé)
→ La base reste cohérente
```

### Exemple : Intégrité référentielle

```sql
-- Une commande doit toujours être liée à un client existant

CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INT REFERENCES clients(id),  -- Contrainte FK
    montant NUMERIC(10,2)
);

INSERT INTO clients (id, nom) VALUES (1, 'Alice');

-- Tentative de créer une commande pour un client inexistant
BEGIN;

    INSERT INTO commandes (client_id, montant)
    VALUES (999, 150.00);  -- Le client 999 n'existe pas !

    -- ERREUR : Violation de la contrainte de clé étrangère !

ROLLBACK;
```

**Résultat** : La commande n'est **pas créée**, préservant la cohérence de la base.

### Analogie : Les lois de la physique

Pensez aux contraintes comme aux **lois de la physique** :

- Vous ne pouvez pas créer de l'énergie à partir de rien
- Vous ne pouvez pas voyager plus vite que la lumière
- Un objet ne peut pas être à deux endroits en même temps

De même, dans votre base de données :
- Un solde ne peut pas être négatif
- Un email ne peut pas être dupliqué
- Une commande ne peut pas exister sans client

PostgreSQL **applique ces lois** rigoureusement.

---

## I - Isolation

### Définition

L'**isolation** garantit que les transactions **simultanées** (en parallèle) s'exécutent de manière **indépendante**, comme si elles étaient **seules** dans le système.

Chaque transaction ne voit pas les modifications **non validées** (non COMMITées) des autres transactions.

### Principe : Chacun dans sa bulle

```
    Transaction A              Transaction B
         │                          │
         │ BEGIN                    │
         │ Lit solde = 500€         │
         │                          │ BEGIN
         │                          │ Lit solde = 500€
         │ Retire 100€              │
         │ (solde = 400€)           │
         │ [PAS ENCORE VALIDÉ]      │
         │                          │ Retire 50€
         │                          │ (solde = 450€)
         │                          │ COMMIT ✅
         │ COMMIT ✅                │
         │                          │
    Résultat final : 350€ ✅
    (500 - 100 - 50 = 350)
```

**Sans isolation** (catastrophique) :
```
Les deux transactions lisent "500€" en même temps,  
retirent leur montant séparément,  
et écrivent chacune leur résultat.  

Résultat final : 450€ ❌ (un retrait est perdu !)
```

### Exemple : Réservation de places de cinéma

Imaginons qu'il reste **1 seule place** pour un film très attendu. Deux personnes essaient de la réserver en même temps.

**Scénario sans isolation** (problématique) :
```
10h00:00 - Alice consulte les places : Il reste 1 place ✅
10h00:00 - Bob consulte les places : Il reste 1 place ✅
10h00:05 - Alice réserve la place ✅
10h00:05 - Bob réserve la place ✅

Résultat : 2 personnes ont réservé la même place ! ❌
```

**Scénario avec isolation** (correct) :
```sql
-- Transaction Alice
BEGIN;
    SELECT places_disponibles FROM seances WHERE id = 1;
    -- Résultat : 1 place

    UPDATE seances SET places_disponibles = places_disponibles - 1
    WHERE id = 1 AND places_disponibles > 0;
    -- La ligne est VERROUILLÉE pour Alice
COMMIT;

-- Transaction Bob (démarre presque en même temps)
BEGIN;
    SELECT places_disponibles FROM seances WHERE id = 1;
    -- Bob ATTEND que la transaction d'Alice se termine

    -- Après COMMIT d'Alice, Bob lit : 0 place
    UPDATE seances SET places_disponibles = places_disponibles - 1
    WHERE id = 1 AND places_disponibles > 0;
    -- Aucune ligne mise à jour (places_disponibles = 0)
COMMIT;
```

**Résultat** : Une seule personne obtient la place ✅

### Les niveaux d'isolation

PostgreSQL propose plusieurs **niveaux d'isolation**, du plus permissif au plus strict :

| Niveau | Description | Performances | Anomalies possibles |
|--------|-------------|--------------|---------------------|
| **Read Uncommitted** | Lit les données non validées | Très rapide | Dirty Read |
| **Read Committed** | Lit uniquement les données validées (défaut PG) | Rapide | Non-Repeatable Read |
| **Repeatable Read** | Lecture cohérente dans la transaction | Moyen | Phantom Read |
| **Serializable** | Transactions complètement isolées | Plus lent | Aucune |

PostgreSQL utilise **Read Committed** par défaut, ce qui est un bon compromis pour la plupart des applications.

> 💡 **Note PostgreSQL** : Même si vous demandez le niveau *Read Uncommitted*, PostgreSQL l'exécutera comme *Read Committed*. Les *dirty reads* sont **impossibles** dans PostgreSQL, quel que soit le niveau d'isolation choisi. Les niveaux d'isolation seront détaillés au chapitre 12.

### Analogie : Les cabines d'essayage

Imaginez un magasin de vêtements :

- **Sans isolation** : Tout le monde essaie les vêtements en public, tout le monde voit tout  
- **Avec isolation** : Chacun a sa cabine privée, personne ne voit ce que vous essayez  
- **Niveau Serializable** : Cabines insonorisées + interdiction de sortir tant que quelqu'un d'autre essaie les mêmes vêtements

L'isolation vous donne votre **espace privé** pour travailler sans être perturbé par les autres.

---

## D - Durabilité

### Définition

La **durabilité** (ou **persistance**) garantit qu'une fois qu'une transaction est **validée** (COMMITée), ses modifications sont **permanentes**, même en cas de :

- Panne de courant 💥
- Crash du système 💻
- Défaillance matérielle 🔥

### Principe : Ce qui est fait est fait

```
Transaction :
┌────────────────────────────────┐
│ BEGIN                          │
│ UPDATE comptes SET ...         │
│ INSERT INTO logs ...           │
│ COMMIT  ✅                     │ ← À partir d'ici, c'est PERMANENT
└────────────────────────────────┘
         │
         ▼
    💥 PANNE !
         │
         ▼
    Redémarrage
         │
         ▼
Les modifications sont TOUJOURS présentes ✅
```

### Comment PostgreSQL garantit la durabilité

PostgreSQL utilise le **WAL** (*Write-Ahead Log*) ou **journal des transactions** :

1. **Avant** d'écrire les modifications sur le disque, PostgreSQL les écrit dans le WAL  
2. Lors du COMMIT, le WAL est **synchronisé** sur le disque (fsync)  
3. En cas de crash, PostgreSQL **rejoue** le WAL au redémarrage

```
┌─────────────────────────────────────────────────────┐
│             Fonctionnement du WAL                   │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. BEGIN                                           │
│  2. Modification en mémoire                         │
│  3. Écriture dans le WAL (journal)    ✅ Sur disque │
│  4. COMMIT → Synchronisation du WAL   ✅ Garanti    │
│  5. Écriture des données réelles      🕐 Différée   │
│                                                     │
└─────────────────────────────────────────────────────┘

💥 Si panne entre 4 et 5 :
→ Au redémarrage, PostgreSQL lit le WAL
→ Rejoue les transactions validées
→ Aucune perte de données ✅
```

### Exemple : Paiement en ligne

```sql
BEGIN;

    -- 1. Enregistrer la commande
    INSERT INTO commandes (client_id, total) VALUES (1, 99.99);

    -- 2. Débiter le compte
    UPDATE comptes SET solde = solde - 99.99 WHERE client_id = 1;

    -- 3. Envoyer à l'entrepôt
    INSERT INTO expeditions (commande_id, statut) VALUES (1, 'en_attente');

COMMIT;  -- Tout est écrit dans le WAL et synchronisé
```

**Scénario** :
```
14h30:00 - Client valide le paiement
14h30:01 - PostgreSQL écrit dans le WAL
14h30:02 - COMMIT → Synchronisation du WAL sur disque ✅
14h30:03 - "Paiement confirmé" affiché au client
14h30:04 - 💥 PANNE DE COURANT !

14h35:00 - Redémarrage du serveur
14h35:05 - PostgreSQL rejoue le WAL
14h35:10 - La commande, le débit et l'expédition sont présents ✅
```

Le client a vu "Paiement confirmé", et c'est garanti. Il ne perdra pas sa commande.

### Analogie : Le notaire

Un notaire enregistre les actes importants (ventes, mariages, etc.) :

1. Vous signez l'acte  
2. Le notaire l'enregistre **officiellement** dans un registre  
3. Ce registre est **archivé de manière sécurisée**  
4. Même si le bureau du notaire brûle, les archives officielles existent ailleurs

De même, PostgreSQL :
1. Vous faites un COMMIT  
2. PostgreSQL écrit dans le WAL  
3. Le WAL est **synchronisé sur disque**  
4. Même si le serveur crash, les données validées sont récupérables

---

## Résumé des propriétés ACID

Récapitulons les 4 propriétés avec un exemple unifié : **virement bancaire de 100 € d'Alice vers Bob**.

| Propriété | Garantie | Exemple virement |
|-----------|----------|------------------|
| **Atomicité** | Tout ou rien | Si le débit d'Alice réussit mais le crédit de Bob échoue, on annule tout |
| **Cohérence** | Règles respectées | Le total des soldes reste constant (500+200 = 400+300) |
| **Isolation** | Indépendance | Si Carol fait aussi un virement en même temps, pas de conflit |
| **Durabilité** | Permanence | Après COMMIT, même si le serveur crash, le virement est enregistré |

### Schéma visuel ACID

```
┌─────────────────────────────────────────────────────────────┐
│                    TRANSACTION BANCAIRE                     │
│                 Virement 100€ : Alice → Bob                 │
└─────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
           ┌────▼────┐                 ┌────▼────┐
           │ ATOMICITÉ│                │COHÉRENCE│
           │          │                │         │
           │ Débit ET │                │ Soldes  │
           │ Crédit   │                │ positifs│
           │ ou rien  │                │ Total = │
           └────┬────┘                 │ constant│
                │                      └────┬────┘
                │                           │
                └──────────┬────────────────┘
                           │
                    ┌──────▼──────┐
                    │  ISOLATION  │
                    │             │
                    │ Autres      │
                    │ transactions│
                    │ isolées     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ DURABILITÉ  │
                    │             │
                    │ WAL sync    │
                    │ Permanent   │
                    │ après COMMIT│
                    └─────────────┘
```

---

## ACID vs BASE : Comparaison avec NoSQL

Les bases NoSQL utilisent souvent un modèle différent appelé **BASE** :

| Modèle | ACID (SQL) | BASE (NoSQL) |
|--------|------------|--------------|
| **Signification** | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| **Cohérence** | Immédiate et stricte | À terme (éventuelle) |
| **Disponibilité** | Peut être impactée | Priorité absolue |
| **Partitionnement** | Plus difficile | Facile et natif |
| **Cas d'usage** | Banque, commerce, ERP | Réseaux sociaux, analytics, IoT |

**Exemple BASE** (Cassandra, DynamoDB) :

```
11h00 - Alice publie une photo sur Instagram
11h01 - Ses amis à Paris voient la photo ✅
11h05 - Ses amis à Tokyo ne la voient pas encore ⏳
11h10 - Ses amis à Tokyo la voient enfin ✅

→ Cohérence "à terme" (eventually consistent)
```

**Exemple ACID** (PostgreSQL) :

```
11h00 - Alice fait un virement de 100 €
11h01 - Bob voit son solde augmenter instantanément ✅
11h01 - Tous les utilisateurs voient la même information ✅

→ Cohérence immédiate et stricte
```

**Principe** :
- **ACID** : Cohérence > Performance  
- **BASE** : Disponibilité > Cohérence stricte

Pour des données transactionnelles critiques (argent, santé, inventaire), **ACID est essentiel**.

---

## Pourquoi ACID est crucial

### Domaines où ACID est indispensable

1. **Finance et Banque** 💰
   - Virements
   - Paiements
   - Comptabilité

2. **E-commerce** 🛒
   - Commandes
   - Gestion de stock
   - Paiements en ligne

3. **Santé** 🏥
   - Dossiers médicaux
   - Prescriptions
   - Rendez-vous

4. **Réservations** ✈️
   - Billets d'avion
   - Hôtels
   - Événements

5. **Systèmes d'inventaire** 📦
   - Stock magasin
   - Logistique
   - Supply chain

### Conséquences sans ACID

**Scénario catastrophe sans ACID** :

```
Système de billetterie de concert :
- 1000 places disponibles
- 10 000 personnes réservent en même temps
- Sans isolation : 3000 places "vendues"
→ 2000 personnes se présentent sans place ! ❌

Système bancaire :
- Virement de 1000 €
- Panne au milieu
- Sans atomicité : l'argent disparaît
→ Procès, perte de confiance, faillite ! ❌

E-commerce :
- Article en stock : 1
- 5 commandes simultanées
- Sans cohérence : 5 commandes validées
→ 4 clients mécontents ! ❌
```

**Avec ACID (PostgreSQL)** :

```
✅ Une seule personne obtient la dernière place
✅ Le virement est complet ou annulé, jamais partiel
✅ Une seule commande validée pour le dernier article
```

---

## Transactions dans PostgreSQL : Commandes avancées

### SAVEPOINT : Points de sauvegarde

Permet d'annuler partiellement une transaction :

```sql
BEGIN;

    INSERT INTO commandes (client_id, total) VALUES (1, 100);
    -- Commande créée

    SAVEPOINT before_items;

    INSERT INTO lignes_commande (produit_id, quantite) VALUES (1, 5);
    INSERT INTO lignes_commande (produit_id, quantite) VALUES (2, 3);
    -- Oups, erreur sur le deuxième produit !

    ROLLBACK TO SAVEPOINT before_items;
    -- On annule juste les lignes, pas la commande !

    -- On réessaie avec les bonnes valeurs
    INSERT INTO lignes_commande (produit_id, quantite) VALUES (1, 5);
    INSERT INTO lignes_commande (produit_id, quantite) VALUES (3, 2);

COMMIT;
```

### Transaction en lecture seule

Pour des analyses sans risque de modification :

```sql
BEGIN TRANSACTION READ ONLY;

    -- Requêtes de lecture
    SELECT * FROM commandes;
    SELECT SUM(total) FROM factures;

    -- Cette commande échouerait :
    -- DELETE FROM commandes WHERE id = 1;  ❌

COMMIT;
```

### Transaction avec niveau d'isolation

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

    -- Transaction strictement isolée
    SELECT * FROM compte WHERE id = 1 FOR UPDATE;
    UPDATE compte SET solde = solde - 100 WHERE id = 1;

COMMIT;
```

---

## Récapitulatif

### Points clés à retenir

✅ **Une transaction** est un ensemble d'opérations indivisible : tout ou rien

✅ **ACID** garantit la fiabilité des données :  
   - **A**tomicité : Tout ou rien  
   - **C**ohérence : Règles respectées  
   - **I**solation : Transactions indépendantes  
   - **D**urabilité : Modifications permanentes après COMMIT

✅ **PostgreSQL respecte strictement ACID**, ce qui le rend idéal pour les applications critiques

✅ **Le WAL** (Write-Ahead Log) garantit la durabilité des données

✅ **Les transactions protègent** contre les erreurs, les pannes et les accès concurrents

✅ **Commandes SQL** : BEGIN, COMMIT, ROLLBACK, SAVEPOINT

### Ce que vous avez appris

Dans cette section, vous avez découvert :

- Le concept de transaction et son importance
- Les 4 propriétés ACID en détail avec des exemples concrets
- Comment PostgreSQL garantit chaque propriété
- La différence entre ACID (SQL) et BASE (NoSQL)
- Les commandes pour gérer les transactions en SQL
- Pourquoi ACID est crucial pour les applications critiques

### Pourquoi c'est important ?

Les propriétés ACID sont **la fondation** de la fiabilité de PostgreSQL. C'est ce qui permet de :

- Gérer des millions de transactions bancaires par jour sans erreur
- Vendre des billets en ligne sans survente
- Garantir que vos données sont cohérentes et fiables
- Dormir tranquille en sachant que vos données sont protégées

**Sans ACID, les bases de données modernes n'existeraient pas.**

### Et maintenant ?

Maintenant que vous comprenez les concepts fondamentaux (données, bases de données, SGBD, modèle relationnel, transactions ACID), nous allons dans le **Chapitre 2** commencer à découvrir **PostgreSQL en détail** :

- Son histoire et sa philosophie
- Ses versions et son écosystème
- Ses forces et son positionnement dans l'industrie

Vous êtes maintenant prêt à entrer dans le vif du sujet !

---


⏭️ [Présentation de PostgreSQL](/02-presentation-de-postgresql/README.md)
