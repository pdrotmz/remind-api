# REMIND

REMIND is a backend engineering lab and MVP for coordinating emergency reports between users and organizations capable of responding to them.

The project is intentionally designed around a simple principle:

> **Complexity must be earned.**
>
> Architectural and infrastructure complexity should only be introduced when a concrete requirement, measurable constraint, or correctness problem justifies it.

## Overview

The core REMIND flow is straightforward:

1. An authenticated user reports an emergency.
2. The report is classified by an emergency category and severity.
3. Companies authorized for that category can discover the open report.
4. One eligible company assumes responsibility for the occurrence.
5. The report progresses through the response lifecycle until it is resolved or cancelled.
6. The occurrence remains available as historical information.

The MVP focuses on modeling this workflow correctly before introducing distributed infrastructure or advanced automation.

## MVP Actors

### User

A user can:

- manage their own profile;
- create emergency reports;
- view their own reports and history;
- change the severity of an active report;
- cancel an `OPEN` or `IN_PROGRESS` report.

### Company

A company can:

- manage its permitted profile information;
- operate in one or more emergency categories;
- discover compatible `OPEN` reports;
- assume an eligible report;
- resolve reports for which it is responsible;
- record response information;
- view its response history.

### Admin

An administrator manages users and companies. Administrative operations do not implicitly mean physical deletion of emergency history.

## Emergency Categories

The initial MVP categories are:

```text
MEDICAL
FIRE
CRIME
ACCIDENT
```

Categories describe the **emergency**, not the organization responding to it.

A company may handle multiple categories, and multiple companies may be eligible for the same category.

## Emergency Lifecycle

```text
                 +----------------+
                 |      OPEN      |
                 +----------------+
                    |          |
          Company   |          | User
           accepts  |          | cancels
                    v          v
            +---------------+  +-----------+
            | IN_PROGRESS   |  | CANCELLED |
            +---------------+  +-----------+
               |         |
       Company |         | User
      resolves |         | cancels
               v         v
        +----------+   +-----------+
        | RESOLVED |   | CANCELLED |
        +----------+   +-----------+
```

`RESOLVED` and `CANCELLED` are terminal states.

Severity and status are independent concepts: status represents where the report is in the response lifecycle, while severity represents how urgent the emergency currently is.

## Architecture

REMIND starts as a **Modular Monolith** organized using **Package by Feature**.

The application is deployed as a single unit, while business capabilities maintain explicit boundaries and should avoid unnecessary coupling.

A conceptual structure is:

```text
remind/
├── auth/
├── user/
├── company/
├── emergency/
└── shared/
```

The exact internal structure of each module is allowed to evolve with its real complexity. The project does not require every feature to contain identical layers, ports, adapters, or interfaces.

The `shared` package should contain genuinely shared concepts and must not become a generic dumping ground.

### Why a Modular Monolith?

For the MVP, there is no demonstrated requirement for independent deployment, distributed processing, independent scaling, or service-level isolation.

A modular monolith provides:

- low operational complexity;
- straightforward transactions;
- simple local development and deployment;
- explicit business boundaries;
- room for architectural evolution without prematurely paying distributed-system costs.

Microservices are therefore **not an MVP objective**. If a module eventually develops concrete requirements for independent scaling, deployment, isolation, or evolution, its extraction can be evaluated through a new Architecture Decision Record (ADR).

## Technology Stack

| Technology | Purpose |
| --- | --- |
| Java 21 | Language and runtime |
| Spring Boot | Application foundation |
| Maven | Build and dependency management |
| Spring Security | Authentication and authorization |
| JWT | API authentication tokens |
| PostgreSQL | Relational persistence |
| Flyway | Versioned database migrations |
| Testcontainers | Integration tests with real infrastructure |
| Package by Feature | Source-code organization |

## Domain Principles

Some important MVP invariants are:

- an emergency report always belongs to its authenticated author;
- only eligible companies may assume a report;
- only `OPEN` reports may be assumed;
- only one company may be responsible for a report at a time;
- only the assigned company may resolve an `IN_PROGRESS` report;
- only the author may change the severity or cancel their report;
- cancelling a report does not delete its history;
- closed reports cannot be reopened;
- authorization depends on both role and ownership/responsibility;
- infrastructure concerns must not define domain rules.

A particularly important concurrency invariant is that if multiple eligible companies attempt to assume the same `OPEN` report simultaneously, **only one operation may succeed**.

## Persistence Strategy

The MVP uses a single PostgreSQL database.

Sharing a database does not mean that modules have unrestricted access to each other's persistence models. Data ownership should remain aligned with module boundaries.

Database schema evolution is managed through Flyway migrations. Manual and untracked schema changes are not part of the intended workflow.

## Testing Strategy

The project prioritizes automated verification of business invariants and authority rules.

Testcontainers is used for integration tests that require PostgreSQL, reducing differences between the test environment and the real persistence technology.

The testing strategy should cover, among other scenarios:

- valid and invalid lifecycle transitions;
- ownership and responsibility rules;
- category eligibility;
- persistence constraints;
- concurrent attempts to assume the same emergency report;
- authentication and authorization boundaries.

## Explicitly Outside the MVP

The following technologies and capabilities are intentionally excluded until a demonstrated requirement justifies them:

- microservices;
- Kafka or another distributed message broker;
- Redis or distributed caching;
- AI/LLM integrations;
- automatic emergency classification;
- automatic geolocation-based dispatch;
- fleet, vehicle, or physical-team management;
- automatic routing;
- unnecessary emergency subcategories.

Their absence is intentional rather than a limitation accidentally left unresolved.

## Documentation

The project documentation includes:

```text
docs/
├── functional-requirements
├── non-functional-requirements
├── entity-relational-model
├── domain-model
└── adr/
    └── ADR-001-modular-monolith
```

`ADR-001` establishes the Modular Monolith as the architectural baseline for the MVP.

Future significant architectural decisions should be documented as additional ADRs instead of silently changing the architecture.

## Current Status

**Status: Initial implementation / MVP foundation.**

The problem, initial domain, functional requirements, non-functional requirements, preliminary relational model, domain model, technology baseline, and initial architecture have been defined.

Implementation is beginning from this baseline, and decisions intentionally left open will be resolved when the domain or implementation provides enough evidence to justify them.

## Engineering Philosophy

REMIND is also a learning project focused on backend and software engineering practices.

The goal is not to maximize the number of technologies used. The goal is to understand **why each decision exists**, what problem it solves, and what trade-offs it introduces.

> Build the simplest architecture that protects the requirements we actually have.