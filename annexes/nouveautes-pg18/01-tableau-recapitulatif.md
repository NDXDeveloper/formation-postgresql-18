🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe F : Nouveautés PostgreSQL 18 en un Coup d'Œil

## Tableau Récapitulatif des Features Majeures

PostgreSQL 18, sorti en septembre 2025, apporte des améliorations significatives en termes de performances, sécurité et facilité d'utilisation. Voici un aperçu des fonctionnalités majeures pour vous aider à comprendre ce que cette version apporte.

---

## 🚀 Performances et Optimisations

| Fonctionnalité | Description | Avantages pour les Débutants | Impact |
|----------------|-------------|------------------------------|---------|
| **I/O Asynchrone (AIO)** | Nouveau système de lecture/écriture des données qui ne bloque plus le serveur pendant l'attente des données sur le disque | Vos requêtes sont traitées plus rapidement, surtout sur de gros volumes de données | ⭐⭐⭐⭐⭐ Jusqu'à 3× plus rapide |
| **Skip Scan sur Index Multi-colonnes** | PostgreSQL peut maintenant "sauter" intelligemment dans les index composés de plusieurs colonnes | Moins besoin de créer des index supplémentaires, requêtes plus rapides automatiquement | ⭐⭐⭐⭐ |
| **Optimisation OR → ANY** | Les conditions avec plusieurs "OR" sont automatiquement transformées en une forme plus efficace | Vos requêtes avec plusieurs conditions "OU" s'exécutent plus rapidement sans modification | ⭐⭐⭐ |
| **Auto-élimination des Self-Joins** | Le planificateur détecte et supprime les jointures inutiles d'une table avec elle-même | Code plus propre, performances améliorées sans intervention | ⭐⭐⭐ |
| **Réorganisation DISTINCT** | Optimisation automatique de l'ordre des colonnes dans les clauses DISTINCT | Meilleure performance sur les requêtes de déduplication | ⭐⭐⭐ |
| **Optimisation IN (VALUES)** | Conversion automatique des listes de valeurs en opérateur ANY plus performant | Requêtes avec de longues listes de valeurs beaucoup plus rapides | ⭐⭐⭐ |

---

## 🔐 Sécurité et Authentification

| Fonctionnalité | Description | Avantages pour les Débutants | Impact |
|----------------|-------------|------------------------------|---------|
| **Authentification OAuth 2.0** | Support natif de l'authentification moderne OAuth 2.0 | Intégration facile avec les systèmes d'authentification d'entreprise (Google, Microsoft, etc.) | ⭐⭐⭐⭐ |
| **SCRAM Passthrough** | Propagation sécurisée de l'authentification à travers les connexions distantes (FDW, dblink) | Sécurité renforcée lors de l'accès à des bases distantes | ⭐⭐⭐ |
| **Mode FIPS** | Conformité avec les standards cryptographiques gouvernementaux américains | Important pour les environnements réglementés (banque, santé, administration) | ⭐⭐⭐ |
| **Configuration TLS 1.3** | Paramètre dédié pour configurer les algorithmes de chiffrement TLS 1.3 | Chiffrement moderne et performant des connexions | ⭐⭐⭐ |
| **Data Checksums par Défaut** | Vérification automatique de l'intégrité des données activée à l'installation | Détection précoce des corruptions de données, meilleure fiabilité | ⭐⭐⭐⭐ |

---

## 📊 Manipulation de Données

| Fonctionnalité | Description | Avantages pour les Débutants | Impact |
|----------------|-------------|------------------------------|---------|
| **OLD et NEW dans RETURNING** | Accès aux anciennes ET nouvelles valeurs dans les clauses RETURNING | Facilite le suivi des modifications sans requête supplémentaire | ⭐⭐⭐⭐ |
| **OLD et NEW dans MERGE** | Support complet de OLD/NEW dans les opérations MERGE | Opérations de fusion de données plus puissantes et expressives | ⭐⭐⭐ |
| **Améliorations COPY** | Meilleure gestion du marqueur de fin `\.` dans les fichiers CSV | Import/export de données CSV plus fiable et moins d'erreurs | ⭐⭐⭐ |
| **Contraintes Temporelles** | Nouvelles contraintes pour valider des périodes de temps (dates de début/fin) | Validation automatique des plages de dates, moins d'erreurs logiques | ⭐⭐⭐⭐ |

---

## 🏗️ Modélisation et Structure

| Fonctionnalité | Description | Avantages pour les Débutants | Impact |
|----------------|-------------|------------------------------|---------|
| **Colonnes Générées Virtuelles** | Colonnes calculées qui ne sont pas stockées mais calculées à la volée | Économie d'espace disque tout en ayant des colonnes calculées | ⭐⭐⭐⭐ |
| **UUIDv7** | Nouveau type d'identifiant universel unique avec composante temporelle | Identifiants uniques triables chronologiquement, meilleurs pour les index | ⭐⭐⭐⭐ |

---

## 🔧 Administration et Maintenance

| Fonctionnalité | Description | Avantages pour les Débutants | Impact |
|----------------|-------------|------------------------------|---------|
| **pg_upgrade Amélioré** | Préservation des statistiques lors des montées de version | Upgrade plus rapide et performances immédiates après migration | ⭐⭐⭐⭐⭐ |
| **Option --swap pour pg_upgrade** | Échange rapide des répertoires de données sans copie complète | Migration ultra-rapide (quelques secondes vs heures) | ⭐⭐⭐⭐⭐ |
| **Vérifications Parallèles (--jobs)** | Utilisation de plusieurs CPU pendant l'upgrade pour accélérer les vérifications | Upgrade beaucoup plus rapide sur machines multi-cœurs | ⭐⭐⭐⭐ |
| **Autovacuum Dynamique** | Ajustement automatique du nombre de workers autovacuum selon la charge | Maintenance automatique plus efficace, moins d'intervention manuelle | ⭐⭐⭐⭐ |
| **autovacuum_vacuum_max_threshold** | Nouveau paramètre pour limiter le vacuum sur les très grandes tables | Meilleur contrôle de l'impact du vacuum sur les performances | ⭐⭐⭐ |
| **Statistiques VACUUM/ANALYZE** | Nouvelles colonnes dans pg_stat_all_tables pour suivre la maintenance | Monitoring amélioré, détection facile des tables mal entretenues | ⭐⭐⭐ |
| **Statistiques I/O par Backend** | Suivi détaillé des lectures/écritures disque par processus | Diagnostic précis des problèmes de performance | ⭐⭐⭐⭐ |
| **Statistiques WAL par Backend** | Suivi de la génération de journaux de transactions par processus | Identification des processus générant beaucoup d'écritures | ⭐⭐⭐ |

---

## 📈 Observabilité

| Fonctionnalité | Description | Avantages pour les Débutants | Impact |
|----------------|-------------|------------------------------|---------|
| **EXPLAIN avec Buffers Automatique** | Affichage automatique des informations de cache dans EXPLAIN | Analyse plus facile des performances de requêtes | ⭐⭐⭐⭐ |
| **Configuration I/O Method** | Possibilité de choisir entre I/O synchrone et asynchrone | Optimisation fine selon le matériel et la charge de travail | ⭐⭐⭐ |

---

## 📖 Légende des Niveaux d'Impact

- ⭐⭐⭐⭐⭐ **Critique** : Fonctionnalité majeure qui change significativement l'utilisation ou les performances  
- ⭐⭐⭐⭐ **Important** : Amélioration notable qui mérite attention  
- ⭐⭐⭐ **Utile** : Amélioration appréciable dans certains contextes

---

## 🎯 Les 5 Fonctionnalités à Retenir Absolument

Si vous devez retenir seulement les fonctionnalités les plus impactantes de PostgreSQL 18 :

### 1. **I/O Asynchrone (AIO)** 🚀
**Pourquoi ?** Amélioration de performances jusqu'à 3× sur les opérations disque. C'est la fonctionnalité la plus impactante.

**En pratique :** Vos requêtes complexes et vos imports/exports de données seront significativement plus rapides sans aucune modification de code.

---

### 2. **pg_upgrade Amélioré avec --swap** ⚡
**Pourquoi ?** Migration de version en quelques secondes au lieu de plusieurs heures.

**En pratique :** Passer de PostgreSQL 17 à 18 devient presque instantané, ce qui facilite grandement la mise à jour des serveurs de production.

---

### 3. **Authentification OAuth 2.0** 🔐
**Pourquoi ?** Intégration moderne avec les systèmes d'authentification d'entreprise.

**En pratique :** Vous pouvez utiliser votre compte Google/Microsoft pour vous connecter à PostgreSQL, comme vous le faites pour d'autres applications.

---

### 4. **Colonnes Générées Virtuelles** 💾
**Pourquoi ?** Calculs à la volée sans consommer d'espace disque.

**En pratique :** Vous pouvez avoir une colonne "prix_ttc" calculée automatiquement à partir de "prix_ht" sans stocker deux fois l'information.

---

### 5. **Data Checksums par Défaut** ✅
**Pourquoi ?** Protection automatique contre la corruption de données.

**En pratique :** PostgreSQL détecte automatiquement si vos données ont été corrompues (disque défectueux, bug matériel, etc.) et vous alerte immédiatement.

---

## 💡 Impact sur la Migration

### Compatibilité
✅ **Excellente rétrocompatibilité** : Vos applications fonctionnant sur PostgreSQL 15, 16 ou 17 continueront de fonctionner sans modification.

### Considérations de Migration
- **Data Checksums** : Si vous migrez depuis une version plus ancienne sans checksums, cette fonctionnalité sera activée par défaut sur les nouvelles installations  
- **Dépréciation MD5** : Continuer à migrer vers SCRAM-SHA-256 pour l'authentification  
- **Configuration I/O** : Le mode asynchrone est activé automatiquement quand le système le supporte

### Recommandations d'Adoption
- ✅ **Nouveaux projets** : Utiliser PostgreSQL 18 sans hésitation  
- ✅ **Production existante** : Planifier la migration dans les 6-12 mois pour bénéficier des gains de performance  
- ⚠️ **Tests préalables** : Toujours tester en environnement de développement/staging avant la production

---

## 📚 Pour Aller Plus Loin

Pour approfondir ces fonctionnalités, consultez :
- **Chapitre 3.5** : I/O Asynchrone en détail  
- **Chapitre 16.2** : Authentification OAuth 2.0  
- **Chapitre 11.6** : Colonnes Générées Virtuelles  
- **Chapitre 19.3** : pg_upgrade et stratégies de migration

---

⏭️ [Impact sur migration et compatibilité](/annexes/nouveautes-pg18/02-impact-migration-compatibilite.md)
