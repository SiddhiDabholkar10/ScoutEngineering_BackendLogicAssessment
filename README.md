# ScoutEngineering_BackendLogicAssessment
---

## Section 1 - SQL Schema
For the schema, I assumed PostgreSQL as a design choice compatible with C#(.NET).


### Enums

```sql
CREATE TYPE sale_scope    AS ENUM ('PRODUCT', 'MERCHANT', 'SITEWIDE');
CREATE TYPE discount_type AS ENUM ('PERCENT_OFF', 'FIXED_AMOUNT_OFF', 'BOGO');
CREATE TYPE sale_status   AS ENUM ('PENDING', 'ACTIVE', 'REJECTED');
```

### Core sales table

```sql
CREATE TABLE sales (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Merchant & scope
    merchant            VARCHAR(255) NOT NULL,
    scope               sale_scope NOT NULL,

    -- Discount mechanics
    discount_type       discount_type NOT NULL,
    discount_percent    DECIMAL(5,2),         -- e.g. 30.00 = 30% off. Required if PERCENT_OFF
    discount_amount     DECIMAL(10,2),        -- e.g. 10.00. Required if FIXED_AMOUNT_OFF
    currency            CHAR(3),              -- ISO 4217, e.g. 'USD'. Required if discount_amount is set
    BG_buy_quantity     INT,                  -- Required if BOGO
    BG_get_quantity     INT,                  -- Required if BOGO
    bogo_discount_pct   DECIMAL(5,2),         -- e.g. 100.00 = free, 50.00 = half off. Required if BOGO

    -- Validity window
    sale_start_time     TIMESTAMPTZ NOT NULL,
    sale_end_time       TIMESTAMPTZ,          -- NULL = open-ended sale

    -- Confidence & status
    confidence_score    DECIMAL(4,3) NOT NULL CHECK (confidence_score BETWEEN 0 AND 1),
    status              sale_status NOT NULL DEFAULT 'PENDING',

   
    source_url          TEXT   -- could also be VARCHAR
);
```

### Join table

One sale can apply to many products. Omitted for `MERCHANT` and `SITEWIDE` scoped sales.

```sql
CREATE TABLE sale_products (
    sale_id             UUID NOT NULL REFERENCES sales(id),
    product_id          UUID NOT NULL REFERENCES products(id),
    PRIMARY KEY (sale_id, product_id)
);
```

### Indexes 

```sql
-- Filter by status (ACTIVE / REJECTED / PENDING)
CREATE INDEX idx_sales_status
    ON sales(status);

-- Core index for iOS "Sales tab" query - status + time window
CREATE INDEX idx_sales_status_time
    ON sales(status, sale_start_time, sale_end_time);

-- Merchant-scoped and sitewide lookups
CREATE INDEX idx_sales_merchant
    ON sales(merchant);

-- Future: admin review of borderline confidence scores
CREATE INDEX idx_sales_confidence_score
    ON sales(confidence_score);

-- Core index for GET /products/{id}/sales
CREATE INDEX idx_sale_products_product_id
    ON sale_products(product_id);
```


Note: The schema separates sale scope from discount mechanics intentionally: a sitewide sale is still either a percentage off or a fixed amount off, so scope and discount_type are independent columns. Nullable discount columns (discount_percent, discount_amount, BOGO fields) are enforced at the API layer rather than the database, keeping the schema flexible. Every sale is written to the database regardless of confidence score: the status column gates what surfaces to the iOS app, ensuring nothing is silently lost.

---

## Section 2 - API Design

### POST /sales

Used by the internal upstream service to write a detected sale.

**Authentication**

Internal service endpoint: uses an API key in the request header, not a Bearer token.

```
X-Internal-API-Key: <key>
```

**Request Body**

```json
{
  
  "merchant":           "xyz.com",                // required
  "scope":              "PRODUCT",                // required - PRODUCT | MERCHANT | SITEWIDE
  "discount_type":      "PERCENT_OFF",            // required - PERCENT_OFF | FIXED_AMOUNT_OFF | BOGO
  "sale_start_time":    "2026-05-08T00:00:00Z",   // required
  "confidence_score":   0.91,                     // required - 0.000 to 1.000

  
  "discount_percent":   30.00,                    // required if PERCENT_OFF - e.g. 30.00 = 30% off

  
  "discount_amount":    10.00,                    // required if FIXED_AMOUNT_OFF
  "currency":           "USD",                    // required if discount_amount is 

  "BG_buy_quantity":    2,                        // required if BOGO - buy X
  "BG_get_quantity":    1,                        // required if BOGO - get Y
  "bogo_discount_pct":  100.00,                   // required if BOGO - 100.00 = free, 50.00 = half off


  "product_ids": ["uuid-1", "uuid-2"],            // required if scope = PRODUCT - omit for MERCHANT | SITEWIDE


  "sale_end_time":      "2026-05-10T23:59:59Z",   // optional - null means open-ended sale
  "source_url":         "https://xyz.com/sale"    // optional - where the sale was detected
}
```

**Confidence Logic**

```
confidence_score >= 0.85 → status = ACTIVE  → 201 Created
confidence_score <  0.85 → status = REJECTED → 202 Accepted
```

Both cases are written to the database. The status field controls whether the sale surfaces to the iOS app.

**Validation Rules**

| Rule | Code |
|---|---|
| `merchant`, `scope`, `discount_type`, `sale_start_time`, `confidence_score` missing | `400` |
| `discount_percent` missing when `discount_type = PERCENT_OFF` | `400` |
| `discount_amount` or `currency` missing when `discount_type = FIXED_AMOUNT_OFF` | `400` |
| `BG_buy_quantity`, `BG_get_quantity`, `bogo_discount_pct` missing when `discount_type = BOGO` | `400` |
| `product_ids` empty or missing when `scope = PRODUCT` | `400` |
| `sale_end_time` is before or equal to `sale_start_time` | `400` |
| `confidence_score` outside 0.000–1.000 range | `400` |
| One or more `product_ids` not found in `products` table | `404` |
| Same `merchant + discount_type + sale_start_time` already exists | `409` |

**Response - 201 Created (ACTIVE)**

```json
{
  "id":               "a1b2c3d4-...",
  "status":           "ACTIVE",
  "merchant":         "xyz.com",
  "scope":            "PRODUCT",
  "discount_type":    "PERCENT_OFF",
  "discount_percent": 30.00,
  "sale_start_time":  "2026-05-08T00:00:00Z",
  "sale_end_time":    "2026-05-10T23:59:59Z",
  "confidence_score": 0.91,
  "product_ids":      ["uuid-1", "uuid-2"],
  "source_url":       "https://xyz.com/sale",
  "created_at":       "2026-05-11T10:00:00Z"
}
```

**Response - 202 Accepted (REJECTED)**

```json
{
  "id":               "a1b2c3d4-...",
  "status":           "REJECTED",
  "confidence_score": 0.71,
  "reason":           "confidence_score below threshold of 0.85",
  "created_at":       "2026-05-11T10:00:00Z"
}
```

**Status Codes**

| Code | When |
|---|---|
| `201 Created` | Sale accepted, confidence ≥ 0.85, status = ACTIVE |
| `202 Accepted` | Sale received, confidence < 0.85, status = REJECTED |
| `400 Bad Request` | Validation failure |
| `401 Unauthorized` | Missing or invalid API key |
| `404 Not Found` | One or more product_ids not found |
| `409 Conflict` | Duplicate sale detected |


---

### GET /products/{id}/sales

Used by the iOS app to fetch all active sales for a single product.

**Authentication**

End-user facing - uses a Bearer token (JWT).

```
Authorization: Bearer <token>
```

**Path Parameter**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | UUID | Yes | The product ID to fetch sales for |

**What this returns**

For a given `product_id`, this endpoint merges and returns:
1. Product-scoped sales tied directly to this product via `sale_products`
2. Merchant-scoped sales where `merchant` matches and `scope = MERCHANT`
3. Sitewide sales where `scope = SITEWIDE`

All three are returned in a single response - the iOS app does not need to make separate calls.

**Validation Rules**

| Rule | Code |
|---|---|
| `id` is not a valid UUID | `400` |
| `id` does not exist in `products` table | `404` |
| No active sales found | `200` with empty array |

**Response - 200 OK**

```json
{
  "product_id": "uuid-1",
  "sales": [
    {
      "id":               "a1b2c3d4-...",
      "scope":            "PRODUCT",
      "discount_type":    "PERCENT_OFF",
      "discount_percent": 30.00,
      "sale_start_time":  "2026-05-08T00:00:00Z",
      "sale_end_time":    "2026-05-10T23:59:59Z",
      "source_url":       "https://xyz.com/sale"
    },
    {
      "id":               "b2c3d4e5-...",
      "scope":            "SITEWIDE",
      "discount_type":    "PERCENT_OFF",
      "discount_percent": 15.00,
      "sale_start_time":  "2026-05-01T00:00:00Z",
      "sale_end_time":    null,
      "source_url":       "https://xyz.com/promo"
    }
  ]
}
```

Note: `confidence_score` and `status` are intentionally excluded from the response - the iOS app only receives clean, already-filtered active sales.

**Status Codes**

| Code | When |
|---|---|
| `200 OK` | Success - empty array if no active sales |
| `400 Bad Request` | Malformed UUID in path |
| `401 Unauthorized` | Missing or invalid Bearer token |
| `404 Not Found` | Product ID does not exist |

---

### GET /sales?status=active

Used by the iOS app to fetch all currently active sales for the "Sales" tab.

**Authentication**

End-user facing - uses a Bearer token (JWT).

```
Authorization: Bearer <token>
```

**Query Parameters**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `status` | ENUM | No | `ACTIVE` | `ACTIVE` \| `REJECTED` \| `PENDING` |
| `merchant` | VARCHAR | No | null | Filter by merchant - e.g. `?status=active&merchant=xyz.com` |
| `scope` | ENUM | No | null | Filter by scope - `PRODUCT` \| `MERCHANT` \| `SITEWIDE` |
| `page` | INT | No | `1` | Page number |
| `page_size` | INT | No | `20` | Results per page - max 100 |

**Validation Rules**

| Rule | Code |
|---|---|
| `status` is not a valid enum value | `400` |
| `scope` is not a valid enum value | `400` |
| `page` is not a positive integer | `400` |
| `page_size` is outside 1–100 range | `400` |
| No active sales exist | `200` with empty array |

**Response - 200 OK**

```json
{
  "page": 1,
  "page_size": 20,
  "total": 84,
  "sales": [
    {
      "id":               "a1b2c3d4-...",
      "merchant":         "xyz.com",
      "scope":            "PRODUCT",
      "discount_type":    "PERCENT_OFF",
      "discount_percent": 30.00,
      "sale_start_time":  "2026-05-08T00:00:00Z",
      "sale_end_time":    "2026-05-10T23:59:59Z",
      "source_url":       "https://xyz.com/sale",
      "product_ids":      ["uuid-1", "uuid-2"]
    },
    {
      "id":               "b2c3d4e5-...",
      "merchant":         "abc.com",
      "scope":            "SITEWIDE",
      "discount_type":    "FIXED_AMOUNT_OFF",
      "discount_amount":  10.00,
      "currency":         "USD",
      "sale_start_time":  "2026-05-01T00:00:00Z",
      "sale_end_time":    null,
      "source_url":       "https://abc.com/promo",
      "product_ids":      []
    }
  ]
}
```

**Status Codes**

| Code | When |
|---|---|
| `200 OK` | Success - empty array if no active sales |
| `400 Bad Request` | Invalid query parameter |
| `401 Unauthorized` | Missing or invalid Bearer token |
| `403 Forbidden` | Valid token but insufficient scope |


---

## Section 3 - Pseudocode: POST /sales

```

// Confidence threshold: sales below 0.85 are stored but never surfaced to iOS

CONST CONFIDENCE_THRESHOLD = 0.85

FUNCTION HandlePostSale(request, headers):

    // ── Step 1: Authenticate ─────────────────────────────────────
    apiKey = headers["X-Internal-API-Key"]
    IF apiKey is missing OR not valid:
        RETURN 401 Unauthorized { error: "Invalid or missing API key" }

    // ── Step 2: Validate request body ────────────────────────────
    errors = []

    IF merchant, scope, discount_type, sale_start_time, confidence_score are missing:
        ADD "required field missing" TO errors

    IF confidence_score not between 0.000 and 1.000:
        ADD "confidence_score out of range" TO errors

    IF sale_end_time is provided AND sale_end_time <= sale_start_time:
        ADD "sale_end_time must be after sale_start_time" TO errors

    // Conditional validation based on discount_type
    IF discount_type == PERCENT_OFF AND discount_percent is missing:
        ADD "discount_percent required for PERCENT_OFF" TO errors

    IF discount_type == FIXED_AMOUNT_OFF AND discount_amount or currency is missing:
        ADD "discount_amount and currency required for FIXED_AMOUNT_OFF" TO errors

    IF discount_type == BOGO AND BG_buy_quantity, BG_get_quantity, bogo_discount_pct are missing:
        ADD "BOGO fields required for BOGO" TO errors

    // Conditional validation based on scope
    IF scope == PRODUCT AND product_ids is empty or missing:
        ADD "product_ids required when scope is PRODUCT" TO errors

    IF errors is not empty:
        RETURN 400 Bad Request { errors: errors }

    // ── Step 3: Validate product_ids exist in DB ─────────────────
    IF scope == PRODUCT:
        missingIds = DB.products.FindMissing(request.product_ids)
        IF missingIds is not empty:
            RETURN 404 Not Found { error: "product_ids not found", missing: missingIds }

    // ── Step 4: Duplicate detection ──────────────────────────────
    existing = DB.sales.FindOne(
        merchant      == request.merchant,
        discount_type == request.discount_type,
        sale_start_time == request.sale_start_time
    )
    IF existing is found:
        RETURN 409 Conflict { error: "Duplicate sale detected" }

    // ── Step 5: Confidence gating ─────────────────────────────────
    // Core false-positive defense - every sale is stored for audit
    // but only high-confidence sales are surfaced to the iOS app
    IF request.confidence_score >= CONFIDENCE_THRESHOLD:
        status = ACTIVE
    ELSE:
        status = REJECTED

    // ── Step 6: Persist sale ──────────────────────────────────────
    newSale = {
        id:               generate new UUID,
        created_at:       now(),
        updated_at:       now(),
        merchant:         request.merchant,
        scope:            request.scope,
        discount_type:    request.discount_type,
        discount_percent: request.discount_percent,    // null if not PERCENT_OFF
        discount_amount:  request.discount_amount,     // null if not FIXED_AMOUNT_OFF
        currency:         request.currency,            // null if not FIXED_AMOUNT_OFF
        BG_buy_quantity:  request.BG_buy_quantity,     // null if not BOGO
        BG_get_quantity:  request.BG_get_quantity,     // null if not BOGO
        bogo_discount_pct:request.bogo_discount_pct,   // null if not BOGO
        sale_start_time:  request.sale_start_time,
        sale_end_time:    request.sale_end_time,       // null if open-ended
        confidence_score: request.confidence_score,
        status:           status,
        source_url:       request.source_url
    }
    DB.sales.Insert(newSale)

    // ── Step 7: Persist sale_products ────────────────────────────
    // Must happen AFTER sale insert - FK constraint requires parent row
    IF scope == PRODUCT:
        FOR EACH productId IN request.product_ids:
            DB.sale_products.Insert({ sale_id: newSale.id, product_id: productId })

    // ── Step 8: Return response ───────────────────────────────────
    IF status == ACTIVE:
        RETURN 201 Created { full sale object }

    IF status == REJECTED:
        RETURN 202 Accepted {
            id:               newSale.id,
            status:           "REJECTED",
            confidence_score: newSale.confidence_score,
            reason:           "confidence_score below threshold of 0.85",
            created_at:       newSale.created_at
        }

END FUNCTION
```

---

## Section 4 - Tradeoffs & Open Questions

- The 0.85 confidence threshold is a starting point, but it is ultimately a guess from my end. In production, the right number depends on real data: how many sales at 0.83 were real? How many at 0.90 were false positives? Without a feedback mechanism that lets someone flag a surfaced sale as fake or a rejected sale as missed, the threshold never improves. Before shipping, I would want a way to retroactively re-evaluate rejected sales if the threshold is adjusted, and a simple admin view to review borderline scores. The threshold should be treated as a tunable business parameter, not a hardcoded constant.

- The duplicate detection logic does a check-then-write: it looks for an existing sale, and if none is found, inserts a new one. If the upstream service sends two identical sales at the same time, both could pass the duplicate check simultaneously and be stored. Before shipping, I would enforce uniqueness at the database level so the second write fails cleanly, and I would align with the upstream team on whether duplicate writes are expected behavior or a bug on their end.

- When the iOS app loads a product page, the backend needs to check three things: is there a sale on this specific product, is there a merchant-wide sale, and is there a sitewide sale? Today, that is manageable, but as the number of merchants and products grows, the sitewide and merchant-wide checks run on every single product page load. A sitewide sale from a major retailer could technically apply to thousands of products simultaneously. Before scaling this feature, I would want to think carefully about whether sitewide sales should be cached separately and pushed to the app rather than queried on demand to reduce latency.



