# PRD: E-commerce Agent Shopping Assistant (Market/Bidding Topology)

## 1. Overview
Build an agentic shopping assistant that helps users discover, compare, and purchase products across multiple retailers. The system uses a market/bidding topology: retailer agents compete with offers; a coordinator orchestrator merges results with policy, constraints, and explainability. Checkout requires explicit user approval and idempotent payment/order execution.

## 2. Goals
- Find best-fit products for a user request across multiple sellers/retailers.
- Provide transparent comparison: price, ETA, return policy, seller trust, reviews.
- Maintain user trust: no hallucinated prices/stock; clear disclaimers for uncertainty.
- Support secure checkout: explicit consent + idempotent payments/orders.
- Full traceability: reconstruct "what user saw" and "why it was chosen."

## 3. Non-Goals
- No autonomous purchasing without explicit user action.
- No dynamic price guarantees unless contractually supported.
- No inventory reservation unless retailer supports it.
- No personal finance/credit recommendations beyond basic affordability constraints.

## 4. Target Users
- Shoppers who want to quickly shortlist and buy.
- Power users who value best deals and return policies.
- Users shopping with constraints (budget, size, delivery date, brand preference).

## 5. Key Use Cases
1. Product discovery: “Running shoes for wide forefoot under ₹8k.”
2. Comparison: “Show top 3 with pros/cons and delivery by Tuesday.”
3. Deal optimization: “Apply best coupons; prefer easy returns.”
4. Checkout: “Buy option #2 using UPI/card; deliver to saved address.”
5. Post-purchase: receipt, order tracking link, cancellation/return policy recap.

## 6. User Experience Requirements
### 6.1 Progress Events (SSE/WebSocket)
- Show truthful stage updates:
  - “Collecting offers from 5 sellers (3 received)…”
  - “Checking delivery dates…”
  - “Applying coupons…”
  - “Ready for checkout — awaiting your approval”
- Never show “order placed” before confirmation from order service/OMS.

### 6.2 Result Presentation
- Shortlist table/cards with:
  - final price (incl. shipping/discounts)
  - ETA range
  - return window + restocking fee if any
  - seller rating/trust indicator
  - top review summary (with source attribution if available)
- Explainability section:
  - “Chosen because: lowest total price + delivery before Tue + 30-day returns.”

### 6.3 Explicit Consent
- Checkout must be a distinct user confirmation step.
- Show final payable amount and key terms (returns, delivery, cancellations).

## 7. Functional Requirements
### 7.1 Orchestrator (Coordinator)
- Parse user intent + constraints.
- Trigger auction (fan-out) for offers.
- Merge offers with shipping, deals, reviews.
- Enforce policies and safety constraints.
- Ask for clarifications if constraints are missing (e.g., size).
- Produce final recommendation + rationale.
- Gate side-effects (payment/order) behind explicit consent.

### 7.2 Bid Manager (Time-boxed Auction)
- Broadcast Request For Proposal (RFP) to retailer agents.
- Collect bids for a fixed time window T (e.g., 2s) with max retailers N (e.g., 5).
- Accept partial results; label missing retailers.
- Normalize bid schemas into a canonical Offer object.

### 7.3 Retailer Agents
- Retrieve product matches and return structured bids:
  - product_id/sku, title, price, availability confidence, images (optional), URL, seller policy snippets
- Must include evidence fields:
  - timestamp, source, confidence scores
- Must not place orders.

### 7.4 Review/Quality Agent
- Summarize pros/cons from available review sources.
- Produce a quality score and flags (counterfeit risk, return policy risk).

### 7.5 Shipping/ETA Agent
- Compute ETA and shipping cost; optionally detect delivery constraints.

### 7.6 Coupon/Deal Agent
- Apply eligible coupons; output final price breakdown.

### 7.7 Ranking & Explainability
- Score offers with configurable weights:
  - price, ETA, return policy, trust, review score, user preferences
- Output:
  - top-K ranked offers
  - explanation payload (scores + key reasons)

### 7.8 Checkout/Payment Agent
- Create payment intent with idempotency key.
- Collect user authorization token/confirmation.
- Confirm payment; handle failure safely.
- Proceed to order placement only after successful payment authorization.

### 7.9 Order Service
- Place order with idempotency key.
- Record “user saw price X” and “charged price Y” invariants.
- Return receipt and order id.

## 8. Reliability Requirements
### 8.1 Timeouts
- Per-call timeouts:
  - retailer bid: 1–2s each
  - reviews/ETA/deals: 1–3s
  - payment: per PSP SLA
- Whole-request timebox (UX SLA): e.g., 8–12s for shortlist.

### 8.2 Retries
- Safe retries (read-only): search, ETA, coupon lookup (with backoff + jitter).
- Side-effect retries (payment/order): ONLY with idempotency keys and only on network/timeout, never on logical declines.

### 8.3 Degraded Mode
- If some retailers fail: proceed with partial shortlist + disclosure.
- If coupons fail: show pre-discount price + note “coupon service unavailable.”
- If ETA uncertain: show range + label “estimate.”

## 9. Security & Compliance
- No storing sensitive payment data; rely on PSP tokenization.
- Consent logs: user approval time, amount, and key terms displayed.
- Prevent prompt injection from retailer content; sanitize and constrain tool outputs.
- Rate-limits and abuse prevention (scraping, bot detection).

## 10. Observability (Debugging + Auditability)
### 10.1 Tracing (Required)
- trace_id per shopping session; propagate through every agent/tool call.
- Store:
  - bids received + timestamps
  - scoring inputs/outputs
  - progress events emitted
  - consent event + displayed price snapshot
  - payment intent id + order id

### 10.2 Metrics
- Auction: bids_received_count, bid_timeout_rate, time_to_first_bid, time_to_decision
- Ranking: decision_confidence, score_spread (winner vs runner-up)
- Checkout: payment_success_rate, duplicate_prevented_count, order_success_rate
- UX: time_to_first_result, abandonment_rate_before_checkout

### 10.3 Logs
- Structured logs with: trace_id, agent, stage, latency_ms, error_type.

## 11. Data Model (Canonical Offer)
Offer fields:
- offer_id, retailer_id, product_sku, title
- base_price, shipping_fee, discount, final_price
- currency
- availability_status + confidence
- eta_min/max
- return_window_days, return_fee
- seller_trust_score
- review_score + summary
- evidence: source, fetched_at, url

## 12. Acceptance Criteria
- Given a product request, system returns top 3 offers within SLA with rationale.
- No hallucinated prices: every price shown is backed by evidence fields.
- Checkout requires explicit user approval and logs consent.
- Duplicate orders/payments are prevented via idempotency.
- A complete trace reconstructs the session end-to-end.

## 13. Risks & Mitigations
- Retailer API instability → time-box auctions + degraded mode.
- Price changes between quote and checkout → refresh price at checkout + user re-confirmation if changed.
- Prompt injection from web content → strict schema validation + tool output sanitization.
- Legal/user trust issues → consent logging + transparent disclaimers.

## 14. Future Enhancements
- Personal preference learning (opt-in)
- Returns initiation assistant
- Multi-cart optimization (bundle across retailers)
- Negotiation agent (if supported by merchants)
