# PART 1 — PRODUCTION CHECKLIST (E-COMMERCE AGENT SYSTEM)

### 🎯 E-commerce Goal (anchor this first)

> “Help the user discover, compare, decide, and checkout products **correctly, transparently, and safely**.”

Key constraints:

- Money involved
    
- Legal implications
    
- User trust is fragile
    
- External dependencies everywhere
    

---

## LAYER 1 — TOPOLOGY (Market / Bidding)

### ✅ Why Market / Bidding fits best

- Multiple retailers / sellers
    
- Different prices, stock, delivery times
    
- Competition improves outcome quality
    
- Natural parallelism
    

### Agents involved

- **Coordinator (Orchestrator)** – owns final decision
    
- Retailer Agents (Amazon, Flipkart, Brand site, etc.)
    
- Review Agent
    
- Coupon/Deal Agent
    
- Shipping/ETA Agent
    
- Checkout/Payment Agent
    

### Checklist

-  Coordinator is the **only one** allowed to finalize checkout
    
-  Retailer agents never place orders directly
    
-  All offers are treated as _candidates_, not truth
    

---

## LAYER 2 — FAN-OUT (Parallel Offers)

### Where fan-out happens

- Product search across retailers
    
- Coupon application
    
- Shipping ETA lookup
    
- Review summarization
    

### Checklist

-  Fan-out happens only at **safe read-only steps**
    
-  Max fan-out count defined (e.g., 5 retailers)
    
-  Each retailer response tagged with:
    
    - price
        
    - stock confidence
        
    - ETA
        
    - return policy
        

### Merge rule (MANDATORY)

> **Time-boxed scoring merge**

Example scoring:

```
score = w1*price + w2*delivery_speed + w3*return_policy + w4*trust
```

Checklist:

-  Timebox bids (e.g., 2 seconds)
    
-  Late bids ignored or marked “informational”
    
-  Winning decision logged with scores
    

---

## LAYER 3 — PROGRESS EVENTS (Trust & UX)

### What users should see

- “Collecting offers from 5 sellers…”
    
- “3 offers received, waiting for 2 more…”
    
- “Applying best available coupons…”
    
- “Ready for checkout — awaiting confirmation”
    

### Checklist

-  Progress events are **truthful**
    
-  No optimistic language before confirmation
    
-  User always knows **what step they are in**
    

🚨 Never show:

- “Order placed” before payment success
    
- “Lowest price guaranteed” unless contractually true
    

---

## LAYER 4 — TIMEOUTS (Critical)

### Where timeouts apply

- Retailer APIs
    
- Coupon services
    
- Shipping estimators
    
- Payment gateways
    

### Checklist

-  Each external call has a timeout
    
-  Whole flow time-boxed (UX SLA)
    
-  If a retailer times out → system continues without it
    
-  User informed when results are partial
    

Example:

> “Prices shown from 3 sellers. 2 sellers did not respond in time.”

---

## LAYER 5 — RETRIES (Extremely sensitive here)

### Safe retries (allowed)

- Product search
    
- Coupon lookup
    
- Shipping ETA fetch
    

### Dangerous retries (must be idempotent)

- Checkout initiation
    
- Payment authorization
    
- Order placement
    

### Checklist (CRITICAL)

-  **Idempotency keys** for:
    
    - payment intent
        
    - order creation
        
-  Retry only on:
    
    - network failure
        
    - timeout (not logical errors)
        
-  Never retry blindly on payment failure
    

🔥 This is where most real e-commerce systems break.

---

## LAYER 6 — DEGRADED MODE (Reality of E-commerce)

### When degradation happens

- Some retailers unavailable
    
- Coupon service down
    
- Shipping ETA uncertain
    

### Checklist

-  Degraded results clearly labeled
    
-  Cached prices marked “may have changed”
    
-  User can choose to proceed or wait
    

Example safe degradation:

> “I found 2 good options right now. Want me to proceed or wait for more sellers?”

---

## LAYER 7 — TRACEABILITY & AUDIT (NON-NEGOTIABLE)

### What must be traceable

- Which sellers were queried
    
- Which offers were received
    
- Why winner was chosen
    
- What price user saw
    
- What price was charged
    
- When user approved
    

### Checklist

-  Single trace ID per shopping session
    
-  All bids logged with timestamps
    
-  Decision rule stored
    
-  User consent event logged
    

This protects:

- the company (disputes)
    
- the user (trust)
    
- you (debugging)
    

---

## LAYER 8 — E-COMMERCE SAFETY GUARDRAILS

-  Never hallucinate prices
    
-  Never assume stock
    
-  Never auto-checkout without explicit user consent
    
-  Never hide fees
    
-  Always surface return policy
    

---

## One-line E-commerce production test

> “If the user disputes a charge, can I reconstruct **what they saw, why we chose it, and when they approved**?”

If yes → production-ready.

---
