# Deliveroo Login/Signup Migration Plan to Pedregal (2026)

## Executive Summary
This plan outlines the phased migration of Deliveroo's authentication stack from the legacy split architecture (`orderweb`/`co-accounts`) to the unified **Pedregal Identity Platform**. The strategy minimizes risk by starting with data synchronization and low-risk flows before migrating high-volume consumer traffic.

---

## Phase 1: Foundation & Data Synchronization (Q1 2026)
**Goal:** Establish data parity between Deliveroo and Pedregal without shifting auth traffic. Unlock phase 1 bundles' launch on the Home feed.

### Key Deliverables
1.  **Dual-Write/Sync Mechanism**: Implement a sync job (or event consumer via Franz/Kafka) to replicate Deliveroo user data (Email, Phone, Profile) to Pedregal's **Identity Service** (User Entity).
    *   *Source:* `orderweb` (User table) & `co-accounts`.
    *   *Target:* Pedregal Account/Identity Service.
2.  **Read-Only Integration**: Update Deliveroo "Discovery" service to read user profile data from Pedregal for the Home feed.

### Wins
*   **"Good Morning, [Name]"**: Enables personalization on the Home feed by leveraging the new, faster Account service read path.
*   **De-risked following Migration**: Validates data accuracy and latency before critical login flows depend on it.

### Dependencies & Risks
*   **Dependency**: Pedregal `Identity Service` must support Deliveroo namespace/tenancy.
*   **Risk**: Data inconsistency between `orderweb` and Pedregal. *Mitigation: Implement reconciliation jobs.*

**Resourcing:** 1-2 Backend Engineers.

---

## Phase 2: Credential Login & Mobile SDK Integration (Q2 2026)
**Goal:** Migrate Email/Password flows (~25% traffic) and integrate the shared Mobile Login SDK.

### Key Deliverables
1.  **Mobile SDK Integration**: Integrate **DoorDash Mobile Login SDK** into Deliveroo iOS/Android apps.
    *   Configure SDK for "Internal" mode to talk to Pedregal.
    *   *Reference:* `Intent to RFC_ iOS Login SDK.md`.
2.  **Credential Auth Migration**: Route password login requests from the SDK directly to Pedregal's **Identity Service**.
    *   *Legacy Path:* Deliveroo Apps -> `orderweb` `POST /api/auth/login`.
    *   *New Path:* Deliveroo Apps' SDK -> UG/PG -> Pedregal Identity GraphRunner node

### Business Win
*   **Phone-First Login**: Enable "Phone Number + OTP" login flow (supported natively by Pedregal SDK), simplifying entry for mobile-first users.
*   **Reduced Latency**: Removing `orderweb` monolith hops for auth checks.

### Dependencies
*   **Dependency**: DoorDash Mobile Login SDK readiness (M5 Milestone: Consumer + one additional app). The current plan of the SDK is to be production ready by 4/30/2026 (https://docs.google.com/document/d/1_DT_4CaVNcRyieuj5wamn8KiW-6lE0V-gYc3G2qVGNc/edit?tab=t.0)
*   **Dependency (Critical)**: services must be available in Pedregal:
    *   **Identity Service**: Credential validation. Return a token that is authorized for both legacy Roo and Pedregal stack. 
    *   **Risk Gateway**: Fraud detection, User blocking/SARC (Systematic Account Risk Control) (replaces `Ravelin` and `StickyNotes` in `orderweb`). *Ideally: Native Pedregal Risk Gateway. If not ready: Shim from Pedregal → Ravelin.*

**Resourcing:** 2 Backend, 2 Mobile (1 iOS, 1 Android).

---

## Phase 3: Social Login & High Volume Migration (Q3 2026)
**Goal:** Migrate Social Login (~75% traffic)

### Key Deliverables
1.  **Social Auth Migration**: Switch Apple/Google/Facebook flows to use Pedregal's **Social Auth Service**.
    *   The Mobile SDK handles the OAuth handshake and token exchange with Pedregal.

### Business Win
*   **Maintenance Reduction**: Deprecate complex social auth handling code in `orderweb` and `co-accounts`.

### Dependencies
*   **Dependency (Critical Path)**: Pedregal **Social Auth Service** milestone (Target: Q2 2026 on Platform roadmap).
*   **Dependency**: Supporting services must be available in Pedregal:
    *   **Identity Service**: Federated identity storage and OAuth token validation.
    *   **Risk Gateway**: Fraud scoring for social login attempts. *Ideally: Native. If not ready: Shim to Ravelin.*
    *   **StickyNotes**: User blocking checks during social signup. *Ideally: Native. If not ready: Shim.*
    *   **CATS/Session Service**: Session token creation and management.
    *   **Postal Service**: Email notifications (welcome emails, verification).

### Risks
*   **Risk**: Third-party provider configuration (Apple/Google/Facebook app IDs) migration complexity - requires coordination with Apple/Google developer accounts.
*   **Risk**: Social identity linking - existing users with social accounts in `orderweb` must be properly linked in Pedregal Identity.

**Resourcing:** 2 Backend, 2 Mobile.

---

## Phase 4: Verification, Cleanup & Advanced Features (Q4 2026)
**Goal:** Complete migration of auxiliary flows and decommission legacy code.

### Key Deliverables
1.  **Verification Migration**: Move remaining SMS/Email verification logic from `Sphinx`/`orderweb` to Pedregal's **Postal Service**.
2.  **Legacy Cleanup**: Remove auth logic from `orderweb` and deprecate `co-accounts` endpoints (`/check-email`, `/request-login`).

### Business Win
*   **Passkey Support**: Cutting-edge, passwordless security for users.
*   **Tech Debt Removal**: Decommissioning `co-accounts` auth logic reduces on-call burden.

### Dependencies & Risks
*   **Dependency**: Full parity of Pedregal's notification system with `Sphinx` capabilities.

**Resourcing:** 2 Backend, 1 Mobile.

---

## Summary of Risks & Dependencies

| Risk / Dependency | Impact | Mitigation Owner |
| :--- | :--- | :--- |
| **Pedregal Social Auth Delay** | Blocks Phase 3 (~75% traffic). | Identity Platform (Chunpi Chang) |
| **UI/UX Divergence** | Deliveroo app loses "native feel" if SDK forces webviews. | Mobile Platform / Design Systems |
| **Data Sync Lag** | "Split brain" issues during Phase 1-2. | Deliveroo Backend Team |
| **Session Interop** | Legacy backend services rejecting Pedregal tokens. | Core Infrastructure Team |

## Next Steps
1.  **Feasibility Spike (Q1)**: POC syncing 1,000 users to Pedregal and reading them back in Discovery.
2.  **SDK Audit**: Evaluate the DoorDash Mobile Login SDK against Deliveroo UI requirements.
3.  **Sign-off**: Review this timeline with Pedregal TPMs to ensure alignment with their Q2/Q3 delivery dates.

