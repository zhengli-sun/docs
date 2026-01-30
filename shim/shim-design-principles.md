---
description: Guiding principles for building shims between Pedregal and legacy systems (DoorDash, Wolt)
last_updated: 2026-01-30
---

# Shim Design Principles for Pedregal

Guiding principles for building shims that bridge Pedregal to legacy services.

---

## Principle 0: Evaluate Shim vs. Native — Shims Are a Tradeoff, Not a Default

### Principle

Before implementing a shim, carefully evaluate against building natively in Pedregal. Shims are gap-closing solutions, not destinations. If timeline permits, native implementation is always preferred.

### Why

| Shim Costs | Shim Benefits |
|------------|---------------|
| Extra network hop (5-50ms) | Fast time-to-market |
| Tech debt accumulation | Incremental migration |
| Dual maintenance burden | Risk mitigation (fallback) |
| Debugging across boundaries | Parallel development |

### Example

**Scenario**: Feed node needs consumer subscription tier information.

| Approach | Timeline | Trade-off |
|----------|----------|-----------|
| Shim to DashPass service | 1 week | Fast, but adds latency + coupling |
| Native in Subscription node | 3 weeks | Clean domain model, no coupling |

**Decision**: If Feed launch is blocked and Subscription node is 2+ months out, shim is justified. Otherwise, build native.

### Visualization

```mermaid
flowchart TD
    A[New Feature Requirement] --> B{Is legacy system<br/>retiring soon?}
    B -->|Yes, <6 months| C{Is native build<br/>feasible in timeline?}
    B -->|No| D{Would native take<br/>>3x longer?}
    
    C -->|Yes| E[Build Native]
    C -->|No| F[Build Shim]
    D -->|Yes| G[Build Shim with<br/>Deprecation Plan]
    D -->|No| E
    
    F --> H[Document sunset date]
    G --> H
    
    style E fill:#90EE90
    style F fill:#FFE4B5
    style G fill:#FFE4B5
    style H fill:#FFB6C1
```

---

## Principle 1: Place Shims at Leaf Nodes — Domain Isolation First

### Principle

Shims should be implemented in the domain node that owns the data, not in the calling node. This preserves domain boundaries and decouples callers from internal migration decisions.

### Why

| Shim in Caller (Bad) | Shim in Owner (Good) |
|----------------------|----------------------|
| Coupling spreads to all callers | Single point of change |
| Migration requires N caller changes | Clean interfaces for callers |
| Business logic mixes with integration | Domain team owns complexity |
| Each caller needs legacy mocking | Gradual rollout possible |

### Example

**Scenario**: Feed node needs fee breakdown for restaurants.

- **❌ Anti-pattern**: Feed node branches on brand and calls DoorDash Quote Adjuster or Wolt Pricing directly
- **✅ Correct**: Feed calls Fee node → Fee node internally shims to legacy based on brand

### Visualization

```mermaid
flowchart TB
    subgraph "❌ Anti-pattern: Shim in Caller"
        F1[Feed Node] -->|"brand=DD"| DD1[DoorDash Legacy]
        F1 -->|"brand=Wolt"| W1[Wolt Legacy]
        
        style F1 fill:#FFB6C1
    end
```

```mermaid
flowchart TB
    subgraph "✅ Correct: Shim in Leaf Node"
        F2[Feed Node] -->|GetConsumerFees| FEE[Fee Node]
        FEE -->|internal| SHIM[Fee Shim Layer]
        SHIM -.-> DD2[DoorDash Legacy]
        SHIM -.-> W2[Wolt Legacy]
        
        style F2 fill:#90EE90
        style FEE fill:#90EE90
        style SHIM fill:#FFE4B5
    end
```

```mermaid
flowchart LR
    subgraph Callers["Calling Domains (Clean)"]
        FEED[Feed Node]
        CART[Cart Node]
        CHECKOUT[Checkout Node]
    end
    
    subgraph Owners["Owning Domains (Shim Here)"]
        FEE[Fee Node]
        STORE[Store Node]
        GEO[Geography Node]
    end
    
    subgraph Legacy["Legacy Systems (Hidden)"]
        QA[Quote Adjuster]
        MX[MX Service]
        MAPS[Legacy Maps]
    end
    
    FEED --> FEE
    FEED --> STORE
    CART --> FEE
    CHECKOUT --> FEE
    
    FEE -.->|shim| QA
    STORE -.->|shim| MX
    GEO -.->|shim| MAPS
    
    style Callers fill:#E8F5E9
    style Owners fill:#FFF3E0
    style Legacy fill:#FFEBEE
```

---

## Principle 2: Shims Own Entity Translation — PRNs In, Legacy IDs Out

### Principle

Shims are responsible for all conversions between Pedregal-native entities (PRNs, Pedregal geo, unified models) and legacy entities (numeric IDs, legacy coordinates, brand-specific formats). Translation logic must be encapsulated within the shim, never leaked to callers.

### Why

| Benefit | Impact |
|---------|--------|
| Single source of truth | One place to fix mapping bugs |
| Type safety | Compiler catches mismatches at shim boundary |
| Testability | Unit test translations independently |
| Audit trail | Easy to find all legacy touchpoints |

### Common Translation Patterns

| Pedregal Entity | Legacy Entity | Translation |
|-----------------|---------------|-------------|
| `prn:consumer:dd:12345` | `consumer_id: 12345` | Extract numeric suffix |
| `prn:store:wolt:abc-123` | `venue_id: "abc-123"` | Extract Wolt venue slug |
| `Location { lat, lng, h3_index }` | `{"latitude": x, "longitude": y}` | Project to legacy format |
| `Money { amount_micros, currency }` | `{"cents": 1234}` | Convert micros to cents |
| `FulfillmentType.DELIVERY` | `delivery_type: 1` | Enum to integer mapping |

### Visualization

```mermaid
sequenceDiagram
    participant C as Caller Node
    participant S as Shim
    participant L as Legacy Service
    
    C->>S: GetList(prn:consumer:dd:123)
    
    Note over S: Translation: PRN → Legacy ID
    S->>S: extractID() → 123
    
    S->>L: GetConsumerLists(consumer_id=123)
    L-->>S: LegacyListResponse
    
    Note over S: Translation: Legacy → Pedregal
    S->>S: mapToProto()
    
    S-->>C: List Proto (Pedregal native)
```

---

## Principle 3: Defer Performance Optimization — Boundaries Over Latency (For Now)

### Principle

As of January 2026, establishing correct domain boundaries and shim architecture takes precedence over latency optimization. Performance solutions exist and can be applied incrementally as we approach production readiness.

### Why

**Current phase priorities**:
1. ✅ Correct domain modeling
2. ✅ Clean shim boundaries
3. ✅ Maintainable code structure
4. ⏸️ Sub-10ms latency (deferred)

**Why deferral is safe**:

| Concern | Available Solution | When to Apply |
|---------|-------------------|---------------|
| Extra network hop latency | Request-level caching | Pre-production |
| Repeated legacy calls | Talau (Graph Runner cache) | When APIs stabilize |
| Legacy system slowness | Caching in legacy layer | Production hardening |

**Why premature optimization hurts**:
- Caches obscure bugs during development
- Optimized code is harder to refactor
- Cache invalidation adds complexity

### Visualization

```mermaid
timeline
    title Optimization Timeline
    
    section Current Phase (Q1 2026)
        Focus: Domain boundaries, Shim correctness
        Acceptable: 50-100ms latency
    
    section Pre-Production (Q2 2026)
        Add: Request-level caching, Batch APIs
        Target: 30-50ms latency
    
    section Production (Q3+ 2026)
        Add: Talau caching, Connection pooling
        Target: < 20ms latency
```

```mermaid
flowchart LR
    subgraph "Now: Clean Boundaries"
        F1[Feed] --> FEE1[Fee Node] --> SHIM1[Shim] --> LEG1[Legacy]
    end
    
    subgraph "Later: Optimized"
        F2[Feed] --> CACHE[Cache] --> FEE2[Fee Node] --> TALAU[Talau] --> SHIM2[Shim] --> LEG2[Legacy]
    end
    
    style CACHE fill:#E3F2FD
    style TALAU fill:#E3F2FD
    style SHIM1 fill:#FFE4B5
    style SHIM2 fill:#FFE4B5
```

---

## Quick Reference

| Principle | One-Liner | Key Question |
|-----------|-----------|--------------|
| **0. Evaluate First** | Shims are tradeoffs, not defaults | Can we build native in time? |
| **1. Leaf Nodes** | Shim where data lives, not where it's used | Who owns this data? |
| **2. Entity Translation** | PRNs in, legacy IDs hidden | Is translation encapsulated? |
| **3. Boundaries First** | Correct now, fast later | Are we optimizing prematurely? |

---

## Related Documentation

- [Shim Examples in Pedregal](./Pedregal-shim-examples.md)
- [Sandbox Testing Guide](./sandbox-testing-guide.md)
