🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 1 : Introduction et Concepts Fondamentaux

**Niveau** : Débutant  
**Durée estimée** : 5 à 7 heures  
**Prérequis** : Aucun (connaissances de base en informatique suffisantes)  

---

## Vue d'ensemble

Bienvenue dans cette première partie de la formation. Elle pose les fondations essentielles pour comprendre PostgreSQL et les bases de données relationnelles. Elle s'adresse particulièrement aux développeurs et DevOps qui débutent avec les systèmes de gestion de bases de données, ainsi qu'aux praticiens qui souhaitent consolider leurs connaissances théoriques.

### Pourquoi cette partie est cruciale ?

Les bases de données sont au cœur de presque toutes les applications modernes. Avant de plonger dans les requêtes SQL complexes ou l'administration avancée, il est indispensable de comprendre :

- **Ce qu'est réellement une base de données** et pourquoi nous en avons besoin  
- **Comment PostgreSQL fonctionne** sous le capot  
- **Les concepts fondamentaux** qui reviendront tout au long de votre parcours  
- **L'écosystème** dans lequel PostgreSQL évolue  
- **Le modèle économique et la licence** (PostgreSQL License, type BSD, **gratuit et libre pour tout usage** y compris commercial)

Cette compréhension théorique vous permettra de prendre de meilleures décisions techniques, de déboguer plus efficacement, d'éviter les anti-patterns classiques, et d'exploiter pleinement la puissance de PostgreSQL.

> 💡 **Investissement durable** : PostgreSQL existe depuis 1996 (et son ancêtre POSTGRES depuis 1986). Apprendre ses fondamentaux est un investissement à long terme : les concepts présentés ici resteront valables pendant des décennies, contrairement aux technologies plus volatiles.

---

## Objectifs d'apprentissage

À l'issue de cette première partie, vous serez capable de :

### 🎯 Concepts généraux
- Expliquer ce qu'est une donnée, une base de données et un SGBD  
- Distinguer DDL, DML, DQL, DCL et TCL (catégories de commandes SQL)  
- Différencier les modèles relationnels (SGBDR) et NoSQL  
- Comprendre les propriétés ACID et leur importance  
- Situer PostgreSQL dans le paysage des bases de données

### 🏗️ Architecture PostgreSQL
- Décrire l'architecture client-serveur de PostgreSQL (modèle process-per-connection)  
- Comprendre le rôle du processus principal (`postgres`, historiquement appelé `postmaster`) et des processus d'arrière-plan (WAL writer, checkpointer, autovacuum, background writer, etc.)  
- Expliquer la gestion de la mémoire (Shared Buffers, Local Memory, `work_mem`)  
- Connaître la structure physique des données (Heap, TOAST, WAL, Visibility Map)  
- Comprendre les principes du **MVCC** (Multiversion Concurrency Control) et leur impact  
- Découvrir les nouveautés d'architecture de PostgreSQL 18 (AIO, Wire Protocol 3.2)

### 🧱 Objets de base de données
- Maîtriser la hiérarchie logique (Instance → Database → Schema → Table)  
- Créer des tables avec les types de données appropriés  
- Utiliser les types spécifiques de PostgreSQL (JSONB, ARRAY, UUID, ENUM)  
- Gérer les séquences et la génération automatique d'identifiants  
- Comprendre le rôle des schémas et du `search_path`

### 🔧 Outillage
- Utiliser les outils essentiels de l'écosystème (`psql`, pgAdmin, DBeaver)  
- Comprendre l'importance du **connection pooling** (PgBouncer, pgcat, pgpool-II)  
- Se familiariser avec le **protocole filaire** PostgreSQL (Wire Protocol 3.2 en PG 18)

---

## Structure de cette partie

Cette première partie est organisée en **4 chapitres progressifs** :

| Chapitre | Titre | Durée estimée | Difficulté |
|----------|-------|---------------|------------|
| 1 | Introduction aux Bases de Données | ~1 h | ⭐ |
| 2 | Présentation de PostgreSQL | ~1 h | ⭐ |
| 3 | Architecture de PostgreSQL | ~2 h | ⭐⭐⭐ |
| 4 | Les Objets de la Base de Données | ~2-3 h | ⭐⭐ |

### 📚 **Chapitre 1 : Introduction aux Bases de Données**
Concepts généraux indépendants de PostgreSQL : qu'est-ce qu'une donnée, le modèle relationnel, les transactions ACID, et une comparaison avec le NoSQL.

### 🐘 **Chapitre 2 : Présentation de PostgreSQL**
Histoire, philosophie et positionnement de PostgreSQL dans l'industrie. Découverte de la communauté open-source et comparaison avec les autres SGBD majeurs (Oracle, SQL Server, MySQL, SQLite…).

### ⚙️ **Chapitre 3 : Architecture de PostgreSQL**
Plongée technique dans le fonctionnement interne : modèle client-serveur, processus, gestion mémoire, structure physique des fichiers (heap, TOAST, WAL), MVCC, et les innovations de PostgreSQL 18 comme l'I/O asynchrone.

### 🗂️ **Chapitre 4 : Les Objets de la Base de Données**
Apprentissage pratique du **DDL** (Data Definition Language) : création de tables, types de données fondamentaux et spécifiques PostgreSQL (JSONB, ARRAY, UUID, ENUM…), séquences, domaines, types personnalisés, et gestion des modifications de schéma.

---

## Philosophie pédagogique

### 📖 Théorie d'abord, pratique au fil du texte

Cette formation est **principalement théorique** : elle se concentre sur la compréhension des concepts, des mécanismes et des principes. Les chapitres incluent toutefois de **nombreux exemples SQL illustratifs** que vous pouvez exécuter sur une instance PostgreSQL pour ancrer la compréhension. La création d'exercices ou de projets dédiés est laissée à des supports complémentaires : ici, l'objectif est d'acquérir une **maîtrise conceptuelle solide** sur laquelle s'appuiera ensuite votre pratique.

### 🎓 Approche progressive

Chaque concept s'appuie sur les précédents. Nous commençons par les fondamentaux universels avant de nous spécialiser dans PostgreSQL. Cette progression vous permet de construire une compréhension solide et durable.

### 🔍 Pourquoi et comment

Au-delà du « quoi », nous expliquons le **« pourquoi »** (les raisons des choix d'architecture) et le **« comment »** (les mécanismes sous-jacents). Cette approche vous rend autonome face à de nouveaux problèmes : quand vous comprenez les principes, vous pouvez raisonner sur des situations inédites, lire le code source au besoin, et choisir la bonne solution même sans recette toute faite.

### 🆕 Focus PostgreSQL 18

Tout au long de cette partie, nous mettons en évidence les **nouveautés de PostgreSQL 18** (sortie en septembre 2025) pour que vous soyez immédiatement à jour avec la version la plus récente. Les nouveautés sont signalées par le marqueur 🆕 dans les chapitres concernés.

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

Bien que cette partie soit théorique, si vous souhaitez expérimenter en parallèle :
- Un système Linux, macOS ou Windows  
- **PostgreSQL 18** installé (ou accès à un serveur)  
- Un client SQL : `psql` (en ligne de commande), DBeaver ou pgAdmin

#### Démarrage rapide avec Docker

L'installation native (paquets `apt`, `dnf`, Homebrew, EnterpriseDB Installer…) reste la voie recommandée pour un usage prolongé. Mais pour suivre cette formation et tester rapidement, **Docker** est la solution la plus simple :

```bash
# Lancer une instance PostgreSQL 18 jetable, sur le port 5432 local
docker run --name pg18 \
  -e POSTGRES_PASSWORD=motdepasse \
  -p 5432:5432 \
  -d postgres:18

# S'y connecter avec psql
docker exec -it pg18 psql -U postgres

# Arrêter et supprimer (les données sont perdues)
docker stop pg18 && docker rm pg18
```

Pour conserver vos données entre redémarrages, ajoutez un volume :

```bash
docker run --name pg18 \
  -e POSTGRES_PASSWORD=motdepasse \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  -d postgres:18
```

> 💡 **Cloud gratuit** : si vous ne voulez rien installer, des services comme Neon, Supabase ou Aiven proposent des instances PostgreSQL gratuites en quelques clics. Pratique pour suivre la formation depuis n'importe quel ordinateur.

---

## Nouveautés PostgreSQL 18 abordées dans cette partie

Cette première partie introduit les innovations suivantes de PostgreSQL 18 :

### ⚡ Performance et I/O (Chapitre 3)
- **Sous-système I/O asynchrone (AIO)** : nouveau mécanisme de lectures parallélisées qui peut accroître nettement le débit (**jusqu'à ~3× selon le scénario**, surtout avec `io_uring` sur un stockage à forte latence) sur les workloads en lecture intensive (sequential scans, bitmap heap scans, `VACUUM`). Les écritures restent synchrones dans PG 18.  
- **Paramètre `io_method`** (à définir dans `postgresql.conf`, nécessite redémarrage) :  
  - `sync` : comportement historique (équivalent PG 17), lectures bloquantes avec `posix_fadvise`  
  - `worker` : processus dédiés en arrière-plan (défaut PG 18)  
  - `io_uring` : interface noyau Linux moderne, débit maximal sur stockages à forte latence  
- **Wire Protocol 3.2** : clés d'annulation de requête sur 256 bits (vs 32 bits auparavant) pour une sécurité renforcée

### 📊 Types de données (Chapitre 4)
- **UUIDv7** : nouvelle version des UUID avec tri temporel natif, idéale pour les clés primaires distribuées et l'efficacité des index B-tree  
- **Colonnes générées VIRTUAL** : `GENERATED ALWAYS AS … VIRTUAL` calcule la valeur **à la lecture** (sans coût de stockage). C'est désormais le **mode par défaut** en PG 18 lorsqu'on omet le mot-clé, là où `STORED` (valeur matérialisée sur disque) était l'unique mode — et obligatoire — de PG 12 à 17

> 💡 Les autres nouveautés majeures de PostgreSQL 18 (OAuth 2.0, contraintes temporelles `WITHOUT OVERLAPS`, Data Checksums activés par défaut, `pg_upgrade` amélioré, etc.) sont couvertes dans les parties suivantes de la formation.

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

Chaque chapitre peut soulever des questions. Notez-les et cherchez à y répondre via la documentation officielle PostgreSQL ou la communauté. Les ressources sont rassemblées dans le **chapitre 22** (« Conclusion et perspectives ») et dans les **annexes** (glossaire, référence des requêtes SQL, référence de configuration).

### 🚨 Erreurs courantes à éviter

Tout au long de la formation, plusieurs **anti-patterns** sont signalés (encadrés ⚠️). Lisez-les attentivement : ils correspondent à des erreurs réellement observées en production qui coûtent du temps, de l'argent ou des données. Mieux vaut les connaître avant de les commettre.

Exemples typiques que vous croiserez :
- Utiliser `FLOAT` pour stocker de l'argent (erreurs d'arrondi garanties)  
- Utiliser `TIMESTAMP` sans fuseau pour des événements (ambiguïtés temporelles)  
- Oublier `SET search_path` sur les fonctions `SECURITY DEFINER` (faille de sécurité)  
- Lancer un `ALTER TABLE` lourd en production sans `lock_timeout` (blocage en cascade)  
- Stocker des fichiers volumineux dans la base au lieu d'un stockage externe (S3, CDN)

---

## Conventions utilisées dans cette formation

### 📝 Notation

- **Gras** : termes importants et concepts clés  
- *Italique* : termes étrangers, citations courtes  
- `Code` : commandes, mots-clés SQL, noms de fichiers, paramètres, expressions techniques  
- ```` ```sql ```` : blocs de code SQL exécutables  
- 🆕 : nouveautés spécifiques à PostgreSQL 18  
- ⚠️ : avertissements et points d'attention  
- 🚨 : alertes de sécurité ou de production critiques  
- 💡 : conseils, astuces et bonnes pratiques  
- 📚 : références à la documentation officielle  
- 📌 : recommandation forte de la communauté  
- 🔐 : points de sécurité importants  
- 📦 : détails de stockage interne (TOAST, WAL, etc.)  
- 🌍 : aspects d'internationalisation (locales, encodages, fuseaux horaires)  
- 🔤 : précisions sur les caractères, Unicode, collations

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
- [Listes de diffusion PostgreSQL](https://www.postgresql.org/list/) (notamment `pgsql-general` pour les questions générales, `pgsql-hackers` pour le développement du moteur, `pgsql-bugs` pour les rapports de bugs)  
- [Reddit r/PostgreSQL](https://www.reddit.com/r/PostgreSQL/) — discussions, news, partages d'expérience  
- [Discord « People, Postgres, Data »](https://discord.com/invite/bW2hsax8We) — serveur communautaire le plus actif autour de PostgreSQL et son écosystème  
- [Discord « PostgreSQL Hacking »](https://discord.com/invite/bx2G9KWyrY) — pour ceux qui veulent contribuer au code de PostgreSQL ou à des extensions  
- [Stack Overflow — tag `postgresql`](https://stackoverflow.com/questions/tagged/postgresql) — pour les questions techniques avec réponses validées  
- [DBA Stack Exchange — tag `postgresql`](https://dba.stackexchange.com/questions/tagged/postgresql) — questions plus pointues sur l'administration  
- [Liens et événements officiels](https://www.postgresql.org/community/) — point de départ officiel pour toute la communauté (conférences PGConf, événements régionaux)

### 📖 Blogs recommandés
- [EDB Blog](https://www.enterprisedb.com/blog) — éditeur d'EDB Postgres Advanced Server (a absorbé 2ndQuadrant), nombreuses analyses techniques  
- [CrunchyData Blog](https://www.crunchydata.com/blog) — pratique production, performance, partitionnement, PostGIS  
- [Percona Database Performance Blog](https://www.percona.com/blog/) — perspective multi-SGBD avec focus PostgreSQL et MySQL  
- [pganalyze Blog](https://pganalyze.com/blog) — articles fouillés sur l'internals (série vidéo « 5mins of Postgres »)  
- [depesz](https://www.depesz.com/) — blog de Hubert Lubaczewski, référence pour la série **« Waiting for PostgreSQL »** (les nouveautés de chaque version, depuis 2008) et l'outil d'analyse de plans `explain.depesz.com`  
- [CYBERTEC Blog](https://www.cybertec-postgresql.com/en/blog/) — éditeur autrichien spécialisé PostgreSQL, contenus très techniques

### 📨 Newsletters et veille
- [Postgres Weekly](https://postgresweekly.com/) — newsletter hebdomadaire de référence (articles, releases, projets de l'écosystème)  
- [Planet PostgreSQL](https://planet.postgresql.org/) — agrégateur officiel des blogs de la communauté  
- [PostgreSQL News](https://www.postgresql.org/about/newsarchive/) — annonces officielles du projet

---

## Mot de conclusion

Cette première partie est votre passeport pour le monde PostgreSQL. Prenez le temps nécessaire pour bien assimiler chaque concept. La solidité de vos fondations déterminera votre capacité à maîtriser les aspects avancés dans les parties suivantes.

PostgreSQL est un système puissant et élégant. Comprendre son architecture et sa philosophie vous permettra non seulement de l'utiliser efficacement, mais aussi d'apprécier la beauté de sa conception — fruit de près de quarante années d'amélioration continue par une communauté mondiale.

> 💡 **Conseil final** : ne lisez pas cette formation comme un roman. Revenez régulièrement sur les chapitres déjà lus, en particulier après avoir pratiqué sur une vraie base. Les concepts abstraits prennent tout leur sens quand vous les avez vus à l'œuvre sur des données réelles.

**Bonne formation ! 🐘**

---


⏭️ [Introduction aux Bases de Données](/01-introduction-aux-bases-de-donnees/README.md)
