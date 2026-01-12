---
description: Examples of shim patterns in Pedregal for integrating with legacy DoorDash and Wolt services
---

# Shim Examples in Pedregal

This document catalogs existing shim implementations that can be used as reference patterns when building new integrations with legacy services.

## What is a Shim?

A **shim** is a Graph Runner component that bridges Pedregal to legacy services (DoorDash, Wolt, or external APIs). Shims typically:

1. Define a clean interface for internal use
2. Call the legacy service (gRPC or HTTP)
3. Transform legacy responses to Pedregal's internal models
4. Handle errors and observability

---

## 1. Feed Sibyl Shim (gRPC → DoorDash ML Service)

**Location**: [`nodes/consumer/feed/shims/sibyl/scorer.go`](https://github.com/doordash/pedregal/blob/main/nodes/consumer/feed/shims/sibyl/scorer.go)

**Purpose**: Calls DoorDash's Sibyl Prediction Service for ML-based scoring of feed entities.

**Key Characteristics**:
- gRPC async client
- Vault-based authentication
- Batch processing with chunking
- Internal proto definitions in `protos/public/feed/v2/temporary/sibyl/`

**Pattern Highlights**:

```go
type scorerNode struct {
    _         gr.Node[Score] `gr:"name=feed-sibyl-shim"`
    spsClient spspb.SibylPredictionServiceAsyncClient `gr:"authority=sibyl-prediction-service-web-feed-hp.service.prod.ddsd,metadata={dd-api-secret:(vault 'feed/SIBYL_PREDICTION_SERVICE_FEED_HP_API_KEY/SIBYL_PREDICTION_SERVICE_FEED_HP_API_KEY')},attemptDelay=[0ms;30ms;60ms],retryCodes=[ABORTED;UNAVAILABLE;DEADLINE_EXCEEDED]"`
}
```

---

## 2. Wolt Market Busyness Shim (HTTP → Wolt REST API)

**Location**: [`nodes/maps/legacygeography/busyness_client.go`](https://github.com/doordash/pedregal/blob/main/nodes/maps/legacygeography/busyness_client.go)

**Graph Registration**: [`nodes/maps/legacygeography/shim/graphs.go`](https://github.com/doordash/pedregal/blob/main/nodes/maps/legacygeography/shim/graphs.go)

**Purpose**: Retrieves fleet busyness metrics from Wolt's Market Busyness REST API.

**Key Characteristics**:
- HTTP client with service mesh routing
- JSON request/response handling
- Custom headers for S2S authentication

**Pattern Highlights**:

```go
type busynessClient struct {
    _ gr.Component[BusynessClient]
    client grhttp.Client `gr:"provider=http,service=market-busyness,timeout=10s,routeServiceMesh=true,headers={Accept:application/json;X-S2S-Auth-Audience:wolt.market-busyness}"`
}

func (c *busynessClient) GetFleetBusyness(ctx context.Context, fleetName string) future.Future[*FleetBusynessResponse] {
    httpReq, _ := http.NewRequestWithContext(ctx, http.MethodGet, reqURL, nil)
    return future.MapValue(ctx, c.client.Do(ctx, httpReq), func(ctx context.Context, resp *http.Response) (*FleetBusynessResponse, error) {
        return c.processFleetBusynessResponse(ctx, resp, fleetName)
    })
}
```

---

## 3. Wolt Order Tracking Context Shim (Data Transformation)

**Location**: [`nodes/order/tracking/internal/order_tracking_context/internal/shims/wolt_order_tracking_context_shim.go`](https://github.com/doordash/pedregal/blob/main/nodes/order/tracking/internal/order_tracking_context/internal/shims/wolt_order_tracking_context_shim.go)

**DoorDash equivalent**: [`nodes/order/tracking/internal/order_tracking_context/internal/shims/doordash_order_tracking_context_shim.go`](https://github.com/doordash/pedregal/blob/main/nodes/order/tracking/internal/order_tracking_context/internal/shims/doordash_order_tracking_context_shim.go)

**Purpose**: Transforms Wolt's order tracking data to Pedregal's unified `OrderTrackingContext` format.

**Key Characteristics**:
- Adapter/transformation pattern
- Calls another client component
- Maps between different proto schemas
- Wide event observability

**Pattern Highlights**:

```go
type WoltOrderTrackingContextShim interface {
    GetMappedOrderTrackingContext(ctx context.Context, purchaseId string, requestContext *woltordertrackingpb.RequestContext) future.Future[*trackingpb.OrderTrackingContext]
}

type woltShimImpl struct {
    _ gr.Component[WoltOrderTrackingContextShim]
    woltClient clients.WoltOrderTrackingServiceClient
}

func (s *woltShimImpl) GetMappedOrderTrackingContext(...) future.Future[*trackingpb.OrderTrackingContext] {
    woltFuture := s.woltClient.GetWoltOrderTrackingContext(ctx, purchaseId, requestContext)
    return future.MapValue(ctx, woltFuture, func(...) (*trackingpb.OrderTrackingContext, error) {
        pedregalOtc := order_tracking_context_builder.MapWoltOrderTrackingContextToPedregal(ctx, woltResponse.GetOrderTrackingContext())
        return pedregalOtc, nil
    })
}
```

---

## 4. DoorDash Legacy Fee Provider Shim (Proto Translation)

**Location**: [`nodes/order/pricing/fee/internal/adaptor/fee_shim/dd_legacy_fee_provider.go`](https://github.com/doordash/pedregal/blob/main/nodes/order/pricing/fee/internal/adaptor/fee_shim/dd_legacy_fee_provider.go)

**Mapper**: [`nodes/order/pricing/fee/internal/adaptor/fee_shim/dd_legacy_fee_mapper.go`](https://github.com/doordash/pedregal/blob/main/nodes/order/pricing/fee/internal/adaptor/fee_shim/dd_legacy_fee_mapper.go)

**Purpose**: Translates Pedregal fee requests to legacy Quote Adjuster service format and back.

**Key Characteristics**:
- Bidirectional proto mapping
- Separates mapper logic from client logic
- Interface-based provider pattern

**Pattern Highlights**:

```go
type DDLegacyFeeProvider struct {
    ddFeeClient quotepb.ConsumerPricingQuoteAdjusterServiceAsyncClient
}

func (p *DDLegacyFeeProvider) GetBatchConsumerFees(ctx context.Context, req *feepb.GetBatchConsumerFeesRequest) future.Future[*feepb.GetBatchConsumerFeesResponse] {
    // Map internal proto → legacy proto
    quoteAdjusterReq, _ := mapBatchConsumerFeesRequestToQuoteAdjuster(req)
    
    // Call legacy service
    quoteAdjusterRespFuture := p.GetBatchQuoteAdjustments(ctx, quoteAdjusterReq)
    
    // Map legacy proto → internal proto
    return future.MapValue(ctx, quoteAdjusterRespFuture, func(...) (*feepb.GetBatchConsumerFeesResponse, error) {
        // Transform response
    })
}
```

---

## 5. Legacy Supply Demand Service Shim (gRPC with Shared Secrets)

**Location**: [`nodes/logistics/labor/mobilization/legacysupplydemand/joint_optimization_client.go`](https://github.com/doordash/pedregal/blob/main/nodes/logistics/labor/mobilization/legacysupplydemand/joint_optimization_client.go)

**Graph Registration**: [`nodes/logistics/labor/mobilization/legacysupplydemand/shim/graphs.go`](https://github.com/doordash/pedregal/blob/main/nodes/logistics/labor/mobilization/legacysupplydemand/shim/graphs.go)

**Documentation**: [`nodes/logistics/labor/mobilization/legacysupplydemand/shim/README.md`](https://github.com/doordash/pedregal/blob/main/nodes/logistics/labor/mobilization/legacysupplydemand/shim/README.md)

**Purpose**: Wraps DoorDash's Supply Demand Service for joint optimization supply signals.

**Key Characteristics**:
- gRPC async client
- Shared Vault secret path (`graph-runner/shared/`)
- Wide event observability
- Test helper constructor

**Pattern Highlights**:

```go
type JointOptimizationClient interface {
    GetSupplySignalForJointOptimization(ctx context.Context, startingPointIds []int64) future.Future[*sdspb.GetSupplySignalForJointOptimizationResponse]
}

type jointOptimizationClient struct {
    _ gr.Component[JointOptimizationClient]
    supplyDemandClient sdspb.SupplyDemandServiceAsyncClient `gr:"authority=supply-demand-service-web.service.prod.ddsd,metadata={dd-api-secret:(vault 'graph-runner/shared/SUPPLY_DEMAND_API_KEY')}"`
}

// Test helper for mocking
func NewJointOptimizationClientForTesting(supplyDemandClient sdspb.SupplyDemandServiceAsyncClient) JointOptimizationClient {
    return &jointOptimizationClient{supplyDemandClient: supplyDemandClient}
}
```

---

## Summary Comparison

| Shim | Protocol | Target | Auth Method | Key Files |
|------|----------|--------|-------------|-----------|
| **Sibyl Scorer** | gRPC | DoorDash | Vault secret | `shims/sibyl/scorer.go` |
| **Wolt Busyness** | HTTP/REST | Wolt | Service mesh headers | `legacygeography/busyness_client.go` |
| **Wolt Order Tracking** | Component | Wolt → Pedregal | N/A (uses other client) | `shims/wolt_order_tracking_context_shim.go` |
| **Fee Provider** | gRPC | DoorDash | Async client | `fee_shim/dd_legacy_fee_provider.go` |
| **Supply Demand** | gRPC | DoorDash | Shared Vault secret | `legacysupplydemand/joint_optimization_client.go` |

---

## Creating a New Shim

### Step 1: Choose Your Pattern

- **gRPC to DoorDash service** → Follow Sibyl or Supply Demand pattern
- **HTTP to Wolt/external service** → Follow Busyness Client pattern
- **Data transformation between schemas** → Follow Order Tracking Context pattern

### Step 2: Define Interface

```go
type MyLegacyClient interface {
    GetData(ctx context.Context, ids []string) future.Future[*MyResponse]
}
```

### Step 3: Implement as Component

```go
type myLegacyClient struct {
    _ gr.Component[MyLegacyClient]
    
    // For gRPC:
    legacyClient pb.ServiceAsyncClient `gr:"authority=service.prod.ddsd,metadata={dd-api-secret:(vault 'path/to/secret')}"`
    
    // For HTTP:
    httpClient grhttp.Client `gr:"provider=http,service=my-service,timeout=10s,routeServiceMesh=true"`
}
```

### Step 4: Add Observability

```go
wevent.Add(ctx, "my_shim.request.count", len(ids))
// ... call legacy service ...
wevent.Add(ctx, "my_shim.response.success", true)
```

### Step 5: Register Graph (if externally callable)

```go
func init() {
    gr.Graph[MyFunc]("my-shim-graph-name", pb.Service_Method_FullMethodName)
}
```

---

## Related Documentation

- [Scaffold Graph Runner Shim Skill](https://github.com/doordash/pedregal/blob/main/.cursor/rules/scaffold-graph-runner-shim.mdc)
- [Legacy Integration Patterns](https://github.com/doordash/pedregal/blob/main/.claude/skills/scaffold-graph-runner-shim/references/legacy_integration_patterns.md)
- [Graph Runner Node API](https://github.com/doordash/pedregal/blob/main/docs/graph-runner/reference/node-api.md)
- [Future API Guide](https://github.com/doordash/pedregal/blob/main/docs/graph-runner/reference/future-api.md)

