🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.4. Anomalies transactionnelles : Dirty Read, Non-Repeatable Read, Phantom Read

## Introduction : Quand les transactions se rencontrent

Imaginez une bibliothèque où plusieurs personnes modifient simultanément le même catalogue de livres. Sans règles claires, le chaos s'installe : quelqu'un lit un livre qui vient d'être retiré, un autre voit le prix changer au milieu de sa consultation, un troisième trouve de nouveaux livres apparaître magiquement dans sa recherche...

Ces situations problématiques s'appellent des **anomalies transactionnelles**. Ce sont des comportements inattendus et souvent indésirables qui peuvent survenir lorsque plusieurs transactions accèdent aux mêmes données simultanément.

Dans ce chapitre, nous allons explorer en profondeur chaque type d'anomalie, comprendre comment elle se produit, et découvrir comment PostgreSQL les prévient grâce aux niveaux d'isolation.

---

## Vue d'ensemble des anomalies transactionnelles

### Les trois anomalies classiques du standard SQL ANSI

Le standard SQL ANSI définit trois anomalies principales :

1. **Dirty Read** (Lecture sale) : Lire des données non validées
2. **Non-Repeatable Read** (Lecture non répétable) : Relire la même ligne et obtenir un résultat différent
3. **Phantom Read** (Lecture fantôme) : Relire avec la même requête et obtenir des lignes supplémentaires/manquantes

### Au-delà du standard ANSI

PostgreSQL et les bases de données modernes reconnaissent aussi d'autres anomalies :

4. **Lost Update** (Mise à jour perdue) : Une modification écrase silencieusement une autre
5. **Write Skew** (Biais d'écriture) : Deux transactions violent une contrainte en écrivant simultanément
6. **Read Skew** (Biais de lecture) : Lire des données incohérentes entre elles

### Tableau de prévention selon les niveaux d'isolation

| Anomalie | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|----------|------------------|----------------|-----------------|--------------|
| Dirty Read | ❌ Possible | ✅ Impossible | ✅ Impossible | ✅ Impossible |
| Non-Repeatable Read | ❌ Possible | ❌ Possible | ✅ Impossible | ✅ Impossible |
| Phantom Read | ❌ Possible | ❌ Possible | ✅ Impossible* | ✅ Impossible |
| Lost Update | ❌ Possible | ⚠️ Parfois | ✅ Impossible | ✅ Impossible |
| Write Skew | ❌ Possible | ❌ Possible | ❌ Possible | ✅ Impossible |
| Read Skew | ❌ Possible | ❌ Possible | ✅ Impossible | ✅ Impossible |

*PostgreSQL va au-delà du standard ANSI : même Repeatable Read empêche les Phantom Reads.

---

## 1. Dirty Read (Lecture sale)

### Définition

Une **Dirty Read** se produit lorsqu'une transaction lit des données qui ont été modifiées par une autre transaction **non encore validée** (non commitée). Si cette autre transaction fait un ROLLBACK, les données lues n'ont "jamais existé" réellement.

**Analogie** : Vous lisez un brouillon d'article sur l'écran d'un collègue pendant qu'il le rédige. Vous prenez des notes, mais votre collègue décide finalement de tout supprimer. Vos notes référencent un contenu qui n'a jamais été publié.

### Exemple détaillé

#### Contexte

Deux utilisateurs travaillent sur un système bancaire :
- **Alice** (Transaction A) : Consulte son solde
- **Bob** (Transaction B) : Effectue un retrait mais annule ensuite

#### Scénario avec Dirty Read (niveau Read Uncommitted)

**État initial** :
```sql
-- Table comptes
id | proprietaire | solde
---|--------------|-------
1  | Alice        | 1000.00
```

**Timeline des événements** :

```
T1 (10:00:00) - Transaction B (Bob) démarre
```
```sql
BEGIN;  -- Transaction B
UPDATE comptes SET solde = solde - 500 WHERE id = 1;
-- Le solde est maintenant 500€ DANS LA TRANSACTION B (pas encore commité)
```

```
T2 (10:00:05) - Transaction A (Alice) démarre
```
```sql
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;  -- Transaction A

SELECT solde FROM comptes WHERE id = 1;
-- Dirty Read ! Alice voit : 500€
-- Elle lit une donnée NON COMMITÉE de Transaction B

-- Alice prend une décision basée sur ce solde
SELECT CASE
    WHEN solde < 600 THEN 'Compte bientôt à découvert !'
    ELSE 'Solde OK'
END FROM comptes WHERE id = 1;
-- Résultat : "Compte bientôt à découvert !"

COMMIT;
```

```
T3 (10:00:10) - Transaction B annule tout
```
```sql
-- Transaction B
ROLLBACK;  -- Bob annule le retrait !
```

**État final** :
```sql
SELECT solde FROM comptes WHERE id = 1;
-- Résultat : 1000€ (le solde n'a jamais changé !)
```

#### Conséquences

Alice a pris une décision (penser que son compte est bientôt à découvert) basée sur des données qui **n'ont jamais existé** réellement. C'est une **Dirty Read**.

**Problèmes causés** :
- 🚨 Décisions métier incorrectes
- 🚨 Calculs faux (statistiques, rapports)
- 🚨 Violation de l'intégrité logique de l'application
- 🚨 Données incohérentes affichées aux utilisateurs

### Comment PostgreSQL prévient les Dirty Reads

PostgreSQL **n'autorise JAMAIS les Dirty Reads**, même si vous demandez explicitement Read Uncommitted :

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- PostgreSQL traite cela comme READ COMMITTED
-- Aucune donnée non commitée ne sera jamais visible
```

**Mécanisme MVCC** : Grâce au MVCC, chaque transaction voit seulement les versions de lignes qui ont été validées (xmax = 0 ou xmax correspondant à une transaction commitée).

### Visualisation du MVCC empêchant Dirty Read

```
Transaction B : UPDATE solde = 500
    ↓
Crée une nouvelle version : xmin=B, xmax=0, solde=500
    ↓
Version pas encore visible (Transaction B pas commitée)
    ↓
Transaction A : SELECT solde
    ↓
Voit seulement l'ancienne version : xmin=ancien, solde=1000
    ↓
✅ Pas de Dirty Read !
```

---

## 2. Non-Repeatable Read (Lecture non répétable)

### Définition

Une **Non-Repeatable Read** se produit lorsqu'une transaction lit la **même ligne deux fois** et obtient des **valeurs différentes** parce qu'une autre transaction a modifié et validé cette ligne entre-temps.

**Analogie** : Vous consultez le prix d'un produit sur un site e-commerce à 10h00 : 100€. Vous réfléchissez, puis à 10h05 vous rafraîchissez la page : le prix est maintenant 150€. Le prix a changé pendant votre consultation !

### Exemple détaillé

#### Contexte

Un système de réservation d'hôtel où les prix changent fréquemment.

**État initial** :
```sql
-- Table chambres
id | numero | prix_nuit
---|--------|----------
1  | 101    | 100.00
```

#### Scénario avec Non-Repeatable Read (niveau Read Committed)

**Transaction A (Client consulte et réfléchit)** :

```
T1 (10:00:00) - Transaction A démarre
```
```sql
BEGIN;  -- Read Committed par défaut

-- Première lecture
SELECT prix_nuit FROM chambres WHERE id = 1;
-- Résultat : 100€

-- Le client calcule le coût total pour 3 nuits
SELECT prix_nuit * 3 AS cout_total FROM chambres WHERE id = 1;
-- Résultat : 300€

-- [Le client réfléchit pendant 2 minutes...]
```

**Transaction B (Système ajuste les prix)** :

```
T2 (10:00:30) - Transaction B démarre et se termine rapidement
```
```sql
BEGIN;

-- Augmentation automatique du prix (peak hours)
UPDATE chambres SET prix_nuit = 150.00 WHERE id = 1;

COMMIT;  -- ✅ Validé !
```

**Transaction A (suite)** :

```
T3 (10:02:00) - Transaction A continue
```
```sql
-- Le client décide de réserver, il relit le prix pour confirmation
SELECT prix_nuit FROM chambres WHERE id = 1;
-- Résultat : 150€ (!!)  <-- Différent de la première lecture !

-- Son calcul initial était faux !
SELECT prix_nuit * 3 AS cout_total FROM chambres WHERE id = 1;
-- Résultat : 450€ (au lieu des 300€ attendus)

COMMIT;
```

#### Analyse de l'anomalie

Dans la **même transaction** (Transaction A), la même requête (`SELECT prix_nuit FROM chambres WHERE id = 1`) a retourné :
- Première fois : **100€**
- Deuxième fois : **150€**

C'est une **Non-Repeatable Read** : la lecture n'est pas répétable avec le même résultat.

#### Conséquences métier

```
Client voit initialement : "Réservation : 300€ pour 3 nuits"
                ↓
Client confirme la réservation
                ↓
Système facture : 450€ (!!)
                ↓
🚨 Confusion du client, plaintes, perte de confiance
```

**Autres problèmes causés** :
- 💸 Incohérences de facturation
- 📊 Rapports avec des totaux incorrects
- 🎯 Violations de règles métier basées sur des valeurs stables
- 🔄 Calculs complexes produisant des résultats erronés

### Diagramme temporel de Non-Repeatable Read

```
Transaction A                          Transaction B
    |                                      |
    | BEGIN                                |
    |                                      |
    | SELECT prix = 100€                   |
    |                                      |
    | [Traitement...]                      | BEGIN
    |                                      | UPDATE prix = 150€
    |                                      | COMMIT ✅
    |                                      |
    | SELECT prix = 150€ (!) <-- Changement visible
    |                                      |
    | COMMIT                               |
    |                                      |
```

### Comment prévenir les Non-Repeatable Reads

#### Solution 1 : Repeatable Read

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Première lecture (snapshot pris ici et FIGÉ)
SELECT prix_nuit FROM chambres WHERE id = 1;
-- Résultat : 100€

-- [Transaction B modifie et commit]

-- Deuxième lecture (même snapshot)
SELECT prix_nuit FROM chambres WHERE id = 1;
-- Résultat : TOUJOURS 100€ ✅

COMMIT;
```

**Avantage** : Cohérence garantie dans toute la transaction.

#### Solution 2 : Verrouillage explicite (FOR UPDATE)

```sql
BEGIN;

-- Verrouiller la ligne pour empêcher les modifications
SELECT prix_nuit FROM chambres WHERE id = 1 FOR UPDATE;
-- Résultat : 100€

-- Transaction B doit attendre si elle essaie de modifier cette ligne

-- Relecture (ligne toujours verrouillée)
SELECT prix_nuit FROM chambres WHERE id = 1;
-- Résultat : 100€ (garanti)

COMMIT;  -- Libère le verrou
```

**Avantage** : Empêche les autres transactions de modifier la ligne.
**Inconvénient** : Crée des contentions (autres transactions bloquées).

---

## 3. Phantom Read (Lecture fantôme)

### Définition

Une **Phantom Read** se produit lorsqu'une transaction exécute la **même requête deux fois** et obtient un **nombre différent de lignes** parce qu'une autre transaction a inséré ou supprimé des lignes correspondant aux critères de recherche.

**Analogie** : Vous comptez le nombre de livres sur une étagère : 10 livres. Vous vous retournez 30 secondes, puis recomptez : maintenant 12 livres ! Des livres "fantômes" sont apparus.

### Exemple détaillé

#### Contexte

Un système de gestion de commandes où un manager calcule des statistiques.

**État initial** :
```sql
-- Table commandes
id | client_id | montant | statut
---|-----------|---------|----------
1  | 42        | 100.00  | En attente
2  | 42        | 150.00  | En attente
3  | 42        | 200.00  | Livrée
```

#### Scénario avec Phantom Read (niveau Read Committed)

**Transaction A (Manager génère un rapport)** :

```
T1 (11:00:00) - Transaction A démarre
```
```sql
BEGIN;  -- Read Committed par défaut

-- Compter les commandes en attente pour le client 42
SELECT COUNT(*) FROM commandes
WHERE client_id = 42 AND statut = 'En attente';
-- Résultat : 2 commandes

-- Calculer le montant total
SELECT SUM(montant) FROM commandes
WHERE client_id = 42 AND statut = 'En attente';
-- Résultat : 250€ (100 + 150)

-- [Génération du reste du rapport...]
```

**Transaction B (Nouvelle commande créée)** :

```
T2 (11:00:15) - Transaction B démarre et se termine
```
```sql
BEGIN;

-- Un nouveau client crée une commande
INSERT INTO commandes (client_id, montant, statut)
VALUES (42, 75.00, 'En attente');

COMMIT;  -- ✅ Validé !
```

**Transaction A (suite)** :

```
T3 (11:00:30) - Transaction A continue son rapport
```
```sql
-- Recompter pour vérification finale
SELECT COUNT(*) FROM commandes
WHERE client_id = 42 AND statut = 'En attente';
-- Résultat : 3 commandes (!!)  <-- Une ligne "fantôme" est apparue !

-- Recalculer le montant
SELECT SUM(montant) FROM commandes
WHERE client_id = 42 AND statut = 'En attente';
-- Résultat : 325€ (au lieu de 250€)

COMMIT;
```

#### Analyse de l'anomalie

Dans la **même transaction**, la même requête a retourné :
- Première fois : **2 lignes** (montant total 250€)
- Deuxième fois : **3 lignes** (montant total 325€)

Une ligne "fantôme" est apparue. C'est une **Phantom Read**.

#### Conséquences métier

Le rapport généré contient des **incohérences** :

```
Début du rapport : "Client 42 a 2 commandes en attente pour 250€"
Fin du rapport : "Total général : 3 commandes pour 325€"

🚨 Les chiffres ne correspondent pas !
```

**Autres problèmes causés** :
- 📊 Rapports analytiques incohérents
- 🧮 Totaux qui ne correspondent pas aux détails
- 📈 Graphiques basés sur des comptages instables
- ✅ Violations de contraintes d'agrégation

### Diagramme temporel de Phantom Read

```
Transaction A                             Transaction B
    |                                         |
    | BEGIN                                   |
    |                                         |
    | SELECT COUNT(*) = 2                     |
    | SELECT SUM() = 250€                     |
    |                                         |
    | [Génération rapport...]                 | BEGIN
    |                                         | INSERT nouvelle ligne
    |                                         | COMMIT ✅
    |                                         |
    | SELECT COUNT(*) = 3 (!)  <-- Phantom!   |
    | SELECT SUM() = 325€                     |
    |                                         |
    | COMMIT                                  |
    |                                         |
```

### Phantom Reads vs Non-Repeatable Reads

**Différence clé** :

| Anomalie | Ce qui change | Requête affectée |
|----------|---------------|------------------|
| **Non-Repeatable Read** | **Valeur** d'une ligne existante | SELECT sur une ligne spécifique (WHERE id = ...) |
| **Phantom Read** | **Nombre** de lignes | SELECT avec agrégation (COUNT, SUM) ou plage (WHERE prix > ...) |

**Exemple visuel** :

```
Non-Repeatable Read:
T1: SELECT prix WHERE id=1 → 100€
T2: (autre transaction UPDATE)
T3: SELECT prix WHERE id=1 → 150€  (même ligne, valeur différente)

Phantom Read:
T1: SELECT COUNT(*) WHERE statut='En attente' → 5 lignes
T2: (autre transaction INSERT)
T3: SELECT COUNT(*) WHERE statut='En attente' → 6 lignes  (nouvelle ligne)
```

### Comment prévenir les Phantom Reads

#### Solution 1 : Repeatable Read (PostgreSQL)

PostgreSQL est plus strict que le standard ANSI : **Repeatable Read empêche aussi les Phantom Reads**.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Première requête (snapshot figé)
SELECT COUNT(*) FROM commandes WHERE statut = 'En attente';
-- Résultat : 5

-- [Transaction B insère une nouvelle commande et commit]

-- Deuxième requête (même snapshot)
SELECT COUNT(*) FROM commandes WHERE statut = 'En attente';
-- Résultat : TOUJOURS 5 ✅ (pas de phantom)

COMMIT;
```

#### Solution 2 : Serializable

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*) FROM commandes WHERE statut = 'En attente';
-- Résultat : 5

-- Les insertions concurrentes sont détectées
-- La transaction peut échouer avec une erreur de sérialisation

COMMIT;
```

#### Solution 3 : Verrouillage de plage (limité)

Avec une clé primaire précise :

```sql
BEGIN;

-- Verrouiller toutes les lignes correspondantes
SELECT * FROM commandes
WHERE client_id = 42 AND statut = 'En attente'
FOR UPDATE;

-- Empêche les modifications, mais PAS les insertions de nouvelles lignes
-- Donc ne prévient pas complètement les Phantom Reads

COMMIT;
```

---

## 4. Lost Update (Mise à jour perdue)

### Définition

Une **Lost Update** se produit lorsque deux transactions lisent la même valeur, la modifient indépendamment, et la dernière à écrire **écrase silencieusement** la modification de la première.

**Analogie** : Deux personnes éditent le même document Word en même temps sans le savoir. Les deux sauvegardent. La personne qui sauvegarde en dernier écrase le travail de l'autre !

### Exemple détaillé : Le compteur classique

#### Contexte

Un système de likes sur les réseaux sociaux.

**État initial** :
```sql
-- Table posts
id | titre         | nb_likes
---|---------------|----------
1  | "Mon voyage"  | 100
```

#### Scénario avec Lost Update

**Transaction A (Alice aime le post)** :

```
T1 (12:00:00)
```
```sql
BEGIN;

-- Alice lit le nombre actuel de likes
SELECT nb_likes FROM posts WHERE id = 1;
-- Résultat : 100

-- [Alice fait un traitement dans son application]
-- Elle calcule : nouveau_nb = 100 + 1 = 101
```

**Transaction B (Bob aime aussi le post, EN PARALLÈLE)** :

```
T2 (12:00:01) - Presque en même temps
```
```sql
BEGIN;

-- Bob lit aussi le nombre de likes
SELECT nb_likes FROM posts WHERE id = 1;
-- Résultat : 100 (même valeur qu'Alice !)

-- Bob calcule : nouveau_nb = 100 + 1 = 101
```

**Transaction A (suite)** :

```
T3 (12:00:02)
```
```sql
-- Alice écrit sa nouvelle valeur
UPDATE posts SET nb_likes = 101 WHERE id = 1;

COMMIT;  -- ✅
```

**Transaction B (suite)** :

```
T4 (12:00:03)
```
```sql
-- Bob écrit aussi sa valeur (basée sur l'ancienne lecture)
UPDATE posts SET nb_likes = 101 WHERE id = 1;

COMMIT;  -- ✅
```

**État final** :
```sql
SELECT nb_likes FROM posts WHERE id = 1;
-- Résultat : 101
-- 🚨 MAUVAIS ! Devrait être 102 (Alice + Bob)
```

#### Analyse de l'anomalie

Les deux transactions ont réussi, mais **une mise à jour a été perdue** :
- Alice a ajouté +1 : 100 → 101 ✅
- Bob a ajouté +1 : mais a écrasé le travail d'Alice → 101 (au lieu de 102) ❌

C'est un **Lost Update** : la modification d'Alice est "perdue".

### Pattern dangereux : Read-Modify-Write

```
Lire une valeur
    ↓
Calculer dans l'application
    ↓
Écrire la nouvelle valeur
```

Ce pattern est **dangereux** en Read Committed car deux transactions peuvent lire la même valeur initiale.

### Solutions pour prévenir Lost Update

#### Solution 1 : Modification atomique (RECOMMANDÉE)

```sql
-- Au lieu de Read-Modify-Write, faire l'opération en SQL
BEGIN;

UPDATE posts SET nb_likes = nb_likes + 1 WHERE id = 1;
-- PostgreSQL gère atomiquement l'incrémentation
-- Pas de risque de Lost Update

COMMIT;
```

**Avantage** : Simple, performant, pas de risque.

#### Solution 2 : Repeatable Read ou Serializable

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT nb_likes FROM posts WHERE id = 1;  -- 100

-- [Calcul dans l'application : 101]

UPDATE posts SET nb_likes = 101 WHERE id = 1;
-- Si une autre transaction a modifié la ligne, ERREUR !
-- ERROR: could not serialize access due to concurrent update

COMMIT;
```

**Avantage** : Détection automatique des conflits.
**Inconvénient** : Nécessite une logique de retry.

#### Solution 3 : Verrouillage optimiste (version column)

```sql
-- Ajouter une colonne de version
ALTER TABLE posts ADD COLUMN version INTEGER DEFAULT 0;

-- Transaction A
BEGIN;
SELECT nb_likes, version FROM posts WHERE id = 1;
-- Résultat : nb_likes=100, version=5

UPDATE posts
SET nb_likes = 101, version = version + 1
WHERE id = 1 AND version = 5;
-- UPDATE 1 (succès si version toujours = 5)

COMMIT;

-- Transaction B (en parallèle)
BEGIN;
SELECT nb_likes, version FROM posts WHERE id = 1;
-- Résultat : nb_likes=100, version=5

UPDATE posts
SET nb_likes = 101, version = version + 1
WHERE id = 1 AND version = 5;
-- UPDATE 0 (échec ! Version a changé)
-- L'application détecte et réessaie

ROLLBACK;
```

**Avantage** : Contrôle fin, fonctionne en Read Committed.
**Inconvénient** : Plus de code applicatif.

#### Solution 4 : Verrouillage pessimiste (FOR UPDATE)

```sql
BEGIN;

SELECT nb_likes FROM posts WHERE id = 1 FOR UPDATE;
-- Verrouille la ligne immédiatement
-- Les autres transactions doivent attendre

-- [Calcul]

UPDATE posts SET nb_likes = 101 WHERE id = 1;

COMMIT;
```

**Avantage** : Prévention garantie.
**Inconvénient** : Contention, autres transactions bloquées.

---

## 5. Write Skew (Biais d'écriture)

### Définition

Un **Write Skew** se produit lorsque deux transactions lisent les **mêmes données**, prennent des **décisions contradictoires** basées sur ces lectures, et écrivent ensuite des modifications qui, combinées, **violent une contrainte métier**.

**Analogie** : Deux gardiens d'un musée consultent le planning : ils voient tous les deux qu'il y a 2 gardiens de service. Chacun décide qu'il peut partir (puisqu'il en restera 1). Résultat : le musée se retrouve sans gardien !

### Exemple détaillé : Les médecins de garde

#### Contexte

Un hôpital doit **toujours** avoir au moins 1 médecin de garde. Contrainte : `COUNT(*) WHERE en_service = true >= 1`

**État initial** :
```sql
-- Table medecins_garde
id | nom    | en_service
---|--------|------------
1  | Alice  | true
2  | Bob    | true
```

#### Scénario avec Write Skew (Repeatable Read ne suffit pas !)

**Transaction A (Alice veut partir)** :

```
T1 (14:00:00)
```
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Vérifier qu'il y a assez de médecins
SELECT COUNT(*) FROM medecins_garde WHERE en_service = true;
-- Résultat : 2 ✅

-- OK, Alice peut partir (il restera Bob)
UPDATE medecins_garde SET en_service = false WHERE id = 1;

-- Pas encore commité
```

**Transaction B (Bob veut partir, EN PARALLÈLE)** :

```
T2 (14:00:01)
```
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Vérifier qu'il y a assez de médecins
SELECT COUNT(*) FROM medecins_garde WHERE en_service = true;
-- Résultat : 2 ✅ (snapshot avant les modifications d'Alice)

-- OK, Bob peut partir (il restera Alice... pense-t-il)
UPDATE medecins_garde SET en_service = false WHERE id = 2;

-- Pas encore commité
```

**Validation** :

```
T3 (14:00:02)
```
```sql
-- Transaction A
COMMIT;  -- ✅ Succès

-- Transaction B
COMMIT;  -- ✅ Succès aussi ! (Repeatable Read ne détecte pas ce conflit)
```

**État final catastrophique** :
```sql
SELECT COUNT(*) FROM medecins_garde WHERE en_service = true;
-- Résultat : 0 (!!)  🚨 Violation de la contrainte métier !
```

#### Analyse de l'anomalie

- Les deux transactions ont lu : `COUNT(*) = 2`
- Les deux ont conclu : "Je peux partir"
- Les deux ont modifié des **lignes différentes** (Alice: ligne 1, Bob: ligne 2)
- PostgreSQL Repeatable Read ne détecte **pas** ce type de conflit car les écritures sont sur des lignes différentes
- Résultat : contrainte métier violée

C'est un **Write Skew** : les écritures combinées créent une incohérence.

### Pourquoi Repeatable Read ne suffit pas

Repeatable Read détecte les conflits **sur la même ligne** :

```
Transaction A : UPDATE ligne 1
Transaction B : UPDATE ligne 1
→ Conflit détecté ! ✅
```

Mais **pas** les conflits **entre lignes différentes** :

```
Transaction A : UPDATE ligne 1
Transaction B : UPDATE ligne 2
→ Pas de conflit détecté ❌ (même si la combinaison viole une règle)
```

### Solution : Serializable

```sql
-- Transaction A
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*) FROM medecins_garde WHERE en_service = true;
-- Résultat : 2

UPDATE medecins_garde SET en_service = false WHERE id = 1;

COMMIT;  -- ✅ Succès

-- Transaction B
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*) FROM medecins_garde WHERE en_service = true;
-- Résultat : 2

UPDATE medecins_garde SET en_service = false WHERE id = 2;

COMMIT;
-- ❌ ERREUR: could not serialize access due to read/write dependencies
-- among transactions
```

PostgreSQL détecte que la combinaison des deux transactions créerait une incohérence et rejette la seconde.

### Autre exemple : Double réservation de place

**Règle** : Une place ne peut être réservée qu'une fois.

```sql
-- État initial : Place 42 disponible
disponible = true

-- Transaction A
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT disponible FROM places WHERE id = 42;  -- true
UPDATE places SET disponible = false, client_id = 'Alice' WHERE id = 42;
COMMIT;  -- ✅

-- Transaction B (parallèle)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT disponible FROM places WHERE id = 42;  -- true (snapshot ancien)
UPDATE places SET disponible = false, client_id = 'Bob' WHERE id = 42;
COMMIT;  -- ❌ Erreur (modification concurrente détectée)
```

Ce cas **est détecté** par Repeatable Read car les deux transactions modifient **la même ligne**.

**Différence avec Write Skew** : Write Skew implique des écritures sur **lignes différentes** avec une contrainte d'agrégation.

---

## 6. Read Skew (Biais de lecture)

### Définition

Un **Read Skew** se produit lorsqu'une transaction lit plusieurs lignes et voit un état **incohérent** car d'autres transactions ont modifié certaines de ces lignes entre les lectures.

**Analogie** : Vous prenez en photo le panneau d'affichage d'un aéroport. Pendant que vous photographiez, les panneaux changent. Résultat : votre photo montre des infos incohérentes (vols arrivés + vols pas encore partis).

### Exemple détaillé : Transfert bancaire

#### Contexte

Un virement entre deux comptes.

**État initial** :
```sql
-- Table comptes
id | proprietaire | solde
---|--------------|-------
A  | Alice        | 1000.00
B  | Bob          | 500.00
-- Total : 1500€
```

#### Scénario avec Read Skew (Read Committed)

**Transaction T (Transfert 300€ de A vers B)** :

```
T1 (15:00:00)
```
```sql
BEGIN;

UPDATE comptes SET solde = solde - 300 WHERE id = 'A';
-- Compte A : 700€ (commité)

-- [Petit délai]
```

**Transaction R (Audit - calcul du total, EN PARALLÈLE)** :

```
T2 (15:00:05) - Entre les deux UPDATE du transfert
```
```sql
BEGIN;  -- Read Committed

-- Lire le solde de A
SELECT solde FROM comptes WHERE id = 'A';
-- Résultat : 700€ (voit la modification de T !)
```

**Transaction T (suite)** :

```
T3 (15:00:07)
```
```sql
-- Terminer le transfert
UPDATE comptes SET solde = solde + 300 WHERE id = 'B';
-- Compte B : 800€

COMMIT;  -- Transfert terminé
```

**Transaction R (suite)** :

```
T4 (15:00:10)
```
```sql
-- Lire le solde de B
SELECT solde FROM comptes WHERE id = 'B';
-- Résultat : 800€ (voit aussi la modification !)

-- Calculer le total
SELECT SUM(solde) FROM comptes WHERE id IN ('A', 'B');
-- Résultat : 1500€... MAIS !

COMMIT;
```

**Analyse** :

Transaction R a vu :
- Compte A après le débit : 700€ ✅
- Compte B après le crédit : 800€ ✅
- Total : 1500€ ✅

Mais si elle avait calculé le total **pendant** sa transaction :

```sql
-- Transaction R au moment T2 (entre les deux UPDATE)
SELECT SUM(solde) FROM comptes WHERE id IN ('A', 'B');
-- Aurait vu : 700€ + 500€ = 1200€ (!!)  🚨
-- Les 300€ auraient "disparu" temporairement !
```

C'est un **Read Skew** : une vue incohérente de l'état de la base.

### Comment prévenir Read Skew

#### Solution : Repeatable Read ou Serializable

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Toutes les lectures utilisent le MÊME snapshot
SELECT solde FROM comptes WHERE id = 'A';  -- 1000€
-- [Transfert se produit dans une autre transaction]
SELECT solde FROM comptes WHERE id = 'B';  -- 500€
SELECT SUM(solde) FROM comptes WHERE id IN ('A', 'B');  -- 1500€

-- Vue cohérente garantie ! ✅

COMMIT;
```

---

## Résumé des anomalies et solutions

### Tableau comparatif

| Anomalie | Description courte | Ce qui change | Solution |
|----------|-------------------|---------------|----------|
| **Dirty Read** | Lire des données non commitées | Données temporaires | Read Committed+ |
| **Non-Repeatable Read** | Même ligne, valeur différente | Valeur d'une ligne | Repeatable Read+ |
| **Phantom Read** | Même requête, nb lignes différent | Nombre de lignes | Repeatable Read+ (PG) |
| **Lost Update** | Modification écrasée | Deux écritures en conflit | Atomicité SQL ou RR+ |
| **Write Skew** | Écritures contradictoires | État global incohérent | Serializable |
| **Read Skew** | Lectures incohérentes entre elles | Vue fragmentée | Repeatable Read+ |

### Gravité des anomalies

| Anomalie | Gravité | Fréquence | Difficulté de détection |
|----------|---------|-----------|------------------------|
| **Dirty Read** | 🔴 Critique | Rare (PG empêche) | Facile |
| **Non-Repeatable Read** | 🟡 Modérée | Fréquente | Moyenne |
| **Phantom Read** | 🟡 Modérée | Fréquente | Moyenne |
| **Lost Update** | 🔴 Critique | Fréquente | Difficile |
| **Write Skew** | 🔴 Critique | Rare | Très difficile |
| **Read Skew** | 🟡 Modérée | Moyenne | Moyenne |

---

## Stratégies de prévention par type d'application

### Application CRUD simple (Blog, CMS)

```
Anomalies acceptables : Non-Repeatable Read, Phantom Read, Read Skew
Niveau : Read Committed (défaut)
Justification : Performance et simplicité
```

### E-commerce (Panier, Stock)

```
Anomalies critiques : Lost Update (stock)
Niveau : Read Committed + UPDATE atomique
```

```sql
-- Bon pattern pour gérer le stock
BEGIN;
UPDATE produits
SET stock = stock - quantite_achetee
WHERE id = produit_id AND stock >= quantite_achetee;

IF NOT FOUND THEN
    RAISE EXCEPTION 'Stock insuffisant';
END IF;
COMMIT;
```

### Système bancaire

```
Anomalies critiques : TOUTES
Niveau : Serializable
Justification : Argent en jeu, cohérence absolue requise
```

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Vérifier solde
SELECT solde FROM comptes WHERE id = source_id;

-- Effectuer transfert
UPDATE comptes SET solde = solde - montant WHERE id = source_id;
UPDATE comptes SET solde = solde + montant WHERE id = dest_id;

COMMIT;
-- Retry automatique si erreur de sérialisation
```

### Rapports et Analytics

```
Anomalies critiques : Phantom Read, Read Skew
Niveau : Repeatable Read
Justification : Vue cohérente nécessaire
```

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Toutes les requêtes voient le même snapshot
SELECT SUM(ventes) FROM commandes WHERE mois = 'Janvier';
SELECT AVG(panier) FROM commandes WHERE mois = 'Janvier';
SELECT COUNT(*) FROM commandes WHERE mois = 'Janvier';

COMMIT;
```

### Système de réservation

```
Anomalies critiques : Write Skew, Lost Update
Niveau : Serializable
```

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Vérifier disponibilité
SELECT COUNT(*) FROM reservations WHERE place_id = 42 AND date = '2024-01-15';

-- Réserver
INSERT INTO reservations (place_id, client_id, date)
VALUES (42, 123, '2024-01-15');

COMMIT;
```

---

## Checklist de diagnostic des anomalies

### Symptômes d'anomalies dans votre application

**Dirty Read** :
- ❌ N/A dans PostgreSQL (automatiquement empêché)

**Non-Repeatable Read** :
- ✋ Les utilisateurs rapportent des prix qui changent pendant la navigation
- ✋ Les totaux calculés ne correspondent pas aux détails affichés
- ✋ Les validations échouent avec des messages "données modifiées"

**Phantom Read** :
- ✋ Les rapports montrent des comptages incohérents
- ✋ Les listes d'éléments changent de longueur sans action utilisateur
- ✋ Les agrégations (SUM, COUNT) varient entre les requêtes

**Lost Update** :
- ✋ Les compteurs (likes, vues, stocks) sont incorrects
- ✋ Les modifications utilisateur "disparaissent"
- ✋ Les inventaires ne correspondent pas aux ventes

**Write Skew** :
- ✋ Les contraintes métier sont violées (ex: pas de gardien)
- ✋ Les limites (quotas, capacités) sont dépassées
- ✋ Les allocations de ressources créent des conflits

**Read Skew** :
- ✋ Les totaux ne correspondent pas à la somme des parties
- ✋ Les données liées semblent désynchronisées
- ✋ Les exports contiennent des incohérences

---

## Bonnes pratiques générales

### 1. Comprendre les besoins métier

Avant de choisir un niveau d'isolation, posez-vous ces questions :

- ✅ Quelle est la gravité d'une lecture incohérente ?
- ✅ Les utilisateurs peuvent-ils tolérer des données légèrement obsolètes ?
- ✅ Y a-t-il des contraintes métier qui doivent être absolument respectées ?
- ✅ Quelle est la fréquence de modification des données ?

### 2. Privilégier les opérations atomiques

```sql
-- ❌ MAUVAIS : Read-Modify-Write
SELECT valeur FROM table WHERE id = 1;
-- [Calcul dans l'application]
UPDATE table SET valeur = nouvelle_valeur WHERE id = 1;

-- ✅ BON : Opération atomique
UPDATE table SET valeur = valeur + increment WHERE id = 1;
```

### 3. Utiliser les contraintes de base de données

```sql
-- Contrainte CHECK pour empêcher les violations
ALTER TABLE comptes ADD CONSTRAINT solde_positif CHECK (solde >= 0);

-- Contrainte UNIQUE pour empêcher les doublons
ALTER TABLE reservations ADD CONSTRAINT place_date_unique
UNIQUE (place_id, date);
```

### 4. Implémenter une logique de retry

Pour Repeatable Read et Serializable, implémentez **toujours** un mécanisme de retry :

```python
def execute_with_retry(operation, max_retries=3):
    for attempt in range(max_retries):
        try:
            return operation()
        except SerializationError:
            if attempt == max_retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))
```

### 5. Surveiller les métriques

```sql
-- Taux de rollback (indicateur de conflits)
SELECT
    datname,
    xact_commit,
    xact_rollback,
    ROUND(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2) AS rollback_pct
FROM pg_stat_database
WHERE datname = current_database();
```

Un taux élevé peut indiquer :
- Niveau d'isolation trop strict
- Contentions sur les données
- Transactions trop longues

---

## Conclusion

Les anomalies transactionnelles sont des comportements indésirables qui peuvent survenir lorsque plusieurs transactions accèdent aux mêmes données simultanément. Comprendre ces anomalies est essentiel pour :

- ✅ Choisir le bon niveau d'isolation
- ✅ Concevoir des schémas de données robustes
- ✅ Éviter les bugs subtils et difficiles à reproduire
- ✅ Garantir l'intégrité des données de votre application

**Points clés** :

1. **Dirty Read** : Jamais dans PostgreSQL (MVCC l'empêche)
2. **Non-Repeatable Read** : Fréquent en Read Committed, empêché par Repeatable Read
3. **Phantom Read** : Également empêché par Repeatable Read dans PostgreSQL
4. **Lost Update** : Dangereux, utilisez des opérations atomiques
5. **Write Skew** : Subtil, nécessite Serializable
6. **Read Skew** : Vue incohérente, utilisez Repeatable Read

**Règle d'or** : Commencez par Read Committed, puis montez en isolation seulement si votre logique métier le justifie, en acceptant le coût (complexité, retries, performance).

Dans la prochaine section (12.5), nous explorerons la **gestion des verrous (Locks)** et comment PostgreSQL coordonne l'accès concurrent aux données au niveau le plus bas.

---

**Points clés à retenir :**

- 🔑 6 anomalies principales : Dirty Read, Non-Repeatable Read, Phantom Read, Lost Update, Write Skew, Read Skew
- 🔑 PostgreSQL empêche automatiquement les Dirty Reads
- 🔑 Lost Update est l'anomalie la plus courante → utilisez UPDATE atomique
- 🔑 Write Skew nécessite Serializable pour être détecté
- 🔑 Chaque anomalie a un impact métier spécifique
- 🔑 Le choix du niveau d'isolation dépend des anomalies que vous devez absolument prévenir
- 🔑 Les opérations atomiques SQL sont toujours préférables au Read-Modify-Write

⏭️ [Gestion des verrous (Locks) : Types et Deadlocks](/12-concurrence-et-transactions/05-gestion-des-verrous.md)
