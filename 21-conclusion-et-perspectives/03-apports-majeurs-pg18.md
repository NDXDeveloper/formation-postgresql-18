🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.3. Les Apports Majeurs de PostgreSQL 18

## Introduction

Le **25 septembre 2025**, la communauté PostgreSQL a officiellement publié **PostgreSQL 18**, marquant une étape importante dans l'évolution de la base de données open-source la plus avancée au monde. Cette version majeure apporte des améliorations significatives qui touchent à la fois les performances, l'expérience développeur et les capacités opérationnelles.

PostgreSQL 18 n'est pas une simple mise à jour incrémentale : c'est une version qui repense certains fondamentaux de l'architecture du moteur de base de données, tout en facilitant la vie des équipes de développement et d'exploitation.

---

## Contexte et Philosophie de la Version 18

### Un Cycle de Développement Ambitieux

Comme chaque version majeure de PostgreSQL, la version 18 est le fruit d'environ **un an de développement intensif** par une communauté mondiale de contributeurs. Cette version se distingue par :

- **Des changements architecturaux profonds** : Le sous-système d'I/O a été repensé
- **Une modernisation de la sécurité** : Adoption des standards d'authentification actuels
- **Une amélioration de l'expérience opérationnelle** : Mises à jour majeures simplifiées
- **Des fonctionnalités développeur attendues** : UUIDv7, colonnes virtuelles, et plus

### Les Grands Axes de Cette Version

PostgreSQL 18 s'articule autour de quatre axes majeurs d'amélioration :

| Axe | Objectif | Impact |
|-----|----------|--------|
| **Performance I/O** | Exploiter le matériel moderne | Jusqu'à 3× plus rapide |
| **Productivité Développeur** | Simplifier le code applicatif | Moins de code, plus d'efficacité |
| **Sécurité Moderne** | S'intégrer aux systèmes SSO | Authentification OAuth 2.0 |
| **Opérations Simplifiées** | Réduire les temps d'arrêt | Mises à jour quasi-instantanées |

---

## Vue d'Ensemble des Nouveautés Majeures

### Performance : L'I/O Asynchrone

PostgreSQL a historiquement utilisé un modèle d'**I/O synchrone** : chaque lecture de données bloquait le processus en attendant la réponse du disque. Ce modèle, bien que simple et robuste, ne tirait pas pleinement parti des capacités des systèmes de stockage modernes.

PostgreSQL 18 introduit un **sous-système d'I/O asynchrone** qui permet au moteur de soumettre plusieurs requêtes de lecture simultanément, puis de continuer à travailler pendant que les données arrivent. Les benchmarks montrent des gains de performance pouvant atteindre **2 à 3 fois** les performances des versions précédentes pour les charges de travail intensives en lecture.

Cette amélioration est particulièrement bénéfique pour :
- Les parcours séquentiels de grandes tables
- Les opérations de maintenance (VACUUM)
- Les environnements cloud où la latence de stockage est significative

### Développeur : UUIDv7 et Colonnes Virtuelles

PostgreSQL 18 apporte deux fonctionnalités très attendues par les développeurs :

**UUIDv7** : Une nouvelle génération d'identifiants uniques qui combinent :
- L'unicité globale des UUID traditionnels
- L'ordre chronologique des identifiants séquentiels
- Des performances d'indexation optimales

**Colonnes Virtuelles** : Des colonnes calculées à la demande qui :
- N'occupent aucun espace disque
- Sont calculées lors de la lecture
- S'ajoutent instantanément aux tables existantes

Ces fonctionnalités permettent d'écrire moins de code applicatif tout en bénéficiant de meilleures performances.

### Sécurité : OAuth 2.0 et Standards Modernes

La sécurité de PostgreSQL 18 fait un bond en avant avec :

**Support natif OAuth 2.0** : PostgreSQL peut maintenant s'intégrer directement aux fournisseurs d'identité modernes (Google, Azure AD, Okta, Keycloak...), permettant :
- L'authentification Single Sign-On (SSO)
- L'utilisation de tokens temporaires plutôt que de mots de passe
- Une gestion centralisée des identités

**Dépréciation de MD5** : L'algorithme MD5, considéré comme obsolète depuis des années, est officiellement déprécié au profit de SCRAM-SHA-256. Cette transition renforce la sécurité des mots de passe stockés.

**Améliorations TLS** : Un nouveau paramètre `ssl_tls13_ciphers` permet un contrôle précis des suites de chiffrement TLS 1.3.

### Opérations : Mises à Jour Simplifiées

L'un des points de friction majeurs lors des mises à jour de PostgreSQL était la **perte des statistiques de l'optimiseur**, entraînant des performances dégradées pendant des heures après la migration.

PostgreSQL 18 résout ce problème avec :

**Préservation des statistiques** : Les statistiques du planificateur sont automatiquement transférées lors des mises à jour, permettant des performances optimales dès le redémarrage.

**Option --swap** : Une nouvelle méthode de transfert qui échange les répertoires de données au lieu de les copier, réduisant le temps de migration à quelques secondes.

**Vérifications parallèles** : L'option `--jobs` permet d'exécuter les vérifications de compatibilité en parallèle, accélérant la phase de préparation.

---

## Autres Améliorations Notables

Au-delà des quatre axes majeurs, PostgreSQL 18 inclut de nombreuses autres améliorations :

### Performances du Planificateur

| Amélioration | Description |
|--------------|-------------|
| **Skip Scan** | Les index multi-colonnes sont utilisables même sans condition sur les premières colonnes |
| **OR → ANY** | Les clauses OR sont automatiquement transformées pour utiliser les index |
| **Self-Join Elimination** | Les auto-jointures inutiles sont automatiquement supprimées |

### Nouvelles Fonctionnalités SQL

| Fonctionnalité | Description |
|----------------|-------------|
| **OLD/NEW dans RETURNING** | Accès aux valeurs avant et après modification |
| **Contraintes Temporelles** | PRIMARY KEY et UNIQUE avec `WITHOUT OVERLAPS` |
| **MERGE amélioré** | Support complet de OLD et NEW |

### Monitoring et Observabilité

| Amélioration | Description |
|--------------|-------------|
| **EXPLAIN amélioré** | Affichage automatique des buffers, statistiques CPU et WAL |
| **pg_stat_all_tables** | Temps passé en VACUUM et ANALYZE |
| **Statistiques I/O par backend** | Visibilité sur l'activité I/O de chaque connexion |

### Administration

| Changement | Description |
|------------|-------------|
| **Checksums par défaut** | Activés automatiquement sur les nouveaux clusters |
| **Protocole Wire 3.2** | Première mise à jour du protocole depuis 2003 |
| **Autovacuum amélioré** | Gel proactif des pages, nouveau paramètre `autovacuum_vacuum_max_threshold` |

---

## Impact sur les Différents Profils

### Pour les Développeurs

PostgreSQL 18 simplifie le développement avec :
- `uuidv7()` pour des clés primaires performantes et distribuées
- Les colonnes virtuelles pour les calculs automatiques
- `OLD`/`NEW` dans `RETURNING` pour réduire le nombre de requêtes
- De nouvelles fonctions de manipulation de tableaux (`array_sort`, `array_reverse`)

### Pour les DBAs et DevOps

Les opérations deviennent plus fluides grâce à :
- Des mises à jour majeures quasi-instantanées
- Une préservation des statistiques qui élimine la phase d'ANALYZE post-upgrade
- Un monitoring enrichi avec plus de métriques
- Des checksums par défaut pour détecter la corruption

### Pour les Architectes Sécurité

La conformité et la sécurité sont renforcées par :
- L'intégration OAuth 2.0 native
- La migration facilitée vers SCRAM-SHA-256
- Le support du mode FIPS
- Un contrôle granulaire des suites TLS 1.3

---

## Compatibilité et Migration

### Changements de Comportement

Quelques changements peuvent affecter les applications existantes :

| Changement | Impact | Action |
|------------|--------|--------|
| **Checksums par défaut** | Nouveaux clusters avec checksums activés | Utiliser `--no-data-checksums` si nécessaire |
| **MD5 déprécié** | Avertissements lors de l'utilisation | Migrer vers SCRAM-SHA-256 |
| **Full-text search** | Utilise le collation provider par défaut | Réindexer si nécessaire |

### Versions Supportées pour la Migration

PostgreSQL 18 peut être mis à jour depuis :
- PostgreSQL 9.2 et versions ultérieures (via pg_upgrade)
- Toute version via dump/restore

La préservation des statistiques fonctionne quelle que soit la version source.

---

## Structure de Ce Chapitre

Les sections suivantes détaillent chacun des quatre apports majeurs de PostgreSQL 18 :

| Section | Contenu |
|---------|---------|
| **21.3.1** | I/O Asynchrone — Architecture, configuration, et gains de performance |
| **21.3.2** | UUIDv7 et Colonnes Virtuelles — Nouvelles fonctionnalités développeur |
| **21.3.3** | OAuth et Sécurité Moderne — Authentification et chiffrement |
| **21.3.4** | Upgrade Simplifié — Préservation des statistiques et options pg_upgrade |

Chaque section est conçue pour être accessible aux débutants tout en fournissant les détails techniques nécessaires à une mise en œuvre en production.

---

## Conclusion

PostgreSQL 18 représente une évolution majeure qui positionne PostgreSQL pour les défis des années à venir :

- **Performance** : L'I/O asynchrone exploite enfin le potentiel des systèmes de stockage modernes
- **Productivité** : UUIDv7 et les colonnes virtuelles réduisent le code applicatif nécessaire
- **Sécurité** : OAuth 2.0 et SCRAM-SHA-256 alignent PostgreSQL sur les standards actuels
- **Opérations** : Les mises à jour majeures ne sont plus synonymes d'indisponibilité prolongée

Cette version confirme la capacité de la communauté PostgreSQL à innover tout en préservant la stabilité et la compatibilité qui font la réputation de ce SGBD.

---


⏭️ [I/O asynchrone (jusqu'à 3× plus rapide)](/21-conclusion-et-perspectives/03.1-io-asynchrone.md)
