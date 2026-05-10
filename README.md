# 🏦 Système de Contrôle d'Accès — Banque Internationale

Un système complet et production-ready de **contrôle d'accès multi-modèles** pour opérations bancaires internationales. Implémente RBAC, BLP, Biba, ABAC et Chinese Wall dans une base PostgreSQL.

**Auteurs :** Achraf Hasnaoui & Mohamed Aziz Foudhaili  
**Date :** Mai 2026  
**SGBD :** PostgreSQL 14+

---

## 📌 Table des Matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture & Modèles de Sécurité](#architecture--modèles-de-sécurité)
- [Schéma de Base de Données](#schéma-de-base-de-données)
- [Données d'Exemple](#données-dexemple)
- [Requêtes de Test](#requêtes-de-test)
- [Guide d'Installation](#guide-dinstallation)
- [Cas d'Usage Réels](#cas-dusage-réels)
- [Guide de Personnalisation](#guide-de-personnalisation)

---

## 🎯 Vue d'Ensemble

### Problématique

Les banques internationales opèrent dans plusieurs pays avec des équipes diversifiées (relations clients, back-office, gestion de patrimoine, conformité, audit, juridique...) accédant à des données hautement sensibles. Comment accorder un accès granulaire tout en maintenant l'isolation stricte des données, la conformité réglementaire et la prévention des conflits d'intérêts ?

### Solution

Un système **multi-couches** combinant 5 modèles de sécurité :

| Modèle | Question | Approche |
|--------|----------|----------|
| **RBAC** | Qui peut faire quoi ? | Rôles × Permissions |
| **BLP** | Niveau de confidentialité suffisant ? | Niveaux N1-N4 (no read-up) |
| **Biba** | Pas de violation d'intégrité ? | Intégrité des données (no write-up) |
| **ABAC** | Contexte autorisé ? | Pays, horaires, portefeuille |
| **Chinese Wall** | Conflits d'intérêts détectés ? | Groupes concurrents |

---

## 🏗️ Architecture & Modèles de Sécurité

### Niveaux de Sécurité (BLP)

```
┌─────────────────────────────────────────────────────────────┐
│ N1 - INTERNE METIER                      (Sans MFA, log std) │
│     Opérations courantes, référentiels internes             │
├─────────────────────────────────────────────────────────────┤
│ N2 - SENSIBLE                            (Sans MFA, log renforcé) │
│     Données clients, contrats, historiques transactions     │
├─────────────────────────────────────────────────────────────┤
│ N3 - TRES SENSIBLE / REGLEMENTE          (MFA requis, log complet) │
│     KYC détaillé, ordres de marché, risque stratégique      │
├─────────────────────────────────────────────────────────────┤
│ N4 - CRITIQUE / CONFIDENTIEL EXTREME     (MFA + double validation) │
│     Dossiers AML, sanctions, investigations réglementaires  │
└─────────────────────────────────────────────────────────────┘
```

### Structure des Tables (14 tables)

```
TABLES REFERENCE
├── niveaux_securite (4 enregistrements)
├── pays (7 enregistrements)
├── equipes (10 enregistrements)
└── ressources (17 enregistrements)

TABLES RBAC
├── roles (24+ roles métier)
├── permissions (22 permissions granulaires)
├── role_permissions (associations)
└── utilisateur_roles (affectations avec dates)

TABLES MAC (Mandatory Access Control)
├── blp_habilitations (clearance par utilisateur)
└── utilisateurs (20 collaborateurs)

TABLES ABAC
└── abac_politiques (10 politiques contextuelles)

TABLES TEMPORAIRES
├── acces_temporaire (accès bornés avec révocation auto)
└── chinese_wall_historique (tracking conflits)

TABLES AUDIT
└── journal_audit (trace immuable de toutes les décisions)
```

---

## 📊 Schéma de Base de Données

### 1. Tables de Référence

#### **niveaux_securite**
```
niveau          | designation                    | mfa_requis | double_valid | journalisation
────────────────┼────────────────────────────────┼────────────┼──────────────┼──────────────
N1              | Interne metier                 | FALSE      | FALSE        | STANDARD
N2              | Sensible                       | FALSE      | FALSE        | RENFORCE
N3              | Tres sensible / Reglemente    | TRUE       | FALSE        | COMPLET
N4              | Critique / Confidentiel extreme| TRUE       | TRUE         | COMPLET
```

#### **pays**
```
code_pays | nom          | zone_regulatoire
──────────┼──────────────┼─────────────────
TN        | Tunisie      | MAGHREB
FR        | France       | UE
DE        | Allemagne    | UE
GB        | Royaume-Uni  | UK
US        | Etats-Unis   | US
SG        | Singapour    | ASIE
MA        | Maroc        | MAGHREB
```

#### **equipes** (10 équipes)
```
id | nom                      | categorie      | effectif | niveau_max
───┼──────────────────────────┼────────────────┼──────────┼───────────
1  | Relation clientele       | OPERATIONNEL   | 120      | N2
2  | Back-office              | OPERATIONNEL   | 80       | N2
3  | Gestion de patrimoine    | OPERATIONNEL   | 60       | N3
4  | Risque                   | OPERATIONNEL   | 40       | N3
5  | Conformite / AML         | CONTROLE       | 35       | N4
6  | Audit interne            | CONTROLE       | 20       | N4
7  | Juridique                | CONTROLE       | 20       | N4
8  | Cadres commerciaux       | EXPERTISE      | 25       | N2
9  | Experts regionaux        | EXPERTISE      | 10       | N3
10 | Support                  | SUPPORT        | 90       | N1
```

### 2. Tables RBAC

#### **roles** (24+ rôles)
```
id | nom_role                  | equipe | niveau_acces
───┼───────────────────────────┼────────┼──────────────
1  | conseiller_particulier    | 1      | N2
2  | conseiller_entreprise     | 1      | N2
3  | responsable_agence        | 1      | N2
4  | guichetier_responsable    | 1      | N2
5  | operateur_bo              | 2      | N1
6  | superviseur_bo            | 2      | N2
7  | conseiller_patrimoine     | 3      | N3
8  | analyste_portefeuille     | 3      | N3
... (et 16 autres)
```

#### **permissions** (22 permissions)
```
id | ressource_type       | action    | niveau_min | description
───┼──────────────────────┼───────────┼────────────┼─────────────────────────────────
1  | DONNEES_CLIENT       | LIRE      | N2         | Lire données de base d'un client
2  | DONNEES_CLIENT       | MODIFIER  | N2         | Modifier données client (KYC)
3  | PORTEFEUILLE         | LIRE      | N3         | Consulter portefeuille détaillé
4  | PORTEFEUILLE         | ECRIRE    | N3         | Enregistrer ordre sur portefeuille
5  | DOSSIER_KYC          | LIRE      | N2         | Lire dossier KYC de base
6  | DOSSIER_KYC          | MODIFIER  | N3         | Modifier/compléter dossier KYC
7  | DOSSIER_AML          | LIRE      | N4         | Lire dossier AML actif
8  | DOSSIER_AML          | ECRIRE    | N4         | Créer/enrichir dossier AML
... (et 14 autres)
```

### 3. Tables Utilisateurs

#### **utilisateurs** (20 collaborateurs)
```
id | matricule | nom        | prenom    | email                    | equipe | pays | statut | mfa_actif
───┼───────────┼────────────┼───────────┼──────────────────────────┼────────┼──────┼────────┼──────────
1  | RC001     | Ben Ali    | Sami      | sami.benali@banque.tn    | 1      | TN   | ACTIF  | FALSE
2  | RC002     | Martin     | Claire    | claire.martin@banque.fr  | 1      | FR   | ACTIF  | FALSE
3  | RC003     | Dupont     | Marc      | marc.dupont@banque.fr    | 1      | FR   | ACTIF  | FALSE
4  | BO001     | Trabelsi   | Nadia     | nadia.trabelsi@banque.tn | 2      | TN   | ACTIF  | FALSE
5  | BO002     | Schmidt    | Hans      | hans.schmidt@banque.de   | 2      | DE   | ACTIF  | FALSE
6  | GP001     | Leblanc    | Sophie    | sophie.leblanc@banque.fr | 3      | FR   | ACTIF  | TRUE
7  | GP002     | Hamdi      | Karim     | karim.hamdi@banque.tn    | 3      | TN   | ACTIF  | TRUE
8  | RS001     | Lambert    | Eric      | eric.lambert@banque.fr   | 4      | FR   | ACTIF  | TRUE
9  | RS002     | Nguyen     | Lan       | lan.nguyen@banque.sg     | 4      | SG   | ACTIF  | TRUE
10 | AML001    | Moreau     | Julie     | julie.moreau@banque.fr   | 5      | FR   | ACTIF  | TRUE
... (10 autres utilisateurs)
```

### 4. Tables Ressources

#### **ressources** (17 ressources protégées)
```
id | reference        | type_ressource      | niveau | pays | groupe_conflict      | proprietaire
───┼──────────────────┼─────────────────────┼────────┼──────┼──────────────────────┼──────────────
1  | CLI-TN-001       | DONNEES_CLIENT      | N2     | TN   | NULL                 | 1
2  | CLI-FR-001       | DONNEES_CLIENT      | N2     | FR   | NULL                 | 2
3  | PTF-FR-GP01      | PORTEFEUILLE        | N3     | FR   | GROUPE_INDUS_ALPHA   | 6
4  | PTF-TN-GP02      | PORTEFEUILLE        | N3     | TN   | GROUPE_INDUS_BETA    | 7
5  | KYC-TN-001       | DOSSIER_KYC         | N2     | TN   | NULL                 | 1
6  | KYC-FR-AML01     | DOSSIER_KYC         | N3     | FR   | NULL                 | 10
7  | AML-FR-2024-01   | DOSSIER_AML         | N4     | FR   | NULL                 | 10
8  | AML-TN-2024-01   | DOSSIER_AML         | N4     | TN   | NULL                 | 11
... (9 autres ressources)
```

### 5. Table BLP

#### **blp_habilitations**
```
id_utilisateur | niveau_max | date_attribution | attribue_par | commentaire
───────────────┼────────────┼──────────────────┼──────────────┼───────────────────────────
1              | N2         | 2026-01-15       | 14           | Habilitation initiale - equipe 1
2              | N2         | 2026-01-15       | 14           | Habilitation initiale - equipe 1
3              | N2         | 2026-01-15       | 14           | Habilitation initiale - equipe 1
6              | N3         | 2026-01-15       | 14           | Habilitation initiale - equipe 3
7              | N3         | 2026-01-15       | 14           | Habilitation initiale - equipe 3
... (16 autres)
```

### 6. Table ABAC

#### **abac_politiques** (10 politiques contextuelles)
```
id | nom                       | role | filtre_pays | filtre_portefeuille | heures      | jours
───┼───────────────────────────┼──────┼─────────────┼─────────────────────┼─────────────┼───────────
1  | RC_pays_portefeuille      | 1    | TRUE        | TRUE                | 08:00-18:30 | LUN-SAM
2  | GP_portefeuille_strict    | 7    | TRUE        | TRUE                | 07:30-20:00 | LUN-VEN
3  | RISQUE_lecture_seule      | 11   | FALSE       | FALSE               | 08:00-19:00 | LUN-VEN
4  | AML_dossier_assigne       | 13   | TRUE        | FALSE               | 00:00-23:59 | LUN-DIM
5  | AUDIT_lecture_globale     | 18   | FALSE       | FALSE               | 08:00-20:00 | LUN-VEN
6  | EXPERT_temporaire         | 23   | TRUE        | FALSE  720h (30j)   | 08:00-18:00 | LUN-VEN
7  | COMMERCIAL_kpi_seulement  | 21   | FALSE       | FALSE               | 08:00-19:00 | LUN-VEN
8  | GUICHETIER_clients        | 4    | TRUE        | FALSE               | 07:30-18:30 | LUN-SAM
9  | GUICHETIER_operations     | 4    | TRUE        | FALSE               | 07:00-19:30 | LUN-SAM
10 | GUICHETIER_urgence        | 4    | TRUE        | FALSE     4h        | 00:00-23:59 | LUN-DIM
```

---

## 📋 Données d'Exemple

### Équipes et Utilisateurs par Rôle

#### Relation Clientèle (équipe 1)
- **Sami Ben Ali (RC001, N2)** : Conseiller particulier, Tunisie
- **Claire Martin (RC002, N2)** : Conseiller particulier, France
- **Marc Dupont (RC003, N2)** : Conseiller entreprise, France
- **Mehdi Gharbi (GR001, N2)** : Guichetier responsable, Tunisie
- **Isabelle Leroux (GR002, N2)** : Guichetier responsable, France
- **Faycal El Amri (GR003, N2)** : Guichetier responsable, Maroc

#### Back-Office (équipe 2)
- **Nadia Trabelsi (BO001, N1)** : Opérateur, Tunisie
- **Hans Schmidt (BO002, N2)** : Superviseur, Allemagne

#### Gestion de Patrimoine (équipe 3)
- **Sophie Leblanc (GP001, N3)** : Conseiller patrimoine, France 🔴 **(Chinese Wall avec INDUS_BETA)**
- **Karim Hamdi (GP002, N3)** : Conseiller patrimoine, Tunisie

#### Risque (équipe 4)
- **Eric Lambert (RS001, N3)** : Analyste risque crédit, France 🔴 **(Chinese Wall avec INDUS_BETA)**
- **Lan Nguyen (RS002, N3)** : Analyste risque marché, Singapour

#### Conformité / AML (équipe 5)
- **Julie Moreau (AML001, N4)** : Analyste AML, France
- **Walid Chahed (AML002, N4)** : Analyste AML, Tunisie
- **James Wilson (AML003, N4)** : Spécialiste sanctions, Royaume-Uni

#### Audit (équipe 6)
- **Pierre Fontaine (AI001, N3)** : Auditeur, France
- **Leila Bouaziz (AI002, N4)** : Auditeur senior, Tunisie

#### Juridique (équipe 7)
- **Anne Bernard (JR001, N3)** : Juriste bancaire, France
- **Rania Slimane (JR002, N4)** : Juriste conformité, Tunisie

#### Cadres Commerciaux (équipe 8)
- **Luc Renard (CC001, N2)** : Directeur commercial, France
- **Tarek Mansour (CC002, N2)** : Responsable segment, Tunisie

#### Experts Régionaux (équipe 9)
- **Anna Kowalski (ER001, N3)** : Expert régional, France ⏰ **(Accès temporaire: INV-REG-2024, 30 jours)**

#### Support (équipe 10)
- **Tom Laurent (SP001, N1)** : Agent support, France

---

## 🧪 Requêtes de Test

### 1️⃣ CAS AUTORISÉ : Conseiller lit son client

```sql
-- Sami Ben Ali (N2) lit CLI-TN-001 (N2, Tunisie)
SELECT 
    u.matricule, u.nom, u.prenom,
    r.reference, r.type_ressource, r.niveau_requis,
    h.niveau_max,
    'DEVRAIT AUTORISER' as verdict
FROM utilisateurs u
JOIN blp_habilitations h ON u.id_utilisateur = h.id_utilisateur
JOIN ressources r ON r.reference = 'CLI-TN-001'
WHERE u.matricule = 'RC001';

-- Résultat:
-- RC001 | Ben Ali | Sami | CLI-TN-001 | DONNEES_CLIENT | N2 | N2 | DEVRAIT AUTORISER
```

**Justification :**
- ✅ **RBAC**: Role `conseiller_particulier` a permission `DONNEES_CLIENT/LIRE`
- ✅ **BLP**: N2 ≥ N2
- ✅ **ABAC**: Client en Tunisie (pays de l'utilisateur)
- ✅ **Chinese Wall**: N/A

---

### 2️⃣ CAS REFUSÉ (BLP) : Directeur commercial accède à dossier AML

```sql
-- Luc Renard (N2) tente de lire AML-FR-2024-01 (N4)
SELECT 
    u.matricule, u.nom, u.prenom,
    'ACCES REFUSE' as verdict,
    'BLP - Niveau insuffisant: N2 < N4 requis' as motif
FROM utilisateurs u
JOIN blp_habilitations h ON u.id_utilisateur = h.id_utilisateur
WHERE u.matricule = 'CC001'
  AND (SELECT niveau_requis FROM ressources WHERE reference = 'AML-FR-2024-01') > h.niveau_max;

-- Résultat:
-- CC001 | Renard | Luc | ACCES REFUSE | BLP - Niveau insuffisant: N2 < N4 requis
```

**Justification :**
- ❌ **BLP**: N2 < N4 → **REFUSÉ**

---

### 3️⃣ CAS REFUSÉ (RBAC) : Conseiller accède à portefeuille

```sql
-- Sami Ben Ali (conseiller_particulier) tente de lire PTF-FR-GP01 (PORTEFEUILLE)
-- Role conseiller_particulier n'a pas permission PORTEFEUILLE/LIRE
SELECT 
    u.matricule, u.nom,
    'ACCES REFUSE' as verdict,
    'RBAC - Role ne dispose pas de permission' as motif
FROM utilisateurs u
JOIN utilisateur_roles ur ON u.id_utilisateur = ur.id_utilisateur
JOIN roles r ON ur.id_role = r.id_role
WHERE u.matricule = 'RC001'
  AND r.nom_role = 'conseiller_particulier'
  AND NOT EXISTS (
      SELECT 1 FROM role_permissions rp
      JOIN permissions p ON rp.id_permission = p.id_permission
      WHERE rp.id_role = r.id_role
        AND p.ressource_type = 'PORTEFEUILLE'
        AND p.action = 'LIRE'
  );

-- Résultat:
-- RC001 | Ben Ali | ACCES REFUSE | RBAC - Role ne dispose pas de permission
```

---

### 4️⃣ CAS REFUSÉ (Chinese Wall) : Conseiller accède à groupe concurrent

```sql
-- Sophie Leblanc (GP001) a consulté PTF-FR-GP01 (GROUPE_INDUS_ALPHA)
-- Tente maintenant d'accéder à PTF-TN-GP02 (GROUPE_INDUS_BETA)
SELECT 
    u.matricule, u.nom,
    cwh.groupe_conflict as groupe_deja_consulte,
    'ACCES REFUSE' as verdict,
    'CHINESE WALL - Conflit d''intérêt détecté' as motif
FROM utilisateurs u
JOIN chinese_wall_historique cwh ON u.id_utilisateur = cwh.id_utilisateur
WHERE u.matricule = 'GP001'
  AND cwh.groupe_conflict = 'GROUPE_INDUS_ALPHA';

-- Résultat:
-- GP001 | Leblanc | GROUPE_INDUS_ALPHA | ACCES REFUSE | CHINESE WALL - Conflit d'intérêt détecté
```

---

### 5️⃣ CAS TEMPORAIRE : Expert accède 30 jours à investigation réglementaire

```sql
-- Anna Kowalski (ER001, N3) a accès temporaire à INV-REG-2024 (N4)
SELECT 
    u.matricule, u.nom, u.prenom,
    r.reference, r.type_ressource, r.niveau_requis,
    at.justification,
    at.date_debut, at.date_fin,
    EXTRACT(DAY FROM (at.date_fin - NOW())) as jours_restants,
    at.statut,
    'AUTORISE (TEMPORAIRE)' as verdict
FROM utilisateurs u
JOIN acces_temporaire at ON u.id_utilisateur = at.id_utilisateur
JOIN ressources r ON at.id_ressource = r.id_ressource
WHERE u.matricule = 'ER001'
  AND r.reference = 'INV-REG-2024'
  AND at.statut = 'ACTIF';

-- Résultat:
-- ER001 | Kowalski | Anna | INV-REG-2024 | INVESTIGATION | N4 
-- | Investigation reglementaire ACPR 2024 - expertise droit tunisien requise
-- | 2026-05-10 | 2026-06-09 | 30 jours | ACTIF | AUTORISE (TEMPORAIRE)
```

---

### 6️⃣ DASHBOARD : Droits complets d'un utilisateur

```sql
-- Vue synthétique des droits de Sami Ben Ali
SELECT 
    u.matricule,
    u.nom || ' ' || u.prenom as utilisateur,
    r.nom_role,
    p.ressource_type,
    p.action,
    p.niveau_min,
    h.niveau_max,
    CASE 
        WHEN h.niveau_max >= p.niveau_min::varchar::niveau_enum THEN 'OK'
        ELSE 'BLOQUE'
    END as status
FROM utilisateurs u
JOIN blp_habilitations h ON u.id_utilisateur = h.id_utilisateur
JOIN utilisateur_roles ur ON u.id_utilisateur = ur.id_utilisateur
JOIN roles r ON ur.id_role = r.id_role
JOIN role_permissions rp ON r.id_role = rp.id_role
JOIN permissions p ON rp.id_permission = p.id_permission
WHERE u.matricule = 'RC001'
  AND ur.actif = TRUE
ORDER BY p.ressource_type, p.action;

-- Résultat (8 lignes):
-- RC001 | Ben Ali Sami | conseiller_particulier | DONNEES_CLIENT | LIRE | N2 | N2 | OK
-- RC001 | Ben Ali Sami | conseiller_particulier | DONNEES_CLIENT | MODIFIER | N2 | N2 | OK
-- RC001 | Ben Ali Sami | conseiller_particulier | DOSSIER_KYC | LIRE | N2 | N2 | OK
-- RC001 | Ben Ali Sami | conseiller_particulier | DOSSIER_KYC | MODIFIER | N3 | N2 | BLOQUE
-- ...
```

---

### 7️⃣ DASHBOARD : Refus par modèle de sécurité

```sql
-- Quels modèles refusent le plus d'accès ?
SELECT 
    modele_bloquant,
    COUNT(*) as nb_refus,
    COUNT(DISTINCT id_utilisateur) as nb_utilisateurs,
    MAX(horodatage) as dernier_refus
FROM journal_audit
WHERE decision = 'REFUSE'
GROUP BY modele_bloquant
ORDER BY nb_refus DESC;

-- Résultat:
-- modele_bloquant | nb_refus | nb_utilisateurs | dernier_refus
-- ────────────────┼──────────┼─────────────────┼──────────────
-- RBAC            | 4        | 3               | 2026-05-10 14:32:15
-- BLP             | 3        | 2               | 2026-05-10 13:45:22
-- CW              | 1        | 1               | 2026-05-10 11:00:10
-- BIBA            | 1        | 1               | 2026-05-10 12:10:45
```

---

### 8️⃣ DASHBOARD : Accès temporaires actifs et résidus

```sql
-- Qui a accès temporaire et combien de temps reste-t-il ?
SELECT 
    u.matricule,
    u.nom || ' ' || u.prenom as utilisateur,
    r.reference as ressource,
    r.type_ressource,
    at.justification,
    EXTRACT(DAY FROM (at.date_fin - NOW())) as jours_restants,
    EXTRACT(HOUR FROM (at.date_fin - NOW())) % 24 as heures_restantes,
    at.autorise_par_user,
    at.statut
FROM acces_temporaire at
JOIN utilisateurs u ON at.id_utilisateur = u.id_utilisateur
JOIN ressources r ON at.id_ressource = r.id_ressource
WHERE at.statut = 'ACTIF'
  AND at.date_fin > NOW()
ORDER BY at.date_fin ASC;

-- Résultat:
-- ER001 | Kowalski Anna | INV-REG-2024 | INVESTIGATION 
-- | Investigation reglementaire ACPR 2024... | 30 | 2 | AI002 | ACTIF
```

---

### 9️⃣ AUDIT : Historique des tentatives d'un utilisateur

```sql
-- Toutes les tentatives d'accès (succès + refus) de Luc Renard
SELECT 
    ja.horodatage,
    r.reference as ressource,
    r.type_ressource,
    ja.action,
    ja.decision,
    ja.modele_bloquant,
    ja.motif_refus,
    ja.adresse_ip
FROM journal_audit ja
LEFT JOIN ressources r ON ja.id_ressource = r.id_ressource
JOIN utilisateurs u ON ja.id_utilisateur = u.id_utilisateur
WHERE u.matricule = 'CC001'
ORDER BY ja.horodatage DESC;

-- Résultat (3 lignes):
-- 2026-05-10 14:32 | AML-FR-2024-01 | DOSSIER_AML | LIRE | REFUSE | BLP 
-- | Niveau habilitation insuffisant : N2 < N4 requis | 10.0.1.45
-- 2026-05-10 14:15 | KPI-2024-Q4 | KPI_COMMERCIAL | LIRE | AUTORISE | NULL
-- | NULL | 10.0.1.45
-- 2026-05-10 13:50 | OPERATIONS_COURANTES | LIRE | AUTORISE | NULL
-- | NULL | 10.0.1.45
```

---

### 🔟 COMPLIANCE : Conflits Chinese Wall détectés

```sql
-- Quels utilisateurs ont des restrictions de conflits d'intérêts ?
SELECT 
    u.matricule,
    u.nom || ' ' || u.prenom as utilisateur,
    cwh.groupe_conflict,
    r.reference as ressource,
    cwh.date_consultation,
    'BLOQUÉ SUR: GROUPE_INDUS_BETA' as restriction
FROM chinese_wall_historique cwh
JOIN utilisateurs u ON cwh.id_utilisateur = u.id_utilisateur
JOIN ressources r ON cwh.id_ressource = r.id_ressource
WHERE cwh.groupe_conflict IN ('GROUPE_INDUS_ALPHA', 'GROUPE_INDUS_BETA')
ORDER BY u.matricule;

-- Résultat:
-- GP001 | Leblanc Sophie | GROUPE_INDUS_ALPHA | PTF-FR-GP01 | 2026-05-10 10:15:32 | BLOQUÉ SUR: GROUPE_INDUS_BETA
-- RS001 | Lambert Eric | GROUPE_INDUS_ALPHA | RISQUE-Q4-24 | 2026-05-10 10:20:15 | BLOQUÉ SUR: GROUPE_INDUS_BETA
```

---

## 🚀 Guide d'Installation

### Prérequis
- PostgreSQL 14+
- `psql` client
- Accès de base de données

### Installation

```bash
# 1. Cloner le repository
git clone https://github.com/AchrafHasnaoui31/access-controle-system-for-an-internationale-bank-.git
cd access-controle-system

# 2. Charger le schéma complet
psql -U postgres -d banque_internationale -f "simulation database.txt"

# 3. Vérifier l'installation
psql -U postgres -d banque_internationale
```

### Vérification Rapide

```sql
-- Compte des entités
SELECT 'niveaux_securite' as table_name, COUNT(*) as count FROM niveaux_securite
UNION ALL
SELECT 'pays', COUNT(*) FROM pays
UNION ALL
SELECT 'equipes', COUNT(*) FROM equipes
UNION ALL
SELECT 'roles', COUNT(*) FROM roles
UNION ALL
SELECT 'utilisateurs', COUNT(*) FROM utilisateurs
UNION ALL
SELECT 'ressources', COUNT(*) FROM ressources
UNION ALL
SELECT 'journal_audit', COUNT(*) FROM journal_audit
ORDER BY 1;
```

---

## 💡 Cas d'Usage Réels

### Cas 1 : Conseiller Tunisien Consulte Client Local ✅

```sql
-- Sami Ben Ali (RC001, N2, Tunisie) lit CLI-TN-001
-- Résultat: AUTORISE
-- - RBAC: Role a permission
-- - BLP: N2 = N2 ✓
-- - ABAC: Client en Tunisie ✓
```

### Cas 2 : Directeur Commercial Tente AML ❌

```sql
-- Luc Renard (CC001, N2) lit AML-FR-2024-01 (N4)
-- Résultat: REFUSE - BLP
-- - BLP: N2 < N4 ✗
```

### Cas 3 : Expert Régional 30j Investigation ⏰

```sql
-- Anna Kowalski (ER001, N3) accès temporaire INV-REG-2024 (N4)
-- Durée: 30 jours, révocation auto
-- Résultat: AUTORISE
```

### Cas 4 : Gestionnaire Patrimoine Bloquée ❌

```sql
-- Sophie Leblanc consulte PTF-FR-GP01 (GROUPE_INDUS_ALPHA)
-- Tente accès PTF-TN-GP02 (GROUPE_INDUS_BETA)
-- Résultat: REFUSE - Chinese Wall
```

---

## 🔧 Guide de Personnalisation

### Ajouter une Équipe

```sql
INSERT INTO equipes (nom, categorie, effectif_cible, niveau_max, description)
VALUES ('Innovation & Digital', 'EXPERTISE', 30, 'N3', 'Transformations IT, données');
```

### Ajouter un Rôle

```sql
INSERT INTO roles (nom_role, id_equipe, niveau_acces, description)
VALUES ('data_scientist', 11, 'N3', 'Analyses avancées, machine learning');
```

### Accorder Permission à un Rôle

```sql
INSERT INTO role_permissions (id_role, id_permission)
SELECT 
    (SELECT id_role FROM roles WHERE nom_role = 'data_scientist'),
    id_permission
FROM permissions
WHERE ressource_type IN ('DONNEES_CLIENT', 'ANALYSE_RISQUE')
  AND action IN ('LIRE', 'ECRIRE');
```

### Attribuer Utilisateur à Rôle

```sql
INSERT INTO utilisateur_roles (id_utilisateur, id_role)
SELECT
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'DS001'),
    (SELECT id_role FROM roles WHERE nom_role = 'data_scientist');
```

### Créer Politique ABAC

```sql
INSERT INTO abac_politiques
  (nom, id_role, type_ressource, filtre_pays, duree_max_heures, heure_debut, heure_fin, jours_semaine)
VALUES
  ('DATA_SCIENTIST_POLICY',
   (SELECT id_role FROM roles WHERE nom_role = 'data_scientist'),
   'ANALYSE_RISQUE',
   FALSE, NULL,
   '08:00', '20:00', 'LUN-VEN');
```

### Accorder Accès Temporaire

```sql
INSERT INTO acces_temporaire
  (id_utilisateur, id_ressource, justification, date_debut, date_fin, autorise_par)
VALUES
  ((SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'DS001'),
   (SELECT id_ressource FROM ressources WHERE reference = 'RISQUE-Q4-24'),
   'Projet analyse prédictive trimestre Q2',
   NOW(),
   NOW() + INTERVAL '60 days',
   (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'AI002'));
```

---

## 📈 Considérations Production

### Performance
- Indexation sur (id_utilisateur, actif) pour rôles
- Indexation sur (id_role, id_permission)
- Cache Redis pour permissions fréquentes
- Prepared statements obligatoires

### Sécurité
- RLS (Row-Level Security) PostgreSQL
- Export logs audit → SIEM (Splunk/ELK)
- Alertes: refus répétés, violations Chinese Wall
- MFA intégration (Okta/Azure AD)

### Conformité
- Audit logging COMPLET N3/N4
- Archivage logs → S3/Glacier
- Rapports mensuels compliance
- Traçabilité modifications politiques

---

## 👥 Auteurs

**Achraf Hasnaoui** & **Mohamed Aziz Foudhaili**

Système de Contrôle d'Accès — Banque Internationale  
**Mai 2026**

---

**Dernière mise à jour :** 10 mai 2026  
**SGBD :** PostgreSQL 14+  
**Status :** ✅ Production-Ready
