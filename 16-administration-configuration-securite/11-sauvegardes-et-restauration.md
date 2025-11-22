🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.11. Sauvegardes et Restauration

## Introduction : Pourquoi les Sauvegardes Sont Critiques

Imaginez ce scénario : il est 10h30 un mardi matin. Votre entreprise fonctionne normalement, les commandes clients s'accumulent, les transactions sont enregistrées dans votre base de données PostgreSQL. Soudain, à 10h37, un développeur exécute par erreur une commande DELETE sans clause WHERE, supprimant des milliers d'enregistrements clients. Ou pire, à 14h00, un disque dur tombe en panne, emportant avec lui toute votre base de données.

**Question** : Pouvez-vous récupérer vos données ? En combien de temps ? Avec quelle perte ?

La réponse à ces questions dépend entièrement de votre **stratégie de sauvegarde**.

### Les Sauvegardes : Une Police d'Assurance

Les sauvegardes sont comme une assurance :
- On espère ne jamais en avoir besoin
- On paie un coût (temps, stockage, ressources)
- Mais quand un sinistre arrive, elles peuvent sauver l'entreprise

**Statistiques réelles** :
- 60% des entreprises ayant perdu leurs données ferment dans les 6 mois
- 93% des entreprises sans plan de sauvegarde ayant subi une perte majeure de données font faillite dans l'année
- Le coût moyen d'une heure d'indisponibilité : 100,000€ à 1,000,000€ selon le secteur

### Les Risques Contre Lesquels On Se Protège

Les sauvegardes protègent contre de multiples menaces :

#### 1. Pannes Matérielles 💾
```
Disque dur défaillant
Contrôleur RAID en panne
Serveur qui ne démarre plus
Incendie dans le datacenter
```

**Fréquence** : Plus courante qu'on ne le pense (MTBF des disques)

#### 2. Erreurs Humaines 👤
```
DELETE sans WHERE
UPDATE avec mauvaise condition
DROP TABLE par erreur
Migration SQL mal testée
```

**Fréquence** : Cause n°1 de perte de données (40% des incidents)

#### 3. Corruptions Logicielles 🐛
```
Bug dans PostgreSQL (rare mais possible)
Corruption mémoire
Problème système de fichiers
Crash durant une écriture
```

**Fréquence** : Rare mais impactante

#### 4. Attaques Malveillantes 🦠
```
Ransomware chiffrant les données
SQL Injection avec DROP TABLE
Sabotage d'un employé mécontent
Intrusion réseau
```

**Fréquence** : En augmentation constante (+300% en 5 ans)

#### 5. Catastrophes Naturelles 🌊
```
Inondation
Incendie
Tremblement de terre
Panne électrique prolongée
```

**Fréquence** : Faible mais impact total

---

## Concepts Fondamentaux

Avant de plonger dans les techniques spécifiques de sauvegarde PostgreSQL, il est essentiel de comprendre quelques concepts clés qui guideront toutes vos décisions.

### RTO : Recovery Time Objective (Objectif de Temps de Reprise)

Le **RTO** est le temps maximum acceptable pendant lequel votre système peut être indisponible après un incident.

**Question** : Combien de temps votre entreprise peut-elle survivre sans sa base de données ?

**Exemples** :

| Type d'entreprise | RTO acceptable | Implication |
|-------------------|---------------|-------------|
| Blog personnel | 24-48 heures | Sauvegarde quotidienne suffit |
| Site e-commerce PME | 2-4 heures | Sauvegarde fréquente + procédure claire |
| Banque en ligne | < 15 minutes | Haute disponibilité + réplication |
| Service d'urgence | < 1 minute | Réplication synchrone + failover automatique |

**Impact sur la stratégie** :
- RTO élevé (24h+) → Sauvegardes simples, quotidiennes
- RTO moyen (1-4h) → Sauvegardes fréquentes + documentation
- RTO faible (<15min) → Réplication + automatisation
- RTO très faible (<1min) → Architecture haute disponibilité

### RPO : Recovery Point Objective (Objectif de Point de Reprise)

Le **RPO** est la quantité maximale de données que votre entreprise peut se permettre de perdre, mesurée en temps.

**Question** : Combien de transactions pouvez-vous vous permettre de perdre ?

**Exemples** :

```
RPO = 24 heures
→ Perte acceptable : toutes les transactions de la dernière journée
→ Solution : Sauvegarde quotidienne nocturne

RPO = 1 heure
→ Perte acceptable : 1 heure de transactions
→ Solution : Sauvegarde toutes les heures + archives WAL

RPO = 5 minutes
→ Perte acceptable : 5 minutes de transactions
→ Solution : Archivage WAL continu + réplication

RPO = 0 seconde (aucune perte)
→ Perte acceptable : aucune
→ Solution : Réplication synchrone
```

**Calcul du RPO** :

Pour une boutique en ligne :
```
Transactions moyennes par minute : 10
Valeur moyenne d'une transaction : 50€
RPO de 1 heure = 10 × 60 × 50€ = 30,000€ de perte potentielle
```

**Impact sur la stratégie** :
- RPO élevé (24h+) → Sauvegardes quotidiennes suffisent
- RPO moyen (1-4h) → Sauvegardes fréquentes
- RPO faible (<15min) → Archivage WAL continu
- RPO très faible (<1min) → Réplication streaming

### Relation RTO/RPO et Coût

Il existe une relation directe entre vos objectifs RTO/RPO et le coût de votre infrastructure de sauvegarde :

```
Coût
 ↑
 │                                    ╱
 │                                 ╱
 │                              ╱
 │                           ╱
 │                        ╱
 │                     ╱
 │                  ╱
 │               ╱
 │            ╱
 │         ╱
 │      ╱
 │   ╱
 └────────────────────────────────────────→ RTO/RPO
  24h    8h     2h    1h   15min  1min   0

Courbe exponentielle : réduire RTO/RPO de moitié peut doubler le coût
```

**Principe** : Il faut trouver le bon équilibre entre :
- La criticité de vos données (impact business d'une perte)
- Le budget disponible
- La complexité acceptable

---

## Types de Sauvegardes PostgreSQL

PostgreSQL offre plusieurs approches de sauvegarde, chacune avec ses avantages et cas d'usage.

### Vue d'Ensemble

```
┌───────────────────────────────────────────────────────────┐
│                 SAUVEGARDES POSTGRESQL                    │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌────────────────────┐         ┌────────────────────┐    │
│  │   LOGIQUES         │         │    PHYSIQUES       │    │
│  ├────────────────────┤         ├────────────────────┤    │
│  │ • pg_dump          │         │ • pg_basebackup    │    │
│  │ • pg_dumpall       │         │ • File system copy │    │
│  │                    │         │ • Snapshots        │    │
│  │ Format: SQL        │         │ Format: Binaire    │    │
│  │ Granularité: Fine  │         │ Granularité: Totale│    │
│  │ Portabilité: +++   │         │ Rapidité: +++      │    │
│  └────────────────────┘         └────────────────────┘    │
│                                                           │
│  ┌─────────────────────────────────────────────────┐      │
│  │           ARCHIVAGE CONTINU (WAL)               │      │
│  ├─────────────────────────────────────────────────┤      │
│  │ • Write-Ahead Logs archivés                     │      │
│  │ • Permet Point-In-Time Recovery (PITR)          │      │
│  │ • Complément aux sauvegardes de base            │      │
│  └─────────────────────────────────────────────────┘      │
│                                                           │
│  ┌─────────────────────────────────────────────────┐      │
│  │           RÉPLICATION                           │      │
│  ├─────────────────────────────────────────────────┤      │
│  │ • Streaming Replication                         │      │
│  │ • Logical Replication                           │      │
│  │ • Haute disponibilité + Backup                  │      │
│  └─────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────┘
```

### 1. Sauvegardes Logiques

**Concept** : Exporter les données sous forme de commandes SQL

**Outils** : `pg_dump`, `pg_dumpall`

**Avantages** :
- ✅ Portabilité maximale (entre versions, OS, architectures)
- ✅ Granularité fine (une table, un schéma, une base)
- ✅ Lisible et éditable (format SQL)
- ✅ Flexibilité de restauration
- ✅ Idéal pour migrations et clonage

**Inconvénients** :
- ❌ Plus lent pour grandes bases de données
- ❌ Restauration lente (rejeu des commandes SQL)
- ❌ Pas de PITR natif
- ❌ Cohérence limitée entre bases (pg_dumpall)

**Cas d'usage typiques** :
- Migration vers nouvelle version PostgreSQL
- Clonage d'environnement (prod → dev)
- Backup de bases de données < 100 GB
- Export pour analyse externe
- Sauvegarde d'une table ou schéma spécifique

**Exemple conceptuel** :
```sql
-- Ce qui est sauvegardé
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255)
);

INSERT INTO clients VALUES (1, 'Dupont', 'dupont@example.com');
INSERT INTO clients VALUES (2, 'Martin', 'martin@example.com');
-- ... etc
```

### 2. Sauvegardes Physiques

**Concept** : Copier directement les fichiers binaires de PostgreSQL

**Outils** : `pg_basebackup`, snapshots, copies filesystem

**Avantages** :
- ✅ Très rapide pour grandes bases de données
- ✅ Restauration rapide (copie de fichiers)
- ✅ Base pour PITR
- ✅ Efficace pour bases volumineuses (To)
- ✅ Cohérence garantie du cluster entier

**Inconvénients** :
- ❌ Dépendance à la version majeure PostgreSQL
- ❌ Dépendance à l'architecture (x86_64, ARM)
- ❌ Pas de restauration partielle (tout ou rien)
- ❌ Moins portable
- ❌ Fichiers non lisibles directement

**Cas d'usage typiques** :
- Backup de production pour grandes bases
- Base pour haute disponibilité
- Disaster recovery
- Clonage rapide de serveur
- Setup de serveur standby

**Exemple conceptuel** :
```
Ce qui est copié :
/var/lib/postgresql/data/
├── base/           # Fichiers de données
├── global/         # Objets globaux
├── pg_wal/         # Write-Ahead Logs
├── pg_tblspc/      # Tablespaces
├── postgresql.conf # Configuration
└── ...
```

### 3. Archivage WAL (Write-Ahead Logs)

**Concept** : Sauvegarder continuellement les journaux de transactions

**Mécanisme** : `archive_command` copie automatiquement les fichiers WAL

**Avantages** :
- ✅ Point-In-Time Recovery (PITR)
- ✅ RPO très faible (secondes)
- ✅ Complète les sauvegardes physiques
- ✅ Récupération granulaire dans le temps
- ✅ Protection contre erreurs logiques

**Inconvénients** :
- ❌ Nécessite sauvegarde de base (pg_basebackup)
- ❌ Gestion de l'espace disque
- ❌ Configuration plus complexe
- ❌ Restauration peut être longue

**Cas d'usage typiques** :
- Environnements de production critiques
- Quand RPO < 15 minutes requis
- Protection contre DELETE/UPDATE accidentels
- Récupération après ransomware
- Audit trail complet

**Exemple conceptuel** :
```
Timeline du PITR :

Base Backup         Incident            Point de restauration désiré
    |                   |                            |
    v                   v                            v
[10:00] ──WAL──WAL──WAL──[14:37]──WAL──WAL──────[14:36:59]
        ↑                 ↑                         ↑
    Snapshot         Suppression              On restaure ICI
    cohérent         accidentelle          (1 seconde avant)
```

### 4. Réplication

**Concept** : Copie en temps réel vers un autre serveur PostgreSQL

**Types** :
- **Streaming Replication** : Réplication physique
- **Logical Replication** : Réplication sélective

**Avantages** :
- ✅ Haute disponibilité (HA)
- ✅ Failover rapide (secondes)
- ✅ RPO très faible (< 1 seconde)
- ✅ Possibilité de lecture sur standby
- ✅ Protection géographique

**Inconvénients** :
- ❌ Coût (serveur supplémentaire)
- ❌ Complexité de configuration
- ❌ Pas de protection contre erreurs logiques
- ❌ Nécessite compléments (backups traditionnels)

**Cas d'usage typiques** :
- Haute disponibilité (uptime > 99.9%)
- Disaster recovery géographique
- Load balancing des lectures
- Zero-downtime migrations
- RTO < 1 minute

---

## Comparaison des Approches

### Tableau Comparatif

| Critère | Logique (pg_dump) | Physique (pg_basebackup) | WAL Archiving | Réplication |
|---------|-------------------|--------------------------|---------------|-------------|
| **Vitesse backup** | Lente | Rapide | Continue | Temps réel |
| **Vitesse restore** | Lente | Rapide | Moyenne | Instantanée |
| **Taille fichier** | Moyenne | Grande | Petite (par fichier) | N/A |
| **Portabilité** | Excellente | Limitée | Limitée | Limitée |
| **Granularité** | Fine | Globale | Globale | Globale |
| **PITR** | Non | Non | **Oui** | Non |
| **Coût stockage** | Moyen | Élevé | Moyen | Élevé |
| **Complexité** | Faible | Moyenne | Élevée | Élevée |
| **RPO** | Heures | Heures | Minutes/Secondes | Secondes |
| **RTO** | Heures | Minutes | Minutes/Heures | Secondes |
| **HA** | Non | Non | Non | **Oui** |

### Quelle Approche Choisir ?

**Il n'y a pas de "meilleure" approche universelle**. Le choix dépend de :

#### Critère 1 : Taille de la Base de Données

```
< 10 GB        → pg_dump suffisant
10-100 GB      → pg_dump ou pg_basebackup
100-500 GB     → pg_basebackup recommandé
> 500 GB       → pg_basebackup + WAL archiving
> 1 TB         → pg_basebackup + WAL + Réplication
```

#### Critère 2 : Criticité Business

```
Blog personnel               → pg_dump quotidien
Application interne PME      → pg_dump + WAL archiving
Site e-commerce              → pg_basebackup + WAL + Réplication asynchrone
Application bancaire         → pg_basebackup + WAL + Réplication synchrone
Service vie/mort (urgences)  → Multi-site + Réplication + Automatisation complète
```

#### Critère 3 : Budget

```
Budget minimal        → pg_dump + stockage local
Budget moyen         → pg_dump + cloud storage
Budget confortable   → pg_basebackup + WAL + cloud
Budget élevé         → Tout : backups multiples + réplication + multi-site
```

#### Critère 4 : Compétences Équipe

```
Débutant             → Commencer par pg_dump simple
Intermédiaire        → pg_basebackup + WAL basique
Avancé               → Architecture complète avec automatisation
Expert               → Solutions custom + outils tiers (Barman, pgBackRest)
```

---

## Stratégie de Sauvegarde Recommandée

### Approche "Defense in Depth" (Défense en Profondeur)

La meilleure stratégie combine **plusieurs** méthodes complémentaires.

#### Niveau 1 : Protection de Base (Minimum Vital)

```
Sauvegardes quotidiennes avec pg_dump
+ Rétention 7 jours
+ Stockage local sécurisé
+ Test de restauration mensuel

Coût : Minimal
Protection : Pannes matérielles, erreurs récentes
RPO : 24 heures
RTO : 2-4 heures
```

**Pour qui** : Petites structures, applications non critiques

#### Niveau 2 : Protection Standard (PME)

```
Sauvegardes quotidiennes avec pg_basebackup
+ Archivage WAL continu
+ Rétention 30 jours
+ Stockage local + cloud
+ Test de restauration hebdomadaire

Coût : Modéré
Protection : + PITR, erreurs récentes récupérables
RPO : 5-15 minutes
RTO : 30 minutes - 2 heures
```

**Pour qui** : PME, applications business importantes

#### Niveau 3 : Haute Disponibilité (Entreprise)

```
Sauvegardes quotidiennes avec pg_basebackup
+ Archivage WAL continu
+ Réplication streaming vers standby
+ Stockage multi-site (local + distant + cloud)
+ Automatisation complète
+ Tests automatisés
+ Monitoring avancé

Coût : Élevé
Protection : Complète (HA + DR + PITR)
RPO : < 1 minute
RTO : < 15 minutes
```

**Pour qui** : Entreprises, applications critiques

#### Niveau 4 : Mission Critique (Finance, Santé)

```
Architecture multi-site
+ Réplication synchrone
+ Backup automatisé depuis standby
+ Archivage WAL multi-destination
+ Réplication cloud multi-région
+ Failover automatique (Patroni)
+ Tests continus
+ Monitoring temps réel
+ Documentation runbooks

Coût : Très élevé
Protection : Maximale (99.99%+ uptime)
RPO : 0 (aucune perte)
RTO : < 1 minute (automatique)
```

**Pour qui** : Banques, hôpitaux, services critiques

---

## Éléments Essentiels d'une Stratégie

Quelle que soit votre approche, certains éléments sont **indispensables** :

### 1. Automatisation

**Principe** : Les sauvegardes manuelles ne sont PAS fiables.

❌ **Mauvais** :
```
"Je fais une sauvegarde chaque vendredi avant de partir"
→ Oublis inévitables
→ Dépendance à une personne
→ Pas de sauvegarde en son absence
```

✅ **Bon** :
```bash
# Cron exécute automatiquement
0 2 * * * /usr/local/bin/backup_postgresql.sh
```

### 2. Tests Réguliers

**Principe** : Une sauvegarde non testée = pas de sauvegarde

❌ **Mauvais** :
```
"J'ai des sauvegardes depuis 2 ans, mais je n'ai jamais testé"
→ Découverte lors d'un incident réel que les sauvegardes sont corrompues
→ Catastrophe
```

✅ **Bon** :
```
Tests mensuels :
1. Restaurer sur serveur de test
2. Vérifier intégrité des données
3. Mesurer le temps de restauration
4. Documenter la procédure
```

### 3. Monitoring et Alertes

**Principe** : Détecter les échecs immédiatement

❌ **Mauvais** :
```
Aucune alerte configurée
→ Découverte 1 semaine plus tard que les sauvegardes échouent
→ Aucune sauvegarde valide disponible en cas d'incident
```

✅ **Bon** :
```
Alertes configurées :
• Échec de sauvegarde → Email + SMS immédiat
• Espace disque < 20% → Alerte préventive
• Sauvegarde > 2 jours → Alerte critique
• WAL archiving échoue → Alerte immédiate
```

### 4. Documentation

**Principe** : En cas de crise, pas le temps de chercher

❌ **Mauvais** :
```
Procédures dans la tête de l'admin
→ Admin en vacances lors de l'incident
→ Panique, perte de temps
```

✅ **Bon** :
```
Runbook accessible 24/7 :
1. Où sont les sauvegardes ?
2. Comment restaurer (étapes précises) ?
3. Qui contacter en cas de problème ?
4. Temps de restauration estimé ?
5. Procédures d'escalade
```

### 5. Sécurité

**Principe** : Protéger les sauvegardes elles-mêmes

❌ **Mauvais** :
```
Sauvegardes non chiffrées sur NAS accessible par tous
→ Vol de données
→ Ransomware peut chiffrer aussi les backups
```

✅ **Bon** :
```
Sécurité des sauvegardes :
• Chiffrement at-rest et in-transit
• Accès restreint (ACL, IAM)
• Immutabilité (S3 Object Lock)
• Audit logs activés
• MFA sur accès cloud
```

### 6. Redondance (Règle 3-2-1)

**Principe** : Ne jamais avoir une seule copie

❌ **Mauvais** :
```
1 copie : Production
1 backup : Disque externe dans la même salle
→ Incendie = tout perdu
```

✅ **Bon** :
```
Règle 3-2-1 :
• 3 copies de données minimum
• 2 supports différents
• 1 copie hors-site (off-site)
```

---

## Planification de Votre Stratégie

### Questions à Se Poser

Avant de mettre en place votre stratégie de sauvegarde, répondez à ces questions :

#### 1. Criticité

- [ ] Quel est l'impact business d'une indisponibilité de 1 heure ? 1 jour ?
- [ ] Quelle quantité de données pouvons-nous nous permettre de perdre ?
- [ ] Y a-t-il des contraintes légales/réglementaires (RGPD, HIPAA, PCI-DSS) ?
- [ ] Quel est notre SLA vis-à-vis des clients ?

#### 2. Ressources

- [ ] Quel est notre budget annuel pour les sauvegardes ?
- [ ] Combien d'espace de stockage avons-nous disponible ?
- [ ] Quel est le niveau de compétence de l'équipe ?
- [ ] Avons-nous du temps pour maintenir une solution complexe ?

#### 3. Données

- [ ] Quelle est la taille actuelle de notre base de données ?
- [ ] Quel est le taux de croissance (par mois/an) ?
- [ ] Quelle est la fréquence de modification des données ?
- [ ] Y a-t-il des pics d'activité (heures, jours) ?

#### 4. Infrastructure

- [ ] Où sont nos serveurs (on-premise, cloud, hybride) ?
- [ ] Avons-nous accès à plusieurs sites/datacenters ?
- [ ] Quelle est la bande passante réseau disponible ?
- [ ] Avons-nous déjà des solutions de stockage (NAS, SAN, cloud) ?

### Matrice de Décision

```
┌────────────────────────────────────────────────────────────┐
│         MATRICE DE SÉLECTION DE STRATÉGIE                  │
├──────────────┬─────────────────────────────────────────────┤
│ SI...        │ ALORS...                                    │
├──────────────┼─────────────────────────────────────────────┤
│ DB < 50 GB   │ → pg_dump quotidien                         │
│ Budget min   │ → + Stockage local                          │
│ RPO = 24h    │ → + Cloud hebdomadaire                      │
├──────────────┼─────────────────────────────────────────────┤
│ DB 50-500 GB │ → pg_basebackup quotidien                   │
│ Budget moyen │ → + WAL archiving                           │
│ RPO = 1h     │ → + Cloud + Local                           │
├──────────────┼─────────────────────────────────────────────┤
│ DB > 500 GB  │ → pg_basebackup + parallélisation           │
│ Budget élevé │ → + WAL archiving continu                   │
│ RPO < 15min  │ → + Réplication standby                     │
│ RTO < 1h     │ → + Multi-site                              │
└──────────────┴─────────────────────────────────────────────┘
```

---

## Roadmap de Mise en Place

### Phase 1 : Mise en Place Immédiate (Semaine 1)

**Objectif** : Avoir AU MINIMUM une sauvegarde fonctionnelle

```
Jour 1-2 : Configuration pg_dump basique
- Script de sauvegarde simple
- Test de restauration manuelle
- Cron quotidien

Jour 3-4 : Monitoring basique
- Vérifier succès/échec
- Alerte email simple
- Documentation minimale

Jour 5 : Test
- Restauration complète
- Mesure du temps
- Validation des données
```

**Livrable** : Protection de base fonctionnelle

### Phase 2 : Amélioration (Semaine 2-4)

**Objectif** : Réduire RPO/RTO, ajouter redondance

```
Semaine 2 : Sauvegarde physique
- Configuration pg_basebackup
- Compression optimisée
- Rotation automatique

Semaine 3 : WAL Archiving
- Configuration archive_command
- Tests PITR
- Documentation procédure

Semaine 4 : Off-site
- Configuration cloud (S3/Azure/GCS)
- Automatisation upload
- Tests de récupération
```

**Livrable** : Stratégie 3-2-1 fonctionnelle

### Phase 3 : Optimisation (Mois 2-3)

**Objectif** : Automatisation complète, HA

```
Mois 2 : Automatisation avancée
- Scripts robustes avec gestion erreurs
- Monitoring complet (Grafana)
- Tests automatisés

Mois 3 : Haute disponibilité (si besoin)
- Configuration réplication
- Failover automatique (Patroni)
- Procédures d'urgence
```

**Livrable** : Architecture production-ready

### Phase 4 : Maintenance Continue

**Objectif** : Maintenir et améliorer

```
Mensuel :
- Tests de restauration
- Revue des métriques
- Ajustement rétention

Trimestriel :
- Drill disaster recovery
- Audit de sécurité
- Mise à jour documentation

Annuel :
- Revue complète stratégie
- Évaluation coûts/bénéfices
- Formation équipe
```

---

## Structure de Ce Chapitre

Ce chapitre 16.11 est organisé pour vous guider progressivement à travers tous les aspects des sauvegardes PostgreSQL :

### 16.11.1. Sauvegardes Logiques (pg_dump, pg_dumpall)
- **Ce que vous apprendrez** : Comment utiliser pg_dump et pg_dumpall
- **Niveau** : Débutant à Intermédiaire
- **Cas d'usage** : Sauvegardes flexibles, migrations, clonage

### 16.11.2. Sauvegardes Physiques (pg_basebackup)
- **Ce que vous apprendrez** : Sauvegardes complètes rapides
- **Niveau** : Intermédiaire
- **Cas d'usage** : Production, grandes bases, HA

### 16.11.3. Point-In-Time Recovery (PITR) et WAL Archiving
- **Ce que vous apprendrez** : Récupération à un instant précis
- **Niveau** : Avancé
- **Cas d'usage** : Protection contre erreurs logiques, audit

### 16.11.4. Stratégies de Sauvegarde 3-2-1
- **Ce que vous apprendrez** : Orchestration complète
- **Niveau** : Avancé
- **Cas d'usage** : Architecture production robuste

---

## Points Clés à Retenir

Avant de plonger dans les détails techniques, gardez en tête ces principes :

### 🎯 Principes Fondamentaux

1. **Les sauvegardes sont obligatoires, pas optionnelles**
   - C'est une question de "quand" pas de "si" un incident arrivera

2. **Automatisation = Fiabilité**
   - Les humains oublient, les machines non

3. **Tester, tester, tester**
   - Une sauvegarde non testée ne vaut rien

4. **Diversifier (3-2-1)**
   - Ne jamais avoir tous ses œufs dans le même panier

5. **Documenter**
   - En crise, pas le temps de chercher

6. **Sécuriser**
   - Protéger les sauvegardes comme les données elles-mêmes

### 📊 Métriques à Connaître

- **RTO** : Temps maximal d'indisponibilité acceptable
- **RPO** : Quantité maximale de données perdues acceptable
- **Fréquence** : À quelle cadence sauvegarder
- **Rétention** : Combien de temps garder les sauvegardes
- **Coût** : Budget total (stockage + temps + ressources)

### ✅ Checklist Minimale

Avant de continuer, assurez-vous d'avoir :

- [ ] Compris l'importance des sauvegardes
- [ ] Identifié vos objectifs RTO/RPO
- [ ] Évalué votre budget et ressources
- [ ] Compris les différentes approches (logique vs physique vs WAL)
- [ ] Réfléchi à votre stratégie globale

---

## Prêt à Commencer ?

Maintenant que vous comprenez **pourquoi** et **quoi**, passons au **comment**.

Dans les sections suivantes, nous allons explorer en détail chaque technique de sauvegarde, avec des exemples concrets, des scripts prêts à l'emploi, et des recommandations pratiques.

**Parcours recommandé** :

```
Débutant :
    Commencer par → 16.11.1 (pg_dump)
    Puis → Tests et pratique
    Puis → 16.11.4 (Stratégie 3-2-1)

Intermédiaire :
    Commencer par → 16.11.2 (pg_basebackup)
    Puis → 16.11.3 (PITR)
    Puis → 16.11.4 (Stratégie 3-2-1)

Avancé :
    Lire dans l'ordre complet
    Implémenter architecture complète
    Automatiser et monitorer
```

---

## Message Final

> "Il existe deux types d'administrateurs de bases de données : ceux qui ont perdu des données, et ceux qui vont en perdre."

Ne faites pas partie de ceux qui apprennent à leurs dépens. Investissez du temps maintenant dans une stratégie de sauvegarde solide. Vos données, votre entreprise, et votre tranquillité d'esprit en dépendent.

**La meilleure sauvegarde est celle que vous faites aujourd'hui, pas celle que vous ferez demain.**

---


⏭️ [Sauvegardes logiques (pg_dump, pg_dumpall)](/16-administration-configuration-securite/11.1-sauvegardes-logiques.md)
