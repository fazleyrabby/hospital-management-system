# Comprehensive Product Requirements Document (PRD) & Technical Specification

## Multi-Tenant Hospital Management System (HMS)

* **Backend Framework**: Laravel 11 (PHP 8.3+)
* **Database**: MySQL 8.0+ (InnoDB with TDE)
* **Cache & Queues**: Redis 7.0+
* **Frontend Architecture**: Laravel Blade + Livewire 3 + Alpine.js + Tailwind CSS
* **Product Type**: Multi-Tenant B2B Healthcare SaaS Platform
* **Document Status**: Production-Ready Technical Specification
* **Version**: 2.1.0 (Hardened Architecture)

---

# Table of Contents
1. [Product Overview & Architectural Philosophy](#1-product-overview--architectural-philosophy)
2. [Multi-Tenancy Engine & Defense-in-Depth Isolation](#2-multi-tenancy-engine--defense-in-depth-isolation)
3. [Frontend Architecture: Blade + Livewire 3](#3-frontend-architecture-blade--livewire-3)
4. [Exhaustive Database Schema & Data Dictionary](#4-exhaustive-database-schema--data-dictionary)
   * 4.1 [Tenancy & Platform Core](#41-tenancy--platform-core)
   * 4.2 [Platform B2B SaaS Subscriptions & Metering](#42-platform-b2b-saas-subscriptions--metering)
   * 4.3 [Hospital Structure & Clinical Staff](#43-hospital-structure--clinical-staff)
   * 4.4 [Patients & Split Clinical Demographics (PHI)](#44-patients--split-clinical-demographics-phi)
   * 4.5 [Clinical Encounters & Medical Records](#45-clinical-encounters--medical-records)
   * 4.6 [Pharmacy & Inventory](#46-pharmacy--inventory)
   * 4.7 [Diagnostic Laboratory](#47-diagnostic-laboratory)
   * 4.8 [Inpatient (IPD) & Bed Management](#48-inpatient-ipd--bed-management)
   * 4.9 [Hospital Billing, Invoicing & Payments](#49-hospital-billing-invoicing--payments)
   * 4.10 [Audit Logs & Security](#410-audit-logs--security)
5. [Role-Based Access Control (RBAC) Matrix](#5-role-based-access-control-rbac-matrix)
6. [Core Business Engines & State Machines](#6-core-business-engines--state-machines)
7. [Livewire Component Specifications](#7-livewire-component-specifications)
8. [REST API & Webhook Specification](#8-rest-api--webhook-specification)
9. [Background Jobs, Queues & Notifications](#9-background-jobs-queues--notifications)
10. [Audit Logging, PHI Security & GDPR Right-to-Erasure](#10-audit-logging-phi-security--gdpr-right-to-erasure)
11. [Decisions on Architecture & Open Questions](#11-decisions-on-architecture--open-questions)
12. [Scaffolding & Implementation Sequence](#12-scaffolding--implementation-sequence)

---

# 1. Product Overview & Architectural Philosophy

The system is a production-grade, multi-tenant hospital management software platform engineered as a modular monolith in Laravel 11. It allows independent hospitals, clinics, and medical centers to operate autonomously on a single unified infrastructure while enforcing absolute data isolation, strict regulatory compliance (HIPAA and GDPR), and high-throughput operational efficiency.

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
         │(Encrypted Tables)│   │ (Cache, Locks, Q)│   │ (S3 / MinIO PHI) │
         └──────────────────┘   └─────────┬────────┘   └──────────────────┘
                                          │
                                          ▼
                               ┌─────────────────────┐
                               │ Laravel Queue Worker│
                               └─────────────────────┘
```

### Core Design Tenets
1. **Defense-in-Depth Data Isolation**: Tenant isolation cannot rely on a single software layer. It is enforced across multiple tiers: domain-level middleware, an automated Eloquent Global Scope, CI architecture validation, and composite database constraints.
2. **Strict PHI & Demographics Segregation**: Personal demographic data (name, phone) is separated at the schema level from sensitive clinical records (allergies, conditions, vitals, diagnoses) to enforce role-based access physically, not just conceptually.
3. **Monolithic Simplicity with Livewire 3**: No complex decoupled SPA overhead. High-fidelity, reactive user interfaces are built with Laravel Blade, Livewire 3, Alpine.js, and Tailwind CSS.
4. **Dual-Layer Encryption & GDPR Compliance**: Database tables use MySQL InnoDB Transparent Data Encryption (TDE) at rest, supplemented by application-level encrypted field casts for sensitive clinical notes. GDPR Right-to-Erasure is supported via a cryptographically irreversible anonymization engine.

---

# 2. Multi-Tenancy Engine & Defense-in-Depth Isolation

### 2.1 Tenancy Strategy: Shared Database / Shared Schema
The system utilizes a **Shared Database with Tenant Discriminator Columns (`tenant_id`)**. Every tenant-owned table stores an indexed foreign key `tenant_id` linked to the `tenants` table.

### 2.2 Tenant Identification & Resolution Pipeline
Incoming HTTP requests pass through the following strict resolution middleware:

```
[ Incoming Request: tenant1.medihms.test/appointments ]
                      │
                      ▼
[ Step 1: Subdomain / Custom Domain Resolution ]
  - Parse host: extract subdomain or match custom_domain
  - Query Tenant::where('subdomain', $subdomain)->where('status', 'ACTIVE')->firstOrFail()
  - If root domain or "admin", mark request as Platform Admin Context.
                      │
                      ▼
[ Step 2: Tenant Context Singleton & DB Session Binding ]
  - Bind resolved tenant into app(TenantManager::class)->setTenant($tenant)
  - Execute DB::statement("SET @current_tenant_id = ?", [$tenant->id]) for database-level tracking
                      │
                      ▼
[ Step 3: Authenticate User & Validate Membership ]
  - Verify auth()->user()->tenant_id === $tenant->id (Platform admins have tenant_id = NULL)
  - Abort with HTTP 403 if user does not belong to the target tenant
                      │
                      ▼
[ Step 4: Eloquent Global Scope Injection ]
  - BelongsToTenant trait automatically injects where('tenant_id', $tenant->id)
  - Automatically sets $model->tenant_id = $tenant->id on model creation
```

### 2.3 Defense-in-Depth (Mitigating Single Point of Failure)
Relying solely on an Eloquent global scope creates a single point of failure (raw SQL, console commands, or a model that omits the trait could leak data). The architecture counters this with **three additional safety nets**:

1. **Automated Architectural CI Tests (Pest Arch / PHPStan)**:
   * A continuous integration check scans all models in `app/Models/Tenant/`.
   * **Rule**: If a model extends `Illuminate\Database\Eloquent\Model` and is not explicitly whitelisted as a platform-level model, the build **fails** if it does not use the `BelongsToTenant` trait and the `SoftDeletes` trait.
2. **Database Composite Unique Constraints**:
   * All business keys (e.g. `patient_number`, `invoice_number`, `order_number`) use composite unique keys prefixed with `tenant_id` (`UNIQUE(tenant_id, patient_number)`). Cross-tenant collisions or accidental cross-inserts violate database constraints immediately.
3. **Automated Route-Model Binding Scoping**:
   * Laravel 11 scoped bindings (`/tenants/{tenant:subdomain}/patients/{patient:uuid}`) verify that the queried entity's `tenant_id` strictly matches the route's tenant context before reaching controller actions.

---

# 3. Frontend Architecture: Blade + Livewire 3

The user interface utilizes **Laravel Blade + Livewire 3 + Alpine.js + Tailwind CSS**, avoiding decoupled SPA complexity.

### 3.1 Stack Composition
* **Livewire 3**: Handles reactive UI state, real-time input validation, dynamic form fields, asynchronous pagination, modals, and event dispatch.
* **Alpine.js**: Handles lightweight client-side state (dropdowns, fly-out drawers, tab switches, tooltips, local date formatters) without server round-trips.
* **Tailwind CSS**: Medical palette (slate, clinical teal, emerald, amber, and crimson).
* **WireUI / Lucide Icons**: High-contrast, clean medical icons for clinical clarity.

### 3.2 Layout Hierarchy
1. `layouts.app`: Main authenticated wrapper for hospital staff with role-aware sidebar navigation, global patient search (`Cmd+K`), active tenant indicator, notification center, and flash toasts.
2. `layouts.admin`: Platform administration layout for SaaS subscriptions, tenant onboarding, and infrastructure health.
3. `layouts.guest`: Clean layout for authentication (login, 2FA challenge, password reset).
4. `layouts.print`: Print-optimized layout for patient prescriptions, diagnostic reports, and billing invoices.

---

# 4. Exhaustive Database Schema & Data Dictionary

All tables run on MySQL 8.0+ with InnoDB and Transparent Data Encryption (TDE). All soft-deletable tables include `deleted_at TIMESTAMP NULL`.

---

### 4.1 Tenancy & Platform Core

#### `tenants`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Internal identifier |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUIDv4 |
| `name` | VARCHAR(191) | | No | | Legal hospital name |
| `subdomain` | VARCHAR(64) | UNIQUE | No | | e.g. `mercy-general` |
| `custom_domain` | VARCHAR(191) | UNIQUE | Yes | NULL | Custom CNAME domain |
| `ssl_status` | ENUM | 'PENDING','PROVISIONED','FAILED' | No | 'PENDING' | Automated SSL state |
| `email` | VARCHAR(191) | | No | | Hospital contact email |
| `phone` | VARCHAR(32) | | No | | Hospital reception phone |
| `address` | TEXT | | Yes | NULL | Physical address |
| `timezone` | VARCHAR(64) | | No | 'UTC' | Hospital timezone |
| `currency_code` | CHAR(3) | | No | 'USD' | ISO currency code |
| `status` | ENUM | 'ACTIVE','SUSPENDED','INACTIVE' | No | 'ACTIVE' | Operational status |
| `logo_path` | VARCHAR(255) | | Yes | NULL | Storage path for brand logo |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Created timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Updated timestamp |
| `deleted_at` | TIMESTAMP | | Yes | NULL | Soft delete timestamp |

#### `tenant_settings`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `key` | VARCHAR(64) | | No | | Configuration key name |
| `value` | JSON | | Yes | NULL | Setting payload |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `key`)*

#### `users`
*Fixes the MySQL NULL-in-unique-index limitation for Platform Admins via a virtual generated column.*

| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Internal user ID |
| `uuid` | CHAR(36) | UNIQUE | No | | External UUIDv4 |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | Yes | NULL | NULL for platform admins |
| `tenant_scope_key`| BIGINT UNSIGNED | GENERATED ALWAYS AS (COALESCE(tenant_id, 0)) STORED | No | 0 | Virtual column for indexing |
| `name` | VARCHAR(191) | | No | | Full name |
| `email` | VARCHAR(191) | | No | | User login email |
| `password` | VARCHAR(255) | | No | | Bcrypt hashed password |
| `phone` | VARCHAR(32) | | Yes | NULL | Contact phone |
| `status` | ENUM | 'ACTIVE','INVITED','DEACTIVATED' | No | 'ACTIVE' | Account state |
| `email_verified_at`| TIMESTAMP | | Yes | NULL | Email verification time |
| `two_factor_secret`| TEXT | Encrypted at rest (`casts => encrypted`) | Yes | NULL | 2FA TOTP secret |
| `last_login_at` | TIMESTAMP | | Yes | NULL | Last authentication |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `deleted_at` | TIMESTAMP | | Yes | NULL | Soft delete timestamp |
*Indexes: UNIQUE(`tenant_scope_key`, `email`)*

---

### 4.2 Platform B2B SaaS Subscriptions & Metering

#### `platform_plans`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Plan ID |
| `code` | VARCHAR(32) | UNIQUE | No | | e.g., 'BASIC', 'PRO', 'ENTERPRISE' |
| `name` | VARCHAR(100) | | No | | Display name |
| `price_monthly` | DECIMAL(10,2) | | No | 0.00 | Monthly subscription price |
| `max_doctors` | SMALLINT UNSIGNED| | No | 10 | Doctor seat ceiling |
| `max_beds` | SMALLINT UNSIGNED| | No | 25 | Inpatient bed ceiling |
| `has_pharmacy` | BOOLEAN | | No | TRUE | Feature flag |
| `has_lab` | BOOLEAN | | No | TRUE | Feature flag |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |

#### `platform_subscriptions`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Subscription ID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Target hospital tenant |
| `plan_id` | BIGINT UNSIGNED | FK -> platform_plans(id) | No | | Active SaaS plan |
| `stripe_customer_id`| VARCHAR(128)| | Yes | NULL | Stripe Customer reference |
| `stripe_subscription_id`| VARCHAR(128)| UNIQUE | Yes | NULL | Stripe Subscription reference |
| `status` | ENUM | 'ACTIVE','TRIALING','PAST_DUE','CANCELED' | No | 'ACTIVE' | Subscription status |
| `current_period_start`| DATETIME | | No | | Start of billing cycle |
| `current_period_end` | DATETIME | | No | | End of billing cycle |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: INDEX(`tenant_id`, `status`)*

#### `tenant_usage_metering`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Metering log ID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant |
| `recorded_at` | DATE | | No | | Snapshot date |
| `active_doctors_count`| SMALLINT UNSIGNED| | No | 0 | Seat utilization |
| `active_beds_count`| SMALLINT UNSIGNED| | No | 0 | Bed capacity utilization |
| `storage_bytes_used`| BIGINT UNSIGNED | | No | 0 | S3/MinIO consumption |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
*Indexes: UNIQUE(`tenant_id`, `recorded_at`)*

---

### 4.3 Hospital Structure & Clinical Staff

#### `departments`
* `id`, `tenant_id`, `name`, `code`, `description`, `head_doctor_id`, `status` ('ACTIVE','INACTIVE'), `created_at`, `updated_at`, `deleted_at`.
* *Indexes: UNIQUE(`tenant_id`, `code`), FK -> tenants(id)*

#### `doctors`
* `id`, `tenant_id`, `user_id`, `department_id`, `license_number`, `specialization`, `qualification`, `consultation_fee`, `slot_duration_mins` (default 15), `bio`, `status` ('ACTIVE','ON_LEAVE','RESIGNED'), `created_at`, `updated_at`, `deleted_at`.
* *Indexes: UNIQUE(`tenant_id`, `user_id`), UNIQUE(`tenant_id`, `license_number`), FK -> departments(id)*

#### `doctor_schedules` (Weekly Availability)
* `id`, `tenant_id`, `doctor_id`, `day_of_week` (0-6), `start_time`, `end_time`, `break_start_time`, `break_end_time`, `max_patients`, `created_at`, `updated_at`.
* *Indexes: INDEX(`tenant_id`, `doctor_id`, `day_of_week`)*

#### `doctor_leaves`
* `id`, `tenant_id`, `doctor_id`, `start_date`, `end_date`, `reason`, `created_at`, `updated_at`.

---

### 4.4 Patients & Split Clinical Demographics (PHI)

*Patients are divided into two physical tables to decouple non-sensitive demographic data from sensitive clinical PHI.*

#### `patients` (Demographic Layer — Accessible by Reception & Billing)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUIDv4 |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Owning tenant |
| `patient_number`| VARCHAR(64) | | No | | Human readable (e.g. `HOSP-2026-00012`) |
| `first_name` | VARCHAR(100) | | No | | First name |
| `last_name` | VARCHAR(100) | | No | | Last name |
| `date_of_birth` | DATE | | No | | Birth date |
| `gender` | ENUM | 'MALE','FEMALE','OTHER' | No | | Biological gender |
| `phone` | VARCHAR(32) | | No | | Contact phone |
| `email` | VARCHAR(191) | | Yes | NULL | Contact email |
| `address` | TEXT | | Yes | NULL | Residential address |
| `emergency_contact_name` | VARCHAR(191) | | Yes | NULL | Kin name |
| `emergency_contact_phone`| VARCHAR(32) | | Yes | NULL | Kin phone |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `deleted_at` | TIMESTAMP | | Yes | NULL | Soft delete timestamp |
*Indexes: UNIQUE(`tenant_id`, `patient_number`), INDEX(`tenant_id`, `phone`), INDEX(`tenant_id`, `first_name`, `last_name`)*

#### `patient_clinical_profiles` (Protected Health Information Layer — Doctor & Nurse Access Only)
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Owning tenant |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE CASCADE | No | | 1-to-1 link to patient demographics |
| `blood_group` | ENUM | 'A+','A-','B+','B-','AB+','AB-','O+','O-','UNKNOWN' | No | 'UNKNOWN' | Blood classification |
| `national_id` | VARCHAR(255) | Encrypted at rest (`casts => encrypted`) | Yes | NULL | SSN / National Identity |
| `allergies` | TEXT | Encrypted at rest (`casts => encrypted`) | Yes | NULL | Known drug/food allergies |
| `chronic_conditions` | TEXT | Encrypted at rest (`casts => encrypted`) | Yes | NULL | Ongoing clinical conditions |
| `immunization_notes` | TEXT | Encrypted at rest (`casts => encrypted`) | Yes | NULL | Vaccine records |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `deleted_at` | TIMESTAMP | | Yes | NULL | Soft delete timestamp |
*Indexes: UNIQUE(`tenant_id`, `patient_id`)*

---

### 4.5 Clinical Encounters & Medical Records

#### `appointments`
| Column | Type | Constraints | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | BIGINT UNSIGNED | PRIMARY KEY, AUTO_INCREMENT | No | | Primary key |
| `uuid` | CHAR(36) | UNIQUE | No | | Public UUID |
| `tenant_id` | BIGINT UNSIGNED | FK -> tenants(id) ON DELETE CASCADE | No | | Tenant owner |
| `patient_id` | BIGINT UNSIGNED | FK -> patients(id) ON DELETE RESTRICT | No | | Patient |
| `doctor_id` | BIGINT UNSIGNED | FK -> doctors(id) ON DELETE RESTRICT | No | | Attending doctor |
| `department_id` | BIGINT UNSIGNED | FK -> departments(id) ON DELETE RESTRICT | No | | Clinical department |
| `appointment_date`| DATE | | No | | Scheduled date |
| `start_time` | TIME | | No | | Slot start time |
| `end_time` | TIME | | No | | Slot end time |
| `type` | ENUM | 'OUTPATIENT','FOLLOW_UP','EMERGENCY'| No | 'OUTPATIENT' | Category of visit |
| `status` | ENUM | 'SCHEDULED','CONFIRMED','CHECKED_IN','IN_PROGRESS','COMPLETED','CANCELLED','NO_SHOW' | No | 'SCHEDULED' | Appointment lifecycle |
| `chief_complaint` | TEXT | | Yes | NULL | Initial symptom description |
| `cancellation_reason`| VARCHAR(255)| | Yes | NULL | Cancellation reason |
| `created_by_user_id`| BIGINT UNSIGNED | FK -> users(id) | No | | Staff who scheduled |
| `created_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `updated_at` | TIMESTAMP | | Yes | CURRENT_TIMESTAMP | Timestamp |
| `deleted_at` | TIMESTAMP | | Yes | NULL | Soft delete timestamp |
*Indexes: INDEX(`tenant_id`, `doctor_id`, `appointment_date`, `start_time`, `end_time`), INDEX(`tenant_id`, `patient_id`)*

#### `visits` (Encounters)
* `id`, `tenant_id`, `patient_id`, `doctor_id`, `appointment_id`, `encounter_date`, `visit_type` ('OPD','IPD','EMERGENCY'), `clinical_notes` (Encrypted), `examination` (Encrypted), `treatment_plan` (Encrypted), `status` ('OPEN','COMPLETED','DISCHARGED'), `created_at`, `updated_at`, `deleted_at`.
* *Indexes: INDEX(`tenant_id`, `patient_id`), INDEX(`tenant_id`, `encounter_date`)*

#### `vitals`
* `id`, `tenant_id`, `visit_id`, `patient_id`, `systolic_bp`, `diastolic_bp`, `heart_rate`, `respiratory_rate`, `temperature_c`, `oxygen_saturation`, `weight_kg`, `height_cm`, `bmi`, `recorded_by_user_id`, `created_at`.

#### `diagnoses`
* `id`, `tenant_id`, `visit_id`, `patient_id`, `icd10_code`, `diagnosis_name` (Encrypted), `diagnosis_type` ('PROVISIONAL','FINAL','DIFFERENTIAL'), `comments` (Encrypted), `created_at`.

---

### 4.6 Pharmacy & Inventory

* `medicine_categories`: `id`, `tenant_id`, `name`, `description`, `created_at`, `updated_at`. *UNIQUE(`tenant_id`, `name`)*
* `medicines`: `id`, `tenant_id`, `category_id`, `name`, `generic_name`, `dosage_form`, `strength`, `unit`, `unit_price`, `cost_price`, `reorder_level`, `current_stock`, `status`, `created_at`, `updated_at`, `deleted_at`.
* `medicine_batches`: `id`, `tenant_id`, `medicine_id`, `batch_number`, `expiry_date`, `quantity`, `created_at`, `updated_at`. *UNIQUE(`tenant_id`, `medicine_id`, `batch_number`)*
* `prescriptions`: `id`, `uuid`, `tenant_id`, `visit_id`, `patient_id`, `doctor_id`, `status` ('PENDING','DISPENSED','PARTIALLY_DISPENSED','CANCELLED'), `notes`, `created_at`, `updated_at`, `deleted_at`.
* `prescription_items`: `id`, `tenant_id`, `prescription_id`, `medicine_id`, `dosage`, `frequency`, `duration_days`, `quantity`, `instructions`, `dispensed_quantity`, `created_at`.
* `inventory_transactions`: `id`, `tenant_id`, `medicine_id`, `batch_id`, `transaction_type` ('PURCHASE_RECEIPT','DISPENSE','RETURN','ADJUSTMENT_LOSS','EXPIRED'), `quantity`, `balance_after`, `reference_type`, `reference_id`, `performed_by_user_id`, `remarks`, `created_at`.

---

### 4.7 Diagnostic Laboratory

* `lab_test_types`: `id`, `tenant_id`, `code`, `name`, `category`, `sample_type`, `price`, `reference_range`, `created_at`, `updated_at`. *UNIQUE(`tenant_id`, `code`)*
* `lab_orders`: `id`, `uuid`, `tenant_id`, `visit_id`, `patient_id`, `doctor_id`, `test_type_id`, `order_number`, `status` ('ORDERED','SAMPLE_COLLECTED','PROCESSING','COMPLETED','CANCELLED'), `clinical_notes`, `created_at`, `updated_at`, `deleted_at`.
* `lab_results`: `id`, `tenant_id`, `lab_order_id`, `result_data` (JSON), `summary`, `is_abnormal`, `file_attachment_path`, `verified_by_user_id`, `verified_at`, `created_at`.

---

### 4.8 Inpatient (IPD) & Bed Management

* `wards`: `id`, `tenant_id`, `name`, `gender`, `capacity`, `status`, `created_at`, `updated_at`.
* `rooms`: `id`, `tenant_id`, `ward_id`, `room_number`, `type`, `rate_per_day`, `created_at`, `updated_at`.
* `beds`: `id`, `tenant_id`, `room_id`, `bed_number`, `status` ('AVAILABLE','OCCUPIED','MAINTENANCE','RESERVED'), `created_at`, `updated_at`.
* `admissions`: `id`, `uuid`, `tenant_id`, `admission_number`, `patient_id`, `attending_doctor_id`, `bed_id`, `admission_date`, `discharge_date`, `admission_reason`, `discharge_summary`, `status` ('ADMITTED','DISCHARGED','TRANSFERRED'), `created_at`, `updated_at`, `deleted_at`.

---

### 4.9 Hospital Billing, Invoicing & Payments

* `invoices`: `id`, `uuid`, `tenant_id`, `invoice_number`, `patient_id`, `visit_id`, `admission_id`, `subtotal`, `discount_amount`, `tax_amount`, `total_amount`, `paid_amount`, `balance_due`, `status` ('DRAFT','ISSUED','PARTIALLY_PAID','PAID','CANCELLED','REFUNDED'), `due_date`, `created_by_user_id`, `created_at`, `updated_at`, `deleted_at`.
* `invoice_items`: `id`, `tenant_id`, `invoice_id`, `item_type`, `description`, `quantity`, `unit_price`, `total_price`, `reference_type`, `reference_id`, `created_at`.
* `payments`: `id`, `uuid`, `tenant_id`, `payment_number`, `invoice_id`, `amount`, `payment_method` ('CASH','CREDIT_CARD','DEBIT_CARD','BANK_TRANSFER','STRIPE','MOBILE_PAYMENT'), `transaction_reference`, `idempotency_key` (UNIQUE), `received_by_user_id`, `payment_date`, `created_at`.

---

### 4.10 Audit Logs & Security

#### `audit_logs` (Append-Only Event Log)
* `id`, `tenant_id`, `user_id`, `action`, `entity_type`, `entity_id`, `old_values` (JSON), `new_values` (JSON), `ip_address`, `user_agent`, `created_at`.
* *Indexes: INDEX(`tenant_id`, `action`, `created_at`), INDEX(`tenant_id`, `user_id`, `created_at`), INDEX(`entity_type`, `entity_id`)*

---

# 5. Role-Based Access Control (RBAC) Matrix

### 5.1 Comprehensive Permission Catalog
* **Patient Operations**: `patient.view_basic` (demographics), `patient.view_phi` (clinical history & vitals), `patient.create`, `patient.edit`, `patient.anonymize`
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
| `patient.anonymize` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `appointment.create`| ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ (Request) |
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
| `staff.manage` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `tenant.configure` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `audit.view` | ✅ (Platform) | ✅ (Tenant) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

# 6. Core Business Engines & State Machines

### 6.1 Doctor Availability & Overlapping Slot Reservation Engine
To prevent double-booking when appointments have varying durations (e.g. 15-minute checkups vs. 45-minute emergency consultations), the booking engine checks for **range overlaps** rather than exact start-time matches.

```
[ Incoming Booking Request: Doctor ID, Date, Start Time, End Time ]
                               │
                               ▼
[ Step 1: Check Doctor Leave Exceptions ]
  - Verify DoctorLeaves does not cover the requested date
                               │
                               ▼
[ Step 2: Check Schedule Bounds ]
  - DayOfWeek matches DoctorSchedules
  - Slot falls within [start_time, end_time] and outside [break_start, break_end]
                               │
                               ▼
[ Step 3: Acquire Redis Distributed Lock ]
  - Key: "lock:doctor_schedule:{tenant_id}:{doctor_id}:{date}"
  - TTL: 10 seconds
                               │
                               ▼
[ Step 4: Database Range Overlap Check (Pessimistic Locking) ]
  - Query with FOR UPDATE:
    SELECT * FROM appointments 
    WHERE tenant_id = :tenant_id 
      AND doctor_id = :doctor_id 
      AND appointment_date = :date 
      AND start_time < :new_end_time 
      AND end_time > :new_start_time 
      AND status NOT IN ('CANCELLED', 'NO_SHOW') 
    FOR UPDATE;
  - If any row returned: Throw SlotConflictException ("Requested time range overlaps with an existing appointment")
                               │
                               ▼
[ Step 5: Insert Appointment & Commit ]
  - Insert record with status = 'SCHEDULED'
  - Release Redis Lock
```

### 6.2 Pharmacy FEFO Dispensation & Atomic Stock Depletion
Prescriptions are dispensed strictly following **First-Expired, First-Out (FEFO)**:
1. When confirmed for dispensing, query batches: `WHERE medicine_id = ? AND quantity > 0 ORDER BY expiry_date ASC`.
2. Inside a database transaction:
   * Allocate units from earliest expiring batches.
   * Decrement `medicine_batches.quantity`.
   * Log an immutable row in `inventory_transactions` (`type = 'DISPENSE'`, `quantity = -X`).
3. Decrement `medicines.current_stock`.
4. If `current_stock <= reorder_level`, dispatch `LowStockAlertNotification`.
5. Update prescription status to `'DISPENSED'`.

### 6.3 Inpatient Bed Occupancy & Room Charge Calculation
* **Bed State Transitions**: `AVAILABLE` ↔ `OCCUPIED` ↔ `RESERVED` ↔ `MAINTENANCE`.
* **Daily Room Charge Calculation**:
  * Executed daily at midnight via `CalculateDailyBedChargesJob`:
  * Calculated as `max(1, ceil((discharge_or_now - admission_date) in hours / 24)) * rooms.rate_per_day`.
  * Upserts item in `invoice_items` (`item_type = 'BED_CHARGE'`).

---

# 7. Livewire Component Specifications

### 7.1 Front Desk & Scheduling
* **`Reception::PatientRegistrationModal`**: Real-time duplicate phone validation, automatic patient number assignment, writes basic demographics to `patients`.
* **`Reception::AppointmentScheduler`**: Interactive calendar with doctor slot picker, real-time availability calculation, range conflict validation.

### 7.2 Clinical Desk
* **`Doctor::ConsultationDesk`**: Two-column layout:
  * *Left Column*: Historical patient timeline (diagnoses, previous visits, lab history).
  * *Right Column*: Live consultation workspace (Vitals, ICD-10 diagnosis picker, e-prescription rows, lab test checkboxes).
  * Persists visit notes and clinical data with application-level encryption.

### 7.3 Pharmacy & Laboratory
* **`Pharmacy::DispenseCounter`**: Barcode/UUID prescription lookup, FEFO batch allocation preview, physical dispensing confirmation, printable instruction slips.
* **`Lab::ResultEntryDesk`**: Specimen collection logging, dynamic JSON parameter entry based on test type (e.g. Hemoglobin, Platelets), automatic flag for out-of-range values.

### 7.4 Inpatient Bed Grid
* **`Inpatient::BedBoard`**: Color-coded ward layout (Emerald = Available, Crimson = Occupied, Amber = Reserved, Gray = Maintenance). Drag-and-drop or modal-driven patient bed transfers.

### 7.5 Cashier & Invoicing
* **`Billing::CashierTerminal`**: Aggregated line-item invoice inspector, concession/discount authorization, idempotent payment processing (Cash, POS, Stripe), printable PDF receipts.

---

# 8. REST API & Webhook Specification

### 8.1 Universal Response Envelope
```json
{
  "success": true,
  "status_code": 200,
  "data": {},
  "meta": {
    "timestamp": "2026-09-14T10:30:00Z",
    "tenant": "mercy-general"
  }
}
```

### 8.2 Endpoint Catalog
* `POST /api/v1/auth/login`: Sanctum token issuance (rate-limited).
* `GET /api/v1/patients`: Demographic search (`patient.view_basic`).
* `GET /api/v1/patients/{uuid}/clinical`: PHI profile (`patient.view_phi`).
* `POST /api/v1/appointments`: Book appointment with range lock.
* `POST /api/v1/visits`: Start or record clinical encounter.
* `POST /api/v1/prescriptions`: Prescribe medications.
* `POST /api/v1/lab-orders`: Issue lab order.
* `POST /api/v1/payments`: Idempotent payment recording.
* `POST /api/v1/webhooks/stripe`: Stripe payment & subscription reconciliation (signature verified).

---

# 9. Background Jobs, Queues & Notifications

Managed via Laravel Horizon across three Redis queues:
* `high`: Immediate OTPs, 2FA tokens, Redis lock releases.
* `notifications`: Appointment confirmations, SMS alerts, reminder emails.
* `reports`: Asynchronous PDF invoice creation, lab report compilation, month-end financial rollups.

### Asynchronous Job Registry

| Job Class | Queue | Trigger Event | Action |
| :--- | :--- | :--- | :--- |
| `SendAppointmentNotificationJob` | `notifications` | `AppointmentCreatedEvent` | Sends SMS & email confirmation to patient |
| `SendAppointmentReminderJob` | `notifications` | Scheduled (24h prior) | Sends clinic reminder to patient |
| `GenerateInvoicePdfJob` | `reports` | `InvoiceIssuedEvent` | Renders PDF invoice and stores on disk |
| `SendLabResultsReadyJob` | `notifications` | `LabResultVerifiedEvent` | Alerts attending doctor and patient |
| `ProcessBatchStockWarningJob` | `high` | `InventoryTransactionRecorded` | Alerts pharmacists of low or expiring stock |
| `CalculateDailyBedChargesJob`| `reports` | Scheduled (00:01 daily) | Computes daily occupancy fees for active stays |
| `StripeWebhookReconciliationJob`| `high` | Stripe webhook received | Idempotently updates subscription/invoice status |
| `AnonymizePatientDataJob` | `reports` | `PatientErasureRequested` | Executes GDPR-compliant irreversible anonymization |

---

# 10. Audit Logging, PHI Security & GDPR Right-to-Erasure

### 10.1 Dual-Layer Encryption Strategy
1. **Infrastructure Level**: MySQL InnoDB Transparent Data Encryption (TDE) encrypts all database files and logs on disk.
2. **Application Level**: Sensitive clinical fields are encrypted using AES-256-CBC via Laravel model casts:
   * `patient_clinical_profiles.national_id`, `allergies`, `chronic_conditions`
   * `visits.clinical_notes`, `examination`, `treatment_plan`
   * `diagnoses.diagnosis_name`, `comments`
   * `users.two_factor_secret`

### 10.2 Immutable Audit Logging
* Any read access to `patient_clinical_profiles` or clinical records generates an immutable `VIEW_PHI` audit log entry.
* Database user grants for application connections strictly exclude `UPDATE` or `DELETE` on the `audit_logs` table.

### 10.3 GDPR Right-to-Erasure (Anonymization Engine)
Medical regulations require retaining medical records and financial books for statutory periods (often 5–10 years), conflicting with GDPR Art. 17 (Right to Erasure). The system resolves this conflict through **cryptographic anonymization**:

When `AnonymizePatientDataJob` executes:
1. `patients` demographic record is sanitized:
   * `first_name = 'ANONYMIZED'`, `last_name = 'PATIENT'`
   * `phone = NULL`, `email = NULL`, `address = NULL`
   * `emergency_contact_name = NULL`, `emergency_contact_phone = NULL`
2. `patient_clinical_profiles` row is hard-deleted.
3. Financial ledgers (`invoices`, `payments`) and encounter counters are retained anonymously for statutory audit purposes without linking back to any natural person.
4. An immutable audit record is logged: `action: 'PATIENT_ERASURE_COMPLETED'`.

---

# 11. Decisions on Architecture & Open Questions

1. **Tenant Isolation Single Point of Failure**: Addressed with a 3-tier defense: Eloquent global scope, automated CI architectural tests, and composite unique keys.
2. **Platform Admin Unique Email Constraint**: Resolved using a virtual generated column `tenant_scope_key = COALESCE(tenant_id, 0)` with `UNIQUE(tenant_scope_key, email)`.
3. **Segregation of PHI from Demographics**: Solved by splitting `patients` into `patients` (demographic) and `patient_clinical_profiles` (clinical PHI).
4. **Appointment Slot Collisions**: Range overlap verification (`start_time < :end AND end_time > :start`) inside a pessimistic lock (`FOR UPDATE`) replaces exact-time matching.
5. **Soft Deletes**: Added `deleted_at TIMESTAMP NULL` to all core transactional and demographic tables.
6. **Platform-Side Billing**: Added `platform_plans`, `platform_subscriptions`, and `tenant_usage_metering` tables.
7. **Custom Domain SSL**: Automates SSL issuance via Caddy On-Demand TLS or Cloudflare for SaaS.
8. **Stripe Integration**: Added `StripeWebhookReconciliationJob` with idempotency safeguards.

---

# 12. Scaffolding & Implementation Sequence

1. **Phase 1: Foundation & Multi-Tenancy Core**
   * Setup Laravel 11, Livewire 3, Tailwind CSS.
   * Run migrations for `tenants`, `platform_plans`, `platform_subscriptions`, `users`, and RBAC.
   * Implement `TenantManager` and `BelongsToTenant` trait with CI architecture test.
2. **Phase 2: Master Data & Clinical Rosters**
   * Migrate and seed `departments`, `doctors`, `doctor_schedules`, and `doctor_leaves`.
3. **Phase 3: Front Desk & Demographics**
   * Migrate `patients` and `patient_clinical_profiles`.
   * Implement `Reception::PatientRegistrationModal` and `AppointmentScheduler`.
4. **Phase 4: Clinical Encounters & Medical Records**
   * Migrate `appointments`, `visits`, `vitals`, `diagnoses`, `prescriptions`.
   * Build `Doctor::ConsultationDesk` Livewire split-view.
5. **Phase 5: Pharmacy & Diagnostics**
   * Migrate `medicines`, `batches`, `inventory_transactions`, `lab_orders`, `lab_results`.
   * Implement `Pharmacy::DispenseCounter` (FEFO engine) and `Lab::ResultEntryDesk`.
6. **Phase 6: Inpatient Bed Management (IPD)**
   * Migrate `wards`, `rooms`, `beds`, `admissions`.
   * Implement interactive `BedBoard` component and daily room charge cron job.
7. **Phase 7: Billing & Cashier**
   * Migrate `invoices`, `invoice_items`, `payments`.
   * Implement auto-billing aggregator and `Billing::CashierTerminal`.
8. **Phase 8: Audit Logging, Queues & Security Hardening**
   * Implement `AuditObserver`, encryption casts, Horizon queues, and health check endpoints.
