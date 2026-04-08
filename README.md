# 🐘 Formation Complète PostgreSQL 18

![License](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-blue.svg)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-336791.svg)
![Version](https://img.shields.io/badge/Version-1.0-green.svg)
![Langue](https://img.shields.io/badge/Langue-Français-blue.svg)
![Niveau](https://img.shields.io/badge/Niveau-Débutant%20→%20Expert-orange.svg)

**Un parcours théorique complet pour découvrir, comprendre et approfondir PostgreSQL 18, des fondamentaux à la production.**

![PostgreSQL Logo](https://wiki.postgresql.org/images/a/a4/PostgreSQL_logo.3colors.svg)

---

## 📖 Table des matières

- [À propos](#-à-propos)  
- [Contenu](#-contenu-de-la-formation)  
- [Démarrage rapide](#-démarrage-rapide)  
- [Structure](#-structure-du-projet)  
- [Utilisation](#-comment-utiliser-cette-formation)  
- [Nouveautés PG 18](#-nouveautés-postgresql-18)  
- [Licence](#-licence)  
- [Contact](#-contact)

---

## 📋 À propos

Cette formation propose une approche théorique complète de PostgreSQL 18, pensée pour accompagner développeurs et DevOps dans leur apprentissage du SGBD open source le plus avancé.

**✨ Ce que vous découvrirez :**
- 🎯 **21 chapitres progressifs** du débutant à l'expert  
- 🏗️ **Architecture interne** et concepts fondamentaux (MVCC, WAL, TOAST)  
- 🚀 **Performance** et optimisation (indexation, EXPLAIN, planificateur)  
- 🔒 **Sécurité** moderne (OAuth 2.0, SCRAM, RLS, TLS 1.3)  
- 🌐 **Production** (HA, réplication, monitoring, Kubernetes)  
- 🆕 **Nouveautés PG 18** (I/O asynchrone, UUIDv7, colonnes virtuelles)  
- 🇫🇷 **En français** - Formation théorique sans exercices pratiques

**Durée estimée :** 40-60 heures • **Niveau :** Tous niveaux • **Format :** Théorique

---

## 📚 Contenu de la formation

### 🌱 Partie 1 : Introduction et Concepts Fondamentaux (Débutant)

1. **Introduction aux Bases de Données** - SGBD, modèle relationnel, ACID  
2. **Présentation de PostgreSQL** - Histoire, écosystème, comparaison  
3. **Architecture de PostgreSQL** - Client-serveur, mémoire, I/O asynchrone 🆕  
4. **Objets de la Base** - DDL, types de données, schémas

### 🌿 Partie 2 : Le Langage SQL (Intermédiaire)

5. **Requêtes de Sélection** - SELECT, filtrage, tri, pagination  
6. **Manipulation des Données** - INSERT, UPDATE, DELETE, COPY, MERGE 🆕  
7. **Relations et Jointures** - PK/FK, types de jointures, contraintes temporelles 🆕  
8. **Agrégation et Groupement** - GROUP BY, fonctions d'agrégation, ROLLUP

### 🌳 Partie 3 : SQL Avancé et Modélisation (Avancé)

9. **Techniques SQL Avancées** - CTE, récursivité, optimisations OR → ANY 🆕  
10. **Fonctions de Fenêtrage** - Window functions, RANK, LAG/LEAD  
11. **Modélisation Avancée** - Normalisation, JSONB, partitionnement, colonnes virtuelles 🆕

### 🚀 Partie 4 : Administration et Performance (Avancé)

12. **Concurrence et Transactions** - MVCC, niveaux d'isolation, locks  
13. **Indexation et Optimisation** - B-Tree, GIN, GiST, BRIN, Skip Scan 🆕  
14. **Observabilité et Monitoring** - pg_stat_*, métriques I/O/WAL 🆕, Prometheus  
15. **Programmation Serveur** - PL/pgSQL, triggers, procédures stockées  
16. **Administration et Sécurité** - Authentification, autorisations, OAuth 2.0 🆕, VACUUM  
17. **Haute Disponibilité** - Réplication, failover, Patroni, architectures HA

### 🌐 Partie 5 : Écosystème et Production (Expert)

18. **Extensions et Intégrations** - PostGIS, pgvector, TimescaleDB, FDW  
19. **PostgreSQL en Production** - Cloud, Kubernetes, migrations, pg_upgrade 🆕  
20. **Drivers et Connexion Applicative** - psycopg3, node-postgres, patterns  
20bis. **Architectures Modernes** - Microservices, CQRS, serverless, K8s  
21. **Conclusion et Perspectives** - Roadmap, certifications, ressources

### 📖 Annexes de Référence

- **Glossaire** - Termes PostgreSQL et acronymes  
- **Commandes psql** - Navigation, configuration, export/import  
- **Requêtes SQL** - Administration, monitoring, analyse  
- **Configuration** - Par cas d'usage (OLTP, OLAP, dev)  
- **Checklist Performance** - Audits de configuration et indexation  
- **Nouveautés PG 18** 🆕 - Tableau récapitulatif complet  
- **Scripts Shell** - Backup, monitoring, commandes utiles

---

## 🚀 Démarrage rapide

### Prérequis

- Connaissances de base en SQL
- Familiarité avec les systèmes Unix/Linux
- Curiosité et envie d'apprendre ! 🎓

### Cloner cette formation

```bash
git clone https://github.com/NDXDeveloper/formation-postgresql-18.git  
cd formation-postgresql-18  
```

### Consulter le sommaire complet

```bash
cat SOMMAIRE.md
```

---

## 📁 Structure du projet

```
formation-postgresql-18/
├── README.md
├── SOMMAIRE.md
├── LICENSE
├── 01-introduction-aux-bases-de-donnees/
├── 02-presentation-de-postgresql/
├── 03-architecture-de-postgresql/
├── ...
├── 21-conclusion-et-perspectives/
└── annexes/
    ├── glossaire/
    ├── commandes-psql/
    ├── requetes-sql-reference/
    ├── configuration-reference/
    ├── checklist-performance/
    ├── nouveautes-pg18/ 🆕
    └── commandes-shell-scripts/
```

**📊 Statistiques :**
- 21 chapitres principaux + 1 bonus
- 7 annexes de référence
- 370+ fichiers markdown
- 100+ nouveautés PostgreSQL 18 documentées

---

## 🎯 Comment utiliser cette formation

### 🌱 Débutant complet
👉 Commencez par la [Partie 1](01-introduction-aux-bases-de-donnees/) et suivez l'ordre

### 🌿 Développeur SQL
👉 Allez directement à la [Partie 2](05-requetes-de-selection/) pour approfondir le langage SQL

### 🌳 DBA ou DevOps
👉 Explorez la [Partie 4](12-concurrence-et-transactions/) (Administration & Performance)

### 🚀 Architecture Production
👉 Plongez dans la [Partie 5](18-extensions-et-integrations/) (Écosystème & Production)

### 📖 Référence rapide
👉 Consultez les [Annexes](annexes/) pour des aide-mémoires et checklists

**💡 Conseil :** Cette formation est théorique. Installez PostgreSQL 18 localement pour expérimenter !

---

## 🗓️ Parcours suggéré

| Niveau | Parties | Durée | Objectif |
|--------|---------|-------|----------|
| 🌱 **Débutant** | 1-2 | 12-15h | Découvrir les bases de données et PostgreSQL |
| 🌿 **Intermédiaire** | 2-3 | 15-20h | Approfondir SQL et la modélisation |
| 🌳 **Avancé** | 4 | 15-20h | Performance, administration, sécurité |
| 🚀 **Expert** | 5 | 10-15h | Production, HA, architectures modernes |

**Total :** 40-60 heures sur 8-12 semaines (4-6h/semaine)

---

## 🆕 Nouveautés PostgreSQL 18

PostgreSQL 18 (septembre 2025) apporte des améliorations majeures :

### ⚡ Performance
- **I/O asynchrone (AIO)** - Jusqu'à 3× plus rapide sur les opérations I/O  
- **Skip Scan optimization** - Index multi-colonnes plus efficaces  
- **Auto-élimination des self-joins** - Optimisations automatiques du planificateur

### 🔒 Sécurité
- **OAuth 2.0** - Authentification moderne native  
- **SCRAM passthrough** - Avec postgres_fdw et dblink  
- **Mode FIPS** - Configuration TLS 1.3 (ssl_tls13_ciphers)  
- **Data Checksums** - Activés par défaut

### 🛠️ Fonctionnalités
- **UUIDv7** - Nouveau type UUID avec timestamp  
- **Colonnes virtuelles** - Generated columns sans stockage  
- **Contraintes temporelles** - Validation sur périodes  
- **OLD/NEW dans RETURNING** - Plus de flexibilité DML

### 🔧 Administration
- **pg_upgrade amélioré** - Option --swap, préservation statistiques  
- **Autovacuum dynamique** - Ajustements automatiques  
- **Statistiques étendues** - I/O et WAL par backend

👉 Consultez l'[annexe dédiée](annexes/nouveautes-pg18/) pour tous les détails

---

## 🎓 Public cible

Cette formation s'adresse à :

- 👨‍💻 **Développeurs** - Développer leurs compétences en SQL et concevoir des schémas performants  
- 🔧 **DevOps/SRE** - Déployer, monitorer, optimiser PostgreSQL en production  
- 🛡️ **DBA** - Administrer, sécuriser, maintenir des instances PostgreSQL  
- 🏗️ **Architectes** - Concevoir des architectures HA et cloud-native  
- 🎓 **Étudiants** - Apprendre un SGBD professionnel de A à Z

**Approche pédagogique :** Progressive, théorique, axée sur la compréhension en profondeur.

---

## ❓ FAQ

**Q : Cette formation contient-elle des exercices pratiques ?**
R : Non, c'est une formation théorique. Elle fournit les concepts, à vous de pratiquer !

**Q : Dois-je connaître PostgreSQL avant de commencer ?**
R : Non, la formation part de zéro. Des bases en SQL sont un plus.

**Q : Est-ce applicable aux versions antérieures ?**
R : Oui, 90% du contenu s'applique à PG 12-17. Les nouveautés PG 18 sont clairement marquées 🆕

**Q : Puis-je utiliser cette formation pour enseigner ?**
R : Oui, sous licence CC BY-NC-SA 4.0. Attribution obligatoire, usage commercial interdit.

**Q : PostgreSQL 18 est-il stable ?**
R : PostgreSQL 18 sort en septembre 2025. Cette formation anticipe les fonctionnalités annoncées.

---

## 📝 Licence

Cette formation est sous licence **CC BY-NC-SA 4.0** (Creative Commons Attribution - Pas d'Utilisation Commerciale - Partage dans les Mêmes Conditions 4.0 International).

✅ **Vous êtes libre de :**
- Partager et redistribuer
- Adapter et transformer le contenu
- À des fins non commerciales uniquement

🔒 **Conditions :**
- Attribution requise
- Pas d'usage commercial
- Partage dans les mêmes conditions

**Attribution suggérée :**
```
Formation PostgreSQL 18 par Nicolas DEOUX  
https://github.com/NDXDeveloper/formation-postgresql-18  
Licence CC BY-NC-SA 4.0  
```

👉 Voir le fichier [LICENSE](LICENSE) pour les détails complets.

---

## 👨‍💻 Auteur

**Nicolas DEOUX**

Passionné par les bases de données et l'architecture logicielle, je partage mes connaissances à travers des formations techniques détaillées et accessibles.

- 📧 [NDXDev@gmail.com](mailto:NDXDev@gmail.com)  
- 💼 [LinkedIn](https://www.linkedin.com/in/nicolas-deoux-ab295980/)  
- 🐙 [GitHub](https://github.com/NDXDeveloper)

---

## 🙏 Remerciements

Un grand merci à :

- 🐘 La **communauté PostgreSQL** pour ce SGBD exceptionnel  
- 📚 Les **contributeurs** de la documentation officielle  
- 🌍 L'écosystème **open source** qui inspire ce travail  
- 🎓 **Vous** qui prenez le temps d'apprendre PostgreSQL !

**Ressources complémentaires :**
[Documentation officielle](https://www.postgresql.org/docs/) • [PostgreSQL Wiki](https://wiki.postgresql.org/) • [Planet PostgreSQL](https://planet.postgresql.org/) • [PGConf](https://www.postgresql.org/about/events/)

---

<div align="center">

**🐘 Bon apprentissage avec PostgreSQL 18 ! 🚀**

[![Star on GitHub](https://img.shields.io/github/stars/NDXDeveloper/formation-postgresql-18?style=social)](https://github.com/NDXDeveloper/formation-postgresql-18)
[![Follow](https://img.shields.io/github/followers/NDXDeveloper?style=social)](https://github.com/NDXDeveloper)

**[⬆ Retour en haut](#-formation-complète-postgresql-18)**

*Dernière mise à jour : Novembre 2025*
*Version PostgreSQL : 18 (Septembre 2025)*

</div>
