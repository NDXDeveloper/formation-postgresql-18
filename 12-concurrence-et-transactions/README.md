🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12. Concurrence et Transactions

## Introduction : Le cœur battant de PostgreSQL

Imaginez une bibliothèque où des centaines de personnes travaillent simultanément : certaines lisent des livres, d'autres en ajoutent de nouveaux, d'autres encore mettent à jour les fiches de prêt. Comment garantir que personne ne lise des informations erronées pendant qu'un autre les modifie ? Comment éviter que deux personnes n'empruntent le même exemplaire en même temps ? Comment s'assurer que toutes les opérations restent cohérentes malgré le chaos apparent ?

C'est exactement le défi que relève PostgreSQL dans le monde des bases de données. Chaque seconde, des **dizaines, centaines, voire milliers** d'utilisateurs et d'applications accèdent simultanément aux mêmes données. C'est ce qu'on appelle la **concurrence**.

Ce chapitre explore **l'un des aspects les plus fondamentaux et fascinants** de PostgreSQL : comment il gère des milliers d'opérations simultanées tout en garantissant que vos données restent **cohérentes**, **fiables** et **intègres**.

---

## Qu'est-ce que la concurrence ?

### Définition simple

La **concurrence** (ou *concurrency* en anglais) désigne la capacité d'un système à gérer **plusieurs opérations en même temps** sur les mêmes données, sans créer de conflits ou d'incohérences.

**Exemple du monde réel** :

Imaginez un site e-commerce pendant les soldes :
- **10:00:00.001** : Alice consulte le produit "Laptop XYZ" → Stock : 5 exemplaires  
- **10:00:00.005** : Bob consulte aussi "Laptop XYZ" → Stock : 5 exemplaires  
- **10:00:00.010** : Alice achète 1 exemplaire → Stock : 4  
- **10:00:00.015** : Bob achète 1 exemplaire → Stock : 3  
- **10:00:00.020** : Claire achète 2 exemplaires → Stock : 1  
- **10:00:00.025** : David achète 1 exemplaire → Stock : 0

Tout cela en **25 millisecondes** ! Et pourtant, le stock reste cohérent. C'est la magie de la gestion de la concurrence.

### Sans gestion de la concurrence : Le chaos

Que se passerait-il sans un système de gestion de la concurrence ?

**Scénario catastrophique** :

```
Alice lit : stock = 5  
Bob lit : stock = 5 (en même temps qu'Alice)  
Alice achète 1 → calcule : nouveau stock = 5 - 1 = 4  
Bob achète 1 → calcule : nouveau stock = 5 - 1 = 4 (basé sur l'ancienne valeur !)  
Alice écrit : stock = 4  
Bob écrit : stock = 4 (ÉCRASE la mise à jour d'Alice !)  

Résultat : stock = 4 (MAUVAIS ! Devrait être 3)
→ Un exemplaire vendu a "disparu" dans les comptes
```

C'est ce qu'on appelle une **anomalie de concurrence** ou un **Lost Update** (mise à jour perdue).

### Avec PostgreSQL : Cohérence garantie

PostgreSQL utilise des mécanismes sophistiqués pour **garantir** que ces situations ne se produisent jamais :

```
Alice : BEGIN → UPDATE stock = stock - 1 → COMMIT  
Bob : BEGIN → UPDATE stock = stock - 1 → COMMIT (attend qu'Alice finisse)  

Résultat : stock = 3 ✅ (correct)
```

---

## Qu'est-ce qu'une transaction ?

### Définition simple

Une **transaction** est une **unité de travail complète** qui regroupe une ou plusieurs opérations SQL. Le principe fondamental est simple :

> **Toutes les opérations de la transaction réussissent ensemble, ou aucune ne réussit.**

**Analogie** : Une transaction, c'est comme une recette de cuisine complète.

Si vous préparez un gâteau :
1. Mélanger les ingrédients  
2. Verser dans un moule  
3. Mettre au four

Soit vous faites **toutes** les étapes et vous obtenez un gâteau, soit vous abandonnez et vous **n'avez rien** (pas de demi-gâteau).

### Exemple concret : Virement bancaire

Le cas classique pour comprendre les transactions :

**Opération métier** : Transférer 100€ du compte A vers le compte B

**Sans transaction** (DANGEREUX ❌) :
```sql
UPDATE comptes SET solde = solde - 100 WHERE id = 'A';  -- Débiter A
-- [PANNE DE COURANT !]
UPDATE comptes SET solde = solde + 100 WHERE id = 'B';  -- Créditer B (jamais exécuté)

Résultat : 100€ ont disparu ! 💸
```

**Avec transaction** (SÛR ✅) :
```sql
BEGIN;  -- Début de la transaction

UPDATE comptes SET solde = solde - 100 WHERE id = 'A';  -- Débiter A
-- [PANNE DE COURANT !]
UPDATE comptes SET solde = solde + 100 WHERE id = 'B';  -- Créditer B

COMMIT;  -- Valider la transaction
-- Si panne avant COMMIT → PostgreSQL annule TOUT automatiquement
```

Si une erreur se produit (panne, bug, violation de contrainte), PostgreSQL **annule automatiquement** toutes les opérations de la transaction. C'est le principe du **"tout ou rien"** (atomicité).

---

## Les propriétés ACID : Le contrat de PostgreSQL

PostgreSQL garantit quatre propriétés fondamentales pour chaque transaction, résumées par l'acronyme **ACID** :

### A - Atomicity (Atomicité)

**Principe** : Une transaction est **indivisible**. Soit toutes ses opérations réussissent, soit aucune.

**Analogie** : Un atome ne peut pas être coupé en deux (étymologie : *a-tomos* = indivisible).

**Exemple** :
```sql
BEGIN;  
INSERT INTO commandes (client_id, montant) VALUES (42, 100);  
INSERT INTO lignes_commande (commande_id, produit_id) VALUES (1001, 'ABC');  
INSERT INTO lignes_commande (commande_id, produit_id) VALUES (1001, 'DEF');  
COMMIT;  
```

Si l'une des trois insertions échoue → **aucune** n'est enregistrée.

### C - Consistency (Cohérence)

**Principe** : Une transaction fait passer la base de données d'un **état cohérent** à un autre **état cohérent**.

**Exemple** :
- Avant : Compte A = 1000€, Compte B = 500€, Total = 1500€
- Transaction : Transférer 100€ de A vers B
- Après : Compte A = 900€, Compte B = 600€, Total = 1500€ ✅

Le total reste cohérent. Les contraintes (solde ≥ 0, clés étrangères, etc.) sont respectées.

### I - Isolation (Isolation)

**Principe** : Les transactions **s'exécutent comme si elles étaient seules**, même si elles sont en réalité concurrentes.

**Analogie** : Vous travaillez dans un bureau avec des cloisons. Vous ne voyez pas ce que font vos collègues tant qu'ils n'ont pas fini.

**Exemple** :
```
Transaction A : Transfert de 100€ de Alice vers Bob  
Transaction B : Calcul du solde total de tous les comptes  

Transaction B voit SOIT l'état avant le transfert, SOIT l'état après,  
mais JAMAIS un état intermédiaire incohérent (Alice débitée mais Bob pas encore crédité)  
```

C'est grâce au **MVCC** (Multiversion Concurrency Control) que PostgreSQL réalise cela brillamment.

### D - Durability (Durabilité)

**Principe** : Une fois qu'une transaction est **validée** (COMMIT), ses modifications sont **permanentes**, même en cas de crash du serveur.

**Mécanisme** : PostgreSQL écrit dans le **WAL** (Write-Ahead Log) avant de valider.

**Garantie** :
```sql
BEGIN;  
INSERT INTO commandes (client_id, montant) VALUES (42, 1000000);  
COMMIT;  -- À partir de cet instant, la commande est GARANTIE enregistrée  

-- Même si le serveur plante 1 milliseconde après, la commande sera là au redémarrage
```

---

## Les défis de la concurrence

Gérer la concurrence n'est pas trivial. Voici les principaux défis que PostgreSQL doit relever :

### 1. Performance vs Cohérence

**Le dilemme** :
- Plus on garantit de cohérence → Plus on pose de verrous → Moins de performance
- Plus on veut de performance → Moins de verrous → Risque d'incohérences

**Solution PostgreSQL** : MVCC (Multiversion Concurrency Control)
- Les **lecteurs** ne bloquent jamais les **écrivains**
- Les **écrivains** ne bloquent jamais les **lecteurs**
- Performance ⚡ ET cohérence ✅ en même temps !

### 2. Éviter les deadlocks (interblocages)

**Le problème** :
```
Transaction A : verrouille la ligne 1, attend la ligne 2  
Transaction B : verrouille la ligne 2, attend la ligne 1  
→ DEADLOCK ! (Attente circulaire)
```

**Solution PostgreSQL** :
- Détection automatique des deadlocks
- Annulation d'une transaction pour débloquer
- Vous devez implémenter des retries dans votre code

### 3. Garantir l'isolation sans pénalité excessive

**Niveaux d'isolation** (du moins au plus strict) :
1. **Read Committed** (défaut) : Bonne performance, permet certaines anomalies  
2. **Repeatable Read** : Vue cohérente, plus strict  
3. **Serializable** : Cohérence maximale, comme si les transactions s'exécutaient une par une

Le choix du niveau d'isolation dépend de vos besoins métier.

### 4. Gérer les anomalies transactionnelles

Sans protection adéquate, ces anomalies peuvent survenir :

- **Dirty Read** : Lire des données non validées d'une autre transaction  
- **Non-Repeatable Read** : Lire la même ligne deux fois et obtenir des résultats différents  
- **Phantom Read** : Relire et trouver de nouvelles lignes apparues  
- **Lost Update** : Une modification en écrase une autre silencieusement  
- **Write Skew** : Deux transactions violent une contrainte métier ensemble

PostgreSQL vous protège de ces anomalies selon le niveau d'isolation choisi.

---

## Pourquoi PostgreSQL excelle dans ce domaine

### 1. MVCC : Une innovation majeure

Le **MVCC** (Multiversion Concurrency Control) est la pierre angulaire de PostgreSQL :

**Principe** : Au lieu de verrouiller les données pour les modifications, PostgreSQL crée de **nouvelles versions** des lignes.

**Résultat** :
```
Transaction A lit la ligne → Voit la version 1  
Transaction B modifie la ligne → Crée la version 2  
Transaction A lit à nouveau → Voit TOUJOURS la version 1 (cohérence)  
Transaction C commence après le COMMIT de B → Voit la version 2  
```

**Avantages** :
- ⚡ Haute performance (pas de blocage lecteur/écrivain)  
- ✅ Cohérence forte  
- 🎯 Isolation naturelle

**Inconvénient** :
- Nécessite du ménage (VACUUM) pour nettoyer les anciennes versions

### 2. Des verrous intelligents

PostgreSQL propose une **hiérarchie de verrous** granulaires :

- **Verrous de table** : Pour les opérations DDL (ALTER TABLE, etc.)  
- **Verrous de ligne** : Pour les UPDATE/DELETE spécifiques  
- **Advisory Locks** : Pour la coordination applicative

**Exemple** : Vous pouvez verrouiller une seule ligne sans bloquer toute la table.

### 3. Détection automatique des problèmes

- **Deadlocks** : Détectés et résolus automatiquement en 1 seconde (par défaut)  
- **Wraparound XID** : Prévenu par autovacuum  
- **Anomalies** : Selon le niveau d'isolation, PostgreSQL rejette les transactions conflictuelles

### 4. Flexibilité des niveaux d'isolation

PostgreSQL vous laisse choisir le **bon équilibre** pour chaque transaction :

```sql
-- Transaction simple : Read Committed (défaut, rapide)
BEGIN;  
UPDATE produits SET prix = 100 WHERE id = 1;  
COMMIT;  

-- Rapport financier : Repeatable Read (vue cohérente)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;  
SELECT SUM(montant) FROM commandes WHERE mois = 'Janvier';  
SELECT AVG(montant) FROM commandes WHERE mois = 'Janvier';  
COMMIT;  

-- Transfert bancaire : Serializable (cohérence maximale)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Vérifications et transferts critiques
COMMIT;
```

---

## Ce que vous allez apprendre dans ce chapitre

Ce chapitre est structuré en **7 sections** qui vous guideront progressivement de débutant à expert :

### 12.1. Cycle de vie d'une transaction (BEGIN, SAVEPOINT, COMMIT, ROLLBACK)
Vous apprendrez à :
- Démarrer, valider et annuler des transactions
- Utiliser les savepoints pour des rollbacks partiels
- Comprendre le mode autocommit
- Gérer les erreurs dans les transactions

### 12.2. MVCC (Multiversion Concurrency Control) : Le cœur de PostgreSQL
Vous découvrirez :
- Comment PostgreSQL gère les versions multiples
- Les métadonnées xmin et xmax
- Le rôle de VACUUM
- Pourquoi les lecteurs ne bloquent jamais les écrivains

### 12.3. Niveaux d'isolation ANSI
Vous maîtriserez :
- Read Committed (défaut)
- Repeatable Read
- Serializable
- Comment choisir le bon niveau pour votre application

### 12.4. Anomalies transactionnelles
Vous identifierez :
- Dirty Read, Non-Repeatable Read, Phantom Read
- Lost Update, Write Skew, Read Skew
- Comment les prévenir
- Leurs impacts métier

### 12.5. Gestion des verrous (Locks)
Vous comprendrez :
- Les types de verrous (table, ligne, advisory)
- Comment et quand PostgreSQL les pose
- Les deadlocks : causes et détection
- Les modes de verrouillage (FOR UPDATE, FOR SHARE)

### 12.6. Advisory Locks
Vous implémenterez :
- Des verrous applicatifs personnalisés
- La coordination de workers distribués
- Des mutex et sémaphores au niveau DB
- Des patterns de job queues robustes

### 12.7. Stratégies de détection et résolution des deadlocks
Vous appliquerez :
- Les meilleures pratiques de prévention
- Le monitoring et diagnostic
- Les patterns de retry avec backoff
- Les solutions aux problèmes courants en production

---

## Pour qui est ce chapitre ?

### Vous êtes **développeur** ?

Ce chapitre est **essentiel** pour vous car :
- ✅ Vous éviterez les bugs subtils de concurrence  
- ✅ Vous concevrez des applications robustes  
- ✅ Vous comprendrez pourquoi certaines requêtes échouent avec des erreurs de sérialisation  
- ✅ Vous optimiserez les performances de vos transactions

**Vous apprendrez** à écrire du code qui gère correctement la concurrence, implémente des retries, et choisit les bons niveaux d'isolation.

### Vous êtes **DevOps/SRE** ?

Ce chapitre vous permettra de :
- ✅ Diagnostiquer les problèmes de performance liés aux verrous  
- ✅ Surveiller les deadlocks et transactions longues  
- ✅ Configurer PostgreSQL pour votre workload  
- ✅ Résoudre les incidents de blocage en production

**Vous apprendrez** à utiliser pg_locks, pg_stat_activity, et les logs pour identifier et résoudre les problèmes.

### Vous êtes **architecte** ?

Ce chapitre vous aidera à :
- ✅ Concevoir des architectures résilientes  
- ✅ Choisir les bons patterns de concurrence  
- ✅ Évaluer les trade-offs (cohérence vs performance)  
- ✅ Prévoir la scalabilité

**Vous apprendrez** les principes fondamentaux pour des architectures solides.

---

## Prérequis pour ce chapitre

### Connaissances requises

Avant de plonger dans ce chapitre, vous devriez être à l'aise avec :

✅ **SQL de base** :
- SELECT, INSERT, UPDATE, DELETE
- WHERE, JOIN, GROUP BY

✅ **Concepts PostgreSQL fondamentaux** :
- Connexion à une base de données
- Exécution de requêtes simples
- Tables, colonnes, types de données

✅ **Notions générales** :
- Qu'est-ce qu'une base de données ?
- Pourquoi utiliser PostgreSQL ?

### Connaissances recommandées (mais pas obligatoires)

Ces connaissances aideront, mais nous les expliquerons au fur et à mesure :

⚡ Indexation et performance  
⚡ Contraintes (PRIMARY KEY, FOREIGN KEY)  
⚡ Architecture client-serveur

---

## Comment aborder ce chapitre ?

### Pour les débutants 🌱

**Approche recommandée** :
1. Lisez **dans l'ordre**, ne sautez pas de sections  
2. Testez chaque concept dans psql ou pgAdmin  
3. Prenez le temps de comprendre les analogies  
4. Revenez sur les sections complexes après avoir progressé  
5. **Focus** sur les sections 12.1, 12.2 et 12.3 en priorité

**Objectif** : Comprendre les concepts fondamentaux, savoir utiliser BEGIN/COMMIT, comprendre pourquoi MVCC est important.

### Pour les intermédiaires 📈

**Approche recommandée** :
1. Survolez les sections 12.1-12.3 si vous êtes déjà familier  
2. **Concentrez-vous** sur 12.4, 12.5 et 12.7  
3. Testez les scénarios de deadlocks  
4. Implémentez des retries dans votre langage préféré  
5. Appliquez à vos projets existants

**Objectif** : Maîtriser les anomalies, comprendre les verrous, savoir diagnostiquer et résoudre les problèmes.

### Pour les experts 🚀

**Approche recommandée** :
1. Utilisez ce chapitre comme **référence**  
2. Explorez les **patterns avancés** dans 12.6 et 12.7  
3. Étudiez les cas réels en fin de chaque section  
4. Comparez avec d'autres SGBD (MySQL, Oracle)  
5. Contribuez à l'amélioration de vos pratiques d'équipe

**Objectif** : Peaufiner vos connaissances, découvrir des subtilités, optimiser vos architectures.

---

## Le fil conducteur : Une application e-commerce

Pour rendre les concepts concrets, nous utiliserons tout au long du chapitre un **fil conducteur** : une application e-commerce simplifiée.

### Le schéma de base

```sql
-- Produits
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    prix NUMERIC(10,2),
    stock INTEGER
);

-- Clients
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(100)
);

-- Commandes
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER REFERENCES clients(id),
    date_commande TIMESTAMPTZ DEFAULT NOW(),
    montant_total NUMERIC(10,2),
    statut VARCHAR(20)
);

-- Lignes de commande
CREATE TABLE lignes_commande (
    id SERIAL PRIMARY KEY,
    commande_id INTEGER REFERENCES commandes(id),
    produit_id INTEGER REFERENCES produits(id),
    quantite INTEGER,
    prix_unitaire NUMERIC(10,2)
);
```

### Les scénarios que nous explorerons

Tout au long du chapitre, nous verrons comment gérer :

1. **Plusieurs clients achètent le même produit** (concurrence sur le stock)  
2. **Un client passe une commande multi-articles** (atomicité de la transaction)  
3. **Des rapports générés pendant les achats** (isolation des lectures)  
4. **Des workers qui traitent les commandes** (verrous et coordination)  
5. **Des mises à jour massives de prix** (deadlocks et prévention)

Ces scénarios couvrent les situations les plus courantes dans les applications réelles.

---

## Conventions utilisées dans ce chapitre

### Code SQL

```sql
-- Les commentaires expliquent le code
BEGIN;  -- Démarrer une transaction

-- Les instructions importantes
UPDATE produits SET stock = stock - 1 WHERE id = 1;

COMMIT;  -- Valider la transaction
```

### Résultats et sorties

```
id | nom         | stock
---|-------------|------
1  | Laptop      | 5
2  | Souris      | 50
```

### Avertissements et notes

**⚠️ Attention** : Points critiques à ne pas manquer

**💡 Astuce** : Bonnes pratiques et raccourcis

**🚨 Danger** : Erreurs courantes à éviter

**✅ Recommandation** : Solution ou approche recommandée

### Niveaux de difficulté

🌱 **Débutant** : Concepts fondamentaux, accessibles à tous

📈 **Intermédiaire** : Nécessite une compréhension des bases

🚀 **Avancé** : Pour les utilisateurs expérimentés

---

## Ressources complémentaires

### Documentation officielle PostgreSQL

- [PostgreSQL Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)  
- [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)  
- [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

### Articles recommandés

- "Understanding MVCC in PostgreSQL" par 2ndQuadrant  
- "PostgreSQL Anti-Patterns: Avoiding Pitfalls" par Percona  
- "A Primer on PostgreSQL Transactions" par Citus Data

### Outils utiles

- **psql** : Client en ligne de commande  
- **pgAdmin** : Interface graphique  
- **pg_stat_activity** : Vue système pour surveiller l'activité  
- **pg_locks** : Vue système pour voir les verrous actifs

---

## Un dernier mot avant de commencer

La **concurrence** et les **transactions** sont au cœur de toute application de base de données sérieuse. C'est ce qui différencie une application hobby d'un système de production robuste.

**Ce chapitre est dense** car le sujet est complexe, mais nous avons fait en sorte qu'il soit **accessible** grâce à :
- Des **analogies** du monde réel
- Des **exemples concrets** et testables
- Une **progression logique** du simple au complexe
- Des **schémas visuels** pour clarifier
- Des **cas pratiques** d'applications réelles

**Prenez votre temps**. Certains concepts comme MVCC ou les anomalies transactionnelles peuvent sembler abstraits au début, mais ils deviendront clairs avec la pratique.

**Testez tout**. La meilleure façon de comprendre la concurrence est de **provoquer** les situations décrites dans ce chapitre dans un environnement de test sûr.

**Posez-vous des questions** :
- Que se passe-t-il si deux utilisateurs font ça en même temps ?
- Comment PostgreSQL gère-t-il cette situation ?
- Quelle serait la bonne approche pour mon application ?

---

## Prêt à plonger ?

Vous êtes maintenant prêt à explorer l'un des aspects les plus puissants et fascinants de PostgreSQL. Chaque section s'appuie sur la précédente pour construire une **compréhension complète** de la gestion de la concurrence.

**Objectif final** : À la fin de ce chapitre, vous serez capable de :
- ✅ Concevoir des transactions robustes et efficaces  
- ✅ Choisir le bon niveau d'isolation pour chaque situation  
- ✅ Comprendre et prévenir les anomalies transactionnelles  
- ✅ Diagnostiquer et résoudre les problèmes de verrous et deadlocks  
- ✅ Implémenter des patterns de concurrence avancés  
- ✅ Surveiller et optimiser la concurrence en production

**Commençons par le début : le cycle de vie d'une transaction.**

---

**Navigation du chapitre :**

📍 **Vous êtes ici** : Introduction au chapitre 12

➡️ **Suivant** : 12.1. Cycle de vie d'une transaction (BEGIN, SAVEPOINT, COMMIT, ROLLBACK)

---

**Points clés de l'introduction :**

- 🔑 Concurrence = gérer plusieurs opérations simultanées sans conflits  
- 🔑 Transaction = unité de travail atomique (tout ou rien)  
- 🔑 ACID = Atomicité, Cohérence, Isolation, Durabilité  
- 🔑 MVCC = innovation majeure de PostgreSQL (versions multiples)  
- 🔑 Anomalies transactionnelles = comportements indésirables à prévenir  
- 🔑 Verrous = mécanisme de coordination des accès concurrents  
- 🔑 Deadlocks = attentes circulaires, détectés automatiquement  
- 🔑 Ce chapitre est essentiel pour tout développeur/DevOps sérieux

⏭️ [Cycle de vie d'une transaction (BEGIN, SAVEPOINT, COMMIT, ROLLBACK)](/12-concurrence-et-transactions/01-cycle-de-vie-transaction.md)
