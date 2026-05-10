# 🏦 Système de Contrôle d'Accès — Banque Internationale

> Système **multi-modèles production-ready** combinant RBAC, BLP, Biba, ABAC et Chinese Wall dans PostgreSQL.

**Auteurs :** Achraf Hasnaoui & Mohamed Aziz Foudhaili  
**Date :** Mai 2026  
**SGBD :** PostgreSQL 14+

---

## 📌 Table des Matières

- [Vue d'ensemble](#-vue-densemble)
- [Architecture & Modèles de Sécurité](#-architecture--modèles-de-sécurité)
- [Schéma de Base de Données](#-schéma-de-base-de-données)
- [Guide d'Installation](#-guide-dinstallation)
- [Requêtes de Test](#-requêtes-de-test)
- [Tableaux de Bord](#-tableaux-de-bord)
- [Fonctions Automatiques](#-fonctions-automatiques)
- [Guide de Personnalisation](#-guide-de-personnalisation)
- [Considérations Production](#-considérations-production)

---

## 🎯 Vue d'Ensemble

### Problématique

Les banques internationales opèrent dans plusieurs pays avec des équipes diversifiées accédant à des données hautement sensibles. Comment accorder un accès granulaire tout en maintenant l'isolation stricte des données, la conformité réglementaire et la prévention des conflits d'intérêts ?

### Solution : 5 couches de sécurité superposées

```
Demande d'accès
      │
      ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│    RBAC     │────▶│     BLP     │────▶│    BIBA     │────▶│    ABAC     │────▶│ CHINESE WALL │
│  Qui peut   │     │ Confidentiel│     │  Intégrité  │     │  Contexte   │     │   Conflits   │
│  faire quoi │     │  no-read-up │     │ no-write-up │     │ pays/horaire│     │  d'intérêts  │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └──────────────┘
      │                   │                   │                   │                     │
   REFUSE              REFUSE              REFUSE              REFUSE               REFUSE
                                                                                       │
                                                                                   AUTORISÉ ✅
```

| Modèle | Règle clé | Exemple de blocage |
|--------|-----------|-------------------|
| **RBAC** | Rôle × Permission | Un conseiller RC ne peut pas lire un portefeuille GP |
| **BLP** | no-read-up / no-write-down | N2 ne peut pas lire un dossier N4 |
| **Biba** | no-write-up | N1 ne peut pas modifier une ressource N3 |
| **ABAC** | Pays, horaires, durée | Guichet bloqué hors 07h30-18h30 |
| **Chinese Wall** | Groupes concurrents | GP ayant vu INDUS_ALPHA bloqué sur INDUS_BETA |

---

## 🏗️ Architecture & Modèles de Sécurité

### Niveaux de Sécurité (N1 → N4)

```
┌──────────────────────────────────────────────────────────────────┐
│  N4 — CRITIQUE / CONFIDENTIEL EXTRÊME  (MFA + double validation) │
│       Dossiers AML, sanctions OFAC/ONU, investigations ACPR      │
├──────────────────────────────────────────────────────────────────┤
│  N3 — TRÈS SENSIBLE / RÉGLEMENTÉ       (MFA obligatoire)         │
│       KYC détaillé, ordres de marché, risque stratégique         │
├──────────────────────────────────────────────────────────────────┤
│  N2 — SENSIBLE                         (log renforcé)            │
│       Données clients, contrats, historiques transactions        │
├──────────────────────────────────────────────────────────────────┤
│  N1 — INTERNE MÉTIER                   (log standard)            │
│       Opérations courantes, référentiels internes                │
└──────────────────────────────────────────────────────────────────┘
```

### Les 10 Équipes

| ID | Équipe | Catégorie | Niveau Max |
|----|--------|-----------|------------|
| 1 | Relation clientèle | OPÉRATIONNEL | N2 |
| 2 | Back-office | OPÉRATIONNEL | N2 |
| 3 | Gestion de patrimoine | OPÉRATIONNEL | N3 |
| 4 | Risque | OPÉRATIONNEL | N3 |
| 5 | Conformité / AML | CONTRÔLE | N4 |
| 6 | Audit interne | CONTRÔLE | N4 |
| 7 | Juridique | CONTRÔLE | N4 |
| 8 | Cadres commerciaux | EXPERTISE | N2 |
| 9 | Experts régionaux | EXPERTISE | N3 |
| 10 | Support | SUPPORT | N1 |

---

## 📊 Schéma de Base de Données

```
                        ┌──────────────────┐
                        │  niveaux_securite│
                        └────────┬─────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
  ┌───────▼──────┐    ┌──────────▼──────┐    ┌─────────▼────────┐
  │    equipes   │    │  utilisateurs   │    │    ressources    │
  └───────┬──────┘    └────────┬────────┘    └─────────┬────────┘
          │                    │                       │
  ┌───────▼──────┐    ┌────────▼────────┐    ┌─────────▼────────┐
  │    roles     │    │blp_habilitations│    │chinese_wall_hist │
  └───────┬──────┘    └─────────────────┘    └──────────────────┘
          │
  ┌───────▼──────┐    ┌─────────────────┐
  │ permissions  │    │utilisateur_roles│
  └───────┬──────┘    └─────────────────┘
          │
  ┌───────▼──────┐    ┌─────────────────┐    ┌──────────────────┐
  │role_perms    │    │ abac_politiques │    │  acces_temporaire│
  └──────────────┘    └─────────────────┘    └──────────────────┘
                                                       │
                                             ┌─────────▼────────┐
                                             │  journal_audit   │
                                             └──────────────────┘
```

**14 tables — 4 types énumérés — 6 vues — 8 fonctions — 2 triggers**

---

## 🚀 Guide d'Installation

### Prérequis

- PostgreSQL 14+
- Client `psql`
- Accès superutilisateur

### Installation en une commande

```bash
# Cloner le repository
git clone https://github.com/AchrafHasnaoui31/access-controle-system-for-an-internationale-bank-.git
cd access-controle-system

# Déployer sur la base cible
psql -U postgres -d ma_base -f banque_internationale_complet.sql
```

### Vérification post-installation

```sql
-- Compter les entités chargées
SELECT table_name, COUNT(*) AS nb_lignes
FROM (
    SELECT 'utilisateurs'      AS table_name FROM utilisateurs
    UNION ALL
    SELECT 'roles'             FROM roles
    UNION ALL
    SELECT 'permissions'       FROM permissions
    UNION ALL
    SELECT 'ressources'        FROM ressources
    UNION ALL
    SELECT 'blp_habilitations' FROM blp_habilitations
    UNION ALL
    SELECT 'journal_audit'     FROM journal_audit
) t
GROUP BY table_name
ORDER BY table_name;

-- Résultat attendu :
-- blp_habilitations │ 20
-- journal_audit     │  9
-- permissions       │ 22
-- ressources        │ 17
-- roles             │ 24
-- utilisateurs      │ 20
```

---

## 🧪 Requêtes de Test

> Toutes les requêtes utilisent la **fonction centrale `controler_acces()`** qui orchestre les 5 couches de sécurité et journalise automatiquement chaque décision.

### Signature de la fonction

```sql
SELECT * FROM controler_acces(
    p_id_utilisateur  INT,      -- ID de l'utilisateur
    p_id_ressource    INT,      -- ID de la ressource
    p_action          action_enum,  -- LIRE | ECRIRE | MODIFIER | SUPPRIMER | VALIDER | EXPORTER
    p_justification   TEXT      -- Obligatoire pour N3/N4, NULL sinon
);
-- Retourne : decision | modele_bloquant | message
```

---

### TEST 1 — ✅ AUTORISÉ : Conseiller lit son propre client

```sql
-- Sami Ben Ali (RC001, N2, Tunisie) lit CLI-TN-001 (DONNEES_CLIENT, N2, Tunisie)
-- Toutes les couches doivent passer

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'RC001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'CLI-TN-001'),
    'LIRE',
    NULL
);

-- Résultat attendu :
-- decision  │ modele_bloquant │ message
-- ──────────┼─────────────────┼──────────────────────────────────────
-- AUTORISE  │ NULL            │ Acces accorde - toutes les politiques validees
```

**Pourquoi autorisé ?**
- ✅ RBAC : `conseiller_particulier` possède `DONNEES_CLIENT/LIRE`
- ✅ BLP  : niveau user N2 ≥ niveau ressource N2
- ✅ Biba : N2 = N2 (pas de write-up)
- ✅ ABAC : client en Tunisie = pays de l'utilisateur
- ✅ Chinese Wall : ressource sans groupe de conflit

---

### TEST 2 — ❌ REFUSÉ (BLP) : Directeur commercial accède à un dossier AML

```sql
-- Luc Renard (CC001, N2) tente de lire AML-FR-2024-01 (DOSSIER_AML, N4)
-- BLP doit bloquer : no-read-up → N2 < N4

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'CC001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'AML-FR-2024-01'),
    'LIRE',
    NULL
);

-- Résultat attendu :
-- decision │ modele_bloquant │ message
-- ─────────┼─────────────────┼──────────────────────────────────────────
-- REFUSE   │ BLP             │ Violation Bell-LaPadula : no-read-up ou no-write-down
```

**Vérification directe via la fonction BLP :**

```sql
-- Confirmer le refus BLP isolément
SELECT verifier_blp(
    (SELECT id_utilisateur FROM utilisateurs  WHERE matricule  = 'CC001'),
    (SELECT niveau_requis  FROM ressources    WHERE reference  = 'AML-FR-2024-01'),
    'LIRE'
) AS blp_passe;

-- Résultat : false
```

---

### TEST 3 — ❌ REFUSÉ (RBAC) : Conseiller RC accède à un portefeuille GP

```sql
-- Sami Ben Ali (RC001) tente de lire PTF-FR-GP01 (PORTEFEUILLE, N3)
-- RBAC doit bloquer : conseiller_particulier n'a pas la permission PORTEFEUILLE/LIRE

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'RC001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'PTF-FR-GP01'),
    'LIRE',
    NULL
);

-- Résultat attendu :
-- decision │ modele_bloquant │ message
-- ─────────┼─────────────────┼──────────────────────────────────────────
-- REFUSE   │ RBAC            │ Role insuffisant pour PORTEFEUILLE/LIRE
```

**Vérification directe via la fonction RBAC :**

```sql
SELECT verifier_rbac(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'RC001'),
    'PORTEFEUILLE',
    'LIRE'
) AS rbac_passe;

-- Résultat : false
```

---

### TEST 4 — ❌ REFUSÉ (Biba) : Opérateur BO modifie une analyse risque

```sql
-- Nadia Trabelsi (BO001, N1) tente de modifier RISQUE-Q4-24 (ANALYSE_RISQUE, N3)
-- Biba doit bloquer : no-write-up → N1 ne peut pas écrire sur N3

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'BO001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'RISQUE-Q4-24'),
    'MODIFIER',
    'Correction erreur saisie'
);

-- Résultat attendu :
-- decision │ modele_bloquant │ message
-- ─────────┼─────────────────┼──────────────────────────────────────────
-- REFUSE   │ BIBA            │ Violation Biba : no-write-up interdit
```

**Vérification directe via la fonction Biba :**

```sql
SELECT verifier_biba(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule  = 'BO001'),
    (SELECT niveau_requis  FROM ressources    WHERE reference = 'RISQUE-Q4-24'),
    'MODIFIER'
) AS biba_passe;

-- Résultat : false
```

---

### TEST 5 — ✅ AUTORISÉ : Analyste AML lit son dossier (N4 avec MFA + justification)

```sql
-- Julie Moreau (AML001, N4, MFA actif) lit AML-FR-2024-01 (DOSSIER_AML, N4)
-- MFA actif + justification fournie → toutes les couches passent

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'AML001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'AML-FR-2024-01'),
    'LIRE',
    'Investigation blanchiment dossier 2024-01 — assigné en date du 10/05/2026'
);

-- Résultat attendu :
-- decision  │ modele_bloquant │ message
-- ──────────┼─────────────────┼──────────────────────────────────────────
-- AUTORISE  │ NULL            │ Acces accorde - toutes les politiques validees
```

---

### TEST 6 — ❌ REFUSÉ (POLITIQUE) : N4 sans justification

```sql
-- Même utilisateur, même ressource — mais sans justification
-- La couche POLITIQUE doit bloquer avant même RBAC/BLP

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'AML001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'AML-FR-2024-01'),
    'LIRE',
    NULL   -- <-- justification manquante
);

-- Résultat attendu :
-- decision │ modele_bloquant │ message
-- ─────────┼─────────────────┼──────────────────────────────────────────────────────────
-- REFUSE   │ POLITIQUE       │ Justification metier obligatoire pour niveau N4
```

---

### TEST 7 — ✅ AUTORISÉ : Auditeur senior lit le journal d'audit

```sql
-- Leila Bouaziz (AI002, N4) lit AUDIT-LOG-01 (JOURNAL_AUDIT, N3)
-- Auditeur senior a lecture globale sans filtre

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'AI002'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'AUDIT-LOG-01'),
    'LIRE',
    'Mission audit trimestrielle Q4 2026'
);

-- Résultat attendu :
-- decision  │ modele_bloquant │ message
-- ──────────┼─────────────────┼──────────────────────────────────────────
-- AUTORISE  │ NULL            │ Acces accorde - toutes les politiques validees
```

---

### TEST 8 — ❌ REFUSÉ (Chinese Wall) : Gestionnaire patrimoine accède au groupe concurrent

```sql
-- Sophie Leblanc (GP001) a déjà consulté PTF-FR-GP01 (GROUPE_INDUS_ALPHA)
-- Elle tente maintenant d'accéder à PTF-TN-GP02 (GROUPE_INDUS_BETA, même classe INDUSTRIE)
-- Chinese Wall doit détecter le conflit et bloquer

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'GP001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'PTF-TN-GP02'),
    'LIRE',
    'Analyse comparative portefeuilles secteur industriel'
);

-- Résultat attendu :
-- decision │ modele_bloquant │ message
-- ─────────┼─────────────────┼───────────────────────────────────────────
-- REFUSE   │ CHINESE_WALL    │ Conflit d interet detecte (Chinese Wall)
```

**Vérification directe via la fonction Chinese Wall :**

```sql
SELECT verifier_chinese_wall(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'GP001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'PTF-TN-GP02')
) AS chinese_wall_passe;

-- Résultat : false
```

---

### TEST 9 — ❌ REFUSÉ (BLP) : Guichetier tente de lire un dossier AML

```sql
-- Isabelle Leroux (GR002, N2) tente de lire AML-FR-2024-01 (N4)

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'GR002'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'AML-FR-2024-01'),
    'LIRE',
    NULL
);

-- Résultat attendu :
-- decision │ modele_bloquant │ message
-- ─────────┼─────────────────┼──────────────────────────────────────────────────────
-- REFUSE   │ BLP             │ Violation Bell-LaPadula : no-read-up ou no-write-down
```

---

### TEST 10 — ✅ AUTORISÉ : Guichetier valide une opération guichet

```sql
-- Mehdi Gharbi (GR001, N2) valide GUICHET-TN-2024 (OPERATIONS_GUICHET, N1)
-- Rôle guichetier_responsable possède OPERATIONS_GUICHET/VALIDER

SELECT *
FROM controler_acces(
    (SELECT id_utilisateur FROM utilisateurs WHERE matricule = 'GR001'),
    (SELECT id_ressource   FROM ressources    WHERE reference = 'GUICHET-TN-2024'),
    'VALIDER',
    NULL
);

-- Résultat attendu :
-- decision  │ modele_bloquant │ message
-- ──────────┼─────────────────┼──────────────────────────────────────────
-- AUTORISE  │ NULL            │ Acces accorde - toutes les politiques validees
```

---

## 📊 Tableaux de Bord

### Vue 1 — Droits effectifs d'un utilisateur

```sql
-- Tous les droits actifs de Sophie Leblanc (GP001)
SELECT *
FROM v_droits_utilisateur
WHERE matricule = 'GP001'
ORDER BY ressource_type, action;

-- Colonnes : matricule | utilisateur | equipe | role
--            ressource_type | action | niveau_requis | niveau_habilite
```

**Droits effectifs de n'importe quel utilisateur :**

```sql
-- Passer le matricule souhaité
SELECT
    role,
    ressource_type,
    action,
    niveau_requis,
    niveau_habilite,
    CASE
        WHEN niveau_habilite >= niveau_requis THEN '✅ OK'
        ELSE '❌ BLOQUÉ (BLP)'
    END AS statut_blp
FROM v_droits_utilisateur
WHERE matricule = 'RC001'   -- ← changer ici
ORDER BY ressource_type, action;
```

---

### Vue 2 — Refus par modèle de sécurité (tableau de bord sécurité)

```sql
-- Quels modèles bloquent le plus ? Depuis combien de temps ?
SELECT *
FROM v_refus_par_modele;

-- Résultat :
-- modele_bloquant │ nb_refus │ utilisateurs_concernes │ dernier_refus
-- ────────────────┼──────────┼───────────────────────┼────────────────────────
-- RBAC            │        4 │                      3 │ 2026-05-10 14:32:15
-- BLP             │        3 │                      2 │ 2026-05-10 13:45:22
-- BIBA            │        1 │                      1 │ 2026-05-10 12:10:45
-- CW              │        1 │                      1 │ 2026-05-10 11:00:10
```

---

### Vue 3 — Accès temporaires actifs

```sql
-- Qui a encore un accès temporaire actif et combien de temps reste-t-il ?
SELECT *
FROM v_acces_temporaires_actifs
ORDER BY heures_restantes ASC;

-- Colonnes : matricule | utilisateur | ressource | type_ressource
--            justification | date_debut | date_fin | heures_restantes | autorise_par
```

---

### Vue 4 — Alertes Chinese Wall

```sql
-- Quels utilisateurs sont bloqués par des conflits d'intérêts ?
SELECT *
FROM v_chinese_wall_alertes
ORDER BY date_consultation DESC;

-- Résultat :
-- matricule │ utilisateur     │ equipe                 │ groupe_consulte      │ classe_conflict │ date_consultation    │ statut
-- ──────────┼─────────────────┼────────────────────────┼──────────────────────┼─────────────────┼──────────────────────┼──────────────────────────────────────
-- GP001     │ Leblanc Sophie  │ Gestion de patrimoine  │ GROUPE_INDUS_ALPHA   │ INDUSTRIE       │ 2026-05-10 10:15:32  │ Bloque sur groupes conflictuels de meme classe
-- RS001     │ Lambert Eric    │ Risque                 │ GROUPE_INDUS_ALPHA   │ INDUSTRIE       │ 2026-05-10 10:20:15  │ Bloque sur groupes conflictuels de meme classe
```

---

### Journal d'audit complet

```sql
-- Les 20 dernières décisions d'accès (toutes natures)
SELECT
    ja.horodatage,
    u.matricule,
    u.nom || ' ' || u.prenom          AS utilisateur,
    ja.type_ressource,
    ja.action,
    ja.decision,
    ja.modele_bloquant,
    ja.motif_refus,
    ja.adresse_ip
FROM   journal_audit ja
JOIN   utilisateurs  u  ON ja.id_utilisateur = u.id_utilisateur
ORDER  BY ja.horodatage DESC
LIMIT  20;
```

**Historique d'un utilisateur précis :**

```sql
-- Toutes les tentatives de Luc Renard (succès + refus)
SELECT
    ja.horodatage,
    r.reference          AS ressource,
    ja.type_ressource,
    ja.action,
    ja.decision,
    ja.modele_bloquant,
    ja.motif_refus,
    ja.adresse_ip
FROM   journal_audit ja
JOIN   utilisateurs  u  ON ja.id_utilisateur = u.id_utilisateur
LEFT   JOIN ressources r ON ja.id_ressource  = r.id_ressource
WHERE  u.matricule = 'CC001'
ORDER  BY ja.horodatage DESC;
```

---

## ⚙️ Fonctions Automatiques

### Attribution automatique d'un nouvel utilisateur

```sql
-- Attribue automatiquement : rôles RBAC + habilitation BLP + MFA si N3/N4 + login PostgreSQL
SELECT *
FROM attribuer_acces_utilisateur('RC001', 'AI002');

-- Résultat :
-- etape                    │ statut      │ detail
-- ─────────────────────────┼─────────────┼──────────────────────────────────────────────
-- Verification utilisateur │ OK          │ id=1 equipe=Relation clientele niveau=N2
-- Role PostgreSQL          │ OK          │ login=rc001 → banque_rc
-- Roles RBAC               │ OK          │ 4 role(s) attribue(s) pour Relation clientele
-- Habilitation BLP         │ OK          │ niveau_max=N2
-- MFA                      │ NON_REQUIS  │ Niveau N2 - MFA optionnel
-- Journalisation           │ OK          │ Attribution enregistree dans journal_audit
```

---

### Accès temporaire contrôlé

```sql
-- Accorder 720h (30 jours) à Anna Kowalski sur l'investigation ACPR
SELECT *
FROM accorder_acces_temporaire(
    'ER001',                                                   -- utilisateur
    'INV-REG-2024',                                            -- ressource
    720,                                                       -- durée en heures
    'Expertise droit tunisien requise pour investigation ACPR 2024',
    'AI002'                                                    -- admin qui autorise
);

-- Résultat :
-- etape                    │ statut │ detail
-- ─────────────────────────┼────────┼──────────────────────────────────────────────
-- Verification BLP admin   │ OK     │ Admin niveau N4 >= ressource niveau N4
-- Verification ABAC duree  │ OK     │ 720h <= max=720
-- Creation acces temporaire│ OK     │ Valide 720h - expire le 09/06/2026 14:30
-- Journalisation           │ OK     │ Acces temporaire enregistre dans journal_audit
```

---

### Révocation complète d'un utilisateur

```sql
-- Révoquer tous les accès de RC001 (départ entreprise)
SELECT *
FROM revoquer_acces_utilisateur(
    'RC001',
    'Depart entreprise - fin de contrat 10/05/2026',
    'AI002'
);

-- Résultat :
-- etape                │ statut │ detail
-- ─────────────────────┼────────┼──────────────────────────────
-- Revocation RBAC      │ OK     │ 4 role(s) revoques
-- Acces temporaires    │ OK     │ 0 acces temporaire(s) expires
-- Suspension compte    │ OK     │ Statut -> SUSPENDU
-- Role PostgreSQL      │ OK     │ rc001 -> NOLOGIN
-- Journalisation       │ OK     │ Revocation enregistree dans journal_audit
```

---

## 🔧 Guide de Personnalisation

### Ajouter une équipe

```sql
INSERT INTO equipes (nom, categorie, effectif_cible, niveau_max, description)
VALUES ('Innovation & Digital', 'EXPERTISE', 30, 'N3', 'Transformations IT, data science');
```

### Ajouter un rôle

```sql
INSERT INTO roles (nom_role, id_equipe, niveau_acces, description)
VALUES (
    'data_scientist',
    (SELECT id_equipe FROM equipes WHERE nom = 'Innovation & Digital'),
    'N3',
    'Analyses avancées, machine learning, données agrégées'
);
```

### Attribuer des permissions à un rôle

```sql
INSERT INTO role_permissions (id_role, id_permission)
SELECT
    (SELECT id_role FROM roles WHERE nom_role = 'data_scientist'),
    id_permission
FROM permissions
WHERE ressource_type IN ('DONNEES_CLIENT', 'ANALYSE_RISQUE')
  AND action IN ('LIRE', 'ECRIRE');
```

### Créer une politique ABAC

```sql
INSERT INTO abac_politiques
  (nom, id_role, type_ressource, filtre_pays, duree_max_heures,
   heure_debut, heure_fin, jours_semaine, description)
VALUES (
    'DS_analyse_horaires',
    (SELECT id_role FROM roles WHERE nom_role = 'data_scientist'),
    'ANALYSE_RISQUE',
    FALSE,
    NULL,
    '08:00', '20:00', 'LUN-VEN',
    'Data scientist : accès analyses en semaine uniquement'
);
```

### Accorder un accès temporaire

```sql
-- Via la fonction sécurisée (recommandé)
SELECT * FROM accorder_acces_temporaire(
    'DS001',           -- matricule utilisateur
    'RISQUE-Q4-24',    -- référence ressource
    240,               -- durée en heures (10 jours)
    'Projet analyse prédictive Q2 2026',
    'AI002'            -- matricule admin
);
```

---

## 📈 Considérations Production

### Performance

```sql
-- Index clés déjà en place
CREATE INDEX idx_ur_user_actif    ON utilisateur_roles(id_utilisateur, actif);
CREATE INDEX idx_rp_role_perm     ON role_permissions(id_role, id_permission);
CREATE INDEX idx_cw_user_groupe   ON chinese_wall_historique(id_utilisateur, groupe_conflict);
CREATE INDEX idx_journal_user_date ON journal_audit(id_utilisateur, horodatage DESC);
```

Pour les environnements à fort volume :
- Cache Redis sur les permissions fréquentes (TTL 5 min)
- Partitionnement de `journal_audit` par mois (`PARTITION BY RANGE`)
- Prepared statements obligatoires côté applicatif

### Sécurité

- **RLS** (Row Level Security) activé sur `utilisateurs`, `ressources`, `journal_audit`, `chinese_wall_historique`
- Rôles PostgreSQL hiérarchiques : `banque_employe` → `banque_rc`, `banque_aml`, etc.
- Export logs audit → SIEM (Splunk / ELK)
- Alertes automatiques sur refus répétés et violations Chinese Wall
- Intégration MFA externe : Okta / Azure AD recommandé

### Conformité

- Journalisation **COMPLÈTE** pour tous les accès N3/N4
- Archivage logs → S3/Glacier (rétention 7 ans ACPR)
- Rapports mensuels conformité générés via `v_refus_par_modele`
- Traçabilité complète des modifications de politiques dans `journal_audit`
- Révocation automatique des accès temporaires via trigger ou `pg_cron`

```sql
-- Planifier la révocation automatique toutes les 15 min (pg_cron)
SELECT cron.schedule(
    'revoc_acces_expires',
    '*/15 * * * *',
    $$UPDATE acces_temporaire
      SET    statut = 'EXPIRE', date_revocation = NOW()
      WHERE  statut = 'ACTIF'
        AND  date_fin <= NOW()
        AND  revocation_auto = TRUE$$
);
```

---

## 👥 Auteurs

**Achraf Hasnaoui** & **Mohamed Aziz Foudhaili**

Système de Contrôle d'Accès Bancaire Multi-Modèles — Mai 2026

---

*Dernière mise à jour : 10 mai 2026 — PostgreSQL 14+ — Status : ✅ Production-Ready*
