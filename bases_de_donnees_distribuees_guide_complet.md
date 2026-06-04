# 📚 Guide Complet — Bases de Données Distribuées
## PostgreSQL · FDW · Citus · Docker · Prometheus · Grafana

> **Objectif :** Tout ce qu'il faut savoir pour maîtriser les bases de données distribuées avec l'écosystème PostgreSQL.  
> **Public :** Étudiants en Licence IA & Big Data / Ingénieurs bases de données  
> **Dernière mise à jour :** Juin 2026

---

## Table des Matières

1. [Fondamentaux des Bases de Données Distribuées](#1-fondamentaux-des-bases-de-données-distribuées)
2. [PostgreSQL — Rappels Essentiels](#2-postgresql--rappels-essentiels)
3. [Réplication PostgreSQL](#3-réplication-postgresql)
4. [Foreign Data Wrapper (FDW)](#4-foreign-data-wrapper-fdw)
5. [Citus — PostgreSQL Distribué](#5-citus--postgresql-distribué)
6. [Docker pour les Bases de Données Distribuées](#6-docker-pour-les-bases-de-données-distribuées)
7. [Prometheus — Collecte de Métriques](#7-prometheus--collecte-de-métriques)
8. [Grafana — Visualisation et Dashboards](#8-grafana--visualisation-et-dashboards)
9. [Architecture Complète : Intégration de la Stack](#9-architecture-complète--intégration-de-la-stack)
10. [Questions d'Examen Types et Réponses](#10-questions-dexamen-types-et-réponses)

---

# 1. Fondamentaux des Bases de Données Distribuées

## 1.1 Définition et Motivations

Une **base de données distribuée** est un ensemble de bases de données logiquement reliées, réparties sur plusieurs nœuds (serveurs) interconnectés par un réseau. Ces nœuds collaborent pour offrir un accès unifié aux données.

**Pourquoi distribuer ?**

| Problème centralisé | Solution distribuée |
|---|---|
| Capacité de stockage limitée | Ajout horizontal de nœuds |
| CPU insuffisant pour les requêtes | Parallélisation sur plusieurs nœuds |
| Point de défaillance unique (SPOF) | Haute disponibilité par réplication |
| Latence pour les utilisateurs éloignés | Données géographiquement proches des utilisateurs |
| Scalabilité difficile | Scale-out (horizontal) vs scale-up (vertical) |

## 1.2 Architectures Distribuées

### Shared-Everything (Partagé-Total)
- Tous les nœuds partagent la mémoire et le stockage
- Exemple : systèmes SMP (Symmetric Multi-Processing)
- Limité par la bande passante du bus

### Shared-Disk (Disque Partagé)
- Chaque nœud a sa propre mémoire, mais partage le stockage
- Exemple : Oracle RAC
- Goulot d'étranglement : le stockage partagé

### Shared-Nothing (Rien Partagé) ← *Modèle de Citus*
- Chaque nœud possède sa propre mémoire ET son propre stockage
- Communication uniquement via réseau
- Scalabilité quasi-linéaire
- Citus utilise cette architecture

## 1.3 Théorème CAP

Le **Théorème CAP** (Brewer, 2000) stipule qu'un système distribué ne peut simultanément garantir que **deux** des trois propriétés suivantes :

```
         Consistency (C)
              /\
             /  \
            /    \
           /      \
          /  CA    \
         /          \
        /____________\
   CP  /              \  AP
      /________________\
Partition Tolerance (P)    Availability (A)
```

| Propriété | Définition |
|---|---|
| **C** — Consistency | Tous les nœuds voient les mêmes données au même moment (cohérence linéarisable) |
| **A** — Availability | Chaque requête reçoit une réponse (pas nécessairement la plus récente) |
| **P** — Partition Tolerance | Le système continue à fonctionner malgré les pannes réseau |

**⚠️ Point crucial :** En pratique, les partitions réseau sont inévitables. Donc tout système distribué réel doit être **P-tolérant**. Le vrai choix est entre **C** et **A** en cas de partition.

| Combinaison | Exemple | Usage typique |
|---|---|---|
| **CP** | PostgreSQL+Citus, MongoDB (mode strict), HBase | Finance, transactions critiques |
| **AP** | CouchDB, Cassandra, DynamoDB | Réseaux sociaux, e-commerce |
| **CA** | SGBD centralisé, PostgreSQL mono-nœud | Applications mono-serveur |

**Différence CAP vs ACID :**
- **ACID Consistency** = l'état de la base ne viole pas les contraintes d'intégrité après une transaction
- **CAP Consistency** = tous les nœuds du cluster voient les mêmes données (cohérence entre répliques)

## 1.4 Propriétés ACID vs BASE

### ACID (Systèmes relationnels traditionnels)
```
A — Atomicity    : la transaction est tout ou rien
C — Consistency  : la base passe d'un état cohérent à un autre
I — Isolation    : les transactions concurrentes s'exécutent comme si elles étaient seules
D — Durability   : les données validées survivent aux pannes
```

### BASE (Systèmes NoSQL/distribués AP)
```
BA — Basically Available   : le système répond toujours
S  — Soft State            : l'état peut changer sans input (due à la cohérence éventuelle)
E  — Eventually Consistent : le système sera cohérent... un jour
```

**Citus et PostgreSQL visent le CP** : ils maintiennent la cohérence ACID même en distribué via le 2PC (Two-Phase Commit).

## 1.5 Techniques de Distribution des Données

### Sharding (Fragmentation Horizontale)
Division des **lignes** d'une table entre plusieurs nœuds. Une table de 100 millions de lignes peut être répartie sur 10 nœuds (10M lignes/nœud).

```
Table Users (100M lignes)
    ↓ Sharding par user_id
Nœud 1 : user_id 0–9 999 999
Nœud 2 : user_id 10M–19 999 999
...
Nœud 10 : user_id 90M–99 999 999
```

### Stratégies de Sharding

| Stratégie | Principe | Avantages | Inconvénients |
|---|---|---|---|
| **Hash Sharding** | `shard = hash(clé) % nb_shards` | Distribution uniforme | Pas de range queries efficaces |
| **Range Sharding** | Plages de valeurs par shard | Bon pour les range queries | Risque de hotspot |
| **List Sharding** | Valeurs spécifiques → shard défini | Intuitif pour données géo/catégories | Moins flexible |
| **Schema Sharding** (Citus 12+) | Un schéma = un shard | Idéal pour multi-tenant/microservices | Moins flexible pour analytics |

### Partitionnement vs Sharding
- **Partitionnement** : division de données au sein d'**un seul** nœud (natif PostgreSQL)
- **Sharding** : distribution de données sur **plusieurs** nœuds

## 1.6 Réplication

La **réplication** consiste à maintenir des copies (répliques) des données sur plusieurs nœuds.

| Type | Description | Cohérence | Latence |
|---|---|---|---|
| **Synchrone** | Le primaire attend la confirmation de toutes les répliques avant de valider | Forte | Haute |
| **Asynchrone** | Le primaire valide sans attendre les répliques | Éventuelle | Basse |
| **Semi-synchrone** | Le primaire attend un sous-ensemble de répliques | Intermédiaire | Intermédiaire |

**Topologies de réplication :**
- **Primaire → Secondaire(s)** : réplication unidirectionnelle (le plus courant)
- **Multi-primaire** : plusieurs nœuds acceptent les écritures (complexe à gérer)
- **Cascade** : Primaire → Secondaire → Secondaire

---

# 2. PostgreSQL — Rappels Essentiels

## 2.1 Architecture Interne PostgreSQL

```
Application / Client
        |
        | (TCP/IP ou socket Unix)
        ↓
  Postmaster (processus superviseur)
        |
        ↓ fork()
  Backend Process (par connexion)
        |
   ┌────┴────────────────────────────────────────┐
   │                Shared Memory                │
   │  ┌──────────────┐  ┌──────────────────────┐ │
   │  │ Shared Buffers│  │  WAL Buffers          │ │
   │  └──────────────┘  └──────────────────────┘ │
   └────────────────────────────────────────────┘
        |
   ┌────┴────────────────────────────────────────┐
   │              Stockage Disque                │
   │  ┌────────┐  ┌────────┐  ┌────────────────┐ │
   │  │Data Dir│  │WAL/pg_wal│  │pg_stat_statements│ │
   │  └────────┘  └────────┘  └────────────────┘ │
   └────────────────────────────────────────────┘
```

## 2.2 Fichiers de Configuration Clés

### `postgresql.conf` — Configuration principale

```ini
# Connexions
listen_addresses = '*'          # Écouter sur toutes les interfaces
max_connections = 100           # Connexions simultanées max

# Mémoire
shared_buffers = 256MB          # Cache partagé (25% de la RAM recommandé)
work_mem = 4MB                  # Mémoire par opération de tri/hash
maintenance_work_mem = 64MB     # Mémoire pour VACUUM, CREATE INDEX

# WAL (Write-Ahead Logging)
wal_level = replica             # minimal | replica | logical
max_wal_senders = 5             # Connexions de réplication max
wal_keep_size = 128MB           # WAL conservé pour les répliques

# Performance
checkpoint_completion_target = 0.9
effective_cache_size = 1GB      # Estimation du cache OS

# Monitoring
shared_preload_libraries = 'pg_stat_statements'
log_min_duration_statement = 1000   # Logger requêtes > 1 sec
```

### `pg_hba.conf` — Authentification des hôtes

```
# TYPE   DATABASE   USER    ADDRESS        METHOD
local    all        all                    peer
host     all        all     127.0.0.1/32   scram-sha-256
host     all        all     0.0.0.0/0      scram-sha-256
host     replication replicator 10.0.0.0/24 scram-sha-256
```

## 2.3 Commandes Essentielles

```sql
-- Informations système
SELECT version();
SHOW max_connections;
SHOW shared_buffers;

-- Statistiques des bases
SELECT datname, numbackends, xact_commit, xact_rollback
FROM pg_stat_database;

-- Statistiques des tables
SELECT schemaname, tablename, n_live_tup, n_dead_tup, last_vacuum
FROM pg_stat_user_tables;

-- Requêtes actives
SELECT pid, state, query, query_start
FROM pg_stat_activity
WHERE state != 'idle';

-- Verrous
SELECT l.pid, l.mode, r.relname
FROM pg_locks l
JOIN pg_class r ON l.relation = r.oid
WHERE l.granted = false;

-- Extensions installées
SELECT * FROM pg_extension;

-- Taille des bases
SELECT pg_database.datname, pg_size_pretty(pg_database_size(pg_database.datname))
FROM pg_database ORDER BY pg_database_size(pg_database.datname) DESC;
```

## 2.4 WAL — Write-Ahead Log

Le **WAL** est le journal de transactions de PostgreSQL. Avant d'écrire sur le disque principal, PostgreSQL écrit dans le WAL.

```
Transaction → WAL Buffer → WAL Files (pg_wal/) → Data Files
                   ↑
        (checkpoint force sync)
```

**Rôles du WAL :**
- Durabilité (Durability dans ACID)
- Récupération après crash
- Base de la réplication (streaming & logical)

**Paramètres WAL importants :**

| Paramètre | Valeur | Rôle |
|---|---|---|
| `wal_level` | `minimal` / `replica` / `logical` | Niveau de détail du WAL |
| `max_wal_size` | ex: `1GB` | Taille max avant checkpoint forcé |
| `checkpoint_timeout` | ex: `5min` | Intervalle entre checkpoints |
| `archive_mode` | `on` / `off` | Activer l'archivage WAL |
| `archive_command` | shell command | Commande d'archivage |

---

# 3. Réplication PostgreSQL

## 3.1 Réplication en Streaming (Streaming Replication)

La réplication en streaming envoie les enregistrements WAL en **temps réel** du primaire vers le(s) standby.

```
Primaire                    Standby
┌─────────────┐             ┌─────────────┐
│  Écritures  │             │  Lectures   │
│  WAL gen.   │──WAL stream─▶│  WAL apply  │
│             │             │  Hot Standby│
└─────────────┘             └─────────────┘
```

### Configuration sur le Primaire

**`postgresql.conf`** :
```ini
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
wal_keep_size = 128MB
hot_standby = on
listen_addresses = '*'
```

**`pg_hba.conf`** :
```
host replication replicateur 10.0.0.0/24 scram-sha-256
```

**Création de l'utilisateur de réplication** :
```sql
CREATE USER replicateur REPLICATION LOGIN ENCRYPTED PASSWORD 'motdepasse_fort';
```

### Configuration sur le Standby

```bash
# Copier les données du primaire
pg_basebackup -h <ip_primaire> -U replicateur -D /var/lib/postgresql/data \
  -P -Xs -R

# Le flag -R crée automatiquement le fichier standby.signal
# et postgresql.auto.conf avec primary_conninfo
```

**`postgresql.conf`** sur le standby :
```ini
hot_standby = on   # Permet les lectures sur le standby
```

**`postgresql.auto.conf`** (généré par pg_basebackup -R) :
```ini
primary_conninfo = 'host=<ip_primaire> port=5432 user=replicateur password=motdepasse_fort application_name=standby1'
```

### Vérification de la réplication

```sql
-- Sur le PRIMAIRE
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag, sync_state
FROM pg_stat_replication;

-- Sur le STANDBY
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
SELECT pg_is_in_recovery();  -- Retourne TRUE si c'est un standby
```

## 3.2 Réplication Synchrone vs Asynchrone

```ini
# Sur le primaire : réplication SYNCHRONE
synchronous_standby_names = 'standby1'    -- Un standby synchrone
synchronous_standby_names = 'ANY 1 (standby1, standby2)'  -- Au moins 1 sur 2
synchronous_standby_names = 'FIRST 2 (standby1, standby2, standby3)'  -- 2 premiers
```

| Mode | Comportement | COMMIT retourne quand... |
|---|---|---|
| Asynchrone | Défaut, risque de perte de données | WAL écrit localement |
| Synchrone (`remote_write`) | Standby a écrit en mémoire | WAL écrit en mémoire du standby |
| Synchrone (`on`) | Standby a écrit sur disque | WAL écrit sur disque du standby |
| Synchrone (`remote_apply`) | Standby a appliqué | Transaction rejouée sur standby |

## 3.3 Réplication Logique

La **réplication logique** réplique les **modifications de données** (INSERT, UPDATE, DELETE) à un niveau logique (par opposition au niveau physique/octet du streaming).

**Avantages :**
- Réplication sélective (table par table)
- Réplication entre versions PostgreSQL différentes
- Réplication vers des bases non-PostgreSQL

**Concepts :**
- **Publication** : définit ce qui est repliqué (sur le primaire)
- **Abonnement** : définit où et comment recevoir (sur le subscriber)

### Mise en Place

**Sur le Publisher (primaire)** :
```sql
-- Prérequis : wal_level = logical dans postgresql.conf
ALTER SYSTEM SET wal_level = 'logical';
SELECT pg_reload_conf();

-- Créer une publication
CREATE PUBLICATION ma_publication FOR TABLE commandes, clients;

-- Ou pour toutes les tables
CREATE PUBLICATION tout FOR ALL TABLES;
```

**Sur le Subscriber** :
```sql
-- Créer les tables identiques d'abord
CREATE TABLE commandes (...);
CREATE TABLE clients (...);

-- S'abonner
CREATE SUBSCRIPTION mon_abonnement
CONNECTION 'host=primaire port=5432 dbname=madb user=rep password=xxx'
PUBLICATION ma_publication;

-- Vérifier
SELECT * FROM pg_stat_subscription;
```

## 3.4 Slots de Réplication

Un **slot de réplication** garantit que le WAL n'est pas supprimé avant que le standby l'ait consommé.

```sql
-- Créer un slot
SELECT pg_create_physical_replication_slot('mon_slot');
SELECT pg_create_logical_replication_slot('mon_slot_logique', 'pgoutput');

-- Voir les slots
SELECT slot_name, slot_type, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;

-- Supprimer un slot (ATTENTION : peut libérer WAL accumulé)
SELECT pg_drop_replication_slot('mon_slot');
```

**⚠️ Attention :** Un slot inactif peut causer l'accumulation de WAL et saturer le disque !

---

# 4. Foreign Data Wrapper (FDW)

## 4.1 Définition et Architecture

Un **Foreign Data Wrapper (FDW)** est une implémentation PostgreSQL du standard **SQL/MED** (Management of External Data). Il permet d'accéder à des données externes comme si elles étaient des tables locales.

```
PostgreSQL Local                    Source Externe
┌─────────────────────────┐         ┌──────────────────┐
│  SELECT * FROM ft_users │         │  PostgreSQL distant│
│          ↓              │         │  MySQL           │
│   Foreign Table (proxy) │──FDW──▶ │  CSV/fichier     │
│          ↓              │         │  Oracle          │
│   FDW Handler           │         │  Redis, MongoDB  │
└─────────────────────────┘         └──────────────────┘
```

**Composants FDW :**
1. **Extension FDW** : le driver (ex: `postgres_fdw`)
2. **Foreign Server** : la connexion au serveur distant
3. **User Mapping** : correspondance d'authentification
4. **Foreign Table** : la table virtuelle locale

## 4.2 FDWs Disponibles

| FDW | Source | Notes |
|---|---|---|
| `postgres_fdw` | PostgreSQL distant | Inclus dans PostgreSQL, le plus utilisé |
| `file_fdw` | Fichiers CSV/texte | Inclus dans PostgreSQL |
| `mysql_fdw` | MySQL/MariaDB | Extension tierce |
| `oracle_fdw` | Oracle DB | Extension tierce |
| `tds_fdw` | SQL Server / Sybase | Extension tierce |
| `mongo_fdw` | MongoDB | Extension tierce |
| `redis_fdw` | Redis | Extension tierce |
| `odbc_fdw` | N'importe quelle source ODBC | Extension tierce |

## 4.3 postgres_fdw — Guide Complet

### Étape 1 : Installation de l'extension

```sql
-- Sur la base locale (coordinator)
CREATE EXTENSION postgres_fdw;

-- Vérifier
SELECT * FROM pg_extension WHERE extname = 'postgres_fdw';
```

### Étape 2 : Créer le Foreign Server

```sql
CREATE SERVER serveur_distant
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (
    host '192.168.1.100',   -- IP ou hostname du serveur distant
    port '5432',             -- Port PostgreSQL
    dbname 'base_distante'   -- Nom de la base distante
  );

-- Vérifier
SELECT srvname, srvowner, srvfdw, srvoptions FROM pg_foreign_server;
```

### Étape 3 : Créer le User Mapping

```sql
-- Mapper l'utilisateur local vers l'utilisateur distant
CREATE USER MAPPING FOR utilisateur_local
  SERVER serveur_distant
  OPTIONS (
    user 'utilisateur_distant',
    password 'mot_de_passe'
  );

-- Pour le superuser
CREATE USER MAPPING FOR postgres
  SERVER serveur_distant
  OPTIONS (user 'postgres', password 'secret');

-- Vérifier
SELECT * FROM pg_user_mappings;
```

### Étape 4 : Créer les Foreign Tables

**Méthode 1 — Création manuelle :**
```sql
CREATE FOREIGN TABLE ft_commandes (
  id          INTEGER NOT NULL,
  client_id   INTEGER,
  montant     DECIMAL(10,2),
  date_cmd    TIMESTAMP,
  statut      VARCHAR(50)
)
SERVER serveur_distant
OPTIONS (
  schema_name 'public',
  table_name  'commandes'
);
```

**Méthode 2 — IMPORT FOREIGN SCHEMA (plus pratique) :**
```sql
-- Importer toutes les tables d'un schéma distant
IMPORT FOREIGN SCHEMA public
  FROM SERVER serveur_distant
  INTO schema_local;

-- Importer des tables spécifiques seulement
IMPORT FOREIGN SCHEMA public
  LIMIT TO (commandes, clients, produits)
  FROM SERVER serveur_distant
  INTO schema_local;

-- Exclure certaines tables
IMPORT FOREIGN SCHEMA public
  EXCEPT (logs, temp_data)
  FROM SERVER serveur_distant
  INTO schema_local;
```

### Étape 5 : Utilisation

```sql
-- SELECT : récupère les données du serveur distant
SELECT * FROM ft_commandes WHERE statut = 'PENDING';

-- JOIN entre table locale et table distante
SELECT c.nom, c.email, cmd.montant
FROM clients c                      -- table locale
JOIN ft_commandes cmd               -- table distante
  ON c.id = cmd.client_id
WHERE cmd.date_cmd > NOW() - INTERVAL '30 days';

-- INSERT : insère sur le serveur distant
INSERT INTO ft_commandes (client_id, montant, statut)
VALUES (42, 150.00, 'NEW');

-- UPDATE
UPDATE ft_commandes SET statut = 'SHIPPED' WHERE id = 1001;

-- DELETE
DELETE FROM ft_commandes WHERE statut = 'CANCELLED' AND date_cmd < '2024-01-01';
```

**⚠️ Note :** INSERT/UPDATE/DELETE nécessitent que le user mapping ait les droits appropriés sur la table distante.

### Options Importantes de postgres_fdw

```sql
-- Options du Foreign Server
CREATE SERVER s FOREIGN DATA WRAPPER postgres_fdw OPTIONS (
  host '...',
  port '5432',
  dbname '...',
  connect_timeout '10',       -- Timeout de connexion (sec)
  keepalives '1',             -- Activer TCP keepalives
  keepalives_idle '60'        -- Intervalle keepalive
);

-- Options de la Foreign Table (surcharge les options serveur)
CREATE FOREIGN TABLE ft (...)
SERVER s OPTIONS (
  schema_name 'public',
  table_name 'ma_table',
  updatable 'true',           -- Autoriser INSERT/UPDATE/DELETE
  fetch_size '1000',          -- Nombre de lignes par fetch
  batch_size '100'            -- Taille du batch pour INSERT
);
```

### Analyse des Performances FDW

```sql
-- EXPLAIN pour voir si les filtres sont poussés (pushed down)
EXPLAIN (VERBOSE, ANALYZE) 
SELECT * FROM ft_commandes WHERE client_id = 42;

-- Chercher "Remote SQL:" dans le plan
-- Remote SQL: SELECT id, client_id, montant FROM public.commandes WHERE client_id = 42
-- ↑ Le filtre WHERE est exécuté sur le serveur distant = BIEN
```

**Pushdown FDW** : PostgreSQL tente de pousser les conditions WHERE, les tris ORDER BY, et les agrégations vers le serveur distant pour minimiser le transfert de données.

## 4.4 file_fdw — Lire des Fichiers CSV

```sql
CREATE EXTENSION file_fdw;

CREATE SERVER fichiers_server FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE ft_csv_ventes (
  date_vente  DATE,
  produit     TEXT,
  quantite    INTEGER,
  prix        NUMERIC
)
SERVER fichiers_server
OPTIONS (
  filename '/tmp/ventes.csv',
  format 'csv',
  header 'true',
  delimiter ',',
  null 'NULL'
);

SELECT * FROM ft_csv_ventes WHERE quantite > 100;
```

## 4.5 FDW vs dblink

| Critère | `postgres_fdw` | `dblink` |
|---|---|---|
| Standard SQL | SQL/MED ✅ | Propriétaire |
| Syntaxe | Tables normales | Fonctions spéciales |
| Pushdown | Oui (filtres, JOIN, agrégats) | Non |
| Performance | Meilleure | Plus lente |
| INSERT/UPDATE/DELETE | Oui | Possible mais complexe |
| Transactions | Intégrées | Manuelles |
| Usage recommandé | ✅ Privilégier | Cas legacy uniquement |

---

# 5. Citus — PostgreSQL Distribué

## 5.1 Présentation de Citus

**Citus** est une **extension PostgreSQL open-source** (acquise par Microsoft) qui transforme PostgreSQL en base de données distribuée. C'est une extension, pas un fork : vous utilisez toujours PostgreSQL, avec des capacités de distribution ajoutées.

```
Application
    |
    | (connexion PostgreSQL standard)
    ↓
┌──────────────────────────────┐
│     Coordinateur (Coordinator)│
│  - Reçoit toutes les requêtes │
│  - Planifie et distribue      │
│  - Agrège les résultats       │
└──────────────────────────────┘
         |          |
    ┌────┘          └────┐
    ↓                    ↓
┌─────────┐          ┌─────────┐
│ Worker 1 │          │ Worker 2 │
│ Shards   │          │ Shards   │
│ A, C, E  │          │ B, D, F  │
└─────────┘          └─────────┘
```

**Citus est 100% compatible PostgreSQL :** les requêtes SQL standard, les extensions, et les outils PostgreSQL fonctionnent normalement.

## 5.2 Concepts Fondamentaux

### Types de Tables dans Citus

#### 1. Tables Distribuées (Distributed Tables)

```sql
-- Créer une table distribuée par hash sur customer_id
SELECT create_distributed_table('commandes', 'customer_id');

-- La table est divisée en shards répartis sur les workers
-- Chaque shard = une table normale PostgreSQL sur un worker
-- Ex: commandes_102001, commandes_102002, ..., commandes_102032
```

#### 2. Tables de Référence (Reference Tables)

```sql
-- Répliquée intégralement sur TOUS les workers
-- Idéale pour les petites tables de lookup
SELECT create_reference_table('pays');
SELECT create_reference_table('categories');
SELECT create_reference_table('devises');

-- Avantage : JOINs sans surcharge réseau
-- Utilise 2PC (two-phase commit) pour la cohérence
```

#### 3. Tables Locales

Tables ordinaires PostgreSQL stockées uniquement sur le coordinateur. Peuvent être promues en tables gérées via :
```sql
SELECT citus_add_local_table_to_metadata('ma_table');
```

### Shards

Un **shard** est un fragment d'une table distribuée, stocké comme une table ordinaire sur un worker.

```
Table distribuée : orders (10M lignes, distribution: customer_id)
    ↓ hash(customer_id) % 32
Shard 0 : orders_102000 → Worker 1 (~312K lignes)
Shard 1 : orders_102001 → Worker 2 (~312K lignes)
...
Shard 31: orders_102031 → Worker 1 (~312K lignes)
```

```sql
-- Voir les shards
SELECT * FROM citus_shards;

-- Voir le placement des shards
SELECT s.shardid, s.logicalrelid, sp.nodename, sp.nodeport
FROM pg_dist_shard s
JOIN pg_dist_shard_placement sp ON s.shardid = sp.shardid
ORDER BY s.shardid;
```

### Colocation (Co-location)

Des tables avec **la même clé de distribution** et **créées ensemble** ont leurs shards correspondants sur les mêmes workers. Cela permet des JOINs et des transactions locales sans surcharge réseau.

```sql
-- customers et orders colocalisées sur customer_id
SELECT create_distributed_table('customers', 'customer_id');
SELECT create_distributed_table('orders', 'customer_id', colocate_with => 'customers');

-- Ce JOIN s'exécute localement sur chaque worker (pas de réseau entre workers)
SELECT c.name, SUM(o.amount)
FROM customers c JOIN orders o USING (customer_id)
GROUP BY c.customer_id;
```

## 5.3 Installation et Configuration

### Via Docker (Recommandé pour le développement)

```bash
# Lancer un cluster Citus simple (1 coordinateur + 2 workers)
docker run -d --name citus -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  citusdata/citus

# Se connecter
docker exec -it citus psql -U postgres
```

### Via Docker Compose (Cluster complet)

```yaml
# docker-compose.yml
version: '3.8'

services:
  coordinator:
    image: citusdata/citus:12.1
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_USER: postgres
      POSTGRES_DB: citus_demo
    volumes:
      - coordinator_data:/var/lib/postgresql/data
    networks:
      - citus_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  worker1:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_USER: postgres
      POSTGRES_DB: citus_demo
    volumes:
      - worker1_data:/var/lib/postgresql/data
    networks:
      - citus_net
    depends_on:
      coordinator:
        condition: service_healthy

  worker2:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_USER: postgres
      POSTGRES_DB: citus_demo
    volumes:
      - worker2_data:/var/lib/postgresql/data
    networks:
      - citus_net
    depends_on:
      coordinator:
        condition: service_healthy

volumes:
  coordinator_data:
  worker1_data:
  worker2_data:

networks:
  citus_net:
    driver: bridge
```

### Enregistrer les Workers

```sql
-- Se connecter au coordinateur
-- Puis enregistrer les workers
SELECT citus_add_node('worker1', 5432);
SELECT citus_add_node('worker2', 5432);

-- Vérifier le cluster
SELECT * FROM citus_get_active_worker_nodes();

-- Ou
SELECT nodeid, nodename, nodeport, isactive FROM pg_dist_node;
```

## 5.4 Gestion des Tables Distribuées

### Créer et Distribuer

```sql
-- 1. Activer l'extension
CREATE EXTENSION citus;

-- 2. Créer les tables normalement
CREATE TABLE clients (
  client_id  BIGSERIAL PRIMARY KEY,
  nom        TEXT NOT NULL,
  email      TEXT UNIQUE,
  pays       CHAR(2),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE commandes (
  commande_id BIGSERIAL,
  client_id   BIGINT NOT NULL REFERENCES clients(client_id),
  montant     DECIMAL(10,2),
  statut      TEXT DEFAULT 'PENDING',
  created_at  TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (commande_id, client_id)  -- client_id dans la PK pour distribution
);

-- 3. Distribuer
SELECT create_distributed_table('clients', 'client_id');
SELECT create_distributed_table('commandes', 'client_id', colocate_with => 'clients');

-- 4. Tables de référence
CREATE TABLE statuts (code TEXT PRIMARY KEY, libelle TEXT);
SELECT create_reference_table('statuts');
```

### Requêtes sur Citus

```sql
-- Requête routée vers UN seul worker (filter sur la clé de distribution)
SELECT * FROM commandes WHERE client_id = 42;
-- Le coordinateur route directement vers le worker contenant client_id=42

-- Requête parallélisée sur TOUS les workers (pas de filtre sur clé de distribution)
SELECT pays, COUNT(*), SUM(montant)
FROM clients c JOIN commandes cmd USING (client_id)
GROUP BY pays;
-- Le coordinateur collecte et agrège les résultats de tous les workers
```

### Rééquilibrage des Shards

```sql
-- Après ajout d'un nouveau worker, rééquilibrer
SELECT rebalance_table_shards('commandes');

-- Voir la progression
SELECT * FROM get_rebalance_progress();

-- Déplacer un shard spécifique
SELECT citus_move_shard_placement(102001, 'worker1', 5432, 'worker3', 5432);
```

## 5.5 Schéma-Based Sharding (Citus 12+)

Une nouveauté majeure : sharding basé sur les **schémas**, idéal pour le multi-tenant.

```sql
-- Activer le sharding par schéma
SET citus.enable_schema_based_sharding = ON;

-- Créer un schéma = automatiquement distribué
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_globex;

-- Les tables dans ces schémas sont automatiquement distribuées
SET search_path = tenant_acme;
CREATE TABLE orders (id SERIAL PRIMARY KEY, product TEXT, amount DECIMAL);
-- Cette table est automatiquement un shard distribué !

-- Chaque tenant a son propre schéma, isolé des autres
```

## 5.6 Transactions Distribuées et 2PC

Citus utilise le **Two-Phase Commit (2PC)** pour les transactions multi-workers.

```
Phase 1 — PREPARE :
  Coordinateur → Worker1 : "PREPARE TRANSACTION tx1"
  Coordinateur → Worker2 : "PREPARE TRANSACTION tx1"
  Workers répondent : "PRÊT" ou "ERREUR"

Phase 2 — COMMIT ou ROLLBACK :
  Si tous préparés → COMMIT sur tous les workers
  Si au moins un a échoué → ROLLBACK sur tous les workers
```

```sql
-- Transaction distribuée (transparente pour l'utilisateur)
BEGIN;
  UPDATE commandes SET statut = 'SHIPPED' WHERE client_id = 42;
  UPDATE stock SET quantite = quantite - 1 WHERE produit_id = 100;
COMMIT;
-- Citus gère le 2PC en interne

-- Voir les transactions distribuées en cours
SELECT * FROM citus_dist_stat_activity;
```

## 5.7 Cas d'Usage de Citus

| Cas d'Usage | Pattern | Configuration |
|---|---|---|
| **Multi-tenant SaaS** | Sharding par tenant_id ou par schéma | Row-based ou schema-based sharding |
| **IoT / Time Series** | Sharding par device_id + partitionnement par temps | Distributed + time partitioning |
| **Analytics / OLAP** | Tables de faits distribuées | Hash sharding sur dimension principale |
| **Microservices** | Schema-based sharding | Un schéma par service |
| **Géospatial** | Couplage avec PostGIS | Sharding par région/grid |

## 5.8 Monitoring Citus

```sql
-- Vue d'ensemble du cluster
SELECT * FROM citus_get_active_worker_nodes();

-- Répartition des shards par worker
SELECT nodename, nodeport, COUNT(*) as nb_shards
FROM pg_dist_shard_placement
GROUP BY nodename, nodeport;

-- Taille des shards
SELECT shardid, shard_size
FROM citus_shards
ORDER BY shard_size DESC;

-- Requêtes distribuées en cours
SELECT * FROM citus_dist_stat_activity WHERE state = 'active';

-- Statistiques Citus
SELECT run_command_on_workers('SELECT count(*) FROM pg_stat_user_tables');
```

---

# 6. Docker pour les Bases de Données Distribuées

## 6.1 Concepts Docker Essentiels

### Images, Conteneurs, Volumes, Réseaux

```
Image Docker      = Template immuable (ex: postgres:16, citusdata/citus:12.1)
Conteneur         = Instance en cours d'exécution d'une image
Volume            = Stockage persistant (survit à la suppression du conteneur)
Réseau Docker     = Communication entre conteneurs
```

### Commandes Docker Fondamentales

```bash
# Images
docker pull postgres:16                 # Télécharger une image
docker images                           # Lister les images locales
docker rmi postgres:16                  # Supprimer une image

# Conteneurs
docker run -d \                         # Lancer en arrière-plan
  --name mon_postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mabase \
  -p 5432:5432 \                        # host_port:container_port
  -v pgdata:/var/lib/postgresql/data \  # Monter un volume
  postgres:16

docker ps                               # Conteneurs actifs
docker ps -a                            # Tous les conteneurs
docker stop mon_postgres                # Arrêter
docker start mon_postgres               # Démarrer
docker rm mon_postgres                  # Supprimer le conteneur
docker logs mon_postgres                # Voir les logs
docker logs -f mon_postgres             # Suivre les logs en temps réel

# Exécuter une commande dans un conteneur
docker exec -it mon_postgres bash
docker exec -it mon_postgres psql -U postgres

# Volumes
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata

# Réseaux
docker network create mon_reseau
docker network ls
docker network inspect mon_reseau
```

## 6.2 Docker Compose

Docker Compose orchestre plusieurs conteneurs liés.

### Syntaxe Docker Compose

```yaml
version: '3.8'

services:               # Définition des services (conteneurs)
  mon_service:
    image: postgres:16  # Image à utiliser
    build: ./dossier    # Ou : construire depuis un Dockerfile
    container_name: nom_fixe
    restart: unless-stopped  # Politique de redémarrage
    
    environment:        # Variables d'environnement
      POSTGRES_PASSWORD: secret
    env_file:           # Ou depuis un fichier .env
      - .env
    
    ports:              # Mapping des ports (host:container)
      - "5432:5432"
    
    volumes:            # Montages de volumes
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    
    networks:           # Réseaux auxquels appartient le service
      - mon_reseau
    
    depends_on:         # Dépendances de démarrage
      autre_service:
        condition: service_healthy
    
    healthcheck:        # Vérification de santé
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    
    deploy:             # Ressources (mode swarm ou informationnel)
      resources:
        limits:
          memory: 512M

volumes:                # Définition des volumes nommés
  pgdata:

networks:               # Définition des réseaux
  mon_reseau:
    driver: bridge
```

### Commandes Docker Compose

```bash
docker compose up -d              # Démarrer en arrière-plan
docker compose up -d --scale worker=3  # Scaler les workers
docker compose down               # Arrêter et supprimer les conteneurs
docker compose down -v            # + supprimer les volumes
docker compose ps                 # État des services
docker compose logs -f            # Voir tous les logs
docker compose logs -f postgres   # Logs d'un service spécifique
docker compose exec postgres bash # Accéder à un service
docker compose restart postgres   # Redémarrer un service
docker compose pull               # Mettre à jour les images
```

## 6.3 Cluster Citus Complet avec Docker Compose

```yaml
# docker-compose-citus-monitoring.yml
version: '3.8'

services:

  # === CITUS CLUSTER ===
  coordinator:
    image: citusdata/citus:12.1
    container_name: citus_coordinator
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-citus}
      POSTGRES_USER: postgres
      POSTGRES_DB: demo
    volumes:
      - coordinator_data:/var/lib/postgresql/data
      - ./init-coordinator.sql:/docker-entrypoint-initdb.d/01-init.sql
    networks:
      - citus_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  worker1:
    image: citusdata/citus:12.1
    container_name: citus_worker1
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-citus}
      POSTGRES_USER: postgres
      POSTGRES_DB: demo
    volumes:
      - worker1_data:/var/lib/postgresql/data
    networks:
      - citus_net
    depends_on:
      coordinator:
        condition: service_healthy

  worker2:
    image: citusdata/citus:12.1
    container_name: citus_worker2
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-citus}
      POSTGRES_USER: postgres
      POSTGRES_DB: demo
    volumes:
      - worker2_data:/var/lib/postgresql/data
    networks:
      - citus_net
    depends_on:
      coordinator:
        condition: service_healthy

  # === MONITORING ===
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: postgres_exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres:${POSTGRES_PASSWORD:-citus}@coordinator:5432/demo?sslmode=disable"
    ports:
      - "9187:9187"
    networks:
      - citus_net
    depends_on:
      coordinator:
        condition: service_healthy

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - citus_net
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - citus_net
    depends_on:
      - prometheus

volumes:
  coordinator_data:
  worker1_data:
  worker2_data:
  prometheus_data:
  grafana_data:

networks:
  citus_net:
    driver: bridge
```

### Script d'initialisation du coordinateur

```sql
-- init-coordinator.sql
-- Activé automatiquement par docker-entrypoint-initdb.d/

-- Activer Citus
CREATE EXTENSION IF NOT EXISTS citus;

-- Activer pg_stat_statements pour le monitoring
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Créer l'utilisateur de monitoring
CREATE USER monitoring WITH PASSWORD 'monitoring_password';
GRANT pg_monitor TO monitoring;
```

### Enregistrer les workers (après démarrage)

```bash
# Script bash pour enregistrer les workers
#!/bin/bash
sleep 10  # Attendre que les workers soient prêts

docker exec citus_coordinator psql -U postgres -d demo -c "
  SELECT citus_add_node('worker1', 5432);
  SELECT citus_add_node('worker2', 5432);
  SELECT * FROM citus_get_active_worker_nodes();
"
```

## 6.4 Dockerfile Personnalisé PostgreSQL

```dockerfile
FROM postgres:16

# Installer des extensions
RUN apt-get update && apt-get install -y \
    postgresql-16-citus-12.1 \
    postgresql-contrib \
    && rm -rf /var/lib/apt/lists/*

# Copier des scripts d'initialisation
COPY init.sql /docker-entrypoint-initdb.d/

# Configuration personnalisée
COPY postgresql.conf /etc/postgresql/postgresql.conf

# Variables d'environnement par défaut
ENV POSTGRES_PASSWORD=secret
ENV POSTGRES_DB=mabase

# Modifier la commande de démarrage pour utiliser notre config
CMD ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
```

---

# 7. Prometheus — Collecte de Métriques

## 7.1 Architecture de Prometheus

Prometheus est un système de monitoring **open-source** qui collecte des métriques via un modèle **pull** (il scrape les endpoints, contrairement aux systèmes push).

```
┌─────────────────────────────────────────────────────────┐
│                    PROMETHEUS SERVER                     │
│  ┌─────────────┐    ┌──────────────┐   ┌─────────────┐ │
│  │  Retrieval   │    │  TSDB        │   │  HTTP API   │ │
│  │  (Scraper)   │───▶│  (Time Series│──▶│  (Query)    │ │
│  └─────────────┘    │   Database)  │   └─────────────┘ │
│         ↑           └──────────────┘          │         │
│         │                                     │         │
│  ┌──────┴───────┐                    ┌────────▼───────┐ │
│  │ Service      │                    │  Alert Manager │ │
│  │ Discovery    │                    │  (notifications)│ │
│  └──────────────┘                    └────────────────┘ │
└─────────────────────────────────────────────────────────┘
         ↑                                    ↑
         │ scrape                             │ query
    ┌────┴─────────────────┐         ┌────────┴──────┐
    │ postgres_exporter    │         │ Grafana       │
    │ node_exporter        │         │ (Dashboards)  │
    │ :9187  :9100         │         └───────────────┘
    └──────────────────────┘
```

## 7.2 Concepts Prometheus

### Types de Métriques

| Type | Description | Exemple PostgreSQL |
|---|---|---|
| **Counter** | Valeur qui ne fait qu'augmenter | Nombre total de transactions |
| **Gauge** | Valeur qui peut monter/descendre | Connexions actives |
| **Histogram** | Distribution de valeurs (buckets) | Durée des requêtes |
| **Summary** | Résumé statistique (quantiles) | Latence des requêtes (p50, p95, p99) |

### Labels

Les labels permettent de filtrer et regrouper les métriques :

```
pg_stat_database_numbackends{datname="mydb", instance="localhost:9187"}
^-- nom métrique ^-- labels                                              ^-- valeur
```

### PromQL — Langage de Requête

```promql
# Sélection simple
pg_stat_database_numbackends

# Filtrer par label
pg_stat_database_numbackends{datname="mabase"}

# Filtres regex
pg_stat_database_numbackends{datname=~"prod.*"}
pg_stat_database_numbackends{datname!~"test.*"}

# Opérations mathématiques
100 * pg_stat_database_xact_rollback / (pg_stat_database_xact_commit + pg_stat_database_xact_rollback)

# Fonctions sur le temps (rate = taux par seconde)
rate(pg_stat_database_xact_commit[5m])     # Transactions/sec (moyenne 5 min)
irate(pg_stat_database_xact_commit[5m])    # Taux instantané

# Aggrégations
sum(pg_stat_database_numbackends)          # Total connexions toutes bases
sum by (datname) (pg_stat_database_numbackends)  # Par base
avg(pg_stat_database_numbackends)
max(pg_stat_database_numbackends)

# Opérations sur les Counters
increase(pg_stat_database_xact_commit[1h]) # Augmentation sur 1 heure

# Comparaisons et alertes
pg_stat_database_numbackends > 80          # Connexions > 80
(pg_stat_database_numbackends / pg_settings_max_connections) * 100 > 80
```

## 7.3 Configuration de Prometheus

### `prometheus.yml`

```yaml
# prometheus.yml
global:
  scrape_interval: 15s          # Fréquence de collecte (défaut)
  evaluation_interval: 15s      # Fréquence d'évaluation des alertes
  scrape_timeout: 10s           # Timeout par scrape

# Alerting
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Fichiers de règles d'alertes
rule_files:
  - "alerts/postgresql_alerts.yml"
  - "alerts/node_alerts.yml"

# Jobs de scraping
scrape_configs:

  # Prometheus lui-même
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # PostgreSQL via postgres_exporter
  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres_exporter:9187']
    metrics_path: /metrics
    scrape_interval: 30s        # Surcharge la valeur globale
    params:
      collect[]:
        - 'pg_stat_statements'
        - 'pg_stat_bgwriter'

  # Métriques système (Node Exporter)
  - job_name: 'node'
    static_configs:
      - targets:
          - 'node_exporter:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+).*'
        replacement: '$1'

  # Citus workers (si nécessaire)
  - job_name: 'citus_workers'
    static_configs:
      - targets:
          - 'worker1_exporter:9187'
          - 'worker2_exporter:9187'
```

## 7.4 postgres_exporter

**postgres_exporter** est l'agent qui expose les métriques PostgreSQL au format Prometheus.

### Installation

```bash
# Via Docker (recommandé)
docker run -d \
  --name postgres_exporter \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://monitoring:password@postgres:5432/mabase?sslmode=disable" \
  prometheuscommunity/postgres-exporter

# Via binaire
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xvf postgres_exporter-*.tar.gz
mv postgres_exporter /usr/local/bin/
```

### Configuration de l'utilisateur PostgreSQL pour le monitoring

```sql
-- Créer l'utilisateur de monitoring (principe du moindre privilège)
CREATE USER monitoring WITH PASSWORD 'monitoring_password';

-- Accorder les droits nécessaires
GRANT pg_monitor TO monitoring;          -- PostgreSQL 10+
-- Équivalent à : pg_read_all_settings + pg_read_all_stats + pg_stat_scan_tables

-- Pour pg_stat_statements (requêtes lentes)
GRANT SELECT ON pg_stat_statements TO monitoring;
```

### Requêtes personnalisées (`queries.yaml`)

```yaml
# queries.yaml — Métriques personnalisées
pg_stat_statements_top:
  query: |
    SELECT
      queryid::text as queryid,
      left(query, 100) as query_preview,
      calls,
      total_exec_time / 1000.0 as total_exec_time_seconds,
      mean_exec_time / 1000.0 as mean_exec_time_seconds,
      rows
    FROM pg_stat_statements
    ORDER BY total_exec_time DESC
    LIMIT 10;
  metrics:
    - queryid:
        usage: "LABEL"
        description: "ID de la requête"
    - query_preview:
        usage: "LABEL"
        description: "Aperçu de la requête"
    - calls:
        usage: "COUNTER"
        description: "Nombre d'appels"
    - total_exec_time_seconds:
        usage: "COUNTER"
        description: "Temps total d'exécution (secondes)"
    - mean_exec_time_seconds:
        usage: "GAUGE"
        description: "Temps moyen d'exécution (secondes)"
    - rows:
        usage: "COUNTER"
        description: "Lignes retournées"

pg_locks_count:
  query: |
    SELECT mode, count(*) as count
    FROM pg_locks
    GROUP BY mode;
  metrics:
    - mode:
        usage: "LABEL"
    - count:
        usage: "GAUGE"
        description: "Nombre de verrous par mode"

pg_table_bloat:
  query: |
    SELECT
      schemaname,
      tablename,
      pg_size_pretty(pg_total_relation_size(quote_ident(schemaname) || '.' || quote_ident(tablename))) as total_size,
      n_dead_tup,
      n_live_tup,
      CASE WHEN n_live_tup > 0
           THEN round(100.0 * n_dead_tup / n_live_tup, 2)
           ELSE 0 END as dead_ratio
    FROM pg_stat_user_tables
    WHERE n_live_tup > 0
    ORDER BY n_dead_tup DESC
    LIMIT 20;
  metrics:
    - schemaname:
        usage: "LABEL"
    - tablename:
        usage: "LABEL"
    - n_dead_tup:
        usage: "GAUGE"
    - dead_ratio:
        usage: "GAUGE"
        description: "Ratio de tuples morts (%)"
```

```bash
# Lancer postgres_exporter avec requêtes personnalisées
docker run -d \
  -e DATA_SOURCE_NAME="postgresql://monitoring:password@postgres:5432/mabase?sslmode=disable" \
  -v $(pwd)/queries.yaml:/queries.yaml \
  prometheuscommunity/postgres-exporter \
  --extend.query-path=/queries.yaml
```

## 7.5 Métriques PostgreSQL Clés

### Connexions

```promql
# Connexions actives
pg_stat_database_numbackends{datname="mabase"}

# Taux d'utilisation des connexions (%)
(sum(pg_stat_database_numbackends) / pg_settings_max_connections) * 100

# Connexions par état
pg_stat_activity_count{state="active"}
pg_stat_activity_count{state="idle"}
pg_stat_activity_count{state="idle in transaction"}
```

### Transactions

```promql
# Transactions commitées par seconde
rate(pg_stat_database_xact_commit{datname="mabase"}[5m])

# Transactions rollbackées par seconde
rate(pg_stat_database_xact_rollback{datname="mabase"}[5m])

# Taux d'erreur de transactions (%)
rate(pg_stat_database_xact_rollback[5m]) /
(rate(pg_stat_database_xact_commit[5m]) + rate(pg_stat_database_xact_rollback[5m])) * 100
```

### Cache et I/O

```promql
# Taux de cache hit (doit être > 90%)
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)

# Blocs lus du disque par seconde
rate(pg_stat_database_blks_read[5m])
```

### Réplication

```promql
# Lag de réplication (secondes)
pg_replication_lag

# Répliques connectées
count(pg_stat_replication)
```

## 7.6 Règles d'Alerte

```yaml
# alerts/postgresql_alerts.yml
groups:
  - name: postgresql_alerts
    rules:

      # Trop de connexions
      - alert: PostgreSQLHighConnections
        expr: >
          (sum(pg_stat_database_numbackends) by (instance) /
           pg_settings_max_connections) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL : trop de connexions sur {{ $labels.instance }}"
          description: "{{ $value | printf \"%.1f\" }}% des connexions sont utilisées"

      # Connexions proches de la limite
      - alert: PostgreSQLConnectionsCritical
        expr: >
          (sum(pg_stat_database_numbackends) by (instance) /
           pg_settings_max_connections) * 100 > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "CRITIQUE : connexions PostgreSQL quasi saturées sur {{ $labels.instance }}"

      # Lag de réplication élevé
      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Lag de réplication PostgreSQL élevé : {{ $value }}s"

      # Deadlocks
      - alert: PostgreSQLDeadlocks
        expr: rate(pg_stat_database_deadlocks[5m]) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Deadlocks détectés sur {{ $labels.instance }}"

      # Cache hit ratio bas
      - alert: PostgreSQLLowCacheHitRatio
        expr: >
          (pg_stat_database_blks_hit{datname!="postgres"} /
           (pg_stat_database_blks_hit{datname!="postgres"} +
            pg_stat_database_blks_read{datname!="postgres"})) < 0.90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio bas ({{ $value | printf \"%.1f%%\" }}) sur {{ $labels.datname }}"

      # PostgreSQL down
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL est DOWN sur {{ $labels.instance }}"
```

---

# 8. Grafana — Visualisation et Dashboards

## 8.1 Architecture Grafana

```
Data Sources                 Grafana                    Utilisateurs
┌─────────────┐         ┌────────────────┐         ┌──────────────┐
│ Prometheus  │──Query──▶│  Dashboard     │──HTTP──▶│  Navigateur  │
│ PostgreSQL  │         │  Panels        │         └──────────────┘
│ MySQL       │         │  Alerts        │
│ InfluxDB    │         │  Users/Teams   │
│ Elasticsearch│         └────────────────┘
└─────────────┘
```

## 8.2 Concepts Grafana

| Concept | Description |
|---|---|
| **Data Source** | Connexion à une source de données (Prometheus, PostgreSQL...) |
| **Dashboard** | Ensemble de panels organisés sur une page |
| **Panel** | Unité de visualisation (graphe, jauge, tableau, etc.) |
| **Row** | Regroupement de panels |
| **Variable** | Paramètre dynamique (ex: choisir la base de données) |
| **Alert** | Règle de notification basée sur une requête |
| **Organization** | Espace de travail isolé |
| **Folder** | Regroupement de dashboards |

## 8.3 Installation et Démarrage

```bash
# Via Docker
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  -v grafana_data:/var/lib/grafana \
  grafana/grafana:latest

# Accéder : http://localhost:3000
# Login par défaut : admin / admin
```

## 8.4 Configurer Prometheus comme Data Source

```
Grafana → Configuration → Data Sources → Add data source
→ Sélectionner "Prometheus"
→ URL: http://prometheus:9090
→ Save & Test
```

**Via provisioning (pour Docker)** :

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "15s"

  - name: PostgreSQL Direct
    type: postgres
    url: coordinator:5432
    database: demo
    user: monitoring
    secureJsonData:
      password: monitoring_password
    jsonData:
      sslmode: disable
      postgresVersion: 1600
      timescaledb: false
```

## 8.5 Dashboard PostgreSQL — Métriques Clés

### Panel : Connexions Actives

```
Type: Time series
Query (PromQL):
  pg_stat_database_numbackends{datname="$database", instance="$instance"}

Axes:
  - Y: Nombre de connexions
  - X: Temps

Seuils:
  - 0-80: Vert
  - 80-95: Orange
  - 95+: Rouge
```

### Panel : Transactions par Seconde

```
Type: Time series
Query:
  rate(pg_stat_database_xact_commit{datname="$database"}[5m])
  +
  rate(pg_stat_database_xact_rollback{datname="$database"}[5m])

Légende: {{datname}} - TPS
```

### Panel : Cache Hit Ratio

```
Type: Gauge (jauge)
Query:
  (
    sum(pg_stat_database_blks_hit{datname="$database"})
    /
    sum(pg_stat_database_blks_hit{datname="$database"} + pg_stat_database_blks_read{datname="$database"})
  ) * 100

Min: 0, Max: 100
Unité: Percent
Seuils:
  - < 90%: Rouge
  - 90-95%: Orange
  - > 95%: Vert
```

### Panel : Top Requêtes Lentes

```
Type: Table
Query (Prometheus avec pg_stat_statements):
  topk(10, pg_stat_statements_mean_exec_time_seconds)

Colonnes: query_preview, calls, mean_exec_time_seconds
Tri: mean_exec_time_seconds DESC
```

## 8.6 Importer un Dashboard Prêt à l'Emploi

```
Grafana → Dashboards → Import
→ ID: 9628 (PostgreSQL Database - dashboard communautaire populaire)
→ Sélectionner la source Prometheus
→ Import
```

**Dashboards utiles :**
| ID | Nom | Description |
|---|---|---|
| `9628` | PostgreSQL Database | Vue générale complète |
| `455` | PostgreSQL Statistics | Statistiques avancées |
| `12485` | PostgreSQL Prometheus | Métriques Prometheus |
| `1860` | Node Exporter Full | Métriques système |

## 8.7 Variables de Dashboard

Les variables permettent de créer des dashboards interactifs.

```
Dashboard Settings → Variables → Add variable

Type: Query
Name: database
Label: Base de données
Data source: Prometheus
Query: label_values(pg_stat_database_numbackends, datname)
Refresh: On dashboard load
```

Utilisation dans les panels :
```promql
pg_stat_database_numbackends{datname="$database"}
```

## 8.8 Alertes Grafana

```
Panel → Alert → Create alert rule

Condition:
  WHEN avg() OF query(A, 5m, now) IS ABOVE 90

Notifications:
  - Email: admin@domaine.com
  - Slack: #alerts-db
  - PagerDuty

Message:
  "PostgreSQL {{$labels.instance}}: connexions à {{$value}}% de la limite"
```

## 8.9 Provisioning de Dashboards

```yaml
# grafana/provisioning/dashboards/postgresql.yml
apiVersion: 1

providers:
  - name: 'postgresql-dashboards'
    orgId: 1
    folder: 'PostgreSQL'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

---

# 9. Architecture Complète : Intégration de la Stack

## 9.1 Vue d'Ensemble de l'Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     APPLICATIONS                                   │
│           App 1                App 2              App 3            │
└───────────────┬────────────────────┬─────────────────┬────────────┘
                │ SQL                │                 │
                ▼                   ▼                 ▼
┌───────────────────────────────────────────────────────────────────┐
│              CITUS COORDINATOR (port 5432)                        │
│         Planificateur de requêtes distribuées                     │
│         Métadonnées (pg_dist_shard, pg_dist_node...)             │
└───────────────────────────────────────────────────────────────────┘
              │                     │
    ┌─────────┘                     └─────────┐
    ▼                                         ▼
┌───────────────┐                     ┌───────────────┐
│   WORKER 1    │                     │   WORKER 2    │
│ Shards A,C,E  │                     │ Shards B,D,F  │
│ + Ref Tables  │                     │ + Ref Tables  │
└───────────────┘                     └───────────────┘
        │                                     │
        │ (accès via FDW depuis autre source) │
        ▼                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                              │
│                                                                  │
│  postgres_exporter ──scrape──▶ Prometheus ──query──▶ Grafana    │
│  :9187                          :9090                 :3000     │
│                                    │                            │
│                            AlertManager                         │
│                            Email/Slack                          │
└─────────────────────────────────────────────────────────────────┘
```

## 9.2 Scénario Complet : Mise en Place Pas à Pas

### Étape 1 : Démarrer l'infrastructure

```bash
# Cloner la configuration
mkdir mon-cluster && cd mon-cluster

# Créer les fichiers de configuration (prometheus.yml, etc.)
# Lancer toute la stack
docker compose up -d

# Vérifier que tout est opérationnel
docker compose ps
```

### Étape 2 : Initialiser Citus

```bash
# Se connecter au coordinateur
docker exec -it citus_coordinator psql -U postgres -d demo

# Enregistrer les workers
SELECT citus_add_node('worker1', 5432);
SELECT citus_add_node('worker2', 5432);

# Vérifier
SELECT * FROM citus_get_active_worker_nodes();
```

### Étape 3 : Créer le schéma distribué

```sql
-- Activer les extensions
CREATE EXTENSION citus;
CREATE EXTENSION pg_stat_statements;

-- Créer les tables
CREATE TABLE tenants (
  tenant_id  BIGSERIAL PRIMARY KEY,
  nom        TEXT NOT NULL,
  plan       TEXT DEFAULT 'free'
);

CREATE TABLE events (
  event_id   BIGSERIAL,
  tenant_id  BIGINT NOT NULL,
  type       TEXT,
  payload    JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (event_id, tenant_id)
);

-- Distribuer
SELECT create_reference_table('tenants');
SELECT create_distributed_table('events', 'tenant_id');

-- Insérer des données de test
INSERT INTO tenants (nom, plan) VALUES ('Acme Corp', 'pro'), ('Globex', 'enterprise');
INSERT INTO events (tenant_id, type, payload)
  SELECT t.tenant_id, 'PAGE_VIEW', '{"url": "/dashboard"}'
  FROM tenants t, generate_series(1, 1000);
```

### Étape 4 : Configurer le monitoring

```sql
-- Activer pg_stat_statements (nécessite shared_preload_libraries)
-- Dans postgresql.conf : shared_preload_libraries = 'pg_stat_statements'
-- Puis redémarrer PostgreSQL, puis :
CREATE EXTENSION pg_stat_statements;

-- Vérifier les requêtes collectées
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

### Étape 5 : Configurer Grafana

```
1. Aller sur http://localhost:3000 (admin/admin)
2. Ajouter Data Source → Prometheus → http://prometheus:9090
3. Importer Dashboard ID 9628
4. Créer des alertes sur les métriques critiques
```

### Étape 6 : Ajouter un FDW vers une source externe

```sql
-- Accéder à une base externe depuis le coordinateur Citus
CREATE EXTENSION postgres_fdw;

CREATE SERVER base_legacy
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host '192.168.1.200', port '5432', dbname 'legacy_db');

CREATE USER MAPPING FOR postgres
  SERVER base_legacy
  OPTIONS (user 'readonly_user', password 'password');

IMPORT FOREIGN SCHEMA public
  LIMIT TO (old_customers)
  FROM SERVER base_legacy
  INTO legacy_schema;

-- Requête cross-cluster !
SELECT t.nom, COUNT(e.event_id)
FROM tenants t
JOIN events e USING (tenant_id)
WHERE t.tenant_id IN (
  SELECT id FROM legacy_schema.old_customers WHERE migrated = false
)
GROUP BY t.nom;
```

---

# 10. Questions d'Examen Types et Réponses

## 10.1 Questions Théoriques

### Q1 : Qu'est-ce que le théorème CAP et comment s'applique-t-il à Citus ?

**Réponse :** Le théorème CAP stipule qu'un système distribué ne peut garantir simultanément que deux des trois propriétés : Consistency (cohérence), Availability (disponibilité), et Partition Tolerance (tolérance aux partitions). En pratique, les partitions réseau étant inévitables, tout système distribué doit être tolérant aux partitions (P), et le vrai choix est entre C et A.

**Citus se positionne comme CP :** il privilégie la cohérence sur la disponibilité absolue. Il utilise le protocole Two-Phase Commit (2PC) pour garantir que les transactions distribuées sont atomiques. En cas de panne d'un worker, le coordinateur peut retourner une erreur plutôt que des données incohérentes.

---

### Q2 : Quelle est la différence entre la réplication physique et la réplication logique dans PostgreSQL ?

**Réponse :**

| Aspect | Physique (Streaming) | Logique |
|---|---|---|
| Niveau | Octets/blocs WAL | Opérations SQL (INSERT/UPDATE/DELETE) |
| Granularité | Cluster entier | Table par table |
| Compatibilité versions | Même version majeure | Versions différentes possible |
| Filtrage | Non | Oui (par table) |
| Standby | Read-only (Hot Standby) | Read-write possible |
| Usage | HA, DR | Migration, intégration sélective |
| Délai de setup | Plus simple | Plus flexible mais plus complexe |

---

### Q3 : Expliquez le concept de colocation dans Citus et pourquoi il est important.

**Réponse :** La colocation (co-location) dans Citus signifie que les shards de plusieurs tables distribuées sur la même clé de distribution sont placés sur les mêmes workers. Par exemple, si `customers` et `orders` sont toutes deux distribuées sur `customer_id` et colocalisées, toutes les commandes d'un client X et les informations du client X se trouvent sur le même worker.

**Pourquoi c'est important :**
1. **Performance des JOINs** : un JOIN entre deux tables colocalisées s'exécute localement sur chaque worker, sans transfert de données entre workers.
2. **Transactions locales** : les modifications de tables colocalisées sur la même clé peuvent être des transactions locales, plus rapides que le 2PC distribué.
3. **Scalabilité** : sans colocation, un JOIN distribué nécessite un "shuffle" de données entre workers, créant un goulot d'étranglement réseau.

```sql
-- Exemple de colocation
SELECT create_distributed_table('customers', 'customer_id');
SELECT create_distributed_table('orders', 'customer_id', colocate_with => 'customers');

-- Ce JOIN est LOCAL sur chaque worker (pas de réseau inter-workers)
SELECT c.name, SUM(o.amount)
FROM customers c JOIN orders o USING (customer_id)
GROUP BY c.customer_id;
```

---

### Q4 : Qu'est-ce qu'un Foreign Data Wrapper et quels sont ses composants ?

**Réponse :** Un FDW est une implémentation du standard SQL/MED (Management of External Data) qui permet à PostgreSQL d'accéder à des données externes comme si elles étaient des tables locales.

**Composants :**
1. **Extension FDW** (`CREATE EXTENSION postgres_fdw`) : le driver qui sait communiquer avec la source
2. **Foreign Server** (`CREATE SERVER`) : configuration de la connexion au serveur distant
3. **User Mapping** (`CREATE USER MAPPING`) : authentification (utilisateur local → utilisateur distant)
4. **Foreign Table** (`CREATE FOREIGN TABLE` ou `IMPORT FOREIGN SCHEMA`) : la table virtuelle locale

**Avantage clé du pushdown** : PostgreSQL tente de pousser les filtres WHERE et les agrégations vers le serveur distant, minimisant le volume de données transférées.

---

### Q5 : Expliquez le rôle du WAL dans PostgreSQL et son lien avec la réplication.

**Réponse :** Le **WAL (Write-Ahead Log)** est le journal de transactions de PostgreSQL. Son principe : avant d'écrire sur les pages de données, PostgreSQL écrit d'abord la modification dans le WAL. Cela garantit la **durabilité** (le D d'ACID) : en cas de crash, PostgreSQL peut rejouer le WAL pour se remettre dans un état cohérent.

**Lien avec la réplication :**
- **Réplication physique (streaming)** : le WAL est envoyé en temps réel aux serveurs standby, qui le rejouent pour maintenir une copie identique du primaire.
- **Réplication logique** : le WAL est décodé en opérations logiques (INSERT/UPDATE/DELETE) pour une réplication sélective.

**Paramètre clé** : `wal_level`
- `minimal` : juste pour le crash recovery
- `replica` : + informations pour la réplication physique
- `logical` : + informations pour la réplication logique et les FDWs

---

## 10.2 Questions Pratiques

### Q6 : Écrivez le SQL complet pour mettre en place un accès FDW vers une base PostgreSQL distante.

```sql
-- Étape 1 : Installer l'extension
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- Étape 2 : Créer le serveur distant
CREATE SERVER serveur_ventes
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (
    host '192.168.1.100',
    port '5432',
    dbname 'db_ventes'
  );

-- Étape 3 : Créer le mapping utilisateur
CREATE USER MAPPING FOR CURRENT_USER
  SERVER serveur_ventes
  OPTIONS (
    user 'lecteur_ventes',
    password 'motdepasse123'
  );

-- Étape 4 : Importer le schéma
IMPORT FOREIGN SCHEMA public
  LIMIT TO (ventes, produits)
  FROM SERVER serveur_ventes
  INTO public;

-- Étape 5 : Tester
SELECT COUNT(*) FROM ventes WHERE date_vente >= NOW() - INTERVAL '7 days';

-- Étape 6 : Analyser les performances
EXPLAIN (VERBOSE, ANALYZE)
SELECT produit_id, SUM(quantite)
FROM ventes
WHERE date_vente >= '2024-01-01'
GROUP BY produit_id;
```

---

### Q7 : Décrivez un docker-compose.yml minimal pour un cluster Citus avec monitoring.

*(Voir la section 6.3 pour le fichier complet.)*

**Structure minimale :**
```yaml
services:
  coordinator:     # PostgreSQL + Citus
  worker1:         # PostgreSQL + Citus
  worker2:         # PostgreSQL + Citus
  postgres_exporter: # Exposition des métriques PG → Prometheus
  prometheus:      # Collecte et stockage des métriques
  grafana:         # Visualisation
```

**Points clés pour l'examen :**
- Tous les services doivent être sur le même réseau Docker
- Utiliser `depends_on` + `healthcheck` pour l'ordre de démarrage
- Les workers se connectent au coordinateur via le nom du service (ex: `coordinator`)
- Les volumes assurent la persistance des données

---

### Q8 : Comment écrire une requête PromQL pour surveiller le taux de cache hit de PostgreSQL ?

```promql
# Taux de cache hit en pourcentage (valeur entre 0 et 100)
(
  sum(pg_stat_database_blks_hit{datname="$database"}) 
  /
  (
    sum(pg_stat_database_blks_hit{datname="$database"}) 
    + 
    sum(pg_stat_database_blks_read{datname="$database"})
  )
) * 100

# Alerte si < 90%
(pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)) * 100 < 90
```

**Interprétation :**
- `blks_hit` : blocs servis depuis le cache (shared_buffers)
- `blks_read` : blocs lus depuis le disque
- Un ratio < 90% indique un cache insuffisant (`shared_buffers` trop petit, ou dataset trop grand)

---

### Q9 : Quels sont les étapes pour configurer la réplication en streaming PostgreSQL ?

**Sur le PRIMAIRE :**

```bash
# 1. Modifier postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 128MB
listen_addresses = '*'

# 2. Modifier pg_hba.conf
# host replication replicateur <ip_standby>/32 scram-sha-256
```

```sql
-- 3. Créer l'utilisateur de réplication
CREATE USER replicateur REPLICATION LOGIN PASSWORD 'secret';
```

```bash
# 4. Recharger la configuration
SELECT pg_reload_conf();
```

**Sur le STANDBY :**

```bash
# 5. Copier les données (pg_basebackup)
pg_basebackup -h <ip_primaire> -U replicateur \
  -D /var/lib/postgresql/data \
  -P -Xs -R

# Le flag -R crée automatiquement :
# - standby.signal (fichier vide qui indique que c'est un standby)
# - postgresql.auto.conf avec primary_conninfo

# 6. Démarrer le standby
pg_ctl start -D /var/lib/postgresql/data
```

**Vérification :**

```sql
-- Sur le primaire
SELECT client_addr, state, replay_lag FROM pg_stat_replication;

-- Sur le standby  
SELECT pg_is_in_recovery();  -- Doit retourner TRUE
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

---

### Q10 : Comment Citus distribue-t-il les requêtes ? Expliquez le processus de traitement d'une requête SELECT.

**Réponse étape par étape :**

```sql
SELECT pays, COUNT(*), SUM(montant)
FROM commandes
WHERE date_cmd > '2024-01-01'
GROUP BY pays;
```

1. **Réception** : L'application envoie la requête au coordinateur (port 5432)
2. **Parsing** : Le coordinateur parse la requête SQL normalement
3. **Planification** : Le planificateur Citus consulte les métadonnées (`pg_dist_shard`, `pg_dist_node`) pour déterminer où sont les données
4. **Distribution** :
   - Pas de filtre sur la clé de distribution (`customer_id`) → requête doit aller sur TOUS les workers
   - Le coordinateur génère des requêtes fragmentées pour chaque worker :
   ```sql
   -- Envoyé à Worker 1 (shards 102001, 102003...)
   SELECT pays, COUNT(*), SUM(montant) FROM commandes_102001 WHERE date_cmd > '2024-01-01' GROUP BY pays
   UNION ALL
   SELECT ... FROM commandes_102003 ...
   ```
5. **Exécution parallèle** : Les workers exécutent leurs requêtes localement en parallèle
6. **Agrégation** : Le coordinateur collecte les résultats partiels et effectue l'agrégation finale
7. **Retour** : Le résultat final est retourné à l'application

**Optimisation (cas d'un filtre sur la clé de distribution) :**
```sql
SELECT * FROM commandes WHERE customer_id = 42;
-- Le coordinateur calcule : hash(42) % 32 = shard 7 → Worker 2
-- La requête est routée DIRECTEMENT vers Worker 2
-- Aucune communication avec Worker 1 nécessaire !
```

---

## 10.3 Mémo Rapide — Commandes à Connaître

### PostgreSQL

```sql
-- Extensions
CREATE EXTENSION nom_extension;
SELECT * FROM pg_extension;

-- Réplication
SELECT * FROM pg_stat_replication;
SELECT pg_is_in_recovery();
SELECT now() - pg_last_xact_replay_timestamp() AS lag;

-- Monitoring
SELECT * FROM pg_stat_activity WHERE state = 'active';
SELECT * FROM pg_stat_database;
SELECT * FROM pg_stat_user_tables;
SELECT * FROM pg_locks WHERE NOT granted;
```

### FDW

```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER ... FOREIGN DATA WRAPPER postgres_fdw OPTIONS (...);
CREATE USER MAPPING FOR ... SERVER ... OPTIONS (...);
IMPORT FOREIGN SCHEMA public FROM SERVER ... INTO ...;
SELECT * FROM pg_foreign_server;
SELECT * FROM pg_foreign_table;
SELECT * FROM pg_user_mappings;
```

### Citus

```sql
CREATE EXTENSION citus;
SELECT citus_add_node('host', port);
SELECT create_distributed_table('table', 'colonne_distribution');
SELECT create_reference_table('table');
SELECT * FROM citus_get_active_worker_nodes();
SELECT * FROM citus_shards;
SELECT * FROM pg_dist_shard_placement;
SELECT rebalance_table_shards('table');
```

### Docker

```bash
docker compose up -d
docker compose down [-v]
docker compose ps
docker compose logs -f [service]
docker exec -it <container> <command>
docker compose exec <service> psql -U postgres
```

### Prometheus / Grafana

```promql
# Connexions
pg_stat_database_numbackends{datname="$db"}

# Transactions/sec
rate(pg_stat_database_xact_commit[5m])

# Cache hit ratio
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)

# Lag réplication
pg_replication_lag
```

---

## Annexe : Schéma Récapitulatif Global

```
THÉORIE                          OUTILS
────────────────────────────────────────────────────────────────────
Bases Distribuées                 PostgreSQL (moteur de base)
  ├── CAP Theorem (CP/AP/CA)         ├── WAL (Write-Ahead Log)
  ├── ACID vs BASE                   ├── Streaming Replication
  ├── Sharding (Hash/Range/List)     ├── Logical Replication
  ├── Réplication (sync/async)       └── pg_stat_statements
  └── Architectures
       ├── Shared-Nothing ←── Citus      FDW (SQL/MED)
       └── Shared-Disk                   ├── postgres_fdw
                                         ├── file_fdw
DISTRIBUTION (Citus)                     └── mysql_fdw, oracle_fdw
  ├── Coordinator Node
  ├── Worker Nodes                   Docker
  ├── Distributed Tables             ├── Images (postgres, citus, grafana...)
  ├── Reference Tables               ├── Conteneurs
  ├── Shards                         ├── Volumes (persistance)
  ├── Co-location                    ├── Réseaux (communication)
  ├── 2PC (transactions dist.)       └── Docker Compose
  └── Row-based vs Schema-based
                                     MONITORING
                                     ├── Prometheus (collecte TSDB)
                                     │    ├── postgres_exporter :9187
                                     │    ├── node_exporter :9100
                                     │    ├── prometheus.yml (scrape configs)
                                     │    ├── PromQL (langage de requête)
                                     │    └── AlertManager (notifications)
                                     └── Grafana (visualisation)
                                          ├── Data Sources
                                          ├── Dashboards & Panels
                                          ├── Variables
                                          └── Alertes
```

---

*Bonne chance pour l'examen ! 🎓*

> **Conseil final :** Pour réussir, savoir expliquer **pourquoi** on utilise chaque outil est aussi important que de connaître les commandes. Citus pour le scale horizontal, FDW pour l'intégration de données, Docker pour l'infrastructure reproductible, Prometheus+Grafana pour l'observabilité — chaque brique a son rôle précis dans l'architecture.
