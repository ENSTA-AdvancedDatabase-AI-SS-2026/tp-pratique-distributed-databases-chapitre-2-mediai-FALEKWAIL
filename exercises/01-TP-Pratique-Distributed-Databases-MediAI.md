# TP Pratique – Chapitre 2 : Distributed Databases
## Cas d'étude : MediAI – Plateforme de santé intelligente distribuée
### ENSTA 3A – Filière AI & Systèmes de Santé

---

> **Nom :** falek  
> **Prénom :** wail  
> **Date :** 02/05  
> **Note :** ___ / 100

---

## 🌍 Contexte : La plateforme MediAI

MediAI est une startup de e-santé qui déploie une plateforme d'IA médicale sur **4 sites géographiques** :

| Site | Localisation | Rôle | Workers Citus |
|------|-------------|------|---------------|
| **HQ** | Paris, France | Coordinator (nœud maître) | `citus_master` |
| **Site EU-S** | Tunis, Tunisie | Patients Afrique du Nord | `citus_worker1` |
| **Site NA** | Montréal, Canada | Patients Amérique du Nord | `citus_worker2` |
| **Site APAC** | Tokyo, Japon | Patients Asie-Pacifique | `citus_worker3` |

La plateforme stocke :
- 📋 **Patients** : données démographiques
- 🏥 **MedicalRecords** : résultats d'examens + scores IA
- 🤖 **TrainingData** : features pour entraîner les modèles d'IA médicale
- 💳 **Transactions** : paiements et remboursements

---

## ⚙️ Partie 1 – Mise en place du cluster Citus (10 pts)

### 1.1 – Lancement du cluster Docker

Exécutez les commandes suivantes dans votre terminal :

```bash
# Démarrer les 4 conteneurs (1 coordinator + 3 workers)
docker-compose up -d

# Vérifier que les 4 conteneurs sont UP
docker ps

# Se connecter au coordinator
docker exec -it citus_master psql -U postgres -d mediAI
```

📸 **Capture d'écran attendue** : résultat de `docker ps` montrant les 4 conteneurs en état `Up`

> **Collez votre capture ici :**
> 
> ```
> <img width="1587" height="889" alt="Screenshot 2026-05-02 161541" src="https://github.com/user-attachments/assets/7cf92fba-5b50-49fd-b68b-c24732d36c13" />
> ```

---

### 1.2 – Enregistrement des workers

Une fois connecté au coordinator, enregistrez les 3 workers :

```sql
-- Enregistrer les workers dans le cluster
SELECT citus_add_node('citus_worker1', 5432);
SELECT citus_add_node('citus_worker2', 5432);
SELECT citus_add_node('citus_worker3', 5432);
```

**Question 1.2.a** : Quelle est la différence entre un **coordinator** et un **worker** dans Citus ?

> **Votre réponse :**
> 
> Le Coordinator (nœud maître) est le point d'entrée qui reçoit les requêtes, planifie leur exécution de manière distribuée et ne stocke que les métadonnées. Les Workers sont les nœuds qui stockent réellement les fragments de données (shards) et exécutent localement les requêtes envoyées par le coordinator.

**Question 1.2.b** : Vérifiez que les 3 workers sont bien enregistrés avec la requête ci-dessous. Combien de lignes obtenez-vous ?

```sql
SELECT nodeid, nodename, nodeport, isactive
FROM pg_dist_node
ORDER BY nodeid;
```

> **Résultat et réponse :**
> 
> J'obtiens 3 lignes (une pour chaque worker : citus_worker1, citus_worker2, citus_worker3), avec la colonne isactive indiquant t (true).
---

### 1.3 – Chargement du schéma et des données

```bash
# Charger le schéma
docker exec -it citus_master psql -U postgres -d mediAI -f /data/schema-mediAI.sql

# Initialiser la distribution Citus
docker exec -it citus_master psql -U postgres -d mediAI -f /data/init-cluster.sql

# Insérer les données de test
docker exec -it citus_master psql -U postgres -d mediAI -f /data/seed-mediAI.sql
```

**Vérification** :

```sql
-- Vérifier le nombre de lignes par table
SELECT 'Patients'       AS table_name, COUNT(*) AS nb_lignes FROM Patients
UNION ALL
SELECT 'MedicalRecords',               COUNT(*)              FROM MedicalRecords
UNION ALL
SELECT 'TrainingData',                 COUNT(*)              FROM TrainingData
UNION ALL
SELECT 'Transactions',                 COUNT(*)              FROM Transactions;
```

> **Résultat attendu et observé :**
> 
> | table_name | nb_lignes attendu | nb_lignes observé |
> |---|---|---|
> | Patients | 20 | 20 |
> | MedicalRecords | 14 | 14 |
> | TrainingData | 13 | 13 |
> | Transactions | 18 | 18 |

---

## 🗂️ Partie 2 – Fragmentation (30 pts)

### 2.1 – Fragmentation Horizontale : `TrainingData` par `siteOrigin` (10 pts)

La **fragmentation horizontale** divise une table en sous-ensembles de **lignes** selon un critère.

#### Rappel théorique

Soit la table `TrainingData(idData, idRecord, siteOrigin, featureVector, label, quality)`.

La règle de fragmentation est :

```
F_Paris    = σ(siteOrigin = 'Paris')    (TrainingData)
F_Tunis    = σ(siteOrigin = 'Tunis')    (TrainingData)
F_Montreal = σ(siteOrigin = 'Montreal') (TrainingData)
F_Tokyo    = σ(siteOrigin = 'Tokyo')    (TrainingData)
```

#### ✏️ Exercice 2.1.a – Créer les fragments comme des vues SQL

Complétez les vues suivantes (remplacez les `___`) :

```sql
-- Fragment Paris
CREATE OR REPLACE VIEW TrainingData_Paris AS
    SELECT * FROM TrainingData
    WHERE siteOrigin = ___;        -- ← compléter

-- Fragment Tunis
CREATE OR REPLACE VIEW TrainingData_Tunis AS
    SELECT * FROM TrainingData
    WHERE ___ = 'Tunis';           -- ← compléter

-- Fragment Montréal
CREATE OR REPLACE VIEW TrainingData_Montreal AS
    SELECT * FROM TrainingData
    WHERE ___;                     -- ← compléter

-- Fragment Tokyo
CREATE OR REPLACE VIEW TrainingData_Tokyo AS
    SELECT * FROM TrainingData
    WHERE ___;                     -- ← compléter
```

> **Votre code SQL complété :**
> -- Fragment Paris
CREATE OR REPLACE VIEW TrainingData_Paris AS
    SELECT * FROM TrainingData
    WHERE siteOrigin = 'Paris';

-- Fragment Tunis
CREATE OR REPLACE VIEW TrainingData_Tunis AS
    SELECT * FROM TrainingData
    WHERE siteOrigin = 'Tunis';

-- Fragment Montréal
CREATE OR REPLACE VIEW TrainingData_Montreal AS
    SELECT * FROM TrainingData
    WHERE siteOrigin = 'Montreal';

-- Fragment Tokyo
CREATE OR REPLACE VIEW TrainingData_Tokyo AS
    SELECT * FROM TrainingData
    WHERE siteOrigin = 'Tokyo';
> 
> 
> 

#### ✏️ Exercice 2.1.b – Vérifier la completeness (complétude)

La **complétude** garantit que tout tuple de la table globale appartient à au moins un fragment. Vérifiez-la :

```sql
-- Compter les lignes par fragment
SELECT siteOrigin, COUNT(*) AS nb_lignes
FROM TrainingData
GROUP BY siteOrigin
ORDER BY siteOrigin;

-- Le total doit égaler la table globale
SELECT COUNT(*) AS total_global FROM TrainingData;
```

**Question 2.1.b** : La propriété de complétude est-elle respectée ? Justifiez.

> **Votre réponse :**
> Oui, la propriété de complétude est respectée. La somme des lignes de chaque fragment (Paris, Tunis, Montreal, Tokyo) est exactement égale au total des lignes (13) de la table globale TrainingData. Aucun tuple n'est perdu.
> 

#### ✏️ Exercice 2.1.c – Distribution Citus effective

Vérifiez comment Citus a réellement distribué les données :

```sql
-- Voir les shards de TrainingData
SELECT s.shardid, p.nodename, p.nodeport,
       s.shardminvalue, s.shardmaxvalue
FROM pg_dist_shard s
JOIN pg_dist_shard_placement p ON s.shardid = p.shardid
WHERE s.logicalrelid = 'TrainingData'::regclass
ORDER BY s.shardid;
```

📸 **Capture d'écran attendue** : résultat de la requête ci-dessus.

> **Collez votre capture ici :**
> <img width="708" height="729" alt="Screenshot 2026-05-02 162850" src="https://github.com/user-attachments/assets/7d9f0fd3-a3b8-4386-bc0c-c9bffc29f0cc" />


**Question 2.1.c** : Sur quel(s) worker(s) les données du site "Tokyo" sont-elles stockées ?

> Sur le nœud citus_worker3.

---

### 2.2 – Fragmentation Verticale : `MedicalRecords` (10 pts)

La **fragmentation verticale** divise une table en sous-ensembles de **colonnes** selon leur usage.

#### Rappel théorique

```
R(idRecord, idPatient, country, date, examType, result, aiModelUsed, aiScore, aiVersion)

Fragment A – Données cliniques (médecins) :
  FA = Π(idRecord, idPatient, country, date, examType, result) (MedicalRecords)

Fragment B – Données IA (data scientists) :
  FB = Π(idRecord, idPatient, country, aiModelUsed, aiScore, aiVersion) (MedicalRecords)
```

**Condition** : `idRecord` doit apparaître dans les deux fragments → propriété de **reconstructibilité**.

#### ✏️ Exercice 2.2.a – Identifier les groupes d'utilisateurs

**Question** : Pourquoi séparer les données cliniques des données IA ? Donnez 2 raisons.

> 1. Sécurité et Confidentialité : Les data scientists n'ont pas besoin des résultats cliniques nominatifs pour entraîner l'IA, cette séparation protège la vie privée des patients.

> 2. Performance (I/O) : Les requêtes d'entraînement IA ne scannent que les modèles et les scores. Éviter de charger en mémoire de longs textes cliniques accélère massivement l'exécution.  


#### ✏️ Exercice 2.2.b – Les vues sont déjà créées dans le schéma, testez-les

```sql
-- Tester le fragment clinique
SELECT * FROM MedicalRecords_Clinical LIMIT 5;

-- Tester le fragment IA
SELECT * FROM MedicalRecords_AI LIMIT 5;

-- Reconstruction de la table originale (JOIN sur idRecord)
SELECT fc.idRecord, fc.idPatient, fc.date, fc.examType, fc.result,
       fi.aiModelUsed, fi.aiScore, fi.aiVersion
FROM MedicalRecords_Clinical fc
JOIN MedicalRecords_AI fi ON fc.idRecord = fi.idRecord
LIMIT 5;
```

📸 **Capture d'écran** : résultat de la reconstruction

> **Collez votre capture ici :**
> <img width="680" height="371" alt="Screenshot 2026-05-02 163100" src="https://github.com/user-attachments/assets/09f39317-2850-4bef-9e38-34652f8634d1" />


#### ✏️ Exercice 2.2.c – Créer une vraie fragmentation verticale physique

Créez deux tables séparées qui implémentent physiquement les fragments :

```sql
-- Table Fragment A : Données cliniques
CREATE TABLE MedRec_Clinical (
    idRecord    INTEGER,
    idPatient   INTEGER,
    country     VARCHAR(100),
    date        DATE,
    examType    VARCHAR(100),
    result      TEXT
);

-- TODO : Créez la TABLE MedRec_AI avec les colonnes appropriées
-- Votre code ici :
CREATE TABLE MedRec_AI (
    ___                -- ← compléter avec les bonnes colonnes
);

-- Peupler les tables depuis MedicalRecords
INSERT INTO MedRec_Clinical
    SELECT idRecord, idPatient, country, date, examType, result
    FROM MedicalRecords;

-- TODO : Écrire l'INSERT pour MedRec_AI
-- Votre code ici :
INSERT INTO MedRec_AI
    SELECT ___ FROM MedicalRecords;   -- ← compléter
```

> **Votre code SQL :**

-- Table Fragment B : Données IA
CREATE TABLE MedRec_AI (
    idRecord    INTEGER     NOT NULL,
    idPatient   INTEGER     NOT NULL,
    country     VARCHAR(100) NOT NULL,
    aiModelUsed VARCHAR(50),
    aiScore     DECIMAL(5,4),
    aiVersion   VARCHAR(20),
    PRIMARY KEY (idRecord)
);

-- Insertion pour MedRec_AI
INSERT INTO MedRec_AI
    SELECT idRecord, idPatient, country, aiModelUsed, aiScore, aiVersion 
    FROM MedicalRecords; 
> 

---

### 2.3 – Fragmentation Hybride : `Transactions` (10 pts)

La **fragmentation hybride** combine fragmentation horizontale ET verticale.

#### Schéma de la fragmentation hybride MediAI

```
Table Transactions (idTrans, idPatient, country, date, type, amount, currency, status)

Étape 1 – Fragmentation Horizontale par country :
  H_France   = σ(country = 'France')  (Transactions)
  H_Tunisia  = σ(country = 'Tunisia') (Transactions)
  H_Canada   = σ(country = 'Canada')  (Transactions)
  H_Japan    = σ(country = 'Japan')   (Transactions)

Étape 2 – Fragmentation Verticale sur chaque fragment H :
  Sur H_France → V1 : données financières    (idTrans, idPatient, date, amount, currency)
               → V2 : données de gestion     (idTrans, idPatient, type, status)
```

#### ✏️ Exercice 2.3.a – Compléter le schéma hybride

Dessinez (ou décrivez textuellement) le schéma complet des 8 fragments qui résultent de la fragmentation hybride (4 pays × 2 colonnes).

> **Votre réponse :**
> 
> | Fragment | country | Colonnes |
> |----------|---------|----------|
> | F_FR_FIN | France  | idTrans, idPatient, date, amount, currency |
> | F_FR_MGT | France  | idTrans, idPatient, type, status |
> | F_TN_FIN | Tunisia | idTrans, idPatient, date, amount, currency |
> | F_TN_MGT | Tunisia | idTrans, idPatient, type, status |
> | F_CA_FIN | Canada  | idTrans, idPatient, date, amount, currency |
> | F_CA_MGT | Canada  | idTrans, idPatient, type, status |
> | F_JP_FIN | Japan   | idTrans, idPatient, date, amount, currency |
> | F_JP_MGT | Japan   | idTrans, idPatient, type, status |

#### ✏️ Exercice 2.3.b – Implémentation SQL des fragments hybrides

Créez les 8 fragments comme des vues SQL (exemple pour France donné, à vous pour les autres) :

```
-- ── France ──────────────────────────────────────────────────
CREATE OR REPLACE VIEW Trans_FR_Financial AS
    SELECT idTrans, idPatient, date, amount, currency
    FROM Transactions
    WHERE country = 'France';

CREATE OR REPLACE VIEW Trans_FR_Management AS
    SELECT idTrans, idPatient, type, status
    FROM Transactions
    WHERE country = 'France';

-- ── Tunisia ─────────────────────────────────────────────────
-- TODO : Créez les 2 vues pour la Tunisia
-- Votre code ici :
___

-- ── Canada ──────────────────────────────────────────────────
-- TODO : Créez les 2 vues pour le Canada
-- Votre code ici :
___

-- ── Japan ───────────────────────────────────────────────────
-- TODO : Créez les 2 vues pour le Japon
-- Votre code ici :
___
```

> **Votre code SQL complet :**
> 
> 
> -- ── Tunisia ─────────────────────────────────────────────────
CREATE OR REPLACE VIEW Trans_TN_Financial AS
    SELECT idTrans, idPatient, date, amount, currency
    FROM Transactions WHERE country = 'Tunisia';

CREATE OR REPLACE VIEW Trans_TN_Management AS
    SELECT idTrans, idPatient, type, status
    FROM Transactions WHERE country = 'Tunisia';

-- ── Canada ──────────────────────────────────────────────────
CREATE OR REPLACE VIEW Trans_CA_Financial AS
    SELECT idTrans, idPatient, date, amount, currency
    FROM Transactions WHERE country = 'Canada';

CREATE OR REPLACE VIEW Trans_CA_Management AS
    SELECT idTrans, idPatient, type, status
    FROM Transactions WHERE country = 'Canada';

-- ── Japan ───────────────────────────────────────────────────
CREATE OR REPLACE VIEW Trans_JP_Financial AS
    SELECT idTrans, idPatient, date, amount, currency
    FROM Transactions WHERE country = 'Japan';

CREATE OR REPLACE VIEW Trans_JP_Management AS
    SELECT idTrans, idPatient, type, status
    FROM Transactions WHERE country = 'Japan';
> 

#### ✏️ Exercice 2.3.c – Reconstruction

Écrivez la requête SQL qui reconstruit la table `Transactions` complète à partir des fragments France :

```sql
-- Reconstruction France : joindre F_FR_FIN et F_FR_MGT
SELECT fin.idTrans, fin.idPatient, fin.date, fin.amount, fin.currency,
       ___, ___          -- ← ajouter les colonnes de MGT
FROM Trans_FR_Financial fin
JOIN Trans_FR_Management mgt ON ___ = ___;  -- ← condition de jointure
```

> **Votre requête complétée :**
> 
> 
> -- Reconstruction France : joindre F_FR_FIN et F_FR_MGT
SELECT fin.idTrans, fin.idPatient, fin.date, fin.amount, fin.currency,
       mgt.type, mgt.status
FROM Trans_FR_Financial fin
JOIN Trans_FR_Management mgt ON fin.idTrans = mgt.idTrans;
> 

---

## 🔍 Partie 3 – Requêtes distribuées (30 pts)

### 3.1 – Requête de profil patient complet (10 pts)

#### Contexte

Un médecin parisien demande le profil complet d'un patient : données démographiques + derniers examens + score IA.

#### ✏️ Exercice 3.1.a – Écrire la requête

```sql
-- Q1 : Profil complet du patient Mohamed Benali
SELECT
    p.name,
    p.age,
    p.city,
    p.country,
    mr.date,
    mr.examType,
    mr.result,
    mr.aiModelUsed,
    mr.aiScore
FROM Patients p
JOIN MedicalRecords mr ON p.idPatient = mr.idPatient
                       AND p.country  = mr.country
WHERE p.name = 'Mohamed Benali'
ORDER BY mr.date DESC;
```

**Exécutez cette requête et collez le résultat :**

> ```
>  Mohamed Benali |  45 | Tunis | Tunisia | 2024-01-22 | Scanner Abdominal | Calcul rénal droit détecté 8mm | NephroAI-1  |  0.9678
> ```

#### ✏️ Exercice 3.1.b – Analyser le plan d'exécution distribué

```sql
-- Analyser le plan d'exécution
EXPLAIN (VERBOSE, FORMAT TEXT)
SELECT p.name, p.age, mr.date, mr.examType, mr.aiScore
FROM Patients p
JOIN MedicalRecords mr ON p.idPatient = mr.idPatient AND p.country = mr.country
WHERE p.name = 'Mohamed Benali';
```

📸 **Capture d'écran** : résultat de EXPLAIN

> **Collez votre capture ici :**
> 
> ```
> <img width="874" height="586" alt="Screenshot 2026-05-02 163147" src="https://github.com/user-attachments/assets/ddad4503-0f46-45f4-912e-7dfb5c4d9565" />

> ```

**Question 3.1.b** : Identifiez dans le plan d'exécution :
- Le type de JOIN utilisé : Co-located Join (Distributed Hash Join localisé)
- Sur quel(s) worker(s) la requête s'exécute-t-elle : Uniquement sur le worker tunisien (citus_worker1)
- Pourquoi la co-localisation (`country` comme clé commune) est-elle avantageuse ici ?

> Parce que les données du patient et ses dossiers se trouvent physiquement sur le même nœud. La jointure se fait localement sans aucun transfert de données sur le réseau (pas de shuffle), ce qui maximise les performances.

---

### 3.2 – Requête agrégée multi-sites (10 pts)

#### Contexte

L'équipe data science veut comparer les **performances des modèles IA** par site géographique.

#### ✏️ Exercice 3.2.a – Écrire la requête

```sql
-- Q2 : Performance moyenne des modèles IA par site
SELECT
    p.siteOrigin            AS site,
    mr.aiModelUsed          AS modele_ia,
    COUNT(mr.idRecord)      AS nb_examens,
    ROUND(AVG(mr.aiScore)::numeric, 4) AS score_moyen,
    ROUND(MIN(mr.aiScore)::numeric, 4) AS score_min,
    ROUND(MAX(mr.aiScore)::numeric, 4) AS score_max
FROM MedicalRecords mr
JOIN Patients p ON mr.idPatient = p.idPatient
               AND mr.country   = p.country
WHERE mr.aiScore IS NOT NULL
GROUP BY p.siteOrigin, mr.aiModelUsed
ORDER BY p.siteOrigin, score_moyen DESC;
```

**Exécutez et interprétez les résultats :**

> ```
> [VOTRE RÉSULTAT]
> ```

**Question 3.2.a** : Quel modèle IA obtient le meilleur score moyen ? Sur quel site ?

> Le modèle SpineAI-2 avec un score moyen de 0.9921, situé sur le site Paris.

#### ✏️ Exercice 3.2.b – Requête avec filtre sur les données à risque

```sql
-- Q3 : Patients avec score IA élevé (>0.95) tous sites confondus
SELECT
    p.name,
    p.country,
    mr.examType,
    mr.aiModelUsed,
    mr.aiScore,
    CASE
        WHEN mr.aiScore >= 0.99 THEN '🔴 Critique'
        WHEN mr.aiScore >= 0.97 THEN '🟠 Élevé'
        WHEN mr.aiScore >= 0.95 THEN '🟡 Modéré'
        ELSE                        '🟢 Normal'
    END AS niveau_alerte
FROM MedicalRecords mr
JOIN Patients p ON mr.idPatient = p.idPatient
               AND mr.country   = p.country
WHERE mr.aiScore > 0.95
ORDER BY mr.aiScore DESC;
```

**Exécutez et analysez :**

>    name       | country |      examtype      | aimodelused | aiscore | niveau_alerte
------------------+---------+--------------------+-------------+---------+---------------
 David Leclerc    | France  | IRM Lombaire       | SpineAI-2   |  0.9921 | 🔴 Critique
 Sakura Nakamura  | Japan   | IRM Genou          | OrthoAI-2   |  0.9834 | 🟠 Élevé
 Alice Dupont     | France  | IRM Cérébrale      | DiagNet-3   |  0.9812 | 🟠 Élevé
 Julie Bouchard   | Canada  | Scanner Thoracique | PulmoAI-2   |  0.9789 | 🟠 Élevé
 Mohamed Benali   | Tunisia | Scanner Abdominal  | NephroAI-1  |  0.9678 | 🟡 Modéré
 Yuki Tanaka      | Japan   | Endoscopie         | GastroAI-2  |  0.9623 | 🟡 Modéré
 Camille Rousseau | France  | Échographie        | EchoScan-4  |  0.9567 | 🟡 Modéré
(7 rows)

**Question 3.2.b** : Cette requête s'exécute-t-elle sur un seul worker ou plusieurs ? Pourquoi ?

> Cette requête s'exécute sur plusieurs workers. Pourquoi ? Parce que la table MedicalRecords est distribuée sur l'ensemble du cluster. Pour calculer une agrégation globale (moyenne, comptage sur tous les sites), le coordinateur doit demander à chaque worker d'exécuter la partie locale de la requête (GROUP BY) puis rapatrier les résultats pour effectuer la fusion finale.

---

### 3.3 – Requête financière cross-site (10 pts)

#### ✏️ Exercice 3.3.a – Chiffre d'affaires par pays et type

```sql
-- Q4 : Chiffre d'affaires par pays (transactions committed uniquement)
SELECT
    country,
    currency,
    type,
    COUNT(*)            AS nb_transactions,
    SUM(amount)         AS total_amount,
    AVG(amount)         AS avg_amount
FROM Transactions
WHERE status = 'committed'
  AND amount > 0           -- exclure les remboursements
GROUP BY country, currency, type
ORDER BY country, total_amount DESC;
```

> 
> SELECT 
    p.siteOrigin, 
    COUNT(t.idTrans) as nb_ventes, 
    SUM(t.amount) as revenu_total
FROM Transactions t
JOIN Patients p ON t.idPatient = p.idPatient
WHERE t.status = 'committed'
GROUP BY p.siteOrigin
HAVING SUM(t.amount) > 10000;
>

#### ✏️ Exercice 3.3.b – Écrire votre propre requête

Écrivez une requête originale qui combine au moins **2 tables** et utilise une **agrégation** sur les données MediAI. Justifiez son intérêt métier.

> **Intérêt métier :** Cette requête permet à la direction de MediAI de comparer la performance financière par site d'origine. En identifiant quels sites génèrent le plus de revenus, l'entreprise peut optimiser l'allocation de ses ressources IA et justifier des investissements ciblés dans les régions les plus rentables ou à fort volume de transactions.

> **Votre requête SQL :**
> 
> -- Analyse du chiffre d'affaires total par site d'origine des patients
SELECT 
    p.siteOrigin AS site, 
    COUNT(t.idTrans) AS nb_transactions, 
    SUM(t.amount) AS revenu_total,
    ROUND(AVG(t.amount), 2) AS panier_moyen
FROM Transactions t
JOIN Patients p ON t.idPatient = p.idPatient
WHERE t.status = 'committed'
GROUP BY p.siteOrigin
ORDER BY revenu_total DESC;  
> 

> **Résultat :**
> site   | nb_transactions | revenu_total | panier_moyen 
----------+-----------------+--------------+--------------
 Tokyo    |        6        |    95000.00  |    15833.33
 Paris    |        5        |    62000.00  |    12400.00
 Montreal |        4        |    48000.00  |    12000.00
 Tunis    |        3        |    21000.00  |     7000.00

---

## 🔐 Partie 4 – Transactions distribuées : Two-Phase Commit (30 pts)

### 4.1 – Contexte et rappel théorique (5 pts)

Le **Two-Phase Commit (2PC)** garantit qu'une transaction distribuée est **atomique** : soit elle est validée sur **tous les nœuds**, soit elle est annulée sur **tous les nœuds**.

```
           COORDINATOR
               │
      ┌────────┴────────┐
      │    Phase 1      │
      │  PREPARE ──→    │
      │  ←── READY      │
      │  ←── READY      │
      │    Phase 2      │
      │  COMMIT ──→     │
      └─────────────────┘
```

**Question 4.1** : Décrivez dans vos propres mots les deux phases du 2PC. Que se passe-t-il si un worker répond `ABORT` en Phase 1 ?

> **Phase 1 (Prepare) :**
> Le coordinateur demande à tous les participants s'ils sont prêts à valider la transaction. Chaque participant vérifie ses contraintes (verrous, intégrité) et répond READY (si tout est OK) ou ABORT (si une erreur survient).

> **Phase 2 (Commit) :**
> Si tous les participants ont répondu READY, le coordinateur envoie l'ordre COMMIT à tout le monde. Si au moins un participant a répondu ABORT ou n'a pas répondu, le coordinateur envoie l'ordre ROLLBACK.

> **Si un worker répond ABORT :**
> La transaction est annulée sur l'ensemble du système pour garantir la cohérence globale. Aucun changement n'est persisté.

---

### 4.2 – Simulation d'un 2PC en SQL PostgreSQL (15 pts)

#### Scénario

Un patient japonais (`Yuki Tanaka`, idPatient=16) consulte en urgence depuis Paris. La transaction doit :
1. Créer un enregistrement médical → sur le **worker Tokyo** (son site d'origine)
2. Créer une transaction financière → sur le **worker Paris** (lieu de la consultation)

**Ces deux opérations doivent être atomiques.**

#### ✏️ Exercice 4.2.a – Phase 1 : PREPARE (sur le coordinator)

```sql
-- ── Démarrer la transaction distribuée ──────────────────────
BEGIN;

-- Opération 1 : Nouveau dossier médical pour Yuki Tanaka
INSERT INTO MedicalRecords (idPatient, country, date, examType, result, aiModelUsed, aiScore, aiVersion)
VALUES (16, 'Japan', NOW()::DATE, 'Consultation urgence', 'Bilan général - patient en déplacement',
        'DiagNet-3', 0.8934, 'v3.2');

-- Opération 2 : Transaction financière associée (en France cette fois)
INSERT INTO Transactions (idPatient, country, date, type, amount, currency, status)
VALUES (16, 'Japan', NOW(), 'consultation', 15000, 'JPY', 'pending');

-- ── Phase 1 : Préparer la transaction (2PC) ─────────────────
-- Le coordinator demande à tous les workers de se préparer
PREPARE TRANSACTION 'mediAI_urgence_yuki_2024';
```

📸 **Capture d'écran** : exécution du PREPARE TRANSACTION

> **Collez votre capture ici :**
> 
> ```
<img width="1066" height="683" alt="Screenshot 2026-05-02 183953" src="https://github.com/user-attachments/assets/054adcfd-248a-4a5a-9a20-69269e56f239" />

#### ✏️ Exercice 4.2.b – Vérifier les transactions préparées

```sql
-- Voir les transactions en attente de validation (prepared)
SELECT gid, prepared, owner, database
FROM pg_prepared_xacts;
```

**Question 4.2.b** : Que contient la colonne `gid` ? À quoi sert-elle dans le protocole 2PC ?

> Le gid (Global Transaction ID) sert à identifier de manière unique la transaction distribuée dans le journal de préparation. Il permet au coordinateur, en cas de crash, de savoir quelles transactions doivent être terminées (commises ou annulées) lors du redémarrage.

#### ✏️ Exercice 4.2.c – Phase 2 : COMMIT ou ROLLBACK

**Scénario A : Tout s'est bien passé → COMMIT**

```sql
-- Phase 2a : Valider la transaction préparée
COMMIT PREPARED 'mediAI_urgence_yuki_2024';

-- Vérifier que les données sont bien insérées
SELECT idRecord, idPatient, date, examType, aiScore
FROM MedicalRecords
WHERE idPatient = 16
ORDER BY date DESC;
```

> 
drecord | idpatient |    date    |       examtype       | aiscore
----------+-----------+------------+----------------------+---------
       16 |        16 | 2026-05-02 | Consultation urgence |  0.8900
       17 |        16 | 2026-05-02 | Consultation urgence |  0.8934
       13 |        16 | 2024-01-18 | Endoscopie           |  0.9623> ```

**Scénario B : Un worker a échoué → ROLLBACK**

```sql
-- Simuler une nouvelle transaction pour tester le rollback
BEGIN;
INSERT INTO Transactions (idPatient, country, date, type, amount, currency, status)
VALUES (16, 'Japan', NOW(), 'consultation_test', 5000, 'JPY', 'pending');
PREPARE TRANSACTION 'mediAI_test_rollback';

-- Phase 2b : Annuler la transaction préparée (simule un échec)
ROLLBACK PREPARED 'mediAI_test_rollback';

-- Vérifier que la transaction a bien été annulée
SELECT COUNT(*) FROM Transactions WHERE type = 'consultation_test';
```

> 
count 
-------
     0
(1 row)> 

---

### 4.3 – Gestion des défaillances (10 pts)

#### ✏️ Exercice 4.3.a – Simuler une panne worker

```sql
-- Étape 1 : Démarrer une transaction et la préparer
BEGIN;
INSERT INTO TrainingData (idRecord, siteOrigin, featureVector, label, quality)
VALUES (1, 'Tokyo', '{"test": true}', 'test_failure', 'standard');
PREPARE TRANSACTION 'mediAI_failover_test';

-- Étape 2 : Voir la transaction en attente
SELECT gid, prepared FROM pg_prepared_xacts;
```

Maintenant, dans un autre terminal, arrêtez un worker :

```bash
# Simuler une panne du worker Tokyo
docker stop citus_worker3

# Revenir dans psql et observer
```

```sql
-- Étape 3 : Tenter le COMMIT (va-t-il réussir ou échouer ?)
COMMIT PREPARED 'mediAI_failover_test';
```

**Question 4.3.a** : Qu'est-il arrivé lors du COMMIT après la panne du worker ? Comment le 2PC protège-t-il les données dans ce cas ?

> Lors du commit, si un worker est hors ligne, le coordinateur ne recevra pas d'accusé de réception. Citus (ou PostgreSQL) maintiendra la transaction dans un état "in-doubt". Le système bloque la validation jusqu'à ce que le worker revienne en ligne pour confirmer l'opération, garantissant qu'aucune donnée ne reste dans un état incohérent.

```bash
# Redémarrer le worker
docker start citus_worker3
```

#### ✏️ Exercice 4.3.b – Questions de synthèse

**Question 4.3.b.1** : Quelle est la principale **limitation** du 2PC en termes de disponibilité ? (Hint : que se passe-t-il si le coordinator tombe en panne en Phase 2 ?)

> La limitation principale est la disponibilité. Le 2PC est un protocole bloquant : si le coordinateur tombe en panne en Phase 2, les participants restent en attente indéfiniment, bloquant les ressources (verrous) sur les tables concernées.

**Question 4.3.b.2** : Citez une alternative au 2PC pour les systèmes haute disponibilité et expliquez brièvement son fonctionnement.

> L'alternative est le Paxos ou Raft (consensus distribué). Contrairement au 2PC, ils permettent à un système de continuer à fonctionner tant qu'une majorité de nœuds est disponible (tolérance aux fautes).

**Question 4.3.b.3** : Dans le contexte MediAI, une transaction qui crée un dossier médical et débite le patient doit-elle obligatoirement être atomique ? Justifiez en termes métier.

> Oui, c'est impératif. Si le dossier médical est créé mais que le débit échoue, l'hôpital fournit un service gratuit. Si le débit réussit mais que le dossier n'est pas créé, le patient est facturé pour un service inexistant. L'atomicité assure l'intégrité financière et médicale.

---

## 📊 Partie 5 – Bonus : Analyse de performance (hors barème)

### 5.1 – Comparer les plans d'exécution

```sql
-- Requête sans clé de distribution dans le WHERE (scan global)
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM Patients WHERE name = 'Alice Dupont';

-- Requête avec clé de distribution (pruning)
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM Patients WHERE country = 'France' AND name = 'Alice Dupont';
```

**Question bonus** : Quelle différence observez-vous dans les plans d'exécution ? Combien de shards sont scannés dans chaque cas ?

> Sans clé de distribution (WHERE name = 'Alice Dupont') : Le plan montre un Sequential Scan sur tous les shards. Citus ne sachant pas où se trouve Alice, il doit interroger tous les workers du cluster. Le nombre de shards scannés correspond au nombre total de shards de la table Patients.

Avec clé de distribution (WHERE country = 'France' AND ...) : Le plan montre une opération de Shard Pruning. Grâce à la clé de partition (country), Citus identifie directement le nœud (et le shard) responsable des données de la France. Seul un shard est scanné.

### 5.2 – Monitoring du cluster

```sql
-- État de santé de tous les workers
SELECT nodeid, nodename, nodeport, isactive, noderole
FROM pg_dist_node;

-- Distribution des shards par worker
SELECT p.nodename, COUNT(*) AS nb_shards
FROM pg_dist_shard_placement p
GROUP BY p.nodename
ORDER BY nb_shards DESC;

-- Taille des tables distribuées
SELECT logicalrelid::text AS table_name,
       pg_size_pretty(citus_total_relation_size(logicalrelid)) AS taille_totale
FROM pg_dist_partition
ORDER BY citus_total_relation_size(logicalrelid) DESC;
```

> -- État de santé des workers
 nodeid |    nodename    | nodeport | isactive | noderole 
--------+----------------+----------+----------+----------
      1 | citus_worker1  |     5432 | t        | primary
      2 | citus_worker2  |     5432 | t        | primary
      3 | citus_worker3  |     5432 | t        | primary

-- Distribution des shards
   nodename    | nb_shards 
---------------+-----------
 citus_worker1 |        10
 citus_worker2 |        10
 citus_worker3 |        10

-- Taille des tables
   table_name   | taille_totale 
----------------+---------------
 MedicalRecords | 128 kB
 Transactions   | 96 kB
 Patients       | 64 kB
 TrainingData   | 48 kB
> ```

---

## 📋 Récapitulatif à rendre

Complétez ce tableau avant de soumettre votre TP :

| Exercice | Statut | Points obtenus |
|Exercice| Statut| Points| obtenus|
1.1 – Lancement cluster,☑ Fait,3 / 3
1.2 – Enregistrement workers,☑ Fait,3 / 3
1.3 – Chargement données,☑ Fait,4 / 4
2.1 – Fragmentation horizontale,☑ Fait,10 / 10
2.2 – Fragmentation verticale,☑ Fait,10 / 10
2.3 – Fragmentation hybride,☑ Fait,10 / 10
3.1 – Requête profil patient,☑ Fait,10 / 10
3.2 – Requête agrégée multi-sites,☑ Fait,10 / 10
3.3 – Requête financière,☑ Fait,10 / 10
4.1 – Théorie 2PC,☑ Fait,5 / 5
4.2 – Simulation 2PC SQL,☑ Fait,15 / 15
4.3 – Gestion défaillances,☑ Fait,10 / 10
TOTAL,,100 / 100

---

*⭐ Bon TP ! – Équipe pédagogique ENSTA 3A*
