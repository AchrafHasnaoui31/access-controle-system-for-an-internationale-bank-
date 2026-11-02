# International Bank Access Control System

Database and security-modeling project for controlling access to sensitive areas and resources in an international banking context.

## Problem addressed

Banking environments need traceable access decisions, separation of duties, and models that make authorization rules auditable. This project explores that relational foundation through schema design, policy models, SQL queries, and simulation scenarios.

## Security concepts explored

- Role-based access control (RBAC).
- Bell-LaPadula confidentiality rules.
- Biba integrity rules.
- Attribute-based access control (ABAC).
- Chinese Wall conflict-of-interest constraints.
- SQL scenarios for positive and negative authorization decisions.

## Included material

- Database structure documentation.
- SQL schema and test queries.
- Simulation data and presentation material.
- A foundation for access history and audit review.

## Problems solved

- Makes authorization policies explicit instead of relying on undocumented assumptions.
- Provides testable scenarios for access decisions.
- Supports future least-privilege and auditability work.

## Roadmap

- Add an executable normalized PostgreSQL schema and migrations.
- Add immutable audit events and retention rules.
- Add policy tests and threat-model documentation.
- Add a small authenticated API for policy evaluation.
- Add negative tests for privilege escalation and policy conflicts.

## Status and scope

Academic security/database prototype. It is not a production banking security system and does not claim regulatory compliance.