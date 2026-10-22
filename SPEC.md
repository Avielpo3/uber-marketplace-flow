# Uber Marketplace Flow — Technical Spec

- **Version:** v6, 2026-10-22
- **Market:** ph-uk (EMEA)
- **Scope:** DaaS side only. Repos: `daas-uber`, `daas-gateway`, `tracker-service`, `mp-uber`, plus the DevOps infra task.
- **Out of scope:** all Algo implementation. It belongs to [ALGO-2532](https://dragontailsystems.atlassian.net/browse/ALGO-2532). This spec only states what the DaaS side expects from Algo (§8.0).
- **Jira:** epic [DAAS-634](https://dragontailsystems.atlassian.net/browse/DAAS-634) (relates to ALGO-2532). Stories DAAS-635 (daas-uber), DAAS-636 (daas-gateway), DAAS-637 (tracker-service), DAAS-638 (mp-uber). Infra task DAAS-639 (DevOps, blocks DAAS-637).
- **Visual version:** https://avielpo3.github.io/uber-marketplace-flow/
- **Status:** design agreed, ready for an implementation plan. Open items are in §11. In-scope items have an owner, and out-of-scope items are marked as such.

File:line references point at the local checkouts under `~/dragontail/<repo>` as of 2026-10-22. Re-check them before editing.

---

## 1. Summary

When an Uber Eats order arrives and no store driver is free in time, Algo books an Uber Direct courier through daas-uber. Uber Eats is never told to switch fulfilment. The order stays a restaurant delivery (BYOC) from start to end, and Uber Eats only receives **order statuses** and **courier locations**.

mp-uber does not know which kind of carrier delivers an order, a store driver or a DaaS courier. Both kinds reach it the same way:

- **Order registration:** Algo sends the existing `NewOrderMakeline` status call. It replaces the switch. mp-uber inserts its row and subscribes to the order in tracker-service, as it already does today.
- **Locations and statuses:** tracker-service's existing **forwarding job** pushes them to mp-uber's existing subscription callback `/mp/uber/order/events`. **Nothing polls.**
- **DaaS courier data:** for marketplace deliveries, daas-uber sends **every** Uber Direct callback to a new gateway route. The gateway publishes it to a new RabbitMQ queue, and tracker-service consumes the queue and saves to Redis.

## 2. Goals and non-goals

**Goals**
1. Deliver Uber Eats marketplace orders with an Uber Direct courier (through daas-uber) when no store driver is free within the threshold.
2. Uber Eats gets the same status updates (`started`, `delivered`) and live courier location for DaaS-courier orders as for store-driver orders.
3. mp-uber stays carrier-agnostic.
4. Every call between services goes through daas-gateway.

**Non-goals (out of scope)**
- **Any change in the `algo` repo.** It is implemented under ALGO-2532. Algo appears here only as the caller whose behaviour the DaaS side relies on (§8.0). Do not plan Algo tasks from this spec.
- Handling Uber Direct `canceled` / `returned` after the order is registered (§11, O-4).
- Enforcing the Uber Direct webhook signature (`Uber.AuthWebhook`) (§11, O-5).
- Auth on the new gateway route beyond what `/callbacks` has today (pilot decision D-11).
- Provisioning infrastructure (Redis/Valkey, tracker-service ECS). DevOps does this in a separate ticket (D-14).
- Changes to location-service, the old `tracker` repo, the Proxy mobile-app flow and DragonDrive.

## 3. Glossary

| Term | Meaning |
|---|---|
| Store driver | Algo's internal carrier. Its locations come from the mobile app through the Proxy to tracker-service `POST /events/location`. |
| DaaS courier | Uber Direct courier booked through daas-uber. |
| Marketplace delivery | A daas-uber delivery created with `"marketplace": "uber"`. |
| Marketplace callback | Any Uber Direct webhook (`courier_status` or `delivery_status`) for a marketplace delivery. |
| Subscription | tracker v2 subscription that mp-uber creates on `NewOrderMakeline`, with callback `{gateway}/mp/uber/order/events` and events `location` and `order`. |
| Forwarding job | Existing tracker-service job that runs every interval and pushes new points and events to subscribers. |

## 4. Decisions

Each decision is final unless §11 says otherwise. Decisions marked **[Algo: ALGO-2532]** need Algo work, which is out of scope here. In this spec they only define the contract the DaaS side relies on.

| ID | Decision |
|---|---|
| D-1 | **The switch is removed.** Nothing calls Uber Eats `switch-fulfillment-type` (`DELIVERY_BY_UBER`). DaaS side: `/mp/{provider}/order/switch` is deleted in daas-gateway and mp-uber. **[Algo: ALGO-2532]** Algo stops calling the switch and keeps the order as a delivery. |
| D-2 | **One call to mp-uber to register the order: `NewOrderMakeline`.** The switch, the proposed create-delivery call and `NewOrderMakeline` are merged into the existing `POST /mp/uber/order/status` with `orderStatus: "NewOrderMakeline"`. It is sent in **both** branches (store driver and DaaS). No new mp-uber endpoint. **[Algo: ALGO-2532]** |
| D-3 | **Carrier choice is unchanged.** If a store driver is free within 30 min, the store driver delivers. Otherwise Algo books daas-uber. **[Algo: ALGO-2532]** |
| D-4 | **daas-uber only.** **[Algo: ALGO-2532]** For marketplace orders Algo sends create delivery with `aggProvider: "uber"`. DaaS side: **the gateway honours `aggProvider`** and forwards to that provider only, with no fan-out. |
| D-5 | **The `marketplace` field.** **[Algo: ALGO-2532]** Algo adds `"marketplace": "uber"` to the create-delivery JSON. DaaS side: daas-uber stores it in `algo_discovery` (`marketplace`, `marketplace_order_id` = `altOrderId`). |
| D-6 | **All marketplace callbacks are forwarded.** For a marketplace delivery, daas-uber sends **every** Uber Direct callback (every status, with or without a location) to the new gateway route. The existing callback to Algo (`/callbacks/…`) is unchanged and still sent first. |
| D-7 | **New gateway route `POST /marketplace/{provider}/callbacks`** (`/marketplace/uber/callbacks`). It replaces the earlier locations-only idea. It sits outside the `/mp/*` reverse proxy. |
| D-8 | **New RabbitMQ queue `marketplace_callbacks`**, between daas-gateway (publisher) and tracker-service (consumer). It is durable and persistent, uses the default exchange, has DLQ `marketplace_callbacks_DLQ`, and messages expire after 15 minutes. |
| D-9 | **tracker-service (the new service, `~/dragontail/tracker-service`) stores in Redis** with a 24 h TTL. Every callback goes to a per-order events list. Every location goes to the existing per-order locations list, the same key store-driver points use. **The list is capped at 360 points** (`REDIS_MAX_LOCATIONS_PER_ORDER=360`, raised from 100, `configuration.yaml:53`). 100 points is about 33 min at one point every 20 s, and 360 is about 2 h. The 24 h TTL matches the existing `orderLocations` setting (`configuration.yaml:55-58`) and is refreshed on every write. |
| D-10 | **The market comes from daas-uber.** daas-uber puts `market` (e.g. `ph-uk`) in the callback body, resolved from brand+country in its config, with a new default `AGG_DEFAULTMARKET` (`ph-uk` in EMEA). tracker-service falls back to `DEFAULT_MARKET` when `market` is empty. |
| D-11 | **No auth on the new route for the pilot**, the same as `/callbacks`. Later, add a service JWT on both routes by mounting the existing `Authenticator` (`daas-gateway api/middleware/authenticator.go:14`, defined but never mounted today). |
| D-12 | **No polling. Use the existing subscription.** tracker-service's forwarding job pushes new points **and order events** to mp-uber's subscription (`/mp/uber/order/events`). There is no poller, no new mp-uber job, no `pullLocations` or `lastLocationTs` column, and no push-to-pull cutover. |
| D-13 | **Statuses are pushed too.** Uber Direct statuses reach mp-uber as tracker order events, and mp-uber turns them into Uber Eats `started` / `delivered`. Algo still sends `EnRoute` / `Delivered` through `/order/status`. mp-uber's existing 30 min dedup per order and status prevents double calls to Uber Eats. |
| D-14 | **Infrastructure is DevOps, in a separate ticket:** EMEA Redis/Valkey (TLS) and the tracker-service EMEA deployment, plus RabbitMQ access for the queue. |
| D-15 | **Out of scope:** Uber Direct cancel/return handling, and the webhook signature check. |
| D-16 | **Uber Eats only receives statuses and locations:** `restaurantdelivery/status` and `byoc/…/event/location`. |

## 5. Flow

```
Algo ──NewOrderMakeline──▶ gateway /mp/uber/order/status ──▶ mp-uber ─(row 'received' + subscribe)─▶ tracker-service
Algo ──create delivery (aggProvider=uber, marketplace=uber)──▶ gateway /api/v2/deliveries ──▶ daas-uber ──▶ Uber Direct
Uber Direct ──webhooks (~20 s)──▶ daas-uber
   daas-uber ──/callbacks/… (unchanged)──▶ gateway ──▶ store callback queue ──▶ Algo
   daas-uber ──/marketplace/uber/callbacks (every callback)──▶ gateway ──publish──▶ RabbitMQ marketplace_callbacks ──consume──▶ tracker-service ──▶ Redis
tracker-service forwarding job ──push points + order events──▶ gateway /mp/uber/order/events ──▶ mp-uber ──▶ Uber Eats (status, BYOC location)
Algo ──EnRoute / Delivered (as today)──▶ gateway /mp/uber/order/status ──▶ mp-uber ──▶ Uber Eats (deduplicated)
Proxy (store-driver app) ──/events/location (as today)──▶ tracker-service (same per-order locations key)
```

### Steps

1. **Algo registers the order** (Algo side is ALGO-2532, out of scope). Algo picks the carrier (D-3) and sends `NewOrderMakeline` in both branches (D-2). DaaS side: mp-uber inserts its `marketplaceorderstatus` row as `received` and creates the tracker subscription (`SubscribeToLocationsOnStatus = NewOrderMakeline`, `mp-uber configuration.yaml:23`, `uberMpEventsService.go:42, :99-116`). Later, Algo sends `EnRoute` and `Delivered` as today. mp-uber turns these into Uber Eats statuses through `TriggerUpdatesOrderStatuses` (default `EnRoute,Delivered`, `mp-uber configuration.yaml:22`).
2. **Algo books daas-uber only** when the store driver is too far (ALGO-2532, out of scope). See D-4 and D-5. DaaS side: the gateway forwards it only to daas-uber. If the booking fails, Algo falls back to the store driver. mp-uber is not affected, because nothing was promised to Uber Eats.
3. **daas-uber marks the delivery** in `algo_discovery` (D-5).
4. **daas-uber forwards callbacks.** For each Uber Direct webhook of a marketplace delivery: first the existing Algo callback, then `POST {CallbackURL}/marketplace/uber/callbacks` in a goroutine (D-6). Up to 3 attempts, backoff 1 s then 2 s, 3 s timeout each, failures logged. It never blocks the Algo callback.
5. **The gateway publishes** the body unchanged to `marketplace_callbacks` and replies `202` (D-7, D-8).
6. **tracker-service consumes and stores.** It writes the event list, and the locations list when the callback has a location (D-9). It adds the new items to the order's forwarding source (`tr-svc:fwd:…`).
7. **The forwarding job pushes** to `{gateway}/mp/uber/order/events`. The subscription was created on `NewOrderMakeline` with callback `/mp/uber/order/events` and events `location` and `order` (`mp-uber services/trackerV2Service.go:30-80`). mp-uber's existing `ProcessOrderEventsBatch` (`uberMpEventsService.go:120-170`) handles order events (statuses, D-13) first. Once the order is `started` (`:162-164`), it sends the newest point with `processLocationEvents` (`:197-251`). `externalStoreId` comes from the subscription key, as today. The push interval is the existing forwarding job's (`tracker-service configuration/startup.go:115-120`).

## 6. Contracts

### 6.1 Algo → mp-uber status (existing, unchanged shape; Algo side is ALGO-2532)

```http
POST {gateway}/mp/uber/order/status
{
  "aggOrderId": "d3b93b60-c336-4b32-87e6-bed8eb0c2cab",
  "brand": "Pizza Hut:GH Pizza LTD",
  "country": "United Kingdom",
  "storeNo": "7507",
  "orderId": "26101402978",
  "orderStatus": "NewOrderMakeline" | "EnRoute" | "Delivered"
}
```
- `NewOrderMakeline`: row `received` + tracker subscription. It replaces the switch.
- `EnRoute`: Uber Eats `started`. `Delivered`: Uber Eats `delivered`. Both are deduplicated for 30 min.

### 6.2 Algo → daas-uber create delivery (excerpt, new fields marked; Algo side is ALGO-2532, the DaaS side consumes it)

```http
POST {gateway}/api/v2/deliveries
{
  "aggProvider": "uber",            // NEW: forced for marketplace orders (D-4)
  "storeId": "7507",
  "brand": "Pizza Hut:GH Pizza LTD-United Kingdom",
  "targets": [{ "details": { "id": "26101402978", "altOrderId": "d3b93b60-c336-4b32-87e6-bed8eb0c2cab" } }],
  "marketplace": "uber"             // NEW (D-5)
}
```
The gateway forwards it to daas-uber `POST /api/v1/deliveries/`.

### 6.3 daas-uber → gateway → queue (new)

```http
POST {gateway}/marketplace/uber/callbacks      → 202 Accepted
queue: marketplace_callbacks (same body)
{
  "marketplace": "uber",
  "market": "ph-uk",
  "brand": "Pizza Hut",
  "franchisee": "GH Pizza LTD",
  "country": "United Kingdom",
  "storeNo": "7507",
  "orderId": "26101402978",
  "deliveryId": "del_UowjW4AJTSGDNwqOOFCwCg",
  "event": "courier_status",          // or "delivery_status"
  "status": "dropoff",                // raw Uber Direct status
  "targetStatus": 3,                  // Algo target status number
  "courierStatus": 5,                 // Algo courier status number
  "location": { "lat": 51.913002, "lng": -0.20857728 },   // null on pending, delivered, canceled, returned
  "timestamp": 1791975067000
}
```
The gateway requires `orderId`, `storeNo` and `status`. `location` is optional.

### 6.4 Redis (tracker-service)

| Key | Value | TTL / limit |
|---|---|---|
| `tr-svc:market:{m}:store:{s}:posorder:{o}:locations` (existing, shared with store-driver points) | `LocationDto` JSON `{"lat","lng","timestamp"}` | 24 h, refreshed on every write. Trimmed to 360. |
| `tr-svc:market:{m}:store:{s}:posorder:{o}:mpcallbacks` (new, **name proposed**) | full callback JSON | 24 h |
| `tr-svc:fwd:…` (existing forwarding source) | new items to forward | existing `sourceTtl` 15 min |

### 6.5 tracker-service → mp-uber (existing subscription push)

`POST {gateway}/mp/uber/order/events` (mp-uber role `tracker`, `controllers/marketplaceOrderEventsController.go:18-23`) carries the key, new location events and order events, in the existing tracker v2 forwarding format. mp-uber's controller rejects points with lat/lng = 0 or older than 2 h (existing, `:92-98`).

### 6.6 mp-uber → Uber Eats (existing)

- `POST /eats/orders/{aggOrderId}/restaurantdelivery/status` `{ "status": "started" | "delivered" }`
- `POST /eats/byoc/restaurants/orders/event/location` `{ "location_request": { "order_workflow_uuid": aggOrderId, "restaurant_uuid": externalStoreId, "location_events": [...] } }`

## 7. Status mapping

Algo mapping as in `daas-uber providers/uber/delivery.go:414-439`.

| Uber Direct | Location | Algo target / courier | Algo order | Algo → mp-uber | Uber Eats | Pushed to mp-uber |
|---|---|---|---|---|---|---|
| pending | no | 1 Scheduled / 0 Unassigned | – | – (NewOrderMakeline already sent) | – | order event |
| pickup | yes | 2 Assigned / 2 EnrouteToPickup, 3 ArrivedAtStore | – | – | – | order event + point |
| pickup_complete | yes | 2 Assigned / 3 ArrivedAtStore | – | – | `started`? (O-1) | order event + point |
| dropoff | yes | 3 PickedUp / 5 EnrouteToDropoff, 8 Nearby | EnRoute (`AutoEnrouteOnAggPickup`) | EnRoute | `started` | order event + point |
| delivered | no | 4 Delivered / 7 DroppedOff | Delivered | Delivered | `delivered` | order event |
| canceled / returned | no | 5 Cancelled / 6 CouldNotDeliver | back to dispatch | – | – (out of scope, O-4) | order event |

Notes:
- Uber Eats gets `started` from whichever arrives first, the pushed order event or Algo's `EnRoute`.
- mp-uber sends locations only after `started` (`uberMpEventsService.go:162`). Earlier points are stored but not sent.
- Uber Direct rarely sends `pickup_complete` (1 in 96 webhooks in the 2026-10-14 sample).

## 8. Changes per repo

Each item is one plan task. "Done when" is the acceptance check.

### 8.0 Algo: out of scope, what the DaaS side relies on (ALGO-2532)

No Algo tasks are planned from this spec. The DaaS side assumes Algo provides the following. Track it as a dependency on ALGO-2532.

| # | Algo provides | DaaS side depends on it in |
|---|---|---|
| A-1 | `NewOrderMakeline` to `/mp/uber/order/status` for every marketplace order, store driver or DaaS. No `/order/switch` call. | mp-uber row and subscription (§8.4). Removing the switch route (§8.2). |
| A-2 | Create delivery to `/api/v2/deliveries` with `aggProvider: "uber"`, `marketplace: "uber"`, and the Uber Eats order id in `targets[].details.altOrderId` (§6.2). | gateway aggProvider routing (§8.2), daas-uber marketplace flag (§8.1). |
| A-3 | `EnRoute` and `Delivered` to `/mp/uber/order/status` for DaaS-delivered orders (O-2). | mp-uber dedup with the pushed statuses (§8.4). |
| A-4 | The order stays a delivery (no carry-out), and the Uber Eats `aggOrderId` is kept on the order. | Uber Eats status and location calls. |
| A-5 | Algo still consumes daas-uber's existing `/callbacks/…` unchanged. | daas-uber keeps sending them first (§8.1). |

### 8.1 daas-uber (`~/dragontail/daas-uber`), DAAS-635

1. **Accept `marketplace`.** Add it to `parsers.Order` (`internal/server/parsers/v1.go:116`) and `algo.Delivery` (`internal/algo/deliveries.go:6`), and map it in `V1.ToAlgoDelivery` (`v1.go:35`). Today the field is silently dropped.
2. **Migration `migrations/19.add_marketplace_to_algo_discovery.sql`** (sql-migrate, `-- +migrate Up`): `marketplace text NULL`, `marketplace_order_id text NULL`.
3. **Carry the columns** through `OrderRequest` (`internal/algo/repository.go:21`), the `SaveOnce` insert (`:55`), the `GetOrderByAggID` select (`:100`) and the `getAlgoDeliveryByAggID` DB fallback (`internal/server/providers.go:165-193`). *Done when:* the flag survives a cache miss and a restart.
4. **Add `SendMarketplaceCallback`** in `internal/server/gatewayAPI.go` (next to `SendCallbackUpdate`, `:25`, same `CallbackURL`). Call it with `go` after the Algo callback in `Uber()` (`providers.go:52-145`, callback at `:136`), for every webhook where `marketplace != ""`. Body as in §6.3. 3 attempts, 3 s timeout each.
5. **Copy courier lat/lng whenever present**, not only when the courier name or vehicle is set (`providers/uber/delivery.go:372-381`).
6. **Market config:** add a brand+country → market map and `AGG_DEFAULTMARKET` (default `ph-uk`) to `Configuration` (`internal/config/config.go`, env prefix `AGG`).

### 8.2 daas-gateway (`~/dragontail/daas-gateway`), DAAS-636

1. **Add the `MarketplaceCallback` handler**, modelled on `api/handler/callback.go:59-110`. Decode, require `orderId`/`storeNo`/`status`, `queue.SendMessage` (`pkg/queue/client.go:252`) to `MarketplaceCallbacksQueue`, reply 202.
2. **Register `POST /marketplace/{provider}/callbacks`** in `api/handler/router.go`, next to the callbacks group (`:106-115`) and outside the `/mp/*` reverse proxy (`MakeMpProxyHandler`, `:148-160`), with no auth (D-11).
3. **Config:** add `MarketplaceCallbacksQueue` (default `marketplace_callbacks`) to `config/queue.go`, env `DT_QUEUE_MARKETPLACECALLBACKSQUEUE`.
4. **Remove `/mp/{provider}/order/switch`** from `MakeMpProxyHandler` (`router.go:153-155`). `/order/status` and `/order/events` stay.
5. **`/api/v2/deliveries` honours `aggProvider`** (D-4): forward straight to that provider with no quote fan-out (`router.go:92-103`). Check what it does today first.
6. **Handler test** with `httptest` and a buffered `messages` channel in the context.

### 8.3 tracker-service (`~/dragontail/tracker-service`, the new tracker, not `tracker` or `tracksvc`), DAAS-637

1. **RabbitMQ consumer:** add `amqp091-go` and `consumers/marketplaceCallbacks.go`, started from `StartUp()` (`configuration/startup.go:115`) behind `RABBITMQ_ENABLED`. Prefetch 50, manual ack, nack without requeue on bad JSON (goes to the DLQ), reconnect loop.
2. **Store:** take `market` from the message (fall back to `DEFAULT_MARKET`). Append the callback to `…:mpcallbacks` (24 h). If it has a location, write a `LocationDto` with the existing repository call (`dataAccess/repositories/ordersRepository.go:404-431`).
3. **Forward:** add each new point and order event to the order's forwarding source (`tr-svc:fwd:…`, `types/consts.go:34-36`), so the existing forwarding job pushes them to subscribers. Map Uber Direct statuses to the order-event types mp-uber understands (coordinate with O-1).
4. **Config:** add RabbitMQ settings to `configuration.yaml` (`${VAR||default}` pattern), and set `REDIS_MAX_LOCATIONS_PER_ORDER=360`.
5. **Dockerfile:** align the Go base image (1.23.4) with `go.mod` (1.25).
6. **Tests** (testify + `redismock`): callback with a location, callback without a location, bad JSON, unknown market.

### 8.4 mp-uber (`~/dragontail/mp-uber`), DAAS-638

1. **Delete the switch:** `POST order/switch` (`controllers/marketplaceOrderEventsController.go:65`), `SwitchToUberDelivery` (`services/uberMpEventsService.go:172-195`), `SwitchToUberDeliveryRequest` (`types/types.go:25`), the switch DTO (`types/mpDtos.go:36`) and `DELIVERY_BY_UBER` (`types/consts.go:41`).
2. **Keep** the `NewOrderMakeline` row insert and tracker subscription as they are. No new endpoint, job or migration.
3. **Order events:** in `processOrderEvents` (`uberMpEventsService.go:253-292`), map the order events that come from Uber Direct statuses to `started` / `delivered`.
4. **Dedup:** check that the 30 min status dedup (`:324`, key in `types/consts.go:15-20`) stops a double `started` / `delivered` when both the push and Algo send it.

### 8.5 Infra, DAAS-639 (DevOps, Ehab Jaber)

EMEA Redis/Valkey (TLS), RabbitMQ access for `marketplace_callbacks` and its DLQ, and the tracker-service ECS deployment in EMEA. It blocks DAAS-637 going live.

## 9. Failure handling

| What fails | Effect | Handling |
|---|---|---|
| `NewOrderMakeline` (1) | No row and no subscription, so no location reaches Uber Eats | 6 attempts (backoff 1–16 s). mp-uber also inserts the row and subscribes if the first call it sees is EnRoute. |
| daas-uber booking (2) | No courier | The store driver delivers. Uber Eats sees no difference. |
| Marketplace callback POST (4) | Callback missing from tracker. Algo still has it. | 3 attempts, then logged. A lost location is replaced by the next webhook (~20 s). A lost status is not, but Algo's status still reaches mp-uber. |
| Queue or tracker down (5, 6) | Items wait | Durable queue, 15 min expiry, DLQ. The gateway buffers 2000 messages in memory (`cmd/daas-gateway/server.go:98`). |
| Forwarding push or Uber Eats (7) | Position lags | The forwarding job's existing behaviour (source TTL 15 min). mp-uber sends only the newest point. |

## 10. Rollout and tests

**Release order**
1. DevOps (separate ticket): EMEA Redis/Valkey and the queue, and deploy tracker-service with the consumer.
2. daas-gateway with the new callbacks route. Keep the switch route until ALGO-2532 is released.
3. daas-uber with the migration. It only acts when `marketplace` is set.
4. mp-uber with the order-event mapping. The subscription is unchanged.
5. (Out of scope, ALGO-2532) Algo release with the switch replaced by `NewOrderMakeline`, piloted on one store. The DaaS side must be live before this.
6. Remove the switch route (daas-gateway) and the switch code (mp-uber).

**End-to-end checks (staging)**
- A store-driver order behaves exactly as today.
- A DaaS order: the `algo_discovery` row has `marketplace='uber'` and the Uber Eats UUID.
- A DaaS order: `NewOrderMakeline` creates the tracker subscription, and points from the queue are pushed to `/mp/uber/order/events`.
- Each webhook, including `delivered`, gives one Algo callback and one marketplace callback, both 202.
- The Redis locations key grows, its `TTL` is about 86400, and `LLEN` ≤ 360.
- Uber Eats gets `started`, then locations, then `delivered`, each status once.
- A normal (non-marketplace) daas-uber delivery sends no marketplace callback.

## 11. Open items

| ID | Item | Owner | Status |
|---|---|---|---|
| O-1 | Which Uber Direct status means `started` for Uber Eats? Algo only reaches EnRoute at `dropoff`. Now that order events are pushed, mp-uber can send `started` earlier. Options: `pickup` (courier assigned, driving to the store), `pickup_complete` (rarely sent by Uber), or `dropoff` (left the store, the same moment as Algo's EnRoute). | Sergey Shelekhov | Asked in Slack 2026-10-22 |
| O-2 | Does Algo send `EnRoute` / `Delivered` to `/mp/uber/order/status` when a DaaS carrier delivers? If not, statuses rely only on the push. | Algo team (ALGO-2532) | Dependency A-3 |
| O-3 | Confirm Uber accepts `restaurantdelivery/status` and BYOC location for DaaS-courier orders. Uber has returned 400 "Cannot update status of this order type" before (DAAS-633). | Unassigned | Verify in staging |
| O-4 | Uber Direct `canceled` / `returned` after registration: hand back to a store driver, rebook, or report a failure to Uber Eats. | – | Out of scope |
| O-5 | Uber Direct webhook signature (`Uber.AuthWebhook`, `providers/uber/auth.go:17`, never called). | – | Out of scope |
| O-6 | Name of the Redis events key (`…:mpcallbacks` is a placeholder). | Implementer | Pick during build |

## 12. Blockers

| Blocker | Owner |
|---|---|
| tracker-service is not deployed anywhere: README says "Not yet deployed", and there are no ECS/Terraform definitions in the checkout (`docs/cutover-2026-09-18/callers-report.md:23`). | DevOps, separate ticket |
| No Redis in EMEA prod (`docs/cutover-2026-09-18/uk-tracker-cutover.html:94`). ElastiCache would be Valkey, and serverless needs `REDIS_TLS` (`tracker-service/CLAUDE.md:121`). | DevOps, separate ticket |

## 13. Superseded ideas (do not implement)

| Idea | Replaced by |
|---|---|
| Algo asks Uber Eats to switch fulfilment (`/order/switch` → `switch-fulfillment-type`) | D-1, D-2 |
| daas-uber posts courier updates straight to mp-uber (`/mp/uber/order/courier`) | D-6 – D-9, D-12 |
| A new mp-uber "create delivery" endpoint (`/mp/uber/order/delivery`) and `orderId` on the switch | D-2 |
| A locations-only gateway route (`/marketplace/uber/locations`, queue `marketplace_courier_locations`) | D-6, D-7, D-8 |
| An mp-uber poller (`GET /tracker/.../locations` every 10 s, `pullLocations` and `lastLocationTs` columns) and the push-to-pull cutover | D-12 |
| tracker-service mapping brand+country to market | D-10 |

## 14. References

- Visual spec: https://avielpo3.github.io/uber-marketplace-flow/ and the claude.ai artifact "Uber Marketplace Flow"
- `docs/cutover-2026-09-18/uk-tracker-cutover.html`, `docs/cutover-2026-09-18/callers-report.md`
- `tracker-service/README.md`, `tracker-service/CLAUDE.md`
- Slack: questions to Sergey Shelekhov (DM, 2026-10-22). Spec shared with Sergey, Moshe Basall and Ariel Pinchover.
