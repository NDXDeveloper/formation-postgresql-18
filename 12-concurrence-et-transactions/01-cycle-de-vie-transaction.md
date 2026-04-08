🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.1. Cycle de vie d'une transaction (BEGIN, COMMIT, ROLLBACK, SAVEPOINT)

## Introduction aux transactions

Une **transaction** est un concept fondamental dans les bases de données. Elle représente une unité de travail complète qui regroupe une ou plusieurs opérations SQL. Le principe essentiel d'une transaction est simple : **toutes les opérations réussissent ensemble, ou aucune ne réussit**.

Imaginez que vous effectuez un virement bancaire entre deux comptes. Cette opération nécessite deux étapes :
1. Retirer de l'argent du compte A  
2. Ajouter cet argent au compte B

Si la première étape réussit mais que la seconde échoue, l'argent disparaît dans la nature ! C'est là qu'intervient la transaction : elle garantit que soit les deux opérations s'effectuent, soit aucune.

---

## Les états d'une transaction

Une transaction dans PostgreSQL passe par plusieurs états durant son cycle de vie :

### 1. État initial (Mode autocommit)

Par défaut, PostgreSQL fonctionne en **mode autocommit** : chaque instruction SQL est automatiquement une transaction complète qui est immédiatement validée (committée).

```sql
-- Cette instruction est automatiquement une transaction complète
INSERT INTO utilisateurs (nom, email) VALUES ('Alice', 'alice@example.com');
-- Elle est immédiatement enregistrée dans la base de données
```

### 2. État actif (Transaction en cours)

Lorsque vous démarrez explicitement une transaction avec `BEGIN`, PostgreSQL entre dans un mode où les modifications ne sont pas immédiatement définitives.

### 3. État terminé

Une transaction se termine de deux façons possibles :
- **Validation (COMMIT)** : Les modifications sont enregistrées définitivement  
- **Annulation (ROLLBACK)** : Les modifications sont complètement abandonnées

---

## Les commandes du cycle de vie

### BEGIN - Démarrer une transaction

La commande `BEGIN` (ou `START TRANSACTION`) marque le début d'une transaction explicite.

```sql
BEGIN;
-- À partir d'ici, toutes les opérations font partie de la transaction
```

**Variantes équivalentes :**
```sql
START TRANSACTION;  -- Équivalent à BEGIN  
BEGIN WORK;         -- Autre variante  
```

**Analogie :** `BEGIN` est comme ouvrir un brouillon dans un éditeur de texte. Vous pouvez faire toutes les modifications que vous voulez, mais elles ne sont pas encore "sauvegardées" de manière permanente.

### COMMIT - Valider définitivement

La commande `COMMIT` valide toutes les modifications effectuées dans la transaction et les rend permanentes dans la base de données.

```sql
BEGIN;

INSERT INTO produits (nom, prix) VALUES ('Ordinateur', 999.99);  
UPDATE stock SET quantite = quantite - 1 WHERE produit = 'Ordinateur';  

COMMIT;  -- Toutes les modifications sont maintenant permanentes
```

**Variantes équivalentes :**
```sql
COMMIT WORK;        -- Équivalent à COMMIT  
END TRANSACTION;    -- Autre variante  
END;               -- Forme courte  
```

**Analogie :** `COMMIT` est comme cliquer sur "Sauvegarder" dans votre éditeur. Vos modifications deviennent définitives.

**Important :** Après un `COMMIT`, il est impossible de revenir en arrière. Les données sont écrites définitivement dans la base.

### ROLLBACK - Annuler les modifications

La commande `ROLLBACK` annule toutes les modifications effectuées dans la transaction en cours et ramène la base de données à son état avant le `BEGIN`.

```sql
BEGIN;

DELETE FROM commandes WHERE id = 123;
-- Oups, erreur ! Je voulais supprimer la commande 124, pas 123

ROLLBACK;  -- Annule la suppression, la commande 123 est toujours là
```

**Variantes équivalentes :**
```sql
ROLLBACK WORK;     -- Équivalent à ROLLBACK  
ABORT;             -- Autre variante  
```

**Analogie :** `ROLLBACK` est comme cliquer sur "Annuler" ou "Ne pas sauvegarder" dans votre éditeur. Toutes vos modifications depuis l'ouverture du brouillon sont perdues.

**Cas d'utilisation typiques du ROLLBACK :**
- Détection d'une erreur dans votre logique métier
- Échec d'une opération critique dans une séquence
- Tests et expérimentations (essayer des requêtes sans modifier réellement les données)
- Gestion d'erreurs dans le code applicatif

### SAVEPOINT - Créer des points de sauvegarde

Un **SAVEPOINT** (point de sauvegarde) est un marqueur à l'intérieur d'une transaction qui permet d'annuler partiellement les modifications, sans abandonner toute la transaction.

**Syntaxe :**
```sql
SAVEPOINT nom_du_point;
```

**Concept :** Imaginez que vous écrivez un long document. Au lieu de tout recommencer si vous faites une erreur à la fin, vous pouvez placer des "marque-pages" et revenir uniquement à l'un de ces points.

#### Exemple complet avec SAVEPOINT

```sql
BEGIN;

-- Opération 1 : Créer un utilisateur
INSERT INTO utilisateurs (nom, email)  
VALUES ('Bob', 'bob@example.com');  

-- Créer un point de sauvegarde après cette opération
SAVEPOINT after_user_creation;

-- Opération 2 : Créer l'adresse de l'utilisateur
INSERT INTO adresses (user_id, ville, rue)  
VALUES (123, 'Paris', '10 rue de la Paix');  

-- Créer un second point de sauvegarde
SAVEPOINT after_address_creation;

-- Opération 3 : Créer les préférences (oups, erreur!)
INSERT INTO preferences (user_id, langue)  
VALUES (999, 'fr');  -- user_id incorrect !  

-- Annuler uniquement depuis le dernier savepoint
ROLLBACK TO SAVEPOINT after_address_creation;
-- L'insertion dans preferences est annulée
-- Mais l'utilisateur et son adresse sont toujours dans la transaction

-- Corriger l'erreur
INSERT INTO preferences (user_id, langue)  
VALUES (123, 'fr');  -- Maintenant c'est correct  

COMMIT;  -- Valider toutes les opérations (sauf celle qu'on a annulée)
```

#### ROLLBACK TO SAVEPOINT

La commande `ROLLBACK TO SAVEPOINT` permet de revenir à un point de sauvegarde spécifique :

```sql
ROLLBACK TO SAVEPOINT nom_du_point;
-- ou
ROLLBACK TO nom_du_point;  -- Forme courte
```

**Important :** Après un `ROLLBACK TO SAVEPOINT`, le point de sauvegarde reste valide et vous pouvez y revenir à nouveau si nécessaire.

#### RELEASE SAVEPOINT

Si vous n'avez plus besoin d'un point de sauvegarde, vous pouvez le libérer :

```sql
RELEASE SAVEPOINT nom_du_point;
-- ou
RELEASE nom_du_point;  -- Forme courte
```

**Effet :** Le point de sauvegarde est supprimé, mais les modifications effectuées depuis ce point restent dans la transaction.

**Pourquoi libérer un savepoint ?**
- Optimisation des ressources (PostgreSQL n'a plus besoin de maintenir ce point)
- Clarté du code (indiquer qu'on ne reviendra plus en arrière)

---

## Schéma du cycle de vie complet

Voici le cycle de vie d'une transaction avec toutes ses possibilités :

```
[Mode autocommit]
       |
       | BEGIN / START TRANSACTION
       ↓
[Transaction active]
       |
       |→ Opération SQL 1
       |→ Opération SQL 2
       |→ SAVEPOINT point_1
       |→ Opération SQL 3
       |    |
       |    |→ (Erreur) → ROLLBACK TO SAVEPOINT point_1
       |                          ↓
       |                   Retour avant opération 3
       |
       |→ Opération SQL 4
       |
       |
       ↓
   Terminaison :
       |
       |→ COMMIT → [Modifications permanentes] → [Mode autocommit]
       |
       |→ ROLLBACK → [Modifications annulées] → [Mode autocommit]
```

---

## Exemples pratiques et cas d'usage

### Exemple 1 : Transaction simple (cas nominal)

```sql
-- Virement bancaire de 100€ du compte A vers le compte B
BEGIN;

UPDATE comptes SET solde = solde - 100 WHERE id = 'A';  
UPDATE comptes SET solde = solde + 100 WHERE id = 'B';  

COMMIT;  -- Les deux opérations sont validées ensemble
```

### Exemple 2 : Annulation en cas d'erreur

```sql
BEGIN;

DELETE FROM anciens_clients WHERE derniere_connexion < '2020-01-01';

-- Oups, je viens de voir que j'ai supprimé 10 000 clients au lieu de 100 !
-- SELECT COUNT(*) FROM anciens_clients;  -- Vérification

ROLLBACK;  -- Annuler immédiatement !
```

### Exemple 3 : Utilisation des savepoints pour une logique complexe

```sql
BEGIN;

-- Créer une commande
INSERT INTO commandes (client_id, date)  
VALUES (42, CURRENT_DATE)  
RETURNING id AS commande_id;  -- Supposons que ça retourne 100  

SAVEPOINT commande_creee;

-- Ajouter des produits à la commande
INSERT INTO lignes_commande (commande_id, produit_id, quantite)  
VALUES  
  (100, 1, 5),
  (100, 2, 3);

SAVEPOINT produits_ajoutes;

-- Appliquer une réduction (mais le code de promo est invalide !)
UPDATE commandes SET reduction = 20  
WHERE id = 100 AND code_promo = 'INVALID';  

-- Aucune ligne n'a été mise à jour, le code promo est invalide
-- On annule juste la tentative de réduction
ROLLBACK TO SAVEPOINT produits_ajoutes;

-- On continue sans réduction
COMMIT;  -- La commande et ses produits sont validés
```

### Exemple 4 : Test sans modification

Très utile pour tester des requêtes en production sans risque :

```sql
BEGIN;

-- Tester une requête de mise à jour massive
UPDATE utilisateurs SET statut = 'inactif'  
WHERE derniere_connexion < CURRENT_DATE - INTERVAL '1 year';  

-- Vérifier le nombre de lignes affectées
-- Si le résultat semble correct, vous pouvez COMMIT
-- Sinon, faites ROLLBACK

ROLLBACK;  -- On annule, c'était juste un test
```

---

## Comportements importants et pièges à éviter

### 1. Les transactions imbriquées n'existent pas vraiment

PostgreSQL ne supporte **pas les transactions imbriquées** au sens strict. Si vous exécutez `BEGIN` alors qu'une transaction est déjà active, PostgreSQL émet simplement un avertissement et ignore le second `BEGIN`.

```sql
BEGIN;
  BEGIN;  -- Avertissement : une transaction est déjà en cours
  -- ...
  COMMIT;  -- Ceci NE valide PAS la transaction externe !
COMMIT;  -- C'est ce COMMIT qui valide réellement
```

**Solution :** Utilisez des `SAVEPOINT` pour obtenir un comportement similaire aux transactions imbriquées.

### 2. Les erreurs annulent automatiquement la transaction en cours

Si une erreur SQL se produit dans une transaction, PostgreSQL entre en **état d'erreur** et refuse d'exécuter d'autres commandes jusqu'à ce que vous fassiez un `ROLLBACK`.

```sql
BEGIN;

INSERT INTO produits (nom, prix) VALUES ('Livre', 15.00);

-- Cette commande provoque une erreur (violation de contrainte)
INSERT INTO produits (id, nom, prix) VALUES (1, 'Stylo', 2.00);
-- ERREUR: la valeur 1 existe déjà

-- PostgreSQL est maintenant en "état d'erreur"
UPDATE produits SET prix = 20.00 WHERE nom = 'Livre';
-- ERREUR: la transaction actuelle est abandonnée,
-- les commandes sont ignorées jusqu'à la fin du bloc de transaction

ROLLBACK;  -- Obligatoire pour sortir de l'état d'erreur
```

**Important :** Dans le code applicatif, vous devez **toujours** gérer les erreurs et faire un `ROLLBACK` explicite en cas de problème.

### 3. Durée des transactions

Une transaction qui reste ouverte trop longtemps peut causer des problèmes :

- **Verrous maintenus** : Les lignes modifiées restent verrouillées, bloquant potentiellement d'autres utilisateurs  
- **Bloat de la base** : PostgreSQL ne peut pas nettoyer les anciennes versions des lignes  
- **Risque de wraparound** : Dans les cas extrêmes, problèmes avec les identifiants de transaction

**Bonne pratique :** Gardez vos transactions aussi courtes que possible. Faites-les, validez-les, et passez à autre chose.

### 4. ROLLBACK vs ROLLBACK TO SAVEPOINT

```sql
BEGIN;
  INSERT INTO table1 VALUES (1);
  SAVEPOINT sp1;
  INSERT INTO table2 VALUES (2);

  -- ROLLBACK TO sp1 : annule seulement l'INSERT dans table2
  -- La transaction reste active

  -- ROLLBACK : annule TOUT (table1 ET table2)
  -- La transaction est terminée
COMMIT;
```

### 5. Autocommit dans les outils clients

Certains outils (comme psql) ont un mode autocommit activé par défaut. Dans ce mode, chaque commande est automatiquement commitée. Pour les désactiver temporairement :

```sql
-- Dans psql
\set AUTOCOMMIT off

-- Maintenant vous devez explicitement COMMIT ou ROLLBACK
```

---

## Résumé et bonnes pratiques

### Résumé des commandes

| Commande | Action | Effet sur la transaction |
|----------|--------|--------------------------|
| `BEGIN` | Démarre une transaction | Entre en mode transaction explicite |
| `COMMIT` | Valide les modifications | Termine la transaction, modifications permanentes |
| `ROLLBACK` | Annule toutes les modifications | Termine la transaction, retour à l'état initial |
| `SAVEPOINT nom` | Crée un point de sauvegarde | Transaction reste active |
| `ROLLBACK TO SAVEPOINT nom` | Revient au point de sauvegarde | Transaction reste active |
| `RELEASE SAVEPOINT nom` | Supprime le point de sauvegarde | Transaction reste active |

### Bonnes pratiques

1. **Toujours gérer les erreurs** : Dans le code applicatif, capturez les exceptions et faites un `ROLLBACK` en cas d'erreur

2. **Transactions courtes** : Ne laissez pas une transaction ouverte pendant des opérations longues (appels réseau, attente utilisateur, etc.)

3. **Utiliser les savepoints** : Pour les opérations complexes avec plusieurs étapes, les savepoints permettent une gestion d'erreur granulaire

4. **Tester sans risque** : Utilisez `BEGIN...ROLLBACK` pour tester des requêtes sans modifier réellement les données

5. **Explicite plutôt qu'implicite** : Même si PostgreSQL a un mode autocommit, utilisez `BEGIN...COMMIT` explicitement pour les opérations critiques

6. **Documentation** : Commentez vos transactions complexes pour expliquer la logique métier

### Quand utiliser des SAVEPOINT ?

Les savepoints sont utiles dans ces situations :

- **Opérations en plusieurs étapes** où certaines étapes peuvent échouer sans invalider toute l'opération  
- **Boucles et traitements par lots** pour pouvoir continuer en cas d'erreur sur un élément  
- **Logique métier complexe** avec plusieurs chemins de décision  
- **Intégration avec des langages procéduraux** (PL/pgSQL) pour une gestion d'erreur fine

---

## Conclusion

Le cycle de vie d'une transaction est un concept fondamental pour garantir la cohérence et l'intégrité des données dans PostgreSQL. Comprendre `BEGIN`, `COMMIT`, `ROLLBACK` et `SAVEPOINT` vous permet de :

- Regrouper des opérations logiquement liées
- Garantir que les données restent cohérentes même en cas d'erreur
- Annuler des modifications en cas de problème
- Gérer des scénarios complexes avec des points de sauvegarde

Dans le prochain chapitre (12.2), nous approfondirons le mécanisme **MVCC (Multiversion Concurrency Control)**, qui est le cœur technologique permettant à PostgreSQL de gérer les transactions de manière efficace et concurrente.

---

**Points clés à retenir :**

- ✅ Une transaction = une unité de travail atomique (tout ou rien)  
- ✅ `BEGIN` démarre, `COMMIT` valide, `ROLLBACK` annule  
- ✅ Les `SAVEPOINT` permettent des annulations partielles  
- ✅ Gardez vos transactions courtes et gérez toujours les erreurs  
- ✅ PostgreSQL entre en "état d'erreur" après une erreur SQL dans une transaction

⏭️ [MVCC (Multiversion Concurrency Control) : Le cœur de PostgreSQL](/12-concurrence-et-transactions/02-mvcc-multiversion-concurrency.md)
