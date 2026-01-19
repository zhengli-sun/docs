# Argo Search Framework Guide

A comprehensive guide to understanding the Argo Search framework used by Feed in Pedregal for content retrieval, ranking, and discovery.

---

## Table of Contents
1. [What is Argo Search?](#what-is-argo-search)
2. [Architecture Overview](#architecture-overview)
3. [Key Concepts](#key-concepts)
4. [Request Structure](#request-structure)
5. [Namespaces & Pipelines](#namespaces--pipelines)
6. [Integration with Feed](#integration-with-feed)
7. [Query Building](#query-building)
8. [Response Handling](#response-handling)
9. [Common Patterns](#common-patterns)

---

## What is Argo Search?

**Argo Search** is DoorDash's internal search and retrieval infrastructure. It provides:

- **Distributed Search**: Scalable search across millions of documents (stores, items, collections)
- **Configurable Pipelines**: ML-powered ranking, filtering, and reordering stages
- **Geo-aware Retrieval**: Location-based search with distance calculations
- **Real-time Results**: Low-latency retrieval for consumer-facing applications

In the Feed context, Argo is the **primary retrieval mechanism** for getting store and item candidates during feed generation.

```mermaid
flowchart TB
    subgraph "Feed System"
        Surface[Surface Node<br/>Homepage/Search]
        Retriever[Store Retriever<br/>Product Override]
    end
    
    subgraph "Argo Search Infrastructure"
        Broker[Argo Broker<br/>Request Handler]
        QueryPlanner[Query Planner<br/>Pipeline Selection]
        Stages[Processing Stages<br/>Filter → Rank → Reorder]
        Index[Search Index<br/>Stores/Items/Collections]
    end
    
    Surface --> Retriever
    Retriever -->|BrokerRequest| Broker
    Broker --> QueryPlanner
    QueryPlanner --> Stages
    Stages --> Index
    Index -->|Documents| Stages
    Stages -->|Ranked Results| Broker
    Broker -->|BrokerResponse| Retriever
    Retriever -->|StoreMetadata| Surface
```

---

## Architecture Overview

### High-Level Components

```mermaid
flowchart LR
    subgraph "Client (Feed)"
        Client[Feed Node]
        ArgoRepo[Argo Repository<br/>platform/pkg/repositories/argo]
    end
    
    subgraph "Argo Broker Service"
        Broker[BrokerService<br/>gRPC Endpoint]
        
        subgraph "Query Processing"
            Parser[Query Parser]
            Planner[Query Planner]
            Executor[Pipeline Executor]
        end
    end
    
    subgraph "Search Backends"
        Discovery[Discovery Namespace<br/>Stores]
        StoreCarousel[Store Carousel<br/>Collection Members]
        ItemCarousel[Item Carousel<br/>Item Members]
        WoltVenue[Wolt Venue<br/>Wolt Stores]
        QuerySuggestions[Query Suggestions<br/>Autocomplete]
    end
    
    Client --> ArgoRepo
    ArgoRepo -->|BrokerRequest| Broker
    Broker --> Parser
    Parser --> Planner
    Planner --> Executor
    
    Executor --> Discovery
    Executor --> StoreCarousel
    Executor --> ItemCarousel
    Executor --> WoltVenue
    Executor --> QuerySuggestions
    
    Executor -->|BrokerResponse| Broker
    Broker --> ArgoRepo
```

### Component Responsibilities

| Component | Description |
|-----------|-------------|
| **Broker Service** | gRPC endpoint accepting search requests |
| **Query Planner** | Selects and configures the processing pipeline |
| **Pipeline** | Sequence of stages (filtering, ranking, reordering) |
| **Namespace** | Logical grouping of documents (stores, items, etc.) |
| **Index** | Searchable document store with geo support |

---

## Key Concepts

### Namespace

A **namespace** defines what type of documents you're searching:

| Namespace | Description | Use Case |
|-----------|-------------|----------|
| `discovery` | Main store index for DoorDash | Homepage store retrieval |
| `store_carousel` | Pregenerated store collections | Collection member retrieval |
| `item_carousel` | Pregenerated item collections | Item carousel members |
| `wolt_venue` | Wolt venue index | Wolt homepage/search |
| `query_suggestions` | Autocomplete suggestions | Search autocomplete |
| `retail_item` | Retail/grocery items | New vertical items |

### Pipeline

A **pipeline** is a named sequence of processing stages:

```mermaid
flowchart LR
    subgraph "Pipeline: store_discovery_pipeline"
        F1[Geo Filter<br/>Distance Check] --> F2[Eligibility Filter<br/>Active/Test/Hidden]
        F2 --> R1[L1 Ranking<br/>Fast Scoring]
        R1 --> R2[L2 Ranking<br/>ML Model]
        R2 --> RO[Reordering<br/>Store Status]
    end
    
    Input[Candidates] --> F1
    RO --> Output[Ranked Results]
```

Common pipelines:
- `store_discovery_pipeline` - Homepage store discovery
- `global_search_store_vertical` - Global search stores
- `pipeline_v1` - Wolt venue retrieval
- Autocomplete pipelines for query suggestions

### Context Features

**Context features** provide dynamic parameters for each request:

```go
map[string]*querypb.ContextFeatureValue{
    "user_lat":          {DoubleValue: 37.7749},
    "user_lon":          {DoubleValue: -122.4194},
    "asap_delivery_ts":  {LongValue: 1706000000000},
    "asap_pickup_ts":    {LongValue: 1706000000000},
    "experience":        {StringValue: "doordash"},
}
```

### Planner Context

**Planner context** provides configuration for the pipeline stages:

```go
map[string]*typespb.PayloadValue{
    "consumer_lat":      {DoubleValue: 37.7749},
    "consumer_lon":      {DoubleValue: -122.4194},
    "district":          {LongValue: 1234},
    "submarket_id":      {LongValue: 5678},
    "timezone":          {StringValue: "America/Los_Angeles"},
    "is_pickup":         {BoolValue: false},
    "experience":        {StringValue: "doordash"},
}
```

---

## Request Structure

### BrokerRequest Anatomy

```mermaid
flowchart TB
    subgraph "BrokerRequest"
        SQ[SearchQuery]
        CC[ConsumerContext]
        CLC[ClientContext]
    end
    
    subgraph "SearchQuery"
        NS[Namespace<br/>e.g., 'discovery']
        KW[KeywordsQuery<br/>Search terms]
        FLT[Filter<br/>Store ID filters]
        SORT[SortBy<br/>Distance, etc.]
        RF[ReturnFields<br/>Fields to return]
        SIZE[Size<br/>Result limit]
        CF[ContextFeatures<br/>User location, time]
        QP[QueryPlanner<br/>Pipeline & Context]
    end
    
    SQ --> NS
    SQ --> KW
    SQ --> FLT
    SQ --> SORT
    SQ --> RF
    SQ --> SIZE
    SQ --> CF
    SQ --> QP
```

### Code Example: Building a Request

```go
// ArgoQueryConfig defines the static configuration for a search
type ArgoQueryConfig struct {
    Namespace    string           // "discovery", "store_carousel", etc.
    Experience   string           // "doordash", "wolt"
    Pipeline     string           // "store_discovery_pipeline"
    ReturnFields []string         // Fields to return in documents
    Size         int32            // Max results
    StoreIds     []int64          // Optional: filter to specific stores
    Query        string           // Search keywords (for search surfaces)
}

// Build the broker request
request := &argopb.BrokerRequest{
    SearchQuery: &querypb.SearchQuery{
        Namespace:     "discovery",
        KeywordsQuery: buildKeyWordsQuery(queryConfig),
        Filter:        buildStoreIdFilter(queryConfig.StoreIds),
        SortBy: []*querypb.SortField{
            {
                Field:        "arc_distance_between_store_user",
                DefaultValue: math.MaxFloat64,
                SortOrder:    querypb.SortOrder_SORT_ORDER_ASCENDING,
            },
        },
        ReturnFields:    queryConfig.ReturnFields,
        Size:            queryConfig.Size,
        ContextFeatures: contextFeatures,
        QueryPlanner: &querypb.QueryPlanner{
            Pipeline: queryConfig.Pipeline,
            Context:  plannerContext,
        },
    },
    ConsumerContext: &consumerpb.ConsumerContext{
        ConsumerId:  consumerId,
        DdSessionId: sessionId,
        DdDeviceId:  deviceId,
    },
    ClientContext: &clientpb.ClientContext{
        ClientId:  "pedregal_feed",
        UseCaseId: "discovery/store_discovery_feed",
    },
}
```

---

## Namespaces & Pipelines

### DoorDash Namespaces

```mermaid
flowchart TB
    subgraph "argo-search-discovery (DoorDash)"
        Discovery[discovery<br/>Main store index]
        StoreCarousel[store_carousel<br/>Pregenerated collections]
        ItemCarousel[item_carousel<br/>Item collections]
        RetailItem[retail_item<br/>Grocery/retail items]
    end
    
    subgraph "argo-search-wolt-venue (Wolt)"
        WoltVenue[wolt_venue<br/>Wolt stores]
        WoltVenueCollections[wolt_venue_collections<br/>Wolt collections]
    end
    
    subgraph "argo-search-query-suggestions"
        QuerySuggestions[query_suggestions<br/>Autocomplete]
    end
```

### Namespace Configuration by Surface

| Surface | Namespace | Pipeline | Description |
|---------|-----------|----------|-------------|
| Homepage (DD) | `discovery` | `store_discovery_pipeline` | Store retrieval |
| Homepage (Wolt) | `wolt_venue` | `pipeline_v1` | Venue retrieval |
| Global Search (DD) | `store` | `global_search_store_vertical` | Search stores |
| Global Search (Wolt) | `wolt_venue` | `pipeline_v1` | Search venues |
| Autocomplete | `query_suggestions` | Various | Query suggestions |
| Collection Members | `store_carousel` | N/A | Pregenerated stores |
| Item Collections | `item_carousel` | N/A | Pregenerated items |

---

## Integration with Feed

### Store Retrieval Flow

```mermaid
sequenceDiagram
    participant Surface as Surface Node
    participant Override as StoreRetrieverConfig
    participant FPR as First Pass Ranker
    participant ArgoRepo as Argo Repository
    participant Broker as Argo Broker
    
    Surface->>Override: Retrieve(ctx, feedScope)
    
    Note over Override: Check cache
    alt Cache Hit
        Override-->>Surface: Cached StoreMetadata[]
    else Cache Miss
        Override->>FPR: GetStores(consumerId, geohash)
        FPR-->>Override: FPR Store IDs (candidates)
        
        Override->>ArgoRepo: BuildDiscoveryQuery(scope, config)
        ArgoRepo-->>Override: BrokerRequest
        
        Override->>Broker: Broker(ctx, request)
        Broker-->>Override: BrokerResponse
        
        Override->>Override: convertArgoResponseToEntities
        Override->>Override: DeduplicateStoresByBusiness
        
        Override-->>Surface: StoreMetadata[]
    end
```

### Code Location in Feed

```
nodes/consumer/feed/
├── platform/pkg/repositories/argo/
│   ├── argo.go                      # Core query building
│   ├── constants.go                 # Field names, context keys
│   ├── store_metadata_converter.go  # Response → StoreMetadata
│   └── converters/
│       └── efs_filters.go           # Filter context conversion
│
├── surface/homepage/productoverrides/restaurants/storeretrieve/
│   └── config.go                    # Homepage DD store retrieval
│
├── surface/homepage/wolt/productoverrides/restaurants/storeretrieve/
│   └── config.go                    # Homepage Wolt store retrieval
│
├── surface/search/global/productoverrides/restaurants/storeretrieve/
│   ├── config.go                    # Global search DD
│   └── wolt/config.go               # Global search Wolt
│
└── platform/pkg/collection/impl/retriever/pregenerated/
    ├── store_member_retriever.go    # Collection store members
    └── item_member_retriever.go     # Collection item members
```

---

## Query Building

### Helper Functions

The Argo repository provides several helper functions for building requests:

```mermaid
flowchart TB
    subgraph "BuildDiscoveryQuery"
        BDQ[BuildDiscoveryQuery<br/>Main entry point]
    end
    
    subgraph "Context Builders"
        SCF[buildSearchContextFeatures<br/>User location, timestamps]
        PCP[buildPlannerContextPayload<br/>Geo, eligibility, filters]
    end
    
    subgraph "Component Builders"
        KWQ[buildKeyWordsQuery<br/>Search keywords]
        SIF[buildStoreIdFilter<br/>Store ID constraints]
        PEV[buildPlannerEligibilityValues<br/>Active/test filters]
        PGV[buildPlannerGeoValues<br/>Lat/lng, district]
        CC[buildConsumerContext<br/>Consumer ID, session]
        CLC[buildClientContext<br/>Client identification]
    end
    
    BDQ --> SCF
    BDQ --> PCP
    BDQ --> KWQ
    BDQ --> SIF
    BDQ --> CC
    BDQ --> CLC
    
    PCP --> PEV
    PCP --> PGV
```

### Common Context Features

| Feature Key | Type | Description |
|-------------|------|-------------|
| `user_lat` / `user_lon` | Double | Consumer location |
| `asap_delivery_ts` | Long | Current timestamp (millis) |
| `asap_pickup_ts` | Long | Current timestamp (millis) |
| `experience` | String | "doordash" or "wolt" |
| `consumer_id` | Long | Consumer identifier |
| `district` | Long | District ID |
| `submarket_id` | Long | Submarket ID |
| `timezone` | String | e.g., "America/Los_Angeles" |

### Return Fields

Common fields returned from the `discovery` namespace:

```go
ReturnFields: []string{
    // Identifiers
    "store_id", "business_id", "name",
    
    // Classification
    "tier_level", "price_range", "business_vertical_id",
    "vertical_ids", "primary_vertical_ids",
    
    // Location
    "location", "delivery_radius_circle",
    
    // Eligibility
    "is_consumer_subscription_eligible", "is_partner",
    "is_snap_eligible", "is_hsa_fsa_eligible",
    "offers_pickup", "hide_from_homepage_list",
    
    // Pricing
    "starting_point", "avg_subtotal_per_person",
    "saf_st_price_inflation_rate",
    
    // Other
    "brand_business_group_id", "catering",
}
```

---

## Response Handling

### BrokerResponse Structure

```mermaid
flowchart TB
    subgraph "BrokerResponse"
        SR[SearchResults]
        Meta[Metadata]
    end
    
    subgraph "SearchResults"
        Docs[Documents<br/>Array of Document]
        Total[TotalHits]
    end
    
    subgraph "Document"
        PK[PrimaryKey<br/>Store ID]
        Fields[Fields<br/>Map of field values]
    end
    
    SR --> Docs
    SR --> Total
    Docs --> PK
    Docs --> Fields
```

### Converting Response to StoreMetadata

```go
// MapArgoDocumentToStore converts Argo Documents to StoreMetadata
func MapArgoDocumentToStore(documents []*srpb.Document) ([]types.StoreMetadata, error) {
    return lo.Map(documents, func(document *srpb.Document, _ int) types.StoreMetadata {
        businessID := document.GetFields()["business_id"].GetLongValue()
        location := document.GetFields()["location"].GetFieldValueList().GetFieldValue()
        
        return types.StoreMetadata{
            ID:                          types.StoreId(document.PrimaryKey),
            Name:                        document.GetFields()["name"].GetStringValue(),
            BusinessID:                  &businessID,
            IsStoreSubscriptionEligible: document.GetFields()["is_consumer_subscription_eligible"].GetStringValue() == "t",
            Location: types.StoreLocation{
                Lat: location[0].GetDoubleValue(),
                Lng: location[1].GetDoubleValue(),
            },
            StoreVerticalIDs:        extractIntList(document.GetFields()["vertical_ids"]),
            StorePrimaryVerticalIDs: extractIntList(document.GetFields()["primary_vertical_ids"]),
        }
    })
}
```

---

## Common Patterns

### Pattern 1: Homepage Store Retrieval

```mermaid
flowchart TB
    subgraph "Homepage Store Retrieval"
        FPR[1. Get FPR Candidates<br/>First Pass Ranker]
        Build[2. Build Argo Query<br/>With FPR store IDs]
        Call[3. Call Argo Broker<br/>discovery namespace]
        Convert[4. Convert Response<br/>To StoreMetadata]
        Dedup[5. Deduplicate<br/>By Business ID]
        Cache[6. Cache Results<br/>By Consumer + Location]
    end
    
    FPR --> Build
    Build --> Call
    Call --> Convert
    Convert --> Dedup
    Dedup --> Cache
```

**Key Points:**
- Uses FPR (First Pass Ranker) to get initial candidate store IDs
- Passes candidate IDs to Argo for final filtering and ranking
- Caches results by consumer ID + location

### Pattern 2: Collection Member Retrieval

```mermaid
flowchart TB
    subgraph "Collection Member Retrieval"
        GetColl[1. Get Collection IDs<br/>From Collection Metadata]
        BuildJoin[2. Build Join Query<br/>Collection → Stores]
        CallArgo[3. Call Argo<br/>store_carousel namespace]
        GroupBy[4. Group By Collection<br/>Map collection → stores]
        Filter[5. Filter Deliverable<br/>Using FPR stores]
    end
    
    GetColl --> BuildJoin
    BuildJoin --> CallArgo
    CallArgo --> GroupBy
    GroupBy --> Filter
```

**Key Points:**
- Uses `store_carousel` or `item_carousel` namespace
- Groups results by collection ID
- Filters using deliverable stores from FPR

### Pattern 3: Wolt Venue Search

```mermaid
flowchart TB
    subgraph "Wolt Venue Search"
        BuildGeo[1. Build Geo Filter<br/>GeoDistanceQuery]
        AddProduct[2. Add Product Line Filter<br/>Optional: restaurants, retail]
        BuildReq[3. Build BrokerRequest<br/>wolt_venue namespace]
        Call[4. Call Wolt Argo Broker<br/>Different endpoint]
        Convert[5. Convert to StoreMetadata<br/>venue_id → StoreId]
    end
    
    BuildGeo --> AddProduct
    AddProduct --> BuildReq
    BuildReq --> Call
    Call --> Convert
```

**Key Points:**
- Uses separate Argo cluster for Wolt (`argo-search-wolt-venue-broker`)
- Requires API key authentication
- Uses `GeoDistanceQuery` for radius filtering

### Pattern 4: Query Suggestions (Autocomplete)

```mermaid
flowchart TB
    subgraph "Autocomplete Suggestions"
        Parse[1. Parse Query<br/>User input]
        BuildQuery[2. Build Suggestion Query<br/>query_suggestions namespace]
        CallArgo[3. Call Argo<br/>Autocomplete pipeline]
        Extract[4. Extract Suggestions<br/>Type: store, item, query]
        Rank[5. Rank & Dedupe<br/>By relevance score]
    end
    
    Parse --> BuildQuery
    BuildQuery --> CallArgo
    CallArgo --> Extract
    Extract --> Rank
```

---

## Configuration & Clients

### gRPC Client Configuration

```go
// DoorDash Discovery Argo Client
argoClient argopb.BrokerServiceAsyncClient `gr:"authority=argo-search-discovery-broker.service.prod.ddsd,metadata={dd-api-secret:(vault 'feed/ARGO_SEARCH_DISCOVERY_FEED_API_KEY/ARGO_SEARCH_DISCOVERY_FEED_API_KEY')},attemptDelay=[0ms;30ms;60ms],retryCodes=[ABORTED;UNAVAILABLE;DEADLINE_EXCEEDED]"`

// Wolt Venue Argo Client
argoClient argopb.BrokerServiceAsyncClient `gr:"host=argo-search-wolt-venue-broker.insecure-svc.prod-wolt.doordash.red,port=50051,metadata={dd-api-secret:(vault 'feed/ARGO_SEARCH_DISCOVERY_WOLT_FEED_API_KEY/ARGO_SEARCH_DISCOVERY_WOLT_FEED_API_KEY')}"`
```

### Environment-Specific Handling

```go
// Select client based on environment
var argoClient argopb.BrokerServiceAsyncClient
if guardian.IsLocal() {
    argoClient = s.argoClientLocal  // Uses devbox tunnel
} else {
    argoClient = s.argoClient       // Production/Sandbox
}
```

---

## Debugging & Observability

### Logging Patterns

```go
// Log request building
feedScope.Logger().Info(ctx, "store_retrieval.build_discovery_query.success", 
    log.Any("namespace", "discovery"),
    log.Any("pipeline", "store_discovery_pipeline"),
    log.Any("size", defaultDocRetrievalSize))

// Log response
feedScope.Logger().Info(ctx, "store_retrieval.argo_response.received",
    log.Any("doc_count", len(argoResponse.SearchResults.Documents)),
    log.Any("total_hits", argoResponse.SearchResults.TotalHits))

// Log errors
feedScope.Logger().Error(ctx, "store_retrieval.convert_response.error",
    log.Any("error", err.Error()))
```

### Key Metrics

- **Latency**: Time from request to response
- **Document Count**: Number of results returned
- **Cache Hit Rate**: Effectiveness of result caching
- **Error Rate**: Failed Argo calls

---

## Summary

### Quick Reference

| Concept | Description |
|---------|-------------|
| **Argo** | DoorDash's distributed search infrastructure |
| **Broker** | gRPC service accepting search requests |
| **Namespace** | Document type (discovery, store_carousel, etc.) |
| **Pipeline** | Processing stages (filter → rank → reorder) |
| **Context Features** | Dynamic request parameters (location, time) |
| **Planner Context** | Pipeline configuration (eligibility, geo) |

### Common Namespaces

| Namespace | Content | Used By |
|-----------|---------|---------|
| `discovery` | DoorDash stores | Homepage, Search |
| `wolt_venue` | Wolt venues | Wolt Homepage, Search |
| `store_carousel` | Pregenerated store collections | Collection members |
| `item_carousel` | Pregenerated item collections | Item collections |
| `query_suggestions` | Autocomplete | Search autocomplete |

### Key Files

| File | Purpose |
|------|---------|
| `platform/pkg/repositories/argo/argo.go` | Core query building |
| `platform/pkg/repositories/argo/constants.go` | Field and context keys |
| `platform/pkg/repositories/argo/store_metadata_converter.go` | Response conversion |
| `surface/*/storeretrieve/config.go` | Surface-specific retrieval |

---

## Related Documentation

- [Feed Onboarding Guide](./PEDREGAL_FEED_ONBOARDING.md) - Feed system overview
- [Feed Terminology](https://github.com/doordash/pedregal/blob/main/nodes/consumer/feed/docs/reference/terminology.md)
- [Argo Search Discovery Configs](https://github.com/doordash/argo-search/tree/master/configurations/discovery)

---

*Generated from Pedregal Feed codebase analysis. Last updated: January 2026.*
