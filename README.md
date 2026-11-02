# International Bank Access Control System

> Academic PostgreSQL security and database-modeling project for a multinational banking environment, combining RBAC, ABAC, Bell–LaPadula, Biba, Chinese Wall, row-level security, temporary access, automatic provisioning, revocation, and auditability.

[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-14%2B-336791?style=flat-square&logo=postgresql&logoColor=white)](#technical-stack)
[![Security models](https://img.shields.io/badge/security%20models-5-7C3AED?style=flat-square)](#security-models)
[![Status](https://img.shields.io/badge/status-academic%20prototype-2563EB?style=flat-square)](#status-and-scope)
[![SQL](https://img.shields.io/badge/implementation-PL%2FpgSQL-0F766E?style=flat-square)](#database-implementation)
[![License](https://img.shields.io/badge/license-not%20specified-6B7280?style=flat-square)](#license)

## Table of contents

- [Project overview](#project-overview)
- [Problem statement](#problem-statement)
- [Objectives](#objectives)
- [Status and scope](#status-and-scope)
- [Architecture at a glance](#architecture-at-a-glance)
- [Security models](#security-models)
- [Security levels](#security-levels)
- [Domain model](#domain-model)
- [Database implementation](#database-implementation)
- [Access-decision pipeline](#access-decision-pipeline)
- [RBAC design](#rbac-design)
- [ABAC policies](#abac-policies)
- [Bell–LaPadula confidentiality](#belllapadula-confidentiality)
- [Biba integrity](#biba-integrity)
- [Chinese Wall conflict prevention](#chinese-wall-conflict-prevention)
- [Row-level security](#row-level-security)
- [Temporary access](#temporary-access)
- [Provisioning and revocation](#provisioning-and-revocation)
- [Audit and reporting](#audit-and-reporting)
- [Repository contents](#repository-contents)
- [Technical stack](#technical-stack)
- [Prerequisites](#prerequisites)
- [Installation and execution](#installation-and-execution)
- [Validation scenarios](#validation-scenarios)
- [Expected security behavior](#expected-security-behavior)
- [Threats and design considerations](#threats-and-design-considerations)
- [Known limitations](#known-limitations)
- [Production hardening roadmap](#production-hardening-roadmap)
- [Contributing](#contributing)
- [License](#license)

## Project overview

Banks operate across countries, business lines, regulatory regimes, and sensitivity levels. A customer adviser, a back-office operator, an investment analyst, an AML investigator, an internal auditor, and a legal specialist should not receive the same access simply because they use the same application.

This project models that problem as a database-enforced authorization system. Instead of treating permissions as a collection of frontend conditions, it represents users, teams, roles, resources, security levels, context policies, conflict histories, temporary grants, audit events, and PostgreSQL roles as explicit data and executable policies.

The central decision function is:

```sql
controler_acces(
    p_id_utilisateur,
    p_id_ressource,
    p_action,
    p_justification
)
```

It returns a structured decision:

```text
decision | modele_bloquant | message
```

An access request is granted only when all required layers validate it. The first failing layer produces a refusal reason and an audit record.

## Problem statement

The modeled bank needs to protect:

- Customer and company data.
- Credit contracts and transaction histories.
- Wealth-management portfolios.
- Risk analyses and regulatory reports.
- KYC and AML investigations.
- Fraud and sanctions alerts.
- Internal audit journals.
- Regulatory investigations and communications.
- Branch and back-office operations.
- Commercial KPIs.

The authorization problem is difficult because access depends on more than a username:

- The employee's business role.
- The employee's team and country.
- The sensitivity level of the resource.
- The requested action.
- The time and day of the request.
- The employee's previous consultations.
- Whether an access is temporary and justified.
- Whether MFA is active.
- Whether a second validation is required.
- Whether the action would compromise confidentiality or integrity.

The project addresses these requirements through multiple complementary models rather than relying on one authorization mechanism.

## Objectives

### Primary objectives

- Define a relational representation of banking users, resources, roles, permissions, and policies.
- Demonstrate layered authorization decisions.
- Enforce least privilege through role and resource constraints.
- Protect confidential information using security levels.
- Protect data integrity against unsafe writes.
- Prevent conflicts of interest dynamically.
- Support controlled temporary access for exceptional investigations.
- Record authorized and refused decisions in an audit journal.
- Demonstrate PostgreSQL roles, grants, and row-level security.
- Provide repeatable SQL scenarios for positive and negative testing.

### Non-goals

- Replacing a certified banking IAM platform.
- Demonstrating regulatory compliance by itself.
- Providing production-ready identity proofing, MFA, key management, or incident response.
- Storing real customer, employee, or regulatory data.
- Acting as a complete banking core system.

## Status and scope

**Academic security and database prototype.**

The repository contains SQL, a visual database structure, test scenarios, simulation data, a presentation, and a written project report. It does not contain a production API, frontend, deployment pipeline, migration framework, secrets-management system, or external identity provider.

The design is intentionally educational: it makes security concepts visible and executable in PostgreSQL. Before production use, the policies, privilege boundaries, database security posture, operational controls, and regulatory assumptions would require independent review.

## Architecture at a glance

```text
Application or SQL client
          │
          ▼
PostgreSQL roles and grants
          │
          ▼
Row-level security policies
          │
          ▼
controler_acces(...)
          │
          ├── Authentication and user status
          ├── MFA requirement
          ├── Business justification
          ├── RBAC permission check
          ├── Bell–LaPadula confidentiality check
          ├── Biba integrity check
          ├── ABAC context check
          ├── Chinese Wall conflict check
          └── Audit journal entry
```

### Layered decision principle

```text
Request
  │
  ├─ Resource active?
  ├─ User active?
  ├─ MFA sufficient?
  ├─ Justification present for N3/N4?
  ├─ RBAC allows role/action?
  ├─ BLP allows confidentiality flow?
  ├─ Biba preserves integrity?
  ├─ ABAC allows context?
  └─ Chinese Wall allows conflict class?
          │
          ├── Any failure → REFUSE + reason + audit record
          └── All pass    → AUTORISE + audit record
```

The ordering is deliberate: inexpensive and fundamental checks happen before the more contextual policy checks, and every refusal identifies the blocking model.

## Security models

| Model | Question answered | Main protection |
|---|---|---|
| RBAC | Is this role allowed to perform this action on this resource type? | Least privilege and separation of duties |
| ABAC | Is the current context valid for this user, resource, time, country, or assignment? | Context-aware authorization |
| Bell–LaPadula | Is the confidentiality level compatible with the request? | Confidentiality and information-flow control |
| Biba | Would this operation compromise integrity? | Protection against unsafe writes and contamination |
| Chinese Wall | Has this user already accessed a conflicting organization? | Conflict-of-interest prevention |

These models are not interchangeable. RBAC can express job responsibilities, but it does not by itself express time windows, country restrictions, information-flow levels, or dynamic conflicts. The project demonstrates how the models can be combined.

## Security levels

The database defines four ordered levels:

| Level | Meaning | Typical examples | Controls |
|---|---|---|---|
| `N1` | Internal business | Routine operations, branch activity, internal references | Standard authentication |
| `N2` | Sensitive | Customer data, contracts, transaction history | Reinforced logging |
| `N3` | Very sensitive / regulated | Detailed KYC, strategic risk, wealth portfolios | MFA and business justification |
| `N4` | Critical / highly confidential | AML files, sanctions, regulatory investigations | MFA, double validation, complete logging |

The SQL implementation stores level metadata in `niveaux_securite`, including:

- `mfa_requis`.
- `double_valid`.
- `journalisation`.
- Human-readable designation and description.

The current decision function enforces MFA and justification requirements for high levels. The `double_valid` field documents the intended control, but a complete second-approval workflow remains future work.

## Domain model

```text
pays
  └── equipes
        ├── roles
        │     └── role_permissions ── permissions
        └── utilisateurs
              ├── utilisateur_roles ── roles
              ├── blp_habilitations
              ├── journal_audit
              ├── acces_temporaire
              └── chinese_wall_historique

ressources
  ├── security level
  ├── country
  ├── owner
  └── conflict group/class

abac_politiques
  └── contextual constraints by role and resource type
```

### Main entities

| Table | Responsibility |
|---|---|
| `pays` | Countries and regulatory zones |
| `niveaux_securite` | Ordered confidentiality/integrity levels and control requirements |
| `equipes` | Business teams, categories, target staffing, and maximum level |
| `roles` | Business roles linked to teams and access levels |
| `permissions` | Resource/action pairs with minimum required level |
| `role_permissions` | RBAC association between roles and permissions |
| `utilisateurs` | Employee identity, team, country, status, and MFA state |
| `utilisateur_roles` | Active and date-bounded role assignments |
| `ressources` | Protected banking objects and their classification |
| `blp_habilitations` | Maximum level authorized for a user |
| `abac_politiques` | Contextual restrictions such as country, time, day, and duration |
| `acces_temporaire` | Explicitly justified, time-bounded exceptional access |
| `chinese_wall_historique` | Consultation history by conflict group/class |
| `journal_audit` | Authorized and refused decisions, reasons, and metadata |

## Database implementation

The main implementation is stored in the repository file:

```text
simulation database.txt
```

The filename is historical and has no `.sql` extension, but it contains PostgreSQL SQL and PL/pgSQL. The script is designed as a full initialization script:

1. Remove prior triggers, functions, views, tables, types, and `banque_*` roles.
2. Create enum types.
3. Create reference and security tables.
4. Insert simulation data.
5. Create RBAC associations.
6. Create ABAC policies and temporary access.
7. Create conflict history and audit records.
8. Add indexes and reporting views.
9. Create security verification functions.
10. Create PostgreSQL roles and grants.
11. Enable row-level security.
12. Create automatic provisioning, revocation, and temporary-access functions.
13. Create triggers.
14. Leave validation queries available at the end of the script.

### PostgreSQL types

```sql
CREATE TYPE niveau_enum   AS ENUM ('N1','N2','N3','N4');
CREATE TYPE action_enum   AS ENUM ('LIRE','ECRIRE','MODIFIER','SUPPRIMER','VALIDER','EXPORTER');
CREATE TYPE decision_enum AS ENUM ('AUTORISE','REFUSE','EN_ATTENTE');
CREATE TYPE statut_enum   AS ENUM ('ACTIF','INACTIF','SUSPENDU','EXPIRE');
```

Enums make the policy vocabulary explicit and prevent invalid action, decision, level, or status values from silently entering the database.

### Referential integrity

Foreign keys connect users to teams and countries, resources to owners and countries, role assignments to users and roles, permissions to roles, and audit records to users and resources. Cascading behavior is used selectively for associations such as role-permission links and user-role records.

### Indexing

The script adds indexes for the main authorization and reporting paths:

- Active roles by user.
- Permissions by role.
- BLP habilitations by user.
- Chinese Wall history by user and conflict group/class.
- Active resources by type.
- Active resource references.
- Audit entries by user and timestamp.
- Audit decisions and blocking models.
- Temporary access by user/status and expiration.

## Access-decision pipeline

The `controler_acces()` function is the main policy orchestrator.

### Step 1 — Resource validation

The function refuses inactive or unknown resources with a `SYSTEME` blocking model.

### Step 2 — User status

Only users with `statut = 'ACTIF'` continue. Inactive or suspended users are refused before policy evaluation.

### Step 3 — MFA

The resource's security level determines whether MFA is required. N3 and N4 resources require `mfa_actif = TRUE`.

### Step 4 — Business justification

Requests involving N3 or N4 resources require a non-empty justification. This makes sensitive access explainable and gives the audit record meaningful context.

### Step 5 — RBAC

`verifier_rbac()` checks whether an active role assignment maps to a permission matching the resource type and requested action.

### Step 6 — Bell–LaPadula

`verifier_blp()` compares the user's maximum authorized level with the resource level and applies confidentiality flow rules.

### Step 7 — Biba

`verifier_biba()` blocks writes that would move information toward a higher integrity level than the user's authorization.

### Step 8 — ABAC

`verifier_abac()` evaluates role-specific context constraints such as country, time window, day of week, resource type, and temporary duration policy.

### Step 9 — Chinese Wall

`verifier_chinese_wall()` checks whether the user has already consulted another group in the same conflict class. A conflicting request is refused and does not become an allowed consultation.

### Step 10 — Audit

Every final decision is inserted into `journal_audit`, including:

- User.
- Resource.
- Resource type.
- Action.
- Decision.
- Blocking model.
- Refusal reason.
- Justification.
- Timestamp.
- Optional IP address in seeded examples.

## RBAC design

RBAC represents business responsibility through roles and granular permissions.

### Example business teams

- Relation clientèle.
- Back-office.
- Gestion de patrimoine.
- Risque.
- Conformité / AML.
- Audit interne.
- Juridique.
- Cadres commerciaux.
- Experts régionaux.
- Support.

### Example roles

- `conseiller_particulier`.
- `conseiller_entreprise`.
- `responsable_agence`.
- `operateur_bo`.
- `superviseur_bo`.
- `conseiller_patrimoine`.
- `analyste_risque_credit`.
- `analyste_aml`.
- `specialiste_sanctions`.
- `auditeur`.
- `auditeur_senior`.
- `juriste_conformite`.
- `directeur_commercial`.
- `expert_regional`.
- `agent_support`.
- `guichetier_responsable`.

### Permission granularity

Permissions are defined as a resource type/action pair with a minimum level, for example:

```text
DONNEES_CLIENT / LIRE
PORTEFEUILLE / ECRIRE
DOSSIER_AML / LIRE
ALERTE_FRAUDE / VALIDER
JOURNAL_AUDIT / LIRE
RAPPORT_REGULATEUR / EXPORTER
OPERATIONS_GUICHET / VALIDER
```

This makes the difference between “can access a domain” and “can perform a specific operation” explicit.

## ABAC policies

ABAC adds context to role-based authorization. The table `abac_politiques` supports:

- Role association.
- Resource type filtering.
- Country filtering.
- Portfolio filtering metadata.
- Maximum temporary duration.
- Start and end time.
- Allowed days.
- Human-readable policy descriptions.

Examples included in the simulation:

| Policy | Intended constraint |
|---|---|
| `RC_pays_portefeuille` | Customer adviser is limited to country and assigned portfolio |
| `GP_portefeuille_strict` | Wealth adviser is limited to assigned portfolio |
| `RISQUE_lecture_seule` | Credit risk analyst reads credit contracts during working hours |
| `AML_dossier_assigne` | AML analyst accesses assigned AML dossiers |
| `AUDIT_lecture_globale` | Senior auditor receives broad read access during audit hours |
| `EXPERT_temporaire` | Regional expert receives bounded investigation access |
| `COMMERCIAL_kpi_seulement` | Commercial manager sees aggregated KPIs |
| `GUICHET_resp_operations` | Branch manager handles branch operations in a time window |

### Important implementation note

The current ABAC function evaluates country, hour, and day constraints. Some schema fields, such as `filtre_portefeuille`, are modeled but require a richer assignment model to enforce portfolio ownership completely. This is a documented extension point rather than an assumption that all contextual controls are already complete.

## Bell–LaPadula confidentiality

The project uses Bell–LaPadula to protect confidentiality.

### Implemented rules

For read-like operations:

```text
No read up
```

A user cannot read a resource classified above the user's maximum level.

For write-like operations:

```text
No write down
```

A higher-level user cannot write to a lower-level resource through the BLP check.

The implementation converts `N1` to `N4` into ordered values and compares the user's habilitation with the resource's required level.

### Example

```text
User: CC001, maximum N2
Resource: AML-FR-2024-01, level N4
Action: LIRE
Result: REFUSE
Blocking model: BLP
```

## Biba integrity

Biba complements Bell–LaPadula by protecting the integrity of information.

### Implemented rule

For modifying operations:

```text
No write up
```

A lower-level user cannot write to a higher-level resource.

### Example

```text
User: BO001, maximum N1
Resource: RISQUE-Q4-24, level N3
Action: MODIFIER
Result: REFUSE
Blocking model: BIBA
```

This prevents a routine back-office account from modifying a strategic risk report or another higher-integrity dataset.

## Chinese Wall conflict prevention

The Chinese Wall model prevents a user from crossing conflict boundaries after accessing one organization in a conflict class.

### Data representation

`ressources` stores:

- `groupe_conflict`.
- `classe_conflict`.

`chinese_wall_historique` records:

- User.
- Consulted resource.
- Conflict group.
- Conflict class.
- Consultation timestamp.

A uniqueness constraint prevents duplicate history rows for the same user and group.

### Decision logic

1. If a resource has no conflict group, no Chinese Wall restriction applies.
2. If the user has already accessed a different group in the same class, the request is refused.
3. If no conflict exists, the consultation is allowed and recorded.

### Example

```text
Previous consultation: GROUPE_INDUS_ALPHA / INDUSTRIE
Requested resource:    GROUPE_INDUS_BETA  / INDUSTRIE
Result:                REFUSE
Blocking model:        CHINESE_WALL
```

## Row-level security

The script enables and forces PostgreSQL RLS on:

- `utilisateurs`.
- `ressources`.
- `journal_audit`.
- `chinese_wall_historique`.

### Current policy examples

- A user can read their own user row.
- Audit, DBA, and readonly roles receive broader visibility where appropriate.
- Resources are filtered by country unless the connected PostgreSQL role has elevated audit, AML, or DBA privileges.
- Audit journal entries are visible to the requesting user and approved audit, AML, legal, DBA, or readonly roles.
- Chinese Wall history is visible to the user and approved audit/DBA roles.

The helper function `current_user_id()` maps the PostgreSQL login naming convention to an active business user.

### Security caveat

RLS policies are powerful but require production review for owner bypass, `SECURITY DEFINER` behavior, role membership, search paths, and administrative connections. The included policies are an educational demonstration, not a complete database-hardening baseline.

## Temporary access

Exceptional access is represented by `acces_temporaire`.

Each grant stores:

- User.
- Protected resource.
- Business justification.
- Start time.
- End time.
- Authorizing administrator.
- Status.
- Automatic-revocation flag.
- Revocation timestamp.

`accorder_acces_temporaire()` checks:

1. User, resource, and administrator existence.
2. Administrator BLP level.
3. Maximum duration from ABAC policies.
4. Insertion of the time-bounded grant.
5. Audit journal creation.

The seed scenario grants a regional expert 30-day access to a regulatory investigation, authorized by a senior auditor.

## Provisioning and revocation

### Automatic provisioning

`attribuer_acces_utilisateur()` performs a multi-step assignment:

1. Verify the active user and resolve their team.
2. Map the team to a PostgreSQL role.
3. Create a login role if required by the simulation.
4. Grant the team role.
5. Assign RBAC roles.
6. Create or update the BLP habilitation.
7. Enable MFA automatically for N3/N4 teams.
8. Write an audit entry.

An `AFTER INSERT` trigger calls this function for new users.

### Revocation

`revoquer_acces_utilisateur()`:

- Deactivates all RBAC assignments.
- Expires active temporary access.
- Suspends the business user.
- Changes the PostgreSQL login to `NOLOGIN`.
- Writes a revocation audit entry.

### Expired temporary access

The trigger `trg_revocation_auto` updates expired temporary records after insert/update activity. The script also documents a production-oriented `pg_cron` schedule for periodic expiration, because a trigger alone does not guarantee time-based execution when no write event occurs.

## Audit and reporting

The `journal_audit` table is the central evidence source for access decisions.

### Recorded information

- User and resource.
- Type of resource.
- Requested action.
- Final decision.
- Refusal reason.
- Blocking security model.
- IP address where supplied.
- Timestamp.
- Business justification.

### Reporting views

| View | Purpose |
|---|---|
| `v_droits_utilisateur` | Effective active RBAC rights and habilitation levels |
| `v_refus_par_modele` | Refusal counts grouped by blocking model |
| `v_acces_temporaires_actifs` | Active temporary grants and remaining hours |
| `v_chinese_wall_alertes` | Users and conflict groups that create dynamic barriers |

These views create a foundation for security dashboards, review workflows, and audit investigations.

## Repository contents

```text
access-controle-system-for-an-internationale-bank-/
├── simulation database.txt
│   └── Complete PostgreSQL schema, data, functions, roles, RLS, triggers, and tests
├── Test querries.txt
│   └── Focused positive/negative access and automatic-provisioning examples
├── database structure.html
│   └── Interactive visual schema reference
├── banque_controle_acces.pptx
│   └── Architecture, security models, team distribution, and validation presentation
├── Banque_Internationale_Achraf hasnaoui & mohamedaziz foudhaili.pdf
│   └── Written project report
├── `internationale banque `
│   └── Domain scenario and problem context
└── README.md
```

The source filenames are preserved from the original academic project. In a future cleanup, the SQL and documentation artifacts should be renamed to conventional, extension-aware names without changing their meaning.

## Technical stack

### Database

- PostgreSQL 14 or newer.
- SQL and PL/pgSQL.
- PostgreSQL enum types.
- Foreign keys, unique constraints, indexes, views, triggers, roles, grants, and RLS.

### Security concepts

- RBAC.
- ABAC.
- Bell–LaPadula.
- Biba.
- Chinese Wall / Brewer–Nash.
- Least privilege.
- Separation of duties.
- MFA gating.
- Audit logging.
- Temporary access and revocation.

### Documentation

- HTML/CSS visual schema.
- PowerPoint architecture and validation presentation.
- PDF project report.
- SQL comments and test scenarios.

## Prerequisites

- PostgreSQL 14+.
- A database user allowed to create:
  - Types.
  - Tables.
  - Functions.
  - Triggers.
  - Views.
  - PostgreSQL roles and grants.
  - RLS policies.
- `psql` command-line client.
- Permission to create or alter the target database.

The script creates roles and uses dynamic `CREATE ROLE`, `GRANT`, `ALTER ROLE`, and RLS statements. It should therefore be executed only in an isolated development database with an administrative account.

## Installation and execution

### Create an isolated database

```bash
createdb -U postgres banque_controle_acces
```

Alternatively:

```bash
psql -U postgres -c "CREATE DATABASE banque_controle_acces;"
```

### Execute the complete simulation

Because the SQL file has a space and no extension, quote the path:

```bash
psql -U postgres -d banque_controle_acces \
  -v ON_ERROR_STOP=1 \
  -f 'simulation database.txt'
```

The script intentionally resets objects with names used by the project. Never point it at a database containing unrelated production objects.

### Check the created objects

```bash
psql -U postgres -d banque_controle_acces -c "\dt"
psql -U postgres -d banque_controle_acces -c "\df controler_acces"
psql -U postgres -d banque_controle_acces -c "\dv"
```

### Run the focused examples

```bash
psql -U postgres -d banque_controle_acces \
  -v ON_ERROR_STOP=1 \
  -f 'Test querries.txt'
```

The file contains calls to `controler_acces()` and `attribuer_acces_utilisateur()`. Some examples are designed to demonstrate refusal paths and should not be interpreted as failures when they return `REFUSE` as expected.

## Validation scenarios

### Scenario 1 — MFA refusal

```sql
SELECT * FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'CC001'),
    (SELECT id_ressource FROM ressources WHERE reference = 'AML-FR-2024-01'),
    'LIRE',
    NULL
);
```

Expected behavior:

```text
decision: REFUSE
modele_bloquant: AUTH
message: MFA obligatoire pour niveau N4
```

### Scenario 2 — RBAC refusal

```sql
SELECT * FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'RC001'),
    (SELECT id_ressource FROM ressources WHERE reference = 'OPS-BO-2024'),
    'VALIDER',
    NULL
);
```

Expected behavior:

```text
decision: REFUSE
modele_bloquant: RBAC
```

The adviser role does not receive the supervisor-only validation permission.

### Scenario 3 — Chinese Wall refusal

```sql
SELECT * FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'RS001'),
    (SELECT id_ressource FROM ressources WHERE reference = 'PTF-TN-GP02'),
    'LIRE',
    'Analyse exposition croisee groupes'
);
```

Expected behavior:

```text
decision: REFUSE
modele_bloquant: CHINESE_WALL
```

The seeded history records previous access to `GROUPE_INDUS_ALPHA` in the `INDUSTRIE` conflict class.

### Scenario 4 — Automatic access assignment

```sql
SELECT * FROM attribuer_acces_utilisateur('GR001', 'AI002');
```

The function reports steps for:

1. User verification.
2. PostgreSQL role mapping.
3. RBAC role assignment.
4. BLP habilitation.
5. MFA status.
6. Audit journal insertion.

### Scenario 5 — Temporary access

```sql
SELECT * FROM accorder_acces_temporaire(
    'ER001',
    'INV-REG-2024',
    720,
    'Expertise droit tunisien requise pour investigation',
    'AI002'
);
```

Expected checks include administrator level, maximum duration, grant creation, and audit recording.

### Scenario 6 — Revocation

```sql
SELECT * FROM revoquer_acces_utilisateur(
    'RC001',
    'Depart entreprise',
    'AI002'
);
```

Expected effects:

- Active RBAC assignments become inactive.
- Temporary access becomes expired.
- The business account becomes suspended.
- The PostgreSQL login becomes `NOLOGIN`.
- A revocation event is written to the audit journal.

## Expected security behavior

| Request | Expected result | Blocking layer |
|---|---|---|
| N2 adviser reads assigned N2 customer data | Allowed | All checks pass |
| N2 commercial manager reads N4 AML file without MFA | Refused | Authentication/MFA |
| Adviser validates a back-office operation | Refused | RBAC |
| N1 operator modifies N3 risk analysis | Refused | Biba |
| N2 user reads N4 AML investigation | Refused | Bell–LaPadula |
| User crosses two industry conflict groups | Refused | Chinese Wall |
| User accesses an inactive resource | Refused | System/resource validation |
| Suspended user requests any resource | Refused | User status |
| N3/N4 request has no justification | Refused | Policy gate |
| Authorized regional expert receives bounded investigation access | Allowed temporarily | ABAC and temporary-access checks |

## Threats and design considerations

This project is useful because it exposes threats that a simple role check would miss:

### Privilege escalation

An employee may receive a role that is too broad or retain a previous role after a transfer. The design addresses this through explicit role assignments, active/date-bounded records, automatic provisioning, and revocation.

### Confidentiality leakage

A user with a legitimate business role may still request a resource above their classification. BLP and MFA/justification gates address this case.

### Integrity contamination

A low-level operational account may attempt to alter a higher-integrity risk or audit record. Biba and role permissions provide separate protections.

### Conflict of interest

A user may legitimately access one company's data and then attempt to access a competitor's data. Chinese Wall history creates a dynamic barrier based on previous consultation.

### Excessive temporary access

Emergency or expert access can become permanent if not bounded. Temporary grants include duration, authorization, status, and revocation fields, with ABAC duration checks.

### Audit gaps

If allowed and denied decisions are not recorded, investigations become difficult. The access function writes a journal event for every terminal decision path.

### Database privilege bypass

PostgreSQL owners, superusers, `SECURITY DEFINER` functions, role inheritance, and administrative connections can bypass application assumptions. Production deployment must review database ownership, search paths, function privileges, and operational accounts.

## Known limitations

- The repository is an academic prototype, not a certified banking system.
- No production identity provider or real MFA integration is included.
- Passwords generated by the demonstration provisioning function are placeholders and must never be used in a real environment.
- Dynamic role creation and grants require strict administrative controls.
- The SQL script resets objects and should not be executed against an unrelated database.
- The `double_valid` field is modeled, but a complete two-person approval workflow is not implemented.
- Some ABAC fields, including portfolio filtering metadata, need richer ownership/assignment data for complete enforcement.
- Time and day evaluation depends on the PostgreSQL session/server timezone and should be made explicit in production.
- The expiration trigger is not a true scheduler; a periodic job such as `pg_cron` is needed for reliable time-based revocation.
- The Chinese Wall history and audit journal require stronger immutability controls for production.
- The RLS policies are educational and require a formal privilege-bypass review.
- There is no API, frontend, migration framework, CI pipeline, backup strategy, or disaster-recovery procedure.
- The seeded data is synthetic and should not be mistaken for real bank data.
- No formal threat model, penetration test, compliance assessment, or independent security review is included.
- The database and documentation filenames should be normalized for maintainability.
- Regulatory references in the presentation are contextual discussion, not evidence of compliance.

## Production hardening roadmap

### Database and migrations

- Rename the main SQL file to a conventional `.sql` name.
- Split initialization into versioned migrations.
- Add transactional migration checks and rollback strategy.
- Add explicit schema ownership and `search_path` hardening.
- Restrict `SECURITY DEFINER` functions and set safe search paths.

### Identity and authentication

- Integrate an enterprise identity provider.
- Replace demonstration login creation with managed identity lifecycle.
- Integrate phishing-resistant MFA.
- Remove generated default passwords entirely.
- Add account recovery, session management, device trust, and credential rotation.

### Policy correctness

- Define a formal policy language or policy decision point.
- Add portfolio ownership and assignment tables.
- Implement double validation for N4 resources.
- Make temporal and jurisdiction policies timezone-aware.
- Add policy versioning and approval history.
- Add explicit deny precedence and policy conflict resolution.

### Audit and monitoring

- Make audit records append-only and tamper-evident.
- Stream security events to a SIEM.
- Add alerting for repeated denials, privilege changes, and unusual access.
- Add immutable retention and legal-hold policies.
- Separate operational logs from sensitive business data.

### Testing and assurance

- Add automated PostgreSQL integration tests.
- Test every model independently and in the composed pipeline.
- Add negative tests for privilege escalation and RLS bypass.
- Add property-based tests for level ordering and policy combinations.
- Add threat modeling, dependency scanning, and secret scanning.
- Perform an independent security review before any sensitive deployment.

### Operations

- Add least-privilege deployment roles.
- Add encrypted backups and restore drills.
- Add high-availability and disaster-recovery procedures.
- Add monitoring for database health and policy latency.
- Document incident response and emergency-access procedures.

## Contributing

1. Work only with synthetic data or an authorized isolated environment.
2. Do not add credentials, real customer data, private keys, or production connection strings.
3. Explain any policy change in terms of confidentiality, integrity, authorization, auditability, or operational impact.
4. Add both allowed and denied test scenarios for security changes.
5. Preserve the distinction between academic demonstration and production capability.
6. Run the SQL script only against a disposable development database.
7. Update the visual schema and README when tables, functions, or security models change.
8. Keep security experiments responsible and do not provide unauthorized operational guidance.

## License

No open-source license is currently specified. Until a license file is added, the original SQL, documentation, presentation, report, and visual design should be treated as all rights reserved by the authors. PostgreSQL and any external tools remain subject to their own licenses and terms.
