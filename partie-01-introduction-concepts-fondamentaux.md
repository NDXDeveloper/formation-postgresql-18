🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 1 : Introduction et Concepts Fondamentaux

**Niveau** : Débutant  
**Durée estimée** : 5-7 heures  
**Prérequis** : Aucun (connaissances de base en informatique suffisantes)  

---

## Vue d'ensemble

Bienvenue dans cette première partie de cette formation. Cette section pose les fondations essentielles pour comprendre PostgreSQL et les bases de données relationnelles. Elle s'adresse particulièrement aux développeurs et DevOps qui débutent avec les systèmes de gestion de bases de données ou qui souhaitent consolider leurs connaissances théoriques.

### Pourquoi cette partie est cruciale ?

Les bases de données sont au cœur de presque toutes les applications modernes. Avant de plonger dans les requêtes SQL complexes ou l'administration avancée, il est indispensable de comprendre :

- **Ce qu'est réellement une base de données** et pourquoi nous en avons besoin  
- **Comment PostgreSQL fonctionne** sous le capot  
- **Les concepts fondamentaux** qui reviendront tout au long de votre parcours  
- **L'écosystème** dans lequel PostgreSQL évolue

Cette compréhension théorique vous permettra de prendre de meilleures décisions techniques, de déboguer plus efficacement et d'exploiter pleinement la puissance de PostgreSQL.

---

## Objectifs d'apprentissage

À l'issue de cette première partie, vous serez capable de :

### 🎯 Concepts généraux
- Expliquer ce qu'est une donnée, une base de données et un SGBD
- Différencier les modèles relationnels (SGBDR) et NoSQL
- Comprendre les propriétés ACID et leur importance
- Situer PostgreSQL dans le paysage des bases de données

### 🏗️ Architecture PostgreSQL
- Décrire l'architecture client-serveur de PostgreSQL
- Comprendre le rôle du Postmaster et des processus d'arrière-plan
- Expliquer la gestion de la mémoire (Shared Buffers, Local Memory)
- Connaître la structure physique des données (Heap, TOAST, WAL)
- Découvrir les nouveautés d'architecture de PostgreSQL 18

### 🧱 Objets de base de données
- Maîtriser la hiérarchie logique (Instance → Database → Schema → Table)
- Créer des tables avec les types de données appropriés
- Utiliser les types spécifiques de PostgreSQL (JSONB, ARRAYS, UUID)
- Gérer les séquences et la génération automatique d'identifiants
- Comprendre le rôle des schémas et du search_path

### 🔧 Outillage
- Utiliser les outils essentiels de l'écosystème (psql, pgAdmin, DBeaver)
- Comprendre l'importance du connection pooling
- Se familiariser avec le protocole réseau PostgreSQL

---

## Structure de cette partie

Cette première partie est organisée en **4 chapitres progressifs** :

### 📚 **Chapitre 1 : Introduction aux Bases de Données**
Concepts généraux indépendants de PostgreSQL : qu'est-ce qu'une donnée, le modèle relationnel, les transactions ACID, et une comparaison avec le NoSQL.

### 🐘 **Chapitre 2 : Présentation de PostgreSQL**
Histoire, philosophie et positionnement de PostgreSQL dans l'industrie. Découverte de la communauté open-source et comparaison avec les autres SGBD majeurs.

### ⚙️ **Chapitre 3 : Architecture de PostgreSQL**
Plongée technique dans le fonctionnement interne : modèle client-serveur, processus, gestion mémoire, structure physique des fichiers, et les innovations de PostgreSQL 18 comme l'I/O asynchrone.

### 🗂️ **Chapitre 4 : Les Objets de la Base de Données**
Apprentissage pratique du DDL (Data Definition Language) : création de tables, types de données, séquences, et gestion du schéma logique.

---

## Philosophie pédagogique

### 📖 Théorie d'abord, pratique ensuite

Cette formation est **exclusivement théorique**. Nous nous concentrons sur la compréhension des concepts, des mécanismes et des principes. Les exercices pratiques peuvent être réalisés séparément, mais la maîtrise théorique est un prérequis indispensable.

### 🎓 Approche progressive

Chaque concept s'appuie sur les précédents. Nous commençons par les fondamentaux universels avant de nous spécialiser dans PostgreSQL. Cette progression vous permet de construire une compréhension solide et durable.

### 🔍 Pourquoi et comment

Au-delà du "quoi", nous expliquons le **"pourquoi"** (les raisons des choix d'architecture) et le **"comment"** (les mécanismes sous-jacents). Cette approche vous rend autonome face à de nouveaux problèmes.

### 🆕 Focus PostgreSQL 18

Tout au long de cette partie, nous mettons en évidence les **nouveautés de PostgreSQL 18** (Septembre 2025) pour que vous soyez immédiatement à jour avec la version la plus récente.

---

## Public cible et prérequis

### 👥 Pour qui ?

- **Développeurs** souhaitant comprendre comment fonctionne leur base de données  
- **DevOps/SRE** qui doivent déployer et maintenir PostgreSQL  
- **Data Engineers** débutant avec PostgreSQL  
- **Architectes** voulant prendre des décisions éclairées sur leur stack technique  
- **Étudiants** en informatique cherchant une formation structurée

### 📋 Prérequis

- Connaissances de base en informatique
- Compréhension des concepts de fichiers et répertoires
- Notions de base sur les serveurs et le réseau (HTTP, TCP/IP)
- **Aucune connaissance préalable en SQL ou bases de données requise**

### ⚙️ Environnement (optionnel)

Bien que cette partie soit théorique, si vous souhaitez expérimenter parallèlement :
- Un système Linux, macOS ou Windows
- PostgreSQL 18 installé (ou accès à un serveur)
- Un client SQL (psql, DBeaver, pgAdmin)

---

## Nouveautés PostgreSQL 18 abordées dans cette partie

Cette première partie introduit les innovations suivantes de PostgreSQL 18 :

### ⚡ Performance et I/O (Chapitre 3)
- **Sous-système I/O asynchrone (AIO)** : Amélioration drastique des performances I/O (jusqu'à 3× plus rapide sur certaines opérations)
- Configuration `io_method` avec les modes `worker` (défaut) et `io_uring` (Linux)
- **Wire Protocol 3.2** : Clés d'annulation 256 bits pour une sécurité renforcée

### 📊 Types de données (Chapitre 4)
- **UUIDv7** : Nouvelle version des UUID avec tri temporel natif, idéale pour les clés primaires distribuées

> 💡 Les autres nouveautés majeures de PostgreSQL 18 (OAuth 2.0, colonnes virtuelles, Data Checksums par défaut, pg_upgrade amélioré, etc.) sont couvertes dans les parties suivantes de la formation.

---

## Comment utiliser cette partie ?

### 📖 Lecture séquentielle recommandée

Les chapitres sont conçus pour être lus dans l'ordre. Chaque section s'appuie sur les précédentes. Résistez à la tentation de sauter des sections, même si certains concepts vous semblent familiers.

### ✍️ Prenez des notes

Notez les concepts clés, les définitions importantes et les questions que vous vous posez. Cette partie étant théorique, la synthèse personnelle est essentielle pour la mémorisation.

### 🔄 Répétition espacée

N'hésitez pas à revenir sur les chapitres précédents. La compréhension profonde vient avec la répétition et la mise en relation des concepts.

### 💡 Expérimentation (optionnelle)

Si vous avez accès à une instance PostgreSQL, expérimentez avec les concepts présentés. Créez des tables, explorez les vues système, interrogez les statistiques.

### ❓ Questions et approfondissement

Chaque chapitre peut soulever des questions. Notez-les et cherchez à y répondre via la documentation officielle PostgreSQL ou la communauté. Les ressources sont listées dans le chapitre 21.

---

## Conventions utilisées dans cette formation

### 📝 Notation

- **Gras** : Termes importants et concepts clés  
- `Code` : Commandes, noms de fichiers, paramètres  
- 🆕 : Nouveautés spécifiques à PostgreSQL 18  
- ⚠️ : Avertissements et points d'attention  
- 💡 : Conseils et bonnes pratiques  
- 📖 : Références à la documentation

### 🗂️ Structure des chapitres

Chaque chapitre suit une structure similaire :
1. Introduction et contexte  
2. Concepts théoriques détaillés  
3. Exemples et illustrations  
4. Nouveautés PostgreSQL 18 (si applicable)  
5. Résumé des points clés

---

## Après cette partie

Une fois cette première partie maîtrisée, vous serez prêt à aborder :

### ➡️ **Partie 2 : Le Langage SQL - Interrogation et Manipulation**
Requêtes SELECT, jointures, agrégation, manipulation de données (INSERT, UPDATE, DELETE).

### ➡️ **Partie 3 : SQL Avancé et Modélisation**
Sous-requêtes, CTE, fonctions de fenêtrage, modélisation avancée et partitionnement.

### ➡️ **Partie 4 : Administration, Performance et Architecture Avancée**
Transactions, indexation, monitoring, programmation serveur, haute disponibilité.

### ➡️ **Partie 5 : Écosystème, Production et Ouverture**
Extensions, déploiement production, architectures modernes, intégrations.

---

## Ressources complémentaires

### 📚 Documentation officielle
- [PostgreSQL 18 Documentation](https://www.postgresql.org/docs/18/)  
- [PostgreSQL Wiki](https://wiki.postgresql.org/)

### 🎥 Communauté
- [pgsql-general Mailing List](https://www.postgresql.org/list/)  
- [Reddit r/PostgreSQL](https://www.reddit.com/r/PostgreSQL/)  
- [Discord PostgreSQL](https://discord.gg/postgresql)

### 📖 Blogs recommandés
- EDB Blog (ex-2ndQuadrant)
- Percona Database Performance Blog
- CrunchyData Blog

---

## Mot de conclusion

Cette première partie est votre passeport pour le monde PostgreSQL. Prenez le temps nécessaire pour bien assimiler chaque concept. La solidité de vos fondations déterminera votre capacité à maîtriser les aspects avancés dans les parties suivantes.

PostgreSQL est un système puissant et élégant. Comprendre son architecture et sa philosophie vous permettra non seulement de l'utiliser efficacement, mais aussi d'apprécier la beauté de sa conception.

**Bonne formation ! 🐘**

---


⏭️ [Introduction aux Bases de Données](/01-introduction-aux-bases-de-donnees/README.md)
