# Multi-Tenant Hospital Management System (HMS SaaS)

> An implementation-ready, multi-tenant healthcare operations platform engineered with Laravel 11.x / 12.x (PHP 8.4+), Livewire 3, MySQL 8, and Redis 7.

[![PHP](https://img.shields.io/badge/PHP-8.4+-777BB4?style=for-the-badge&logo=php&logoColor=white)](https://php.net)
[![Laravel](https://img.shields.io/badge/Laravel-11.x%20%2F%2012.x-FF2D20?style=for-the-badge&logo=laravel&logoColor=white)](https://laravel.com)
[![Livewire](https://img.shields.io/badge/Livewire-3.x-FB70A9?style=for-the-badge&logo=livewire&logoColor=white)](https://livewire.laravel.com)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://www.mysql.com)
[![Redis](https://img.shields.io/badge/Redis-7.x-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io)
[![TailwindCSS](https://img.shields.io/badge/Tailwind-3.x-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white)](https://tailwindcss.com)

---

## 📖 Primary Documentation

The comprehensive, end-to-end specification is located in:

👉 **[Read the Full Product Requirements Document (PRD.md)](./PRD.md)** 👈

The PRD includes:
1. **Multi-Tenancy Engine**: Subdomain routing, tenant isolation middleware, and Eloquent `BelongsToTenant` global scoping.
2. **Frontend Architecture**: Component state and reactive workflows using **Laravel Blade + Livewire 3 + Alpine.js + Tailwind CSS**.
3. **Exhaustive Data Dictionary**: Complete schemas, column definitions, constraints, and indexes for all **32 database tables**.
4. **RBAC Matrix**: Granular permissions mapped across all 9 personas.
5. **Business Logic & State Machines**: Doctor slot scheduling (with Redis concurrency locks), Pharmacy FEFO inventory depletion, Inpatient bed state machine, and consolidated billing aggregator.
6. **REST API Specification**: Endpoints, authentication, request validation, and JSON envelopes.
7. **Auditability & Compliance**: HIPAA/GDPR baseline, append-only audit logging, and field-level encryption for PHI.

---

## High-Level Architecture

```
                         ┌─────────────────────────────────┐
                         │   Web / Mobile Browser Clients  │
                         └────────────────┬────────────────┘
                                          │ HTTPS (Subdomains / SSL)
                                          ▼
                         ┌─────────────────────────────────┐
                         │   Reverse Proxy / Load Balancer │
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
         │      MySQL 8     │   │      Redis 7     │   │  Object Storage  │
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

## Repository Structure

```
.
├── README.md               # Executive Summary & Repository Entrypoint
└── PRD.md                  # Comprehensive End-to-End PRD & Technical Spec
```

---

## Project Status

* **Current Stage**: Planning & Technical Specification Phase
* **Document Version**: 2.0.0 (Production-Ready)
* **Next Stage**: Implementation Phase 1 (Foundation & Tenancy Core)
