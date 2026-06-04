# 📚 Guide Complet — Bases de Données NoSQL
### MongoDB · Cassandra · Elasticsearch · Neo4j · CouchDB · DynamoDB

> **À l'usage de :** Préparation d'examen — Licence Intelligence Artificielle & Big Data  
> **Niveau :** Théorie + Pratique + Comparaisons + Exemples de code  
> **Dernière mise à jour :** Juin 2026

---

## Table des Matières

1. [Introduction au NoSQL](#1-introduction-au-nosql)
2. [Théorèmes et Concepts Fondamentaux](#2-théorèmes-et-concepts-fondamentaux)
3. [MongoDB](#3-mongodb)
4. [Apache Cassandra](#4-apache-cassandra)
5. [Elasticsearch](#5-elasticsearch)
6. [Neo4j](#6-neo4j)
7. [CouchDB](#7-couchdb)
8. [Amazon DynamoDB](#8-amazon-dynamodb)
9. [Comparatif Global](#9-comparatif-global)
10. [Questions d'Examen Types](#10-questions-dexamen-types)

---

## 1. Introduction au NoSQL

### 1.1 Définition

**NoSQL** (Not Only SQL) désigne les systèmes de gestion de bases de données qui ne reposent pas exclusivement sur le modèle relationnel. Ils sont apparus dans les années 2000 pour répondre aux limites des SGBDR face à l'explosion du volume de données (Big Data).

Les acteurs historiques à l'origine du mouvement NoSQL :
- **Google** → BigTable (2004)
- **Amazon** → Dynamo (2007)
- **Facebook** → Cassandra (2008), puis HBase
- **LinkedIn** → Voldemort

### 1.2 Pourquoi le NoSQL ?

Les bases de données relationnelles (SQL) atteignent leurs limites dans les contextes suivants :
- **Volume** : Données à l'échelle des pétaoctets
- **Vélocité** : Millions de requêtes/seconde (réseaux sociaux, IoT)
- **Variété** : Données non structurées, semi-structurées (JSON, XML, graphes, images)
- **Distribution** : Déploiements sur des dizaines ou centaines de serveurs

### 1.3 Caractéristiques générales du NoSQL

- **Schéma flexible** (schema-less ou schema-optional)
- **Scalabilité horizontale** (scale-out) : on ajoute des nœuds plutôt que d'upgrader le serveur
- **Haute disponibilité** : réplication native, tolérance aux pannes
- **Modèles de données variés** : documents, colonnes, graphes, clé-valeur
- **Cohérence éventuelle** (eventual consistency) dans la plupart des cas

### 1.4 Les 4 Grandes Familles NoSQL

| Famille | Modèle | Exemples |
|---------|--------|---------|
| **Document** | JSON/BSON imbriqués | MongoDB, CouchDB |
| **Colonne large** (Wide-column) | Lignes + familles de colonnes | Cassandra, HBase |
| **Graphe** | Nœuds + Relations + Propriétés | Neo4j, ArangoDB |
| **Clé-Valeur** | Paires clé → valeur | Redis, DynamoDB, Riak |
| **Moteur de recherche** | Index inversé, analyse full-text | Elasticsearch, Solr |

> **Note :** DynamoDB est hybride (clé-valeur + document). Elasticsearch est souvent classé à part (moteur de recherche/analytique).

### 1.5 CRUD et opérations de base

Le modèle **CRUD** (Create, Read, Update, Delete) s'applique à tous les NoSQL, avec des APIs différentes selon la base.

---

## 2. Théorèmes et Concepts Fondamentaux

### 2.1 Le Théorème CAP (Brewer, 2000)

Le **théorème CAP** stipule qu'un système de données distribué ne peut garantir simultanément que **deux** des trois propriétés suivantes :

```
        Cohérence (C)
           /\
          /  \
         /    \
        /  ??? \
       /________\
Disponibilité (A)  Tolérance aux partitions (P)
```

- **C — Consistency (Cohérence)** : Tous les nœuds voient les mêmes données au même moment. Chaque lecture reçoit la donnée la plus récente ou une erreur.
- **A — Availability (Disponibilité)** : Chaque requête reçoit une réponse (pas forcément la plus récente), même en cas de panne d'un nœud.
- **P — Partition Tolerance (Tolérance aux partitions)** : Le système continue de fonctionner même si des messages entre nœuds sont perdus ou retardés.

**Dans un réseau distribué, P est non-négociable** (les partitions réseau arrivent toujours). En pratique, le choix se fait entre :

| Choix | Signification | Exemples |
|-------|--------------|---------|
| **CP** | Cohérence + Tolérance → sacrifie la disponibilité | MongoDB (par défaut), HBase |
| **AP** | Disponibilité + Tolérance → cohérence éventuelle | Cassandra, CouchDB, DynamoDB |
| **CA** | Cohérence + Disponibilité → pas distribué | SGBDR classique (PostgreSQL, MySQL) |

> **À retenir :** Cassandra est AP. MongoDB est CP (mais peut être configuré AP). CouchDB est AP.

### 2.2 Le Théorème BASE vs ACID

#### ACID (modèle SQL)
- **A**tomicity : La transaction réussit complètement ou échoue complètement
- **C**onsistency : La base reste dans un état cohérent avant et après
- **I**solation : Les transactions concurrentes ne s'interfèrent pas
- **D**urability : Les données validées sont persistantes même après une panne

#### BASE (modèle NoSQL dominant)
- **B**asically **A**vailable : La disponibilité est garantie, même avec des données potentiellement obsolètes
- **S**oft state : L'état du système peut changer même sans entrée (propagation de la cohérence)
- **E**ventually consistent : La cohérence sera atteinte... éventuellement

### 2.3 Cohérence Éventuelle (Eventual Consistency)

Dans un système distribué AP (ex. Cassandra), si on écrit une valeur sur un nœud, les autres nœuds la recevront avec un léger délai. Pendant ce délai, des lectures sur d'autres nœuds peuvent retourner l'ancienne valeur. C'est la **cohérence éventuelle**.

**Compromis Cohérence / Disponibilité / Latence :**
- Plus on demande de cohérence → plus la latence augmente (on attend que plusieurs nœuds confirment)
- Plus on veut de rapidité → moins on est sûr d'avoir les données les plus fraîches

### 2.4 Scalabilité : Horizontale vs Verticale

- **Scale-up (vertical)** : Ajouter du CPU/RAM/disque au serveur existant → limité et coûteux
- **Scale-out (horizontal)** : Ajouter des nœuds au cluster → préféré par le NoSQL

### 2.5 Sharding (Partitionnement)

Le **sharding** consiste à diviser les données en plusieurs partitions (shards) distribuées sur plusieurs nœuds. Chaque nœud ne contient qu'une partie des données.

**Stratégies de sharding :**
- **Hash sharding** : Un algorithme de hachage détermine sur quel nœud va la donnée → distribution uniforme
- **Range sharding** : Partition par plage de valeurs (ex. clés A-M sur nœud 1, N-Z sur nœud 2) → risque de hotspot
- **Directory sharding** : Une table de correspondance indique où est chaque donnée → flexible mais bottleneck

### 2.6 Réplication

La **réplication** consiste à copier les données sur plusieurs nœuds pour assurer la haute disponibilité.

- **Facteur de réplication (RF)** : Nombre de copies des données
- **Réplication synchrone** : Le write attend la confirmation de tous les réplicas → cohérence forte mais latence
- **Réplication asynchrone** : Le write retourne avant la propagation → rapide mais cohérence éventuelle

### 2.7 Consistent Hashing

Algorithme utilisé par Cassandra et DynamoDB pour distribuer les données dans un anneau de tokens. Quand un nœud est ajouté/supprimé, seules les données de ses voisins immédiats sont déplacées → minimisation du mouvement de données.

---

## 3. MongoDB

### 3.1 Présentation

**MongoDB** est la base de données NoSQL **orientée documents** la plus populaire au monde. Open-source, elle stocke les données sous forme de documents **BSON** (Binary JSON), flexibles et imbriqués.

- **Créée par :** 10gen (maintenant MongoDB Inc.) — 2009
- **Type :** Document store
- **Langage de requête :** MQL (MongoDB Query Language), JSON-based
- **Positionnement CAP :** CP (par défaut) mais configurable

### 3.2 Concepts Clés

#### La hiérarchie MongoDB
```
Cluster
  └── Base de données (Database)
        └── Collection (≈ table SQL)
              └── Document (≈ ligne SQL) — format BSON
                    └── Champ (Field) — paire clé-valeur
```

#### Le Document BSON
Un document est une structure JSON enrichie. MongoDB stocke en **BSON** (Binary JSON) qui supporte des types supplémentaires :
- `ObjectId` : identifiant unique généré automatiquement (`_id`)
- `Date` : type date natif
- `NumberLong`, `NumberDecimal` : types numériques précis
- `BinData` : données binaires

```json
{
  "_id": ObjectId("64a1f2b3c4d5e6f7g8h9i0j1"),
  "nom": "Ama Kofi",
  "age": 28,
  "email": "ama@example.tg",
  "adresse": {
    "ville": "Lomé",
    "pays": "Togo"
  },
  "competences": ["Python", "MongoDB", "TensorFlow"],
  "actif": true,
  "createdAt": ISODate("2024-01-15T10:30:00Z")
}
```

#### La Collection
- **Sans schéma** (schema-less) : chaque document peut avoir des champs différents
- **Validation optionnelle** : on peut définir un `$jsonSchema` pour valider
- Equivalent d'une table SQL mais sans contrainte de structure

### 3.3 Opérations CRUD

#### Création (Create)
```javascript
// Insérer un document
db.etudiants.insertOne({
  nom: "Kofi Mensah",
  filiere: "IA & Big Data",
  note: 17.5
})

// Insérer plusieurs documents
db.etudiants.insertMany([
  { nom: "Ama Afi", filiere: "Génie Logiciel", note: 15 },
  { nom: "Koffi Ble", filiere: "IA & Big Data", note: 18 }
])
```

#### Lecture (Read)
```javascript
// Trouver tous les documents
db.etudiants.find()

// Avec filtre
db.etudiants.find({ filiere: "IA & Big Data" })

// Avec condition
db.etudiants.find({ note: { $gte: 16 } })

// Avec projection (sélection de champs)
db.etudiants.find({ filiere: "IA & Big Data" }, { nom: 1, note: 1, _id: 0 })

// findOne : retourne le premier résultat
db.etudiants.findOne({ nom: "Kofi Mensah" })
```

#### Mise à jour (Update)
```javascript
// Modifier un seul document
db.etudiants.updateOne(
  { nom: "Kofi Mensah" },
  { $set: { note: 18, statut: "major" } }
)

// Modifier plusieurs documents
db.etudiants.updateMany(
  { filiere: "IA & Big Data" },
  { $inc: { note: 0.5 } }  // Incrémente la note de 0.5
)

// Opérateurs de mise à jour principaux :
// $set    : modifie/ajoute un champ
// $unset  : supprime un champ
// $inc    : incrémente une valeur numérique
// $push   : ajoute un élément à un tableau
// $pull   : retire un élément d'un tableau
// $rename : renomme un champ
```

#### Suppression (Delete)
```javascript
// Supprimer un document
db.etudiants.deleteOne({ nom: "Kofi Mensah" })

// Supprimer plusieurs
db.etudiants.deleteMany({ note: { $lt: 10 } })

// Supprimer toute la collection
db.etudiants.drop()
```

### 3.4 Opérateurs de Requête

#### Opérateurs de comparaison
| Opérateur | Signification | Exemple |
|-----------|--------------|---------|
| `$eq` | Égal à | `{ note: { $eq: 15 } }` |
| `$ne` | Différent de | `{ note: { $ne: 0 } }` |
| `$gt` | Supérieur à | `{ note: { $gt: 14 } }` |
| `$gte` | Supérieur ou égal | `{ note: { $gte: 16 } }` |
| `$lt` | Inférieur à | `{ note: { $lt: 10 } }` |
| `$lte` | Inférieur ou égal | `{ note: { $lte: 5 } }` |
| `$in` | Parmi une liste | `{ filiere: { $in: ["IA", "Info"] } }` |
| `$nin` | Pas dans une liste | `{ statut: { $nin: ["exclu"] } }` |

#### Opérateurs logiques
```javascript
// $and
db.etudiants.find({ $and: [ { note: { $gte: 14 } }, { filiere: "IA & Big Data" } ] })

// $or
db.etudiants.find({ $or: [ { note: { $lt: 10 } }, { absences: { $gt: 5 } } ] })

// $not
db.etudiants.find({ note: { $not: { $gt: 20 } } })
```

#### Requêtes sur tableaux
```javascript
// Chercher un élément dans un tableau
db.etudiants.find({ competences: "Python" })

// $all : tous les éléments doivent être présents
db.etudiants.find({ competences: { $all: ["Python", "MongoDB"] } })

// $elemMatch : condition sur un élément du tableau
db.notes.find({ scores: { $elemMatch: { $gte: 15, $lte: 20 } } })
```

### 3.5 Agrégation (Aggregation Pipeline)

Le **framework d'agrégation** est l'outil le plus puissant de MongoDB pour analyser des données. Il fonctionne comme un pipeline Unix : les documents passent d'une étape à l'autre.

```
Collection → $match → $group → $sort → $limit → Résultat
```

**Étapes principales du pipeline :**

| Étape | Rôle | Equivalent SQL |
|-------|------|---------------|
| `$match` | Filtrer les documents | WHERE |
| `$group` | Regrouper et agréger | GROUP BY |
| `$project` | Sélectionner/transformer des champs | SELECT |
| `$sort` | Trier les résultats | ORDER BY |
| `$limit` | Limiter le nombre de résultats | LIMIT |
| `$skip` | Sauter N documents | OFFSET |
| `$unwind` | Décomposer un tableau en documents | JOIN/LATERAL |
| `$lookup` | Jointure avec une autre collection | JOIN |
| `$count` | Compter les documents | COUNT |

**Exemple complet :**
```javascript
db.etudiants.aggregate([
  // Étape 1 : Filtrer les étudiants actifs
  { $match: { actif: true } },

  // Étape 2 : Regrouper par filière et calculer la moyenne
  { $group: {
    _id: "$filiere",
    moyenneNote: { $avg: "$note" },
    nombreEtudiants: { $sum: 1 },
    noteMax: { $max: "$note" }
  }},

  // Étape 3 : Trier par moyenne décroissante
  { $sort: { moyenneNote: -1 } },

  // Étape 4 : Garder les 3 premières filières
  { $limit: 3 }
])
```

**Opérateurs d'accumulation dans `$group` :**
- `$sum` : Somme
- `$avg` : Moyenne
- `$min` / `$max` : Min/Max
- `$push` : Crée un tableau avec toutes les valeurs
- `$addToSet` : Crée un tableau sans doublons
- `$first` / `$last` : Premier/dernier document du groupe

### 3.6 Indexation

Les **index** accélèrent les requêtes en évitant les scans complets de collection (COLLSCAN).

```javascript
// Créer un index simple
db.etudiants.createIndex({ nom: 1 })  // 1 = ascendant, -1 = descendant

// Index composé
db.etudiants.createIndex({ filiere: 1, note: -1 })

// Index unique
db.etudiants.createIndex({ email: 1 }, { unique: true })

// Index text (full-text search)
db.articles.createIndex({ contenu: "text", titre: "text" })

// Index géospatial
db.lieux.createIndex({ localisation: "2dsphere" })

// Voir les index d'une collection
db.etudiants.getIndexes()

// Supprimer un index
db.etudiants.dropIndex({ nom: 1 })
```

**Types d'index :**
- **Simple** : sur un champ
- **Composé** : sur plusieurs champs (ordre important !)
- **Unique** : garantit l'unicité des valeurs
- **Sparse** : n'indexe que les documents où le champ existe
- **TTL** : supprime automatiquement les documents après un délai
- **Text** : recherche full-text
- **2dsphere** : requêtes géospatiales

### 3.7 Modélisation des Données

MongoDB permet deux stratégies de modélisation :

#### Embedding (Documents imbriqués)
Idéal quand les données sont toujours accédées ensemble et ont une relation 1-N peu volumineuse.

```json
{
  "_id": "commande_001",
  "client": "Kofi Mensah",
  "lignes": [
    { "produit": "Laptop", "quantite": 1, "prix": 450000 },
    { "produit": "Souris", "quantite": 2, "prix": 5000 }
  ]
}
```

✅ **Avantages :** Une seule lecture, pas de jointure, atomicité native  
❌ **Inconvénients :** Documents trop grands, duplication si données partagées

#### Référencing (Références entre collections)
Idéal pour des relations N-N ou des documents trop volumineux.

```json
// Collection "auteurs"
{ "_id": ObjectId("aaa"), "nom": "Ama Kofi" }

// Collection "livres"
{ "_id": ObjectId("bbb"), "titre": "IA pour Tous", "auteur_id": ObjectId("aaa") }
```

Puis utiliser `$lookup` pour les jointures :
```javascript
db.livres.aggregate([
  {
    $lookup: {
      from: "auteurs",
      localField: "auteur_id",
      foreignField: "_id",
      as: "auteurInfo"
    }
  }
])
```

### 3.8 Transactions

Depuis MongoDB 4.0, les **transactions multi-documents** sont supportées (ACID) :

```javascript
const session = db.getMongo().startSession()
session.startTransaction()
try {
  db.comptes.updateOne({ _id: "A" }, { $inc: { solde: -1000 } }, { session })
  db.comptes.updateOne({ _id: "B" }, { $inc: { solde: +1000 } }, { session })
  session.commitTransaction()
} catch (e) {
  session.abortTransaction()
}
```

### 3.9 Architecture : Replica Sets & Sharding

#### Replica Set
Un **Replica Set** est un groupe de nœuds MongoDB qui maintiennent les mêmes données :
- **1 nœud Primary** : reçoit toutes les écritures
- **N nœuds Secondary** : répliques de la primaire (peuvent servir les lectures)
- **Arbiter (optionnel)** : vote lors de l'élection du Primary mais ne stocke pas de données

Si le Primary tombe, une **élection automatique** désigne un nouveau Primary parmi les Secondary.

#### Sharding (Cluster)
Pour distribuer les données :
- **Config Servers** : stockent les métadonnées du cluster
- **Mongos** : routeur de requêtes (point d'entrée)
- **Shard** : nœud (ou Replica Set) stockant une partition des données

**Clé de sharding :** Champ utilisé pour distribuer les données → choix critique pour l'équilibre

### 3.10 Cas d'Usage

MongoDB est idéal pour :
- Applications web (e-commerce, CMS, profils utilisateurs)
- Catalogues de produits (schémas flexibles)
- Stockage de logs
- Applications temps réel (avec Change Streams)
- Applications mobiles

---

## 4. Apache Cassandra

### 4.1 Présentation

**Apache Cassandra** est une base de données NoSQL **distribuée à large colonnes** (wide-column store). Conçue par Facebook en 2008 pour la fonctionnalité "Inbox Search", elle est devenue open-source sous Apache en 2010.

- **Inspirée de :** Google Bigtable (modèle de données) + Amazon Dynamo (architecture distribuée)
- **Type :** Wide-column store
- **Architecture :** Peer-to-peer (pas de master, pas de SPOF)
- **Positionnement CAP :** AP (configurable)
- **Langage de requête :** CQL (Cassandra Query Language)
- **Utilisateurs :** Apple, Netflix, Uber, Instagram

### 4.2 Architecture P2P

#### L'anneau (Ring)
Cassandra organise ses nœuds en **anneau logique**. Chaque nœud est responsable d'une plage de tokens (via consistent hashing). Il n'y a **pas de nœud maître** : tous les nœuds sont identiques.

```
     Nœud A (token 0-25)
    /                    \
Nœud D (token 75-100)   Nœud B (token 25-50)
    \                    /
     Nœud C (token 50-75)
```

#### Nœuds, Data Centers, Clusters
- **Nœud (Node)** : Un serveur dans le cluster
- **Rack** : Groupe de nœuds physiquement proches
- **Data Center** : Groupe de racks (peut être géographique)
- **Cluster** : L'ensemble du système Cassandra

#### Virtual Nodes (Vnodes)
Par défaut, chaque nœud physique est divisé en **256 vnodes**. Cela permet une distribution plus uniforme des données et facilite l'ajout/suppression de nœuds.

### 4.3 Modèle de Données

#### Keyspace
Equivalent d'une base de données. On y définit la stratégie de réplication.

```sql
CREATE KEYSPACE epl_togo
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
};

-- Ou pour plusieurs datacenters
CREATE KEYSPACE epl_togo
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3,
  'DC2': 2
};
```

#### Table (Column Family)
```sql
USE epl_togo;

CREATE TABLE etudiants (
  matricule     UUID,
  annee         INT,
  nom           TEXT,
  filiere       TEXT,
  note          DECIMAL,
  PRIMARY KEY ((matricule), annee)  -- (partition key), clustering key
);
```

#### La Clé Primaire — Concept Central
La clé primaire est composée de :

1. **Partition Key** (clé de partition) : Détermine sur quel nœud la donnée est stockée (via hachage). Peut être composite : `PRIMARY KEY ((champ1, champ2), ...)`.

2. **Clustering Key(s)** (clé de tri) : Tri des données **à l'intérieur** d'une partition. Permet les requêtes par plage sur ces colonnes.

```
┌─────────────────────────────────────────────┐
│ PRIMARY KEY ((partition_key), clustering_k1, clustering_k2)
│               ↑ distribue       ↑ trie à l'intérieur
└─────────────────────────────────────────────┘
```

**Exemple — Modèle pour un système de messages :**
```sql
CREATE TABLE messages (
  conversation_id UUID,         -- Partition key : tous les messages d'une conv. ensemble
  timestamp       TIMESTAMP,    -- Clustering key : triés par date
  user_id         UUID,
  contenu         TEXT,
  PRIMARY KEY (conversation_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

### 4.4 CQL — Cassandra Query Language

Le CQL ressemble au SQL mais avec des restrictions importantes.

#### DDL (Data Definition Language)
```sql
-- Créer une table
CREATE TABLE produits (
  id        UUID PRIMARY KEY,
  nom       TEXT,
  prix      DECIMAL,
  categorie TEXT,
  tags      SET<TEXT>
);

-- Modifier une table (limité : on ne peut qu'ajouter des colonnes)
ALTER TABLE produits ADD stock INT;

-- Supprimer
DROP TABLE produits;
DROP KEYSPACE epl_togo;
```

#### DML (Data Manipulation Language)
```sql
-- Insérer (toujours un UPSERT en Cassandra !)
INSERT INTO etudiants (matricule, annee, nom, filiere, note)
VALUES (uuid(), 2024, 'Ama Kofi', 'IA & Big Data', 17.5);

-- Lire
SELECT * FROM etudiants WHERE matricule = <uuid>;
SELECT nom, note FROM etudiants WHERE matricule = <uuid> AND annee = 2024;

-- Mettre à jour
UPDATE etudiants SET note = 18.0 WHERE matricule = <uuid> AND annee = 2024;

-- Supprimer
DELETE FROM etudiants WHERE matricule = <uuid> AND annee = 2024;
DELETE note FROM etudiants WHERE matricule = <uuid> AND annee = 2024; -- supprimer un champ
```

#### Types de Données CQL
| Catégorie | Types |
|-----------|-------|
| Texte | `TEXT`, `VARCHAR`, `ASCII` |
| Numérique | `INT`, `BIGINT`, `FLOAT`, `DOUBLE`, `DECIMAL`, `VARINT` |
| Booléen | `BOOLEAN` |
| Temporel | `TIMESTAMP`, `DATE`, `TIME`, `DURATION` |
| Identifiant | `UUID`, `TIMEUUID` |
| Collections | `LIST<T>`, `SET<T>`, `MAP<K,V>` |
| Spéciaux | `BLOB`, `COUNTER`, `FROZEN<T>` |

#### Collections CQL
```sql
CREATE TABLE profils (
  user_id    UUID PRIMARY KEY,
  amis       SET<UUID>,          -- Ensemble unique
  favoris    LIST<TEXT>,         -- Liste ordonnée
  metadata   MAP<TEXT, TEXT>     -- Dictionnaire
);

-- Manipulation
UPDATE profils SET amis = amis + {uuid()} WHERE user_id = <uuid>;
UPDATE profils SET favoris = favoris + ['Python'] WHERE user_id = <uuid>;
UPDATE profils SET metadata['role'] = 'admin' WHERE user_id = <uuid>;
```

### 4.5 Cohérence Tunable (Tunable Consistency)

C'est l'une des caractéristiques les plus importantes de Cassandra. Le niveau de cohérence est configurable **par requête**.

**Niveaux principaux :**

| Niveau | Écriture | Lecture | Description |
|--------|---------|---------|-------------|
| `ONE` | 1 nœud | 1 nœud | Rapide, cohérence faible |
| `TWO` | 2 nœuds | 2 nœuds | |
| `THREE` | 3 nœuds | 3 nœuds | |
| `QUORUM` | Majorité (RF/2+1) | Majorité | Bon équilibre |
| `ALL` | Tous | Tous | Cohérence forte, lent |
| `LOCAL_QUORUM` | Majorité dans le DC local | Majorité locale | Multi-datacenter |
| `ANY` | Au moins 1 (hinted handoff) | — | Écriture toujours réussit |

**Formule de cohérence forte :**
```
W + R > RF  →  Lecture/Écriture cohérente
Exemple : RF=3, W=QUORUM(2), R=QUORUM(2) → 2+2=4 > 3 ✅
```

### 4.6 Mécanismes Internes

#### Write Path (Chemin d'écriture)
1. Écriture dans le **CommitLog** (journal WAL persistant)
2. Écriture dans le **MemTable** (mémoire)
3. Quand la MemTable est pleine → **Flush** vers un **SSTable** (fichier immuable sur disque)
4. **Compaction** : Fusion periodique des SSTables pour nettoyer les données obsolètes

#### Read Path (Chemin de lecture)
1. Vérifier le **Row Cache** (si activé)
2. Vérifier la **MemTable**
3. Vérifier le **Bloom Filter** (probabiliste, évite les lectures inutiles sur SSTables)
4. Lire les **SSTables** pertinentes via l'index

#### Tombstones
En Cassandra, une suppression ne supprime pas immédiatement la donnée — elle insère un **tombstone** (marqueur de suppression). Ce tombstone est nettoyé lors de la compaction (après le `gc_grace_seconds`).

### 4.7 Règles de Modélisation Cassandra

La modélisation Cassandra est guidée par les **requêtes**, pas par les entités (inverse du SQL).

**Règles essentielles :**
1. **Modéliser d'abord les requêtes** (Query-Driven Design)
2. **Pas de JOIN** : dénormaliser et dupliquer les données
3. **Pas de sous-requêtes**
4. **WHERE ne peut filtrer que sur la partition key (et les clustering keys)**
5. **ALLOW FILTERING** : permet des filtres sur d'autres colonnes mais coûteux → à éviter
6. **Une partition ne doit pas dépasser 100MB** (problème de "hot partition")

**Anti-patterns :**
- Utiliser `SELECT *` sans spécifier la partition key
- Créer des partitions trop grandes (unbounded partitions)
- Utiliser `ALLOW FILTERING` en production

### 4.8 Cas d'Usage

Cassandra excelle pour :
- Séries temporelles (IoT, métriques, logs)
- Applications à haute disponibilité (99,999% uptime)
- Données réparties géographiquement (multi-datacenter)
- Systèmes de messagerie à grande échelle
- Historiques d'événements

---

## 5. Elasticsearch

### 5.1 Présentation

**Elasticsearch** est un moteur de recherche et d'analytique distribué, construit sur **Apache Lucene**. Il excelle dans la recherche full-text, les analyses de données, et les requêtes complexes sur de grands volumes de données.

- **Créé par :** Shay Banon (2010), maintenu par Elastic
- **Type :** Moteur de recherche / Analytique (orienté document JSON)
- **Protocole :** REST API over HTTP
- **Positionnement CAP :** CP
- **Stack associée :** ELK Stack (Elasticsearch + Logstash + Kibana)

### 5.2 Architecture

#### Cluster, Nœuds, Index
```
Cluster Elasticsearch
  ├── Nœud Master (1)        : Gère les métadonnées du cluster
  ├── Nœud Data (N)          : Stockent les données (shards)
  ├── Nœud Coordonnateur     : Route les requêtes
  └── Nœud Ingest (optionnel): Pre-processing des documents
```

#### Index, Shards, Réplicas
- **Index** : Collection de documents (≈ table/base de données)
- **Shard** : Subdivision d'un index — chaque shard est un **index Lucene** complet
  - **Primary Shard** : Shard principal (reçoit les écritures)
  - **Replica Shard** : Copie d'un primary shard (haute disponibilité + lecture)
- **Formula de routing** : `shard = hash(document_id) % nombre_primary_shards`

**Exemple de configuration :**
```json
PUT /produits
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

> ⚠️ Le nombre de **primary shards est fixé à la création** de l'index et ne peut pas être modifié. Les replicas, eux, peuvent être changés à tout moment.

### 5.3 Mapping (Schéma)

Le **mapping** définit comment les champs d'un document sont indexés et stockés. Elasticsearch peut inférer automatiquement le mapping (**dynamic mapping**) mais il est recommandé de le définir explicitement.

```json
PUT /etudiants
{
  "mappings": {
    "properties": {
      "nom":       { "type": "text", "analyzer": "french" },
      "email":     { "type": "keyword" },
      "note":      { "type": "float" },
      "filiere":   { "type": "keyword" },
      "createdAt": { "type": "date" },
      "tags":      { "type": "keyword" },
      "description": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      }
    }
  }
}
```

#### Types de champs importants
| Type | Usage | Recherche |
|------|-------|----------|
| `text` | Texte long analysé (articles, descriptions) | Full-text, tokenisé |
| `keyword` | Valeurs exactes (email, statut, tags) | Filtrage exact, agrégation |
| `integer`, `float`, `long` | Nombres | Range queries |
| `date` | Dates | Range queries |
| `boolean` | Vrai/Faux | Filtrage |
| `geo_point` | Coordonnées GPS | Géo-requêtes |
| `nested` | Objets imbriqués avec contexte propre | Requêtes nested |

**`text` vs `keyword` — Différence cruciale :**
- `text` → "Lomé, Togo" est tokenisé en ["lomé", "togo"] → recherche full-text
- `keyword` → "Lomé, Togo" reste intact → filtrage exact, tri, agrégations

### 5.4 CRUD avec l'API REST

#### Indexer (Créer/Remplacer)
```bash
# Créer avec ID automatique (POST)
POST /etudiants/_doc
{
  "nom": "Kofi Mensah",
  "filiere": "IA & Big Data",
  "note": 17.5
}

# Créer avec ID spécifique (PUT)
PUT /etudiants/_doc/001
{
  "nom": "Ama Afi",
  "filiere": "Génie Logiciel",
  "note": 15.0
}
```

#### Lire
```bash
# Par ID
GET /etudiants/_doc/001

# Vérifier l'existence
HEAD /etudiants/_doc/001
```

#### Mettre à jour
```bash
# Mise à jour partielle (ne modifie que les champs spécifiés)
POST /etudiants/_update/001
{
  "doc": {
    "note": 16.0
  }
}

# Mise à jour avec script Painless
POST /etudiants/_update/001
{
  "script": {
    "source": "ctx._source.note += params.bonus",
    "params": { "bonus": 1.0 }
  }
}
```

#### Supprimer
```bash
DELETE /etudiants/_doc/001
DELETE /etudiants  # Supprimer l'index entier
```

### 5.5 Recherche — Query DSL

Le **Query DSL** (Domain Specific Language) est au cœur d'Elasticsearch. C'est un JSON décrivant les requêtes.

#### Structure générale
```json
GET /etudiants/_search
{
  "query": { ... },
  "from": 0,
  "size": 10,
  "sort": [...],
  "_source": ["nom", "note"],
  "aggs": { ... }
}
```

#### Query Context vs Filter Context

- **Query Context** : La requête calcule un **score de pertinence** → utilisé pour la recherche full-text
- **Filter Context** : La requête filtre (oui/non) → utilisé pour les valeurs exactes, mise en cache automatique

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "intelligence artificielle" } }  // Query context → score
      ],
      "filter": [
        { "term":  { "filiere": "IA & Big Data" } },                // Filter context → pas de score
        { "range": { "note": { "gte": 14 } } }                     // Filter context → pas de score
      ]
    }
  }
}
```

#### Requêtes Principales

**match** — Full-text search
```json
{ "match": { "nom": "kofi mensah" } }
// Cherche les documents contenant "kofi" OU "mensah"

{ "match": { "nom": { "query": "kofi mensah", "operator": "and" } } }
// Cherche "kofi" ET "mensah"
```

**term / terms** — Valeur exacte (sur keyword)
```json
{ "term": { "filiere": "IA & Big Data" } }
{ "terms": { "filiere": ["IA & Big Data", "Génie Logiciel"] } }
```

**range** — Plage de valeurs
```json
{ "range": { "note": { "gte": 14, "lte": 20 } } }
{ "range": { "date": { "gte": "2024-01-01", "lt": "2025-01-01" } } }
```

**bool** — Combinaison logique
```json
{
  "bool": {
    "must":     [...],  // ET, contribue au score
    "should":   [...],  // OU, améliore le score
    "must_not": [...],  // NOT, ne contribue pas au score
    "filter":   [...]   // ET strict, pas de score
  }
}
```

**match_phrase** — Recherche de phrase exacte
```json
{ "match_phrase": { "description": "base de données" } }
```

**multi_match** — Recherche sur plusieurs champs
```json
{
  "multi_match": {
    "query": "IA machine learning",
    "fields": ["titre^3", "description"]  // ^3 = boost le champ titre
  }
}
```

**wildcard / regexp**
```json
{ "wildcard": { "nom": "kof*" } }
{ "regexp": { "email": ".*@epl.tg" } }
```

### 5.6 Agrégations

Les **agrégations** permettent des analyses statistiques sur les données.

```json
GET /etudiants/_search
{
  "size": 0,
  "aggs": {
    "par_filiere": {
      "terms": { "field": "filiere" },    // Bucket par filière
      "aggs": {
        "note_moyenne": { "avg": { "field": "note" } },
        "note_max":     { "max": { "field": "note" } }
      }
    },
    "distribution_notes": {
      "histogram": {
        "field": "note",
        "interval": 5
      }
    }
  }
}
```

**Types d'agrégations :**
- **Bucket** : Groupent les documents (`terms`, `range`, `date_histogram`, `histogram`, `nested`)
- **Metric** : Calculent des statistiques (`avg`, `sum`, `min`, `max`, `stats`, `cardinality`)
- **Pipeline** : Agrégations sur des agrégations (`moving_avg`, `derivative`)

### 5.7 Analyseurs (Analyzers)

Un **analyzer** transforme le texte en tokens lors de l'indexation et de la recherche.

```
"Bonjour Monde!" 
   → [character filter] → "Bonjour Monde"
   → [tokenizer]        → ["Bonjour", "Monde"]
   → [token filters]    → ["bonjour", "monde"]  (lowercase)
```

**Composants :**
- **Character filters** : Nettoient le texte brut (ex: `html_strip`)
- **Tokenizer** : Découpe en tokens (ex: `standard`, `whitespace`)
- **Token filters** : Transforment les tokens (ex: `lowercase`, `stop`, `stemmer`)

**Analyzers built-in :**
```json
{ "type": "standard" }   // Tokenisation standard + lowercase
{ "type": "french"   }   // Stemming français (chercher → cherch)
{ "type": "english"  }   // Stemming anglais
{ "type": "keyword"  }   // Pas de tokenisation
```

### 5.8 Cas d'Usage

Elasticsearch est idéal pour :
- Moteurs de recherche applicatifs (e-commerce, sites)
- Analyse de logs (ELK Stack : Logstash → Elasticsearch → Kibana)
- Monitoring et observabilité (métriques, traces, APM)
- Recherche full-text multilingue
- Géo-recherche et recherche approximative

---

## 6. Neo4j

### 6.1 Présentation

**Neo4j** est la base de données NoSQL **orientée graphe** la plus utilisée. Elle modélise les données et leurs relations comme un graphe mathématique, ce qui la rend exceptionnellement performante pour les requêtes traversant de nombreuses relations.

- **Créée par :** Neo Technology (2007), maintenant Neo4j Inc.
- **Type :** Graph database
- **Langage de requête :** Cypher
- **Licences :** Community (GPLv3 — 1 nœud) / Enterprise
- **Cloud :** AuraDB (SaaS managé)
- **Positionnement CAP :** CA/CP selon configuration

### 6.2 Le Modèle de Données Graphe

#### Éléments fondamentaux

1. **Nœuds (Nodes)** : Les entités (personnes, produits, villes…)
   - Peuvent avoir des **labels** (catégories) : `:Person`, `:Movie`, `:City`
   - Peuvent avoir des **propriétés** : `{name: 'Alice', age: 30}`

2. **Relations (Relationships)** : Les connexions entre nœuds
   - **Dirigées** (ont un sens : A → B)
   - **Typées** : `:KNOWS`, `:ACTED_IN`, `:LIVES_IN`
   - Peuvent avoir des **propriétés** : `{since: 2020, weight: 0.8}`

3. **Propriétés** : Paires clé-valeur sur nœuds et relations

```
(Alice:Person {age:30}) --[:KNOWS {since:2020}]--> (Bob:Person {age:25})
     |
     [:LIVES_IN]
     ↓
(Lomé:City {country:"Togo"})
```

#### Pourquoi un graphe ?
Dans un SGBDR, une requête type "amis des amis" nécessite des JOIN exponentiels. Dans Neo4j, la traversée est directe car les relations sont stockées physiquement avec les nœuds → performance constante même à grande profondeur.

### 6.3 Cypher — Le Langage de Requête

**Cypher** est un langage déclaratif et intuitif. Sa syntaxe "dessine" le graphe que l'on cherche.

- `()` = nœud
- `[]` = relation
- `-->` = direction de la relation

#### Créer des données (CREATE / MERGE)
```cypher
// Créer des nœuds
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 25})
CREATE (lome:City {name: 'Lomé', country: 'Togo'})

// Créer des relations
CREATE (alice)-[:KNOWS {since: 2020}]->(bob)
CREATE (alice)-[:LIVES_IN]->(lome)

// MERGE : Crée si n'existe pas, sinon ne fait rien (idempotent)
MERGE (p:Person {name: 'Charlie'})
ON CREATE SET p.created = timestamp()
ON MATCH  SET p.lastSeen = timestamp()
```

#### Lire des données (MATCH)
```cypher
// Trouver tous les nœuds Person
MATCH (p:Person)
RETURN p.name, p.age

// Trouver une personne spécifique
MATCH (p:Person {name: 'Alice'})
RETURN p

// Trouver les amis d'Alice
MATCH (alice:Person {name: 'Alice'})-[:KNOWS]->(ami:Person)
RETURN ami.name

// Trouver dans quelle ville habite Alice
MATCH (p:Person {name: 'Alice'})-[:LIVES_IN]->(c:City)
RETURN p.name, c.name

// Filtrer avec WHERE
MATCH (p:Person)
WHERE p.age > 25 AND p.age < 40
RETURN p.name, p.age
ORDER BY p.age DESC
LIMIT 10
```

#### Traversées avancées
```cypher
// Amis des amis (profondeur 2)
MATCH (alice:Person {name: 'Alice'})-[:KNOWS*2]->(ami_ami:Person)
RETURN ami_ami.name

// Traversée variable (1 à 3 hops)
MATCH (a:Person {name: 'Alice'})-[:KNOWS*1..3]->(b:Person)
RETURN b.name, length(()-[:KNOWS*1..3]-()) AS distance

// Chemin le plus court
MATCH p = shortestPath(
  (alice:Person {name: 'Alice'})-[:KNOWS*]-(bob:Person {name: 'Bob'})
)
RETURN p, length(p)

// Tous les chemins les plus courts
MATCH p = allShortestPaths(
  (a:Person {name: 'Alice'})-[:KNOWS*]-(b:Person {name: 'Bob'})
)
RETURN p
```

#### Mettre à jour et supprimer
```cypher
// Modifier des propriétés
MATCH (p:Person {name: 'Alice'})
SET p.email = 'alice@epl.tg'
SET p.age = 31

// Ajouter un label
MATCH (p:Person {name: 'Alice'})
SET p:Admin

// Supprimer une propriété
MATCH (p:Person {name: 'Alice'})
REMOVE p.age

// Supprimer un nœud (doit être isolé)
MATCH (p:Person {name: 'Charlie'})
DELETE p

// Supprimer un nœud et ses relations
MATCH (p:Person {name: 'Charlie'})
DETACH DELETE p

// Supprimer une relation
MATCH (a)-[r:KNOWS]->(b)
WHERE a.name = 'Alice' AND b.name = 'Bob'
DELETE r
```

### 6.4 Clauses Importantes

| Clause | Rôle |
|--------|------|
| `MATCH` | Chercher des patterns dans le graphe |
| `WHERE` | Filtrer les résultats du MATCH |
| `RETURN` | Spécifier ce qu'on retourne |
| `WITH` | Passer des résultats intermédiaires entre clauses |
| `ORDER BY` | Trier les résultats |
| `LIMIT` | Limiter le nombre de résultats |
| `SKIP` | Sauter N premiers résultats |
| `CREATE` | Créer nœuds et relations |
| `MERGE` | Créer ou trouver (idempotent) |
| `SET` | Modifier/ajouter des propriétés |
| `REMOVE` | Supprimer des propriétés ou labels |
| `DELETE` | Supprimer nœuds ou relations |
| `DETACH DELETE` | Supprimer un nœud et toutes ses relations |
| `UNWIND` | Décomposer une liste en lignes |
| `FOREACH` | Itérer sur une liste pour des effets de bord |
| `CALL` | Appeler une procédure ou sous-requête |

### 6.5 Fonctions Utiles
```cypher
// Fonctions de chaîne
RETURN toUpper("hello"), toLower("WORLD"), trim("  abc  "), size("hello")

// Fonctions numériques
RETURN abs(-5), ceil(3.2), floor(3.9), round(3.5), sqrt(16)

// Fonctions de liste
RETURN [1,2,3,4,5] AS nums,
       size([1,2,3]) AS taille,
       head([1,2,3]),    // Premier élément
       tail([1,2,3]),    // Tout sauf le premier
       reverse([1,2,3])

// Fonctions d'agrégation
MATCH (p:Person)
RETURN count(p), avg(p.age), min(p.age), max(p.age), collect(p.name)

// Fonctions de graphe
MATCH (n)-[r]-(m)
RETURN type(r),        // Type de la relation
       labels(n),      // Labels du nœud
       keys(n),        // Propriétés du nœud
       id(n)           // ID interne du nœud
```

### 6.6 Index et Contraintes
```cypher
// Créer un index
CREATE INDEX FOR (p:Person) ON (p.name)

// Créer un index composé
CREATE INDEX FOR (p:Person) ON (p.name, p.age)

// Contrainte d'unicité (crée aussi un index)
CREATE CONSTRAINT FOR (p:Person) REQUIRE p.email IS UNIQUE

// Contrainte d'existence
CREATE CONSTRAINT FOR (p:Person) REQUIRE p.name IS NOT NULL

// Voir les index et contraintes
SHOW INDEXES
SHOW CONSTRAINTS
```

### 6.7 Algorithmes de Graphe (GDS)

La bibliothèque **Graph Data Science (GDS)** de Neo4j fournit des algorithmes avancés :

**Centralité :**
- **PageRank** : Importance d'un nœud basée sur ses connexions (comme Google)
- **Betweenness Centrality** : Nœuds sur le plus grand nombre de chemins courts
- **Degree Centrality** : Nombre de relations d'un nœud

**Détection de Communautés :**
- **Louvain** : Détection de communautés (clustering)
- **Label Propagation** : Propagation de labels pour le clustering

**Parcours de Graphe :**
- **Shortest Path** : Chemin le plus court (Dijkstra, A*)
- **BFS / DFS** : Parcours en largeur / profondeur

### 6.8 Cas d'Usage

Neo4j est idéal pour :
- Réseaux sociaux (amis, recommandations)
- Détection de fraude bancaire (graphe de transactions)
- Graphes de connaissances (Knowledge Graphs)
- Systèmes de recommandation
- Gestion d'accès basé sur les rôles (RBAC)
- Réseaux logistiques, approvisionnement, dépendances

---

## 7. CouchDB

### 7.1 Présentation

**Apache CouchDB** est une base de données NoSQL **orientée documents**, développée en **Erlang** (langage conçu pour les systèmes concurrents et haute disponibilité). Elle suit une philosophie "offline-first" et expose ses données via une **API HTTP/REST**.

- **Créée par :** Damien Katz (2005), Apache Software Foundation (2008)
- **Type :** Document store
- **Protocole :** HTTP/REST natif
- **Format :** JSON
- **Positionnement CAP :** AP (disponibilité + tolérance aux partitions)
- **Particularité :** MVCC, réplication multi-maître, offline-first

### 7.2 Concepts Fondamentaux

#### Structure d'un Document
Chaque document CouchDB est un objet JSON avec deux champs obligatoires :
- `_id` : Identifiant unique du document (UUID par défaut)
- `_rev` : Numéro de révision (ex: `3-a7f5e...`) → mécanisme de versioning

```json
{
  "_id": "etudiant_001",
  "_rev": "2-f3a9b7c123d456e789",
  "nom": "Kofi Mensah",
  "filiere": "IA & Big Data",
  "note": 17.5,
  "annee": 2024
}
```

Le champ `_rev` suit le format `N-hash` où N est le numéro de révision.

#### MVCC — Multi-Version Concurrency Control

C'est le mécanisme central de CouchDB pour gérer la concurrence :
- **Pas de verrou** : Les lectures ne bloquent jamais les écritures et vice-versa
- **Versioning** : Chaque modification crée une nouvelle révision (`_rev`)
- **Cohérence par document** : Les propriétés ACID sont garanties au niveau document
- **Snapshot isolé** : Chaque lecture voit un snapshot cohérent de la base

**Processus de mise à jour :**
```
Lire document (version 1-abc)
  ↓
Modifier en mémoire
  ↓
Sauvegarder avec _rev: "1-abc" (le _rev actuel doit correspondre)
  ↓
Si succès → nouvelle version "2-xyz" créée
Si conflit (_rev ne correspond plus) → erreur 409 Conflict → relire et réessayer
```

### 7.3 API HTTP REST

Toutes les opérations se font via HTTP standard.

#### Gestion de la Base de Données
```bash
# Créer une base de données
curl -X PUT http://localhost:5984/etudiants

# Lister les bases
curl http://localhost:5984/_all_dbs

# Informations sur la base
curl http://localhost:5984/etudiants

# Supprimer la base
curl -X DELETE http://localhost:5984/etudiants
```

#### CRUD sur les Documents
```bash
# Créer un document (ID automatique)
curl -X POST http://localhost:5984/etudiants \
  -H "Content-Type: application/json" \
  -d '{"nom": "Ama Afi", "note": 15}'

# Créer avec ID spécifique
curl -X PUT http://localhost:5984/etudiants/etud_001 \
  -H "Content-Type: application/json" \
  -d '{"nom": "Ama Afi", "note": 15}'

# Lire un document
curl http://localhost:5984/etudiants/etud_001

# Mettre à jour (DOIT inclure le _rev actuel)
curl -X PUT http://localhost:5984/etudiants/etud_001 \
  -H "Content-Type: application/json" \
  -d '{"_rev": "1-abc123", "nom": "Ama Afi", "note": 17}'

# Supprimer
curl -X DELETE "http://localhost:5984/etudiants/etud_001?rev=2-xyz789"
```

### 7.4 Vues et MapReduce

Les **vues** sont le mécanisme principal de requêtage dans CouchDB. Elles sont définies comme des fonctions JavaScript MapReduce dans des **Design Documents**.

#### Design Document
```json
{
  "_id": "_design/etudiants_views",
  "views": {
    "par_filiere": {
      "map": "function(doc) { if(doc.filiere) { emit(doc.filiere, doc.note); } }",
      "reduce": "_stats"
    },
    "notes_eleves": {
      "map": "function(doc) { if(doc.note && doc.note >= 14) { emit(doc.note, doc.nom); } }"
    }
  }
}
```

**Fonction Map :** Reçoit chaque document et émet des paires `(clé, valeur)` à indexer.
**Fonction Reduce :** Agrège les valeurs émises par Map.

**Reducers built-in :**
- `_count` : Compte les documents
- `_sum` : Somme des valeurs
- `_stats` : Statistiques (count, sum, min, max, sumsqr)

#### Requêter une Vue
```bash
# Requête simple
curl http://localhost:5984/etudiants/_design/etudiants_views/_view/par_filiere

# Avec filtres
curl "http://localhost:5984/etudiants/_design/etudiants_views/_view/par_filiere?key=%22IA+%26+Big+Data%22"

# Plage de clés
curl "http://localhost:5984/etudiants/_design/etudiants_views/_view/notes_eleves?startkey=14&endkey=20"

# Avec réduction
curl "http://localhost:5984/etudiants/_design/etudiants_views/_view/par_filiere?group=true"
```

### 7.5 Mango Query (depuis CouchDB 2.0)

CouchDB 2.0 a introduit **Mango**, un système de requêtes JSON sans MapReduce :

```bash
# Requête Mango
curl -X POST http://localhost:5984/etudiants/_find \
  -H "Content-Type: application/json" \
  -d '{
    "selector": {
      "filiere": "IA & Big Data",
      "note": { "$gte": 14 }
    },
    "fields": ["nom", "note", "filiere"],
    "sort": [{ "note": "desc" }],
    "limit": 10
  }'

# Créer un index pour Mango
curl -X POST http://localhost:5984/etudiants/_index \
  -H "Content-Type: application/json" \
  -d '{
    "index": { "fields": ["filiere", "note"] },
    "name": "filiere-note-index",
    "type": "json"
  }'
```

### 7.6 Réplication

La **réplication** est une fonctionnalité clé de CouchDB. Elle est **bidirectionnelle, incrémentale et multi-maître**.

```bash
# Réplication one-shot : source → destination
curl -X POST http://localhost:5984/_replicate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "http://localhost:5984/etudiants",
    "target": "http://serveur2:5984/etudiants"
  }'

# Réplication continue
curl -X POST http://localhost:5984/_replicate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "etudiants",
    "target": "http://serveur2:5984/etudiants",
    "continuous": true
  }'
```

**Gestion des conflits de réplication :**
Quand deux nœuds modifient le même document hors-ligne, CouchDB crée un **conflit**. Il choisit algorithmiquement un "gagnant" (déterministe) mais conserve l'autre version. L'application doit résoudre le conflit manuellement.

```bash
# Voir les conflits
curl "http://localhost:5984/etudiants/_all_docs?conflicts=true"

# Obtenir toutes les révisions en conflit
curl "http://localhost:5984/etudiants/etud_001?conflicts=true"
```

### 7.7 CouchDB vs PouchDB

**PouchDB** est une implémentation JavaScript de CouchDB qui fonctionne dans le navigateur (IndexedDB) ou Node.js. Elle se synchronise automatiquement avec CouchDB.

C'est la base du modèle **offline-first** : l'application fonctionne sans réseau et se synchronise quand la connexion revient.

### 7.8 Cas d'Usage

CouchDB excelle pour :
- Applications offline-first (mobile, zones à faible connectivité)
- Synchronisation décentralisée entre appareils
- Stockage de données avec historique des versions
- Applications distribuées multi-sites
- Gestion de contenu avec workflow de révision

---

## 8. Amazon DynamoDB

### 8.1 Présentation

**Amazon DynamoDB** est une base de données NoSQL **entièrement managée** (serverless) par AWS. C'est une solution **clé-valeur et document** qui offre des performances en millisecondes quelle que soit l'échelle.

- **Créée par :** Amazon Web Services (2012)
- **Inspirée de :** Amazon Dynamo (article 2007)
- **Type :** Clé-valeur + Document (hybride)
- **Gestion :** Entièrement serverless (pas de serveur à gérer)
- **Positionnement CAP :** AP par défaut (cohérence éventuelle), CP optionnel (lectures fortement cohérentes)
- **Tarification :** Pay-per-use (RCU/WCU ou On-Demand)

### 8.2 Modèle de Données

#### Tables
L'unité de base dans DynamoDB est la **table**. Pas de base de données au sens traditionnel — les tables existent directement.

#### Éléments (Items) et Attributs
- **Item** : Une ligne dans la table (équivalent d'un document JSON)
- **Attribut** : Un champ dans l'item (clé-valeur)
- **Taille max par item** : 400 KB

```json
{
  "PK": "USER#kofi123",
  "SK": "PROFILE",
  "nom": "Kofi Mensah",
  "email": "kofi@epl.tg",
  "filiere": "IA & Big Data",
  "createdAt": "2024-01-15"
}
```

### 8.3 Clés Primaires — Concept Central

#### Option 1 : Clé de Partition Simple (Simple Primary Key)
Seul le **Partition Key** (PK). Doit être unique pour chaque item.

```
Table: Utilisateurs
  Partition Key: userId (unique)
  
  userId = "u001" → Item complet
  userId = "u002" → Item complet
```

#### Option 2 : Clé Composite (Composite Primary Key)
**Partition Key** + **Sort Key** (SK). La combinaison PK+SK doit être unique.

```
Table: Commandes
  PK = customerId (partitionne les données)
  SK = orderId    (trie au sein de la partition)
  
  PK="c001", SK="ord-2024-001" → Commande 1
  PK="c001", SK="ord-2024-002" → Commande 2
  PK="c002", SK="ord-2024-003" → Commande 3
```

**Avantage :** Toutes les commandes d'un client sont dans la même partition → lecture efficace.

#### Comment DynamoDB distribue les données
```
hash(PK) → partition physique → storage
```
Les items avec le même PK sont stockés ensemble et triés par SK.

### 8.4 Index Secondaires

#### Local Secondary Index (LSI)
- **Même Partition Key** que la table
- **Sort Key différent**
- Doit être créé **à la création de la table** (pas modifiable)
- Partage la capacité (WCU/RCU) avec la table
- Limite : **10 GB par partition key** (collection item limit)

```json
// Table: Commandes (PK=customerId, SK=orderId)
// LSI: (PK=customerId, SK=orderDate)
// → Permet de récupérer les commandes d'un client triées par date
```

#### Global Secondary Index (GSI)
- **Partition Key ET/OU Sort Key différents** de la table
- Peut être créé ou supprimé à tout moment
- A sa **propre capacité** (WCU/RCU séparée)
- **Portée globale** : span tous les partitions de la table
- Pas de limite de 10GB
- Maximum **20 GSI par table** (par défaut)

```json
// Table: Commandes (PK=customerId, SK=orderId)
// GSI: (PK=status, SK=orderDate)
// → Permet de récupérer toutes les commandes "expédiées" triées par date
```

**Résumé LSI vs GSI :**
| Critère | LSI | GSI |
|---------|-----|-----|
| Partition Key | Même que table | Différent possible |
| Sort Key | Différent | Différent possible |
| Création | À la création de la table | N'importe quand |
| Capacité | Partagée avec table | Indépendante |
| Cohérence | Forte ou éventuelle | Éventuelle uniquement |
| Limite par partition | 10 GB | Illimité |

### 8.5 Opérations Principales

#### Écriture
```python
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Etudiants')

# PutItem : Crée ou remplace complètement
table.put_item(Item={
    'PK': 'STUDENT#kofi123',
    'SK': 'PROFILE',
    'nom': 'Kofi Mensah',
    'note': Decimal('17.5')
})

# UpdateItem : Mise à jour partielle
table.update_item(
    Key={'PK': 'STUDENT#kofi123', 'SK': 'PROFILE'},
    UpdateExpression='SET note = :val, statut = :s',
    ExpressionAttributeValues={':val': Decimal('18'), ':s': 'major'}
)

# DeleteItem
table.delete_item(Key={'PK': 'STUDENT#kofi123', 'SK': 'PROFILE'})
```

#### Lecture
```python
# GetItem : Lecture par clé exacte (le plus efficace)
response = table.get_item(
    Key={'PK': 'STUDENT#kofi123', 'SK': 'PROFILE'}
)
item = response['Item']

# Query : Tous les items d'une partition (+ conditions sur SK)
from boto3.dynamodb.conditions import Key, Attr
response = table.query(
    KeyConditionExpression=Key('PK').eq('CUSTOMER#c001')
      & Key('SK').begins_with('ORDER#')
)

# Query avec filtre
response = table.query(
    KeyConditionExpression=Key('PK').eq('CUSTOMER#c001'),
    FilterExpression=Attr('status').eq('shipped')
)

# Scan : Parcourt TOUTE la table (coûteux, à éviter)
response = table.scan(
    FilterExpression=Attr('filiere').eq('IA & Big Data')
)
```

### 8.6 Cohérence

- **Lecture éventuellement cohérente** (par défaut) : Peut retourner des données obsolètes de quelques secondes. Coûte **0.5 RCU** par 4KB.
- **Lecture fortement cohérente** : Retourne la donnée la plus récente. Coûte **1 RCU** par 4KB.

```python
# Lecture fortement cohérente
response = table.get_item(
    Key={'PK': 'STUDENT#kofi123', 'SK': 'PROFILE'},
    ConsistentRead=True
)
```

### 8.7 Capacity Units (Capacité)

DynamoDB mesure la capacité en **unités** :

- **RCU (Read Capacity Unit)** : 1 lecture fortement cohérente de 4KB/sec (ou 2 lectures éventuelles)
- **WCU (Write Capacity Unit)** : 1 écriture de 1KB/sec

**Modes de tarification :**
- **Provisioned** : On définit un nombre de RCU/WCU → moins cher mais risque de throttling
- **On-Demand** : Paiement à l'utilisation → plus flexible, plus cher

### 8.8 DynamoDB Streams

Les **Streams** capturent les modifications de la table dans un journal ordonné, utilisable pour :
- Déclencher des Lambdas (event-driven)
- Répliquer les données
- Construire des vues dérivées

### 8.9 Transactions

DynamoDB supporte les transactions ACID multi-items :
```python
dynamodb.meta.client.transact_write(
    TransactItems=[
        {
            'Update': {
                'TableName': 'Comptes',
                'Key': {'PK': 'ACCOUNT#A'},
                'UpdateExpression': 'SET solde = solde - :montant',
                'ExpressionAttributeValues': {':montant': Decimal('1000')}
            }
        },
        {
            'Update': {
                'TableName': 'Comptes',
                'Key': {'PK': 'ACCOUNT#B'},
                'UpdateExpression': 'SET solde = solde + :montant',
                'ExpressionAttributeValues': {':montant': Decimal('1000')}
            }
        }
    ]
)
```

### 8.10 Single-Table Design

Le pattern **Single-Table Design** consiste à mettre toutes les entités dans une seule table DynamoDB, en utilisant des PK/SK génériques :

```
PK                  SK                  Données
USER#kofi123        PROFILE             {nom, email, ...}
USER#kofi123        ORDER#2024-001      {montant, statut, ...}
USER#kofi123        ORDER#2024-002      {montant, statut, ...}
PRODUCT#p001        DETAILS             {nom, prix, ...}
PRODUCT#p001        REVIEW#r001         {note, commentaire, ...}
```

**Avantage :** Une seule requête pour récupérer une entité avec toutes ses relations.

### 8.11 Cas d'Usage

DynamoDB excelle pour :
- Applications serverless AWS (Lambda, API Gateway)
- Gaming (classements, sessions, profils)
- E-commerce (paniers, sessions)
- Applications mobiles (profils, données utilisateur)
- IoT (séries temporelles)
- Tout workload nécessitant une latence sub-milliseconde à grande échelle

---

## 9. Comparatif Global

### 9.1 Tableau Comparatif

| Critère | MongoDB | Cassandra | Elasticsearch | Neo4j | CouchDB | DynamoDB |
|---------|---------|-----------|---------------|-------|---------|----------|
| **Type** | Document | Wide-column | Recherche/Analytique | Graphe | Document | Clé-Valeur + Document |
| **CAP** | CP | AP | CP | CA/CP | AP | AP (configurable) |
| **Langage** | MQL (JSON) | CQL | JSON/DSL REST | Cypher | HTTP/REST + JS | SDK/API AWS |
| **Schéma** | Flexible | Défini (table) | Flexible (mapping) | Schema-optional | Flexible | Flexible |
| **ACID** | Multi-doc (v4+) | Non (par défaut) | Non | Oui | Par document | Oui (transactions) |
| **Scalabilité** | Sharding | Linéaire P2P | Sharding | Clustering Enterprise | Multi-maître | Serverless auto |
| **Réplication** | Replica Set | P2P (RF configurable) | Primary-Replica | Native/Enterprise | Multi-maître | Automatique |
| **Recherche** | Basique | Basique | Excellente (full-text) | Traversée de graphe | MapReduce/Mango | Basique |
| **Jointures** | `$lookup` | ❌ Pas de JOIN | ❌ | Traversée native | ❌ | ❌ |
| **Transactions** | Oui (v4+) | Légères (batch) | ❌ | Oui | Par doc | Oui |
| **Licence** | SSPL / Commercial | Apache 2.0 | Elastic/Apache | GPLv3 / Commercial | Apache 2.0 | Propriétaire AWS |
| **Hébergement** | Atlas (cloud) | DataStax / Auto-hébergé | Elastic Cloud / Auto | AuraDB / Auto | Auto-hébergé | AWS uniquement |

### 9.2 Quand Utiliser Quoi ?

| Besoin | Recommandation |
|--------|---------------|
| Documents JSON flexibles, développement rapide | **MongoDB** |
| Haute disponibilité absolue, écriture massive, IoT | **Cassandra** |
| Recherche full-text, logs, analytique, ELK | **Elasticsearch** |
| Relations complexes, réseaux sociaux, fraude | **Neo4j** |
| Offline-first, synchronisation mobile, révisions | **CouchDB** |
| Serverless AWS, latence ultra-faible, pay-per-use | **DynamoDB** |

### 9.3 Positionnement CAP Visuel

```
         Cohérence (C)
              ●  MongoDB (CP)
             /●  Elasticsearch (CP)
            / 
           /
──────────────────────────────
          /\
     CP  /  \  CA
        /    \  (SGBDR classiques)
       /  ???  \
──────────────────────────────
  AP  ●  Cassandra
      ●  CouchDB
      ●  DynamoDB (par défaut)

  Disponibilité (A) ←————————→ Tolérance aux Partitions (P)
```

### 9.4 Comparaison des Modèles de Données

```
SQL (Relationnel)          MongoDB (Document)
─────────────────          ──────────────────
Base de données      →     Base de données
Table                →     Collection
Ligne                →     Document (BSON)
Colonne              →     Champ (Field)
PRIMARY KEY          →     _id (ObjectId)
JOIN                 →     $lookup / Embedded
INDEX                →     createIndex()
VIEW                 →     Agrégation pipeline


Cassandra (Wide-Column)    Neo4j (Graphe)
───────────────────────    ──────────────
Keyspace             →     Graphe
Table (Column Family)→     Labels de Nœuds
Row                  →     Nœud
Column               →     Propriété
PRIMARY KEY          →     ID interne
Partition Key        →     Pas d'équivalent direct
Clustering Key       →     Ordre de traversée
— (pas de JOIN)      →     Relation (->)
```

---

## 10. Questions d'Examen Types

### 10.1 Questions Théoriques

**Q1 : Expliquez le théorème CAP et positionnez MongoDB, Cassandra et CouchDB.**

> Le théorème CAP stipule qu'un système distribué ne peut garantir simultanément que 2 des 3 propriétés : Cohérence, Disponibilité, Tolérance aux Partitions. MongoDB est CP (cohérence + tolérance, sacrifie disponibilité lors de partitions). Cassandra est AP (disponibilité + tolérance, cohérence éventuelle). CouchDB est AP avec réplication multi-maître et résolution de conflits.

**Q2 : Quelle est la différence entre ACID et BASE ?**

> ACID (Atomicity, Consistency, Isolation, Durability) garantit des transactions strictes → modèle SQL. BASE (Basically Available, Soft state, Eventually consistent) accepte une cohérence éventuelle en échange de disponibilité et scalabilité → modèle NoSQL dominant.

**Q3 : Qu'est-ce que le sharding et pourquoi est-il important ?**

> Le sharding est la partition horizontale des données sur plusieurs nœuds. Il permet de dépasser les limites d'un seul serveur en répartissant les données et la charge. C'est fondamental pour la scalabilité des bases NoSQL à grande échelle.

**Q4 : Expliquez la différence entre réplication synchrone et asynchrone.**

> Synchrone : le write attend la confirmation de tous les réplicas avant de retourner → cohérence forte mais latence élevée. Asynchrone : le write retourne immédiatement après l'écriture sur le nœud primaire, les réplicas sont mis à jour en arrière-plan → faible latence mais cohérence éventuelle.

**Q5 : Pourquoi ne peut-on pas faire de JOIN dans Cassandra ?**

> Cassandra est optimisée pour des lectures rapides sur une partition. Les JOIN nécessiteraient des communications entre nœuds différents, ce qui contredirait l'architecture distribuée et les objectifs de performance. On compense en dénormalisant les données (duplication) selon les patterns de requêtes.

### 10.2 Questions Pratiques MongoDB

**Q : Écrire une requête MongoDB pour trouver tous les étudiants avec une note entre 14 et 16, triés par note décroissante.**

```javascript
db.etudiants.find(
  { note: { $gte: 14, $lte: 16 } },
  { nom: 1, note: 1, _id: 0 }
).sort({ note: -1 })
```

**Q : Écrire un pipeline d'agrégation pour calculer le nombre d'étudiants et la note moyenne par filière.**

```javascript
db.etudiants.aggregate([
  { $group: {
    _id: "$filiere",
    nbEtudiants: { $sum: 1 },
    moyenneNote: { $avg: "$note" }
  }},
  { $sort: { moyenneNote: -1 } }
])
```

### 10.3 Questions Pratiques Cassandra

**Q : Créer une table Cassandra pour stocker les événements IoT (deviceId, timestamp, valeur). Expliquer le choix de la clé primaire.**

```sql
CREATE TABLE iot_events (
  device_id  TEXT,
  event_time TIMESTAMP,
  valeur     DOUBLE,
  unite      TEXT,
  PRIMARY KEY (device_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);
```
> **Explication :** `device_id` comme partition key regroupe tous les événements d'un appareil sur le même nœud. `event_time` comme clustering key trie les événements chronologiquement (DESC pour avoir les plus récents en premier). Cela permet des requêtes efficaces de type "derniers N événements de l'appareil X".

### 10.4 Questions Pratiques Elasticsearch

**Q : Écrire une requête ES pour chercher des articles contenant "machine learning" dans le titre ou la description, filtrés sur la catégorie "IA", avec note >= 4.**

```json
GET /articles/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "machine learning",
            "fields": ["titre^2", "description"]
          }
        }
      ],
      "filter": [
        { "term": { "categorie": "IA" } },
        { "range": { "note": { "gte": 4 } } }
      ]
    }
  }
}
```

### 10.5 Questions Pratiques Neo4j

**Q : Requête Cypher pour trouver les amis d'amis d'Alice qui ne sont pas déjà amis avec elle.**

```cypher
MATCH (alice:Person {name: 'Alice'})-[:KNOWS]->(ami)-[:KNOWS]->(ami_ami)
WHERE NOT (alice)-[:KNOWS]->(ami_ami)
  AND ami_ami <> alice
RETURN DISTINCT ami_ami.name
```

**Q : Trouver le chemin le plus court entre deux personnes.**

```cypher
MATCH p = shortestPath(
  (a:Person {name: 'Alice'})-[:KNOWS*]-(b:Person {name: 'Bob'})
)
RETURN p, length(p) AS distance
```

### 10.6 Questions Pratiques DynamoDB

**Q : Expliquer la différence entre GSI et LSI dans DynamoDB.**

> Un **LSI** (Local Secondary Index) utilise la même Partition Key que la table mais un Sort Key différent. Il est local à une partition et créé obligatoirement à la création de la table. Un **GSI** (Global Secondary Index) peut avoir une Partition Key ET un Sort Key entièrement différents de la table. Il est global (span toutes les partitions), peut être créé/supprimé à tout moment, et possède sa propre capacité indépendante.

### 10.7 Questions de Synthèse

**Q : Vous devez concevoir une base de données pour un réseau social. Comparez MongoDB, Cassandra et Neo4j pour ce cas d'usage.**

| Critère | MongoDB | Cassandra | Neo4j |
|---------|---------|-----------|-------|
| Profils utilisateurs | ✅ Flexible, imbriqué | ✅ Possible | ✅ Nœuds |
| Feed d'actualités | ✅ Array de posts | ✅ Optimisé séries temporelles | Moyen |
| Graphe social (amis, suggestions) | ❌ Jointures complexes | ❌ Pas de jointures | ✅ Natif, performant |
| Messages directs | ✅ | ✅ Haute performance | Moyen |
| Recommandations | Moyen | ❌ | ✅ Algorithmes de graphe |

> **Conclusion :** Une architecture polyglotte combinant Cassandra (flux, messages à forte écriture) + Neo4j (graphe social, recommandations) + Elasticsearch (recherche de contenus) est souvent la meilleure approche pour un réseau social à grande échelle.

---

## Annexe — Aide-Mémoire Rapide

### Commandes Essentielles MongoDB
```javascript
show dbs; use ma_db; show collections
db.coll.find({}).pretty()
db.coll.insertOne({}) ; db.coll.insertMany([])
db.coll.updateOne({filtre}, {$set:{...}})
db.coll.deleteOne({filtre})
db.coll.createIndex({champ: 1})
db.coll.aggregate([{$match:{}}, {$group:{_id:"$champ"}}])
```

### Commandes Essentielles CQL (Cassandra)
```sql
CREATE KEYSPACE ... WITH replication = {...};
USE keyspace;
CREATE TABLE ... (... PRIMARY KEY ((pk), ck));
INSERT INTO ... VALUES (...);
SELECT * FROM ... WHERE pk = ...;
UPDATE ... SET ... WHERE pk = ... AND ck = ...;
DELETE FROM ... WHERE pk = ... AND ck = ...;
```

### API Essentielles Elasticsearch
```bash
PUT  /index                          # Créer index
PUT  /index/_doc/id                  # Indexer document
GET  /index/_doc/id                  # Lire document
POST /index/_update/id               # Mettre à jour
DELETE /index/_doc/id                # Supprimer
POST /index/_search                  # Rechercher
GET  /index/_mapping                 # Voir le mapping
```

### Clauses Essentielles Cypher (Neo4j)
```cypher
MATCH (n:Label {prop:'val'}) RETURN n
MATCH (a)-[r:TYPE]->(b) RETURN a,r,b
CREATE (n:Label {prop:'val'})
MERGE (n:Label {prop:'val'})
SET n.prop = 'newval'
DELETE n ; DETACH DELETE n
MATCH p=shortestPath((a)-[*]-(b)) RETURN p
```

### API Essentielles CouchDB
```bash
PUT  /db                             # Créer base
PUT  /db/doc_id                      # Créer document
GET  /db/doc_id                      # Lire document
PUT  /db/doc_id (avec _rev)          # Mettre à jour
DELETE /db/doc_id?rev=...            # Supprimer
POST /_replicate                     # Répliquer
GET  /db/_design/views/_view/name    # Requête vue
POST /db/_find                       # Mango query
```

### Opérations DynamoDB (Python boto3)
```python
table.put_item(Item={...})
table.get_item(Key={'PK': ..., 'SK': ...})
table.update_item(Key=..., UpdateExpression='SET ...')
table.delete_item(Key={...})
table.query(KeyConditionExpression=Key('PK').eq(...))
table.scan(FilterExpression=Attr('champ').eq(...))
```

---

*© Guide de préparation examen — Licence IA & Big Data, École Polytechnique de Lomé*  
*Contenu compilé à partir de la documentation officielle de MongoDB, Apache Cassandra, Elastic, Neo4j, Apache CouchDB, et Amazon Web Services.*
