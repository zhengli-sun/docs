# Roo Home Feed Generation Audit

> **Note:** This document covers the **legacy Roo/Deliveroo** home feed system in `roo/co-restaurants`. For the **Pedregal/DoorDash Feed system** (the new unified architecture), see [FEED_ONBOARDING_GUIDE.md](./FEED_ONBOARDING_GUIDE.md).

## Overview

This document provides a holistic audit of how the Home Feed is generated in the `roo/co-restaurants` repository. The home feed is the core consumer experience, displaying a personalized list of restaurants and other content.

**Repository:** `roo/co-restaurants`
**Primary Entry Point:** `GetFeed` in `services/home/home.go`
**Orchestration:** `HomeDataPipeline` in `services/home/data.pipeline.go`

---

## 1. High-Level Flow

The Home Feed generation is a highly concurrent process managed by `HomeDataPipeline`. It follows these general steps:

1.  **Initialization:** The `GetFeed` function initializes the request, validates inputs (Location, Time, etc.), and sets up the context (APM spans, Logging).
2.  **Data Pipeline:** `HomeDataPipeline` executes a DAG (Directed Acyclic Graph) of tasks using `safe.Group` for parallelism.
    *   **Blocking Phase:** Critical data required for downstream tasks is fetched first (e.g., `fetchPartners` for retrieval, `getZone`).
    *   **Non-Blocking Phase:** Once the initial candidate set is ready, dozens of services are called in parallel to "decorate" the partners with additional info (Pricing, ETAs, Rewards, Images, etc.).
3.  **Filtering:** Candidates are filtered based on fulfillment type (Delivery/Pickup), availability, hygiene ratings, and other business rules.
4.  **Ranking:** The filtered list is sorted, typically using an ML-based ranker, with fallbacks to simple heuristics.
5.  **Analytics & Response:** The final feed object is constructed, analytics events are queued, and the response is returned to the GraphQL layer.

---

## 2. Indexing and Retrieval (Candidate Selection)

**Location:** `services/restaurants/search.go`

Retrieval is responsible for finding a broad set of "candidate" restaurants that serve the user's location.

*   **Primary Mechanism:** Geographic search via the database (Postgres).
    *   **Function:** `s.restaurants.Search` -> `findForDelivery` (or `findForInPerson`).
    *   **Logic:**
        *   If a **Bounding Box** is provided, it searches within that polygon (`FindRestaurantsByBoundingBox`).
        *   Otherwise, it uses `Selection Management` (if enabled/experimenting) or falls back to a standard location-based lookup (`FindRestaurantsForLocation`).
        *   It identifies the user's **Neighborhood** and **Zone** using `repo.FindNeighborhoodForLocation`.
*   **Filtering at Source:** Basic filters are applied at the query level:
    *   Fulfillment Type (Delivery vs. Collection).
    *   Test Sites / Hidden Sites.
    *   Brand filters (if searching for a specific brand).

---

## 3. Ranking

**Location:** `services/home/sort.go` (`ExecuteSortRestaurants`)

Ranking orders the retrieved candidates to optimize for conversion and user relevance.

*   **Primary Ranker:** An ML-based ranking service.
    *   **Logic:** `ExecuteSortRestaurants` calls `s.restaurantRankingFactory.New(aw).Sort(...)`.
    *   **Features:** It passes `prefetchedFeatures` including:
        *   Customer history (IsNew, RecentlyOrdered).
        *   Restaurant popularity.
        *   Estimated Order Duration (EOD).
        *   Contextual info (Timezone, Country).
*   **Fallback Mechanism (`SimpleSort`):**
    *   If the ranking service fails or panics, a heuristic sort is used:
        1.  **ASAP** (Open Now)
        2.  **Scheduled** (Pre-order)
        3.  **Unavailable** (Closed)
    *   Within these groups, it sorts by EOD (fastest first) or opening time.
*   **Collection Sorting:** If a specific collection is requested (e.g., "Top Rated"), a specialized `CollectionSorter` may be used instead of the main ranker.

---

## 4. Dependencies

The Home Feed aggregates data from numerous internal services:

| Service | Purpose |
| :--- | :--- |
| **Selection Management** | Candidate filtering and experiment management for retrieval. |
| **Ranking Service** | ML-based sorting of restaurants. |
| **Consumer Fees (Pricing)** | Calculates delivery fees, service fees, and surge pricing. |
| **Delivery Targets** | Provides Estimated Order Durations (EOD/ETA). |
| **Plus Rewards** | Checks eligibility and offers for the loyalty program. |
| **Feature Store** | Retrieves pre-computed ML features for users and restaurants. |
| **Offers & Promotions** | Applies discounts, vouchers, and restaurant offers. |
| **RSR (Reliability)** | "Restaurant Service Reliability" - managing load and availability. |
| **Gatekeeper** | Feature flagging and logic gating. |
| **Home Content** | Provides merchandising cards and generic content blocks. |
| **Co-Lists** | Manages user favourites and other lists. |
| **Determinator** | Experimentation platform. |
| **UGC (Reviews)** | Aggregated ratings and reviews. |

---

## 5. Analytics Events

**Location:** `internal/server/graphql/search.go` & `services/home/analytics.go`

Analytics are crucial for tracking feed performance and ML model training.

1.  **Construction:** The `GetAnalytics` function in `services/home/analytics.go` transforms internal state into a `analytics.Feed` struct. This captures:
    *   Request metadata (UUIDs, Time, Location).
    *   Ranking decisions (Algorithm used, Scores).
    *   The list of restaurants returned (ids, availability, positions).
    *   User state (New vs. Returning, Employee).
2.  **Emission:** In the GraphQL resolver (`internal/server/graphql/search.go`):
    *   Events are tracked asynchronously using `safe.Go` to avoid blocking the user response.
    *   **Key Events:**
        *   `TrackServedHomeEvent`: The main impression event for the feed.
        *   `TrackAdFillRate`: Performance of sponsored placements.
        *   `TrackServedModalEvent`: Impressions for any modals shown.
    *   **Technology:** Events are typically published via **Franzalytics** (internal analytics pipeline) or direct business event trackers.

---

## 6. Code Structure Summary

*   `services/home/`
    *   `home.go`: Service interface and `GetFeed`.
    *   `data.pipeline.go`: The orchestration engine (`HomeDataPipeline`).
    *   `data.decorate.go`: Logic to merge data from various services into restaurant objects.
    *   `sort.go`: Ranking logic and fallbacks.
    *   `analytics.go`: Analytics struct mapping.
*   `services/restaurants/`
    *   `search.go`: Database retrieval logic.
*   `internal/server/graphql/`
    *   `search.go`: The API resolver that calls the Home Service and triggers analytics.
