# Comprehensive Product Requirements Document (PRD) & Technical Specification

## Multi-Tenant Hospital Management System (HMS)

* **Backend Framework**: Laravel 11 (PHP 8.3+)
* **Database**: MySQL 8.0+
* **Cache & Queues**: Redis 7.0+
* **Frontend Architecture**: Laravel Blade + Livewire 3 + Alpine.js + Tailwind CSS
* **Product Type**: Multi-Tenant B2B Healthcare SaaS Platform
* **Document Status**: Production-Ready Technical Specification
* **Version**: 2.0.0 (End-to-End Complete)

---

# Table of Contents
1. [Product Overview & Architectural Philosophy](#1-product-overview--architectural-philosophy)
2. [Multi-Tenancy Engine & Isolation Architecture](#2-multi-tenancy-engine--isolation-architecture)
3. [Frontend Architecture: Blade + Livewire 3](#3-frontend-architecture-blade--livewire-3)
4. [Exhaustive Database Schema & Data Dictionary](#4-exhaustive-database-schema--data-dictionary)
5. [Role-Based Access Control (RBAC) Matrix](#5-role-based-access-control-rbac-matrix)
6. [Core Business Engines & State Machines](#6-core-business-engines--state-machines)
7. [Livewire Component Specifications](#7-livewire-component-specifications)
8. [REST API Specification](#8-rest-api-specification)
9. [Background Jobs, Queues & Notifications](#9-background-jobs-queues--notifications)
10. [Audit Logging, PHI Security & Compliance](#10-audit-logging-phi-security--compliance)
11. [Decisions on Previous Open Questions](#11-decisions-on-previous-open-questions)
12. [Scaffolding & Implementation Sequence](#12-scaffolding--implementation-sequence)

---

# 1. Product Overview & Architectural Philosophy

The system is a production-grade, multi-tenant hospital management software platform engineered as a modular monolith in Laravel 11. It allows hundreds of independent hospitals, clinics, and medical centers to operate autonomously on a single unified infrastructure while enforcing absolute data isolation, strict regulatory compliance (HIPAA/GDPR security guidelines), and high-throughput operational efficiency.

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

### Core Design Tenets
1. **Security & Data Isolation Before Everything**: Tenant leakage is a catastrophic failure. Isolation is enforced at the database query level via Eloquent Global Scopes, composite unique keys, and route-model authorization policies.
2. **Monolithic Simplicity with Livewire 3**: No complex decoupled SPA overhead. High-fidelity, real-time reactive user interfaces are implemented using Laravel Blade, Livewire 3, Alpine.js, and Tailwind CSS.
3. **Database as Single Source of Truth**: MySQL enforces relational integrity, foreign key constraints, and transactional consistency. Redis serves purely as a performance accelerator (caching, distributed lock coordination, rate-limiting, and queue broker).
4. **Auditability & Accountability**: Every action touching Protected Health Information (PHI) or financial balances writes an immutable, append-only record to the `audit_logs` table.

---

# 2. Multi-Tenancy Engine & Isolation Architecture

### 2.1 Tenancy Strategy: Shared Database / Shared Schema
To maximize resource efficiency, minimize operational maintenance, and simplify cross-tenant migrations, the system utilizes a **Shared Database with Tenant Discriminator Columns (`tenant_id`)**.

Every tenant-scoped table contains an unsigned foreign key `tenant_id` indexed with foreign key cascading constraints to the `tenants` table.

### 2.2 Tenant Identification & Resolution Pipeline
Incoming HTTP requests pass through the following strict resolution middleware pipeline:

```
[ Incoming Request: tenant1.medihms.test/appointments ]
                      │
                      ▼
[ Step 1: Subdomain Detection Middleware (IdentifyTenantBySubdomain) ]
  - Parse host: extract "tenant1"
  - Query Tenant::where('subdomain', 'tenant1')->where('status', 'ACTIVE')->firstOrFail()
  - If root domain or "admin", mark request as Platform Context.
                      │
                      ▼
[ Step 2: Set Tenant Context Singleton ]
  - Bind resolved tenant into app(TenantManager::class)->setTenant($tenant)
  - Configure dynamic runtime settings (timezone, currency, default branding)
                      │
                      ▼
[ Step 3: Authenticate User & Validate Membership ]
  - Verify auth()->user()->tenant_id === $tenant->id
  - Reject access with HTTP 403 if user does not belong to the resolved tenant
                      │
                      ▼
[ Step 4: Apply Eloquent Global Scope ]
  - TenantScope intercepts all Eloquent queries: $query->where('tenant_id', $tenant->id)
  - Automatically populate $model->tenant_id = $tenant->id on model creation
```

### 2.3 `BelongsToTenant` Model Trait Implementation
Every model owned by a hospital must utilize the `App\Models\Concerns\BelongsToTenant` trait:

* **Global Scope**: Automatically adds `builder->where($table . '.tenant_id', app(TenantManager::class)->getTenantId())`.
* **Creation Hook**: In `static::creating()`, if `tenant_id` is not explicitly set, it automatically injects `app(TenantManager::class)->getTenantId()`.
* **Foreign Key Protection**: Throws a `SecurityException` if an attempt is made to update `tenant_id` after creation.

### 2.4 Super Admin Context Switching
Platform super administrators (`users.tenant_id = NULL` with `role = platform_admin`) have isolated access to the SaaS Control Panel (`admin.medihms.test`).
* Super admins can impersonate a tenant for support purposes.
* Impersonation generates an explicit audit log entry (`action: SUPER_ADMIN_IMPERSONATE_START`).
* Access to clinical records (PHI) during impersonation is masked unless an explicit break-glass reason is entered and logged.

---

# 3. Frontend Architecture: Blade + Livewire 3

The user interface avoids the complexity and deployment friction of decoupled SPAs (Vue/React/Next.js) by using **Laravel Blade + Livewire 3 + Alpine.js + Tailwind CSS**.

### 3.1 Stack Composition
* **Livewire 3**: Handles reactive UI state, real-time input validation, dynamic form fields, asynchronous pagination, modals, and event emission.
* **Alpine.js**: Handles lightweight client-side interactions (dropdowns, off-canvas drawers, tab switches, tooltips, local date formatters) without making roundtrips to the server.
* **Tailwind CSS**: Modern, utility-first design system customized with hospital color palettes (slate, clinical teal, emerald, amber, and crimson).
* **WireUI / Lucide Icons**: High-contrast, clean medical icons for clinical clarity.

### 3.2 Layout Hierarchy
1. `layouts.app`: Main authenticated wrapper for hospital staff.
   * Responsive collapsible sidebar with role-aware navigation links.
   * Top navigation bar with global patient quick-search (`Cmd+K`), active tenant indicator, notification center, and user profile drawer.
   * Main content area with breadcrumbs, page actions, and flash notification toasts.
2. `layouts.admin`: Dedicated platform administration layout for managing SaaS tenants, billing, and system health.
3. `layouts.guest`: Clean, centered layout for authentication (login, 2FA challenge, password reset).
4. `layouts.print`: Minimalist, print-optimized stylesheet layout for patient prescriptions, diagnostic reports, lab slips, and billing invoices.

---

# 4. Exhaustive Database Schema & Data Dictionary

All tables are created using the `utf8mb4_unicode_ci` character set on MySQL 8.0+. Primary keys are `BIGINT UNSIGNED AUTO_INCREMENT` (with corresponding UUIDs for external API exposure).

---

### 4.1 Tenancy & Platform Core

#### `tenants`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Internal unique identifier |
| `uuid` | CHAR(36) | UNIQUE | No | | External UUIDv4 identifier |
| `name` | VARCHAR(191) | | No | | Legal hospital/organization name |
| `subdomain` | VARCHAR(64) | UNIQUE | No | | Unique subdomain (e.g. `mercy-general`) |
| `custom_domain` | VARCHAR(191) | UNIQUE | Yes | NULL | Optional custom CNAME domain |
| `email` | VARCHAR(191) | | No | | Hospital contact email |
| `phone` | VARCHAR(32) | | No | | Hospital emergency/reception phone |
| `address` | TEXT | | Yes | NULL | Physical address |
| `timezone` | VARCHAR(64) | | No | 'UTC' | Hospital timezone for scheduling |
| `currency_code` | CHAR(3) | | No | 'USD' | 3-letter ISO currency code |
| `status` | ENUM | 'ACTIVE','SUSPENDED','INACTIVE' | No | 'ACTIVE' | Tenant operational status |
| `subscription_plan`| VARCHAR(64) | | No | 'STANDARD' | SaaS tier (BASIC, STANDARD, ENTERPRISE) |
| `logo_path` | VARCHAR(255) | | Yes | NULL | Storage path for hospital brand logo |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Record creation timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Record update timestamp |

#### `tenant_settings`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `key` | VARCHAR(64) | | No | | Configuration key name |
| `value` | JSON | | Yes | NULL | Serialized setting value |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `key`)*

#### `users`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Internal user ID |
| `uuid` | CHAR(36) | UNIQUE | No | | External UUIDv4 |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | Yes | NULL | NULL for platform admins |
| `name` | VARCHAR(191) | | No | | Full legal name |
| `email` | VARCHAR(191) | | No | | Email address (unique per tenant) |
| `password` | VARCHAR(255) | | No | | Bcrypt hashed password |
| `phone` | VARCHAR(32) | | Yes | NULL | Contact phone number |
| `status` | ENUM | 'ACTIVE','INVITED','DEACTIVATED'| No | 'ACTIVE' | User account state |
| `email_verified_at`| TIMESTAMP | | Yes | NULL | Email verification time |
| `two_factor_secret`| TEXT | | Yes | NULL | 2FA TOTP secret |
| `last_login_at` | TIMESTAMP | | Yes | NULL | Last successful authentication |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `email`)*

#### `roles` & `permissions` (RBAC)
* `roles`: `id`, `tenant_id` (NULL for system roles), `name`, `guard_name`, `description`, timestamps. *UNIQUE(`tenant_id`, `name`)*.
* `permissions`: `id`, `name`, `guard_name`, `group_name`, timestamps. *UNIQUE(`name`, `guard_name`)*.
* `model_has_roles`: `role_id`, `model_type`, `model_id`, `tenant_id`.
* `role_has_permissions`: `permission_id`, `role_id`.

---

### 4.2 Hospital Structure & Clinical Staff

#### `departments`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Department ID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Owning tenant |
| `name` | VARCHAR(191) | | No | | e.g., 'Cardiology', 'Pediatrics' |
| `code` | VARCHAR(32) | | No | | Short code, e.g. 'CARD' |
| `description` | TEXT | | Yes | NULL | Scope of department |
| `head_doctor_id` | BIGINT UNSIGNED | | Yes | NULL | Reference to head doctor |
| `status` | ENUM | 'ACTIVE','INACTIVE' | No | 'ACTIVE' | Operational status |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `code`)*

#### `doctors`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Doctor profile ID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Owning tenant |
| `user_id` | BIGINT UNSIGNED | FK -> users(id) ON DELETE CASCADE | No | | User login account |
| `department_id` | BIGINT UNSIGNED | FK -> departments(id) ON DELETE RESTRICT | No | | Primary clinical department |
| `license_number` | VARCHAR(64) | | No | | Medical council license number |
| `specialization` | VARCHAR(191) | | No | | Clinical specialty |
| `qualification` | VARCHAR(191) | | No | | e.g. 'MD, MBBS, FACS' |
| `consultation_fee`| DECIMAL(10,2) | | No | 0.00 | Standard outpatient fee |
| `slot_duration_mins`| SMALLINT UNSIGNED| | No | 15 | Default appointment duration |
| `bio` | TEXT | | Yes | NULL | Doctor biographical summary |
| `status` | ENUM | 'ACTIVE','ON_LEAVE','RESIGNED' | No | 'ACTIVE' | Employment status |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `user_id`), UNIQUE(`tenant_id`, `license_number`)*

#### `doctor_schedules` (Weekly Recurring Availability)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Schedule ID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Owning tenant |
| `doctor_id` | BIGINT UNSIGNED | FK -> doctors(id) ON DELETE CASCADE | No | | Doctor reference |
| `day_of_week` | TINYINT UNSIGNED | 0 (Sun) - 6 (Sat) | No | | Day of week |
| `start_time` | TIME | | No | | Consultation start time |
| `end_time` | TIME | | No | | Consultation end time |
| `break_start_time`| TIME | | Yes | NULL | Optional break start |
| `break_end_time` | TIME | | Yes | NULL | Optional break end |
| `max_patients` | SMALLINT UNSIGNED| | Yes | NULL | Patient booking ceiling |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: INDEX(`tenant_id`, `doctor_id`, `day_of_week`)*

#### `doctor_leaves` (Exceptions & Holidays)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Owning tenant |
| `doctor_id` | BIGINT UNSIGNED | FK -> doctors(id) ON DELETE CASCADE | No | | Doctor reference |
| `start_date` | DATE | | No | | Leave start date |
| `end_date` | DATE | | No | | Leave end date |
| `reason` | VARCHAR(255) | | Yes | NULL | Reason for absence |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |

---

### 4.3 Patients & Clinical History

#### `patients`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | External UUIDv4 |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Owning tenant |
| `patient_number`| VARCHAR(64) | | No | | Human readable (e.g. `HOSP-2026-00012`) |
| `first_name` | VARCHAR(100) | | No | | First name |
| `last_name` | VARCHAR(100) | | No | | Last/family name |
| `date_of_birth` | DATE | | No | | Birth date |
| `gender` | ENUM | 'MALE','FEMALE','OTHER' | No | | Biological gender |
| `blood_group` | ENUM | 'A+','A-','B+','B-','AB+','AB-','O+','O-','UNKNOWN' | No | 'UNKNOWN' | Blood type |
| `phone` | VARCHAR(32) | | No | | Contact telephone |
| `email` | VARCHAR(191) | | Yes | NULL | Optional contact email |
| `address` | TEXT | | Yes | NULL | Residential address |
| `national_id` | VARCHAR(64) | Encrypted at rest | Yes | NULL | SSN / National Identity |
| `emergency_contact_name` | VARCHAR(191) | | Yes | NULL | Kin contact name |
| `emergency_contact_phone`| VARCHAR(32) | | Yes | NULL | Kin contact number |
| `allergies` | TEXT | | Yes | NULL | Known drug/food allergies |
| `chronic_conditions` | TEXT | | Yes | NULL | Ongoing conditions |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Registration timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Profile update timestamp |
*Indexes: UNIQUE(`tenant_id`, `patient_number`), INDEX(`tenant_id`, `phone`), INDEX(`tenant_id`, `first_name`, `last_name`)*

#### `appointments`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Patient being seen |
| `doctor_id` | BIGINT UNSIGNED | FK -> doctors(id) ON DELETE RESTRICT | No | | Doctor providing consultation |
| `department_id` | BIGINT UNSIGNED | FK -> departments(id) ON DELETE RESTRICT | No | | Clinical department |
| `appointment_date`| DATE | | No | | Scheduled date |
| `start_time` | TIME | | No | | Slot start time |
| `end_time` | TIME | | No | | Slot end time |
| `type` | ENUM | 'OUTPATIENT','FOLLOW_UP','EMERGENCY'| No | 'OUTPATIENT' | Category of visit |
| `status` | ENUM | 'SCHEDULED','CONFIRMED','CHECKED_IN','IN_PROGRESS','COMPLETED','CANCELLED','NO_SHOW' | No | 'SCHEDULED' | Appointment lifecycle |
| `chief_complaint` | TEXT | | Yes | NULL | Initial symptom description |
| `cancellation_reason`| VARCHAR(255)| | Yes | NULL | Reason if cancelled |
| `created_by_user_id`| BIGINT UNSIGNED | FK -> users(id) | No | | Staff member who scheduled |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: INDEX(`tenant_id`, `doctor_id`, `appointment_date`, `start_time`), INDEX(`tenant_id`, `patient_id`, `appointment_date`)*

#### `visits` (Clinical Encounters)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Encounter ID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Patient |
| `doctor_id` | BIGINT UNSIGNED | FK -> doctors(id) ON DELETE RESTRICT | No | | Attending doctor |
| `appointment_id` | BIGINT UNSIGNED | FK -> appointments(id) ON DELETE SET NULL | Yes | NULL | Linked appointment |
| `encounter_date` | DATETIME | | No | CURRENT_TIMESTAMP | Time consultation began |
| `visit_type` | ENUM | 'OPD','IPD','EMERGENCY' | No | 'OPD' | Encounter context |
| `clinical_notes` | LONGTEXT | | Yes | NULL | Doctor's subjective/objective notes |
| `examination` | LONGTEXT | | Yes | NULL | Physical exam findings |
| `treatment_plan` | LONGTEXT | | Yes | NULL | Prescribed therapeutic plan |
| `status` | ENUM | 'OPEN','COMPLETED','DISCHARGED' | No | 'OPEN' | Encounter status |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: INDEX(`tenant_id`, `patient_id`), INDEX(`tenant_id`, `encounter_date`)*

#### `vitals`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `visit_id` | BIGINT UNSIGNED | FK -> visits(id) ON DELETE CASCADE | No | | Linked clinical visit |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Patient |
| `systolic_bp` | SMALLINT UNSIGNED| | Yes | NULL | mmHg (e.g. 120) |
| `diastolic_bp` | SMALLINT UNSIGNED| | Yes | NULL | mmHg (e.g. 80) |
| `heart_rate` | SMALLINT UNSIGNED| | Yes | NULL | Beats per minute |
| `respiratory_rate`| SMALLINT UNSIGNED| | Yes | NULL | Breaths per minute |
| `temperature_c` | DECIMAL(4,2) | | Yes | NULL | Celsius |
| `oxygen_saturation`| DECIMAL(4,1) | | Yes | NULL | SpO2 percentage (e.g. 98.5) |
| `weight_kg` | DECIMAL(5,2) | | Yes | NULL | Body weight in kilograms |
| `height_cm` | DECIMAL(5,2) | | Yes | NULL | Height in centimeters |
| `bmi` | DECIMAL(4,1) | | Yes | NULL | Body Mass Index (calculated) |
| `recorded_by_user_id`| BIGINT UNSIGNED | FK -> users(id) | No | | Staff / Nurse recorder |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Measurement timestamp |

#### `diagnoses`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `visit_id` | BIGINT UNSIGNED | FK -> visits(id) ON DELETE CASCADE | No | | Linked clinical visit |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Patient |
| `icd10_code` | VARCHAR(16) | | Yes | NULL | Standard ICD-10 code (e.g. 'I10') |
| `diagnosis_name`| VARCHAR(255) | | No | | Diagnosis title |
| `diagnosis_type`| ENUM | 'PROVISIONAL','FINAL','DIFFERENTIAL'| No | 'FINAL' | Classification |
| `comments` | TEXT | | Yes | NULL | Specific remarks |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |

---

### 4.4 Pharmacy & Inventory

#### `medicine_categories`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `name` | VARCHAR(100) | | No | | e.g., 'Antibiotics', 'Analgesics' |
| `description` | TEXT | | Yes | NULL | Category description |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `name`)*

#### `medicines`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `category_id` | BIGINT UNSIGNED | FK -> medicine_categories(id) | No | | Pharmacology classification |
| `name` | VARCHAR(191) | | No | | Brand/commercial name |
| `generic_name` | VARCHAR(191) | | No | | Chemical/generic substance |
| `dosage_form` | ENUM | 'TABLET','CAPSULE','SYRUP','INJECTION','OINTMENT','DROPS','INHALER' | No | 'TABLET' | Formulation |
| `strength` | VARCHAR(64) | | No | | e.g. '500mg', '10mg/ml' |
| `unit` | VARCHAR(32) | | No | 'Pill' | Dispensing unit |
| `unit_price` | DECIMAL(10,2) | | No | 0.00 | Retail price per unit |
| `cost_price` | DECIMAL(10,2) | | No | 0.00 | Acquisition cost per unit |
| `reorder_level` | INT UNSIGNED | | No | 100 | Low-stock alarm threshold |
| `current_stock` | INT | | No | 0 | Cached total quantity in stock |
| `status` | ENUM | 'ACTIVE','DISCONTINUED' | No | 'ACTIVE' | Availability status |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `name`, `strength`), INDEX(`tenant_id`, `current_stock`)*

#### `medicine_batches`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Batch record ID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `medicine_id` | BIGINT UNSIGNED | FK -> medicines(id) ON DELETE CASCADE | No | | Linked medicine |
| `batch_number` | VARCHAR(64) | | No | | Manufacturer batch ID |
| `expiry_date` | DATE | | No | | Expiration threshold date |
| `quantity` | INT | | No | | Remaining stock in this batch |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Inward receipt timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Balance update timestamp |
*Indexes: UNIQUE(`tenant_id`, `medicine_id`, `batch_number`), INDEX(`tenant_id`, `expiry_date`)*

#### `prescriptions`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `visit_id` | BIGINT UNSIGNED | FK -> visits(id) ON DELETE CASCADE | No | | Clinical visit |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Patient |
| `doctor_id` | BIGINT UNSIGNED | FK -> doctors(id) ON DELETE RESTRICT | No | | Prescribing doctor |
| `status` | ENUM | 'PENDING','DISPENSED','PARTIALLY_DISPENSED','CANCELLED' | No | 'PENDING' | Dispensation state |
| `notes` | TEXT | | Yes | NULL | General pharmacist instructions |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Prescription time |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Dispensation time |

#### `prescription_items`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `prescription_id`| BIGINT UNSIGNED| FK -> prescriptions(id) ON DELETE CASCADE | No | | Parent prescription |
| `medicine_id` | BIGINT UNSIGNED | FK -> medicines(id) ON DELETE RESTRICT | No | | Medicine |
| `dosage` | VARCHAR(64) | | No | | e.g., '1 Tablet' |
| `frequency` | VARCHAR(64) | | No | | e.g., 'TDS (3 times a day)' |
| `duration_days` | SMALLINT UNSIGNED| | No | | e.g., 5 |
| `quantity` | INT UNSIGNED | | No | | Total units to dispense |
| `instructions` | VARCHAR(255) | | Yes | NULL | e.g., 'After food' |
| `dispensed_quantity`| INT UNSIGNED | | No | 0 | Units handed to patient |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Creation timestamp |

#### `inventory_transactions` (Immutable Audit Ledger)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `medicine_id` | BIGINT UNSIGNED | FK -> medicines(id) ON DELETE RESTRICT | No | | Medicine |
| `batch_id` | BIGINT UNSIGNED | FK -> medicine_batches(id) | Yes | NULL | Specific batch |
| `transaction_type`| ENUM | 'PURCHASE_RECEIPT','DISPENSE','RETURN','ADJUSTMENT_LOSS','EXPIRED' | No | | Movement reason |
| `quantity` | INT | Positive (In) / Negative (Out) | No | | Stock delta |
| `balance_after` | INT | | No | | Running stock balance |
| `reference_type`| VARCHAR(64) | | Yes | NULL | e.g. 'Prescription', 'PO' |
| `reference_id` | BIGINT UNSIGNED | | Yes | NULL | Linked entity ID |
| `performed_by_user_id`| BIGINT UNSIGNED| FK -> users(id) | No | | Pharmacist/Admin |
| `remarks` | VARCHAR(255) | | Yes | NULL | Justification for movement |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |

---

### 4.5 Diagnostic Laboratory

#### `lab_test_types` (Test Catalog)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `code` | VARCHAR(32) | | No | | e.g. 'CBC', 'LIPID' |
| `name` | VARCHAR(191) | | No | | Complete Blood Count |
| `category` | VARCHAR(64) | | No | 'HAEMATOLOGY'| Lab section |
| `sample_type` | VARCHAR(64) | | No | 'Whole Blood' | Specimen required |
| `price` | DECIMAL(10,2) | | No | 0.00 | Standard charge |
| `reference_range`| TEXT | | Yes | NULL | Normal biological intervals |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `code`)*

#### `lab_orders`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `visit_id` | BIGINT UNSIGNED | FK -> visits(id) ON DELETE CASCADE | No | | Clinical visit |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Patient |
| `doctor_id` | BIGINT UNSIGNED | FK -> doctors(id) ON DELETE RESTRICT | No | | Requesting physician |
| `test_type_id` | BIGINT UNSIGNED | FK -> lab_test_types(id) | No | | Ordered investigation |
| `order_number` | VARCHAR(64) | | No | | Human readable barcode |
| `status` | ENUM | 'ORDERED','SAMPLE_COLLECTED','PROCESSING','COMPLETED','CANCELLED' | No | 'ORDERED' | Operational lifecycle |
| `clinical_notes` | TEXT | | Yes | NULL | Indication for testing |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Ordering timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Lifecycle update |
*Indexes: UNIQUE(`tenant_id`, `order_number`), INDEX(`tenant_id`, `status`)*

#### `lab_results`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `lab_order_id` | BIGINT UNSIGNED | FK -> lab_orders(id) ON DELETE CASCADE | No | | Linked order |
| `result_data` | JSON | | No | | Structured parameters & values |
| `summary` | TEXT | | Yes | NULL | Pathologist interpretation |
| `is_abnormal` | BOOLEAN | | No | FALSE | Critical flag indicator |
| `file_attachment_path`| VARCHAR(255)| | Yes | NULL | S3/MinIO PDF report path |
| `verified_by_user_id`| BIGINT UNSIGNED| FK -> users(id) | Yes | NULL | Lab supervisor/pathologist |
| `verified_at` | TIMESTAMP | | Yes | NULL | Authorization timestamp |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Result entry timestamp |

---

### 4.6 Inpatient (IPD) & Bed Management

#### `wards`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `name` | VARCHAR(100) | | No | | e.g. 'ICU', 'Maternity Ward' |
| `gender` | ENUM | 'MALE','FEMALE','MIXED' | No | 'MIXED' | Occupancy segregation |
| `capacity` | SMALLINT UNSIGNED| | No | 0 | Total beds |
| `status` | ENUM | 'ACTIVE','CLOSED' | No | 'ACTIVE' | Ward status |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `name`)*

#### `rooms`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `ward_id` | BIGINT UNSIGNED | FK -> wards(id) ON DELETE CASCADE | No | | Parent ward |
| `room_number` | VARCHAR(32) | | No | | e.g., '101-A' |
| `type` | ENUM | 'GENERAL','SEMI_PRIVATE','PRIVATE','ISOLATION','ICU' | No | 'GENERAL' | Room comfort tier |
| `rate_per_day` | DECIMAL(10,2) | | No | 0.00 | Daily billing rate |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `room_number`)*

#### `beds`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `room_id` | BIGINT UNSIGNED | FK -> rooms(id) ON DELETE CASCADE | No | | Parent room |
| `bed_number` | VARCHAR(32) | | No | | e.g., 'BED-01' |
| `status` | ENUM | 'AVAILABLE','OCCUPIED','MAINTENANCE','RESERVED' | No | 'AVAILABLE' | Real-time occupancy state |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `room_id`, `bed_number`), INDEX(`tenant_id`, `status`)*

#### `admissions`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `admission_number`| VARCHAR(64) | | No | | e.g. `ADM-2026-0045` |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Inpatient |
| `attending_doctor_id`| BIGINT UNSIGNED| FK -> doctors(id) ON DELETE RESTRICT | No | | Physician in charge |
| `bed_id` | BIGINT UNSIGNED | FK -> beds(id) ON DELETE RESTRICT | No | | Currently occupied bed |
| `admission_date` | DATETIME | | No | CURRENT_TIMESTAMP | Entry time |
| `discharge_date` | DATETIME | | Yes | NULL | Release time |
| `admission_reason`| TEXT | | No | | Clinical reason for admission |
| `discharge_summary`| LONGTEXT | | Yes | NULL | Final medical discharge summary |
| `status` | ENUM | 'ADMITTED','DISCHARGED','TRANSFERRED'| No | 'ADMITTED' | Lifecycle state |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `admission_number`), INDEX(`tenant_id`, `patient_id`, `status`)*

---

### 4.7 Billing, Invoicing & Payments

#### `invoices`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `invoice_number`| VARCHAR(64) | | No | | e.g., `INV-2026-00194` |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Billed patient |
| `visit_id` | BIGINT UNSIGNED | FK -> visits(id) ON DELETE SET NULL | Yes | NULL | Linked encounter |
| `admission_id` | BIGINT UNSIGNED | FK -> admissions(id) ON DELETE SET NULL | Yes | NULL | Linked hospital stay |
| `subtotal` | DECIMAL(12,2) | | No | 0.00 | Sum of items before deductions |
| `discount_amount`| DECIMAL(12,2)| | No | 0.00 | Concessions / discounts |
| `tax_amount` | DECIMAL(12,2) | | No | 0.00 | Applicable VAT/Sales tax |
| `total_amount` | DECIMAL(12,2) | | No | 0.00 | Final payable sum |
| `paid_amount` | DECIMAL(12,2) | | No | 0.00 | Aggregated receipts |
| `balance_due` | DECIMAL(12,2) | | No | 0.00 | Outstanding liability |
| `status` | ENUM | 'DRAFT','ISSUED','PARTIALLY_PAID','PAID','CANCELLED','REFUNDED' | No | 'ISSUED' | Financial state |
| `due_date` | DATE | | Yes | NULL | Payment deadline |
| `created_by_user_id`| BIGINT UNSIGNED| FK -> users(id) | No | | Billing clerk |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Invoice issuance date |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Ledger update |
*Indexes: UNIQUE(`tenant_id`, `invoice_number`), INDEX(`tenant_id`, `patient_id`, `status`)*

#### `invoice_items`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `invoice_id` | BIGINT UNSIGNED | FK -> invoices(id) ON DELETE CASCADE | No | | Parent invoice |
| `item_type` | ENUM | 'CONSULTATION','MEDICINE','LAB_TEST','BED_CHARGE','PROCEDURE','OTHER' | No | | Charge classification |
| `description` | VARCHAR(255) | | No | | Line item label |
| `quantity` | DECIMAL(8,2) | | No | 1.00 | Units billed |
| `unit_price` | DECIMAL(10,2) | | No | 0.00 | Price per unit |
| `total_price` | DECIMAL(12,2) | | No | 0.00 | quantity * unit_price |
| `reference_type`| VARCHAR(64) | | Yes | NULL | e.g., 'App\Models\LabOrder' |
| `reference_id` | BIGINT UNSIGNED | | Yes | NULL | Polymorphic reference ID |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |

#### `payments` (Receipts)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public receipt UUID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `payment_number`| VARCHAR(64) | | No | | e.g. `REC-2026-00088` |
| `invoice_id` | BIGINT UNSIGNED | FK -> invoices(id) ON DELETE RESTRICT | No | | Target invoice |
| `amount` | DECIMAL(12,2) | | No | 0.00 | Monetary amount collected |
| `payment_method`| ENUM | 'CASH','CREDIT_CARD','DEBIT_CARD','BANK_TRANSFER','STRIPE','MOBILE_PAYMENT' | No | 'CASH' | Payment vehicle |
| `transaction_reference`| VARCHAR(128)| | Yes | NULL | External bank / Stripe transaction ID |
| `idempotency_key`| VARCHAR(128)| UNIQUE | Yes | NULL | Prevents duplicate charge submission |
| `received_by_user_id`| BIGINT UNSIGNED| FK -> users(id) | No | | Cashier / Collector |
| `payment_date` | DATETIME | | No | CURRENT_TIMESTAMP | Time of collection |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `payment_number`), INDEX(`tenant_id`, `invoice_id`)*

---

### 4.8 Audit Logs & Security

#### `audit_logs` (Append-Only Event Log)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Log ID |
| `tenant_id` | BIGINT UNSIGNED | | Yes | NULL | Tenant context (NULL for platform operations) |
| `user_id` | BIGINT UNSIGNED | | Yes | NULL | User who triggered the action |
| `action` | VARCHAR(64) | | No | | e.g., 'VIEW_PHI', 'CREATE_PATIENT', 'DISPENSE_MEDICINE' |
| `entity_type` | VARCHAR(100) | | No | | Target model class (e.g. `App\Models\Patient`) |
| `entity_id` | BIGINT UNSIGNED | | Yes | NULL | Target model primary key |
| `old_values` | JSON | | Yes | NULL | State snapshot prior to update |
| `new_values` | JSON | | Yes | NULL | State snapshot after update |
| `ip_address` | VARCHAR(45) | | Yes | NULL | IPv4 or IPv6 client origin |
| `user_agent` | VARCHAR(255) | | Yes | NULL | Client browser agent string |
| `created_at` | TIMESTAMP | | No | CURRENT_TIMESTAMP | Exact immutable event timestamp |
*Indexes: INDEX(`tenant_id`, `action`, `created_at`), INDEX(`tenant_id`, `user_id`, `created_at`), INDEX(`entity_type`, `entity_id`)*

---

# 5. Role-Based Access Control (RBAC) Matrix

Permissions are granular actions assigned to roles. Roles are assigned to users within a tenant.

### 5.1 Comprehensive Permission Catalog
* **Patient Operations**: `patient.view_basic`, `patient.view_phi`, `patient.create`, `patient.edit`, `patient.delete`
* **Scheduling**: `appointment.view`, `appointment.create`, `appointment.reschedule`, `appointment.cancel`
* **Clinical Encounters**: `visit.create`, `visit.edit_notes`, `diagnosis.create`, `vitals.record`
* **Prescriptions**: `prescription.create`, `prescription.view`, `prescription.dispense`
* **Laboratory**: `lab.order`, `lab.collect_sample`, `lab.enter_results`, `lab.verify`
* **Inpatient / Beds**: `ipd.admit`, `ipd.transfer_bed`, `ipd.discharge`, `bed.manage`
* **Pharmacy & Inventory**: `pharmacy.manage_catalog`, `pharmacy.receive_stock`, `pharmacy.adjust_stock`
* **Billing & Finance**: `billing.create_invoice`, `billing.collect_payment`, `billing.apply_discount`, `billing.view_reports`
* **Administration**: `tenant.configure`, `staff.manage`, `audit.view`

### 5.2 Default Role-to-Permission Mapping Matrix

| Permission | Platform Admin | Hospital Admin | Receptionist | Doctor | Nurse | Pharmacist | Lab Tech | Accountant | Patient (Portal) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| `patient.view_basic` | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (Self) |
| `patient.view_phi` | ❌ (Breakglass) | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (Self) |
| `patient.create` | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `patient.edit` | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `appointment.create`| ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ (Request) |
| `appointment.reschedule`| ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `vitals.record` | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `visit.create` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `diagnosis.create` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `prescription.create`| ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `prescription.dispense`| ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `pharmacy.receive_stock`| ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `lab.order` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `lab.enter_results` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `lab.verify` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ (Lead) | ❌ | ❌ |
| `ipd.admit` | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `ipd.discharge` | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `billing.create_invoice`| ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `billing.collect_payment`| ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `billing.apply_discount`| ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (Limited)| ❌ |
| `staff.manage` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `tenant.configure` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `audit.view` | ✅ (Platform) | ✅ (Tenant) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

# 6. Core Business Engines & State Machines

### 6.1 Doctor Availability & Slot Reservation Engine
Appointments use a conflict-free, high-concurrency booking algorithm backed by Redis locks to prevent double-booking.

```
[ Incoming Booking Request: Doctor ID, Date, Start Time ]
                        │
                        ▼
[ Step 1: Check Doctor Leave Exceptions ]
  - Verify DoctorLeaves does not cover target date
                        │
                        ▼
[ Step 2: Check Weekly Schedule & Slot Boundaries ]
  - DayOfWeek matches DoctorSchedules table
  - Requested slot is within [start_time, end_time] and outside [break_start, break_end]
                        │
                        ▼
[ Step 3: Acquire Redis Distributed Lock ]
  - Key: "lock:slot:{tenant_id}:{doctor_id}:{date}:{start_time}"
  - TTL: 10 seconds
  - If lock fails: Abort with HTTP 409 ("Slot is currently being reserved by another user")
                        │
                        ▼
[ Step 4: Database Verification Inside DB Transaction ]
  - SELECT * FROM appointments 
    WHERE tenant_id = ? AND doctor_id = ? AND appointment_date = ? 
      AND start_time = ? AND status NOT IN ('CANCELLED') FOR UPDATE
  - If existing appointment found: Release lock, throw ValidationException
                        │
                        ▼
[ Step 5: Persist Appointment & Commit Transaction ]
  - Insert new Appointment record with status = 'SCHEDULED'
  - Dispatch AppointmentCreatedEvent -> Queue job SendAppointmentNotificationJob
  - Release Redis Lock
```

### 6.2 Pharmacy Stock Ledger & FEFO Dispensation
Dispensing medicine strictly maintains inventory integrity using **First-Expired, First-Out (FEFO)** order:

1. When a prescription item is confirmed for dispensing:
   * Query `medicine_batches` where `medicine_id = ?` and `quantity > 0`, ordered by `expiry_date ASC`.
   * Open a MySQL Database Transaction.
2. For each batch in FEFO order:
   * Allocate `min(remaining_needed, batch.quantity)`.
   * Decrement `medicine_batches.quantity`.
   * Insert row in `inventory_transactions` (`type = 'DISPENSE'`, `quantity = -deducted`, `balance_after = new_batch_bal`).
3. Decrement `medicines.current_stock` by total dispensed quantity.
4. If `medicines.current_stock <= medicines.reorder_level`:
   * Trigger `LowStockAlertNotification` to hospital pharmacists.
5. Update `prescription_items.dispensed_quantity` and prescription status to `'DISPENSED'`.
6. Dispatch auto-billing hook: Append item to patient’s draft/issued invoice.

### 6.3 Bed Occupancy & Inpatient Billing Aggregator
Bed allocation and inpatient room stays are tracked through an automated state machine:

```
                  ┌──────────────┐
                  │  AVAILABLE   │
                  └──────┬───────┘
                         │ Admit Patient
                         ▼
                  ┌──────────────┐
        ┌─────────┤   OCCUPIED   ├─────────┐
        │         └──────┬───────┘         │
        │ Transfer Bed   │ Discharge       │ Bed Maintenance
        ▼                ▼                 ▼
  ┌───────────┐   ┌──────────────┐   ┌───────────┐
  │ RESERVED  │   │  AVAILABLE   │   │MAINTENANCE│
  └───────────┘   └──────────────┘   └───────────┘
```

**Daily Room Rate Calculation Algorithm**:
* Upon discharge, or daily at midnight via scheduled cron job `CalculateDailyBedChargesJob`:
  * Calculate duration: `max(1, ceil((discharge_or_now - admission_date) in hours / 24))`.
  * Multiplied by `rooms.rate_per_day`.
  * Upsert corresponding line items in `invoice_items` (`item_type = 'BED_CHARGE'`).

---

# 7. Livewire Component Specifications

All UI pages are composed of cohesive, reactive Livewire 3 components.

### 7.1 Receptionist & Front-Desk Module

#### `Reception::PatientRegistrationModal`
* **File**: `app/Livewire/Reception/PatientRegistrationModal.php`
* **Blade**: `resources/views/livewire/reception/patient-registration-modal.blade.php`
* **Component State**:
  * `$first_name`, `$last_name`, `$date_of_birth`, `$gender`, `$phone`, `$email`, `$blood_group`, `$national_id`, `$emergency_contact_name`, `$emergency_contact_phone`, `$allergies`
* **Reactive Behaviors**:
  * Real-time duplicate phone validation against `patients` within current tenant.
  * Automatic `patient_number` generation on modal launch (e.g., `HOSP-YYYY-XXXXX`).
* **Livewire Actions**:
  * `save()`: Validates input, begins DB transaction, creates patient, writes audit log, emits `patient-created` browser event to refresh list, closes modal with toast.

#### `Reception::AppointmentScheduler`
* **File**: `app/Livewire/Reception/AppointmentScheduler.php`
* **Blade**: `resources/views/livewire/reception/appointment-scheduler.blade.php`
* **Component State**:
  * `$selectedDepartmentId`, `$selectedDoctorId`, `$selectedDate`, `$availableSlots = []`, `$patientSearchQuery`, `$selectedPatientId`
* **Reactive Behaviors**:
  * `updatedSelectedDoctorId()` / `updatedSelectedDate()`: Recomputes available slot array dynamically by calling `SlotGeneratorService::getAvailableSlots($doctorId, $date)`.
* **Livewire Actions**:
  * `bookSlot($timeString)`: Acquires Redis lock, executes appointment transaction, dispatches confirmation job, shows success badge.

---

### 7.2 Doctor Clinical Workspace Module

#### `Doctor::ConsultationDesk`
* **File**: `app/Livewire/Doctor/ConsultationDesk.php`
* **Blade**: `resources/views/livewire/doctor/consultation-desk.blade.php`
* **Layout**: Two-column responsive split layout.
  * **Left Column**: Collapsible Patient Timeline (Past visits, diagnoses, chronic allergies, vitals chart, previous lab results).
  * **Right Column**: Active Consultation Tabs (Clinical Notes, Vitals, ICD-10 Diagnosis, E-Prescription Builder, Lab Order Selector).
* **Component State**:
  * `$visitId`, `$patient`, `$systolic_bp`, `$diastolic_bp`, `$temperature`, `$clinical_notes`, `$prescriptionItems = []`, `$selectedLabTests = []`
* **Livewire Actions**:
  * `addPrescriptionRow()`: Dynamically appends `{ medicine_id: null, dosage: '', frequency: '', duration_days: 5 }` to `$prescriptionItems`.
  * `removePrescriptionRow($index)`: Slices row from array.
  * `completeConsultation()`: Atomically validates encounter notes, persists diagnoses, creates prescriptions, queues lab orders, generates draft billing invoice, marks appointment as `'COMPLETED'`, redirects to doctor queue.

---

### 7.3 Pharmacy Dispensing Desk

#### `Pharmacy::DispenseCounter`
* **File**: `app/Livewire/Pharmacy/DispenseCounter.php`
* **Blade**: `resources/views/livewire/pharmacy/dispense-counter.blade.php`
* **Component State**:
  * `$prescriptionSearchCode`, `$activePrescription`, `$batchAllocations = []`
* **Livewire Actions**:
  * `searchPrescription()`: Finds pending prescription by UUID or patient registration code.
  * `autoAllocateBatches()`: Runs FEFO algorithm, presents selected batches to pharmacist for manual barcode verification.
  * `confirmDispense()`: Deducts stock, generates inventory transaction logs, updates prescription to `'DISPENSED'`, prints patient medicine instruction slip.

---

### 7.4 Inpatient Bed Grid (IPD)

#### `Inpatient::BedBoard`
* **File**: `app/Livewire/Inpatient/BedBoard.php`
* **Blade**: `resources/views/livewire/inpatient/bed-board.blade.php`
* **Visual Presentation**: Color-coded ward layout (Emerald = Available, Crimson = Occupied, Amber = Reserved, Gray = Maintenance).
* **Component State**:
  * `$selectedWardId`, `$wards`, `$filterStatus`
* **Livewire Actions**:
  * `openAdmitModal($bedId)`: Opens patient admission modal targeting selected bed.
  * `transferPatient($fromBedId, $toBedId)`: Validates target bed availability, shifts admission record, resets old bed to `'AVAILABLE'`, sets new bed to `'OCCUPIED'`.

---

### 7.5 Cashier & Invoicing Terminal

#### `Billing::CashierTerminal`
* **File**: `app/Livewire/Billing/CashierTerminal.php`
* **Blade**: `resources/views/livewire/billing/cashier-terminal.blade.php`
* **Component State**:
  * `$invoiceId`, `$invoice`, `$paymentAmount`, `$paymentMethod = 'CASH'`, `$transactionReference`, `$discountAmount`
* **Livewire Actions**:
  * `applyDiscount($amount)`: Recalculates `total_amount` and `balance_due` (authorized by `billing.apply_discount` permission).
  * `recordPayment()`: Uses UUID `idempotency_key`, inserts `payments` record, updates `invoices.paid_amount` and status (`'PARTIALLY_PAID'` or `'PAID'`), generates printable receipt PDF.

---

# 8. REST API Specification

For external services, mobile integration, or lab device sync, the system exposes a versioned, secure REST API.

### 8.1 Universal Response Envelope
Every API response adheres to a strict JSON structure:

```json
{
  "success": true,
  "status_code": 200,
  "data": {},
  "meta": {
    "timestamp": "2026-09-14T10:30:00Z",
    "tenant": "mercy-general",
    "pagination": {
      "total": 120,
      "per_page": 15,
      "current_page": 1,
      "last_page": 8
    }
  }
}
```

### 8.2 Authentication & Headers
* All API requests require:
  * `Authorization: Bearer <sanctum_token>`
  * `X-Tenant-ID: <subdomain_or_uuid>` (optional if resolved by subdomain)
  * `Accept: application/json`

### 8.3 Core API Endpoints

| Method | Endpoint | Description | Required Permission |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/auth/login` | Authenticate staff & issue Sanctum token | Public (Rate-limited) |
| `GET` | `/api/v1/patients` | Paginated search of patient demographics | `patient.view_basic` |
| `POST` | `/api/v1/patients` | Register a new patient record | `patient.create` |
| `GET` | `/api/v1/patients/{uuid}` | Full patient profile & medical history | `patient.view_phi` |
| `GET` | `/api/v1/doctors` | List doctors filtered by department | `appointment.view` |
| `GET` | `/api/v1/doctors/{id}/slots` | Get free slots for a specific date | `appointment.view` |
| `POST` | `/api/v1/appointments` | Book appointment with concurrency lock | `appointment.create` |
| `PATCH`| `/api/v1/appointments/{id}/status`| Update status (CHECKED_IN, CANCELLED) | `appointment.reschedule` |
| `POST` | `/api/v1/visits` | Start or record clinical consultation | `visit.create` |
| `POST` | `/api/v1/prescriptions` | Issue medication prescription | `prescription.create` |
| `POST` | `/api/v1/lab-orders` | Create laboratory diagnostic order | `lab.order` |
| `POST` | `/api/v1/lab-orders/{id}/results`| Upload diagnostic test results | `lab.enter_results` |
| `GET` | `/api/v1/invoices` | List invoices by status / date / patient | `billing.create_invoice` |
| `POST` | `/api/v1/payments` | Record payment against invoice (Idempotent)| `billing.collect_payment` |

---

# 9. Background Jobs, Queues & Notifications

### 9.1 Redis Queue Infrastructure
Queues are managed via Laravel Horizon using distinct priority queues:
* `high`: Immediate transactional emails, OTP codes, 2FA tokens, Redis lock releases.
* `notifications`: Appointment booking confirmations, SMS alerts, reminder messages.
* `reports`: Asynchronous PDF invoice creation, lab report compilation, month-end financial rollups.

### 9.2 Asynchronous Job Registry

| Job Class | Queue | Trigger Event | Action |
| :--- | :--- | :--- | :--- |
| `SendAppointmentNotificationJob` | `notifications` | `AppointmentCreatedEvent` | Sends SMS & email confirmation to patient with date and time |
| `SendAppointmentReminderJob` | `notifications` | Scheduled (24h before appointment)| Notifies patient of upcoming clinic visit |
| `GenerateInvoicePdfJob` | `reports` | `InvoiceIssuedEvent` | Compiles DomPDF invoice, stores in storage disk, attaches link |
| `SendLabResultsReadyJob` | `notifications` | `LabResultVerifiedEvent` | Alerts attending doctor and patient that test results are verified |
| `ProcessBatchStockWarningJob` | `high` | `InventoryTransactionRecorded` | Dispatches low-stock / expired medication alerts to pharmacists |
| `CalculateDailyBedChargesJob`| `reports` | Scheduled (Every midnight 00:01) | Calculates 24h occupancy room charges for all active admissions |

---

# 10. Audit Logging, PHI Security & Compliance

### 10.1 Immutable Audit Log Engine
Every model accessing or modifying Protected Health Information (PHI) is tracked by the `AuditObserver` class.

* **Trigger Events**:
  * View of sensitive patient records (`VIEW_PHI` recorded through controller/Livewire hook).
  * Create, Update, Delete of any clinical, prescription, or financial row.
* **Payload Structure**:
  * Captures: `user_id`, `tenant_id`, `action`, `ip_address`, `user_agent`, `old_values` (JSON diff), and `new_values` (JSON diff).
* **Immutability**:
  * MySQL database permissions for standard application users do not grant `UPDATE` or `DELETE` on the `audit_logs` table.

### 10.2 Cryptographic Protection
* **At Rest**:
  * Sensitive patient identifiers (`national_id`, passport numbers) are encrypted using Laravel’s AES-256-CBC encryption (`casts => ['national_id' => 'encrypted']`).
* **In Transit**:
  * Strict HTTPS transport security (HSTS headers enabled).
* **Storage Isolation**:
  * Medical lab attachments (radiology scans, PDF reports) are stored in private object storage disks with time-limited pre-signed URLs (15-minute TTL) generated on-demand.

---

# 11. Decisions on Previous Open Questions

Section 53 of the original draft outlined 15 architectural and business ambiguities. Here are the explicit design decisions codified into this specification:

1. **Tenant Boundaries**: One tenant corresponds strictly to one hospital or clinic organization. Branches within an organization are handled as distinct Departments or Facilities under the same `tenant_id`.
2. **User Multi-Hospital Membership**: For MVP, users belong to a single primary hospital (`users.tenant_id`). Visiting consultants with multi-hospital duties are provisioned distinct user accounts linked by verified email.
3. **Patient ID Uniqueness**: Patient records have an auto-increment internal ID, an external UUID, and a tenant-unique human-readable identifier (e.g. `HOSP-YYYY-XXXXX`) unique per tenant.
4. **Platform Admin PHI Access**: Platform Administrators are strictly prohibited from viewing patient clinical data by default. Any support access requires entering a mandatory "Break-Glass Justification" that logs an immutable high-severity audit record.
5. **Regulatory Baseline**: Built to HIPAA and GDPR security baselines (data encryption at rest, HTTPS in transit, role-based access control, comprehensive access audit trails, soft deletes).
6. **Insurance Workflows**: Marked as Post-MVP Phase 2. The MVP supports standard manual and cash/card billing with itemized deductions and insurance reference tracking fields.
7. **Pharmacy Scope**: Fully included in Phase 1 (Medicine catalog, batch tracking, FEFO automated stock deduction, dispensing, low-stock alarms).
8. **Laboratory Scope**: Fully included in Phase 1 (Test catalog, order placement, specimen tracking, JSON result parameters, PDF attachment).
9. **Inpatient / Admission Scope**: Fully included in Phase 1 (Ward, room, bed tracking, admission lifecycle, bed transfer, daily bed charge accumulation).
10. **Patient Portal**: Read-only patient portal included for viewing personal appointments, prescriptions, diagnostic results, and invoices.
11. **Notification Providers**: Multi-driver notification system using standard Laravel drivers (Mail via Resend/SES; SMS via Twilio; in-app database notifications).
12. **Payment Methods**: Manual Cash, Card POS, Direct Bank Transfer, and Stripe Payment Gateway integration for online invoices.
13. **Data Migration / Seeders**: Comprehensive database seeders provided for standard ICD-10 top 100 codes, sample departments, medication categories, and laboratory investigation panels.
14. **Initial Scale Target**: Designed to effortlessly support 1,000 active concurrent hospital tenants, each handling up to 50,000 annual visits on a horizontally-scaled standard cluster.
15. **Tenancy Compliance Verification**: Shared database / shared schema with application-level global scoping and tenant-prefixed foreign keys is verified as compliant with healthcare regulations when backed by row-level isolation and complete audit logging.

---

# 12. Scaffolding & Implementation Sequence

This specification is ready for immediate, deterministic execution. The recommended scaffolding sequence is:

1. **Phase 1: Foundation & Tenancy Core**
   * Install Laravel 11 + Livewire 3 + Tailwind CSS + WireUI.
   * Configure multi-tenancy middleware, `TenantManager` service, and `BelongsToTenant` Eloquent global scope trait.
   * Run initial migrations for `tenants`, `tenant_settings`, `users`, and RBAC tables.
2. **Phase 2: Master Data & Staff Management**
   * Migrate and seed `departments`, `doctors`, `doctor_schedules`, and `doctor_leaves`.
   * Implement Hospital Admin Livewire components for staff and clinic scheduling.
3. **Phase 3: Front Desk & Patient Care Management**
   * Migrate `patients`, `appointments`, `visits`, and `vitals`.
   * Implement `AppointmentScheduler` (Livewire calendar) and `PatientRegistrationModal`.
4. **Phase 4: Clinical Consultation & E-Prescriptions**
   * Migrate `diagnoses`, `prescriptions`, `prescription_items`, and `lab_orders`.
   * Build `Doctor::ConsultationDesk` Livewire split-view.
5. **Phase 5: Pharmacy & Diagnostic Laboratory**
   * Migrate `medicines`, `medicine_batches`, `inventory_transactions`, `lab_test_types`, and `lab_results`.
   * Build `Pharmacy::DispenseCounter` (FEFO engine) and `Lab::ResultEntryDesk`.
6. **Phase 6: Inpatient Bed Management (IPD)**
   * Migrate `wards`, `rooms`, `beds`, `admissions`.
   * Build interactive visual `BedBoard` Livewire component.
7. **Phase 7: Billing, Cashier & Payments**
   * Migrate `invoices`, `invoice_items`, `payments`.
   * Implement automatic billing aggregator and `Billing::CashierTerminal`.
8. **Phase 8: Audit Logging, Queues & Production Hardening**
   * Implement `AuditObserver` on all PHI models.
   * Configure Redis queues, Horizon, health check endpoints, and automated tests.
