# Multi-Tenant Hospital Management System (HMS SaaS)

> An implementation-ready, multi-tenant healthcare operations platform engineered with **Laravel 12 (PHP 8.4+)**, **Livewire 3**, **MySQL 8.0**, and **Redis 7.0**.

[![PHP](https://img.shields.io/badge/PHP-8.4+-777BB4?style=for-the-badge&logo=php&logoColor=white)](https://php.net)
[![Laravel](https://img.shields.io/badge/Laravel-12.x-FF2D20?style=for-the-badge&logo=laravel&logoColor=white)](https://laravel.com)
[![Livewire](https://img.shields.io/badge/Livewire-3.x-FB70A9?style=for-the-badge&logo=livewire&logoColor=white)](https://livewire.laravel.com)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://www.mysql.com)
[![Redis](https://img.shields.io/badge/Redis-7.x-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io)
[![TailwindCSS](https://img.shields.io/badge/Tailwind-3.x-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white)](https://tailwindcss.com)

---

## 📖 Primary Documentation

The comprehensive, end-to-end specification is located in:

👉 **[Read the Full Product Requirements Document (PRD.md)](./PRD.md)** 👈

The specification covers both the **Domain Architecture** and the **Engineering Execution Layer**:

### Core Architectural Pillars
1. **Multi-Tenancy Engine**: Subdomain routing (`mercy.medihms.test`), 3-tier defense-in-depth isolation, session context tracking, and Eloquent `BelongsToTenant` global scoping.
2. **Frontend Architecture**: Server-driven reactive component contracts using **Laravel Blade + Livewire 3 + Alpine.js + Tailwind CSS** (no complex decoupled SPA build step).
3. **Exhaustive Data Dictionary**: Complete schemas, column definitions, stored generated columns, foreign key cascades, and composite indexes for all **32 database tables**.
4. **Separation of Demographics & PHI**: Physical schema separation of non-sensitive `patients` data from `patient_clinical_profiles` (clinical notes, vitals, encrypted national ID).
5. **RBAC Matrix**: Granular permissions mapped across all 9 personas.
6. **Business Logic & State Machines**:
   * **Doctor Availability**: Overlapping range conflict verification with Redis distributed locking.
   * **Pharmacy FEFO Engine**: First-Expired, First-Out stock depletion with row-level pessimistic locking (`FOR UPDATE`).
   * **Inpatient Bed Management**: State machine (`AVAILABLE` ↔ `OCCUPIED` ↔ `RESERVED` ↔ `MAINTENANCE`) and automated daily room rate accrual.
   * **Consolidated Invoicing**: Real-time aggregation of consultation fees, lab orders, medicines, and bed stay charges.
7. **Security & Privacy Alignment**: HIPAA-aligned security architecture, dual-layer encryption (MySQL InnoDB TDE + application AES-256 casts), and GDPR-aligned right-to-erasure anonymization workflows.

### Engineering Execution Layer
8. **Automated Testing Strategy**: Concrete Pest PHP test specifications for **Tenant Isolation**, race-condition concurrency tests, payment idempotency, and Pest Architecture rules.
9. **Phase-by-Phase Definition of Done (DoD)**: Actionable 11-point acceptance checklist per implementation phase.
10. **CI/CD & DevOps Pipeline**: GitHub Actions workflow (Pint, PHPStan Level 8, Pest parallel test matrix with MySQL/Redis services).
11. **Disaster Recovery & Observability**: PITR binary logging (RPO < 5 min, RTO < 30 min), declaring **MySQL as single source of truth** and **Redis as disposable performance infrastructure**.

---

## High-Level System Architecture

```
                         ┌─────────────────────────────────┐
                         │   Web / Mobile Browser Clients  │
                         └────────────────┬────────────────┘
                                          │ HTTPS (Subdomains / Wildcard SSL)
                                          ▼
                         ┌─────────────────────────────────┐
                         │   Reverse Proxy / Load Balancer │
                         │ (Caddy On-Demand TLS/Cloudflare)│
                         └────────────────┬────────────────┘
                                          │
                   ┌──────────────────────┼──────────────────────┐
                   ▼                      ▼                      ▼
           ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
           │ Laravel App1 │       │ Laravel App2 │       │ Laravel AppN │
           └───────┬──────┘       └───────┬──────┘       └───────┬──────┘
                   │                      │                      │
                   ├──────────────────────┼──────────────────────┤
                   │                      │                      │
                   ▼                      ▼                      ▼
         ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
         │  MySQL 8 + TDE   │   │      Redis 7     │   │  Object Storage  │
         │ (Shared DB/Data) │   │ (Cache, Locks, Q)│   │ (S3 / MinIO PHI) │
         └──────────────────┘   └─────────┬────────┘   └──────────────────┘
                                          │
                                          ▼
                               ┌─────────────────────┐
                               │ Laravel Queue Worker│
                               └─────────────────────┘
```

---

## Key Modules Overview

| Module | Core Functionality | Primary Persona |
| :--- | :--- | :--- |
| **SaaS Platform Admin** | Tenant onboarding, subscription plans, platform telemetry, break-glass audit | Platform Super Admin |
| **Hospital Admin** | Hospital profile, department directory, doctor profiles, staff accounts | Hospital Admin |
| **Outpatient (OPD)** | Patient registration, doctor schedule slots, appointment booking | Receptionist |
| **Clinical Desk** | Split-screen consultation workspace, ICD-10 diagnoses, e-prescriptions | Doctor |
| **Inpatient (IPD)** | Ward, room, and bed management; admissions; transfers; discharge summaries | Nurse / Doctor |
| **Pharmacy & Inventory** | Medicine catalog, batch tracking, FEFO automated stock deduction, dispensing | Pharmacist |
| **Diagnostic Laboratory** | Lab test catalog, specimen tracking, test results entry, normal range flags | Lab Technician |
| **Billing & Accounting** | Itemized invoice generation (visits, lab, meds, beds), manual/Stripe receipts | Accountant / Cashier |
| **Security & Compliance** | Immutable audit log, AES-256 field encryption for PHI, RBAC policies | System |

---

## Pre-Configured Demo Accounts (Local / Staging)

All seed accounts use the default password: `Password123!`

| Context | Role | Email | Subdomain |
| :--- | :--- | :--- | :--- |
| **Platform** | Platform Super Admin | `admin@medihms.test` | `admin.medihms.test` |
| **Tenant 1** | Hospital Admin | `admin@mercy.test` | `mercy.medihms.test` |
| **Tenant 1** | Doctor (Cardiology) | `dr.smith@mercy.test` | `mercy.medihms.test` |
| **Tenant 1** | Receptionist | `reception@mercy.test` | `mercy.medihms.test` |
| **Tenant 1** | Pharmacist | `pharmacy@mercy.test` | `mercy.medihms.test` |
| **Tenant 1** | Lab Technician | `lab@mercy.test` | `mercy.medihms.test` |
| **Tenant 1** | Accountant / Cashier| `billing@mercy.test` | `mercy.medihms.test` |
| **Tenant 2** | Hospital Admin (Isolation Test) | `admin@cityclinic.test` | `cityclinic.medihms.test` |

---

## Local Development (Docker Setup)

```bash
# Clone the repository
git clone https://github.com/fazleyrabby/hospital-management-system.git
cd hospital-management-system

# One-command environment setup
make setup
```

The environment boots:
* **App (PHP 8.4-FPM + Nginx)**: `http://localhost:80`
* **MySQL 8.0**: `localhost:3306`
* **Redis 7.0**: `localhost:6379`
* **Mailpit (Local Email)**: `http://localhost:8025`

---

## Repository Structure

```
.
├── README.md               # Executive Summary & Repository Entrypoint
└── PRD.md                  # Comprehensive End-to-End PRD & Technical Spec (v2.2.0)
```

---

## Project Status

* **Current Stage**: Planning & Technical Specification Phase
* **Document Status**: Implementation-Ready MVP Blueprint
* **Document Version**: 2.2.0
* **Next Stage**: Scaffolding Phase 1 (Foundation & Multi-Tenancy Core)
