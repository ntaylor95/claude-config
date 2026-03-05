# Plan: Identify Sellers Who Manually Browse Rates for 30%+ of Orders

## Context

We have been building a body of analysis to identify sellers who are doing manual work that could be automated. This task extends that work to rate browsing specifically: sellers who are manually opening the rate browser for a large share of their orders are strong candidates for rate shopper automation.

The challenge: none of the `rate_browser_*` Segment tables have `fulfillment_plan_id` or `order_id`. The only way to link a rate_browser event to a specific order is through `app_session_guid` → `csw_update_shipment_configuration.fulfillment_plan_id`.

---

## Key Insight From Session Data

Inspecting session `00002e9b-dee4-f077-5ab6-fb51a7fe8f1f` reveals the real pattern:

- **13 `rate_browser_flow_start` events** (each with a unique `workflow_guid`)
- **0 `rate_browser_browse_rates_clicked` events**
- **0 `rate_browser_create_label_clicked` events**
- After nearly every `rate_browser_flow_start`, **3 CSW update events fire in rapid succession (~1 second apart)**

This means:
1. `rate_browser_flow_start` is the **only reliable signal** — `browse_rates_clicked` and `create_label_clicked` do not fire for this workflow pattern (the seller selects a rate in the browser and it auto-applies to the CSW)
2. A **simple `app_session_guid` join** would attribute ALL CSW orders in the session to rate browsing — this session has 100+ CSW events but only 13 rate browser opens. That's a massive inflation problem.
3. The right approach is **temporal proximity**: a rate_browser_flow_start followed by CSW updates within ~90 seconds = one rate-browsed order.

## Key Join Chain

```
rate_browser_flow_start
  (user_id, app_session_guid, timestamp T)
      ↓ JOIN on app_session_guid
        AND csw.timestamp BETWEEN T AND T + 90 seconds
csw_update_shipment_configuration
  (app_session_guid, fulfillment_plan_id, timestamp)
      ↓ DISTINCT fulfillment_plan_id = rate-browsed order
      ↓ user_id → seller_id via segment_app_v3.accounts
shipstation_cdc_silver.order
  (seller_id, total order count, automation rate)
```

---

## Proposed Approaches

---

### Option A — Temporal Proximity Join: Flow Start → CSW Within 90 Seconds (Recommended)

**Signal used:** `rate_browser_flow_start` with `source = 'GRID_CSW'`

**Logic:**
1. For each `rate_browser_flow_start` event, find CSW updates on the same `app_session_guid` that fired within 90 seconds after it
2. Each matched CSW update's `fulfillment_plan_id` = a rate-browsed order
3. `DISTINCT` on `fulfillment_plan_id` per seller to avoid double-counting (multiple CSW bursts per order)
4. Resolve `user_id` → `seller_id` via `accounts`
5. Count total orders per seller from `shipstation_cdc_silver.order`
6. Compute `rate_browsed_pct = rate_browsed_orders / total_orders`
7. Filter: `rate_browsed_pct >= 0.30`

```sql
-- Core join pattern
FROM `rate_browser_flow_start` rb
JOIN `csw_update_shipment_configuration` csw
  ON  csw.app_session_guid = rb.app_session_guid
  AND csw.timestamp BETWEEN rb.timestamp
                        AND TIMESTAMP_ADD(rb.timestamp, INTERVAL 90 SECOND)
WHERE rb.source = 'GRID_CSW'
```

**Why this works:** Session data shows the pattern is always: rate_browser_flow_start → 3 CSW updates within ~30-90 seconds. The 90-second window captures the CSW response to the rate browser selection without pulling in unrelated orders later in the session.

**Caveat:** If a seller has very fast back-to-back orders (two rate_browser events within 90 seconds of each other), the CSW updates for order N+1 might overlap with the window for order N. This is unlikely to be a major distortion but worth noting.

**New file:** `sql/07_rate_browser_adoption_by_seller.sql`

---

### Option B — Seller-Level Rate Browser Frequency (Simpler Denominator)

**Signal used:** `rate_browser_flow_start` events per seller vs. total CSW orders per seller

**Logic:** Rather than linking each rate_browser event to a specific order, measure at the seller level:
- `rate_browser_opens_30d` = COUNT of `rate_browser_flow_start` events per seller
- `csw_orders_30d` = COUNT DISTINCT `fulfillment_plan_id` in `csw_update_shipment_configuration` per seller
- `rate_browser_rate` = rate_browser_opens / csw_orders

**Why useful:** Faster to compute (no temporal join), good for a broad initial sweep. Slightly different question: "how many rate browser opens per order" rather than "what % of orders were rate browsed." Sellers with rate > 1.0 are opening rate browser more than once per order (rate shopping).

**New file:** Can be a STEP 2 in `sql/07_rate_browser_adoption_by_seller.sql`

---

### Option C — Cross-Reference with Existing 321-Seller Cohort

**Signal used:** Option A result joined to `seller-classification-all.md` cohort

**Logic:**
1. Run Option A across the full platform
2. Join to the existing 321 low-automation / high-growth sellers
3. Flag sellers in BOTH lists — these are the highest-priority targets: growth opportunity sellers who are ALSO confirmed rate browsers

**New file:** Adds a join step to `sql/07_rate_browser_adoption_by_seller.sql`

---

## Recommended Execution Order

1. **Start with Option A** — temporal proximity join gives the most accurate "orders rate browsed" count. Filter to `rate_browsed_pct >= 0.30`.

2. **Add Option B side-by-side** — the `rate_browser_opens / csw_orders` ratio is a useful secondary metric. Sellers with ratio > 1.0 are opening rate browser multiple times per order (strong rate shopping signal).

3. **Apply Option C** to cross-reference with the existing cohort.

---

## Files to Create

| File | Contents |
|------|----------|
| `sql/07_rate_browser_adoption_by_seller.sql` | Options A + B side by side. Per-seller rate browser adoption rate using both `flow_start` and `create_label_clicked` as numerators. Includes automation rate and order volume context. |
| `sql/08_rate_browser_funnel_by_seller.sql` | Option C. Full three-stage funnel per seller with dropoff rates. |

---

## Existing Files to Reuse

| File | What to Reuse |
|------|--------------|
| `sql/01_sellers_low_automation_high_growth.sql` | Seller universe pattern: `shipstation_cdc_silver.order`, deduplication with `ROW_NUMBER()`, automation rate calculation, order volume and growth windows |
| `sql/02_seller_manual_change_activity.sql` | `accounts` table UUID resolution pattern: `user_id` → `seller_id` |
| `manual-changes/csw-update-shipment-configuration-columns.md` | `app_session_guid` confirmed as the join key between rate_browser and CSW tables |

---

## Key Caveats to Document in Query Comments

1. **`browse_rates_clicked` and `create_label_clicked` are unreliable** — confirmed from session data that these events do not fire for a common workflow pattern. `rate_browser_flow_start` is the only reliable signal.

2. **Temporal window of 90 seconds** — based on observed session data showing CSW updates fire 30-90 seconds after rate browser open. May need tuning if the pattern differs across seller workflows.

3. **Rate browser without CSW edit is invisible** — if a seller opens rate browser but makes no CSW edit in the same session window, that event won't link to any order. These opens are noise and will be excluded, which is correct behavior.

4. **`source = 'GRID_CSW'` filter is required** — rate browser events fire from multiple contexts. Only `source = 'GRID_CSW'` is the CSW-embedded rate browser relevant to this analysis.

5. **Lookback window** — use 30 days for rate browser events (Segment tables, no partition key) and match the order denominator to the same 30-day window.
