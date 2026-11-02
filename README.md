# International Bank Access Control System

> Academic database and security-modeling project for controlling access to sensitive banking areas and resources through explicit, testable authorization policies.

## Problem addressed

Banking environments need traceable access decisions, separation of duties, and policies that can be reviewed independently from application code. This project explores those foundations through relational modeling, SQL scenarios, and multiple access-control models.

## Security concepts explored

- **RBAC** — role-based access control.
- **Bell-LaPadula** — confidentiality-oriented information flow.
- **Biba** — integrity-oriented information flow.
- **ABAC** — decisions based on user, resource, action, and context attributes.
- **Chinese Wall** — conflict-of-interest constraints.

## Included material

- Database structure documentation.
- SQL schema and test queries.
- Simulation data and access scenarios.
- Presentation and design artifacts.
- A foundation for audit history and least-privilege review.

## Engineering value

The project makes authorization assumptions explicit and creates a basis for testing both allowed and denied access. That is more useful than treating security as a UI-level permission check: the policy should be represented in data, queries, and verifiable rules.

## Roadmap

- Add an executable normalized PostgreSQL schema and migrations.
- Add immutable audit events and retention policies.
- Add policy tests for privilege escalation and conflict cases.
- Add a small authenticated API for policy evaluation.
- Add threat modeling and evidence-based security documentation.

## Scope and status

Academic security/database prototype. It is not a production banking security system and does not claim regulatory compliance.

## License

No open-source license is currently specified.