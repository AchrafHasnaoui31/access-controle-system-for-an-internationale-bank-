# 🏦 Access Control System for an International Bank

A comprehensive, production-ready **access control framework** for international banking operations. This system implements multiple security models (RBAC, BLP, Biba, ABAC, and Chinese Wall) to manage complex, multi-country banking environments with strict regulatory compliance.

---

## 📌 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Security Models](#security-models)
- [Database Schema](#database-schema)
- [Real-World Use Cases](#real-world-use-cases)
- [Quick Start](#quick-start)
- [SQL Views & Queries](#sql-views--queries)
- [Customization Guide](#customization-guide)
- [Authors](#authors)

---

## 🎯 Overview

International banks operate in multiple countries with diverse teams (client relations, back-office, wealth management, compliance, audit, legal, etc.) accessing highly sensitive data—client portfolios, AML files, regulatory investigations, and transaction histories.

This system solves a critical challenge: **How do you grant fine-grained access while maintaining strict data isolation, regulatory compliance, and preventing conflicts of interest?**

**The Answer:** A layered, multi-model access control system that combines:
- **RBAC** (Role-Based Access Control) – Who can do what?
- **BLP** (Bell-LaPadula) – Confidentiality: no read-up, no write-down
- **Biba** – Integrity: no write-up (low-level users can't modify high-level data)
- **ABAC** (Attribute-Based Access Control) – Context matters (country, time, portfolio)
- **Chinese Wall** – Prevent conflicts of interest in trading/advisory roles

---

## ✨ Key Features

### 🔐 Multi-Model Security Framework
- **RBAC**: 24+ roles across 10 teams (Relation Client, Back-Office, Wealth Management, Risk, Compliance/AML, Audit, Legal, Commercial, Regional Experts, Support)
- **BLP**: 4 security levels (N1 Internal, N2 Sensitive, N3 Highly Regulated, N4 Critical/AML)
- **Biba**: Integrity constraints for data modification
- **ABAC**: Context-based policies (country filtering, portfolio assignment, time windows, temporary access)
- **Chinese Wall**: Conflict-of-interest prevention for competing client groups

### 🌍 Multi-Country Operations
- Support for 7+ countries (Tunisia, France, Germany, UK, USA, Singapore, Morocco)
- Regulatory zone classifications (EU, Maghreb, Asia, US)
- Country-based data filtering and access controls

### 📋 Granular Permissions
- 15+ resource types (DONNEES_CLIENT, PORTEFEUILLE, DOSSIER_KYC, DOSSIER_AML, etc.)
- 6 action types (LIRE, ECRIRE, MODIFIER, SUPPRIMER, VALIDER, EXPORTER)
- Minimum security level required per permission

### ⏰ Temporary Access & Automatic Revocation
- Grant time-bounded access for investigations or expertise
- Automatic revocation after expiry (configurable)
- Audit trail with justification

### 🔍 Comprehensive Audit Logging
- Every access attempt logged with decision reason
- Tracks which security model enforced the denial
- IP address, timestamp, and user justification
- Views for security dashboard and compliance reporting

### 🚨 Advanced Constraints
- **MFA Required** for N3/N4 data access
- **Justification Mandatory** for sensitive operations
- **Double Validation** for critical operations
- **Temporal Controls** (working hours, day-of-week restrictions)

---

## 🏗️ Architecture

### Database Layers

```
┌─────────────────────────────────────────┐
│  ACCESS CONTROL DECISION ENGINE         │
│  (controler_acces function)             │
├─────────────────────────────────────────┤
│  ✓ RBAC Check                           │
│  ✓ BLP Check (Confidentiality)          │
│  ✓ Biba Check (Integrity)               │
│  ✓ ABAC Check (Context)                 │
│  ✓ Chinese Wall Check (Conflicts)       │
├─────────────────────────────────────────┤
│  POLICY TABLES                          │
│  ├── Roles & Permissions                │
│  ├── User-Role Assignments              │
│  ├── BLP Habilitations                  │
│  ├── ABAC Policies                      │
│  ├── Chinese Wall Registry              │
│  └── Temporary Access Grants            │
├─────────────────────────────────────────┤
│  REFERENCE DATA                         │
│  ├── Users & Teams                      │
│  ├── Countries & Regulatory Zones       │
│  ├── Security Levels                    │
│  ├── Resources (Protected Objects)      │
│  └── Audit Journal                      │
└─────────────────────────────────────────┘
```

### Core Tables (16 total)

| Table | Purpose | Records |
|-------|---------|---------|
| `niveaux_securite` | Security level definitions (N1–N4) with MFA requirements | 4 |
| `pays` | Countries and regulatory zones | 7 |
| `equipes` | Team definitions (10 operational/control/expertise teams) | 10 |
| `roles` | Role definitions (24+ roles across all teams) | 24+ |
| `permissions` | Granular permissions (resource type + action combinations) | 22 |
| `role_permissions` | RBAC mapping: which roles have which permissions | 70+ |
| `utilisateurs` | User master data (matricule, email, team, country, MFA status) | 20 |
| `utilisateur_roles` | Active role assignments for users (with date tracking) | 20+ |
| `ressources` | Protected objects (clients, portfolios, investigations, AML files) | 15 |
| `blp_habilitations` | Security clearance level per user (BLP MAC model) | 20 |
| `abac_politiques` | Context rules (country filters, time windows, duration limits) | 7 |
| `acces_temporaire` | Time-bounded access grants with automatic revocation | Variable |
| `chinese_wall_historique` | Conflict-of-interest tracking (groups already accessed) | Variable |
| `journal_audit` | Complete access decision log (all attempts, denials, reasons) | Variable |

---

## 🔒 Security Models Explained

### 1️⃣ RBAC (Role-Based Access Control)

**Question:** Does this user's role allow this action on this resource type?

**Example Teams & Roles:**
- **Relation Clientèle** (3 roles): Advisor, Business Advisor, Agency Manager
- **Back-Office** (2 roles): Operator, Supervisor
- **Wealth Management** (3 roles): Advisor, Portfolio Analyst, Manager
- **Risk** (3 roles): Credit Analyst, Market Analyst, Manager
- **Compliance/AML** (4 roles): AML Analyst, KYC Manager, Sanctions Specialist, Manager
- **Audit** (3 roles): Auditor, Senior Auditor, Audit Director
- **Legal** (3 roles): Banking Counsel, Compliance Counsel, General Counsel

**Rule:** A user can perform `action` on `resource_type` **only if** their role(s) have an explicit permission entry.

**SQL Check:**
```sql
SELECT verifier_rbac(
    p_id_utilisateur := 1,           -- User ID
    p_type_ressource := 'DONNEES_CLIENT',  -- Resource type
    p_action := 'LIRE'               -- Action
);
```

---

### 2️⃣ BLP (Bell-LaPadula) – Confidentiality

**Question:** Is the user's security clearance level sufficient to access this data?

**4 Security Levels:**
- **N1** (Internal Operational): Routine operations, internal reference data
- **N2** (Sensitive): Client data, contracts, transaction history (audit required)
- **N3** (Highly Regulated): Detailed KYC, market orders, strategic risk (MFA required)
- **N4** (Critical/Confidential): AML files, sanctions, regulatory investigations (MFA + double validation)

**Rules:**
- **No Read-Up:** User with N2 clearance cannot read N3/N4 data
- **No Write-Down:** User with N3 clearance cannot write to N1/N2 data (except under Biba rules)

**Example:**
- Back-office operator (N1 clearance) **denied** reading a risk analysis (N3)
- ✅ Reason: BLP violation – insufficient clearance level

---

### 3️⃣ Biba – Integrity

**Question:** Can this user modify data without violating integrity constraints?

**Rules:**
- **No Write-Up:** A low-level user (N1) cannot write to high-level data (N3/N4)
- **No Read-Down (Optional):** A high-level user shouldn't read low-level data (relaxed for auditors)

**Example:**
- Back-office operator (N1) attempts to modify a risk report (N3)
- ✅ Denied: Biba violation – no write-up

---

### 4️⃣ ABAC (Attribute-Based Access Control) – Context Matters

**Question:** Given the context (user attributes, resource attributes, environment), is access allowed?

**Contextual Filters:**
- **Country Filtering**: Client Advisor restricted to clients in their home country
- **Portfolio Assignment**: Wealth Manager can only access portfolios assigned to them
- **Time Windows**: Access only during business hours (08:00–18:30)
- **Duration Limits**: Temporary access expires automatically
- **Day Restrictions**: Access only Mon–Fri (or 24/7 for AML)

**Example Policies:**
```
Policy: RC_pays_portefeuille
├─ Role: conseiller_particulier
├─ Country Filter: ✓ (advisor only sees own country)
├─ Portfolio Filter: ✓ (advisor only sees assigned clients)
├─ Hours: 08:00–18:30
└─ Days: Mon–Sat

Policy: EXPERT_temporaire
├─ Role: expert_regional
├─ Max Duration: 720 hours (30 days)
├─ Hours: 08:00–18:00
└─ Days: Mon–Fri
```

**SQL Example:**
```sql
INSERT INTO abac_politiques (nom, id_role, type_ressource, filtre_pays, duree_max_heures)
VALUES ('GP_portefeuille_strict', 3, 'PORTEFEUILLE', TRUE, NULL);
```

---

### 5️⃣ Chinese Wall – Conflict of Interest

**Question:** Has this user already accessed competing information that would create a conflict?

**Use Case:** A wealth manager cannot advise both Company A and its competitor Company B.

**Rules:**
- Track which "conflict group" (e.g., GROUPE_INDUS_ALPHA) a user accesses
- Once accessed, block access to conflicting groups (e.g., GROUPE_INDUS_BETA)

**Example:**
- Wealth Manager Sophie accesses Portfolio A (Group: Industrial Alpha)
- ✅ Later denied access to Portfolio B (Group: Industrial Beta) due to Chinese Wall

---

## 📊 Database Schema

### Detailed Table Specifications

#### **Reference Tables**

**niveaux_securite** - Security Level Definitions
```sql
CREATE TABLE niveaux_securite (
    niveau          niveau_enum PRIMARY KEY,
    designation     VARCHAR(50),
    description     TEXT,
    mfa_requis      BOOLEAN,
    double_valid    BOOLEAN,
    journalisation  VARCHAR(20)  -- STANDARD, RENFORCE, COMPLET
);
```

**pays** - Countries & Regulatory Zones
```sql
CREATE TABLE pays (
    code_pays       CHAR(2) PRIMARY KEY,  -- ISO 3166-1
    nom             VARCHAR(100),
    zone_regulatoire VARCHAR(50)  -- UE, MAGHREB, ASIE, US
);
```

**equipes** - Team Definitions
```sql
CREATE TABLE equipes (
    id_equipe       SERIAL PRIMARY KEY,
    nom             VARCHAR(100) UNIQUE,
    categorie       VARCHAR(30),  -- OPERATIONNEL, CONTROLE, EXPERTISE, SUPPORT
    effectif_cible  INT,
    niveau_max      niveau_enum,
    description     TEXT
);
```

#### **RBAC Tables**

**roles** - Role Definitions
```sql
CREATE TABLE roles (
    id_role         SERIAL PRIMARY KEY,
    nom_role        VARCHAR(80) UNIQUE,
    id_equipe       INT REFERENCES equipes,
    niveau_acces    niveau_enum,
    description     TEXT
);
```

**permissions** - Granular Action Permissions
```sql
CREATE TABLE permissions (
    id_permission   SERIAL PRIMARY KEY,
    ressource_type  VARCHAR(50),
    action          action_enum,  -- LIRE, ECRIRE, MODIFIER, SUPPRIMER, VALIDER, EXPORTER
    niveau_min      niveau_enum,
    description     TEXT,
    UNIQUE(ressource_type, action)
);
```

**role_permissions** - RBAC Mapping
```sql
CREATE TABLE role_permissions (
    id_role         INT REFERENCES roles,
    id_permission   INT REFERENCES permissions,
    PRIMARY KEY (id_role, id_permission)
);
```

#### **User & Resource Tables**

**utilisateurs** - User Master Data
```sql
CREATE TABLE utilisateurs (
    id_utilisateur  SERIAL PRIMARY KEY,
    matricule       VARCHAR(20) UNIQUE,
    nom             VARCHAR(100),
    prenom          VARCHAR(100),
    email           VARCHAR(150) UNIQUE,
    id_equipe       INT REFERENCES equipes,
    code_pays       CHAR(2) REFERENCES pays,
    statut          statut_enum,  -- ACTIF, INACTIF, SUSPENDU, EXPIRE
    mfa_actif       BOOLEAN DEFAULT FALSE,
    derniere_cnx    TIMESTAMP,
    date_creation   TIMESTAMP DEFAULT NOW()
);
```

**utilisateur_roles** - User Role Assignments
```sql
CREATE TABLE utilisateur_roles (
    id_utilisateur  INT REFERENCES utilisateurs,
    id_role         INT REFERENCES roles,
    date_debut      DATE DEFAULT CURRENT_DATE,
    date_fin        DATE,  -- NULL = permanent
    actif           BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (id_utilisateur, id_role)
);
```

**ressources** - Protected Objects
```sql
CREATE TABLE ressources (
    id_ressource    SERIAL PRIMARY KEY,
    reference       VARCHAR(50) UNIQUE,
    type_ressource  VARCHAR(50),
    niveau_requis   niveau_enum,
    code_pays       CHAR(2) REFERENCES pays,
    id_proprietaire INT REFERENCES utilisateurs,
    description     TEXT,
    actif           BOOLEAN DEFAULT TRUE,
    date_creation   TIMESTAMP DEFAULT NOW()
);
```

#### **MAC (Mandatory Access Control) Tables**

**blp_habilitations** - User Security Clearance
```sql
CREATE TABLE blp_habilitations (
    id_utilisateur  INT PRIMARY KEY REFERENCES utilisateurs,
    niveau_max      niveau_enum,  -- Maximum clearance level
    date_attribution DATE,
    attribue_par    INT REFERENCES utilisateurs,
    commentaire     TEXT
);
```

#### **ABAC & Context Tables**

**abac_politiques** - Contextual Access Policies
```sql
CREATE TABLE abac_politiques (
    id_politique    SERIAL PRIMARY KEY,
    nom             VARCHAR(100),
    id_role         INT REFERENCES roles,
    type_ressource  VARCHAR(50),
    filtre_pays     BOOLEAN,
    filtre_portefeuille BOOLEAN,
    duree_max_heures INT,  -- NULL = no time limit
    heure_debut     TIME,  -- Business hour start
    heure_fin       TIME,  -- Business hour end
    jours_semaine   VARCHAR(20),  -- LUN-VEN, LUN-DIM, etc.
    description     TEXT
);
```

**acces_temporaire** - Time-Bounded Access Grants
```sql
CREATE TABLE acces_temporaire (
    id_acces        SERIAL PRIMARY KEY,
    id_utilisateur  INT REFERENCES utilisateurs,
    id_ressource    INT REFERENCES ressources,
    justification   TEXT NOT NULL,
    date_debut      TIMESTAMP DEFAULT NOW(),
    date_fin        TIMESTAMP NOT NULL,
    autorise_par    INT REFERENCES utilisateurs,
    statut          statut_enum DEFAULT 'ACTIF',
    revocation_auto BOOLEAN DEFAULT TRUE,
    date_revocation TIMESTAMP,
    CONSTRAINT chk_dates CHECK (date_fin > date_debut)
);
```

#### **Conflict & Audit Tables**

**chinese_wall_historique** - Conflict-of-Interest Tracking
```sql
CREATE TABLE chinese_wall_historique (
    id              SERIAL PRIMARY KEY,
    id_utilisateur  INT REFERENCES utilisateurs,
    id_ressource    INT REFERENCES ressources,
    groupe_conflict VARCHAR(50),  -- GROUPE_INDUS_A, GROUPE_INDUS_B, etc.
    date_consultation TIMESTAMP DEFAULT NOW(),
    UNIQUE (id_utilisateur, groupe_conflict)
);
```

**journal_audit** - Complete Access Decision Log
```sql
CREATE TABLE journal_audit (
    id_journal      BIGSERIAL PRIMARY KEY,
    id_utilisateur  INT REFERENCES utilisateurs,
    id_ressource    INT REFERENCES ressources,
    type_ressource  VARCHAR(50),
    action          action_enum,
    decision        decision_enum,  -- AUTORISE, REFUSE, EN_ATTENTE
    motif_refus     TEXT,  -- Reason if REFUSED
    modele_bloquant VARCHAR(20),  -- RBAC, ABAC, BLP, BIBA, CW, SYSTEME
    adresse_ip      INET,
    horodatage      TIMESTAMP DEFAULT NOW(),
    justification   TEXT  -- Required for N3/N4
);
```

### Entity Relationship Diagram (Conceptual)

```
┌──────────────────┐        ┌──────────────┐
│  utilisateurs    │────────│  equipes     │
├──────────────────┤        ├──────────────┤
│ id_utilisateur   │        │ id_equipe    │
│ matricule        │        │ nom          │
│ nom, prenom      │        │ categorie    │
│ email            │        │ niveau_max   │
│ code_pays ───────┼────────┼──┐           │
│ statut           │        │  │           │
│ mfa_actif        │        │  │           │
└──────────────────┘        └──────────────┘
        │                           │
        │ ┌─────────────────────────┘
        │ │
        ├─┤  ┌──────────────────────┐
        │ └──│  roles               │
        │    ├──────────────────────┤
        │    │ id_role              │
        │    │ nom_role             │
        │    │ niveau_acces (N1-N4) │
        │    └──────────────────────┘
        │           │
        │           │ ┌─────────────────────┐
        │           └─│ permissions         │
        │             ├─────────────────────┤
        │             │ ressource_type      │
        │             │ action (LIRE, etc.) │
        │             │ niveau_min          │
        │             └─────────────────────┘
        │
        ├─── utilisateur_roles ───┐
        │                         │
        │                    ┌────────────────┐
        │                    │ blp_habilitations
        │                    ├────────────────┤
        │                    │ niveau_max     │
        │                    │ (N1-N4)        │
        │                    └────────────────┘
        │
        ├─── ressources ──────────┐
        │                         │
        │                    ┌────────────────┐
        │                    │ journal_audit  │
        │                    ├────────────────┤
        │                    │ action         │
        │                    │ decision       │
        │                    │ modele_bloquant│
        │                    └────────────────┘
        │
        └─── acces_temporaire
             ├── id_ressource
             └── date_fin (auto-revocation)
```

---

## 🎬 Real-World Use Cases

### ✅ Case 1: Client Advisor Reading Own Client Data

**User:** Sami Ben Ali (Relation Clientèle, Tunisia)
**Action:** Read client data (CLI-TN-001)
**Decision:** ✅ **AUTORISE**

**Security Checks:**
| Check | Result | Reason |
|-------|--------|--------|
| RBAC | ✅ Pass | Role `conseiller_particulier` has permission `DONNEES_CLIENT/LIRE` |
| BLP | ✅ Pass | User N2 clearance ≥ Resource N2 level |
| Biba | ✅ Pass | N/A (read operation) |
| ABAC | ✅ Pass | Client in Tunisia (user's country) & assigned portfolio |
| Chinese Wall | ✅ Pass | No conflict recorded |

**Audit Log:**
```
Utilisateur: Sami Ben Ali (RC001)
Ressource: CLI-TN-001 (DONNEES_CLIENT, N2)
Action: LIRE
Décision: AUTORISE
Modele: NONE
IP: 196.20.1.10
Justification: NULL (N2 doesn't require justification)
```

---

### ❌ Case 2: Commercial Director Attempts AML Access

**User:** Luc Renard (Commercial Director, France)
**Action:** Read AML investigation file (AML-FR-2024-01)
**Decision:** ❌ **REFUSE**

**Security Checks:**
| Check | Result | Reason |
|-------|--------|--------|
| RBAC | ❌ **FAIL** | Role `directeur_commercial` has no permission on `DOSSIER_AML/LIRE` |
| BLP | ❌ **FAIL** | User N2 clearance < Resource N4 required |
| Biba | N/A | Not reached |
| ABAC | N/A | Not reached |
| Chinese Wall | N/A | Not reached |

**Audit Log:**
```
Utilisateur: Luc Renard (CC001)
Ressource: AML-FR-2024-01 (DOSSIER_AML, N4)
Action: LIRE
Décision: REFUSE
Motif: Niveau habilitation insuffisant : N2 < N4 requis
Modele Bloquant: BLP
IP: 10.0.1.45
```

---

### ⏰ Case 3: Regional Expert Temporary Access

**User:** Anna Kowalski (Expert Regional, France)
**Action:** Read regulatory investigation (INV-REG-2024, N4)
**Duration:** 30 days (auto-revoke)
**Decision:** ✅ **AUTORISE** (with justification)

**Security Checks:**
| Check | Result | Reason |
|-------|--------|--------|
| RBAC | ✅ Pass | Temporary role grant via `acces_temporaire` |
| BLP | ✅ Pass | User N3 clearance + Temporary N4 grant |
| Biba | ✅ Pass | Read-only operation |
| ABAC | ✅ Pass | Temporary access window: 30 days / Hours: 08:00–18:00 |
| Chinese Wall | ✅ Pass | Investigation context doesn't conflict |

**Database Entry:**
```sql
INSERT INTO acces_temporaire
  (id_utilisateur, id_ressource, justification, date_debut, date_fin, autorise_par)
VALUES
  (19, (SELECT id_ressource FROM ressources WHERE reference='INV-REG-2024'),
   'Investigation reglementaire ACPR 2024 - expertise droit tunisien requise',
   NOW(),
   NOW() + INTERVAL '30 days',
   (SELECT id_utilisateur FROM utilisateurs WHERE matricule='AI002'));
```

---

### 🚫 Case 4: Chinese Wall Conflict Detection

**User:** Sophie Leblanc (Wealth Manager, France)
**Action 1:** Read Portfolio A (PTF-FR-GP01, Group: GROUPE_INDUS_ALPHA) ✅ AUTORISE
**Action 2:** Read Portfolio B (PTF-TN-GP02, Group: GROUPE_INDUS_BETA) ❌ REFUSE

**Reason:** Chinese Wall – After accessing GROUPE_INDUS_ALPHA, Sophie is blocked from competing group.

**SQL:**
```sql
-- After first access to Portfolio A (GROUPE_INDUS_ALPHA)
INSERT INTO chinese_wall_historique (id_utilisateur, id_ressource, groupe_conflict)
VALUES (6, 1, 'GROUPE_INDUS_ALPHA');

-- Subsequent access to Portfolio B (GROUPE_INDUS_BETA) is REFUSED
SELECT verifier_chinese_wall(6, 2);  -- Returns FALSE
```

---

## 🚀 Quick Start

### 1. Prerequisites

- **PostgreSQL 14+**
- Git (to clone this repo)

### 2. Load the SQL Schema

```bash
# Clone repository
git clone https://github.com/AchrafHasnaoui31/access-controle-system-for-an-internationale-bank-.git
cd access-controle-system

# Load the schema
psql -U postgres -d your_database -f "simulation database.txt"
```

### 3. Verify Installation

```sql
-- Check tables
\dt

-- Verify functions
\df verifier_*

-- Count users
SELECT COUNT(*) FROM utilisateurs;  -- Should be 20

-- Count roles
SELECT COUNT(*) FROM roles;  -- Should be 24+
```

### 4. Test Access Control

```sql
-- Test Case 1: Client Advisor reads own client data (AUTHORIZED)
SELECT * FROM controler_acces(
    p_id_utilisateur := 1,         -- Sami Ben Ali
    p_id_ressource := 1,           -- CLI-TN-001
    p_action := 'LIRE'::action_enum,
    p_justification := NULL
);

-- Result: AUTORISE, NULL, "Acces accorde - toutes les politiques validees"

-- Test Case 2: Commercial Director attempts AML access (DENIED)
SELECT * FROM controler_acces(
    p_id_utilisateur := 17,        -- Luc Renard (Commercial)
    p_id_ressource := 7,           -- AML-FR-2024-01
    p_action := 'LIRE'::action_enum,
    p_justification := NULL
);

-- Result: REFUSE, "BLP", "Violation Bell-LaPadula..."
```

---

## 📊 SQL Views & Queries

### Dashboard Query 1: User Rights Summary

```sql
-- What can each user do?
SELECT
    utilisateur,
    role,
    ressource_type,
    action,
    niveau_requis,
    niveau_habilite
FROM v_droits_utilisateur
WHERE matricule = 'RC001'
ORDER BY ressource_type, action;
```

**Output:**
```
utilisateur   role                    ressource_type       action    niveau_requis  niveau_habilite
Sami Ben Ali  conseiller_particulier  DONNEES_CLIENT       LIRE      N2             N2
Sami Ben Ali  conseiller_particulier  DONNEES_CLIENT       MODIFIER  N2             N2
Sami Ben Ali  conseiller_particulier  DOSSIER_KYC          LIRE      N2             N2
...
```

### Dashboard Query 2: Access Denials by Security Model

```sql
-- Which security model is denying access the most?
SELECT * FROM v_refus_par_modele
ORDER BY nb_refus DESC;
```

**Output:**
```
modele_bloquant  nb_refus  utilisateurs_concernes  dernier_refus
RBAC             12        8                       2026-05-10 14:32
BLP              7         5                       2026-05-10 13:45
BIBA             3         2                       2026-05-10 12:10
CW               1         1                       2026-05-10 11:00
```

### Dashboard Query 3: Active Temporary Access Grants

```sql
-- Who has temporary access and how much time remains?
SELECT
    utilisateur,
    ressource,
    justification,
    heures_restantes,
    autorise_par
FROM v_acces_temporaires_actifs
ORDER BY heures_restantes ASC;
```

### Dashboard Query 4: Chinese Wall Conflicts

```sql
-- Which users have conflict-of-interest restrictions?
SELECT * FROM v_chinese_wall_alertes;
```

---

## 🔧 Customization Guide

### Add a New Team

```sql
INSERT INTO equipes (nom, categorie, effectif_cible, niveau_max, description)
VALUES ('Compliance Trade', 'CONTROLE', 15, 'N4', 'Trade compliance & KYC');
```

### Add a New Role

```sql
INSERT INTO roles (nom_role, id_equipe, niveau_acces, description)
VALUES ('compliance_officer_trade', 11, 'N3', 'Trade compliance oversight');
```

### Grant Permissions to a Role

```sql
-- Grant 'compliance_officer_trade' permission to read trade contracts
INSERT INTO role_permissions (id_role, id_permission)
SELECT 
    (SELECT id_role FROM roles WHERE nom_role = 'compliance_officer_trade'),
    id_permission
FROM permissions
WHERE ressource_type = 'CONTRAT_CREDIT' AND action = 'LIRE';
```

### Assign a User to a Role

```sql
INSERT INTO utilisateur_roles (id_utilisateur, id_role)
SELECT
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'CT001'),
    (SELECT id_role FROM roles WHERE nom_role = 'compliance_officer_trade');
```

### Create an ABAC Policy

```sql
INSERT INTO abac_politiques
  (nom, id_role, type_ressource, filtre_pays, duree_max_heures, heure_debut, heure_fin, jours_semaine, description)
VALUES
  ('COMPLIANCE_TRADE_POLICY',
   (SELECT id_role FROM roles WHERE nom_role = 'compliance_officer_trade'),
   'CONTRAT_CREDIT',
   TRUE,   -- Filter by country
   NULL,   -- No time limit
   '08:00', '20:00',
   'LUN-VEN',
   'Trade compliance: country-filtered, business hours only');
```

### Grant Temporary Access

```sql
INSERT INTO acces_temporaire
  (id_utilisateur, id_ressource, justification, date_debut, date_fin, autorise_par)
VALUES
  ((SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'EX001'),
   (SELECT id_ressource FROM ressources WHERE reference = 'INV-REG-2024'),
   'Special investigation project',
   NOW(),
   NOW() + INTERVAL '14 days',
   (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'AI002'));
```

### Revoke a Role

```sql
UPDATE utilisateur_roles
SET actif = FALSE, date_fin = CURRENT_DATE
WHERE id_utilisateur = (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'EX001')
  AND id_role = (SELECT id_role FROM roles WHERE nom_role = 'expert_regional');
```

---

## 📈 Production Considerations

### 1. **Scaling Access Control Decisions**
   - Cache permission lookups in Redis for high-volume systems
   - Implement request batching for bulk access checks
   - Use database connection pooling (PgBouncer)

### 2. **Audit & Compliance**
   - Enable PostgreSQL row-level security (RLS) for additional protection
   - Implement log archival to cold storage (S3/Glacier)
   - Generate weekly/monthly compliance reports
   - Export audit logs to SIEM (Splunk, ELK)

### 3. **MFA Integration**
   - Connect to enterprise MFA provider (Okta, Azure AD, Duo)
   - Validate MFA status before N3/N4 resource access
   - Log MFA challenges to audit trail

### 4. **Monitoring & Alerting**
   - Alert on repeated access denials from same user
   - Monitor for suspicious Chinese Wall violations
   - Track performance of `controler_acces` function
   - Dashboard: Denials by model, users with highest permissions, policy changes

### 5. **Performance Optimization**
   - Index on `(id_utilisateur, actif)` for role lookups
   - Index on `(id_role, id_permission)` for permission checks
   - Materialize expensive views (v_droits_utilisateur)
   - Use prepared statements for all queries

---

## 📚 References

- **Bell-LaPadula Model**: Confidentiality (no read-up, no write-down)
- **Biba Model**: Integrity (no write-up)
- **RBAC**: Role-Based Access Control (Sandhu et al.)
- **ABAC**: Attribute-Based Access Control (NIST SP 800-162)
- **Chinese Wall**: Separation of duties (Brewer-Nash Model)

---

## 👥 Authors

**Achraf Hasnaoui** & **Mohamed Aziz Foudhaili**

International Bank Access Control System – Sprint 2  
May 2026

---

## 📝 License

This project is provided as-is for educational and enterprise banking compliance purposes.

---

## 🤝 Support & Questions

For implementation questions or security audits, review the included:
- `database structure.html` – Visual ER diagram
- `simulation database.txt` – Complete SQL schema
- `Sprint_2_Banque_Internationale_...pdf` – Full technical documentation

---

**Last Updated:** May 10, 2026  
**Database:** PostgreSQL 14+  
**Status:** ✅ Production-Ready
