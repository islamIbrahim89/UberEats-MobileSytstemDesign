# 🍔 Uber Eats — Mobile System Design (iOS)

> A food ordering feature system design for iOS, covering requirements, data models, API contracts, architecture, and engineering trade-offs. Scope: Restaurant menu → Add to cart → Order validation → Order status tracking.

---

## Table of Contents

- [Functional Requirements](#functional-requirements)
- [Non-Functional Requirements](#non-functional-requirements)
- [Data Models](#data-models)
- [API Contract](#api-contract)
- [Architecture Overview](#architecture-overview)
- [Layer-by-Layer Breakdown](#layer-by-layer-breakdown)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Trade-offs & Considerations](#trade-offs--considerations)

---

## Functional Requirements

| # | Requirement |
|---|-------------|
| 1 | View restaurant menu — full list of dishes grouped by category |
| 2 | Customize dish — modifiers (size, toppings, extras) |
| 3 | Manage cart — add/remove items, see live total with modifiers |
| 4 | Place order — confirm and pay |
| 5 | Track order status — real-time updates (Accepted → Cooking → On the way) |

> **Out of scope:** Authentication, map rendering, courier tracking in MapKit/MapLibre. We focus on the "Money Flow" — the user journey that drives business value.

---

## Non-Functional Requirements

| # | Requirement |
|---|-------------|
| 1 | **Responsiveness** — 60+ FPS scrolling; UI reacts instantly to user actions (Optimistic UI) |
| 2 | **Offline support** — menu remains browsable after initial load, even without connectivity |
| 3 | **Data consistency** — cart state syncs across devices; prices stay up to date |
| 4 | **Reliability** — no duplicate orders on network failure; idempotent payment submission |
| 5 | **Low update latency** — order status updates in real time, no pull-to-refresh required |

---

## Data Models

### `Restaurant`

```
Restaurant {
  restaurant_id: UUID
  currency:      String            // e.g. "USD" — scoped to menu response
  categories:    [Category]
}
```

### `Category`

```
Category {
  id:    UUID
  name:  String                    // e.g. "Burgers", "Drinks"
  items: [MenuItem]
}
```

### `MenuItem`

```
MenuItem {
  id:               UUID
  name:             String
  description:      String
  price:            Decimal
  is_available:     Bool           // false → "Add" button disabled in UI
  image_thumbnail:  ImageURL       // Low-res — used in menu list cells
  image_full_res:   ImageURL       // High-res — used in item detail view
  modifier_groups:  [ModifierGroup]
}
```

> Two image fields are intentional. The menu list renders dozens of dish cards simultaneously; loading full-res images there wastes bandwidth and memory. Thumbnail is loaded eagerly; full-res is loaded lazily on detail open.

### `ModifierGroup`

```
ModifierGroup {
  id:            UUID
  name:          String            // e.g. "Size", "Toppings"
  min_selection: Int               // Validation: 1 = required choice
  max_selection: Int
  options:       [ModifierOption]
}
```

### `ModifierOption`

```
ModifierOption {
  id:          UUID
  name:        String              // e.g. "Large", "Extra Cheese"
  price_delta: Decimal             // 0.00 if no surcharge
}
```

### `CartItem`

```
CartItem {
  menu_item_id:       UUID
  quantity:           Int
  selected_modifiers: [UUID]       // IDs of selected ModifierOptions only
}
```

### `Cart`

```
Cart {
  cart_id:           UUID?         // Null on first sync; server assigns and returns
  restaurant_id:     UUID
  items:             [CartItem]
  total_price:       Decimal       // Server-calculated — never trust client total
  validation_errors: [String]      // e.g. "Cheese is out of stock"
}
```

> `cart_id` is null on the first `POST /cart/sync`. The server creates a session and returns a `cart_id` to be stored and sent with all subsequent requests, linking items into a single order.

### `Order`

```
Order {
  order_id:               UUID
  status:                 "created" | "cooking" | "delivering" | "delivered"
  estimated_delivery_time: Timestamp
}
```

### `OrderStatusEvent` (WebSocket payload)

```
OrderStatusEvent {
  event:       String              // e.g. "order_updated"
  order_id:    UUID
  status:      "created" | "cooking" | "delivering" | "delivered"
  title:       String              // e.g. "Cooking"
  description: String             // e.g. "Chef is making your burger"
  eta:         Timestamp
}
```

> The event envelope uses a string `status` field rather than an integer enum. This allows the backend to introduce new states (e.g. `"quality_check"`) without a client-side app update.

---

## API Contract

### Menu

```
GET /restaurants/{id}/menu
```

Returns the full menu in a single hierarchical response: `Restaurant → Categories → Items → ModifierGroups → Options`. This eliminates N+1 requests (fetching categories first, then items per category) and enables full offline caching.

**Response:**
```json
{
  "restaurant_id": "r_12345",
  "currency": "USD",
  "categories": [
    {
      "id": "cat_burgers",
      "name": "Burgers",
      "items": [
        {
          "id": "item_bigmac",
          "name": "Big Mac",
          "price": 5.99,
          "is_available": true,
          "modifier_groups": [
            {
              "id": "mod_size",
              "name": "Size",
              "min_selection": 1,
              "max_selection": 1,
              "options": [
                { "id": "opt_m", "name": "Medium", "price_delta": 0.00 },
                { "id": "opt_l", "name": "Large",  "price_delta": 1.50 }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

> This response is cached to disk after first load. That directly implements **offline availability** — the user can browse the menu with no connectivity.

---

### Cart Sync

```
POST /cart/sync
Authorization: Bearer <token>
```

The server is the **Single Source of Truth** for cart state. It recalculates prices and validates availability server-side. `user_id` is never sent in the request body — the `Authorization` header identifies the user, eliminating a security anti-pattern.

**Request:**
```json
{
  "cart_id": "cart_abc_123",
  "restaurant_id": "r_12345",
  "items": [
    {
      "menu_item_id": "item_bigmac",
      "quantity": 2,
      "selected_modifiers": ["opt_l", "opt_cheese"]
    }
  ]
}
```

**Response:**
```json
{
  "cart_id": "cart_abc_123",
  "total_price": 14.98,
  "validation_errors": [],
  "items": [ "..." ]
}
```

| Scenario | Behaviour |
|----------|-----------|
| First sync (`cart_id: null`) | Server creates session, returns new `cart_id` |
| Subsequent syncs | `cart_id` links all items into one order |
| Item becomes unavailable | Server returns `validation_errors`; client rolls back optimistic UI |

---

### Order Submission

```
POST /orders/submit
Authorization: Bearer <token>
Idempotency-Key: <UUID>
```

The `cart_id` from the sync phase is reused — no need to resend the item list. The client **never sends raw card data**; it first tokenises via Apple Pay or Stripe, then sends only the resulting `payment_method_id`.

**Request:**
```json
{
  "cart_id": "cart_abc_123",
  "payment_method_id": "card_visa_4242",
  "delivery_address": {
    "lat": 37.77,
    "lon": -122.41,
    "address_line": "1 Market St"
  },
  "comment": "Leave at door"
}
```

**Response:**
```json
{
  "order_id": "ord_999",
  "status": "created",
  "estimated_delivery_time": "2025-12-19T19:30:00Z"
}
```

> The `Idempotency-Key` UUID is generated client-side before the first attempt and **reused on every retry**. Without it: the user taps "Pay," the network drops, the app retries, and the user is charged twice. With it: the server recognises the duplicate key and returns the original result.

---

### Order Status (WebSocket)

```
WS /orders/{id}/status
```

Real-time event stream for the active order screen. All events share a typed envelope to allow safe client-side dispatch.

**Event payload:**
```json
{
  "event": "order_updated",
  "data": {
    "order_id": "ord_999",
    "status": "cooking",
    "title": "Cooking",
    "description": "Chef is making your burger",
    "eta_timestamp": "2025-12-19T19:35:00Z"
  }
}
```

> **Why WebSocket over polling or push?** Short polling drains battery and burdens the backend. APNs push is unreliable for active screens (delivery delays up to 30s, no guarantee). WebSocket gives sub-second latency while the user is on-screen. APNs is used as a fallback when the app is backgrounded.

---

## Architecture Overview

<img width="1340" height="2356" alt="image" src="https://github.com/user-attachments/assets/53efaf23-f8fc-4484-9866-2c29f6730efe" />

---

## Layer-by-Layer Breakdown

### UI Layer — SwiftUI Views

- `MenuView` renders the restaurant menu with sticky category headers and dish cards, driven entirely by its ViewModel.
- `CartView` renders the current cart and total, with real-time counter updates via Optimistic UI.
- `OrderStatusView` renders live order status, driven by WebSocket events.
- Views hold **zero business logic**. They observe `@Published` properties and call ViewModel methods.
- `UICollectionViewDiffableDataSource` with `reconfigureItems` (not `reloadItems`) updates counters without redrawing images — eliminating the "flickering" problem on `+` / `-` taps.

---

### ViewModel Layer — Combine / ObservableObject

- Observes the local **CoreData** store as the single source of truth, driven by `NSFetchedResultsController`.
- Does not call the API directly. Delegates all data operations to the Service layer.
- Exposes `@Published` state for the View to observe.
- Never reads from in-flight API responses — only from the DB, ensuring consistency between screens.

---

### Navigation Layer — NavigationCoordinator

The `NavigationCoordinator` handles **navigation logic only**: push/pop, deep link routing, and push notification routing into the correct order status screen. It does **not** orchestrate data fetching or service calls.

> ⚠️ Do not confuse this with a data Coordinator. In MVVM-C, the Coordinator is a pure navigation concern. Data coordination belongs in the Repository.

---

### Service Layer

| Service | Responsibility |
|---------|---------------|
| `MenuService` | Menu loading, disk caching, offline availability. Owns `MenuRepository`. |
| `CartService` | Cart state management. Implemented as a Swift `actor` to prevent data races on concurrent taps. Handles Optimistic UI and rollback. Owns `CartRepository`. |
| `OrderService` | Order submission with idempotency, payment tokenisation, and status subscription lifecycle. Owns `OrderRepository`. |

**`CartService` as an `actor`:** All cart mutations (add, remove, change quantity) execute sequentially — even if the user rapidly taps `+` from multiple screens. On `addItem()`, the actor immediately publishes the updated state (Optimistic UI), then calls `CartRepository.sync()` in the background. If the server returns a validation error, the actor rolls back and re-publishes. This protects against data races without manual locking.

---

### Repository Layer

The Repository is the **coordination hub** between remote data sources, WebSocket events, and the local database.

**Responsibilities:**
- Fetches from `RemoteDataSource` or `WS DataSource` as appropriate.
- Writes all incoming data to `LocalDataSource` via **upsert** to deduplicate.
- Applies `CachePolicy` (TTL, disk budget, ETag) to decide when to hit the network vs. serve from cache.
- The DB is the single source of truth. ViewModels only ever read from the DB, never from in-flight responses.

| Repository | Strategy | Rationale |
|------------|----------|-----------|
| `MenuRepository` | **Cache First (Stale-While-Revalidate)** | Menu is relatively static. Show cached data instantly; revalidate in parallel. |
| `CartRepository` | **Network First** | Prices and availability change dynamically. Caching a stale cart is dangerous. |
| `OrderRepository` | **Transactional** | Errors cost real money. Uses idempotency key, handles ambiguous network failures with status fallback polling. |
| `OrderStatusRepository` | **WebSocket with Backoff** | Manages socket lifecycle (open on order created, close on delivered) and exponential backoff reconnection. |

---

### Data Sources Layer

| Source | Wraps | Role |
|--------|-------|------|
| `RemoteDataSource` | `APIClient` | REST calls — menu, cart sync, order submission |
| `WS DataSource` | `WebSocketManager` | Real-time order status event delivery |
| `LocalDataSource` | CoreData (upsert) | Persistent local storage, source of truth |

---

### Infrastructure Layer

**`APIClient`**
- Attaches `Authorization: Bearer <token>` headers on all requests.
- Attaches `Idempotency-Key` header on `POST /orders/submit`.
- Never passes `user_id` in request bodies — identity is always derived from the auth token server-side.

**`WebSocketManager`**
- Manages WebSocket lifecycle explicitly:
  - **Active screen:** Open socket on successful order creation; push `OrderStatusEvent` payloads to subscribers.
  - **Backgrounding:** Close the socket. Use **APNs silent push** to wake the app and fetch missed status updates via REST.
  - **Network changes (Wi-Fi → Cellular):** IP address changes break the TCP connection. On `onClose`, reconnect with **exponential backoff** (1s → 2s → 4s → 8s, capped at 60s).
  - **Heartbeat / ping-pong:** Keep the connection alive and detect silent failures.
- Exposed as an `AsyncStream<OrderStatusEvent>` to callers — no callback pyramids, no delegate boilerplate.

**`CoreData`**
- Source of truth for all UI state: menu items, cart contents, and order status.
- All writes use `NSMergeByPropertyObjectTrumpMergePolicy` to achieve upsert semantics — safely deduplicating menu items fetched from cache and the network simultaneously.

**`NSFetchedResultsController`**
- Sits between CoreData and the ViewModels. Observes the persistent store and delivers batched, diffable change notifications via its delegate.
- Each ViewModel holds its own `NSFetchedResultsController` instance scoped to its relevant fetch request (menu items by category, cart items, order status).
- Eliminates the need for ViewModels to poll or manually re-fetch — CoreData pushes changes up automatically whenever the store is written to.

**`SDWebImage`**
- Provides a two-tier cache: **memory cache** (fast, limited) → **disk cache** (persistent).
- Automatically falls back to a network fetch on cache miss.
- Used for both menu list thumbnails and full-res images in the dish detail view.
- Downsampling is applied on decode: a server-sent 4000×3000 image is downsampled to the `UIImageView` target size in pixels before entering memory cache, preventing OOM crashes during fast scrolling.

> **Why image caching matters beyond performance:** If your app consumes significant device storage, users scanning their storage list will target it for deletion. Monitoring and capping disk usage is a strategic product concern, not just an engineering one.

---

## Key Engineering Decisions

### 1. Hierarchical Menu Response (No N+1)

**Decision:** `GET /restaurants/{id}/menu` returns the full `Restaurant → Categories → Items → ModifierGroups → Options` tree in a single request.

**Why:** A naive design would fetch categories first, then items per category (N+1 requests). On a slow network, this produces multiple sequential spinners. A single structured response enables instant rendering and full offline caching in one network round-trip.

---

### 2. CartService as a Swift Actor

**Decision:** `CartService` is implemented as a Swift `actor`, not a class with a serial `DispatchQueue`.

**Why:** The user can tap `+` on the same item from multiple UI contexts simultaneously (menu cell and cart sheet both visible). An `actor` guarantees all mutations execute sequentially without manual locking. Data races — the most common source of silent cart bugs — are eliminated at the compiler level.

---

### 3. Optimistic UI with Rollback

**Decision:** Cart UI updates immediately on user tap, without waiting for `POST /cart/sync` to respond.

**Why:** 80% of cart sync requests succeed. Making 100% of users wait 0.5s to confirm every tap degrades the experience for the majority. The 20% failure case (item out of stock, network error) is handled via `CartService` rollback — the actor re-publishes the previous valid state and the UI shows an error alert.

---

### 4. Idempotent Order Submission

**Decision:** A UUID `Idempotency-Key` is generated client-side before the first payment attempt and reused on every retry of `POST /orders/submit`.

**Why:** Mobile networks are unreliable. The payment request may time out, leaving the client uncertain whether the server received it. On retry, the server uses the idempotency key to recognise the duplicate and return the original result — no duplicate charge. The same UUID is never regenerated mid-session; generating a new key on retry defeats the purpose entirely.

---

### 5. WebSocket for Order Tracking + APNs Fallback

**Decision:** WebSocket while user is on the order status screen; APNs silent push when backgrounded.

**Why:** Short polling (e.g. `GET /orders/{id}/status` every 5s) drains battery and generates unnecessary backend load. APNs alone has up to 30s delivery delays and no delivery guarantee — unacceptable for a live "Cooking → On the way" status screen. The hybrid approach gives real-time responsiveness on the active screen and battery-safe background updates.

---

### 6. Cache First (Stale-While-Revalidate) for Menu

**Decision:** `MenuRepository` immediately returns the cached menu from CoreData, then revalidates from the network in parallel.

**Why:** Restaurant menus change infrequently. Showing a slightly stale menu instantly is far better UX than a 1–2 second spinner on every app launch. TTL is set to 1 hour. ETag headers allow the server to return `304 Not Modified` when the menu hasn't changed, saving bandwidth.

---

### 7. Repository Owns `cart_id` State

**Decision:** `CartRepository` stores and manages the `cart_id` session token. `CartService` and ViewModels are never aware of it.

**Why:** `cart_id` is a backend session concern, not a business logic concern. Letting it leak into the service or ViewModel layer creates tight coupling between the UI lifecycle and the cart session. The repository abstracts this: callers just send items; the repository handles session continuity.

---

## Trade-offs & Considerations

| Topic | Trade-off |
|-------|-----------|
| **Optimistic UI vs. consistency** | Cart updates instantly, but a server rejection requires rollback and an error alert. Acceptable: 80% of requests succeed. Alternative (pessimistic UI) makes all users wait for every tap. |
| **Stale menu data** | Cache-first shows prices that may have changed. Mitigated by 1-hour TTL and server-side validation at cart sync time — the server catches price mismatches before checkout. |
| **Ambiguous payment failure** | If `POST /orders/submit` times out, the client doesn't know if the order was created. Solution: fall back to `GET /orders/{id}` to check; if found, open WebSocket. If not found, retry with the same `Idempotency-Key`. |
| **WebSocket battery drain** | A persistent socket has a real battery cost. Mitigated by closing it on background (app suspended anyway) and only opening it after a confirmed order — not on menu or cart screens. |
| **Actor re-entrancy** | Swift `actor` methods can interleave at `await` suspension points. Cart mutations must be designed to be safe under re-entrancy — specifically, read-modify-write sequences should not span an `await`. |
| **Image memory on large menus** | A 500-item menu with thumbnails can exhaust memory. Mitigated by: SDWebImage's LRU memory cache with size cap, `UICollectionViewDataSourcePrefetching` for smart preloading, and downsampling images to the display size on decode. |
| **Modifier validation client-side** | `min_selection` / `max_selection` are enforced in the UI (disable "Add to Cart" until required modifiers are selected). But the server also validates — client-side validation is UX only, never a security boundary. |
| **Cart cross-device sync** | `POST /cart/sync` makes the server the source of truth, so switching from iPhone to iPad picks up the same `cart_id`. Requires the user to be authenticated; anonymous carts are device-local only. |

---

## Tech Stack (iOS)

| Concern | Technology |
|---------|-----------|
| UI | SwiftUI |
| Reactive bindings | Combine |
| Local persistence | CoreData |
| DB → VM observation | NSFetchedResultsController |
| Image caching | SDWebImage |
| Network monitoring | NWPathMonitor |
| Real-time | WebSocket (URLSessionWebSocketTask + AsyncStream) |
| Concurrency | Swift Concurrency (`actor`, `async/await`, `AsyncStream`) |
| REST | URLSession + Codable |
| Payment tokenisation | Apple Pay / Stripe SDK |

---

## Resources

- [Uber Eats iOS System Design — Notion](https://boom-vulcanodon-64c.notion.site/System-Design-Interview-Simulation-Uber-Eats-IOS-2cd8d04671df8008b292e6dc15a48ede)

---

*This document reflects the design decisions from a mobile system design interview simulation, incorporating the Uber Eats ordering flow: menu fetching, cart synchronisation, idempotent order submission, and real-time order status tracking via WebSocket.*
