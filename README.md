# twinkit

Shared Go libraries for building [WonderTwin](https://github.com/wondertwin-ai/wondertwin) twins — behavioral clones of third-party APIs used for local development and testing.

```
go get github.com/wondertwin-ai/twinkit@latest
```

## Packages

| Package | Purpose |
|---------|---------|
| `twincore` | HTTP server, CLI flags, middleware (CORS, request logging, fault injection, latency, idempotency) |
| `store` | Generic in-memory data store with pagination, filtering, and snapshots |
| `admin` | Standard `/admin/*` control plane endpoints shared by all twins |
| `webhook` | Outbound webhook dispatcher with retry and pluggable signing |
| `testutil` | Test helpers — HTTP client with JSON assertions and admin client |

## Quick Start

A minimal twin that serves a single resource:

```go
package main

import (
    "net/http"

    "github.com/wondertwin-ai/twinkit/admin"
    "github.com/wondertwin-ai/twinkit/store"
    "github.com/wondertwin-ai/twinkit/twincore"
)

type AppState struct {
    Items *store.Store[Item]
}

type Item struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}

func (s *AppState) Snapshot() any              { return s.Items.Snapshot() }
func (s *AppState) LoadState(data []byte) error { return s.Items.UnmarshalJSON(data) }
func (s *AppState) Reset()                     { s.Items.Reset() }

func main() {
    cfg := twincore.ParseFlags("my-api")
    twin := twincore.New(cfg)

    state := &AppState{Items: store.New[Item]("item")}

    ah := admin.NewHandler(state, twin.Middleware(), nil)
    ah.Routes(twin.Router)

    twin.Router.Get("/items", func(w http.ResponseWriter, r *http.Request) {
        twincore.JSON(w, http.StatusOK, state.Items.List())
    })

    twin.Serve()
}
```

Run it:

```
go run . --port 9000
```

Every twin gets these admin endpoints for free:

```
GET  /admin/health          # health check
POST /admin/reset           # reset all state
GET  /admin/state           # snapshot current state
POST /admin/state           # load state from JSON
POST /admin/fault/{path}    # inject fault on an endpoint
DELETE /admin/fault/{path}  # remove fault
GET  /admin/faults          # list active faults
GET  /admin/requests        # request log
POST /admin/time/advance    # advance simulated clock
GET  /admin/time            # current real + simulated time
POST /admin/webhooks/flush  # flush pending webhooks
```

## twincore

Server scaffolding, CLI flag parsing, middleware, and HTTP helpers.

### Config & Server

```go
// Parse standard twin CLI flags (--port, --latency, --fail-rate, --webhook-url, --seed, --verbose)
cfg := twincore.ParseFlags("stripe")

// Create a twin with chi router, logger, and middleware pre-wired
twin := twincore.New(cfg)

// Access the middleware to wire into admin handler
mw := twin.Middleware()

// Mount your routes on twin.Router (a chi.Mux)
twin.Router.Get("/v1/customers", handleListCustomers)

// Start serving (blocks until SIGINT/SIGTERM)
twin.Serve()
```

### Middleware

All middleware is pre-wired by `twincore.New()`. The middleware stack applies in order: CORS, request logging, latency injection, random failure, fault injection.

```go
mw := twin.Middleware()

// Access request log entries
entries := mw.ReqLog.Entries()
mw.ReqLog.Clear()

// Programmatic fault injection
mw.Faults.Set("/v1/customers", twincore.FaultConfig{
    StatusCode: 503,
    Body:       `{"error": "service unavailable"}`,
    Rate:       1.0, // 100% of requests
})
mw.Faults.Remove("/v1/customers")
mw.Faults.Reset()

// Idempotency tracking (for Stripe-style idempotency keys)
statusCode, body, found := mw.Idempotent.Check("idem_123")
mw.Idempotent.Store("idem_123", 200, responseBytes)
mw.Idempotent.Reset()
```

### HTTP Helpers

```go
twincore.JSON(w, http.StatusOK, payload)                        // JSON response
twincore.Error(w, http.StatusBadRequest, "invalid input")       // {"error": {"message": "..."}}
twincore.StripeError(w, 400, "card_error", "expired", "Card expired") // Stripe error format
```

## store

Generic, thread-safe, in-memory data store with deterministic IDs, cursor pagination, and JSON serialization.

```go
type Customer struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

customers := store.New[Customer]("cus") // IDs: cus_000001, cus_000002, ...

// CRUD
id := customers.NextID()
customers.Set(id, Customer{ID: id, Name: "Alice", Email: "alice@example.com"})
cust, ok := customers.Get("cus_000001")
customers.Delete("cus_000001")

// List & filter
all := customers.List()
ids := customers.ListIDs()
count := customers.Count()
active := customers.Filter(func(id string, c Customer) bool {
    return c.Email != ""
})

// Cursor pagination (Stripe-style)
page := customers.Paginate("", 10)     // first page
page = customers.Paginate(page.Cursor, 10) // next page
// page.Data, page.HasMore, page.Cursor, page.Total

// State management (used by admin endpoints)
snapshot := customers.Snapshot()       // map[string]Customer
customers.LoadSnapshot(snapshot)       // replace all data
customers.Reset()                      // clear everything

// JSON serialization
data, _ := json.Marshal(customers)
json.Unmarshal(data, customers)
```

### Simulated Clock

```go
clock := store.NewClock()
now := clock.Now()           // real time + offset
clock.Advance(24 * time.Hour)
clock.Offset()               // time.Duration
clock.Reset()                // back to real time
```

## admin

Standard `/admin/*` control plane endpoints. Every twin mounts these for state inspection, fault injection, and time simulation.

Your state type must implement `admin.StateStore`:

```go
type StateStore interface {
    Snapshot() any
    LoadState(data []byte) error
    Reset()
}
```

Optionally implement `admin.WebhookFlusher` for webhook flush support:

```go
type WebhookFlusher interface {
    FlushWebhooks() error
}
```

Wiring:

```go
ah := admin.NewHandler(state, twin.Middleware(), clock)
ah.SetFlusher(webhookDispatcher) // optional
ah.Routes(twin.Router)
```

## webhook

Outbound webhook dispatcher with configurable retry, pluggable signing, and queue-based delivery.

```go
dispatcher := webhook.NewDispatcher(webhook.Config{
    URL:         "https://example.com/webhook",
    Secret:      "whsec_test123",
    Signer:      myStripeSigner{}, // implements webhook.Signer
    MaxRetries:  3,                // default: 3
    RetryDelay:  time.Second,      // default: 1s
    EventPrefix: "evt",            // default: "evt" → IDs like evt_000001
    AutoDeliver: false,            // true = deliver async on enqueue
})

// Queue an event
evt := dispatcher.Enqueue("customer.created", map[string]any{
    "id":   "cus_000001",
    "name": "Alice",
})

// Deliver all queued events (synchronous)
dispatcher.Flush()

// Inspect
deliveries := dispatcher.Deliveries()  // delivery attempts with status codes
queued := dispatcher.QueuedEvents()     // pending events
all := dispatcher.AllEvents()

// Implements admin.WebhookFlusher
dispatcher.FlushWebhooks()

// Reset everything
dispatcher.Reset()
```

Custom signing — implement `webhook.Signer`:

```go
type Signer interface {
    Sign(payload []byte, secret string) map[string]string
}
```

## testutil

HTTP test client with JSON assertions and an admin client for testing twin endpoints.

```go
func TestCustomerCreate(t *testing.T) {
    // Set up twin as test server
    server := httptest.NewServer(twin.Router)
    defer server.Close()

    client := testutil.NewTwinClient(t, server)
    adm := testutil.NewAdminClient(client)

    // Reset state before test
    adm.Reset().AssertStatus(200)

    // Create a customer
    resp := client.Post("/v1/customers", map[string]any{
        "name":  "Alice",
        "email": "alice@example.com",
    })
    resp.AssertStatus(200)

    // Parse response
    var cust map[string]any
    resp.JSON(&cust)

    // Or use JSONMap for quick access
    data := resp.JSONMap()
    assert.Equal(t, "Alice", data["name"])

    // Assert body contains substring
    resp.AssertBodyContains("alice@example.com")

    // Verify state
    adm.GetState().AssertStatus(200)

    // Test fault injection
    adm.InjectFault("v1/customers", twincore.FaultConfig{
        StatusCode: 500,
        Rate:       1.0,
    }).AssertStatus(200)

    client.Get("/v1/customers").AssertStatus(500)
    adm.RemoveFault("v1/customers")

    // Custom headers (e.g., idempotency keys)
    client.DoWithHeaders("POST", "/v1/customers", body, map[string]string{
        "Idempotency-Key": "idem_123",
    })

    // Form-encoded POST
    client.PostForm("/v1/tokens", map[string]string{
        "card[number]": "4242424242424242",
    })
}
```

### AdminClient Methods

| Method | Endpoint |
|--------|----------|
| `Reset()` | `POST /admin/reset` |
| `GetState()` | `GET /admin/state` |
| `LoadState(v)` | `POST /admin/state` |
| `InjectFault(endpoint, fault)` | `POST /admin/fault/{endpoint}` |
| `RemoveFault(endpoint)` | `DELETE /admin/fault/{endpoint}` |
| `GetRequests()` | `GET /admin/requests` |
| `FlushWebhooks()` | `POST /admin/webhooks/flush` |
| `AdvanceTime(duration)` | `POST /admin/time/advance` |
| `Health()` | `GET /admin/health` |

All methods return `*Response` and are chainable with `.AssertStatus()`.

## License

MIT
