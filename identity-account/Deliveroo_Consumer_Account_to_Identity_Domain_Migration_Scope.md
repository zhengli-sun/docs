# Scoping Document: Migrating Deliveroo Consumer Account Data into Identity Domain in Pedregal

## Document Information
| Field | Value |
|-------|-------|
| **Author(s)** | Zhengli Sun |
| **Created** | January 12, 2026 |
| **Status** | Draft |
| **Related Documents** | [Consumer Profile Data Ownership in Pedregal](https://docs.google.com/document/d/1C7KMQ-PFAP9M46gtR1CLvHJh1Jq5scSm4lz3IW6-KG0/edit)<br/>[COPS Pedregal Ownership Segmentation](https://docs.google.com/document/d/139x4S-Y31X-1Sc3Pjf8-KfHKMDwZfeB_ipDznxBy2vc/edit)<br/>[RFC Pedregal Consumer](RFC%20Pedregal%20Consumer.md) |

---

## 1. Executive Summary

This document outlines the scope and data mapping for migrating Deliveroo (Roo) consumer account data into the **Identity Domain** within the Pedregal framework. Currently, Deliveroo uses a `users` table in the orderweb database to store consumer account information. As part of the Pedregal Consumer migration, Identity-related fields from Deliveroo's `users` table need to be mapped to appropriate domains, with Identity Domain ownership for key identification and authentication fields.

### Key Objectives:
1. **Identify** all consumer account data fields in Deliveroo's `users` table
2. **Map** these fields to appropriate Pedregal domains based on ownership principles
3. **Define** migration path for Identity Domain fields
4. **Align** with existing Pedregal Consumer and Identity Mapping Layer (IML) design

---

## 2. Background

### 2.1 Current State: Deliveroo (Roo) Users Table

Deliveroo stores consumer data in a PostgreSQL `users` table within the `orderweb` application. This monolithic table contains fields related to:
- **Identity** (email, passwords, DRN identifiers)
- **Personal Information** (names, phone numbers, date of birth)
- **Fraud & Security** (fraud flags, authentication tokens)
- **Payments** (Stripe, Braintree, Checkout.com customer IDs)
- **Status & Metadata** (creation timestamps, last activity)
- **Preferences & Subscriptions** (VIP status, marketing preferences)

**Key Model:** `roo/orderweb/app/models/user.rb`
- Table name: `users`
- Primary Key: `id` (int64)
- DRN Support: `drn_id` (UUID)

### 2.2 Pedregal Target State

According to Pedregal architecture:
- **Pedregal Consumer** is a minimal entity maintaining only `consumer_id`, `actor_id`, `persona`, and `created_at`
- **Identity Domain** owns authentication, account identifiers, and user credentials
- **Other domains** (Geo, Money, Platform, etc.) own their respective profile attributes
- **Identity Mapping Layer (IML)** provides translation between `account_id` and domain-specific identifiers like `consumer_id`

---

## 3. Deliveroo Users Table: Field Inventory and Domain Mapping

Based on database migrations and model analysis, the Deliveroo `users` table contains the following fields:

### 3.1 Core Identity Fields (Identity Domain)

| Deliveroo Field | Data Type | Nullable | Active? | Proposed Owner | Pedregal Mapping | Notes |
|-----------------|-----------|----------|---------|----------------|------------------|-------|
| `id` | INT8 | No | ✅ | Identity | `account_id` (as namespace user ID) | Primary key, maps to Pedregal Account ID |
| `drn_id` | UUID | Yes | ✅ | Identity | `account_id` (DRN format) | Deliveroo Resource Name - unique identifier across Roo |
| `short_drn_id` | INT | Yes | ✅ | Identity | Derived from `drn_id` | Short form of DRN for compact references |
| `email` | VARCHAR | No | ✅ | Identity | `email` in Account | Primary authentication identifier |
| `password_digest` | VARCHAR | No | ✅ | Identity | Password hash in Account | Bcrypt hashed password |
| `email_ctrl_token` | VARCHAR(80) | Yes | ✅ | Identity | Email verification token | Used for email unsubscribe/control |
| `session_token` | VARCHAR | Yes | ❌ | Identity | Session management (deprecated) | Deprecated in favor of JWT tokens |
| `session_expires_at` | TIMESTAMP | Yes | ❌ | Identity | Session management (deprecated) | |
| `password_reset_token` | VARCHAR | Yes | ✅ | Identity | Password reset token | Temporary token for password reset |
| `password_reset_token_expires_at` | TIMESTAMP | Yes | ✅ | Identity | Password reset expiry | |
| `created_at` | TIMESTAMP | No | ✅ | Identity | `created_at` in Account | Account creation timestamp |
| `updated_at` | TIMESTAMP | Yes | ✅ | Identity | `updated_at` in Account | Last modification timestamp |
| `status` | INT | Yes | ✅ | Identity | Account status (enabled/disabled) | Enumerated: 0=enabled, 1=disabled |
| `drn_market` | CHAR(2) | Yes | ✅ | Identity | Market/region identifier | ISO country code (e.g., 'GB', 'FR') |
| `ravelin_id` | VARCHAR | Yes | ✅ | Identity/Fraud | External fraud service identifier | Ravelin fraud detection system |
| `is_invitee` | BOOL | Yes | ✅ | Identity | Referral flag | Whether user was invited/referred |

### 3.2 Personal Information (Consumer Domain / Identity Domain)

| Deliveroo Field | Data Type | Nullable | Active? | Proposed Owner | Pedregal Mapping | Notes |
|-----------------|-----------|----------|---------|----------------|------------------|-------|
| `first_name` | VARCHAR | No | ✅ | Identity | `first_name` in Account PII | Legal first name |
| `last_name` | VARCHAR | No | ✅ | Identity | `last_name` in Account PII | Legal last name |
| `full_name` | VARCHAR(255) | Yes | ✅ | Identity | Computed from first/last | Full display name |
| `preferred_name` | VARCHAR(255) | Yes | ✅ | Identity | Preferred name in Account | How user prefers to be addressed |
| `mobile` | VARCHAR(60) | Yes | ✅ | Identity | `phone_number` in Account | E.164 formatted phone number |
| `mobile_verified_at` | TIMESTAMP | Yes | ✅ | Identity | Phone verification timestamp | When phone was verified |
| `date_of_birth` | DATE | Yes | ✅ | Identity / Consumer | Date of birth | Age verification, compliance |
| `is_valid_e164_phone_number` | BOOL | Yes | ✅ | Identity | Phone validation status | Whether mobile is valid E.164 format |

**Note:** Phone number is also stored in `user_phone` table with country-specific formatting.

### 3.3 Co-Data Service Note

**Important:** Deliveroo has a separate microservice called `co-data` that manages **specific consumer profile attributes** independently from the `users` table. This is a key architectural pattern in Deliveroo.

**Co-Data Service Responsibilities:**
- **Date of Birth:** Stores and manages user date of birth separately from the `users` table
- **Cookie Consent & Advertising Preferences:** Manages GDPR/privacy consent data by device/anonymous ID
- **Consumer Profile Data:** Additional consumer attributes that have been extracted from orderweb

**Key Point:** `co-data` is **NOT** a data store that duplicates the `users` table. It is a **specialized service that owns specific consumer attributes** that have been moved out of orderweb for better domain separation.

**Migration Implications:**
1. The `users.date_of_birth` field is **deprecated** in favor of co-data service
2. Orderweb uses a feature flag (`orderweb_co_data_get_date_of_birth`) to read from co-data instead of `user_birth_dates` table
3. When migrating to Pedregal Identity, we need to coordinate with co-data team to determine:
   - Should date of birth stay in co-data or move to Identity Domain?
   - What other consumer attributes does co-data manage?
   - How does co-data integrate with Pedregal architecture?

**Co-Data API Methods (observed):**
- `GetConsumer(drn_id, field_mask)` - Fetch consumer data by DRN
- `SetDateOfBirth(drn_id, date_of_birth)` - Store DOB
- `GetDateOfBirth(drn_id)` - Retrieve DOB (via GetConsumer with field mask)
- `DeleteDateOfBirth(drn_id)` - Remove DOB
- `GetDeviceDataByAnonymousID(anonymous_id)` - Get device cookie preferences

**Data Flow:**
```
User.date_of_birth (in User model)
    ↓ (if feature flag enabled)
co-data gRPC Service → GetConsumer(drn_id) → date_of_birth
    ↓ (fallback if co-data returns nil)
user_birth_dates table → dob
```

**Recommendation:** Treat co-data as a **separate domain** that will need its own migration strategy. Identity Domain should coordinate with co-data to avoid duplicating date of birth storage.

---

### 3.4 Fraud & Security (Fraud/Risk Domain)

| Deliveroo Field | Data Type | Nullable | Active? | Proposed Owner | Pedregal Mapping | Notes |
|-----------------|-----------|----------|---------|----------------|------------------|-------|
| `fraud_activity` | BOOL | No | ✅ | Fraud | Fraud flag | User flagged for fraudulent activity |
| `comp_abuse_activity` | BOOL | No | ✅ | Fraud | Compensation abuse flag | User flagged for compensation abuse |
| `ctrl_bits` | INT | No | ❌ | Consumer | Bitmask for various flags | Deprecated flags (email subscriptions, etc.) |

**Related Tables:**
- `user_fraud_status` - Detailed fraud status history
- `user_ravelin_tags` - Ravelin fraud detection tags (compensation_abuse, account_takeover, account_hold)

### 3.5 Payment Information (Money/Bank Domain)

| Deliveroo Field | Data Type | Nullable | Active? | Proposed Owner | Pedregal Mapping | Notes |
|-----------------|-----------|----------|---------|----------------|------------------|-------|
| `stripe_customer_id` | TEXT | Yes | ✅ | Money | Stripe customer ID | Payment processor customer ID |
| `braintree_customer_id` | VARCHAR | Yes | ❌ | Money | Braintree customer ID | Legacy payment processor |
| `checkout_customer_id` | VARCHAR | Yes | ✅ | Money | Checkout.com customer ID | Payment processor customer ID |

**Note:** Actual payment tokens stored in `payment_tokens` table.

### 3.6 Preferences & Status (Consumer Domain)

| Deliveroo Field | Data Type | Nullable | Active? | Proposed Owner | Pedregal Mapping | Notes |
|-----------------|-----------|----------|---------|----------------|------------------|-------|
| `vip` | BOOL | Yes | ✅ | Consumer | VIP status flag | Premium customer indicator |
| `last_country_id` | INT | Yes | ✅ | Geo / Consumer | Last active country | Reference to `countries` table |
| `last_hydrated_at` | TIMESTAMP | Yes | ✅ | Consumer | Data sync timestamp | When user data was last synced |
| `checkout_type` | VARCHAR | Yes | ✅ | Consumer / Storefront | Checkout experience type | e.g., "white-label-api--*" |
| `is_admin` | BOOL | No | ✅ | Identity | Admin role flag | Internal user admin status |

**Related Tables:**
- `user_marketing_preferences` - Marketing consent and preferences
- `user_age_preferences` - Age verification preferences (drinking age, etc.)
- `user_order_preferences` - Order-related preferences
- `user_languages` - Language preferences per country

### 3.7 Deprecated/Unused Fields

| Deliveroo Field | Data Type | Status | Notes |
|-----------------|-----------|--------|-------|
| `phone` | VARCHAR | ❌ Removed | Replaced by `mobile` field |
| `max_delivery_m` | INT | ❌ Removed | Delivery radius (moved to other tables) |

---

## 4. Data Mapping: Deliveroo → Pedregal Domains

### 4.1 Reverse Mapping: Pedregal Identity ← Deliveroo Sources

**Note:** This mapping is based on the **actual Pedregal Identity Account schema** from `identity.account.v1.Account` and `identity.account.storage.Account` protobuf definitions.

| Pedregal Identity Field | Data Type | Required? | Deliveroo Source(s) | Transformation Logic | Notes |
|------------------------|-----------|-----------|---------------------|---------------------|-------|
| **Core Identifiers** |
| `account_id` | UUID | No (deprecated) | `users.drn_id` (UUID) or generate from `users.id` | If `drn_id` exists, use as-is; else generate UUID | Being deprecated in favor of account_prn |
| `account_prn` | String | ✅ Yes | **Computed:** `prn:v1:deliveroo:identity:user:{users.id}` | Generate PRN from legacy user ID | Primary identifier going forward |
| `namespace_id` | String | ✅ Yes | **Static:** `"deliveroo"` or `"wolt"` | Determine from source platform | Multi-tenant identifier |
| `legacy_fields` | Map<string,string> | ✅ Yes | **Map:** `{"user_id": users.id, "drn_id": users.drn_id, "short_drn_id": users.short_drn_id}` | Store all Deliveroo IDs for backward compatibility | Key-value pairs for legacy lookups |
| **Personal Information (PII)** |
| `email` | StringValue | No | `users.email` | Direct copy, lowercase, trim | Will be deprecated in favor of encrypted_email |
| `encrypted_email` | SensitiveString | No | `users.email` → encrypt | Encrypt using Pedregal encryption service | New encrypted storage |
| `first_name` | StringValue | No | `users.first_name` | Direct copy (already HTML-escaped) | Will be deprecated in favor of encrypted |
| `encrypted_first_name` | SensitiveString | No | `users.first_name` → encrypt | Encrypt using Pedregal encryption service | New encrypted storage |
| `last_name` | StringValue | No | `users.last_name` | Direct copy (already HTML-escaped) | Will be deprecated in favor of encrypted |
| `encrypted_last_name` | SensitiveString | No | `users.last_name` → encrypt | Encrypt using Pedregal encryption service | New encrypted storage |
| `phone_number` | StringValue | No | `users.mobile` | Direct copy (already E.164 formatted) | Will be deprecated in favor of encrypted |
| `encrypted_phone_number` | SensitiveString | No | `users.mobile` → encrypt | Encrypt using Pedregal encryption service | New encrypted storage |
| `username` | StringValue | No | **Not used in Deliveroo** | Leave empty | Optional field, not applicable |
| `encrypted_username` | SensitiveString | No | **Not used in Deliveroo** | Leave empty | Optional field, not applicable |
| **Status & Metadata** |
| `status` | AccountStatus enum | ✅ Yes | `users.status` | Map: `0 (enabled) → ACTIVE`, `1 (disabled) → DISABLED` | Status enumeration |
| `created_at` | Timestamp | ✅ Yes | `users.created_at` | Direct copy | Account creation timestamp |
| `updated_at` | Timestamp | No | `users.updated_at` or `created_at` if null | Direct copy with fallback | Last modification timestamp |
| **Encryption (KDK)** |
| `latest_wrapped_kdk` | WrappedKeyDerivationKey | No | **Generated:** Call Pedregal encryption service during migration | Create KDK for account encryption | Required for encrypted fields |

---

### 4.1.1 Fields NOT in Pedregal Identity (Must go to other domains or tables)

These Deliveroo fields **do not belong in Pedregal Identity Account**:

| Deliveroo Field | Type | Where It Should Go | Reason |
|----------------|------|-------------------|--------|
| **Password & Auth Secrets** |
| `password_digest` | String | **Separate table:** `account_secrets` in Pedregal Identity | Stored separately for security, NOT in main Account table |
| `password_reset_token` | String | **Transient state** or separate auth service | Not stored in Account |
| `password_reset_token_expires_at` | Timestamp | **Transient state** or separate auth service | Not stored in Account |
| `email_ctrl_token` | String | **Email/Marketing service** | Not core identity data |
| **Date of Birth** |
| `date_of_birth` | Date | **Co-Data service** (already migrated there) | Deliveroo already using co-data for DOB |
| **Fraud & Security** |
| `fraud_activity` | Boolean | **Fraud/Risk Domain** | Not Identity responsibility |
| `comp_abuse_activity` | Boolean | **Fraud/Risk Domain** | Not Identity responsibility |
| `ravelin_id` | String | **Fraud/Risk Domain** | External fraud service integration |
| **Payment** |
| `stripe_customer_id` | String | **Money/Bank Domain** | Payment processor data |
| `checkout_customer_id` | String | **Money/Bank Domain** | Payment processor data |
| `braintree_customer_id` | String | **Money/Bank Domain** (deprecated) | Legacy payment processor |
| **Consumer Preferences** |
| `vip` | Boolean | **Consumer Domain** | Loyalty/subscription status |
| `last_country_id` | Integer | **Geo Domain** or **Consumer Domain** | User preference, not identity |
| `drn_market` | String (2 char) | **Consumer Domain** or store in `legacy_fields` | Market preference |
| `is_admin` | Boolean | **Authorization/Permissions service** | Role/permission, not identity |
| **Referral & Marketing** |
| `referrer_code` | String | **Consumer Domain** or **Marketing Domain** | Referral tracking |
| `is_invitee` | Boolean | **Consumer Domain** or **Marketing Domain** | Referral tracking |
| **Session (Deprecated)** |
| `session_token` | String | **Skip** | Deprecated, using JWT now |
| `session_expires_at` | Timestamp | **Skip** | Deprecated |
| **Phone Verification** |
| `mobile_verified_at` | Timestamp | **Separate verification service** or **Consumer Domain** | Verification state, not core identity |
| **Deliveroo-specific IDs** |
| `short_drn_id` | Integer | **Store in `legacy_fields` map** | Backward compatibility only |

---

### 4.2 Data Source Priority & Fallback Chain

For fields with multiple possible sources, use this priority order:

#### **Date of Birth:**
```
1. Co-Data gRPC Service (primary)
   └─> GrpcServices::CoDataService.instance.get_date_of_birth(drn_id)
2. user_birth_dates table (fallback 1)
   └─> SELECT dob FROM user_birth_dates WHERE user_id = users.id
3. users.date_of_birth field (fallback 2, deprecated)
   └─> users.date_of_birth
```

#### **Account ID:**
```
1. users.drn_id (preferred for Deliveroo)
   └─> Use directly as Pedregal account_id
2. Generate from users.id (if drn_id is null)
   └─> Create UUID or use namespace-prefixed ID
```

#### **Phone Country Code:**
```
1. Parse from users.mobile (E.164 format)
   └─> Extract leading +XX from "+44xxxxxxxxx"
2. Infer from users.last_country_id
   └─> Lookup default calling code from countries table
```

#### **Locale:**
```
1. user_languages table (primary)
   └─> SELECT locale FROM user_languages WHERE user_id = users.id AND country_id = users.last_country_id
2. Default from market_code (fallback)
   └─> Map drn_market to default locale (e.g., 'GB' → 'en-GB')
```

---

### 4.3 Forward Mapping: Deliveroo → Pedregal Identity (Original)

The Identity Domain in Pedregal should own authentication and core account identification data:

| Pedregal Identity Field | Deliveroo Source | Transformation Notes |
|-------------------------|------------------|----------------------|
| `account_id` | `users.id` or `users.drn_id` | Map to Pedregal Account with Deliveroo namespace |
| `email` | `users.email` | Direct mapping |
| `phone_number` | `users.mobile` | E.164 formatted |
| `phone_verified_at` | `users.mobile_verified_at` | Timestamp of verification |
| `password_hash` | `users.password_digest` | Bcrypt hash |
| `first_name` | `users.first_name` | Sanitized/escaped |
| `last_name` | `users.last_name` | Sanitized/escaped |
| `preferred_name` | `users.preferred_name` | Display name |
| `created_at` | `users.created_at` | Account creation |
| `updated_at` | `users.updated_at` | Last modification |
| `status` | `users.status` | enabled (0) / disabled (1) |
| `namespace` | Static: `"Deliveroo"` or `"Wolt"` | Per RFC Pedregal Consumer |
| `market_code` | `users.drn_market` | ISO Alpha-2 country code |
| `employee_flag` | Computed from `users.email` | Email domain = `@deliveroo.com` |
| `date_of_birth` | Co-Data service or `user_birth_dates.dob` | Age verification, check co-data first |

### 4.4 Consumer Domain Mapping (Pedregal Consumer)

Pedregal Consumer is intentionally minimal:

| Pedregal Consumer Field | Deliveroo Source | Notes |
|------------------------|------------------|-------|
| `consumer_id` | **NEW**: UUIDv5 or preserve `users.id` | Unique consumer identifier |
| `actor_id` | Derived from `account_id` | Identity's account_id |
| `persona` | Static: `"default"` | Wolt has no "experience" concept |
| `created_at` | `users.created_at` | Consumer creation timestamp |

**Note:** Deliveroo does not have the concept of "experiences" like DoorDash. All Deliveroo consumers map to `persona = "default"`.

### 4.5 Other Domain Mappings

| Domain | Deliveroo Fields | Target |
|--------|------------------|--------|
| **Fraud/Risk** | `fraud_activity`, `comp_abuse_activity`, `ravelin_id`, `user_fraud_status.*`, `user_ravelin_tags.*` | Fraud detection service |
| **Money** | `stripe_customer_id`, `braintree_customer_id`, `checkout_customer_id`, `payment_tokens.*` | Payment processor references |
| **Geo** | `last_country_id`, `user_address.*` | Location/address services |
| **Platform** | `user_devices.*`, marketing push preferences | Device management, notifications |
| **Consumer Profile** | `vip`, `user_marketing_preferences.*`, `user_languages.*`, `user_order_preferences.*` | Consumer-specific preferences |

---

## 5. Identity Mapping Layer (IML) Integration

### 5.1 IML Requirements

Per [RFC Domain ID Handling in Pedregal](https://docs.google.com/document/d/1Q4hdshz2xHk7SGS94cm8ZcxH_DDXxKEOrmPCGwsmsic/edit):

1. **IML Entry Creation**: For each Deliveroo user, create an IML mapping:
   - `account_id` (Pedregal Identity)
   - `consumer_id` (Pedregal Consumer)
   - `namespace`: `"Deliveroo"` or `"Wolt"`

2. **Lookup Patterns**:
   - `GetConsumerByAccountId(account_id)` → returns `consumer_id`
   - `GetAccountByConsumerId(consumer_id)` → returns `account_id`

### 5.2 Data Flow

```
Deliveroo User (users.id, users.drn_id)
    ↓
Pedregal Account Creation (Identity Domain)
    ├─ account_id (generated or mapped from drn_id)
    ├─ email, phone, names, password_hash
    ├─ namespace = "Deliveroo"
    └─ created_at, status
    ↓
Pedregal Consumer Creation (Consumer Domain)
    ├─ consumer_id (UUIDv5 or preserved users.id)
    ├─ actor_id = account_id
    ├─ persona = "default"
    └─ created_at
    ↓
IML Entry Creation (Identity Domain)
    ├─ account_id ↔ consumer_id mapping
    └─ namespace = "Deliveroo"
```

---

## 6. Migration Strategy

### 6.1 Phased Approach

Following the RFC Pedregal Consumer migration milestones:

#### **Phase 1: Pedregal Account & Identity Setup**
- **Goal:** Create Pedregal Accounts for all Deliveroo users
- **Scope:**
  - Backfill Identity Domain with account data from `users` table
  - Map `users.id` or `users.drn_id` to Pedregal `account_id`
  - Preserve authentication credentials (email, password_digest, phone)
  - Set `namespace = "Deliveroo"`

#### **Phase 2: Pedregal Consumer Creation**
- **Goal:** Create minimal Pedregal Consumer entities
- **Scope:**
  - Generate `consumer_id` (preserve `users.id` as string or generate UUIDv5)
  - Link `consumer_id` to `actor_id` (account_id)
  - Set `persona = "default"` for all Deliveroo users
  - Backfill `consumers_by_id` and `consumers_by_actor_id` Taulu tables

#### **Phase 3: IML Entry Population**
- **Goal:** Establish account ↔ consumer mappings
- **Scope:**
  - Create IML entries for all Deliveroo users
  - Enable lookup APIs: `GetConsumerByAccountId`, `GetAccountByConsumerId`

#### **Phase 4: Async Write (Double Write)**
- **Goal:** Real-time replication of new accounts
- **Scope:**
  - On Deliveroo user creation → create Pedregal Account + Consumer + IML entry
  - On Deliveroo user update → update Pedregal Account (Identity Domain fields only)

#### **Phase 5: Read Cutover**
- **Goal:** Services read from Pedregal instead of Deliveroo `users` table
- **Scope:**
  - Update authentication services to use Pedregal Identity APIs
  - Update consumer services to use Pedregal Consumer APIs via IML

#### **Phase 6: Write Cutover**
- **Goal:** Pedregal becomes source of truth
- **Scope:**
  - New accounts created in Pedregal first
  - Async replicate to Deliveroo for legacy compatibility (if needed)

#### **Phase 7: Deliveroo `users` Table Deprecation**
- **Goal:** Decommission legacy table
- **Scope:**
  - Remove writes to Deliveroo `users` table
  - Archive or retire table after transition period

### 6.2 Data Volume Estimates

**Deliveroo User Base (est.):**
- Total users: ~10-15 million (estimated, needs confirmation)
- Active users (last 90 days): TBD
- Includes: UK, France, Belgium, Ireland, Netherlands, Italy, Australia, Singapore, UAE, Hong Kong, Kuwait

**Wolt User Base (est.):**
- Total users: ~15-20 million (estimated, needs confirmation)
- Active users (last 90 days): TBD
- Includes: Finland, Sweden, Norway, Denmark, Estonia, Latvia, Lithuania, Poland, Czech Republic, Germany, Israel, Japan, Kazakhstan, Georgia

**Combined Pedregal Migration:**
- Total accounts: ~25-35 million
- Backfill strategy: Batch processing (10k-100k users per batch)

---

## 7. Technical Considerations

### 7.1 DRN to Account ID Mapping

**Challenge:** Deliveroo uses DRNs (Deliveroo Resource Names) as primary identifiers:
- Format: `drn:roo:user:{market}:{uuid}`
- Example: `drn:roo:user:uk:550e8400-e29b-41d4-a716-446655440000`

**Options:**
1. **Preserve DRN as Account ID:** Use `users.drn_id` directly as Pedregal `account_id` (requires UUID support)
2. **Generate New Account ID:** Create new Pedregal-native UUIDs, maintain mapping table
3. **Use Integer ID:** Preserve `users.id` as string-encoded `account_id` (simpler but less semantic)

**Recommendation:** Option 1 (Preserve DRN as Account ID) for consistency with Deliveroo ecosystem.

### 7.2 Consumer ID Strategy

**Wolt Note:** Per RFC, "Consumer_id is the same as Wolt user_id per tenant. These tenants are being onboarded as Namespaces on accounts."

**For Deliveroo:**
- **Option A:** Preserve `users.id` as `consumer_id` (string format)
- **Option B:** Generate new UUIDv5 based on `drn_id`

**Recommendation:** Option A (Preserve `users.id`) to maintain backward compatibility with existing Deliveroo services.

### 7.3 Namespace Handling

Deliveroo users should be assigned:
- **Namespace:** `"Deliveroo"` (distinct from `"DoorDash"` and `"Wolt"`)
- **Market:** Stored in `drn_market` field (e.g., `"uk"`, `"fr"`, `"be"`)

### 7.4 Authentication Migration

**Current:** Deliveroo uses `bcrypt` for password hashing
**Target:** Pedregal Identity should support bcrypt hashes (no re-hashing required)

**Key Fields:**
- `password_digest` → `password_hash` in Identity
- `password_reset_token` + `password_reset_token_expires_at` → Identity password reset flow

### 7.5 Phone Number Handling

Deliveroo stores phone numbers in two places:
1. `users.mobile` (E.164 formatted, primary)
2. `user_phone` table (country-specific formatting, legacy)

**Migration:** Use `users.mobile` as primary source for Pedregal Identity `phone_number` field.

### 7.6 Guest Users

**Challenge:** Deliveroo supports guest checkout (users without accounts)
**Current State:** Guest users may have minimal records or no `users` table entry

**Recommendation:** Align with [Pedregal Guest proposal](https://docs.google.com/document/d/10YkCO3YyxtrZkRUbTgnhR5z3-wrGQaKcjfuU9O47y5w/edit) - guest identity managed by Identity Domain, not Consumer Domain.

---

## 8. Data Quality & Validation

### 8.1 Pre-Migration Validation

Before migrating Deliveroo data to Pedregal:

| Validation Check | Query/Logic | Expected Result |
|------------------|-------------|-----------------|
| Unique `id` | `SELECT COUNT(DISTINCT id) FROM users` | Matches total row count |
| Unique `drn_id` | `SELECT COUNT(DISTINCT drn_id) FROM users WHERE drn_id IS NOT NULL` | No duplicates |
| Email format | `SELECT COUNT(*) FROM users WHERE email !~ '^[a-z0-9._%''+-]+@[a-z0-9.-]+\\.[a-z]{2,63}$'` | 0 invalid emails |
| E.164 phone format | `SELECT COUNT(*) FROM users WHERE mobile IS NOT NULL AND is_valid_e164_phone_number = false` | Track invalid phones |
| Status values | `SELECT DISTINCT status FROM users` | Only 0 (enabled) or 1 (disabled) |
| Orphaned references | Check foreign keys to `users.id` in other tables | All valid |
| Created_at timestamps | `SELECT COUNT(*) FROM users WHERE created_at IS NULL` | 0 null timestamps |

### 8.2 Post-Migration Validation

After migrating to Pedregal:

| Validation Check | Logic | Expected Result |
|------------------|-------|-----------------|
| Account count | Pedregal Account count = Deliveroo `users` count (active) | Match |
| Consumer count | Pedregal Consumer count = Deliveroo `users` count (active) | Match |
| IML entries | IML entry count = Deliveroo `users` count (active) | Match |
| Email uniqueness | No duplicate emails in Pedregal Accounts (per namespace) | Unique |
| DRN mapping | All Deliveroo `drn_id` values present in Pedregal | 100% coverage |

---

## 9. Risks & Mitigation

### 9.1 Key Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Data loss during migration** | High | Low | Comprehensive backups, dry-run migrations, rollback plan |
| **Authentication downtime** | High | Medium | Blue-green deployment, gradual cutover, fallback to legacy |
| **Consumer ID conflicts** | Medium | Low | Pre-migration uniqueness validation, conflict resolution logic |
| **Performance degradation** | Medium | Medium | Load testing, query optimization, caching strategy |
| **Cross-domain data inconsistency** | Medium | Medium | Transactional writes where possible, eventual consistency monitoring |

### 9.2 Rollback Strategy

Each migration phase should have a rollback plan:
- **Phase 1-3 (Backfill):** Can be re-run, Deliveroo remains source of truth
- **Phase 4 (Async Write):** Disable double-write, continue using Deliveroo
- **Phase 5 (Read Cutover):** Feature flag to revert reads to Deliveroo
- **Phase 6+ (Write Cutover):** More complex; requires data reconciliation

---

## 10. Success Criteria

### 10.1 Functional Requirements

- [ ] All active Deliveroo users have corresponding Pedregal Accounts
- [ ] All active Deliveroo users have corresponding Pedregal Consumers
- [ ] IML provides bidirectional account ↔ consumer lookups
- [ ] Authentication services can authenticate users via Pedregal Identity
- [ ] Consumer services can fetch consumer data via Pedregal Consumer APIs
- [ ] No user-visible service disruptions during migration

### 10.2 Performance Requirements

- [ ] Account creation: < 100ms p99
- [ ] IML lookup: < 50ms p99
- [ ] Consumer fetch by ID: < 50ms p99
- [ ] Authentication: < 200ms p99
- [ ] Backfill rate: > 10,000 users/minute

### 10.3 Data Quality Requirements

- [ ] 100% of active users migrated
- [ ] 0% data loss (all critical fields preserved)
- [ ] < 0.01% data corruption/errors
- [ ] Audit trail of all migrations

---

## 11. Open Questions

1. **Exact user count:** What is the current active user count in Deliveroo?
2. **Consumer ID strategy:** Should we preserve `users.id` or generate new UUIDs for `consumer_id`?
3. **DRN format:** Should `drn_id` be the primary `account_id` in Pedregal, or create a new UUID?
4. **Guest users:** How are Deliveroo guest users currently tracked? Do they have `users` table entries?
5. **Market/namespace:** Should Deliveroo have a single namespace, or separate namespaces per market (UK, FR, etc.)?
6. **Deprecated fields:** Can we safely ignore deprecated fields like `ctrl_bits`, `braintree_customer_id`?
7. **Fraud data ownership:** Should fraud-related fields remain in Identity or move to dedicated Fraud Domain?
8. **Date of birth storage:** Should DOB stay in Identity for compliance, or move to Consumer Domain?
9. **Subscription data:** How should Deliveroo Plus subscriptions be handled? (Likely separate domain)
11. **Cross-platform unification:** Should Deliveroo and DoorDash accounts ever merge for the same user?
12. **Co-data consumer profile data:** What other consumer attributes does co-data manage beyond DOB? Do they need to be migrated?

---

## 12. Next Steps

### Immediate Actions:
1. **Data Audit:** Run validation queries on Deliveroo `users` table to assess data quality
2. **Stakeholder Alignment:** Review this document with Identity, Consumer, and Deliveroo platform teams
3. **Technical Design:** Create detailed technical design for backfill pipeline
4. **Capacity Planning:** Estimate infrastructure needs for 25-35 million account migration

### Short-Term (Q1 2026):
- Finalize consumer_id and account_id strategies
- Build backfill pipeline for Pedregal Account creation
- Implement IML integration for Deliveroo namespace

### Medium-Term (Q2-Q3 2026):
- Execute phased migration per milestones
- Monitor and iterate on performance
- Gradual read/write cutover

### Long-Term (Q4 2026+):
- Deprecate Deliveroo `users` table (Identity fields)
- Segment remaining fields to appropriate domains
- Full Pedregal cutover for Deliveroo platform

---

## 13. Appendix

### A. Related Tables in Deliveroo

**Identity-Adjacent:**
- `user_phone` - Phone number records with country context
- `user_identity` - OAuth/social login integrations
- `security_principal` - Internal security principal mappings
- `user_acceptances` - Legal document acceptances (TOS, privacy policy)

**Co-Data Service (Separate Microservice):**
- **NOT a table** - This is a separate gRPC service that manages specific consumer attributes
- Stores: date of birth, cookie consent, advertising preferences
- Accessed via: `Consumer::CoData::V1::ConsumerDataService` gRPC stub
- Indexed by: `drn_id` (Deliveroo Resource Name)
- **Migration Note:** This service may need separate integration with Pedregal or coordination with Identity Domain

**Consumer Profile:**
- `user_marketing_preferences` - Marketing consent
- `user_age_preferences` - Age verification preferences
- `user_order_preferences` - Order-related preferences
- `user_languages` - Language preferences per country
- `subscriptions` - Deliveroo Plus subscriptions

**Fraud & Security:**
- `user_fraud_status` - Fraud status history
- `user_ravelin_tags` - Ravelin fraud tags

**Payments:**
- `payment_tokens` - Stored payment methods

**Misc:**
- `user_address` - Delivery addresses
- `user_devices` - Registered devices
- `user_credits` - Account credits/balance
- `user_debits` - Account debits

### B. Deliveroo vs. DoorDash Field Comparison

| Deliveroo `users` Field | DoorDash COPS `consumer` Equivalent | Notes |
|-------------------------|-------------------------------------|-------|
| `id` | `id` | Both use INT8 primary keys |
| `drn_id` | N/A | Deliveroo-specific DRN concept |
| `email` | `sanitized_email` | Deliveroo stores raw email |
| `mobile` | N/A | DoorDash stores phone separately |
| `created_at` | `created_at` | Same concept |
| `status` | `forgotten_at` (implicit disable) | Different approach to deactivation |
| `vip` | `is_vip` | Same concept |
| N/A | `user_id` | DoorDash separates User (Identity) from Consumer |
| N/A | `experience` | Deliveroo has no "experience" concept |
| N/A | `tenant_id` | Deliveroo uses `drn_market` instead |

### C. Glossary

- **DRN (Deliveroo Resource Name):** Unique identifier format used across Deliveroo platform (e.g., `drn:roo:user:uk:uuid`)
- **IML (Identity Mapping Layer):** Pedregal service that maps `account_id` ↔ `consumer_id` across domains
- **Namespace:** Logical partition in Pedregal Identity (e.g., "DoorDash", "Wolt", "Deliveroo")
- **Persona:** Consumer role in Pedregal (e.g., "default", "caviar") - Deliveroo only uses "default"
- **Actor ID:** Pedregal term for the entity (account or guest) associated with a consumer
- **COPS:** Consumer Profile Service (DoorDash legacy, being deprecated)
- **Taulu:** DoorDash's distributed database system used by Pedregal

---

## 14. Approvals

| Role | Name | Status | Date | Notes |
|------|------|--------|------|-------|
| Identity Domain Lead | TBD | Pending | | |
| Consumer Domain Lead | TBD | Pending | | |
| Deliveroo Platform Lead | TBD | Pending | | |
| Pedregal Architecture | TBD | Pending | | |
| Data Platform | TBD | Pending | | |


