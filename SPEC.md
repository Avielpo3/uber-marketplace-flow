# Uber Marketplace Flow — Technical Spec

- **Version:** v7, 2026-10-28 (v6 was the design of 2026-10-22; §13 lists what changed during implementation)
- **v8, 2026-10-29:** verified against the code deployed in QA (Algo 2.65.0_ALGO-2532, daas-uber 11.5.7, daas-gateway 1.6.0, tracker-service 2.0.4, mp-uber 0.1.9_2). Additions only; v7 text kept, corrections marked "As implemented (v8)".
- **Market:** ph-uk (EMEA)
- **Scope:** DaaS side only. Repos: `daas-uber`, `daas-gateway`, `tracker-service`, `mp-uber`, plus the DevOps infra task.
- **Out of scope:** all Algo implementation. It belongs to [ALGO-2532](https://dragontailsystems.atlassian.net/browse/ALGO-2532). This spec only states what the DaaS side expects from Algo (§8.0).
- **Jira:** epic [DAAS-634](https://dragontailsystems.atlassian.net/browse/DAAS-634) (relates to ALGO-2532). Stories DAAS-635 (daas-uber), DAAS-636 (daas-gateway), DAAS-637 (tracker-service), DAAS-638 (mp-uber). Infra task DAAS-639 (DevOps, blocks DAAS-637).
- **Visual version:** https://avielpo3.github.io/uber-marketplace-flow/
- **Status:** implemented on the four DaaS branches (§15), in dev test. Open items are in §11.

> **As implemented (v8, 2026-10-29):** implemented on the four DaaS branches and deployed in QA (daas-uber 11.5.7, daas-gateway 1.6.0, tracker-service 2.0.4, mp-uber 0.1.9_2; §23). Jira on 2026-10-29 shows DAAS-635, DAAS-636 and DAAS-637 in Peer Review and DAAS-638 in Dev Test, so "in dev test" holds for DAAS-638 only (Jira GET 2026-10-29; QA state in §23)


File:line references point at the implementation branches as of 2026-10-28. Re-check them before editing.

> **As implemented (v8, 2026-10-29):** the file:line references in this spec were re-checked against the deployed tags (daas-uber 11.5.7_dev, daas-gateway 1.6.0_dev, tracker-service 2.0.4_dev, mp-uber 0.1.9_2_dev, algo 2.65.0_ALGO-2532). Four mp-uber refs and two Algo refs are wrong, and the daas-uber path lacks `internal/server/`; the corrected table is in §21.2 (research 08, git show of the deployed tags)


---

## 1. Summary

When an Uber Eats order arrives and no store driver is free in time, Algo books an Uber Direct courier through daas-uber. Uber Eats is never told to switch fulfilment. The order stays a restaurant delivery (BYOC) from start to end, and Uber Eats only receives **order statuses** and **courier locations**.

mp-uber does not know which kind of carrier delivers an order, a store driver or a DaaS courier. Both kinds reach it the same way:

- **Order registration:** Algo sends the existing `NewOrderMakeline` status call. mp-uber inserts its row and subscribes to the order in tracker-service, as it already does today. The switch stays in mp-uber but is **disabled by configuration** and answers 404.
- **DaaS courier data:** for marketplace deliveries, daas-uber sends **every** Uber Direct callback to a new gateway route. The gateway publishes it to a new RabbitMQ queue, and tracker-service consumes it and runs it through its **existing Order and Location event flows**, as if Proxy had sent them.
- **Statuses and locations to mp-uber:** through the existing tracker subscription callback `/mp/uber/order/events`. Order events go out **immediately** (the existing order fan-out); courier points go out through the existing **forwarding job**. **Nothing polls.**

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
| Marketplace callback | The callback daas-uber sends to the gateway for every Uber Direct webhook (`courier_update` or `delivery_status`) of a marketplace delivery. It carries an `orderStatusUpdate`, a `locationUpdate`, or both (§6.3). |
| Algo order id | Algo's own order id (`order.OrderId`). It is the `orderId` of the marketplace callback, the `orderId` Algo sends mp-uber, and the order id of mp-uber's tracker subscription. |
| Subscription | tracker v2 subscription that mp-uber creates on `NewOrderMakeline`, with callback `{gateway}/mp/uber/order/events`, events `location` and `order`, and the Algo order id. |
| Order fan-out | tracker-service's existing immediate push of an order event to the order's subscriptions (`SubscriptionsService.ProcessOrderSubscriptions`), used by `/events/order` and Algo notifications. |
| Forwarding job | Existing tracker-service job that runs every 30 s and pushes new courier points to subscribers. It carries locations only. |

## 4. Decisions

Each decision is final unless §11 says otherwise. Decisions marked **[Algo: ALGO-2532]** need Algo work, which is out of scope here. In this spec they only define the contract the DaaS side relies on. Decisions marked *changed* differ from v6 (see §13).

| ID | Decision |
|---|---|
| D-1 | *changed* **The switch is not used, but not deleted.** Nothing calls Uber Eats `switch-fulfillment-type` (`DELIVERY_BY_UBER`) while mp-uber's `switchToUberDeliveryEnabled` (`MARKETPLACE_SWITCHTOUBERDELIVERY_ENABLED`) is `false`, the default. `POST /mp/uber/order/switch` then answers `404 {"error": "switch to Uber delivery is disabled"}`. The gateway proxy route stays. **[Algo: ALGO-2532]** Algo stops calling the switch; if it still does, the 404 makes it fall back to `stayWithStoreCarrier`, which sends `NewOrderMakeline`. |
| D-2 | **One call to mp-uber to register the order: `NewOrderMakeline`** through the existing `POST /mp/uber/order/status`. It is sent in **both** branches (store driver and DaaS). No new mp-uber endpoint. **[Algo: ALGO-2532]** |
| D-3 | **Carrier choice is unchanged.** If a store driver is free within 30 min, the store driver delivers. Otherwise Algo books daas-uber. **[Algo: ALGO-2532]** |
| D-4 | **daas-uber only.** **[Algo: ALGO-2532]** For marketplace orders Algo sends create delivery with `aggProvider: "uber"`. DaaS side: the gateway already honours `aggProvider` (on `master`, `api/handler/proxy.go`): `/api/v2/deliveries` forwards to that provider only, and `/quotes` skips the fan-out when it is set. |
| D-5 | *changed* **The `marketplace` field.** **[Algo: ALGO-2532]** Algo adds `"marketplace": "uber"` to the create-delivery JSON. DaaS side: daas-uber stores it in `algo_discovery.marketplace`. `marketplace_order_id` is **dropped**: there is no reliable source for the Uber Eats id on the DaaS side, and tracker-service correlates by the Algo order id. |
| D-6 | *changed* **All marketplace callbacks are forwarded, in a provider-agnostic shape.** For a marketplace delivery, daas-uber sends a marketplace callback for every Uber Direct webhook after the existing Algo callback (`/callbacks/…`, unchanged, still first). The body says what it carries through two optional groups, `orderStatusUpdate` and `locationUpdate` (§6.3); there are no type flags and no Uber-specific `event` field. |
| D-7 | **New gateway route `POST /marketplace/{provider}/callbacks`** (`/marketplace/uber/callbacks`), outside the `/mp/*` reverse proxy. |
| D-8 | **New RabbitMQ queue `marketplace_callbacks`**, between daas-gateway (publisher) and tracker-service (consumer). Durable, persistent, default exchange, DLQ `marketplace_callbacks_DLQ`, messages expire after 15 minutes. tracker-service declares both queues with the gateway's arguments. |
| D-9 | *changed* **tracker-service reuses its Order and Location event flows.** An `orderStatusUpdate` becomes an Order event (fanned out at once, stored when tracker knows the order); a `locationUpdate` becomes a Location event (`LocationsService.Add`: forwarding lists, plus the order's locations list when tracker knows the order). There is **no** per-order `mpcallbacks` list. `REDIS_MAX_LOCATIONS_PER_ORDER=360` is a deployment value (DAAS-639); the code default stays 100. |
| D-10 | *changed* **The market is resolved by the gateway.** daas-uber sends `brand`, `franchisee` and `country`; the gateway adds `market` to the queue message (`resolveMarketplaceMarket`, `ph-uk` until the brand+country mapping exists, §11 O-7). tracker-service falls back to `DEFAULT_MARKET` when `market` is empty. |
| D-11 | **No auth on the new route for the pilot**, the same as `/callbacks`. Later, add a service JWT on both routes by mounting the existing `Authenticator` (`daas-gateway api/middleware/authenticator.go:14`, defined but never mounted today). |
| D-12 | *changed* **No polling. Use the existing subscription.** Order events reach mp-uber through tracker-service's existing order fan-out, **immediately**; courier points through the existing forwarding job (30 s). There is no poller, no new mp-uber job and no push-to-pull cutover. |
| D-13 | *changed* **Statuses are pushed too, already in mp-uber's language.** tracker-service maps the DaaS statuses the way Algo does (§7): PickedUp → `orderStatus 5, status "Dispatched"`, Delivered → `orderStatus 6, status "Delivered"`. mp-uber needs **no** mapping change: these become Uber Eats `started` / `delivered`. Algo still sends `EnRoute` / `Delivered` through `/order/status`; both paths map to the same Uber status and share mp-uber's 30 min dedup. |
| D-14 | **Infrastructure is DevOps, in a separate ticket:** EMEA Redis/Valkey (TLS) and the tracker-service EMEA deployment, plus RabbitMQ access for the queue. |
| D-15 | **Out of scope:** Uber Direct cancel/return handling, and the webhook signature check. |
| D-16 | **Uber Eats only receives statuses and locations:** `restaurantdelivery/status` and `byoc/…/event/location`. |
| D-17 | *new* **Storing follows tracker's existing rules.** The callback carries only the Algo order id. tracker-service stores the status (and attaches points to the order) only when it finds the order by that id (the `algoid` mapping written when tracker learnt the order, normally on the Algo new-order notification and Proxy refresh). An order it does not know is forwarded but not stored, like a Proxy order event without `posOrderId`. |

> **As implemented (v8, 2026-10-29):** D-1: Algo 2.65.0_ALGO-2532 still calls the switch. When the store's `MarketPlaceConfiguration.daas` is not `"uber-daas"` it runs `go switchOrderToMPCarrier`, which posts `/mp/%s/order/switch`, and a successful switch sets `SaleTypeCarryOut`. The 404 leads to `stayWithStoreCarrier` only after `updateMarketPlaceWithDecision` retried it: 5 retries, waits 1, 2, 4, 8, 16 s, 6 sends in all. Uber Direct is booked only when the store's `daas` value is `"uber-daas"` (algo:AggregatorsAPI.go:3335-3352, :3351, :3380-3389, :3395-3409, :3506-3567; QA store config)


> **As implemented (v8, 2026-10-29):** D-2: `NewOrderMakeline` is sent in the store-driver branch and once before the uber-daas booking, but not for every order: it is not sent when the non-daas switch branch succeeds, for orders skipped by the FlowAlerts check, or when `CalcNextAvailableCarrier` fails; after a failed booking it is not sent again (algo:AggregatorsAPI.go:3332, :3443-3460, :3351, :3303-3306, :3288-3292)


> **As implemented (v8, 2026-10-29):** D-3: `carrierThreshold` defaults to 30 min and is configurable (`GetOptionListIntValue("MarketPlace","carrierThreshold",30)`). Algo books daas-uber only when the store's `MarketPlaceConfiguration.daas` is `"uber-daas"`; otherwise it calls `/order/switch` (404 while disabled) and keeps the store driver (algo:AggregatorsAPI.go:3278, :3296, :3338-3343, :3395-3409; QA store config)


> **As implemented (v8, 2026-10-29):** D-6: "every Uber Direct webhook" is too broad. No marketplace callback is sent when the order is not a marketplace order, when the Algo callback failed (daas-uber returns first), or when the webhook carries no usable update (daas-uber internal/server/providers.go:137-145, internal/server/marketplace.go:134-138)


> **As implemented (v8, 2026-10-29):** D-8: tracker-service declares the DLQ with no arguments, and the gateway never declares the DLQ (it declares only the queue of the message it publishes); only the main queue's arguments (`x-dead-letter-exchange: ""`, `x-dead-letter-routing-key`) are the same on both sides. The 15 min expiry is a per-message `Expiration` of `"900000"` ms, not a queue TTL (tracker-service consumers/marketplaceCallbacks.go:99-104, :131; daas-gateway pkg/queue/client.go:168-177, :265-276)


> **As implemented (v8, 2026-10-29):** D-9: the QA tracker-service runs with `REDIS_MAX_LOCATIONS_PER_ORDER=100`, the code default, so the 360 is not in effect there (tracker-service configuration.yaml:63, dataAccess/repositories/ordersRepository.go:419; QA ConfigMap, §23)


> **As implemented (v8, 2026-10-29):** D-10: the gateway always sets `market`, so tracker-service's fallback to `DEFAULT_MARKET` is not reached for gateway traffic. With `DEFAULT_MARKET` empty a market-less callback is invalid and goes to the DLQ (tracker-service handlers/marketplaceCallbackHandler.go:79-86; daas-gateway api/handler/marketplaceCallback.go:55-59, :128)


> **As implemented (v8, 2026-10-29):** D-12: forwarded points have a window. The forwarding job serves only subscriptions with `createdAt + maxAllowedTimeMins >= now`, default 2 h (`DEFAULTSUBSCRIPTIONMAXALLOWEDTIME`), so points stop reaching mp-uber 2 h after the subscription was created; the order fan-out has no such window (tracker-service jobs/fwdSubscriptionEventsJob.go:79-80, types/eventSubscriptions.go:52-63, configuration.yaml:30, services/subscriptionService.go:114-128)


> **As implemented (v8, 2026-10-29):** D-16: the absolute wording is too broad. In the order flow mp-uber sends only those two calls, but the service still contains the flag-gated `delivery/order/{aggOrderId}/switch-fulfillment-type` call and the store-sync calls `GET delivery/stores` and `GET delivery/store/{uid}` (mp-uber types/consts.go:15-19, services/uberMpEventsService.go:183-206, services/storesService.go:183, :265)


## 5. Flow

```
Algo ──NewOrderMakeline──▶ gateway /mp/uber/order/status ──▶ mp-uber ─(row 'received' + subscribe)─▶ tracker-service
Algo ──create delivery (aggProvider=uber, marketplace=uber)──▶ gateway /api/v2/deliveries ──▶ daas-uber ──▶ Uber Direct
Uber Direct ──webhooks (delivery_status, courier_update ~20 s)──▶ daas-uber
   daas-uber ──/callbacks/… (unchanged, first)──▶ gateway ──▶ store callback queue ──▶ Algo
   daas-uber ──/marketplace/uber/callbacks (orderStatusUpdate | locationUpdate)──▶ gateway ──publish (+market)──▶ RabbitMQ marketplace_callbacks ──consume──▶ tracker-service
tracker-service ──order event, at once (order fan-out)──▶ gateway /mp/uber/order/events ──▶ mp-uber ──▶ Uber Eats status
tracker-service forwarding job (30 s) ──points──▶ gateway /mp/uber/order/events ──▶ mp-uber ──▶ Uber Eats BYOC location
Algo ──EnRoute / Delivered (as today)──▶ gateway /mp/uber/order/status ──▶ mp-uber ──▶ Uber Eats (deduplicated)
Proxy (store-driver app) ──/events/location (as today)──▶ tracker-service (same lists)
```

### Steps

1. **Algo registers the order** (Algo side is ALGO-2532, out of scope). Algo picks the carrier (D-3) and sends `NewOrderMakeline` in both branches (D-2). DaaS side: mp-uber inserts its `marketplaceorderstatus` row as `received` and creates the tracker subscription (`SubscribeToLocationsOnStatus = NewOrderMakeline`, `uberMpEventsService.go:41-43, :97-114`). Later, Algo sends `EnRoute` and `Delivered` as today, turned into Uber Eats statuses through `TriggerUpdatesOrderStatuses` (default `EnRoute,Delivered`).

> **As implemented (v8, 2026-10-29):** the subscription refs are `uberMpEventsService.go:43-44` (trigger list and the `SubscribeToLocationsOnStatus` flag; `:41-42` is the `ParseOrderStatus` normalization) and `:102-120` (the subscribe block). The YAML default of the trigger list is `EnRoute,Delivered` (`MARKETPLACE_TRIGGERUPDATEON`, `configuration.yaml:27`); the code fallback `NewOrderMakeline,Delivered` applies only when the value is empty. Algo 2.65.0 has no sender of `EnRoute` / `Delivered` to `/mp/uber/order/status`: the only builder of that call posts `NewOrderMakeline` (mp-uber services/uberMpEventsService.go:41-44, :102-120, configuration/startup.go:159-161; algo AggregatorsAPI.go:3418-3438)

2. **Algo books daas-uber only** when the store driver is too far (ALGO-2532, out of scope). See D-4 and D-5. If the booking fails, Algo falls back to the store driver. mp-uber is not affected, because nothing was promised to Uber Eats.
3. **daas-uber marks the delivery** in `algo_discovery.marketplace` (D-5).
4. **daas-uber forwards callbacks.** For each Uber Direct webhook of a marketplace delivery: first the existing Algo callback; only after it succeeded, `POST {CallbackURL}/marketplace/uber/callbacks` in a goroutine (D-6). `courier_update` webhooks fill `locationUpdate`, every other webhook `orderStatusUpdate`; a callback with neither (a courier update without coordinates) is not sent. 3 attempts, 3 s timeout each, 1 s between attempts, no retry on a 4xx, failures logged. It never blocks the Algo callback.

> **As implemented (v8, 2026-10-29):** the retry rule is `status < 400 || status >= 500`: a 4xx is never retried, but a 1xx, a 3xx and any 2xx other than 202 (for example 200 or 204) are. The path of `CallbackURL` is replaced, not appended: the sender sets the path to `marketplace/{provider}/callbacks` and keeps scheme and host (daas-uber internal/server/marketplace.go:168-175, :193-196, internal/server/gatewayAPI.go:35-37)

5. **The gateway validates and publishes** the body plus `provider`, `market` and `requestId` to `marketplace_callbacks`, and replies `202` (D-7, D-8, D-10).
6. **tracker-service consumes** (D-9, D-17). A fixed pool of workers, each order always on the same worker, so one order's updates are handled in sequence. Each update goes through the existing flow:
   - `orderStatusUpdate` → Order event → order fan-out to the subscription **at once** → stored with the unchanged `ProcessOrderEvent` when the order is found by its Algo id.
   - `locationUpdate` → Location event (`carrierId` = `deliveryId`, `orders` = Algo order id, stamped with the time tracker received it) → `LocationsService.Add`: forwarding lists `tr-svc:fwd:…`, and the order's locations list when the order is known.
7. **mp-uber receives** on `/mp/uber/order/events` (`ProcessOrderEventsBatch`, `uberMpEventsService.go:119-175`): order events first (`Dispatched` → Uber `started`, row `started`; `Delivered` → Uber `delivered`, then `205`, which deletes the tracker subscription). Location batches from the forwarding job are sent to Uber once the row is `started` (`:162-174`), newest point only. `externalStoreId` comes from the subscription key, as today.

> **As implemented (v8, 2026-10-29):** `ProcessOrderEventsBatch` is `uberMpEventsService.go:124-181`; the `started` gate is `:167` with the location goroutine at `:169-175` (the status read is `:160`). tracker-service deletes the subscription when it receives the 205; mp-uber's `Unsubscribe` is defined but never called (tracker-service services/subscriptionService.go:180-181; mp-uber services/trackerV2Service.go:88-107)


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
- `NewOrderMakeline`: row `received` + tracker subscription.
- `EnRoute`: Uber Eats `started`. `Delivered`: Uber Eats `delivered`. Both are deduplicated for 30 min.
- `orderId` is the Algo order id (`algo AggregatorsAPI.go:2790`), the same id as `targets[].details.id` in §6.2 (`:661`).

> **As implemented (v8, 2026-10-29):** `orderId` is `order.OrderId.ToString()` at algo `AggregatorsAPI.go:3423`, and `targets[].details.id` is `fmt.Sprint(order.OrderId)` at `:767`; `:2790` and `:661` are unrelated code (`busyAggCarriers`, `getAggClientCredentialsHeaders`). The two ids are the same value. On this ref Algo sends only `NewOrderMakeline` on this call; there is no `EnRoute` or `Delivered` sender, although mp-uber accepts both (algo AggregatorsAPI.go:3418-3438, :767, constValues.go:222; mp-uber types/consts.go:103-110)


`POST {gateway}/mp/uber/order/switch` (existing): `404 {"error": "switch to Uber delivery is disabled"}` while `switchToUberDeliveryEnabled` is `false` (D-1).

> **As implemented (v8, 2026-10-29):** Algo 2.65.0 still posts this route in the non-`uber-daas` branch (see the D-1 note). While the flag is off mp-uber answers `404` with `{"error": ...}` and omits `externalStatusCode` (mp-uber controllers/marketplaceOrderEventsController.go:68-73, types/types.go:107-120; algo AggregatorsAPI.go:3351, :3380-3382)


### 6.2 Algo → daas-uber create delivery (excerpt, new fields marked; Algo side is ALGO-2532, the DaaS side consumes it)

```http
POST {gateway}/api/v2/deliveries
{
  "aggProvider": "uber",            // NEW: forced for marketplace orders (D-4)
  "storeId": "7507",
  "brand": "Pizza Hut:GH Pizza LTD-United Kingdom",
  "targets": [{ "details": { "id": "26101402978", "posOrderId": "...", "altOrderId": "d3b93b60-c336-4b32-87e6-bed8eb0c2cab" } }],
  "marketplace": "uber"             // NEW (D-5)
}
```
The gateway forwards it to daas-uber (`/api/v1/deliveries` or `/api/v2/deliveries`, both accept `marketplace`). `details.id` is the Algo order id and becomes `algo_discovery.order_id`.

> **As implemented (v8, 2026-10-29):** daas-uber silently drops these create-delivery fields: at top level `aggProvider` (read by the gateway only), `providerCouriersCount`, `isCash`, `isAlcoholic`, `isNotPaid`, `carrierId`, `rating`, `reason`, `comment`; in `targets[].details` `posOrderId`, `altOrderId`, `contactless`, `dropOffDeadlineAt`; per item `modifier`; per customer `email`; per address `floor`. `marketplace` is lower-cased. The full request is in §17.3 (daas-uber internal/algo/deliveries.go:6-108, internal/server/parsers/v2.go:49-57)


### 6.3 daas-uber → gateway → queue (new)

```http
POST {gateway}/marketplace/uber/callbacks      → 202 Accepted
{
  "marketplace": "uber",
  "brand": "Pizza Hut",
  "franchisee": "GH Pizza LTD",
  "country": "United Kingdom",
  "storeNo": "7507",
  "orderId": "26101402978",                     // Algo order id
  "deliveryId": "del_UowjW4AJTSGDNwqOOFCwCg",
  "timestamp": "2026-10-22T12:31:07Z",          // Uber's data.updated, else now
  "orderStatusUpdate": {                        // optional: status webhooks
    "status": "dropoff",                        //   raw provider status, logs only
    "targetStatus": 3,                          //   Algo DaaS delivery status
    "courierStatus": 5                          //   Algo DaaS courier status
  },
  "locationUpdate": { "lat": 51.913002, "lng": -0.20857728 }   // optional: courier_update webhooks
}
```

- **Shape = meaning.** Each group is omitted when the provider callback did not carry that update; a callback may carry both (other providers). daas-uber (`internal/server/marketplace.go`): `event.courier_update` → `locationUpdate` only (top-level webhook location, else `data.courier.location`); every other kind, an unknown one included → `orderStatusUpdate` only. Status callbacks carry no position.
- **Gateway validation** (`api/handler/marketplaceCallback.go`): `orderId`, `storeNo`, `brand`, `country` required, plus `orderStatusUpdate` or `locationUpdate`; a 0,0 `locationUpdate` is dropped (a status sent with it is still published), a callback left with neither → 400.
- **Queue message** = this body plus `"provider": "uber"`, `"market": "ph-uk"` and `"requestId"`.

> **As implemented (v8, 2026-10-29):** the gateway answers `202` with no body, and the publish is an in-memory channel send: `202` means enqueued in the gateway process, and the publisher worker retries each message 5 times and then only logs. The validation order and the `400` envelope `{"status":"error","message":"..."}` are in §17.4. `marketplace`, `deliveryId`, `timestamp` and `franchisee` are not checked. daas-uber sends `lat` / `lng` as `float32`; tracker-service reads them as `float64` (daas-gateway api/handler/marketplaceCallback.go:63-163, pkg/queue/client.go:128-158, :253-283; daas-uber internal/server/marketplace.go:52-56; tracker-service types/marketplaceCallbacks.go:9-36)


### 6.4 Redis (tracker-service, existing keys only)

| Key | Written by | TTL / limit |
|---|---|---|
| `tr-svc:fwd:{m}:{s}:{algoOrderId}` and `tr-svc:fwd:{m}:{s}` (forwarding source) | every `locationUpdate` | `sourceTtl` 15 min, `maxSourceEntries` 3000 |
| `tr-svc:market:{m}:store:{s}:posorder:{posOrderId}:locations` (shared with store-driver points) | `locationUpdate` of an order tracker knows | `orderLocations` 24 h, refreshed on write; `REDIS_MAX_LOCATIONS_PER_ORDER` (360 in the deployment) |
| `tr-svc:market:{m}:store:{s}:posorder:{posOrderId}` (order entity) | `orderStatusUpdate` of an order tracker knows | `REDIS_DURATION_ORDER` (72 h default) |

> **As implemented (v8, 2026-10-29):** the callback path writes more keys than this table lists: `tr-svc:market:{m}:store:{s}:algoid:{algoId}` (raw posOrderId, 72 h), `tr-svc:market:{m}:store:{s}:carrier:{deliveryId}:currentorders` (set, 720 h) and, through `GetAll`, `tr-svc:subscriptions:index`. The full table is §19. In QA the locations cap is 100, not 360 (tracker-service dataAccess/repositories/ordersRepository.go:221-228, :350-352; services/ordersService.go:171-174, :336; dataAccess/repositories/carrierRepository.go:100-101; dataAccess/repositories/subscriptionsRepository.go:120, :158)


### 6.5 tracker-service → mp-uber (existing subscription push)

`POST {gateway}/mp/uber/order/events` (mp-uber role `tracker`), header `ExtKey`, the existing `EventsBatchDto` format.

```json
// order fan-out, one event per batch, sent when the callback is consumed
{ "key": "…", "events": { "order": [
  { "orderId": 26101402978, "carrierId": "del_UowjW4AJTSGDNwqOOFCwCg",
    "carrierStatus": "EnrouteToDropoff", "orderStatus": 5, "status": "Dispatched" } ] } }

// forwarding job, every 30 s
{ "key": "…", "events": { "location": [ { "lat": 51.913002, "lng": -0.20857728, "timestamp": 1791975067000 } ] } }
```
`carrierStatus` is new and optional (tracker's name of the DaaS courier status, forwarded as is); mp-uber ignores it today. mp-uber's controller rejects location points with lat/lng = 0 or older than 2 h (existing). A `205` answer deletes the subscription (mp-uber answers it on a final status).

> **As implemented (v8, 2026-10-29):** the order-event example is abbreviated. `OrderEventData` also serializes `deliveryETA`, `exitGeofence`, `enterGeofence` and `tmpExitGeofence` as `"0001-01-01T00:00:00Z"` (a `time.Time` with `omitempty` is never omitted), and every batch carries `sender`, `totalEvents`, `timeStamp` and `traceId`. mp-uber rejects the whole batch with 400 if any point has `Lat == 0` or `Lng == 0` (either one) or is older than 2 h, while tracker drops a point only when both are 0; tracker does not retry a 4xx and its cursor has already advanced (tracker-service types/events.go:130-145, types/dtos.go:120-128, services/locationsService.go:39-41, helpers/helpers.go:156-160, jobs/fwdSubscriptionEventsJob.go:206-222; mp-uber controllers/marketplaceOrderEventsController.go:101-107)


### 6.6 mp-uber → Uber Eats (existing)

- `POST /eats/orders/{aggOrderId}/restaurantdelivery/status` `{ "status": "started" | "delivered" }`
- `POST /eats/byoc/restaurants/orders/event/location` `{ "location_request": { "order_workflow_uuid": aggOrderId, "restaurant_uuid": externalStoreId, "location_events": [...] } }`

> **As implemented (v8, 2026-10-29):** the location call body also carries `is_batched_order` (always `false`), and `location_events` always holds exactly one element, the newest point (mp-uber types/mpDtos.go:7-34, services/uberMpEventsService.go:216-221, :237-238, :259-268)


## 7. Status mapping

Uber Direct → Algo DaaS statuses as in `daas-uber providers/uber/delivery.go:414-439`. DaaS → order event as in Algo's own handling of DaaS callbacks (`algo AggregatorsEvents.go parseCallbacks`): only a pickup (marketplace orders go enroute on pickup) and a delivery change the order; a cancelled delivery is not a cancelled order. Implemented in `tracker-service types/marketplaceCallbacks.go`.

| Uber Direct | DaaS target / courier | Order event to mp-uber (`status`, `orderStatus`) | Stored order status | Uber Eats |
|---|---|---|---|---|
| pending | 1 Scheduled / 0 Unassigned | `Scheduled`, current order status | unchanged | – |
| pickup | 2 Assigned / 2 EnrouteToPickup, 3 ArrivedAtStore | `Assigned`, current order status | unchanged | – |
| pickup_complete | 2 Assigned / 3 ArrivedAtStore | `Assigned`, current order status | unchanged | – |
| dropoff | 3 PickedUp / 5 EnrouteToDropoff, 8 Nearby | `Dispatched`, 5 | Enroute | `started` |
| delivered | 4 Delivered / 7 DroppedOff | `Delivered`, 6 | Delivered | `delivered` (then 205) |
| canceled | 5 Cancelled / 0 | `DeliveryCancelled`, current order status | unchanged | – (out of scope, O-4) |
| returned | 6 CouldNotDeliver / 0 | `CouldNotDeliver`, current order status | unchanged | – (out of scope, O-4) |
| (unmapped Uber status) | 0 Waiting / 0 | `Waiting`, current order status | unchanged | – |

> **As implemented (v8, 2026-10-29):** the `–` cells in the Uber Eats column are not inert. When tracker knows the order, an event of a non-trigger status carries the stored order status, and mp-uber maps it through `AllOrderStatuses`; if the stored status is `Enroute` (5) or `Delivered` (6), the event is in the trigger list and posts `started` or `delivered` again, guarded only by the 30 min dedup. This applies to `pending`, `pickup`, `pickup_complete`, `canceled`, `returned` and the unmapped row; after `canceled` / `returned` the stored status stays `Enroute`. "`carrierStatus` ... is not stored" is incomplete: the courier status is not stored, but tracker stores a carrier status derived from the order status (Enroute → `EnrouteToPickup`, Delivered → `DroppedOff`, other final statuses → `Unassigned`), so after `dropoff` it holds `EnrouteToPickup` while the courier is `EnrouteToDropoff`. The `pickup` and `dropoff` courier values split on `courier_imminent` (tracker-service converters/dbEntityConverter.go:141-143, :150-166, :390-406, types/marketplaceCallbacks.go:52-54; mp-uber services/uberMpEventsService.go:285-287, :341-347; daas-uber internal/providers/uber/delivery.go:418-427, internal/providers/uber/types.go:221)


Notes:
- "Current order status" is the stored order's status when tracker knows the order; otherwise `orderStatus` is left out, which mp-uber reads as `NewOrderMakeline` and ignores. These events are no-ops for mp-uber today and become useful once subscriptions can filter by status.
- `carrierStatus` carries the courier status as is (`Accepted`, `EnrouteToPickup`, …); it is not stored.
- A DaaS courier login (7 CourierLogin) produces no order event.
- `courier_update` webhooks (about every 20 s) only carry a point; they never produce an order event.
- Uber Eats gets `started` from whichever arrives first, the pushed `Dispatched` or Algo's `EnRoute` (same Uber status, same dedup key). O-1 is answered: `started` is the pickup (`dropoff`), the same moment as Algo's EnRoute.
- mp-uber sends locations only after its row is `started` (`uberMpEventsService.go:162-174`). Earlier points reach tracker-service but are not sent to Uber.

> **As implemented (v8, 2026-10-29):** the "about every 20 s" `courier_update` interval cannot be verified from code, and `CourierLogin` is never produced from an Uber status, so that path is unreachable. Algo's own auto-EnRoute on pickup applies only when auto-dispatch is on (`AutoEnrouteOnAggPickup` or quote id `mp`) and the order status is already 4 or more. `started` needs `EnRoute` in `triggerUpdateOn` (daas-uber internal/providers/uber/delivery.go:414-440; tracker-service services/marketplaceCallbacksService.go:52-55; algo AggregatorsEvents.go:574-605; mp-uber configuration.yaml:27)


## 8. Changes per repo

Each item is one plan task. "Done when" is the acceptance check.

### 8.0 Algo: out of scope, what the DaaS side relies on (ALGO-2532)

No Algo tasks are planned from this spec. The DaaS side assumes Algo provides the following. Track it as a dependency on ALGO-2532.

| # | Algo provides | DaaS side depends on it in |
|---|---|---|
| A-1 | `NewOrderMakeline` to `/mp/uber/order/status` for every marketplace order, store driver or DaaS. No `/order/switch` call; if one is still sent, mp-uber answers 404 and Algo's `stayWithStoreCarrier` sends `NewOrderMakeline`. | mp-uber row and subscription (§8.4). |
| A-2 | Create delivery to `/api/v2/deliveries` with `aggProvider: "uber"` and `marketplace: "uber"` (§6.2), with the **same Algo order id** in `targets[].details.id` as the `orderId` sent to mp-uber (true today: `AggregatorsAPI.go:661`, `:2790`). | gateway aggProvider routing (§8.2), daas-uber marketplace flag (§8.1), tracker ↔ mp-uber subscription match (§8.3). |
| A-3 | `EnRoute` and `Delivered` to `/mp/uber/order/status` for DaaS-delivered orders (O-2). | mp-uber dedup with the pushed statuses (§8.4). |
| A-4 | The order stays a delivery (no carry-out), and the Uber Eats `aggOrderId` is kept on the order. | Uber Eats status and location calls. |
| A-5 | Algo still consumes daas-uber's existing `/callbacks/…` unchanged. | daas-uber keeps sending them first (§8.1). |
| A-6 | POS notifications (new order) to tracker-service, so tracker knows the order and its Algo id before the courier is assigned. | Storing statuses and order locations (D-17). |

> **As implemented (v8, 2026-10-29):** A-1: Algo 2.65.0 still calls `/order/switch` and does not send `NewOrderMakeline` for every order (see the D-1 and D-2 notes) (algo AggregatorsAPI.go:3351, :3380-3382, :3303-3306, :3288-3292)


> **As implemented (v8, 2026-10-29):** A-2: the order id sits at `AggregatorsAPI.go:767` (create delivery `details.id`) and `:3423` (mp-uber `orderId`), not at `:661` and `:2790`; the equality of the two is true (algo AggregatorsAPI.go:767, :3423)


> **As implemented (v8, 2026-10-29):** A-3: no Algo code on this ref sends `EnRoute` / `Delivered` to `/mp/uber/order/status`. Whether another service does is not visible in the four repos, so O-2 stays open (algo AggregatorsAPI.go:3418-3438, constValues.go:222)


### 8.1 daas-uber, DAAS-635 — done

> **As implemented (v8, 2026-10-29):** Jira DAAS-635 is Peer Review (Jira GET 2026-10-29); the code runs in QA as daas-uber 11.5.7 (§23)


1. **Accept `marketplace`** in `parsers.Order` (V1) and `algo.Delivery` (V2), mapped in both `ToAlgoDelivery`.
2. **Migration `migrations/19.add_marketplace_to_algo_discovery.sql`:** `marketplace text NULL` only.
3. **Carry the column** through `OrderRequest`, `SaveOnce` (insert + `COALESCE` on conflict), `GetOrderByAggID` and the `getAlgoDeliveryByAggID` DB fallback. *Done when:* the flag survives a cache miss and a restart.

> **As implemented (v8, 2026-10-29):** "survives a cache miss and a restart" has no test and was not verified. `GetOpenOrders` does not select `marketplace`, and the in-memory cache key (`ProviderID-ExternalID`) differs from the lookup key (`aggID`), so the DB fallback is the path that carries the flag (daas-uber internal/algo/repository.go:131-137, internal/server/deliveries.go:210, internal/server/providers.go:171, :190)

4. **`internal/server/marketplace.go`:** `newMarketplaceCallback` builds the body of §6.3; `SendMarketplaceCallback` posts it to `{CallbackURL}/marketplace/{marketplace}/callbacks` (3 attempts, 3 s timeout, 1 s apart, no retry on 4xx). Called with `go` from `Uber()` (`providers.go:143-146`) after the Algo callback succeeded, when `orderDB.IsMarketplace()`. `Uber()` and the Algo callback are otherwise unchanged.

> **As implemented (v8, 2026-10-29):** the URL is built by replacing the path of `CallbackURL`, not by appending to it; `Uber()` is `internal/server/providers.go:52` and the call is at `:143-146`. With the QA `AGG_CALLBACKURL` `https://gateway-qa.daas.dtsys.cc/callbacks` the POST goes to `https://gateway-qa.daas.dtsys.cc/marketplace/uber/callbacks` (daas-uber internal/server/marketplace.go:168-175, internal/server/providers.go:143-146; QA config)

5. **Courier location for the marketplace body only** (`courierLocation`); `Delivery.Parse` is unchanged, so Algo callbacks do not start carrying a location.

### 8.2 daas-gateway, DAAS-636 — done

> **As implemented (v8, 2026-10-29):** Jira DAAS-636 is Peer Review (Jira GET 2026-10-29); the code runs in QA as daas-gateway 1.6.0 (§23)


1. **`api/handler/marketplaceCallback.go`:** `HandleMarketplaceCallback` validates (§6.3), adds `provider`, `market` (`resolveMarketplaceMarket`) and `requestId`, `queue.SendMessage` to `MarketplaceCallbacksQueue` with 15 min expiry, replies 202.
2. **`POST /marketplace/{provider}/callbacks`** registered in `router.go` next to the `/mp/{provider}` group, no auth (D-11).
3. **Config:** `MarketplaceCallbacksQueue` (default `marketplace_callbacks`) in `config/queue.go`, env `DT_QUEUE_MARKETPLACECALLBACKSQUEUE`.
4. **`/mp/{provider}/order/switch` stays** in `MakeMpProxyHandler` (D-1, mp-uber answers 404).
5. **`aggProvider`:** already on `master`, no change (D-4).
6. **Handler tests** with `httptest` and a buffered `messages` channel: status only, location only, both, 0,0 location, validation.

### 8.3 tracker-service, DAAS-637 — done

> **As implemented (v8, 2026-10-29):** Jira DAAS-637 is Peer Review (Jira GET 2026-10-29); the code runs in QA as tracker-service 2.0.4 (§23)


1. **`consumers/marketplaceCallbacks.go`:** `amqp091-go` consumer, started by `subscribeToRabbitMQ` in `configuration/startup.go` behind `RABBITMQ_ENABLED`. Declares the queue with the gateway's arguments and its DLQ, prefetch 50, manual ack, a fixed pool of workers (8) keyed by order, reconnect with backoff (5 s → 1 min). Invalid messages are rejected without requeue (to the DLQ); failures while processing are logged and acked.

> **As implemented (v8, 2026-10-29):** the consumer declares the DLQ with no arguments, and the gateway does not declare it (see the D-8 note) (tracker-service consumers/marketplaceCallbacks.go:131; daas-gateway pkg/queue/client.go:168-177)

2. **`handlers/marketplaceCallbackHandler.go`:** decodes and validates (bad JSON, invalid market, non-numeric Algo order id, unknown store, no update → invalid), builds the scope (message market, else `DEFAULT_MARKET`; numeric store, else the alphanumeric store mapping).
3. **`services/marketplaceCallbacksService.go`:** turns the callback into an Order event and/or a Location event and calls the existing `SubscriptionsService.ProcessOrderSubscriptions`, `OrdersService.ProcessOrderEvent` and `LocationsService.Add` (§5 step 6). Mapping in `types/marketplaceCallbacks.go`, event built by `EntityConverter.ToOrderEventFromMarketplaceCallback`; `OrderEventData` gets the optional `carrierStatus`.
4. **Config:** `rabbitmq` block in `configuration.yaml` with split settings `RABBITMQ_ENABLED` (false) `_SCHEME` `_HOST` (host:port) `_USERNAME` `_PASSWORD` `_VHOST` `_MARKETPLACE_CALLBACKS_QUEUE` `_PREFETCH` `_WORKERS` (no URL, like the gateway's `DT_QUEUE_*`).
5. **Dockerfile:** `golang:1.25`, aligned with `go.mod`.
6. **Tests** (testify, mocks): status, location, both, bad JSON, unknown market, unknown order, worker routing, ack/DLQ, config defaults.

### 8.4 mp-uber, DAAS-638 — done

> **As implemented (v8, 2026-10-29):** Jira DAAS-638 is Dev Test, not Done (Jira GET 2026-10-29); the code runs in QA as mp-uber 0.1.9_2 (§23)


1. **Switch disabled by configuration** (instead of deleted): `switchToUberDeliveryEnabled` on `MarketPlaceConfiguration`, `MARKETPLACE_SWITCHTOUBERDELIVERY_ENABLED`, default `false`; `SwitchToUberDelivery` in `controllers/marketplaceOrderEventsController.go` answers 404 with `ErrorResponse` while it is off. The service, DTOs and `DELIVERY_BY_UBER` stay.
2. **Keep** the `NewOrderMakeline` row insert and tracker subscription as they are. No new endpoint, job or migration.
3. **No status mapping change:** `processOrderEvents` already maps `Dispatched` → Enroute → `started` and `Delivered` → `delivered` (D-13).

> **As implemented (v8, 2026-10-29):** 0.1.9_2_dev adds a `ParseOrderStatus(request.Status)` normalization in `UpdateOrderStatus`, so the status text is accepted case-insensitively before the mapping; the mapping table itself is unchanged (mp-uber services/uberMpEventsService.go:38-42, types/consts.go:103-110)

4. **Dedup checked:** Algo's `EnRoute` and tracker's `Dispatched` both become `started` and share the key `marketplaceorderstatusupdate::aggorderid::{id}::status::started` for 30 min (same for `delivered`). Limits: in-memory per instance, and two simultaneous requests can both pass the check.

> **As implemented (v8, 2026-10-29):** the dedup key is lower-cased as a whole, set before the Uber call and removed when the Uber call fails or returns non-2xx. The in-memory claim rests on `common/cache`, an external module whose pinned version was not available locally (an older copy is an in-process map); the check and the set are two separate steps (mp-uber services/uberMpEventsService.go:341-357, types/consts.go:36, configuration/startup.go:90)

5. Tests postponed (no test setup for mp-uber's DI yet).

### 8.5 Infra, DAAS-639 (DevOps, Ehab Jaber)

> **As implemented (v8, 2026-10-29):** Jira DAAS-639 is To Do and its assignee is Alexander Hodak, not Ehab Jaber, yet tracker-service 2.0.4 already runs in QA (§23) (Jira GET 2026-10-29)


EMEA Redis/Valkey (TLS), RabbitMQ access for `marketplace_callbacks` and its DLQ, and the tracker-service ECS deployment in EMEA. tracker-service settings: `RABBITMQ_ENABLED=true` plus the `RABBITMQ_*` connection settings, `REDIS_MAX_LOCATIONS_PER_ORDER=360`, `DEFAULT_MARKET=ph-uk`. It blocks DAAS-637 going live.

## 9. Failure handling

| What fails | Effect | Handling |
|---|---|---|
| `NewOrderMakeline` (1) | No row and no subscription, so no status or location reaches Uber Eats through the push | 6 attempts (backoff 1–16 s). mp-uber also inserts the row and subscribes if the first call it sees is EnRoute. |
| daas-uber booking (2) | No courier | The store driver delivers. Uber Eats sees no difference. |
| Marketplace callback POST (4) | Callback missing from tracker. Algo still has it. | 3 attempts, 1 s apart, no retry on 4xx, then logged. A lost location is replaced by the next `courier_update` (~20 s). A lost status is not, but Algo's status still reaches mp-uber. |
| Queue or tracker down (5, 6) | Items wait | Durable queue, 15 min expiry, DLQ. The gateway buffers 2000 messages in memory (`cmd/daas-gateway/server.go:98`); tracker reconnects with backoff. |
| Invalid message (6) | Not processed | Rejected without requeue → `marketplace_callbacks_DLQ`. |
| Processing failure in tracker (6), e.g. Redis error | That update is lost | Logged and acked, not retried (pilot decision). The order fan-out still runs when only the order lookup failed. |
| tracker does not know the order (6) | Status not stored, points not attached to the order | Still forwarded to mp-uber (order event at once, points through the forwarding lists). D-17. |
| Forwarding push or Uber Eats (7) | Position lags | The forwarding job's existing behaviour (source TTL 15 min, at-most-once cursor). mp-uber sends only the newest point. |

> **As implemented (v8, 2026-10-29):** the `NewOrderMakeline` row is confirmed: 5 retries of any non-2xx answer (waits 1, 2, 4, 8, 16 s), 6 sends in all (algo AggregatorsAPI.go:3506-3567)


> **As implemented (v8, 2026-10-29):** mp-uber answers a batch whose non-final order event failed at Uber with `200` (logged, not retried) and a final one with `500`, so tracker retries only the second; a bad location rejects the whole batch with `400`, which tracker does not retry. A re-subscribe after mp-uber's 10 min subscription cache expires meets tracker's duplicate check (`409`) and would fail `NewOrderMakeline` with 500 (derived from code, not tested) (mp-uber services/uberMpEventsService.go:109-113, :140-154, :180, controllers/marketplaceOrderEventsController.go:101-107; tracker-service controllers/subscriptionsController.go:93-97)


## 10. Rollout and tests

**Release order**
1. DevOps (separate ticket): EMEA Redis/Valkey and the queue, and deploy tracker-service with `RABBITMQ_ENABLED=true`.
2. daas-gateway with the new callbacks route.
3. daas-uber with the migration. It only acts when `marketplace` is set.
4. mp-uber with the switch flag (default off).
5. (Out of scope, ALGO-2532) Algo release with the switch replaced by `NewOrderMakeline`, piloted on one store. The DaaS side must be live before this.

Nothing has to be removed afterwards: the switch stays disabled by configuration.

**End-to-end checks (staging)**
- A store-driver order behaves exactly as today.
- A DaaS order: the `algo_discovery` row has `marketplace='uber'`.
- A DaaS order: `NewOrderMakeline` creates the tracker subscription; status callbacks reach `/mp/uber/order/events` at once, points within the forwarding interval.
- Each webhook, including `delivered`, gives one Algo callback and one marketplace callback (except a `courier_update` without coordinates), both 202.
- The order's locations key grows, its `TTL` is about 86400, and `LLEN` ≤ 360.
- Uber Eats gets `started`, then locations, then `delivered`, each status once.
- `POST /mp/uber/order/switch` answers 404.
- A normal (non-marketplace) daas-uber delivery sends no marketplace callback.

> **As implemented (v8, 2026-10-29):** in QA the cap is 100 (`REDIS_MAX_LOCATIONS_PER_ORDER=100`), so `LLEN` ≤ 360 holds but the intended 360-point window is not in effect until the value is changed (tracker-service configuration.yaml:63, dataAccess/repositories/ordersRepository.go:419; QA ConfigMap, §23)


## 11. Open items

| ID | Item | Owner | Status |
|---|---|---|---|
| O-1 | Which Uber Direct status means `started` for Uber Eats? | Sergey Shelekhov | **Resolved:** the pickup (`dropoff`, DaaS PickedUp), the same moment as Algo's EnRoute (D-13, §7). |
| O-2 | Does Algo send `EnRoute` / `Delivered` to `/mp/uber/order/status` when a DaaS carrier delivers? If not, statuses rely only on the push. | Algo team (ALGO-2532) | Dependency A-3 |
| O-3 | Confirm Uber accepts `restaurantdelivery/status` and BYOC location for DaaS-courier orders. Uber has returned 400 "Cannot update status of this order type" before (DAAS-633). | Unassigned | Verify in staging |
| O-4 | Uber Direct `canceled` / `returned` after registration: hand back to a store driver, rebook, or report a failure to Uber Eats. | – | Out of scope |
| O-5 | Uber Direct webhook signature (`Uber.AuthWebhook`, `providers/uber/auth.go:17`, never called). | – | Out of scope |
| O-6 | ~~Name of the Redis events key~~ | – | Dropped: no `mpcallbacks` list (D-9). |
| O-7 | Brand+country → market mapping in the gateway (`resolveMarketplaceMarket` returns `ph-uk`). | Sergey Shelekhov | Follow-up, needed before a second market |
| O-8 | Does UK/Ireland Uber Direct traffic run on `daas-uber` or on the older `daas-uber-uk` fork, which has none of the DAAS-635 changes? | Sergey Shelekhov / DevOps | To confirm |
| O-9 | mp-uber's market key for Pizza Hut UK must be exactly `ph-uk` (the gateway's value), or the forwarding job does not match the subscription. | Sergey Shelekhov | To confirm in staging |
| O-10 | Optional: pass Algo's `posOrderId` from create delivery through the marketplace callback, so tracker can store statuses of orders it does not know yet (D-17). | – | Not planned |

> **As implemented (v8, 2026-10-29):** O-1: the answer is confirmed in code, `dropoff` → `PickedUp` → `Dispatched` / `orderStatus 5` → `started`; the chain is in §20.5 (daas-uber internal/providers/uber/delivery.go:423-427; tracker-service types/marketplaceCallbacks.go:57-58, :77-78; mp-uber services/uberMpEventsService.go:283-284)


> **As implemented (v8, 2026-10-29):** O-2: still open and not answered by the Algo code on this ref, which has no `EnRoute` / `Delivered` sender on `/mp/uber/order/status` (algo AggregatorsAPI.go:3418-3438)


> **As implemented (v8, 2026-10-29):** O-3: cannot be verified in QA, because daas-uber calls Uber through `mock-api-qa.daas.dtsys.cc` (QA config `AGG_UBERPROVIDERURL`; daas-uber internal/server/deliveries.go:102)


> **As implemented (v8, 2026-10-29):** O-9: BLOCKER in QA. mp-uber's `keys` table has the markets `ph-il`, `phtest-uk` and `phtest-us` and no `ph-uk`, while the gateway stamps `market=ph-uk` on every marketplace callback, so mp-uber subscriptions (market `phtest-uk`) do not match the stored callbacks (mp-uber QA `keys` table; daas-gateway api/handler/marketplaceCallback.go:17, :55-59)


## 12. Blockers

| Blocker | Owner |
|---|---|
| tracker-service is not deployed anywhere: README says "Not yet deployed", and there are no ECS/Terraform definitions in the checkout (`docs/cutover-2026-09-18/callers-report.md:23`). | DevOps, DAAS-639 |
| No Redis in EMEA prod (`docs/cutover-2026-09-18/uk-tracker-cutover.html:94`). ElastiCache would be Valkey, and serverless needs `REDIS_TLS`. | DevOps, DAAS-639 |

> **As implemented (v8, 2026-10-29):** the first blocker is outdated for QA: tracker-service 2.0.4 runs on EKS `nonprod-global-cluster`, namespace `cloud`, with the consumer enabled and connected to `marketplace_callbacks` (§23). The production deployment and a production Redis are not covered by this verification (QA, 2026-10-29)


## 13. Superseded ideas (do not implement)

| Idea | Replaced by |
|---|---|
| Algo asks Uber Eats to switch fulfilment (`/order/switch` → `switch-fulfillment-type`) | D-1, D-2 |
| Deleting the switch route and code (v6 D-1) | Switch disabled by configuration, 404 (D-1) |
| daas-uber posts courier updates straight to mp-uber (`/mp/uber/order/courier`) | D-6 – D-9, D-12 |
| A new mp-uber "create delivery" endpoint (`/mp/uber/order/delivery`) and `orderId` on the switch | D-2 |
| A locations-only gateway route (`/marketplace/uber/locations`, queue `marketplace_courier_locations`) | D-6, D-7, D-8 |
| An mp-uber poller (`GET /tracker/.../locations` every 10 s, `pullLocations` and `lastLocationTs` columns) and the push-to-pull cutover | D-12 |
| tracker-service mapping brand+country to market | D-10 (gateway) |
| daas-uber resolving `market` (`AGG_DEFAULTMARKET`, brand+country map) and sending it (v6 D-10) | D-10 (gateway) |
| `marketplace_order_id` (= `altOrderId`) in `algo_discovery` (v6 D-5) | D-5 (dropped) |
| Flat callback body with `event`, `status`, `targetStatus`, `courierStatus`, `location` (v6 §6.3) | Optional groups `orderStatusUpdate` / `locationUpdate` (D-6, §6.3) |
| Type flags in the callback (`isLocationUpdate`, a `callbackType` enum) | The shape is the flag (D-6) |
| A per-order `mpcallbacks` list in tracker (v6 D-9) | Existing Order and Location flows (D-9) |
| Order events through the forwarding job (v6 D-12) | Existing order fan-out, at once (D-12) |
| mp-uber mapping Uber Direct statuses to `started` / `delivered` (v6 D-13) | tracker maps them like Algo (D-13) |
| Copying courier lat/lng in `Delivery.Parse` for every Algo callback (v6 §8.1.5) | Location taken for the marketplace body only (§8.1.5) |

## 14. References

- Visual spec: https://avielpo3.github.io/uber-marketplace-flow/ and the claude.ai artifact "Uber Marketplace Flow"
- Jira comments on DAAS-635 – DAAS-638 (implementation and design changes, 2026-10-26 – 2026-10-28)
- `docs/cutover-2026-09-18/uk-tracker-cutover.html`, `docs/cutover-2026-09-18/callers-report.md`
- `tracker-service/README.md`, `tracker-service/CLAUDE.md`
- Slack: questions to Sergey Shelekhov (DM, 2026-10-22). Spec shared with Sergey, Moshe Basall and Ariel Pinchover.

## 15. Tracking

| Item | Key | Branch (commits) | Assignee | QA |
|---|---|---|---|---|
| Epic (relates to ALGO-2532) | DAAS-634 | – | – | – |
| daas-uber | DAAS-635 | `DAAS-635-Store-marketplace-flag-and-forward-callbacks` (`87f7145`, `4c59285`) | Sergey Shelekhov | Tal Zamir |
| daas-gateway | DAAS-636 | `DAAS-636-Marketplace-callbacks-route-to-RabbitMQ` (`0ab656c`, `d7ba821`, `84d373d`) | Sergey Shelekhov | Tal Zamir |
| tracker-service (blocked by DAAS-639) | DAAS-637 | `DAAS-637-Consume-marketplace-callbacks` (`483a778`) | Sergey Shelekhov | Tal Zamir |
| mp-uber | DAAS-638 | `DAAS-638-remove-the-switch-and-support-daas-events` (`8f18d94`) | Sergey Shelekhov | Tal Zamir |
| Infra (DevOps task) | DAAS-639 | – | Ehab Jaber | – |

> **As implemented (v8, 2026-10-29):** Jira on 2026-10-29: DAAS-634 is In Progress with assignee Sergey Shelekhov (the Epic row shows `–`); DAAS-635, DAAS-636 and DAAS-637 are Peer Review, DAAS-638 is Dev Test; DAAS-639 is To Do with assignee Alexander Hodak; ALGO-2532 is Open, assignee Ariel Pinchover, fix version Algo 2.65.0 UK (unreleased, release date 2026-11-16). The QA column was not checked. All seven listed commits are in the deployed tags (87f7145, 4c59285 in daas-uber 11.5.7_dev; 0ab656c, d7ba821, 84d373d in daas-gateway 1.6.0_dev; 483a778 in tracker-service 2.0.4_dev; 8f18d94 in mp-uber 0.1.9_2_dev). The subject of 8f18d94 is "Disable the switch to Uber delivery by configuration", so the branch name "remove-the-switch" does not describe the behaviour (Jira GET 2026-10-29; `git merge-base --is-ancestor`, research 08)


Repos: https://bitbucket.org/dragontailcom/daas-uber, https://bitbucket.org/dragontailcom/daas-gateway, https://bitbucket.org/dragontailcom/tracker-service, https://bitbucket.org/dragontailcom/mp-uber

Pending (2026-10-28): O-8 (`daas-uber-uk`), O-9 (mp-uber market key), DAAS-639 deployment.

> **As implemented (v8, 2026-10-29):** O-9 shows a mismatch in QA and the tracker-service deployment exists in QA; see §23 (QA, 2026-10-29)


## 16. Endpoint catalogue

Routes as registered in the deployed refs (daas-uber 11.5.7_dev, daas-gateway 1.6.0_dev, tracker-service 2.0.4_dev, mp-uber 0.1.9_2_dev, algo 2.65.0_ALGO-2532). The middleware column is what the route declaration shows. Sources: research 01, 02, 03, 05, 06, 09.

### 16.1 Algo outbound (callers)

| Method | Path | Purpose | Source (file:line) |
|---|---|---|---|
| POST | `{MP base URL}/mp/uber/order/status` (`MPAggName` = `uber`) | `NewOrderMakeline` only on this ref; header `Content-Type: application/json` only | algo:`AggregatorsAPI.go:3437`, `constValues.go:222`, body `:3418-3438`, header `:3524` |
| POST | `{MP base URL}/mp/uber/order/switch` | Switch, still called in the non-`uber-daas` branch | algo:`AggregatorsAPI.go:3380-3382`, `constValues.go:221`, body `:3360-3365` |
| POST | `{store.AggServiceConfiguration.URL}/api/v2/deliveries` | Create delivery; for marketplace orders `aggProvider` and `marketplace` are set to `uber`; no query string | algo:`constValues.go:216`, `AggregatorsAPI.go:620, 873, 915, 3481-3482` |

Algo's `retryMaxAttempts` (default 5) and `retryMaxSeconds` (default 30) govern `updateMarketPlaceWithDecision` (algo:`AggregatorsAPI.go:3509-3510`). The gateway-side hops of these calls are G3 and G13/G14 below.

### 16.2 daas-gateway (api/handler/router.go, cmd/daas-gateway/server.go)

Global middleware chain (router.go:40-55): hlog, RequestIDHandler, TraceIDHandler, RealIP, URLFormat, SetHeader JSON, CORS, Compress, Heartbeat `/ping`, AccessHandler, Recoverer, `jwtauth.Verifier`. Routes of the flow:

| # | Method | Exact path | Handler (file:line) | Auth / middleware | Route decl (file:line) |
|---|---|---|---|---|---|
| G1 | ANY (chi Heartbeat) | `/ping` | chi `Heartbeat` | none | router.go:52 |
| G2 | POST | `/api/v2/deliveries/quotes` | `HandleQuote` (api/handler/proxy.go:284) | `SetCtxService`(aggService) via `/api` group (router.go:81) | router.go:93-94 |
| G3 | POST | `/api/v2/deliveries`, `/api/v2/deliveries/` | `Proxy` (proxy.go:60) | same | router.go:95 |
| G4 | PATCH | `/api/v2/deliveries/` | `Proxy` (proxy.go:60) | same | router.go:96 |
| G5 | PATCH | `/api/v2/deliveries/{id}` | `Proxy` (proxy.go:60) | same | router.go:97 |
| G6 | POST | `/api/v2/deliveries/eta` | `Proxy` (proxy.go:60) | same | router.go:98 |
| G7 | GET | `/api/v2/deliveries/icon/{providerID}` | `GetIcon` (proxy.go:204) | same | router.go:99 |
| G8 | DELETE | `/api/v2/deliveries/` | `Proxy` (proxy.go:60) | same | router.go:100 |
| G9 | POST | `/api/v2/deliveries/rate` | `Proxy` (proxy.go:60) | same | router.go:101 |
| G10 | POST | `/callbacks/{brand}/{country}/{storeNo}` (+ `/`) | `Callback` (api/handler/callback.go:59), returns 202 (callback.go:109) | none | router.go:106-110 |
| G11 | POST | `/callbacks/{brand}/{franchisee}/{country}/{storeNo}` (+ `/`) | `Callback` (callback.go:59) | none | router.go:111-115 |
| G12 | POST | `/marketplace/{provider}/callbacks` (+ `/`) | `HandleMarketplaceCallback` (api/handler/marketplaceCallback.go:63), returns 202 (marketplaceCallback.go:163) | none | router.go:162-166 |
| G13 | POST | `/mp/{provider}/order/status` | `MarketplaceProxy` (api/handler/marketplaceProxy.go:21) | `SetCtxService(serviceProvider)` (router.go:151), `LogRequest` (router.go:153) | router.go:149-155 |
| G14 | POST | `/mp/{provider}/order/switch` | `MarketplaceProxy` (marketplaceProxy.go:21) | same as G13 | router.go:155 |
| G15 | POST | `/mp/{provider}/order/events` | `MarketplaceProxy` (marketplaceProxy.go:21) | same as G13 | router.go:156 |
| G16 | POST | `/mp/{provider}/events/locations` | `MarketplaceProxy` (marketplaceProxy.go:21) | `SetCtxService` only, no `LogRequest` | router.go:158 |
| G17 | ANY | `/tracker/env/{env}/*` | `TrackerProxy` (api/handler/trackerProxy.go:24) | `LogRequest`, `SetCtxService` (router.go:171) | router.go:170-173 |
| G18 | ANY | `/tracker/region/{region}/market/{market}/*` | `TrackerProxy` (trackerProxy.go:24) | same as G17 | router.go:174 |
| G19 | ANY | `/tracker/*` | `TrackerProxy` (trackerProxy.go:24) | same as G17 | router.go:175 |
| G20 | POST | `/services`, `/services/` | `CreateService` (api/handler/services.go:31) | `SetCtxService`, `LogRequest` (router.go:137) | router.go:136-139 |
| G21 | GET | `/services`, `/services/` | `ListServices` (services.go:167) | same as G20 | router.go:140 |
| G22 | PATCH | `/services/{id}/url` | `UpdateService` (services.go:78) | same as G20 | router.go:141 |
| G23 | GET | `/services/{id}` | `GetService` (services.go:129) | same as G20 | router.go:142 |
| G24 | DELETE | `/services/{id}` | `DeleteService` (services.go:199) | same as G20 | router.go:143 |

Not tabled (outside the flow): `/api/v1/*` (router.go:83-90), `/aggregator-services/*` (router.go:122-130), `/version`, `/login/` (router.go:117-121), `/healthcheck` (router.go:76), `/swagger`. `/mp/{provider}` resolves the target through the service name `mp-<provider>` (marketplaceProxy.go:24-25) and `/tracker` through `tracker-v2` or `tracker-v2:<env>` (trackerProxy.go:17-18, 33-34). The path prefix is stripped: `/tracker/events/subscription` reaches tracker-service as `/events/subscription` (trackerProxy.go:55-56). `/mp/{provider}/events/locations` (G16) has no counterpart route in mp-uber. The gateway forwards the request body and response unchanged and adds only `request-id` and the trace header (marketplaceProxy.go:42-45). Routing by `aggProvider` is `api/handler/proxy.go:98-116`: `provider := delivery.AggProvider`, else `delivery.ProviderID`, `aggService.GetByName`, 404 `provider not found` when the registry has no row.

### 16.3 daas-uber (internal/server/router.go)

`/version` is mounted at the root (router.go:41); everything else under `/api` (router.go:46). v1 and v2 are separate subtrees.

| # | Method | Exact path | Handler (file:line) | Auth / middleware | Route decl (file:line) |
|---|---|---|---|---|---|
| U1 | GET | `/version`, `/version/` | `GetVersion` (internal/server/version.go:9) | `BasicAuthMiddleware` (middleware.go:117) | router.go:41-43 |
| U2 | POST | `/api/v1/deliveries`, `/api/v1/deliveries/` | `CreateDelivery(parsers.V1{})` (internal/server/deliveries.go:35) | `jwtauth.Verifier` + `Authenticator` | router.go:49-57 |
| U3 | DELETE | `/api/v1/deliveries/` | `CancelDelivery(parsers.V1{})` (deliveries.go:438) | same as U2 | router.go:58 |
| U4 | POST | `/api/v1/deliveries/quotes` | `GetQuote(parsers.V1{})` (deliveries.go:258) | same as U2 | router.go:59 |
| U5 | POST | `/api/v1/deliveries/uber/delivery_status` | `s.Uber()` (internal/server/providers.go:52) | none | router.go:62-63 |
| U6 | POST | `/api/v1/deliveries/uber/courier_status` | `s.Uber()` (providers.go:52) | none | router.go:64 |
| U7 | GET | `/api/v1/icon/*` | `GetIcon` (providers.go:28) | none | router.go:67-68 |
| U8 | POST | `/api/v1/discovery`, `/api/v1/discovery/` | `StoreDiscovery` (internal/server/discovery.go:19) | `jwtauth.Verifier` + `Authenticator` | router.go:71-79 |
| U9 | POST | `/api/v2/deliveries`, `/api/v2/deliveries/` | `CreateDelivery(parsers.V2{})` (deliveries.go:35) | `TraceIDHandler` (middleware.go:28, router.go:85) + `jwtauth.Verifier` + `Authenticator` | router.go:88-96 |
| U10 | DELETE | `/api/v2/deliveries/` | `CancelDelivery(parsers.V2{})` (deliveries.go:438) | same as U9 | router.go:97 |
| U11 | POST | `/api/v2/deliveries/quotes` | `GetQuote(parsers.V2{})` (deliveries.go:258) | same as U9 | router.go:98 |
| U12 | POST | `/api/v2/discovery`, `/api/v2/discovery/` | `StoreDiscovery` (discovery.go:19) | same as U8 with TraceIDHandler | router.go:103-111 |
| U13 | GET | `/api/healthcheck` | `HealthCheck` (internal/server/healthcheck.go:8) | none | router.go:116 |
| U14 | GET | `/api/metrics` | `pMetrics.Handler()` | none | router.go:117 |
| U15 | GET | `/api/reloadUberMarketConfigs` | `ReloadUberMarketConfigs` (internal/server/franchisee.go:18) | `BasicAuthMiddleware` + `PermissionMiddleware` (middleware.go:235); only if `AGG_USEDBFORPROVIDERAUTHKEYS=true` (router.go:119) | router.go:120-124 |
| U16 | POST | `/api/uberMarketConfigs`, `/api/uberMarketConfigs/` | `CreateUberMarketConfig` (franchisee.go:184) | same as U15 | router.go:125-126 |
| U17 | GET | `/api/uberMarketConfigs`, `/api/uberMarketConfigs/` | `GetAllUberMarketConfigs` (franchisee.go:269) | same as U15 | router.go:127 |
| U18 | GET | `/api/uberMarketConfigs/{id}` | `GetUberMarketConfig` (franchisee.go:234) | same as U15 | router.go:128 |
| U19 | PUT | `/api/uberMarketConfigs/{id}` | `UpdateUberMarketConfig` (franchisee.go:291) | same as U15 | router.go:129 |
| U20 | DELETE | `/api/uberMarketConfigs/{id}` | `DeleteUberMarketConfig` (franchisee.go:353) | same as U15 | router.go:130 |

### 16.4 tracker-service (app/routes.go)

Each path is registered with and without a trailing slash (routes.go:57-61). JWT helper `middlewares/jwt.go:27-52`, `ApiScope` `middlewares/scope.go:29`, `EventsScope` `middlewares/scope.go:57`.

| # | Method | Exact path | Handler (file:line) | Auth / middleware | Route decl (file:line) |
|---|---|---|---|---|---|
| T1 | GET | `/healthcheck` | `HealthCheckController.Get` (controllers/healthCheckController.go:27) | none | routes.go:24 |
| T2 | GET | `/version` | `HealthCheckController.GetVersion` (controllers/healthCheckController.go:38) | none | routes.go:25 |
| T3 | POST | `/events/location` | `EventsController.PostLocation` (controllers/eventsController.go:42) | `JWT("proxy")` + `EventsScope` | routes.go:30, 32 |
| T4 | POST | `/events/carrier` | `EventsController.PostCarrier` (eventsController.go:147) | `JWT("proxy","tracker")` + `EventsScope` | routes.go:29-33 |
| T5 | POST | `/events/order` | `EventsController.PostOrder` (eventsController.go:88) | `EventsScope` | routes.go:34 |
| T6 | POST | `/events/notification` | `AlgoNotificationController.Post` (controllers/algoNotificationsController.go:24) | `ApiScope` | routes.go:37 |
| T7 | POST | `/events/notification/market/:market` | `AlgoNotificationController.Post` (algoNotificationsController.go:24) | `ApiScope` | routes.go:38 |
| T8 | POST | `/orders/find` | `TrackerController.Find` (controllers/trackerController.go:39) | `JWT("tracker")` + `ApiScope` | routes.go:41-44 |
| T9 | GET | `/order/getuid` | `TrackerController.GetUid` (trackerController.go:139) | `JWT("tracker")` + `ApiScope` | routes.go:45 |
| T10 | GET | `/order/state` | `TrackerController.GetState` (trackerController.go:281) | `JWT("tracker")` + `ApiScope` | routes.go:46 |
| T11 | GET | `/order/track` | `TrackerController.GetTrack` (trackerController.go:223) | `JWT("tracker")` + `ApiScope` | routes.go:47 |
| T12 | GET | `/stores/:storeid/orders/:orderid` | `TrackerController.Get` (trackerController.go:86) | `JWT("tracker")` + `ApiScope` | routes.go:48 |
| T13 | GET | `/stores/:storeid/orders/:orderid/locations` | `TrackerController.GetOrderLocations` (trackerController.go:175) | `JWT("tracker")` + `ApiScope` | routes.go:49 |
| T14 | POST | `/events/subscription` | `SubscriptionController.Post` (controllers/subscriptionsController.go:40) | `JWT("tracker")` + `ApiScope` | routes.go:51 |
| T15 | GET | `/events/subscription` | `SubscriptionController.GetAll` (subscriptionsController.go:170) | `JWT("tracker")` + `ApiScope` | routes.go:52 |
| T16 | DELETE | `/events/subscription/:id` | `SubscriptionController.Delete` (subscriptionsController.go:112) | `JWT("tracker")` + `ApiScope` | routes.go:53 |
| T17 | DELETE | `/events/subscription/key/:externalkey` | `SubscriptionController.DeleteByKey` (subscriptionsController.go:148) | `JWT("tracker")` + `ApiScope` | routes.go:54 |

Not an HTTP route but part of the flow: the `marketplace_callbacks` consumer (`consumers/marketplaceCallbacks.go:99-108`), its handler (`handlers/marketplaceCallbackHandler.go:37`) and the DLQ `<queue>_DLQ` (`consumers/marketplaceCallbacks.go:206`). Subscription responses: 200 `{id, createdAt}`, 400 `{error}`, 409 duplicate, 500 store error, 404 `Subscription not found` on delete of an unknown id (`controllers/subscriptionsController.go:47-99, :112`).

### 16.5 mp-uber (controllers/, route tags on struct fields)

Routes are declared by struct tags and registered by `web.RegisterControllers` in the external `bitbucket.org/dragontailcom/common/web` module (configuration/startup.go:104). That module's pinned version was not available locally, so the base prefix and the meaning of the `auth` / `roles` tags are not verified. Swagger `@Router` agrees with the declared paths.

| # | Method | Exact path (declared) | Handler (file:line) | Auth / middleware (declared tags) | Route decl (file:line) |
|---|---|---|---|---|---|
| M1 | GET | `/healthCheck` (swagger) | `HealthCheckController.Get` (controllers/healthCheckController.go:21) | no tag | healthCheckController.go:19 (@Router) |
| M2 | POST | `/order/status` | `UberMpEventsController.SendOrderStatus` (controllers/marketplaceOrderEventsController.go:40) | `auth:"false"`, `roles:"algo"` | marketplaceOrderEventsController.go:21 |
| M3 | POST | `/order/events` | `UberMpEventsController.ProccessOrderEvents` (marketplaceOrderEventsController.go:96) | `auth:"false"`, `roles:"tracker"` | marketplaceOrderEventsController.go:22 |
| M4 | POST | `/order/switch` | `UberMpEventsController.SwitchToUberDelivery` (marketplaceOrderEventsController.go:68); returns 404 unless `MARKETPLACE_SWITCHTOUBERDELIVERY_ENABLED` (line 70-74) | `auth:"false"`, `roles:"algo"` | marketplaceOrderEventsController.go:23 |
| M5 | GET | `/stores/reload` | `StoresController.ReloadStores` (controllers/storesController.go:20) | `method`/`route` only; comment "closed internal-only endpoint" | storesController.go:13 |
| M6 | GET | `/stores/refreshcache` | `StoresController.RefreshCache` (storesController.go:25) | same as M5 | storesController.go:14 |
| M7 | POST | `/auth/reload` | `AuthController.ReloadAuthConfiguration` (controllers/authController.go:20) | no auth tag; comment "closed internal-only endpoint" | authController.go:14 |

`ProccessOrderEvents` is spelled so in the code. It rejects a batch with 400 for a point with `Lat == 0` or `Lng == 0` or older than 2 h (marketplaceOrderEventsController.go:101-107) and answers 200, 205 (final order event, Uber update OK) or 500 (final order event, Uber update failed) (services/uberMpEventsService.go:140-154).

### 16.6 Uber Eats outbound (mp-uber to Uber)

| # | Method and URL | Auth | Body | Called from (file:line) |
|---|---|---|---|---|
| E1 | `POST {MARKETPLACE_AUTHURL}`, default `https://login.uber.com/oauth/v2/token` | client credentials as form fields | `application/x-www-form-urlencoded`: `client_id`, `client_secret`, `grant_type=client_credentials`, `scope`; response must be 2xx and carry `access_token` | `GetValidMarketToken` `services/uberMpEventsService.go:56, :243`; `GetValidToken` `:63`; `services/authService.go:130-200` |
| E2 | `POST {MARKETPLACE_URL}/eats/orders/{aggOrderId}/restaurantdelivery/status`, default base `https://api.uber.com/v1` | `Authorization: Bearer <access_token>`, `Content-Type: application/json` | `{"status": "started" or "delivered"}` | `UpdateOrderStatus` `services/uberMpEventsService.go:36-98`, `processOrderEvents` `:290-300`; URL `:351`, `types/consts.go:15` |
| E3 | `POST {MARKETPLACE_URL}/eats/byoc/restaurants/orders/event/location` | Bearer from `GetValidMarketToken(market)` | `location_request` with one location event | `processLocationEvents` `services/uberMpEventsService.go:208-274`, in a goroutine `:169-175`; `types/consts.go:17` |
| E4 | `POST {MARKETPLACE_URL}/delivery/order/{aggOrderId}/switch-fulfillment-type` (flag-gated) | Bearer from `GetValidToken(brand, country)` | `{"order_id", "switch_to_fulfillment_type": "DELIVERY_BY_UBER"}` | `services/uberMpEventsService.go:183-206`; gate `controllers/marketplaceOrderEventsController.go:70`; `types/consts.go:16` |
| E5 | `GET {MARKETPLACE_URL}/delivery/stores` (paged by `next_page_token`) and `GET {MARKETPLACE_URL}/delivery/store/{uid}` (store sync, not in the order flow) | Bearer | none | `services/storesService.go:265, :183`; `types/consts.go:18-19` |

Each call is made once; no retry loop exists (a retry only appears as a comment at `uberMpEventsService.go:168`). Default HTTP timeout is `defaultTimeout` 60 s (`configuration.yaml:10`). `SendHttpRequest` (`helpers/helpers.go:32-111`) returns a non-2xx response with a nil error; a 2xx with body `success:false` or `error_details.error_code` is an error (`helpers/helpers.go:253-257`), after which the caller removes the dedup key and returns 500 (`services/uberMpEventsService.go:355-358`). Token scope literal (`services/authService.go:24`): `eats.byoc.fulfillment.config eats.store.orders.restaurantdelivery.status eats.byoc.position`.


## 17. Payload reference

Every example is labelled with its source. "Constructed from struct" means the example was built from the struct at that file:line; values then come from the spec or from test fixtures, as stated. Field order is Go struct order.

### 17.1 Algo to mp-uber: status and switch

Request, Algo to gateway to mp-uber, `NewOrderMakeline`. Struct `OrderStatusUpdate` algo:AggregatorsAPI.go:196-203, built :3418-3426; values from SPEC §6.1 (no fixture exists in the repos):

```json
{
  "aggOrderId": "d3b93b60-c336-4b32-87e6-bed8eb0c2cab",
  "brand": "Pizza Hut:GH Pizza LTD",
  "country": "United Kingdom",
  "orderId": "26101402978",
  "orderStatus": "NewOrderMakeline",
  "storeNo": "7507"
}
```

Request shape mp-uber expects (mp-uber:types/types.go:21-29, `market` optional :18); shown for the `EnRoute` case, constructed from struct, Algo does not send it on this ref:

```json
{
  "aggOrderId": "d3b93b60-c336-4b32-87e6-bed8eb0c2cab",
  "brand": "Pizza Hut:GH Pizza LTD",
  "country": "United Kingdom",
  "market": "ph-uk",
  "orderId": "26101402978",
  "orderStatus": "EnRoute",
  "storeNo": "7507"
}
```

Success: HTTP 200, empty body (mp-uber controllers/marketplaceOrderEventsController.go:53). Uber Eats failure: HTTP 512 (`types.DtErrorStatusCode`, types/consts.go:40) with the body below (controllers/marketplaceOrderEventsController.go:48-51, types/types.go:107-120).

Response, mp-uber on an Uber Eats failure (constructed from mp-uber:controllers/marketplaceOrderEventsController.go:49, types/types.go:107-120):

```json
{
  "error": "failed to send order status. Uber Marketplace returned statusCode: 404",
  "externalStatusCode": 404
}
```

Request, Algo to mp-uber, switch (struct `SwitchOrder` algo:AggregatorsAPI.go:189-194, built :3360-3365; mp-uber `SwitchToUberDeliveryRequest` types/types.go:31-36; values from SPEC §6.1):

```json
{
  "aggOrderId": "d3b93b60-c336-4b32-87e6-bed8eb0c2cab",
  "brand": "Pizza Hut:GH Pizza LTD",
  "country": "United Kingdom",
  "storeNo": "7507"
}
```

Response, mp-uber switch while `switchToUberDeliveryEnabled` is false: HTTP 404 (mp-uber:controllers/marketplaceOrderEventsController.go:70-73, types/types.go:107-120):

```json
{
  "error": "switch to Uber delivery is disabled"
}
```

### 17.2 Algo to gateway to daas-uber: create delivery (full)

Hop 3 answer is wrapped by daas-uber as `{"status":"success","data":...}` with HTTP 200 (daas-uber internal/server/httputil.go:9-33, deliveries.go:217-218); Algo parses `AggResponse{Status, Data, Message}` (algo AggregatorsAPI.go:458-462). Caveat: `dbcon` is an external package whose JSON encoding of `DbString` / `DbFloat` / `DbInt` was not verified; the example assumes plain JSON scalars.

Go struct `AggRequest`, algo:AggregatorsAPI.go:302-326:

```go
// AggRequest structure used to call agg service API
type AggRequest struct {
	AggProvider           string         `json:"aggProvider"`
	ExternalProviderID    string         `json:"externalProviderId,omitempty"`
	BrandCountry          string         `json:"brand"` // this is sent as "brand" but includes both brand and countru
	StoreBrand            string         `json:"storeBrand"`
	StoreID               string         `json:"storeId"`
	StoreName             string         `json:"storeName"`
	StorePhone            string         `json:"storePhone"`
	Location              AggLocation    `json:"location"`
	Address               AggAddress     `json:"address"`
	TZData                TZData         `json:"tzdata"`
	PickupReadyAt         time.Time      `json:"pickupReadyAt"`
	PickupDeadLineAt      time.Time      `json:"pickupDeadLineAt"`
	Targets               []TargetOrder  `json:"targets"`
	CancelForBatch        bool           `json:"cancelForBatch"`
	IsCash                bool           `json:"isCash"`
	IsAlcoholic           bool           `json:"isAlcoholic"`
	ProviderCouriersCount map[string]int `json:"providerCouriersCount"`
	IsNotPaid             bool           `json:"isNotPaid"`
	CarrierId             string         `json:"carrierId"`
	Rating                float64        `json:"rating"`
	Reason                int64          `json:"reason"`
	Comment               string         `json:"comment"`
	Marketplace           string         `json:"marketplace,omitempty"`
}
```

Nested Go types of the request, algo:AggregatorsAPI.go:232-285, :431-433, :452-455:

```go
// AggregatorsAPI.go:232-236
type AggLocation struct {
	Lat dbcon.DbFloat `json:"lat"`
	Lng dbcon.DbFloat `json:"lng"`
}
// AggregatorsAPI.go:237-249
type AggAddress struct {
	Street           string `json:"street"`
	StreetNumber     int64  `json:"streetNumber"`
	City             string `json:"city"`
	State            string `json:"state"`
	ZipCode          string `json:"zipCode"`
	Country          string `json:"country"`
	FullAddress      string `json:"fullAddress"`
	Entrance         string `json:"entrance"`
	Floor            string `json:"floor"`
	Instructions     string `json:"instructions"`
	SecondaryAddress string `json:"secondaryAddress"`
}
// AggregatorsAPI.go:251-259
type Customer struct {
	FirstName string      `json:"firstName"`
	LastName  string      `json:"lastName"`
	Location  AggLocation `json:"location"`
	Address   AggAddress  `json:"address"`
	Note      string      `json:"note"`
	Phone     string      `json:"phone"`
	Email     string      `json:"email"`
}
// AggregatorsAPI.go:270-285
type Order struct {
	ID                string    `json:"id"`
	PosOrderId        string    `json:"posOrderId"`
	AltOrderId        string    `json:"altOrderId"`
	DailyNo           string    `json:"dailyNo"`
	Description       string    `json:"description"`
	Cost              float64   `json:"cost"`
	Cash              float64   `json:"cash"`
	Note              string    `json:"note"`
	PaymentType       int       `json:"paymentType"`
	Contactless       bool      `json:"contactless"`
	Size              string    `json:"size"`
	Items             []Item    `json:"items"`
	CreatedAt         time.Time `json:"createdAt"`
	Tip               float64   `json:"tip"`
	DropOffDeadlineAt time.Time `json:"dropOffDeadlineAt"`
}
// AggregatorsAPI.go:261-268 (Item; dbcon types, tags as read)
type Item struct {
	ID          dbcon.DbString `json:"id"`
	Description dbcon.DbString `json:"description"`
	Quantity    dbcon.DbFloat  `json:"quantity"`
	Weight      dbcon.DbFloat  `json:"weight"`
	IsAlcoholic dbcon.DbInt    `json:"isAlcoholic"`
	Modifier    dbcon.DbString `json:"modifier"`
}
// AggregatorsAPI.go:431-433
type TargetOrder struct {
	Customer Customer `json:"customer"`
	Order    Order    `json:"details"`
}
// AggregatorsAPI.go:452-455
type TZData struct {
	Name   string `json:"name"`
	Offset int    `json:"offset"`
}
```

Marketplace create-delivery request, all fields (constructed from `AggRequest` algo:AggregatorsAPI.go:302-326, :431-433, :251-259, :270-285; `aggProvider` / `marketplace` set at :3481-3482; ids and brand from SPEC §6.2; customer, item, phone and e-mail values are illustrative):

```json
{
  "aggProvider": "uber",
  "externalProviderId": "store-ext-7507",
  "brand": "Pizza Hut:GH Pizza LTD-United Kingdom",
  "storeBrand": "Pizza Hut:GH Pizza LTD",
  "storeId": "7507",
  "storeName": "Pizza Hut Example Store",
  "storePhone": "+441234567890",
  "location": { "lat": 51.5074, "lng": -0.1278 },
  "address": {
    "street": "Example Street",
    "streetNumber": 10,
    "city": "London",
    "state": "",
    "zipCode": "EC1A 1AA",
    "country": "United Kingdom",
    "fullAddress": "10 Example Street, London EC1A 1AA, United Kingdom",
    "entrance": "",
    "floor": "",
    "instructions": "",
    "secondaryAddress": ""
  },
  "tzdata": { "name": "UTC", "offset": 0 },
  "pickupReadyAt": "2026-10-29T12:30:00Z",
  "pickupDeadLineAt": "2026-10-29T12:45:00Z",
  "targets": [
    {
      "customer": {
        "firstName": "Jane",
        "lastName": "Doe",
        "location": { "lat": 51.5155, "lng": -0.1419 },
        "address": {
          "street": "Customer Road",
          "streetNumber": 5,
          "city": "London",
          "state": "",
          "zipCode": "W1D 3QF",
          "country": "United Kingdom",
          "fullAddress": "5 Customer Road, London W1D 3QF, United Kingdom",
          "entrance": "",
          "floor": "2",
          "instructions": "",
          "secondaryAddress": ""
        },
        "note": "Order ID: 41, Note: ring bell",
        "phone": "+447700900123",
        "email": "jane@example.com"
      },
      "details": {
        "id": "26101402978",
        "posOrderId": "98121",
        "altOrderId": "d3b93b60-c336-4b32-87e6-bed8eb0c2cab",
        "dailyNo": "41",
        "description": "",
        "cost": 24.5,
        "cash": 0,
        "note": "ring bell",
        "paymentType": 1,
        "contactless": false,
        "size": "",
        "items": [
          {
            "id": "PIZ-001",
            "description": "Large Pepperoni",
            "quantity": 1,
            "weight": 0,
            "isAlcoholic": 0,
            "modifier": ""
          }
        ],
        "createdAt": "2026-10-29T12:20:00Z",
        "tip": 2,
        "dropOffDeadlineAt": "2026-10-29T13:05:00Z"
      }
    }
  ],
  "cancelForBatch": false,
  "isCash": false,
  "isAlcoholic": false,
  "providerCouriersCount": null,
  "isNotPaid": false,
  "carrierId": "",
  "rating": 0,
  "reason": 0,
  "comment": "",
  "marketplace": "uber"
}
```

What daas-uber parses on the V2 path (daas-uber internal/algo/deliveries.go:6-108; keys match case-insensitively because the structs have no json tags):

```go
// internal/algo/deliveries.go:6-26 (no json tags; keys match case-insensitively)
type Delivery struct {
	ExternalProviderID string
	ExternalID         string
	Brand              string `validate:"required"` // received as brand+country
	StoreBrand         string // only brand
	Franchisee         string
	Country            string
	Region             string
	StoreID            string `validate:"required"`
	StoreName          string `validate:"required"`
	StorePhone         string `validate:"required"`
	TZData             TimezoneData
	Location           Location `validate:"required"`
	Address            AddressDetails
	PickupReadyAt      time.Time `validate:"required"`
	PickupDeadlineAt   time.Time // for T.O
	DropOffDeadlineAt  time.Time // for T.O
	Targets            []Target  `validate:"required,gt=0"`
	CancelForBatch     bool
	Marketplace        string // e.g. "uber" for Uber Eats orders delivered by Uber Direct, empty otherwise
}
// deliveries.go:43-47 (Target)
type Target struct {
	Customer     CustomerDetails `validate:"required"`
	OrderDetails OrderDetails    `json:"details" validate:"required"`
}
// deliveries.go:55-63 (CustomerDetails)
type CustomerDetails struct {
	FirstName string `validate:"required"`
	LastName  string
	Location  Location `validate:"required"`
	Address   AddressDetails
	Note      string
	Phone     string `validate:"required"`
}
// deliveries.go:65-82 (OrderDetails)
type OrderDetails struct {
	ID              string `validate:"required"`
	DailyNo         string
	ExternalID      string
	CreatedAt       time.Time
	Description     string
	Cost            float32 `validate:"required"`
	Cash            float32
	Note            string
	PaymentType     int64 // 0 = cash, 1 = credit, 2 = other
	Size            string
	Items           []Item `validate:"required"`
	Tip             float32
	PickupETA       time.Time
	DropoffETA      time.Time
	ContainsAlcohol bool
}
// deliveries.go:84-91 (Item)
type Item struct {
	ID          string
	Description string  `validate:"required"`
	Quantity    float32 `validate:"required"`
	Weight      float32 // Optional - can define the size if not provided
	IsAlcoholic int64
}
// deliveries.go:93-108 (Location, AddressDetails)
type Location struct {
	Lat float32 //yael`validate:"required"`
	Lng float32 //yael`validate:"required"`
}
type AddressDetails struct {
	Street           string
	StreetNumber     int64
	City             string
	State            string
	ZipCode          string
	Country          string
	FullAddress      string
	Entrance         string
	Instructions     string
	SecondaryAddress string
}
```

V1 `Order` (daas-uber internal/server/parsers/v1.go:117-131); has `Marketplace` (:130) and no `AltOrderId` / `PosOrderId`:

```go
type Order struct {
	ExternalProviderID string
	Brand              string `validate:"required"`
	Country            string
	Region             string
	StoreID            string `validate:"required"`
	StoreName          string `validate:"required"`
	StorePhone         string `validate:"required"`
	TZData             algo.TimezoneData
	Location           algo.Location `validate:"required"`
	Address            algo.AddressDetails
	Customer           CustomerDetails `validate:"required"`
	Details            OrderDetails    `validate:"required"`
	Marketplace        string
}
```

Response structs (daas-uber internal/providers/provider.go:122-150); `RealStatus` is `json:"-"`:

```go
type Delivery struct {
	ID         string         `json:"id"`
	ProviderID string         `json:"providerID"`
	QuoteID    string         `json:"quoteID"`
	Courier    CourierDetails `json:"courier"`
	Fee        float32        `json:"fee"`
	Currency   string         `json:"currency"`
	UpdatedAt  time.Time      `json:"updatedAt"`
	CreatedAt  time.Time      `json:"createdAt"`
	Targets []Target `json:"targets"`
}
type Target struct {
	ID                 string         `json:"id"`
	OrderID            string         `json:"orderID"`
	Status             DeliveryStatus `json:"status"`
	RealStatus         DeliveryStatus `json:"-"`
	PickupDuration     int64          `json:"pickupDuration"`
	DropoffDuration    int64          `json:"dropoffDuration"`
	QuotedDeliveryTime time.Time      `json:"quotedDeliveryTime"`
	QuotedPickupTime   time.Time      `json:"quotedPickupTime"`
	DropoffETA         time.Time      `json:"dropoffETA"`
	PickupETA          time.Time      `json:"pickupETA"`
	ActualDeliveryTime time.Time      `json:"actualDeliveryTime"`
	ActualPickupTime   time.Time      `json:"actualPickupTime"`
	AggCustomID        string         `json:"aggCustomID"`
}
// CourierDetails tags: id, status, firstName, lastName, vehicleType, phone, location{lat,lng}, rating, loginStatus, activeStore
```

Response to Algo (constructed from the structs at daas-uber internal/providers/provider.go:122-150, wrapped per httputil.go:9-33; values illustrative, `status` fields are numeric enums):

```json
{
  "status": "success",
  "data": {
    "id": "d3b93b60-delivery-uber-id",
    "providerID": "uber",
    "quoteID": "quote-abc123",
    "courier": {
      "id": "",
      "status": 0,
      "firstName": "",
      "lastName": "",
      "vehicleType": "",
      "phone": "",
      "location": { "lat": 0, "lng": 0 },
      "rating": 0,
      "loginStatus": "",
      "activeStore": ""
    },
    "fee": 4.5,
    "currency": "GBP",
    "updatedAt": "2026-10-29T12:21:00Z",
    "createdAt": "2026-10-29T12:21:00Z",
    "targets": [
      {
        "id": "uber-delivery-target-id",
        "orderID": "26101402978",
        "status": 0,
        "pickupDuration": 0,
        "dropoffDuration": 0,
        "quotedDeliveryTime": "2026-10-29T13:00:00Z",
        "quotedPickupTime": "2026-10-29T12:40:00Z",
        "dropoffETA": "2026-10-29T13:00:00Z",
        "pickupETA": "2026-10-29T12:40:00Z",
        "actualDeliveryTime": "0001-01-01T00:00:00Z",
        "actualPickupTime": "0001-01-01T00:00:00Z",
        "aggCustomID": "41"
      }
    ]
  }
}
```

Table `algo_discovery` (daas-uber migrations/1.create_algo_discovery.sql):

```sql
CREATE TABLE IF NOT EXISTS algo_discovery(
    id BIGSERIAL PRIMARY KEY,
    store_id  TEXT NOT NULL,
    brand_id TEXT NOT NULL,
    agg_id TEXT,
    order_id TEXT,
    delivery_status INT,
    merchant_id TEXT,
    quote_id TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    canceled_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ,
    CONSTRAINT store_brand_order UNIQUE (store_id,brand_id,order_id)
);
```

Migration 19 (daas-uber migrations/19.add_marketplace_to_algo_discovery.sql, verbatim; no Down):

```sql
-- +migrate Up
ALTER TABLE algo_discovery
  ADD COLUMN IF NOT EXISTS marketplace TEXT;
```

Write path (daas-uber internal/algo/repository.go:71-88): one row per target; `marketplace` is NULL unless `delivery.IsMarketplace()` (:79-82):

```sql
INSERT INTO algo_discovery (store_id,brand_id,franchisee,order_id,merchant_id,marketplace)
VALUES ... -- 6 columns per target
ON CONFLICT (store_id,brand_id,order_id)
DO UPDATE SET updated_at = NOW(),
    delivery_status = 0,
    canceled_at = null,
    marketplace = COALESCE(EXCLUDED.marketplace, algo_discovery.marketplace);
```

Read path `GetOrderByAggID` (daas-uber internal/algo/repository.go:115-128); `GetOpenOrders` (:131-137) does not select `marketplace`:

```sql
SELECT store_id,brand_id,franchisee,order_id,agg_id,merchant_id,delivery_status,job_id,pickup_eta,dropoff_eta,courier_status,marketplace
FROM algo_discovery
WHERE agg_id = $1 And canceled_at is null
LIMIT 1
```

| JSON (Algo) | Go (daas-uber) | Column |
|---|---|---|
| `storeId` | `Delivery.StoreID` | `store_id` |
| `brand` (split) | `Delivery.Brand` (`brand-country`) | `brand_id` |
| `brand` franchisee part | `Delivery.Franchisee` | `franchisee` |
| `targets[].details.id` | `OrderDetails.ID` | `order_id` |
| `externalProviderId` | `Delivery.ExternalProviderID` | `merchant_id` |
| `marketplace` | `Delivery.Marketplace` | `marketplace` |
| `altOrderId`, `posOrderId` | not mapped | not stored |

Kept by daas-uber: `storeId`, `storeName`, `storePhone`, `storeBrand`, `brand` (re-split by `splitBrandCountry`, parsers/v2.go:82-92), `externalProviderId`, `location`, `address` (except `floor`), `tzdata`, `pickupReadyAt`, `pickupDeadLineAt`, `targets`, `cancelForBatch`, `marketplace` (lower-cased, parsers/v2.go:49, :55-57). The dropped fields are listed in the §6.2 note.

### 17.3 daas-uber to gateway: marketplace callback, and the queue message

daas-uber structs (internal/server/marketplace.go:30-56). No `omitempty` on `status`, `targetStatus`, `courierStatus`, `lat`, `lng`; the `DeliveryStatus` / `CourierStatus` types are plain `int` and encode as integers (internal/providers/provider.go:17-28):

```go
type MarketplaceCallback struct {
	Marketplace       string                        `json:"marketplace"`
	Brand             string                        `json:"brand"`
	Franchisee        string                        `json:"franchisee,omitempty"`
	Country           string                        `json:"country"`
	StoreNo           string                        `json:"storeNo"`
	OrderID           string                        `json:"orderId"`
	DeliveryID        string                        `json:"deliveryId"`
	Timestamp         time.Time                     `json:"timestamp"`
	OrderStatusUpdate *MarketplaceOrderStatusUpdate `json:"orderStatusUpdate,omitempty"`
	LocationUpdate    *MarketplaceLocationUpdate    `json:"locationUpdate,omitempty"`
}

type MarketplaceOrderStatusUpdate struct {
	Status        string                   `json:"status"`
	TargetStatus  providers.DeliveryStatus `json:"targetStatus"`
	CourierStatus providers.CourierStatus  `json:"courierStatus"`
}

type MarketplaceLocationUpdate struct {
	Lat float32 `json:"lat"`
	Lng float32 `json:"lng"`
}
```

daas-gateway message struct (api/handler/marketplaceCallback.go:21-53); `MarketplaceCallback` has the same fields and tags as daas-uber's:

```go
type MarketplaceCallbackMessage struct {
	Provider string `json:"provider"`
	Market   string `json:"market"`
	MarketplaceCallback
	RequestID string `json:"requestId,omitempty"`
}
```

tracker-service reads the message into `MarketplaceCallbackMessage` (types/marketplaceCallbacks.go:9-36) with fields in this order: `Provider` `provider`, `Market` `market`, `RequestId` `requestId,omitempty`, `Marketplace` `marketplace`, `Brand` `brand`, `Franchisee` `franchisee,omitempty`, `Country` `country`, `StoreNo` `storeNo`, `OrderId` `orderId` (Algo order id), `DeliveryId` `deliveryId`, `Timestamp` `timestamp`, `OrderStatusUpdate` `orderStatusUpdate,omitempty`, `LocationUpdate` `locationUpdate,omitempty`. `MarketplaceOrderStatusUpdate{Status string "status"; TargetStatus DaasDeliveryStatus "targetStatus"; CourierStatus DaasCourierStatus "courierStatus"}`; `MarketplaceLocationUpdate{Lat float64 "lat"; Lng float64 "lng"}`. The webhook is parsed into `LatLng{Lat float32 "lat", Lng float32 "lng"}` and `MarketplaceWebhook{Kind string "kind", Status string "status", Location *LatLng "location"}` (daas-uber internal/providers/uber/types.go:44-47, :291-295).

Status only, gateway body (constructed from daas-uber internal/server/marketplace.go:30-50 and the gateway struct api/handler/marketplaceCallback.go:21-39; values from daas-gateway api/handler/marketplaceCallback_test.go:39, :65, :101); `franchisee` omitted because empty:

```json
{"marketplace":"uber","brand":"Pizza Hut","country":"United Kingdom","storeNo":"123","orderId":"order-1","deliveryId":"del_1","timestamp":"2026-10-26T12:00:00Z","orderStatusUpdate":{"status":"pickup","targetStatus":2,"courierStatus":2}}
```

Status only, queue form (tracker-service tests/handlers/marketplaceCallbackHandler_test.go:19-20, `statusMessage`):

```json
{"provider":"uber","market":"ph-uk","storeNo":"123","orderId":"9001","deliveryId":"del_1","orderStatusUpdate":{"status":"pickup","targetStatus":3,"courierStatus":5}}
```

Location only, gateway body (constructed from daas-uber internal/server/marketplace.go:32-56; values from daas-gateway api/handler/marketplaceCallback_test.go:95, :101):

```json
{"marketplace":"uber","brand":"Pizza Hut","country":"United Kingdom","storeNo":"123","orderId":"order-1","deliveryId":"del_1","timestamp":"2026-10-26T12:00:00Z","locationUpdate":{"lat":51.5,"lng":-0.1}}
```

Location only, queue form (tracker-service tests/handlers/marketplaceCallbackHandler_test.go:17-18, `locationMessage`):

```json
{"provider":"uber","market":"ph-uk","requestId":"req-1","marketplace":"uber","brand":"Pizza Hut","country":"United Kingdom","storeNo":"123","orderId":"9001","deliveryId":"del_1","timestamp":"2026-10-28T10:00:00Z","locationUpdate":{"lat":51.5,"lng":-0.1}}
```

Both groups (verbatim from daas-gateway api/handler/marketplaceCallback_test.go:64-66, `TestHandleMarketplaceCallbackPublishes`; `franchisee` absent):

```json
{"marketplace":"uber","brand":"Pizza Hut","country":"United Kingdom","storeNo":"123","orderId":"order-1","deliveryId":"del_1","orderStatusUpdate":{"status":"pickup","targetStatus":2,"courierStatus":2},"locationUpdate":{"lat":51.5,"lng":-0.1},"timestamp":"2026-10-26T12:00:00Z"}
```

Gateway 400 body for a request without `orderId` (constructed from daas-gateway api/response/json.go:18-25, :47-59 and api/handler/marketplaceCallback.go:89-118; test api/handler/marketplaceCallback_test.go:126, no message published :141-143), HTTP 400:

```json
{"status":"error","message":"orderId is required"}
```

| Check (in order) | Line | Status | `message` |
|---|---|---|---|
| provider URL param empty | 68-73 | 400 | `provider is required` (not reachable via the route) |
| body read error | 75-80 | 400 | Go error text |
| body not JSON | 82-87 | 400 | Go json error text |
| `orderId` empty | 89-93 | 400 | `orderId is required` |
| `storeNo` empty | 95-99 | 400 | `storeNo is required` |
| 0,0 `locationUpdate` | 101-105 | no error | location set to nil, logged |
| neither update group | 107-111 | 400 | `orderStatusUpdate or locationUpdate is required` |
| `brand` or `country` empty | 114-118 | 400 | `brand and country are required` |
| queue send error | 133-139 | 500 | `failed to queue message` |

Lines are daas-gateway `api/handler/marketplaceCallback.go`. Success is `202 Accepted` with no body (:163). Error envelope `{"status":"error","message":"<msg>"}` (api/response/json.go:14, :18-25, :47-59).

| Queue item | Value | Source |
|---|---|---|
| Name | `marketplace_callbacks` | daas-gateway config/queue.go:13; tracker-service configuration/startup.go:267-268, configuration.yaml:50 |
| DLQ | `marketplace_callbacks_DLQ` | daas-gateway pkg/queue/client.go:258; tracker-service consumers/marketplaceCallbacks.go:205-207 |
| Exchange | default (`""`), routing key = queue name | pkg/queue/client.go:181-182 |
| Durable, persistent | queue durable; `DeliveryMode: amqp.Persistent` | pkg/queue/client.go:169, :188; consumers/marketplaceCallbacks.go:99, :131 |
| Content-Type | `application/json` | pkg/queue/client.go:189 |
| Per-message expiration | `"900000"` ms (15 min) | api/handler/marketplaceCallback.go:134; pkg/queue/client.go:273-276 |
| Queue arguments (gateway and tracker) | `x-dead-letter-exchange: ""`, `x-dead-letter-routing-key: marketplace_callbacks_DLQ` | pkg/queue/client.go:265-271; consumers/marketplaceCallbacks.go:99-104 |
| DLQ arguments | tracker: none (nil); gateway: never declared | consumers/marketplaceCallbacks.go:131; pkg/queue/client.go:168-177 |
| Headers | none (AMQP `Timestamp` = now) | pkg/queue/client.go:185-187 |
| Gateway publish retries | 5 attempts per message, then logged and dropped | pkg/queue/client.go:128-158 |
| Consumer | manual ack, prefetch 50, 8 workers; invalid message `Nack(false,false)` to the DLQ; other failures acked; reconnect 5 s up to 1 min | tracker-service consumers/marketplaceCallbacks.go:22-23, :179-192, configuration/startup.go:270-274 |

### 17.4 Redis values (tracker-service)

Illustrative values with field names and tags from the cited source; the coordinates and the Algo id come from SPEC §6.3.

Forwarding source entry, element of `tr-svc:fwd:ph-uk:7507:26101402978` and `tr-svc:fwd:ph-uk:7507` (tracker-service types/forwarding.go:5-11; built services/locationsService.go:50; carrierId = Uber delivery id, services/marketplaceCallbacksService.go:82; orders = Algo id :85):

```json
{"lat":51.913002,"lng":-0.20857728,"timestamp":1791975067000,"carrierId":"del_UowjW4AJTSGDNwqOOFCwCg","orders":["26101402978"]}
```

Location list element, `tr-svc:market:ph-uk:store:7507:posorder:PO-123:locations` (tracker-service types/dtos.go:32-37; `orders` nil and omitted, services/locationsService.go:69):

```json
{"lat":51.913002,"lng":-0.20857728,"timestamp":1791975067000}
```

Order entity, string at `tr-svc:market:ph-uk:store:7507:posorder:PO-123` (tracker-service types/entities.go:33-52; nullable fields eta, status, carrierStatus, dispatchedAt, lastUpdatedAt, source left out because their JSON shape comes from the external `common` module):

```json
{"storeNo":7507,"posOrderId":"PO-123","orderId":"26101402978","carrierId":"del_UowjW4AJTSGDNwqOOFCwCg","createdAt":"2026-10-22T12:31:07Z","lastRefreshTimestamp":1791975067000}
```

Algo id mapping, string at `tr-svc:market:ph-uk:store:7507:algoid:26101402978` (raw value, not JSON; tracker-service dataAccess/repositories/ordersRepository.go:223-228):

```
PO-123
```

Carrier current orders, set at `tr-svc:market:ph-uk:store:7507:carrier:del_UowjW4AJTSGDNwqOOFCwCg:currentorders` (tracker-service dataAccess/repositories/carrierRepository.go:100; members are posOrderIds):

```json
["PO-123"]
```

### 17.5 Subscription (mp-uber to gateway to tracker-service)

Default URL `POST {GATEWAY_URL}/tracker/events/subscription`; with `GATEWAY_USEREGIONMARKETROUTING` true `POST {GATEWAY_URL}/tracker/region/{Region}/market/{market}/events/subscription` (mp-uber services/trackerV2Service.go:65-67). mp-uber sends no `settings`; `maxAllowedTimeMins` is not sent, so tracker applies its 2 h default.

Subscription request (constructed from mp-uber types/types.go:72-79 and services/trackerV2Service.go:52-59; tracker `CreateSubscriptionRequest` has the same names, types/eventSubscriptions.go:92-110):

```json
{
  "market": "ph-uk",
  "storeNo": 7507,
  "orderId": 26101402978,
  "externalKey": "<encrypted: market::storeNo::orderId::aggOrderId[::marketplaceStoreId]>",
  "url": "{gatewayBaseUrl}/mp/uber/order/events",
  "events": ["location", "order"]
}
```

Subscription response (constructed from tracker-service types/eventSubscriptions.go:113-116, returned controllers/subscriptionsController.go:99; mp-uber reads only `id`, types/types.go:81-84):

```json
{
  "id": 1234,
  "createdAt": "2026-10-28T12:00:00.123456789Z"
}
```

Duplicate, HTTP 409 (constructed from tracker-service controllers/subscriptionsController.go:93-97; the JSON key `error` is assumed because its tag is in the external `common/web` module):

```json
{
  "error": "Subscription with the same combination of parameters already exists. key=market::ph-uk::store::7507::order::26101402978::event::location;order"
}
```

| Rule | Detail | Source (tracker-service) |
|---|---|---|
| market | if present `^[a-zA-Z]{2,8}-[a-zA-Z]{2,32}(-[a-zA-Z]{2,32})?$`, length 2-30 | helpers/helpers.go:28; controllers/subscriptionsController.go:47-53; types/eventSubscriptions.go:94 |
| `storeNo` | required, numeric, negative gives 400 | types/eventSubscriptions.go:97; controllers/subscriptionsController.go:55-61 |
| `orderId` | optional, numeric | types/eventSubscriptions.go:100 |
| `externalKey` | optional, length 1-500 | types/eventSubscriptions.go:103 |
| `url` | required; after `{...}` variables are replaced it must pass `govalidator.IsURL`, else 400 `Invalid url` | types/eventSubscriptions.go:106; controllers/subscriptionsController.go:70-78 |
| `events` | each in `location`, `carrier`, `order` (case-insensitive), no duplicates; stored lower-cased and sorted | controllers/subscriptionsController.go:187-208; types/consts.go:39-46; services/subscriptionService.go:53-58 |
| duplicate key | `market::<m>::store::<s>::order::<o>::event::<events;joined>` gives 409 | types/eventSubscriptions.go:27-43; controllers/subscriptionsController.go:93-97 |
| `maxAllowedTimeMins` | default `defaultMaxAllowedTime` = 2 h | configuration.yaml:30; services/subscriptionService.go:50-52 |

mp-uber builds the `externalKey` as the encrypted `market::storeNo::orderId::aggOrderId[::marketplaceStoreId]`; the 5th part is the `externalStoreId` read at push time (mp-uber helpers/helpers.go:216-225, services/uberMpEventsService.go:223-226). A tracker status outside 200-210 is an error in mp-uber, so a 409 is an error (services/trackerV2Service.go:74-75). The subscribe fails when the store is not in mp-uber's cache (services/trackerV2Service.go:31-36).

### 17.6 tracker-service to mp-uber push

Route `POST {gatewayBaseUrl}/mp/uber/order/events` (subscription URL, `{gatewayBaseUrl}` replaced from `GATEWAYBASEURL`, helpers/helpers.go:63-89). Headers from tracker: `Content-Type: application/json` and `ExtKey: <subscription externalKey>` (services/subscriptionService.go:176, jobs/fwdSubscriptionEventsJob.go:196, helpers/helpers.go:125-131). Push settings: `maxRetries` 5, `retryInterval` 2 s, `httpRequestTimeoutSec` 5 (`EVENTSUBMAXRETRIES`, `EVENTSUBRETRYDELAY`, `EVENTSUBREQUESTTIMEOUTSEC`, configuration.yaml:32-34); retry on timeout, transport error or status 500 and above, not on a 4xx; success is any 2xx; 205 removes the subscription (helpers/helpers.go:156-198, :248-250).

Location batch, forwarding job (constructed from `EventsBatchDto[LocationEventDto]` tracker-service types/dtos.go:120-128, forwarding.go:16-24; events built jobs/fwdSubscriptionEventsJob.go:147-153, :184, :278-302; `Id` has `json:"-"`):

```json
{
  "key": "<subscription externalKey>",
  "events": {
    "location": [
      { "lat": 51.913002, "lng": -0.20857728, "timestamp": 1791975067000 }
    ]
  },
  "sender": "<config.Name>",
  "totalEvents": 1,
  "timeStamp": 1791975097000,
  "traceId": "<uuid per batch>"
}
```

Order event, order fan-out (constructed from `EventsBatchDto[OrderEventData]` tracker-service services/subscriptionService.go:166-175, types/events.go:103-145, converters/dbEntityConverter.go:150-167, types/marketplaceCallbacks.go:55-88; example is a DaaS `dropoff`, courier 5, target 3):

```json
{
  "key": "<subscription externalKey>",
  "events": {
    "order": [
      {
        "orderId": 26101402978,
        "carrierId": "del_UowjW4AJTSGDNwqOOFCwCg",
        "carrierStatus": "EnrouteToDropoff",
        "orderStatus": 5,
        "status": "Dispatched",
        "deliveryETA": "0001-01-01T00:00:00Z",
        "exitGeofence": "0001-01-01T00:00:00Z",
        "enterGeofence": "0001-01-01T00:00:00Z",
        "tmpExitGeofence": "0001-01-01T00:00:00Z"
      }
    ]
  },
  "sender": "<config.Name>",
  "totalEvents": 1,
  "timeStamp": 1791975067123,
  "traceId": "<callback requestId, else a new uuid>"
}
```

Delivered variant: `"orderStatus": 6, "status": "Delivered"`, same fields otherwise; courier 7 gives `"carrierStatus": "DroppedOff"` (types/marketplaceCallbacks.go:94-104). The four zero-time fields appear because no custom marshaller exists in `types` (types/events.go:130-145). mp-uber reads the event as `OrderEvent` (types/types.go:122) with `orderId`, `orderStatus` (`OrderStatusInt`) and `status`; it has no `carrierStatus` field.

| mp-uber condition | Response | Source (mp-uber) |
|---|---|---|
| body does not bind or validate | 400 (handled in the `common` module) | controllers/marketplaceOrderEventsController.go:97 |
| a location with `Lat == 0` or `Lng == 0`, or older than 2 h | 400 `{"error":"invalid location event: <%+v of the point>"}`, whole batch rejected | controllers/marketplaceOrderEventsController.go:101-107 |
| key cannot be decrypted or has fewer than 4 `::` parts | 400 `Invalid key: <key>` | services/uberMpEventsService.go:126-131 |
| final order event (`Delivered`) and Uber update OK | 205 | services/uberMpEventsService.go:145-152; controllers/marketplaceOrderEventsController.go:115 |
| final order event and Uber update failed | 500 | services/uberMpEventsService.go:153-154 |
| non-final order event whose Uber update failed | 200, failure logged | services/uberMpEventsService.go:140-144, :180 |
| normal success | 200, location work in a goroutine | services/uberMpEventsService.go:167-180 |

Only the latest order event of a batch is processed; `Delivered`, `Cancel*` and `NotHome` are preferred (services/uberMpEventsService.go:132-134, :317-337). A batch whose order event is final returns 205 before its location points are looked at (:145-152 precedes :160-179). Locations are sent only when the stored row is `started` (:167), in a goroutine, errors logged only (:169-175), only the newest point (:229-239). The forwarding job groups subscriptions by (externalKey, resolved url), reads each subscription's source list past its cursor, and advances the cursor after the batch is handed to the sender, not after the response, so delivery is at-most-once (jobs/fwdSubscriptionEventsJob.go:158-183, :206-222, :256-266).

### 17.7 mp-uber to Uber Eats

Status request (constructed from mp-uber types/mpDtos.go:3-5, types/consts.go:15, services/uberMpEventsService.go:350-353):

```http
POST https://api.uber.com/v1/eats/orders/{aggOrderId}/restaurantdelivery/status
Authorization: Bearer <access_token>
Content-Type: application/json

{"status":"started"}
```

Location request (constructed from mp-uber types/mpDtos.go:7-34, types/types.go:52-70, services/uberMpEventsService.go:258-268; values illustrative):

```http
POST https://api.uber.com/v1/eats/byoc/restaurants/orders/event/location
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "location_request": {
    "order_workflow_uuid": "<aggOrderId>",
    "restaurant_uuid": "<externalStoreId from subscription key>",
    "is_batched_order": false,
    "location_events": [
      {
        "position_event": {
          "point": {"latitude": 52.52, "longitude": 13.405},
          "time": {"epochMillis": 1761650000000}
        }
      }
    ]
  }
}
```

Token request (constructed from mp-uber services/authService.go:22-25, :169-175, helpers/helpers.go:118-131):

```http
POST https://login.uber.com/oauth/v2/token
Content-Type: application/x-www-form-urlencoded

client_id=<clientId>&client_secret=<clientSecret>&grant_type=client_credentials&scope=eats.byoc.fulfillment.config+eats.store.orders.restaurantdelivery.status+eats.byoc.position
```

Token response (shape from mp-uber types/mpDtos.go:41-46, expiry from services/authService.go:200; `expires_in` illustrative):

```json
{
  "access_token": "<token>",
  "token_type": "Bearer",
  "expires_in": 2592000,
  "scope": "eats.byoc.fulfillment.config eats.store.orders.restaurantdelivery.status eats.byoc.position"
}
```

Error body shape (mp-uber types/mpDtos.go:48-56; checked only on 2xx, helpers/helpers.go:249-257):

```json
{"success": false, "error_details": {"error_code": "<code>", "error_message": "<message>"}}
```

Switch request, flag-gated (constructed from mp-uber types/mpDtos.go:36-39, types/consts.go:16, :58):

```http
POST https://api.uber.com/v1/delivery/order/{aggOrderId}/switch-fulfillment-type
Authorization: Bearer <access_token>

{"order_id":"<aggOrderId>","switch_to_fulfillment_type":"DELIVERY_BY_UBER"}
```


## 18. Configuration reference

Env var names are copied exactly, including misspellings. Defaults are those in code or YAML; QA values come from the QA deployment (09 and the QA state of 2026-10-29). Rows for JWT keys, passwords, admin users and the PII key are not listed. Sources: research 04 and 09.

### 18.1 daas-uber (envconfig, prefix `AGG`, internal/config/config.go)

envconfig has no `split_words` tag, so the env name is `AGG_` plus the field name in capitals with no underscores. Defaults come only from `load()` (config.go:225-276) or are zero values. Marketplace behaviour has no `AGG_*` key of its own beyond `AGG_CALLBACKURL`.

| Env var | Config field (config.go) | Code default | QA value | Meaning | file:line |
|---|---|---|---|---|---|
| `AGG_CALLBACKURL` | `CallbackURL` (:39) | none | `https://gateway-qa.daas.dtsys.cc/callbacks` | Gateway base URL. Provider callbacks build `/callbacks/{brand}/[{franchisee}/]{country}/{store}` on it. Marketplace callbacks overwrite the whole path with `marketplace/{mp}/callbacks`, so the `/callbacks` suffix from QA is dropped in that path. | internal/server/gatewayAPI.go:30-35; internal/server/marketplace.go:169-173 |
| `AGG_UBERPROVIDERURL` | `UberProviderURL` (:35) | none | `https://mock-api-qa.daas.dtsys.cc/uber` | Uber Direct API base used by `uber.New(...)` | internal/server/deliveries.go:102, 338, 509, 644; internal/server/providers.go:243, 364, 368 |
| `AGG_UBERAUTHURL` | `UberAuthURL` (:34) | none | not set | Uber OAuth token URL | internal/server/deliveries.go:102, 338, 509; internal/server/providers.go:243, 364, 369 |
| `AGG_UBERCANADACANCELURL` | `UberCanadaCancelURL` (:48) | none | not set | Base for "direct" cancel mode (only for Canada markets with `CancelUrlStructure=direct`); unset = 400 | internal/providers/uber/cancel.go:25, 45 |
| `AGG_UBERCUSTOMERID` | `UberCustomerId` (:36) | none | not set | Defined but no reader found in the repo | internal/config/config.go:36 |
| `AGG_POLLINGPROVIDERSLIST` | `PollingProvidersList` (:38) | none (empty = no polling) | not set | Providers that run order polling; empty logs a warning | internal/server/providers.go:203-205 |
| `AGG_ORDERPOLLINGTIMEOUTHOURS` | `OrderPollingTimeoutHours` (:37) | `2` (config.go:194, 262-264) | not set | Polling timeout per order | internal/server/providers.go:277, 308-309 |
| `AGG_ONLYONEPICKUPREADYATFIELD` | `OnlyOnePickupReadyAtField` (:42) | `false` (zero) | not set | If true, quote sends only one pickup_ready_at field | internal/providers/uber/quote.go:60-61 |
| `AGG_ENABLEMARKETFIELDS` | `EnableMarketFields` (:41, map) | none | not set | Per-market field enable map; no reader found in the repo | internal/config/config.go:41, 51-56 |
| `AGG_DELIVERYMETHOD` | `DeliveryMethod` (:18, map) | none | not set | Brand (or `brand-franchisee`) to delivery method; lookup keys `uber-<brand>-<franchisee>` then `uber-<brand>` | internal/providers/uber/delivery.go:236-238; config.go:58-65, 270-273 |
| `AGG_USEDBFORPROVIDERAUTHKEYS` | `UseDBForProviderAuthKeys` (:40) | `false` | not set | Loads auth keys/settings from DB and mounts `/api/uberMarketConfigs*` | cmd/uber/main.go:117; internal/server/router.go:119; config.go:123 |
| `AGG_PORT` | `Port` (:19) | `:5000` (config.go:246-248) | not set | Listen address | config.go:246-248 |
| `AGG_CURRENTENV` | `CurrentEnv` (:43) | `qa` (config.go:258-260) | not set | Logged only | cmd/uber/main.go:134; config.go:258-260 |
| `AGG_DEBUG` | `Debug` (:17) | `false` | not set | Debug mode | cmd/uber/main.go:36 |
| `AGG_LOGPATHFILE` | `LogPathFile` (:29) | `/var/log/daas-uber/daas-uber.log` (config.go:184, 242-244) | not set | Log file | config.go:242-244 |
| `AGG_DBHOST` | `DBHost` (:21) | none | not set | Postgres host | config.go:21 |
| `AGG_DBPORT` | `DBPort` (:22) | `5432` (config.go:190, 250-252) | not set | Postgres port | config.go:250-252 |
| `AGG_DBNAME` | `DBName` (:23) | none | not set | Database name | config.go:23 |
| `AGG_DBUSER` | `DBUser` (:24) | none | not set | DB user | config.go:24 |
| `AGG_DBSSLMODE` | `DBSSLMode` (:25) | `disable` (config.go:192, 254-256) | not set | SSL mode | config.go:254-256 |

### 18.2 daas-gateway (koanf, prefix `DT_`, config/*.go)

Loader `config/config.go:37-70`: every `*.toml` in the config dir is loaded, then env overrides. Env name to key: strip `DT_`, lower-case, then every `_` becomes `.` (config.go:57-59), so keys that contain `_` (`DT_CACHE_SERVICES_DATA_EXPIRATION_MINUTES`, `DT_DATABASE_SSL_MODE`) cannot be set by env. `queue.channelBufferSize` is one key read by two fields (`WorkerBufferSize` 2500, `RetryBufferSize` 1000, config/queue.go:10, :12).

| Env var | Config key (koanf) | Code default | QA value | Meaning | file:line |
|---|---|---|---|---|---|
| `DT_SERVER_PORT` | `server.port` | `8080` (config/server.go:5); TOML `port = 8193` (assets/config/0_default.toml:2) | not set | Listen port | config/server.go:5 |
| `DT_SERVER_HOST` | `server.host` | `localhost` (server.go:4) | not set | Listen host | config/server.go:4 |
| `DT_SERVER_SCHEME` | `server.scheme` | `https` (server.go:6) | not set | Scheme for audit URL in `GetProviderCallbacks` | config/server.go:6; api/handler/proxy.go:275-277 |
| `DT_SERVER_REGION` | `server.region` | `""`; falls back to `AWS_REGION` (server.go:14; cmd/daas-gateway/server.go:118-119) | not set | Region | config/server.go:14 |
| `DT_QUEUE_SCHEME` | `queue.scheme` | `amqp` (config/queue.go:4) | `amqps` | AMQP scheme for publisher | config/queue.go:4; cmd/daas-gateway/server.go:100 |
| `DT_QUEUE_HOST` | `queue.host` | `localhost:5672` (queue.go:7) | `daas-qa.mq.dtsys.cc:443` | AMQP host | config/queue.go:7; cmd/daas-gateway/server.go:102 |
| `DT_QUEUE_USERNAME` | `queue.username` | `guest` (queue.go:5) | not set | AMQP user | config/queue.go:5 |
| `DT_QUEUE_SERVICENAME` | `queue.serviceName` | `gateway` (queue.go:8) | not set | Service name | config/queue.go:8 |
| `DT_QUEUE_NUMBEROFWORKERS` | `queue.numberOfWorkers` | `10` (queue.go:9) | not set | Publisher workers | config/queue.go:9 |
| `DT_QUEUE_CHANNELBUFFERSIZE` | `queue.channelBufferSize` (used by **two** fields: `WorkerBufferSize` default 2500 and `RetryBufferSize` default 1000) | 2500 and 1000 (queue.go:10, 12) | not set | Channel buffer; both fields read the same key, so one value sets both | config/queue.go:10, 12 |
| `DT_QUEUE_NUMBEROFRETRYWORKERS` | `queue.numberOfRetryWorkers` | `5` (queue.go:11) | not set | Retry workers | config/queue.go:11 |
| `DT_QUEUE_MARKETPLACECALLBACKSQUEUE` | `queue.marketplacecallbacksqueue` | `marketplace_callbacks` (queue.go:13) | not set | Queue that `/marketplace/{provider}/callbacks` publishes to | config/queue.go:13; api/handler/marketplaceCallback.go:133 |
| `DT_AUDIT_SERVICEURL` | `audit.serviceUrl` | none | not set | Audit service host (GET `/api/v1/deliveries/*` only, out of scope) | config/audit.go:6; api/handler/proxy.go:275-277 |
| `DT_AUDIT_QUOTESENABLED` | `audit.quotesEnabled` | `false` | not set | Audit quote decisions | config/audit.go:7; pkg/proxy_helper/proxyHelper.go:157, 182 |
| `DT_AUDIT_ORDERSENABLED` | `audit.ordersEnabled` | `false` | not set | Audit order requests | config/audit.go:8; api/handler/proxy.go:128 |
| `DT_AUDIT_CALLBACKSENABLED` | `audit.callbacksEnabled` | `false` | not set | Audit provider callbacks (`/callbacks/*` only; marketplace path does not check it) | config/audit.go:9; api/handler/callback.go:97; api/handler/proxy.go:142 |
| `DT_AUDIT_CALLBACKSEXPIRATION` | `audit.callbacksExpiration` | `15m` (audit.go:10) | not set | TTL for audit messages | config/audit.go:10; api/handler/callback.go:105 |
| `DT_ENGINE_SCHEME` | `engine.scheme` | `http` (config/engine.go:4) | not set | Decision engine (quotes) | config/engine.go:4; pkg/proxy_helper/engineAPI.go:34-55 |
| `DT_ENGINE_HOST` | `engine.host` | none | not set | Decision engine host | config/engine.go:5 |
| `DT_ENGINE_PORT` | `engine.port` | none | not set | Decision engine port | config/engine.go:6 |
| `DT_ENGINE_PATH` | `engine.path` | `api/v1` (config/engine.go:9) | not set | Decision engine path | config/engine.go:9 |
| `DT_CACHE_SERVICES_DATA_EXPIRATION_MINUTES` | `cache.services_data_expiration_minutes` (underscored: **cannot bind via env**) | `60` (config/cache.go:6) | not set | TTL of the service registry cache (`tracker-v2`, `mp-*`) | config/cache.go:6; cmd/daas-gateway/server.go:134 |
| `DT_DATABASE_HOST` | `database.host` | none | not set | Postgres for service registry (`services`, `agg_services`) | config/database.go:7 |
| `DT_DATABASE_PORT` | `database.port` | none | not set | Postgres port | config/database.go:8 |
| `DT_DATABASE_USERNAME` | `database.username` | none | not set | DB user | config/database.go:5 |
| `DT_DATABASE_DATABASE` | `database.database` | `daas-gateway` (database.go:9) | not set | DB name | config/database.go:9 |
| `DT_DATABASE_SSL_MODE` | `database.ssl_mode` (underscored: **cannot bind via env**) | `""` (database.go:10) | not set | SSL mode | config/database.go:10 |

### 18.3 tracker-service (configuration.yaml, `${VAR||default}`, unprefixed)

`${VAR}` without a default is empty when unset. Go code reads no Redis, RabbitMQ, forwarding or market env var directly; the only `os.Getenv` calls are `WEB_AUTH`, `MODE`, `VERSION`. Three forwarding names use `SUBSCRTIPTION` (missing I) and must be used exactly: `FWDSUBSCRTIPTIONEVENTSJOBINTERVAL`, `FWDSUBSCRTIPTIONEVENTSJOBMAXCHUNKSIZE`, `FWDSUBSCRTIPTIONEVENTSJOBMAXGOROUTINES`; the other three use `SUBSCRIPTION` with an underscore (`FWDSUBSCRIPTIONEVENTSJOB_SOURCE_TTL`, `_MAX_SOURCE_ENTRIES`, `_VERBOSE`, plus `_ENABLED`). QA values for `RABBITMQ_*` and `DEFAULT_MARKET` are the ones set by PR #571 (§23). The Redis key TTLs and caps are in §19.

| Env var (exact) | YAML key | Default in code | QA value | Meaning | file:line |
|---|---|---|---|---|---|
| `DEFAULT_MARKET` | `defaultMarket` (:8) | none in yaml; warn only if empty (configuration/startup.go:214-217); invalid value fails startup (startup.go:211-212) | `ph-uk` (PR #571) | Market for requests and events without one | app/routes.go:30, 34, 42; configuration/startup.go:207-218 |
| `GATEWAYBASEURL` | `allowedUrlVariables.gatewayBaseUrl` (:27) | none | `https://gateway-qa.daas.dtsys.cc` | Replaces `{gatewayBaseUrl}` in subscription URLs (mp-uber sends `{gatewayBaseUrl}/mp/uber/order/events`) | configuration.yaml:27; configuration/startup.go:173-179; controllers/subscriptionsController.go:69-70 |
| `PROXYBASEURL` | `proxy.url` (:22) and `allowedUrlVariables.proxyBaseUrl` (:28) | none | not set | Single proxy base; required if `PROXY_URLS` empty | configuration.yaml:22, 28; configuration/startup.go:208-210 |
| `PROXY_URLS` | `proxy.urls` (:23) | none | `dt-dev=https://dt-dev.proxy.dtsys.cc, dt-qa=https://dt-qa.proxy.dtsys.cc` | Per-market proxy map `market=url,...`; required with `PROXYBASEURL` | configuration.yaml:23; configuration/startup.go:183, 285-300 |
| `PROXYTIMEOUT` | `proxy.timeoutAfter` (:24) | `15s` | not set | Proxy timeout | configuration.yaml:24 |
| `PROXY_ORDER_REFRESH_STALE_AFTER` | `proxy.orderRefreshStaleAfter` (:25) | `5s` | not set | Order refresh staleness | configuration.yaml:25 |
| `RABBITMQ_ENABLED` | `rabbitmq.enabled` (:44) | `false` | `true` (PR #571) | Starts marketplace_callbacks consumer | configuration.yaml:44 |
| `RABBITMQ_SCHEME` | `rabbitmq.scheme` (:45) | `amqp` (yaml; code default same, startup.go:258-260) | `amqps` (PR #571) | AMQP scheme | configuration.yaml:45 |
| `RABBITMQ_HOST` | `rabbitmq.host` (:46) | `localhost:5672` (yaml; startup.go:261-263) | `daas-qa.mq.dtsys.cc:443` (PR #571) | AMQP host | configuration.yaml:46 |
| `RABBITMQ_USERNAME` | `rabbitmq.username` (:47) | `guest` | from SSM `/nonprod/global/cloud/tracker-service/rabbitmq/username` (PR #571) | AMQP user | configuration.yaml:47 |
| `RABBITMQ_VHOST` | `rabbitmq.vhost` (:49) | `/` | `/` (PR #571) | Vhost | configuration.yaml:49; startup.go:264-266 |
| `RABBITMQ_MARKETPLACE_CALLBACKS_QUEUE` | `rabbitmq.marketplaceCallbacksQueue` (:50) | `marketplace_callbacks` | `marketplace_callbacks` (PR #571) | Queue consumed. DLQ = `<queue>_DLQ` | configuration.yaml:50; consumers/marketplaceCallbacks.go:99-108, 206 |
| `RABBITMQ_PREFETCH` | `rabbitmq.prefetch` (:51) | `50` | not set | QoS prefetch | configuration.yaml:51; consumers/marketplaceCallbacks.go:105 |
| `RABBITMQ_WORKERS` | `rabbitmq.workers` (:52) | `8` | not set | Consumer workers | configuration.yaml:52; startup.go:273-275 |
| `REDIS_MAX_LOCATIONS_PER_ORDER` | `redis.maxLocationsPerOrder` (:63) | `100` | `100` | Max stored locations per order (LTRIM) | configuration.yaml:63; dataAccess/repositories/ordersRepository.go:419 |
| `REDIS_DURATION_ORDER` | `redis.durations.order` (:66) | `72h` | not set | TTL order | configuration.yaml:66 |
| `REDIS_DURATION_ORDER_LOCATIONS` | `redis.durations.orderLocations` (:67) | `24h` | not set | TTL order locations | configuration.yaml:67 |
| `REDIS_DURATION_CARRIER_CURRENT_ORDERS` | `redis.durations.carrierCurrentOrders` (:68) | `720h` | not set | TTL carrier current orders | configuration.yaml:68 |
| `REDIS_HOST` | `redis.host` (:54) | `127.0.0.1` | not set | Redis host | configuration.yaml:54 |
| `REDIS_PORT` | `redis.port` (:55) | `6379` | not set | Redis port | configuration.yaml:55 |
| `REDIS_USERNAME` | `redis.user` (:56) | none | not set | Redis user | configuration.yaml:56 |
| `REDIS_TLS` | `redis.tls` (:58) | `false` | not set | TLS | configuration.yaml:58 |
| `REDIS_VERBOSE_LOGS` | `redis.verboseLogs` (:64) | `true` | not set | Redis logs | configuration.yaml:64 |
| `LOGINCOMINGEVENTS` | `logIncomingEvents` (:6) | `false` | not set | Logs marketplace callbacks when true | configuration.yaml:6; handlers/marketplaceCallbackHandler.go:66 |
| `LOG_INCOMING_REQUESTS` | `logIncomingRequests` (:5) | `true` | not set | HTTP request log | configuration.yaml:5; configuration/startup.go:101 |
| `LOGALGONOTIFICATIONS` | `logAlgoNotifications` (:7) | `true` | not set | Algo notification log | configuration.yaml:7 |
| `ENABLECORS` | `enableCORS` (:4) | `true` | not set | CORS | configuration.yaml:4 |
| `ENABLESWAGGER` | `enableSwagger` (:3) | `false` | not set | Swagger | configuration.yaml:3 |
| `PORT` | `port` (:2) | `5000` | not set | Listen port | configuration.yaml:2 |
| `NAME` | `name` (:1) | `tracker-v2` | not set | Service name | configuration.yaml:1 |
| `MAX_FINDORDERSLIMIT` | `maxFindOrdersLimit` (:9) | `50` | not set | Max rows for `/orders/find` | configuration.yaml:9; configuration/startup.go:170-172 |
| `DEFAULTSUBSCRIPTIONMAXALLOWEDTIME` | `eventSubscriptions.defaultMaxAllowedTime` (:30) | `2h` | not set | Subscription lifetime default | configuration.yaml:30 |
| `DBCLEANUPJOB_REMOVE_EVENT_SUBSCRIPTIONS_OLDER_THAN` | `eventSubscriptions.removeOlderThan` (:31) | `6h` | not set | Cleanup age | configuration.yaml:31 |
| `EVENTSUBMAXRETRIES` | `eventSubscriptions.maxRetries` (:32) | `5` | not set | Forward retries | configuration.yaml:32 |
| `EVENTSUBRETRYDELAY` | `eventSubscriptions.retryInterval` (:33) | `2s` | not set | Retry delay | configuration.yaml:33 |
| `EVENTSUBREQUESTTIMEOUTSEC` | `eventSubscriptions.httpRequestTimeoutSec` (:34) | `5` | not set | Forward HTTP timeout | configuration.yaml:34 |
| `FWDSUBSCRIPTIONEVENTSJOB_ENABLED` | `forwarding.enabled` (:36) | `true` | not set | Forwarder job on/off. Forwards to subscription URLs (mp-uber callback) | configuration.yaml:36 |
| `FWDSUBSCRTIPTIONEVENTSJOBINTERVAL` | `forwarding.interval` (:37) | `30s` | not set | **Misspelled** name ("SUBSCRTIPTION"), must be used exactly | configuration.yaml:37; configuration/startup.go:228-230 |
| `FWDSUBSCRTIPTIONEVENTSJOBMAXCHUNKSIZE` | `forwarding.maxChunkSize` (:38) | `20` | not set | **Misspelled** name | configuration.yaml:38; startup.go:231-233 |
| `FWDSUBSCRTIPTIONEVENTSJOBMAXGOROUTINES` | `forwarding.maxGoRoutines` (:39) | `200` | not set | **Misspelled** name | configuration.yaml:39; startup.go:234-239 |
| `FWDSUBSCRIPTIONEVENTSJOB_SOURCE_TTL` | `forwarding.sourceTtl` (:40) | `15m` | not set | Source list TTL | configuration.yaml:40; startup.go:240-245 |
| `FWDSUBSCRIPTIONEVENTSJOB_MAX_SOURCE_ENTRIES` | `forwarding.maxSourceEntries` (:41) | `3000` | not set | Source list bound | configuration.yaml:41; startup.go:246-248 |
| `FWDSUBSCRIPTIONEVENTSJOB_VERBOSE` | `forwarding.verbose` (:42) | `true` | not set | Verbose logs | configuration.yaml:42 |
| `PII_IN_LOGS` | `pii.piiInLogs` (:20) | `masked` | not set | PII logging | configuration.yaml:20 |
| `ENCRYPTPII` | `pii.encrypt` (:18) | `false` | not set | PII encryption | configuration.yaml:18 |
| `MEMCACHE_DEFAULT_CACHE_DURATION` | `memoryCache.defaultCacheDuration` (:15) | `30m` | not set | Memory cache TTL | configuration.yaml:15 |
| `MEMCACHE_DEFAULT_CLEANUP_INTERVAL` | `memoryCache.defaultCleanupInterval` (:16) | `5m` | not set | Memory cache cleanup | configuration.yaml:16 |
| `REDIS_POOL_SIZE` |  | `200` | not set | go-redis pool size | configuration.yaml:59 |
| `REDIS_TIMEOUT` |  | `10s` | not set | read and write timeout | configuration.yaml:60 |
| `REDIS_RETRY_ATTEMPTS` |  | `10` | not set | startup connect retries | configuration.yaml:61; configuration/startup.go:382 |
| `REDIS_RETRY_DELAY` |  | `2s` | not set | startup retry delay | configuration.yaml:62 |

### 18.4 mp-uber (configuration.yaml, `${VAR||default}`, unprefixed)

Loaded through `common.LoadConfigurationFileName` (configuration/startup.go:53); code defaults apply in `setDefaultValues` (startup.go:130-164). Hard-coded and not configurable: the 30 min status dedup window (services/uberMpEventsService.go:347) and the 10 min tracker subscription cache (:115, :106).

| Env var | YAML key (configuration.yaml line) | Code default | QA value | Meaning | code ref |
|---|---|---|---|---|---|
| `GATEWAY_URL` | `services.gateway.url` (:19) | none | `https://gateway-qa.daas.dtsys.cc` | Gateway base. Subscriptions go to `{GATEWAY_URL}/tracker/events/subscription` (POST) and `/tracker/events/subscription/{id}` (DELETE) | configuration.yaml:19; configuration/startup.go:72; services/trackerV2Service.go:57-67, 94-96 |
| `GATEWAY_USEREGIONMARKETROUTING` | `services.gateway.useRegionMarketRouting` (:20) | `false` | not set | Use `/tracker/region/{region}/market/{market}/...` paths (needs `AWS_REGION`/`Region`) | configuration.yaml:20; services/trackerV2Service.go:65, 94 |
| `AUTH_URL` | `services.auth.url` (:22) | none | not set | Auth service URL. No reader found in the repo (`AuthConfiguration.Url` unused). | configuration.yaml:22; types/configuration.go:50-54 |
| `MARKETPLACE_URL` | `services.marketPlace.url` (:24) | `https://api.uber.com/v1` | not set | Uber Marketplace API base | configuration.yaml:24; helpers/helpers.go:274 |
| `MARKETPLACE_AUTHURL` | `services.marketPlace.authUrl` (:25) | `https://login.uber.com/oauth/v2/token` | not set | OAuth token URL | configuration.yaml:25; services/authService.go:164 |
| `MARKETPLACE_AUTHCONFIGURATION` | `services.marketPlace.authConfiguration` (:26) | none | not set | Map `<market>-svc:tracker-v2` etc. used as fallback tracker token (`TryGetTrackerV2AuthHeader`) | configuration.yaml:26; types/configuration.go:59; helpers/helpers.go:145-160 |
| `MARKETPLACE_TRIGGERUPDATEON` | `services.marketPlace.triggerUpdateOn` (:27) | YAML `EnRoute,Delivered`; code fallback `NewOrderMakeline,Delivered` (startup.go:159-160, reached only if empty) | `Enroute,Delivered` | Statuses that push an update to Uber Marketplace (case-insensitive) | configuration.yaml:27; configuration/startup.go:158-181; services/uberMpEventsService.go:43 |
| `MARKETPLACE_SUBSCRIBEONSTATUS` | `services.marketPlace.subscribeToLocationsOnStatus` (:28) | `NewOrderMakeline` (YAML and startup.go:171-173) | `NewOrderMakeline` | Status that creates the tracker-v2 subscription | configuration.yaml:28; services/uberMpEventsService.go:44 |
| `MARKETPLACE_SWITCHTOUBERDELIVERY_ENABLED` | `services.marketPlace.switchToUberDeliveryEnabled` (:29) | `false` | not set | Enables `/order/switch`; off = 404 | configuration.yaml:29; controllers/marketplaceOrderEventsController.go:70-74 |
| `MARKETPLACE_GETSTORESCHUNKSIZE` | `services.marketPlace.refreshStoresSettings.getStoresChunkSize` (:31) | `500` (YAML; startup.go:166-168) | not set | Stores page size (Uber max 500) | configuration.yaml:31 |
| `MARKETPLACE_REFRESHSTORESSETTINGS_MIN_DELAY_BETWEEN_REQUESTS` | `...minDelayBetweenGetStoreRequests` (:32) | `30ms` | not set | Delay between store requests | configuration.yaml:32 |
| `MODE` | `mode` (:5) | `debug` (startup.go:134-136); production unless dev/debug/stub:local (startup.go:279-283) | not set | Run mode | configuration.yaml:5; configuration/startup.go:41, 279-283 |
| `PORT` | `port` (:3) | `8000` (startup.go:131-133) | not set | Listen port | configuration.yaml:3; configuration/startup.go:131-133 |
| `NAME` | `name` (:2) | `Uber-Marketplace-SP` | not set | Service name | configuration.yaml:2 |
| `TRACERCONFIGURATION` | `tracer` (:4) | none | not set | Tracer config | configuration.yaml:4 |
| `DBHOST` | `dbConnections[0].connectionString` (:8) | none | not set | Postgres host | configuration.yaml:8 |
| `DBNAME` | same (:8) | none | not set | DB name | configuration.yaml:8 |
| `DBUSER` | same (:8) | none | not set | DB user | configuration.yaml:8 |
| `DBSSLMODE` | same (:8) | `require` | not set | SSL mode | configuration.yaml:8 |
| `DBSCHEMA` | `dbConnections[0].schema` (:9) | `public` (startup.go:137-139) | not set | Schema | configuration.yaml:9 |
| `DEFAULTTIMEOUT` | `defaultTimeout` (:10) | `60s` | not set | Default HTTP timeout | configuration.yaml:10; startup.go:142-147 |
| `MEMORYCACHEENABLED` | `memoryCache.enabled` (:34) | `true` | not set | Memory cache | configuration.yaml:34 |
| `MEMORYCACHEDURATION` | `memoryCache.defaultCacheDuration` (:35) | `10m` | not set | Cache TTL | configuration.yaml:35 |
| `MEMORYCACHECLEANUPINTERVAL` | `memoryCache.defaultCleanupInterval` (:36) | `5m` | not set | Cache cleanup | configuration.yaml:36 |
| `JOB_LOADAUTHCREDS_ENABLED` | `jobs[LoadAuthCredsJob].enabled` (:39) | `true` | not set | Credential loader job | configuration.yaml:39 |
| `JOB_LOADAUTHCREDS_INTERVAL` | same (:42) | `10m` | not set | Interval | configuration.yaml:42 |
| `JOB_RELOADSTORESFROMUBER_CRONEXPRESSION` | `jobs[ReloadStoresFromUberJob].cronExpression` (:48) | `0 0 3 * * *` | not set | Store reload cron | configuration.yaml:48 |
| `JOB_DBCLEANUP_INTERVAL` | `jobs[DbCleanUpJob].interval` (:53) | `6h` | not set | Cleanup interval | configuration.yaml:53 |
| `JOB_DBCLEANUP_STORERAWREQUESTSDATAFOR` | `jobs[DbCleanUpJob].parameters.storeRawRequestsDataFor` (:54) | `30d` | not set | Raw request retention | configuration.yaml:54 |
| `SQLBULKINSERT_MAXCHUNKSIZE` | `sqlBulkInsert.maxChunkSize` (:12) | `50` | not set | Bulk insert chunk | configuration.yaml:12 |
| `SQLBULKINSERT_WORKERS` | `sqlBulkInsert.workers` (:13) | `4` | not set | Bulk insert workers | configuration.yaml:13 |
| `SQLBULKINSERT_INTERVAL` | `sqlBulkInsert.interval` (:14) | `500ms` | not set | Flush interval | configuration.yaml:14 |
| `SQLBULKINSERT_MAX_BUFFERED_CHUNKS` | `sqlBulkInsert.maxBufferedChunks` (:15) | `15000` | not set | Buffer bound | configuration.yaml:15 |
| `SQLBULKINSERT_VERBOSE_LOGS` | `sqlBulkInsert.verboseLogs` (:16) | `false` | not set | Verbose | configuration.yaml:16 |

### 18.5 Algo (option list and store configuration, not in the repo)

| Setting | Default in code | Meaning | file:line |
|---|---|---|---|
| `MarketPlace` / `retryMaxAttempts` | 5 | retries of `updateMarketPlaceWithDecision` (any non-2xx) | algo:AggregatorsAPI.go:3509 |
| `MarketPlace` / `retryMaxSeconds` | 30 | upper bound of the backoff (waits 1, 2, 4, 8, 16 s) | algo:AggregatorsAPI.go:3510 |
| `MarketPlace` / `carrierThreshold` | 30 min | store driver used when free within this time | algo:AggregatorsAPI.go:3278 |
| `MarketPlaceConfiguration.URL`, `.MPAggName`, `.Daas` | values not in the repo | mp-uber base URL, provider name (`uber`), and `"uber-daas"` (`mpDaasUber`) to book Uber Direct; any other value takes the switch branch | algo:AggregatorsAPI.go:3263-3273, :3395-3409, Stores.go:314 |

### 18.6 Gateway service registry (database rows, not env keys)

`services` table: columns `id`, `name`, `url`, `region` (NOT NULL), `market` (default `'*'`), `created_at`, `updated_at`; unique `(name, region, market)` (daas-gateway migrations/postgres/000003_create_mp_services_table.up.sql:1-11). `Search(name, region, market)` uses `name = ?`, region `= ?` when given else `region IS NULL OR region = ''`, market `= ?` when given else `market IS NULL OR market = '*'`, `LIMIT 1`, no `ORDER BY` (repository/service_db.go:92-120), cached in `ServicesDataCache` (use_case/service_use_case/service.go:106-125), TTL `cache.services_data_expiration_minutes` 60. A missing row answers 404 (`service not found. name: ... region: ... market: ...`, trackerProxy.go:45-48). No repo seeds a `tracker-v2` or `mp-uber` row.

| name | url in QA | source |
|---|---|---|
| `tracker-v2` | `tracker-service.cloud.global.dtsys.cc` (repointed 2026-10-29 by `PATCH /services/1/url`; before: `dt-qa.locationservice.dtsys.cc`) | QA registry, §23 |
| `mp-uber` | `mpuber-qa.daas.dtsys.cc` | QA registry, §23 |


## 19. Redis key reference

All keys are prefixed `tr-svc:`. Market `%s`, storeNo `%d`, ids `%s`. Constants live in `types/consts.go`. TTL and cap names are the `configuration.yaml` env names. Source: research 04 (tracker-service 2.0.4_dev).

| Key pattern | Const (file:line) | Type | Written | Read | TTL / cap |
|---|---|---|---|---|---|
| `tr-svc:market:%s:store:%d:algoid:%s` | `types/consts.go:17` (`OrderAlgoIdToPosOrderIdCacheKey`) | string (raw posOrderId, not JSON) | `SET ... EX`: `ordersRepository.go:228`, called from `SaveOrder` `:350-352` (only when order has OrderId) | `GET`: `ordersRepository.go:134` (`getOrderById`, Algo type), `:448` (`GetPosOrderIdsByAlgoOrderIds`, in front of an in-process cache `:444-459`) | `REDIS_DURATION_ORDER` 72h (`configuration.yaml:66`) |
| `tr-svc:market:%s:store:%d:posorder:%s` | `consts.go:16` (`OrderCacheKey`) | string, JSON `OrderEntity` | `SET`: `ordersRepository.go:340` in `SaveOrder` (`services/ordersService.go:162`), reached from `ProcessOrderEvent` `ordersService.go:127` | `GET`: `ordersRepository.go:161` (`getOrderByStoreAndId`, via `GetOrder` `:99-124`) | `REDIS_DURATION_ORDER` 72h (`configuration.yaml:66`). No cap |
| `tr-svc:market:%s:store:%d:posorder:%s:locations` | `consts.go:20` (`OrderLocationsCacheKey`) | list, JSON `LocationDto` | `RPUSH` `:418`, `LTRIM -N -1` `:419`, `EXPIRE` `:420` in `AddOrderLocationWithContext` (`ordersRepository.go:404-432`), from `locationsService.go:60-75` | `LRANGE 0 -1`: `ordersRepository.go:377` (`GetOrderLocations`; used by GET endpoints, not by the callback path) | TTL `REDIS_DURATION_ORDER_LOCATIONS` 24h (`configuration.yaml:67`). Cap `REDIS_MAX_LOCATIONS_PER_ORDER` 100 in code/yaml (`configuration.yaml:63`); deployment 360 per D-9 (not in repo) |
| `tr-svc:market:%s:store:%d:carrier:%s:currentorders` | `consts.go:22` (`CarrierCurrentOrdersCacheKey`) | set of posOrderIds | `SADD` + `EXPIRE` `carrierRepository.go:100-101` (`AddCarrierOrderIfNotExists`), from `services/ordersService.go:171-174` (goroutine) and `:336`, only when event has CarrierId and status not final. `SREM` `carrierRepository.go:118` on final (`ordersService.go:329-330`) | `SMEMBERS` `carrierRepository.go:85`, used by `locationsService.go:63-64` only when the location has no algo ids. Not used by the marketplace location path (Orders always set) | `REDIS_DURATION_CARRIER_CURRENT_ORDERS` 720h (`configuration.yaml:68`) |
| `tr-svc:fwd:%s:%d:%s` | `consts.go:34` (`ForwardingOrderSourceKey`, market, storeNo, orderId) | list, JSON `ForwardingSourceEntry` | `RPUSH` `forwardingRepository.go:59`, `LTRIM -N -1` `:60`, `EXPIRE` `:61`; keys from `Orders` `:51-53`; via `locationsService.go:52` | `LRANGE 0 -1`: `forwardingRepository.go:79` (`ReadSince`, forwarding job) | `FWDSUBSCRIPTIONEVENTSJOB_SOURCE_TTL` 15m (`configuration.yaml:40`; floor 10 x interval, `startup.go:243-245`). Cap `FWDSUBSCRIPTIONEVENTSJOB_MAX_SOURCE_ENTRIES` 3000 (`configuration.yaml:41`) |
| `tr-svc:fwd:%s:%d` | `consts.go:35` (`ForwardingStoreSourceKey`, market, storeNo) | list, same as above | same pipeline, written for every location `forwardingRepository.go:49,59-61` | `LRANGE`: `forwardingRepository.go:79` | same as above |
| `tr-svc:subscriptions:index` | `consts.go:29` (`SubscriptionsIndexKey`) | zset (score = expiry ms) | `ZREMRANGEBYSCORE` `subscriptionsRepository.go:120` and `ZREM` `:158` (pruning inside `GetAll`); `ZADD` `:78` on create | `ZRANGEBYSCORE` `:121` in `GetAll`, called by `subscriptionService.go:116,142` on every order fan-out | per-subscription retention `DBCLEANUPJOB_REMOVE_EVENT_SUBSCRIPTIONS_OLDER_THAN` 6h (`configuration.yaml:31`) |
| `tr-svc:subscription:%d` | `consts.go:28` (`SubscriptionKey`) | string, JSON subscription | `SET` `subscriptionsRepository.go:77` (on create, not in callback path) | `GET` `:101`; batch read by id `:136` inside `GetAll` | retention, `configuration.yaml:31` |

`REDIS_MAX_LOCATIONS_PER_ORDER` is 100 in code, YAML and the QA deployment; the spec's 360 is a target for DAAS-639 and is not in the repo (configuration.yaml:63; ordersRepository.go:419).

### Keys the callback and location path does not write

- `tr-svc:market:%s:customerphone:%s` (consts.go:19): `ToOrderEventFromMarketplaceCallback` sets no customer phone, so `setOrderCustomerPhone` is skipped (converters/dbEntityConverter.go:160-166, ordersRepository.go:359).
- `tr-svc:market:%s:store:%d:externalid:%s` (consts.go:18): the event sets no `ExtOrderId`, so `setOrderExternalId` is skipped (ordersRepository.go:354).
- `tr-svc:fwd:lock` (consts.go:36): job lock only (forwardingRepository.go:102 SET NX, release Lua :28, :106).

### Market fallback and Algo id to POS id

- Market fallback: if `callback.market` is blank, `market = DefaultMarket`, then `helpers.ValidateMarket`; failure is `ErrInvalidMessage` (DLQ). `DEFAULT_MARKET` has no default (configuration.yaml:8); boot warns when unset and errors only for an invalid value (configuration/startup.go:211-217). The forwarding job also falls back to `DefaultMarket` (jobs/fwdSubscriptionEventsJob.go:245). (handlers/marketplaceCallbackHandler.go:79-86)
- Store: a numeric `storeNo` is used as is, otherwise the alphanumeric id is mapped by `GetStoreNoFromAlphaNumeric` (handlers/marketplaceCallbackHandler.go:100-116).
- Status path: `marketplaceCallbacksService.go:56` calls `GetOrder(OrderId=algoId)`; `ordersRepository.go:110-112` runs `getOrderById(..., Algo)`: `GET ...:algoid:{algoId}` (:134) gives the posOrderId, then `GET ...:posorder:{posOrderId}` (:147-161). A miss gives nil and the event goes to subscribers only (marketplaceCallbacksService.go:70-73).
- Location path: `locationsService.go:66` calls `GetPosOrderIdsByAlgoOrderIds` (ordersRepository.go:434-462): an in-process cache first (:444-447, expiry `REDIS_DURATION_ORDER_LOCATIONS`), then `GET ...:algoid:{algoId}` (:448); unresolved ids are skipped (:450-452); each resolved posOrderId gets `AddOrderLocation` (locationsService.go:71).


## 20. Real status chain and enums

Chain: Uber Direct webhook or status, daas-uber `parseStatus`, marketplace callback to the gateway, tracker-service `ProcessCallback`, order event, mp-uber `processOrderEvents`, Uber Eats. Source: research 07 (daas-uber 11.5.7_dev, tracker-service 2.0.4_dev, mp-uber 0.1.9_2_dev, algo 2.65.0_ALGO-2532). Path key: DU daas-uber, TS tracker-service, MP mp-uber, AL algo.

### 20.1 Step A: Uber Direct status to daas-uber

| Uber `status` | `courier_imminent` | targetStatus (name = int) | courierStatus (name = int) | Location sent to tracker? | Notes |
|---|---|---|---|---|---|
| `pending` | any | `Scheduled` = 1 | `Unassigned` = 0 | no (status callback) | `DU:delivery.go:416-417` |
| `pickup` | false | `Assigned` = 2 | `EnrouteToPickup` = 2 | no | `DU:delivery.go:418-422` |
| `pickup` | true | `Assigned` = 2 | `ArrivedAtStore` = 3 | no |  |
| `pickup_complete` | any | `Assigned` = 2 | `ArrivedAtStore` = 3 | no | `DU:delivery.go:434-435` |
| `dropoff` | false | `PickedUp` = 3 | `EnrouteToDropoff` = 5 | no | `DU:delivery.go:423-427` |
| `dropoff` | true | `PickedUp` = 3 | `Nearby` = 8 | no |  |
| `delivered` | any | `Delivered` = 4 | `DroppedOff` = 7 | no | `DU:delivery.go:428-429` |
| `canceled` | any | `Cancelled` = 5 | `Unassigned` = 0 | no | `DU:delivery.go:430-431` |
| `returned` | any | `CouldNotDeliver` = 6 | `Unassigned` = 0 | no | `DU:delivery.go:432-433` |
| any other | any | `Waiting` = 0 | `Unassigned` = 0 | no | `DU:delivery.go:439` |
| (webhook kind `event.courier_update`) | n/a | none (`OrderStatusUpdate` nil) | none | yes, `locationUpdate{lat,lng}` (top-level webhook location, else `data.courier.location`) | `DU:internal/server/marketplace.go:86-115` |

`CourierLogin` (7) is never produced from an Uber status. Uber status strings are matched exactly (`switch uberStatus`, DU:`internal/providers/uber/delivery.go:415`); the only input besides `status` is `courier_imminent` (`internal/providers/uber/types.go:221`).

### 20.2 Step B: daas-uber to gateway to tracker-service

- daas-uber sends `MarketplaceOrderStatusUpdate{status (raw), targetStatus (int), courierStatus (int)}` only when the webhook kind is not `courier_update` (DU:`internal/server/marketplace.go:89-99`), and only for marketplace orders (`internal/server/providers.go:143-146`).
- tracker-service copies the same shape (TS:`types/marketplaceCallbacks.go:25-31`); both ints are plain JSON ints.
- `LocationUpdate` is set only on `courier_update`; tracker then calls `locations.Add` with `CarrierId = deliveryId` (TS:`services/marketplaceCallbacksService.go:77-88`).

### 20.3 Step C: tracker-service order event and stored order

| targetStatus (name = int) | Event `status` | Event `orderStatus` | Event `carrierStatus` (from courierStatus) | Stored order `Status` | Stored carrier status (derived) |
|---|---|---|---|---|---|
| `Waiting` = 0 | `Waiting` | current, else omitted | from courier int | unchanged | derived from stored order status |
| `Scheduled` = 1 | `Scheduled` | current, else omitted | from courier int | unchanged | derived |
| `Assigned` = 2 | `Assigned` | current, else omitted | from courier int | unchanged | derived |
| `PickedUp` = 3 | `Dispatched` | `5` (Enroute) | from courier int | `Enroute` | `EnrouteToPickup` |
| `Delivered` = 4 | `Delivered` | `6` | from courier int | `Delivered` | `DroppedOff` |
| `Cancelled` = 5 | `DeliveryCancelled` | current, else omitted | from courier int | unchanged | derived |
| `CouldNotDeliver` = 6 | `CouldNotDeliver` | current, else omitted | from courier int | unchanged | derived |
| `CourierLogin` = 7 | no event (ignored) | – | – | – | – |

Sources: TS:`types/marketplaceCallbacks.go:52-88`; `converters/dbEntityConverter.go:150-166`; `services/marketplaceCallbacksService.go:49-75`, `:92-100`; `converters/dbEntityConverter.go:115-145`, `:390-406`. Courier int to tracker carrier string (`types/marketplaceCallbacks.go:94-112`): 0 `Unassigned`, 1 `Accepted`, 2 `EnrouteToPickup`, 3 `ArrivedAtStore`, 4 `CarrierPickedUp`, 5 `EnrouteToDropoff`, 6 `ArrivedToCustomer`, 7 `DroppedOff`, 8 `Nearby`; out of range gives an empty string. The Stored carrier status column is derived from the stored order status, not from the courier (see the §7 note).

### 20.4 Step D: tracker-service event to mp-uber

`processOrderEvents` (MP:`services/uberMpEventsService.go:276-315`): event `status` `delivered` gives `Delivered`; `dispatched` gives `Enroute`; otherwise `AllOrderStatuses[event.orderStatus]`. The event is acted on only if the result is in `triggerUpdateOn` (configuration.yaml:27, default `EnRoute,Delivered`, case-insensitive; QA `Enroute,Delivered`).

| Tracker event `status` | Tracker `orderStatus` | mp-uber status | In trigger list? | Marketplace target (`ToMarketPlaceDeliveryStatus`) | mp-uber internal status |
|---|---|---|---|---|---|
| `Dispatched` | 5 | `Enroute` | yes | `started` | `started` |
| `Delivered` | 6 | `Delivered` | yes | `delivered` | `delivered` |
| `Scheduled` / `Assigned` / `Waiting` / `DeliveryCancelled` / `CouldNotDeliver` | current (5 → Enroute, 6 → Delivered) or omitted (→ `NewOrderMakeline`, not triggered) | per stored status | only if stored is Enroute or Delivered | `started` / `delivered` | `started` / `delivered` |

| `orderStatus` int | mp-uber internal status |
|---|---|
| 0 | NewOrderMakeline |
| 1 | OnShelf |
| 2 | InOven |
| 3 | Packing |
| 4 | ReadyForDelivery |
| 5 | Enroute |
| 6 | Delivered |
| 7 | CustomerNotHome |
| 8 | CancelledOnDispatch |
| 98 | CancelledByRefund |
| 99 | CancelledOrder |

`isFinal` (answer 205) holds for `Delivered`, `CancelledByRefund`, `CancelledOnDispatch`, `CancelledOrder`, `CustomerNotHome` (MP:`helpers/helpers.go:265-271`); `Dispatched` is not final. Algo status to Uber status (`ToMarketPlaceDeliveryStatus`, types/consts.go:64-73): `NewOrderMakeline`, `OnShelf`, `InOven`, `Packing`, `ReadyForDelivery`, `Enroute` give `started`; `Delivered` gives `delivered`; anything else is an error (row `error`, HTTP 400). Internal row status (types/consts.go:131-135): `received`, `subscribed`, `started`, `delivered`, `error`.

### 20.5 Step E and O-1: mp-uber to Uber Eats

- Call `POST eats/orders/{aggOrderId}/restaurantdelivery/status` with `{"status": "started" or "delivered"}` (MP:`types/consts.go:15`, `services/uberMpEventsService.go:350-352`), deduplicated per `aggOrderId` and status for 30 min (:341-347). Locations go to Uber only when the row is `started` (:160-179) through `eats/byoc/restaurants/orders/event/location`.
- O-1 from code: `dropoff` gives `PickedUp` (3) (DU `delivery.go:423-427`), then order `Enroute` / event `Dispatched` (TS `types/marketplaceCallbacks.go:57-58`, `:77-78`), then `Enroute` in mp-uber (MP `services/uberMpEventsService.go:283-284`), which is in `triggerUpdateOn` and becomes `started` (`types/consts.go:64-67`). `pickup`, `pickup_complete` and `Assigned` do not trigger `started`.
- Config dependency: `started` needs `EnRoute` in `triggerUpdateOn`. The YAML default has it (configuration.yaml:27, `configuration-dev.yaml:27`); the code fallback `NewOrderMakeline,Delivered` (configuration/startup.go:159-161) does not and applies only when the value is empty.
- Algo's own EnRoute on pickup (AL `AggregatorsEvents.go:574-605`, `OrderPickedUp` :141-146) is the same moment but only when `autoDispatch == 1` and the order status is already 4 or more; whether it reaches mp-uber through `/order/status` is not traced and no Algo sender exists on this ref (see A-3).

### 20.6 Enums (verbatim from the deployed refs)

daas-uber `DeliveryStatus`, internal/providers/provider.go:37-54 (Waiting=0, Scheduled=1, Assigned=2, PickedUp=3, Delivered=4, Cancelled=5, CouldNotDeliver=6, CourierLogin=7):

```go
const (
	// Waiting for acceptance or location = 0
	Waiting DeliveryStatus = iota
	// Scheduled - order is scheduled
	Scheduled
	// Assigned - order is assigned
	Assigned
	// PickedUp - order is picked up
	PickedUp
	// Delivered - order is delivered
	Delivered
	// Cancelled = order is cancelled
	Cancelled
	// CouldNotDeliver - order could not be delivered
	CouldNotDeliver
	// CourierLogin - handle special login (offline/online) from stuart
	CourierLogin
)
```

daas-uber `CourierStatus`, internal/providers/provider.go:56-75 (Unassigned=0, Accepted=1, EnrouteToPickup=2, ArrivedAtStore=3, CourierPickedUp=4, EnrouteToDropoff=5, ArrivedToCustomer=6, DroppedOff=7, Nearby=8):

```go
const (
	// Unassigned courier
	Unassigned CourierStatus = iota
	// Accepted - courier accepted
	Accepted
	// EnrouteToPickup - courier on the way to store
	EnrouteToPickup
	// ArrivedAtStore - courier arrived to store
	ArrivedAtStore
	// CourierPickedUp - courier picked up the order
	CourierPickedUp
	// EnrouteToDropoff - on the way to customer
	EnrouteToDropoff
	// ArrivedToCustomer - courier arrived to costumer
	ArrivedToCustomer
	// DroppedOff - courier dropped off the order
	DroppedOff
	// Nearby - courier nearby dropoff
	Nearby
)
```

daas-uber `parseStatus`, internal/providers/uber/delivery.go:414-440 (verbatim; the commented-out line 436 is removed):

```go
func parseStatus(uberStatus string, courierImminent bool) (deliveryStatus providers.DeliveryStatus, courierStatus providers.CourierStatus) {
	switch uberStatus {
	case "pending":
		return providers.Scheduled, providers.Unassigned
	case "pickup":
		if courierImminent {
			return providers.Assigned, providers.ArrivedAtStore
		}
		return providers.Assigned, providers.EnrouteToPickup
	case "dropoff":
		if courierImminent {
			return providers.PickedUp, providers.Nearby
		}
		return providers.PickedUp, providers.EnrouteToDropoff
	case "delivered":
		return providers.Delivered, providers.DroppedOff
	case "canceled":
		return providers.Cancelled, providers.Unassigned
	case "returned":
		return providers.CouldNotDeliver, providers.Unassigned
	case "pickup_complete":
		return providers.Assigned, providers.ArrivedAtStore
	}

	return providers.Waiting, providers.Unassigned
}
```

tracker-service `DaasDeliveryStatus`, types/marketplaceCallbacks.go:41-50 (same integers as daas-uber):

```go
const (
	DaasDeliveryWaiting DaasDeliveryStatus = iota
	DaasDeliveryScheduled
	DaasDeliveryAssigned
	DaasDeliveryPickedUp
	DaasDeliveryDelivered
	DaasDeliveryCancelled
	DaasDeliveryCouldNotDeliver
	DaasDeliveryCourierLogin
)
```

tracker-service courier list, types/marketplaceCallbacks.go:94-103:

```go
var daasCourierStatuses = []CarrierStatus{
	CarrierUnassigned,
	CarrierAccepted,
	CarrierEnrouteToPickup,
	CarrierArrivedAtStore,
	CarrierPickedUp,
	CarrierEnrouteToDropoff,
	CarrierArrivedToCustomer,
	CarrierDroppedOff,
	CarrierNearby,
}
```

tracker-service `OrderStatus`, types/consts.go:141-155 (ints from `ToInt`, types/consts.go:71-97: New=0, OnShelf=1, InOven=2, Packing=3, ReadyForDelivery=4, Enroute=5, Delivered=6, CustomerNotHome=7, CancelledOnDispatch=8, CancelledByRefund=98, CancelledOrder=99):

```go
type OrderStatus string

const (
	OrderStatusNew                 OrderStatus = "New"
	OrderStatusOnShelf             OrderStatus = "OnShelf"
	OrderStatusInOven              OrderStatus = "InOven"
	OrderStatusPacking             OrderStatus = "Packing"
	OrderStatusReadyForDelivery    OrderStatus = "ReadyForDelivery"
	OrderStatusEnRoute             OrderStatus = "Enroute"
	OrderStatusDelivered           OrderStatus = "Delivered"
	OrderStatusCustomerNotHome     OrderStatus = "CustomerNotHome"
	OrderStatusCancelledOnDispatch OrderStatus = "CancelledOnDispatch"
	OrderStatusCancelledByRefund   OrderStatus = "CancelledByRefund"
	OrderStatusCancelledOrder      OrderStatus = "CancelledOrder"
)
```

tracker-service `CarrierStatus`, types/consts.go:164-190 (line 176 spells `"CarrierPickedUp"`, not `"PickedUp"`):

```go
type CarrierStatus string

const (
	// Unassigned courier
	CarrierUnassigned CarrierStatus = "Unassigned"
	// Accepted - courier accepted
	CarrierAccepted CarrierStatus = "Accepted"
	// EnrouteToPickup - courier on the way to store
	CarrierEnrouteToPickup CarrierStatus = "EnrouteToPickup"
	// ArrivedAtStore - courier arrived to store
	CarrierArrivedAtStore CarrierStatus = "ArrivedAtStore"
	// CarrierPickedUp - courier picked up the order
	CarrierPickedUp CarrierStatus = "CarrierPickedUp"
	// EnrouteToDropoff - on the way to customer
	CarrierEnrouteToDropoff CarrierStatus = "EnrouteToDropoff"
	// ArrivedToCustomer - courier arrived to customer
	CarrierArrivedToCustomer CarrierStatus = "ArrivedToCustomer"
	// DroppedOff - courier dropped off the order
	CarrierDroppedOff CarrierStatus = "DroppedOff"
	// Nearby - courier nearby dropoff
	CarrierNearby CarrierStatus = "Nearby"
	// Special statuses used internally
	CarrierInternalStatusRemove CarrierStatus = "remove"
	CarrierStatusInStore        CarrierStatus = "inStore"
	CarrierStatusOutOfStore     CarrierStatus = "outOfStore"
)
```

mp-uber `MarketPlaceOrderStatus`, types/consts.go:42-48:

```go
type MarketPlaceOrderStatus string

const (
	MarketPlaceOrderStatusStarted   MarketPlaceOrderStatus = "started"
	MarketPlaceOrderStatusArriving  MarketPlaceOrderStatus = "arriving"
	MarketPlaceOrderStatusDelivered MarketPlaceOrderStatus = "delivered"
)
```

mp-uber `ToMarketPlaceDeliveryStatus`, types/consts.go:64-73:

```go
func ToMarketPlaceDeliveryStatus(status OrderStatus) (MarketPlaceOrderStatus, error) {
	switch status {
	case OrderStatusNewOrderMakeline, OrderStatusOnShelf, OrderStatusInOven, OrderStatusPacking, OrderStatusReadyForDelivery, OrderStatusEnroute:
		return MarketPlaceOrderStatusStarted, nil
	case OrderStatusDelivered:
		return MarketPlaceOrderStatusDelivered, nil
	default:
		return MarketPlaceOrderStatusStarted, fmt.Errorf("unknown order status: %s", status)
	}
}
```

mp-uber `OrderStatus`, types/consts.go:75-87:

```go
const (
	OrderStatusNewOrderMakeline    OrderStatus = "NewOrderMakeline"
	OrderStatusOnShelf             OrderStatus = "OnShelf"
	OrderStatusInOven              OrderStatus = "InOven"
	OrderStatusPacking             OrderStatus = "Packing"
	OrderStatusReadyForDelivery    OrderStatus = "ReadyForDelivery"
	OrderStatusEnroute             OrderStatus = "Enroute"
	OrderStatusDelivered           OrderStatus = "Delivered"
	OrderStatusCustomerNotHome     OrderStatus = "CustomerNotHome"
	OrderStatusCancelledOnDispatch OrderStatus = "CancelledOnDispatch"
	OrderStatusCancelledByRefund   OrderStatus = "CancelledByRefund"
	OrderStatusCancelledOrder      OrderStatus = "CancelledOrder"
)
```

mp-uber `OrderStatusInt`, types/consts.go:112-126:

```go
type OrderStatusInt int

const (
	OrderStatusIntNew                 OrderStatusInt = 0  // New (makeline)
	OrderStatusIntOnShelf             OrderStatusInt = 1  // On shelf
	OrderStatusIntInOven              OrderStatusInt = 2  // In oven
	OrderStatusIntPacking             OrderStatusInt = 3  // Packing
	OrderStatusIntReadyForDelivery    OrderStatusInt = 4  // Ready for delivery (assigned until 'out of store' event)
	OrderStatusIntEnRoute             OrderStatusInt = 5  // Enroute
	OrderStatusIntDelivered           OrderStatusInt = 6  // Delivered
	OrderStatusIntCustomerNotHome     OrderStatusInt = 7  // Customer not home (sent from device)
	OrderStatusIntCancelledOnDispatch OrderStatusInt = 8  // Cancelled on dispatch
	OrderStatusIntCancelledByRefund   OrderStatusInt = 98 // Cancelled by refund
	OrderStatusIntCancelledOrder      OrderStatusInt = 99 // Cancelled order
)
```

mp-uber `InternalOrderStatus`, types/consts.go:128-136:

```go
type InternalOrderStatus string

const (
	InternalOrderStatusReceived   InternalOrderStatus = "received"   // when we have received order details from Algo
	InternalOrderStatusSubscribed InternalOrderStatus = "subscribed" // when we have subscribed to the order events
	InternalOrderStatusStarted    InternalOrderStatus = "started"    // when the order is started in the marketplace
	InternalOrderStatusDelivered  InternalOrderStatus = "delivered"  // when the order is delivered in the marketplace
	InternalOrderStatusError      InternalOrderStatus = "error"      // when there is an error in the order status update
)
```

algo `DeliveryStatus`, AggregatorsAPI.go:93-110 (same integers):

```go
const (
	// Waiting for acceptance = 0
	Waiting DeliveryStatus = iota
	// Scheduled - order is scheduled
	Scheduled
	// Assigned - order is assigned
	Assigned // 2
	// PickedUp - order is picked up
	PickedUp // 3
	// Delivered - order is delivered
	Delivered // 4
	// Cancelled = order is cancelled
	Cancelled // 5
	// CouldNotDeliver - order could not be delivered
	CouldNotDeliver // 6
	// CourierLogin - check in/out
	CourierLogin // 7
)
```

algo `CourierStatus`, AggregatorsAPI.go:130-149 (same integers):

```go
const (
	// Unassigned courier
	Unassigned CourierStatus = iota
	// Accepted - courier accepted
	Accepted // 1
	// EnrouteToPickup - courier on the way to store
	EnrouteToPickup
	// ArrivedAtStore - courier arrived to store
	ArrivedAtStore // 3
	// CourierPickedUp - courier picked up the order
	CourierPickedUp
	// EnrouteToDropoff - on the way to customer
	EnrouteToDropoff // 5
	// ArrivedToCustomer - courier arrived to customer
	ArrivedToCustomer
	// DroppedOff - courier dropped off the order
	DroppedOff
	// Nearby - courier nearby dropoff
	Nearby
)
```


## 21. Verification log

Every statement of v7 was checked against the code deployed in QA (algo 2.65.0_ALGO-2532, daas-uber 11.5.7_dev, daas-gateway 1.6.0_dev, tracker-service 2.0.4_dev, mp-uber 0.1.9_2_dev) with `git show` and `git grep`, without a checkout. Research files 01 to 10 hold the evidence.

### 21.1 Counts per research file

| Research | v7 sections | Rows | CONFIRMED | MISMATCH | Other |
|---|---|---|---|---|---|
| 01 order status | §6.1, D-1, D-2, A-1, A-3 | 21 | 15 (one with a delay) | 6 | 0 |
| 02 create delivery | §6.2, D-3, D-4, D-5, A-2, §8.1.1-3 | 22 | 20 | 1 (citation only) | 1 not verifiable (row 22) |
| 03 marketplace callback | §6.3, D-6, D-7, D-8, D-10, D-11, §8.1.4-5, §8.2.1-3 | 41 | 38 | 3 | 0 |
| 04 redis | §6.4, D-9, §8.3.3-4, §10 | 22 | 19 | 1 | 2 unverified (deployment value 360, rows 7 and 19) |
| 05 tracker push | D-12, D-13, §5 steps 1 and 7, §6.5, §8.3, §8.4 | 42 | 37 | 4 | 1 unverified (in-memory dedup) |
| 06 Uber Eats | §6.6, D-16, §8.4, §9 | 15 | 13 | 2 | 0 |
| 07 status mapping | §7, D-13, O-1 | 4 header + 8 + 8 + 9 | all except the rows below | 2 (M1, M2) | 1 not verifiable (~20 s webhook interval) |
| 08 references | §5, §8, §9, §15 file:line refs (51 rows) and done claims (22), commits (7) | 51 + 22 + 7 | refs: CONFIRMED 45, MISSING 2, WRONG-LINE 4; done claims: DONE 21; commits: 7 of 7 in the deployed tags | refs: WRONG-LINE and MISSING as listed | done claims: PARTIAL 1 (§8.4.3) |
| 09 inventory | routes and config | routes: daas-uber 20, gateway 24, tracker-service 17, mp-uber 7; config rows: 29, 33, 53, 34 | not a verification | 0 | 0 |
| 10 history and status | v6 vs v7, index.html vs SPEC, Jira | dropped from v6: silent 33, superseded 16, changed 23 | n/a | 6 index-vs-SPEC contradictions (X1-X6), 2 outdated status claims (X7, X8), Jira J1-J8 | J9, J10 not verified |

### 21.2 Corrected file:line references

First the references that are wrong in v7 or in index.html, then the full table of research 08 (spec ref, status, correct ref).

| Spec ref | Wrong | Correct | Evidence |
|---|---|---|---|
| §5 step 1, `uberMpEventsService.go` trigger list and subscribe flag | `:41-43` | `:43-44` (`:41-42` is the `ParseOrderStatus` normalization) | mp-uber services/uberMpEventsService.go:41-44 |
| §5 step 1, tracker subscription block | `:97-114` | `:102-120` | mp-uber services/uberMpEventsService.go:102-120 |
| §5 step 7, `ProcessOrderEventsBatch` | `:119-175` | `:124-181` | mp-uber services/uberMpEventsService.go:124-181 |
| §5 step 7, location batches gate | `:162-174` | `:167-175` (status read `:160`) | mp-uber services/uberMpEventsService.go:160-175 |
| §6.1 and A-1, `algo AggregatorsAPI.go` status `orderId` | `:2790` | `:3423` | algo AggregatorsAPI.go:3423 |
| §6.2 and A-2, `algo AggregatorsAPI.go` `targets[].details.id` | `:661` | `:767` | algo AggregatorsAPI.go:767 |
| §8.1.4 `Uber()` path | `providers.go:143-146` | `internal/server/providers.go:143-146` | daas-uber internal/server/providers.go:52, :143-146 |
| §7 header, `delivery.go` status mapping | `:414-439` | `:414-440` | daas-uber internal/providers/uber/delivery.go:414-440 |
| index.html step 1, dedup | `uberMpEventsService.go:317-339` | `:340-347` | mp-uber services/uberMpEventsService.go:340-347 |
| index.html step 1, trigger list | `mp-uber configuration.yaml:22` | `configuration.yaml:27` | mp-uber configuration.yaml:27 |
| index.html step 1, auto-EnRoute | `AggregatorsEvents.go:614` | `AggregatorsEvents.go:574-605` | algo AggregatorsEvents.go:574-605 |
| index.html step 1, retries | `:3289` | `AggregatorsAPI.go:3506-3567` | algo AggregatorsAPI.go:3506-3567 |

| Spec ref | Status | Correct ref | Note |
|---|---|---|---|
| §5 step 1 (SPEC l.94): `uberMpEventsService.go:41-43` | WRONG-LINE | mp-uber `services/uberMpEventsService.go:43-44` | :41-42 is the ParseOrderStatus normalization added in 0.1.9_2_dev. The `shouldSubscribeToTrackerV2` flag is :44. |
| §5 step 1 (SPEC l.94): `:97-114` (tracker subscription) | WRONG-LINE | mp-uber `services/uberMpEventsService.go:102-120` | Subscribe block is :102-120 (cache check :106, Subscribe :107, SetWithExpiration :115). :97-101 is the non-subscription else branch. |
| §5 step 7 (SPEC l.102): `uberMpEventsService.go:119-175` (`ProcessOrderEventsBatch`) | WRONG-LINE | mp-uber `services/uberMpEventsService.go:124-181` | Function starts :124, closes :181. Line 119 is inside `UpdateOrderStatus`. |
| §5 step 7 (SPEC l.102): `:162-174` (location batches) | WRONG-LINE | mp-uber `services/uberMpEventsService.go:167-175` | Branch `currentInternalOrderStatus == Started` with goroutine is :167-175. :160-166 is the GetOrderStatus read and its error path. |
| §8 preamble A-2: `AggregatorsAPI.go:661` | MISSING | none | No `AggregatorsAPI.go` in any of the four repos at the deployed refs (`ls-tree -r`). Algo-side reference, not checkable here. |
| §8 preamble A-2: `AggregatorsAPI.go:2790` | MISSING | none | Same as above. |
| §8 preamble A-1: `/order/switch` answers 404 | CONFIRMED | mp-uber `controllers/marketplaceOrderEventsController.go:70-74` | Returns 404 with `types.NewErrorResponse` when `SwitchToUberDeliveryEnabled` is false. Gateway still routes it at `api/handler/router.go:155`. |
| §8.1.1 `parsers.Order` (V1) | CONFIRMED | daas-uber `internal/server/parsers/v1.go:117` (struct), `:130` (`Marketplace` field), `:60` (mapped in `V1.ToAlgoDelivery`, func at :35) |  |
| §8.1.1 `algo.Delivery` (V2) | CONFIRMED | daas-uber `internal/algo/deliveries.go:25` (field); `internal/server/parsers/v2.go:49` (`normalizeMarketplace` in `V2.ToAlgoDelivery`, func at :28) |  |
| §8.1.2 `migrations/19.add_marketplace_to_algo_discovery.sql` | CONFIRMED | daas-uber `migrations/19.add_marketplace_to_algo_discovery.sql` | Content: `ALTER TABLE algo_discovery ADD COLUMN IF NOT EXISTS marketplace TEXT;` (matches "text NULL only"). |
| §8.1.3 `OrderRequest` carries column | CONFIRMED | daas-uber `internal/algo/repository.go:35` |  |
| §8.1.3 `SaveOnce` (insert + COALESCE) | CONFIRMED | daas-uber `internal/algo/repository.go:62`; COALESCE `:77` |  |
| §8.1.3 `GetOrderByAggID` | CONFIRMED | daas-uber `internal/algo/repository.go:115`; SELECT includes `marketplace` `:116` |  |
| §8.1.3 `getAlgoDeliveryByAggID` DB fallback | CONFIRMED | daas-uber `internal/server/providers.go:170`; `marketplace` copied `:190` |  |
| §8.1.4 `internal/server/marketplace.go` | CONFIRMED | daas-uber `internal/server/marketplace.go` | Exists (new file, 198 lines added in 11.5.7 vs 11.5.6). |
| §8.1.4 `newMarketplaceCallback` | CONFIRMED | daas-uber `internal/server/marketplace.go:68` |  |
| §8.1.4 `SendMarketplaceCallback` | CONFIRMED | daas-uber `internal/server/marketplace.go:119` |  |
| §8.1.4 path `{CallbackURL}/marketplace/{marketplace}/callbacks` | CONFIRMED | daas-uber `internal/server/marketplace.go:20` (`marketplace/%s/callbacks`), `:168-173` |  |
| §8.1.4 3 attempts / 3 s timeout / 1 s apart / no retry on 4xx | CONFIRMED | daas-uber `marketplace.go:21` (attempts=3), `:22` (timeout 3 s), `:28` (wait 1 s), `:194` (retry only for non-4xx) |  |
| §8.1.4 `Uber()` (`providers.go:143-146`) | CONFIRMED (path corrected) | daas-uber `internal/server/providers.go:143-146`; `Uber()` def `:52` | Spec path lacks `internal/server/`. |
| §8.1.4 `orderDB.IsMarketplace()` | CONFIRMED | daas-uber `internal/server/providers.go:143`; def `internal/algo/repository.go:39` |  |
| §8.1.5 `courierLocation` | CONFIRMED | daas-uber `internal/server/marketplace.go:107` (def), `:101` (call, only in location branch) |  |
| §8.1.5 `Delivery.Parse` unchanged | CONFIRMED | daas-uber `internal/providers/uber/delivery.go:367` | `git diff 11.5.6_dev 11.5.7_dev -- internal/providers/uber/delivery.go` is empty. |
| §8.2.1 `api/handler/marketplaceCallback.go` / `HandleMarketplaceCallback` | CONFIRMED | daas-gateway `api/handler/marketplaceCallback.go:63`; `resolveMarketplaceMarket` `:57`; `queue.SendMessage` `:134` (15 min `:134`); 202 `:163` |  |
| §8.2.2 `POST /marketplace/{provider}/callbacks` next to `/mp/{provider}` | CONFIRMED | daas-gateway `api/handler/router.go:162-166` (route); `/mp/{provider}` group `:149-161` | No auth middleware on the route. |
| §8.2.3 `MarketplaceCallbacksQueue` in `config/queue.go` | CONFIRMED | daas-gateway `config/queue.go:13` (default `marketplace_callbacks`) | Env prefix mapping is `env.Provider(envPrefix+"_")` at `config/config.go:57`; the exact env var name was not traced. |
| §8.2.4 `/mp/{provider}/order/switch` stays in `MakeMpProxyHandler` | CONFIRMED | daas-gateway `api/handler/router.go:155` (`MakeMpProxyHandler` at `:148`) |  |
| §8.2.5 `aggProvider` already on master | CONFIRMED | daas-gateway `pkg/proxy_helper/delivery.go:25` on both `master` and `1.6.0_dev` | Spec gives no path. `master` is an ancestor of `1.6.0_dev`. |
| §8.2.6 handler tests with httptest and buffered channel | CONFIRMED | daas-gateway `api/handler/marketplaceCallback_test.go:16-18` (httptest, `make(chan queue.Message, 1)`), `:63`, `:87`, `:120` |  |
| §8.3.1 `consumers/marketplaceCallbacks.go` (amqp091-go) | CONFIRMED | tracker-service `consumers/marketplaceCallbacks.go:17` |  |
| §8.3.1 declares queue with gateway args + DLQ | CONFIRMED | tracker-service `consumers/marketplaceCallbacks.go:99-103` (queue, `x-dead-letter-*`), `:131` (DLQ `<queue>_DLQ`) | Gateway uses the same `x-dead-letter-exchange ""` and `routingKey+"_DLQ"` (daas-gateway `pkg/queue/client.go:266-269`). |
| §8.3.1 `subscribeToRabbitMQ` in `configuration/startup.go` behind `RABBITMQ_ENABLED` | CONFIRMED | tracker-service `configuration/startup.go:121` (call), `:412` (def), `:415` (`!rabbitMQ.Enabled` gate) |  |
| §8.3.1 prefetch 50 / 8 workers / reconnect 5 s to 1 min | CONFIRMED | tracker-service `consumers/marketplaceCallbacks.go:22-23` (5 s, 1 min); `configuration/startup.go:270-275` (50, 8) |  |
| §8.3.1 invalid rejected without requeue | CONFIRMED | tracker-service `consumers/marketplaceCallbacks.go:180-183` (`Nack(false,false)`) |  |
| §8.3.2 `handlers/marketplaceCallbackHandler.go` | CONFIRMED | tracker-service `handlers/marketplaceCallbackHandler.go:37` (Handle), `:47-55` (validation), `:78` (scope), `:100-113` (store) |  |
| §8.3.3 `services/marketplaceCallbacksService.go` | CONFIRMED | tracker-service `services/marketplaceCallbacksService.go:64` (ProcessOrderSubscriptions), `:74` (ProcessOrderEvent), `:87` (locations.Add) |  |
| §8.3.3 `types/marketplaceCallbacks.go` | CONFIRMED | tracker-service `types/marketplaceCallbacks.go` | Exists. |
| §8.3.3 `EntityConverter.ToOrderEventFromMarketplaceCallback` | CONFIRMED | tracker-service `converters/dbEntityConverter.go:25` (interface), `:150` (impl) |  |
| §8.3.3 `OrderEventData` gets optional `carrierStatus` | CONFIRMED | tracker-service `types/events.go:110` (`CarrierStatus ... json:"carrierStatus,omitempty"`, `OrderEventData` at `:103`) |  |
| §8.3.4 `rabbitmq` block in `configuration.yaml` | CONFIRMED | tracker-service `configuration.yaml:43-52` | All split settings present: enabled, scheme, host, username, password, vhost, marketplaceCallbacksQueue, prefetch, workers. No URL. |
| §8.3.5 Dockerfile `golang:1.25` | CONFIRMED | tracker-service `Dockerfile:1`; `go.mod` line 3 `go 1.25.0` |  |
| §8.3.6 tests | CONFIRMED | tracker-service `tests/consumers/marketplaceCallbacksConsumer_test.go`, `tests/handlers/marketplaceCallbackHandler_test.go`, `tests/services/marketplaceCallbacksService_test.go`, `tests/converters/marketplaceCallbackConverter_test.go`, `tests/configuration/configuration_test.go:129-140` (rabbit defaults) |  |
| §8.4.1 `switchToUberDeliveryEnabled` on `MarketPlaceConfiguration` | CONFIRMED | mp-uber `types/configuration.go:56` (struct), `:63` (field) |  |
| §8.4.1 `MARKETPLACE_SWITCHTOUBERDELIVERY_ENABLED` default false | CONFIRMED | mp-uber `configuration.yaml:29` (`${MARKETPLACE_SWITCHTOUBERDELIVERY_ENABLED\|\|false}`) |  |
| §8.4.1 `SwitchToUberDelivery` in controllers answers 404 | CONFIRMED | mp-uber `controllers/marketplaceOrderEventsController.go:68`, `:70-74` |  |
| §8.4.1 service, DTOs, `DELIVERY_BY_UBER` stay | CONFIRMED | mp-uber `services/uberMpEventsService.go:183`; `types/types.go:31`; `types/consts.go:58` |  |
| §8.4.2 `NewOrderMakeline` row insert kept | CONFIRMED | mp-uber `services/uberMpEventsService.go:46-48` (insert `:47`) |  |
| §8.4.2 tracker subscription kept | CONFIRMED | mp-uber `services/uberMpEventsService.go:44` (flag), `:102-120` (block) |  |
| §8.4.3 `processOrderEvents` Dispatched to started, Delivered to delivered | CONFIRMED | mp-uber `services/uberMpEventsService.go:280-287` (`dispatched` to Enroute, `delivered` to Delivered); `types/consts.go:66-69` (Enroute to started, Delivered to delivered) |  |
| §8.4.4 dedup key `marketplaceorderstatusupdate::aggorderid::{id}::status::{status}` | CONFIRMED | mp-uber `types/consts.go:36`; use `services/uberMpEventsService.go:341-347` (`SetWithExpiration` 30 min at `:347`) |  |
| §9 gateway buffers 2000 messages | CONFIRMED | daas-gateway `cmd/daas-gateway/server.go:98` (`make(chan queue.Message, 2000)`) |  |

### 21.3 Mismatches with evidence

| ID | v7 statement | Finding | Evidence |
|---|---|---|---|
| 01-4, 01-20 | §6.1, §5, A-3: Algo sends `EnRoute` / `Delivered` to `/mp/uber/order/status` | No such sender on algo 2.65.0; the only builder posts `NewOrderMakeline`. O-2 stays open | algo AggregatorsAPI.go:3418-3438, constValues.go:222 |
| 01-9 | §6.1: `orderId` at `AggregatorsAPI.go:2790` | Statement true; line is `:3423` | algo AggregatorsAPI.go:3423 |
| 01-10, 02-18 | §6.2, A-2: `details.id` at `:661`, `:2790` | Statement true; line is `:767` | algo AggregatorsAPI.go:767, :3423 |
| 01-16 | D-1, A-1: no `/order/switch` call from Algo | Algo still calls the switch in the non-`uber-daas` branch; success sets `SaleTypeCarryOut` | algo AggregatorsAPI.go:3351, :3380-3382, :3389 |
| 01-17 | A-1: `NewOrderMakeline` for every marketplace order | Not sent when the switch branch succeeds, for FlowAlerts-skipped orders, or when `CalcNextAvailableCarrier` fails | algo AggregatorsAPI.go:3351, :3303-3306, :3288-3292 |
| 03 M-1 | D-8: tracker declares both queues with the gateway's arguments | DLQ declared with nil arguments by tracker; gateway never declares it | tracker-service consumers/marketplaceCallbacks.go:131; daas-gateway pkg/queue/client.go:168 |
| 03 M-2 | D-6: every Uber Direct webhook is forwarded | Skipped for non-marketplace orders, after an Algo callback failure, and for updates without content | daas-uber internal/server/providers.go:137-145, internal/server/marketplace.go:134-138 |
| 03 M-3 | §8.1.4: POST to `{CallbackURL}/marketplace/...` | The path of `CallbackURL` is replaced | daas-uber internal/server/marketplace.go:168-175, internal/server/gatewayAPI.go:35-37 |
| 04-11 | §6.4: existing keys only, three keys listed | Also `algoid`, `carrier:...:currentorders`, `subscriptions:index` | tracker-service ordersRepository.go:350-352, services/ordersService.go:171-174, subscriptionsRepository.go:120 |
| 05 step 1 | §5 step 1 refs `:41-43`, `:97-114` | `:43-44`, `:102-120` | mp-uber services/uberMpEventsService.go:43-44, :102-120 |
| 05 step 7a | §5 step 7 `:119-175` | `:124-181` | mp-uber services/uberMpEventsService.go:124-181 |
| 05 step 7b | §5 step 7 `:162-174` | `:167-175` | mp-uber services/uberMpEventsService.go:167-175 |
| 05 §6.5 | Order-event example | Four zero-time fields and `sender`, `totalEvents`, `timeStamp`, `traceId` also sent | tracker-service types/events.go:130-145, types/dtos.go:120-128 |
| 06 V2 | §6.6 location body | Also `is_batched_order` (false); exactly one location event | mp-uber types/mpDtos.go:7-34, services/uberMpEventsService.go:237-238 |
| 06 V4 | D-16: Uber Eats only receives two calls | Switch call (flag-gated) and store-sync calls exist in the service | mp-uber services/uberMpEventsService.go:183-206, services/storesService.go:183, :265 |
| 07 M1 | §7: Uber Eats `–` for non-trigger rows | Not inert when the stored status is `Enroute` / `Delivered`; only the 30 min dedup guards | mp-uber services/uberMpEventsService.go:285-287, :341-347; tracker-service converters/dbEntityConverter.go:157-159 |
| 07 M2 | §7: `carrierStatus` is not stored | A carrier status derived from the order status is stored | tracker-service converters/dbEntityConverter.go:141-143, :390-406 |
| 08 PARTIAL | §8.4.3: no status mapping change | `ParseOrderStatus` normalization added in 0.1.9_2_dev; table unchanged | mp-uber services/uberMpEventsService.go:38-42 |
| 08 WRONG-LINE, MISSING | Refs listed in 21.2 | Four mp-uber lines wrong; two Algo refs not checkable in the four DaaS repos (checked in algo: 01-9, 01-10) | research 08, research 01 |
| 10 X1 | index §07: "no location reaches Uber Eats" | SPEC §9 says "no status or location ... through the push" | index.html row, SPEC.md §9 |
| 10 X2 | index §07: "6 retries" | Code: 5 retries, 6 sends (SPEC: "6 attempts") | algo AggregatorsAPI.go:3506-3567 |
| 10 X3 | index §01 owner "DevOps, in a separate ticket" | SPEC §12: DAAS-639 | SPEC.md §12 |
| 10 X4 | index §01: `daas-uber-uk` "has none of it" | SPEC O-8: to confirm | SPEC.md §11 O-8 |
| 10 X5 | index §06 branch `DAAS-638-remove-the-switch...` | Switch is disabled, not removed | mp-uber commit 8f18d94; SPEC.md D-1 |
| 10 X6 | index §05 "Algo → mp-uber" column, no `Waiting` row | SPEC §7 has no such column and has the `Waiting` row | SPEC.md §7 |
| 10 X7, X8 | tracker-service not deployed; "done" chips | Deployed in QA; Jira Peer Review / Dev Test | QA 2026-10-29; Jira GET 2026-10-29 |
| 10 J1-J8 | §8 headings "done", §15 assignees, §12 blocker 1, §11 O-9 | Peer Review (635-637), Dev Test (638), DAAS-639 To Do / Alexander Hodak, DAAS-634 In Progress / Sergey Shelekhov, tracker-service in QA, O-9 mismatch in QA | Jira GET 2026-10-29; QA 2026-10-29 |

### 21.4 Things found that v7 does not say

- Forwarded points stop 2 h after the subscription was created (`defaultMaxAllowedTime`); the order fan-out has no window (tracker-service jobs/fwdSubscriptionEventsJob.go:79-80, services/subscriptionService.go:114-128).
- tracker-service drops a point only when both coordinates are 0, mp-uber rejects a batch for one 0, and tracker does not retry a 4xx, so the rest of that batch is lost (tracker-service services/locationsService.go:39-41; mp-uber controllers/marketplaceOrderEventsController.go:101-107).
- mp-uber's `SqlInsertWorker` bulk insert names an `orderUID` column that no migration in the repo creates (mp-uber queries/rawRequestsData/bulkInsert.sql:1, sqlQueriesBuffer.go:50); QA shows the failure (§23).
- The gateway's `/mp/{provider}/events/locations` route has no mp-uber counterpart (daas-gateway api/handler/router.go:158).
- The `marketplace` flag is not selected by `GetOpenOrders` (daas-uber internal/algo/repository.go:131-137).
- The switch-off commit 8f18d94 is already contained in mp-uber `0.1.9_dev`; `0.1.9_2_dev` adds only `.gitignore` and the `ParseOrderStatus` normalization (research 08).


## 22. Restored from v6

Items present in v6 and absent from v7, restored verbatim and marked "(v6, not in v7)". The ids (S1 to S33) are those of research 10. Items research 10 classes as superseded (P1 to P16) are not restored, because v7 §13 records each replacement, except P5, which §13 does not record and is restored here. Items that are still in v7 with a changed value (C1 to C23) are not restored; their v7 wording stands.

### §0 header (v6 line 3-9 area)

- (v6, not in v7) [S1] "- **Status:** design agreed, ready for an implementation plan. Open items are in §11. In-scope items have an owner, and out-of-scope items are marked as such."

### §1 Summary, Order registration bullet

- (v6, not in v7) [S2] "It replaces the switch."

### §4 Decisions, D-2

- (v6, not in v7) [S3] "The switch, the proposed create-delivery call and `NewOrderMakeline` are merged into the existing `POST /mp/uber/order/status` with `orderStatus: "NewOrderMakeline"`."

### §4 Decisions, D-9

- (v6, not in v7) [S4, S5] "100 points is about 33 min at one point every 20 s, and 360 is about 2 h."
- (v6, not in v7) [S4, S5] "raised from 100, `configuration.yaml:53`"
- (v6, not in v7) [S4, S5] "The 24 h TTL matches the existing `orderLocations` setting (`configuration.yaml:55-58`)"

### §5 Flow, Step 1

- (v6, not in v7) [S6] "(`SubscribeToLocationsOnStatus = NewOrderMakeline`, `mp-uber configuration.yaml:23`, `uberMpEventsService.go:42, :99-116`)"
- (v6, not in v7) [S6] "(default `EnRoute,Delivered`, `mp-uber configuration.yaml:22`)"

### §5 Flow, Step 7

- (v6, not in v7) [S7, S8] "The subscription was created on `NewOrderMakeline` with callback `/mp/uber/order/events` and events `location` and `order` (`mp-uber services/trackerV2Service.go:30-80`)."
- (v6, not in v7) [S7, S8] "The push interval is the existing forwarding job's (`tracker-service configuration/startup.go:115-120`)."

### §6.1

- (v6, not in v7) [S9] "- `NewOrderMakeline`: row `received` + tracker subscription. It replaces the switch."

### §6.3

- (v6, not in v7) [S10] "// null on pending, delivered, canceled, returned"

### §6.5

- (v6, not in v7) [S11] "(mp-uber role `tracker`, `controllers/marketplaceOrderEventsController.go:18-23`)"
- (v6, not in v7) [S11] "(existing, `:92-98`)"

### §7 Status mapping

- (v6, not in v7) [S12, S13, S14, S15] "| Uber Direct | Location | Algo target / courier | Algo order | Algo → mp-uber | Uber Eats | Pushed to mp-uber |"
- (v6, not in v7) [S12, S13, S14, S15] "EnRoute (`AutoEnrouteOnAggPickup`)"
- (v6, not in v7) [S12, S13, S14, S15] "back to dispatch"
- (v6, not in v7) [S12, S13, S14, S15] "Uber Direct rarely sends `pickup_complete` (1 in 96 webhooks in the 2026-10-14 sample)."

### §8.1 daas-uber

- (v6, not in v7) [S16, S17, S18, S28] "Today the field is silently dropped."
- (v6, not in v7) [S16, S17, S18, S28] "(sql-migrate, `-- +migrate Up`)"
- (v6, not in v7) [S16, S17, S18, S28] "in `internal/server/gatewayAPI.go` (next to `SendCallbackUpdate`, `:25`, same `CallbackURL`)"
- (v6, not in v7) [S16, S17, S18, S28] "(`providers.go:52-145`, callback at `:136`)"
- (v6, not in v7) [S16, S17, S18, S28] "(`providers/uber/delivery.go:372-381`)"

### §8.2 daas-gateway

- (v6, not in v7) [S19, S20, S21] "modelled on `api/handler/callback.go:59-110`"
- (v6, not in v7) [S19, S20, S21] "next to the callbacks group (`:106-115`)"
- (v6, not in v7) [S19, S20, S21] "(`MakeMpProxyHandler`, `:148-160`)"
- (v6, not in v7) [S19, S20, S21] "Check what it does today first."
- (v6, not in v7) [S19, S20, S21] "(`router.go:92-103`)"

### §8.3 tracker-service

- (v6, not in v7) [S22, S23, S24, S25, S26, S28] "started from `StartUp()` (`configuration/startup.go:115`)"
- (v6, not in v7) [S22, S23, S24, S25, S26, S28] "(coordinate with O-1)"
- (v6, not in v7) [S22, S23, S24, S25, S26, S28] "(`${VAR||default}` pattern)"
- (v6, not in v7) [S22, S23, S24, S25, S26, S28] "align the Go base image (1.23.4) with `go.mod` (1.25)"
- (v6, not in v7) [S22, S23, S24, S25, S26, S28] "(testify + `redismock`)"
- (v6, not in v7) [S22, S23, S24, S25, S26, S28] "(`dataAccess/repositories/ordersRepository.go:404-431`)"

### §8.4 mp-uber

- (v6, not in v7) [S27, S28] "(`:324`, key in `types/consts.go:15-20`)"
- (v6, not in v7) [S27, S28] "(`services/uberMpEventsService.go:172-195`)"

### §10 Rollout and tests

- (v6, not in v7) [S29, S30] "Keep the switch route until ALGO-2532 is released."
- (v6, not in v7) [S29, S30] "the `algo_discovery` row has `marketplace='uber'` and the Uber Eats UUID."

### §11 Open items, O-1

- (v6, not in v7) [S31] "Options: `pickup` (courier assigned, driving to the store), `pickup_complete` (rarely sent by Uber), or `dropoff` (left the store, the same moment as Algo's EnRoute)."
- (v6, not in v7) [S31] "Asked in Slack 2026-10-22"

### §12 Blockers, blocker 2

- (v6, not in v7) [S32] "(`tracker-service/CLAUDE.md:121`)"

### §15 Tracking, Pending

- (v6, not in v7) [S33] "Sergey's GitHub username for read access to this private repo."

### §6.2 (superseded item research 10 lists as P5; v7 §13 does not record it)

- (v6, not in v7) [P5] "The gateway forwards it to daas-uber `POST /api/v1/deliveries/`."


## 23. QA state on 2026-10-29

Verified directly in QA by the operator on 2026-10-29.

### 23.1 Versions deployed

| Service | Version | Deployed by |
|---|---|---|
| daas-uber | 11.5.7 | CircleCI pipeline 830 |
| daas-gateway | 1.6.0 | CircleCI pipeline 648 |
| mp-uber | 0.1.9_2 | CircleCI pipeline 160, after the CI fix commit ec8813a "scope apt-get update to main sources in deploy job" |
| tracker-service | 2.0.4 | EKS `nonprod-global-cluster`, namespace `cloud`, Helm chart `dragontail-app`, Argo CD app `tracker-service-nonprod-global-cloud`, values in `dragontail-platform` `environments/nonprod/global/cloud-team.yaml` |

### 23.2 tracker-service in QA

- PR #571 (merged, commit 87df9b7) set `DEFAULT_MARKET=ph-uk`, `RABBITMQ_ENABLED=true`, `RABBITMQ_SCHEME=amqps`, `RABBITMQ_HOST=daas-qa.mq.dtsys.cc:443`, `RABBITMQ_VHOST=/`, `RABBITMQ_MARKETPLACE_CALLBACKS_QUEUE=marketplace_callbacks`, and `RABBITMQ_USERNAME` / `RABBITMQ_PASSWORD` from SSM `/nonprod/global/cloud/tracker-service/rabbitmq/{username,password}`.
- After the restart the log shows "Marketplace callbacks consumer connected" (host `daas-qa.mq.dtsys.cc:443`, queue `marketplace_callbacks`).
- The chart has no config checksum, so a ConfigMap change needs a manual restart.
- The QA ConfigMap still has 13 keys whose names 2.0.4 does not read, for example `JOBS_FWDSUBSCRIPTIONEVENTSJOB_INTERVAL` against the code name `FWDSUBSCRTIPTIONEVENTSJOBINTERVAL`, so forwarding runs at the 30 s default, not 1 s.
- `REDIS_MAX_LOCATIONS_PER_ORDER=100` in QA; the spec wants 360 (D-9).

### 23.3 Gateway service registry in QA

- `tracker-v2` was `dt-qa.locationservice.dtsys.cc` (the old location-service 1.1.3_3 on ECS `location-service-qa-qa`); repointed on 2026-10-29 through `PATCH /services/1/url` to `tracker-service.cloud.global.dtsys.cc`.
- `mp-uber` points to `mpuber-qa.daas.dtsys.cc`.
- The old location-service is still running; its removal waits for a check of the dt-qa Proxy push URL.

### 23.4 Blocker, bugs and limits

| Kind | Finding |
|---|---|
| BLOCKER | mp-uber QA `keys` table markets are `ph-il`, `phtest-uk`, `phtest-us`, with no `ph-uk`. The gateway stamps `market=ph-uk` on every marketplace callback, so mp-uber subscriptions (market `phtest-uk`) do not match stored callbacks (spec O-9). |
| BUG | mp-uber QA logs every 10 min `SqlInsertWorker failed to execute insert query` with `pq: column "orderuid" does not exist`. QA `rawrequestdata` columns are id, url, headers, method, payload, response, statuscode, internalerrormsg, timestamp (no orderUID); the bulk insert names it (mp-uber queries/rawRequestsData/bulkInsert.sql:1). |
| BUG | mp-uber QA logs every 10 min "Failed to get token from marketplace API" for market `ph-il` (login.uber.com). |
| LIMIT | daas-uber in QA calls Uber through `mock-api-qa.daas.dtsys.cc`, so O-3 (Uber accepts DaaS-courier status and location) cannot be verified in QA. |
| CONFIG | Algo books Uber Direct only when the store's `MarketPlaceConfiguration.daas` is `"uber-daas"`; otherwise Algo calls `/order/switch` (404 while disabled) and keeps the store driver. |

## 24. Decisions and fixes, 2026-10-29

| ID | Decision / fix | Matches code |
|---|---|---|
| D-17 | **DaaS-order statuses rely only on the callbacks path.** For orders delivered by a DaaS courier, Uber Eats `started` / `delivered` come only from daas-uber → gateway `/marketplace/uber/callbacks` → `marketplace_callbacks` → tracker-service order events (`Dispatched`, `Delivered`) → mp-uber `/mp/uber/order/events`. Algo is **not** required to send `EnRoute` / `Delivered` for these orders; A-3 no longer applies to DaaS orders. | Yes: Algo `2.65.0_ALGO-2532` sends only `NewOrderMakeline` to `/mp/uber/order/status` (`AggregatorsAPI.go:3418-3439`). |
| D-18 | **`daas: "uber-daas"` is mandatory in Algo.** Every store that takes Uber Eats marketplace orders must have `MarketPlaceConfiguration.<source>.daas = "uber-daas"`. Without it Algo takes the `/order/switch` path (404 while the switch is disabled, retried, then stays with the store driver). | Yes: `Stores.go:312-318` (`daas`), `AggregatorsAPI.go:3335-3410` (`bookUberDaas` only for `uber-daas`). |
| F-1 | **QA market aligned to `ph-uk`.** mp-uber QA DB: `keys` id 6 (`Pizza Hut:YUM! III (UK) LIMITED`, United Kingdom) and 575 `marketplacestores` rows renamed `phtest-uk` → `ph-uk`; mp-uber restarted. Subscriptions and the gateway's `market=ph-uk` now use the same market. | Yes: gateway `marketplaceCallback.go:17,57-59` stamps `ph-uk`; mp-uber resolves the market from `keys` by brand+country (`authService.go:108-119`). |
| F-2 | **Root cause of `pq: column "orderuid" does not exist`.** Commit `cdad07d` ("added new table rawrequestdata") added the insert with an `orderUID` column (`dataAccess/queries/rawRequestsData/bulkInsert.sql:1`, `sqlQueriesBuffer.go:50`, `types/types.go:168`) but the table migration (`dataAccess/queries/_migrations/1_createTables.sql:32-42`) has no such column. Every raw-request insert fails, so all outbound request logging is lost. Nothing sets (`helpers/helpers.go:312`) or reads the field. Recommended fix: drop `orderUID` from the insert, the format string and the struct; alternative: new migration `4_addRawRequestOrderUid.sql` with `ALTER TABLE $schema.rawrequestdata ADD COLUMN IF NOT EXISTS orderUID TEXT NULL;`. Owner: mp-uber (DAAS-638). | Yes (code evidence above). Not fixed yet. |
