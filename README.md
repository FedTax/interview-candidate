# TaxCloud Technical Assessment — Sales Tax Rate Engine (Screening)

Welcome! Plan for about **2 hours** of focused work. The project is intentionally
larger than what most candidates finish in that time — we want to see how you
scope the problem, what you choose to build first, and how you communicate
about what you skip. **A working partial solution is better than a broken
complete one.**

---

## Getting Started

### 1. Create your own copy of this project

Click the green **Use this template** button at the top of this repo →
**Create a new repository** → set it to **Private** → click **Create**.

### 2. Clone your copy and launch the dev environment

Our Docker dev box has Claude Code, Python, Node, and Go pre-installed and
pre-configured for you. From the directory where you cloned your new repo:

```bash
docker run -it \
  -e ANTHROPIC_API_KEY="<your-api-key>" \
  -v $(pwd):/workspace \
  -p 8080:8080 \
  ghcr.io/fedtax/interview-devbox:latest
```

Replace `<your-api-key>` with the key from your onboarding email. Inside the
container, run `claude` — it's pre-wired to your key and our proxy.

> Prefer your own machine? `npm install -g @anthropic-ai/claude-code`, then
> `export ANTHROPIC_BASE_URL="https://interview-proxy.purplerock-e0f9c506.eastus2.azurecontainerapps.io"`
> and `export ANTHROPIC_API_KEY="<your-api-key>"` before running `claude`.

### 3. Build

Read the rules below, implement, and test. Use AI however you like.

### 4. Submit

`git push` to your own repo. Reply to your onboarding email with the URL.

---

## API Specification

### Endpoint: `POST /calculate-tax`

**Request body:**
```json
{
  "items": [
    { "id": "item-1", "category": "clothing",    "price": 95.00,  "quantity": 2 },
    { "id": "item-2", "category": "electronics", "price": 499.99, "quantity": 1 }
  ],
  "shipping": { "cost": 12.99 },
  "destination": { "state": "NY", "zip": "10001" },
  "seller_nexus_states": ["NY", "CA", "TX"],
  "exemption_certificate": null
}
```

**Response body:**
```json
{
  "items": [
    {
      "id": "item-1",
      "taxable_amount": 0.00,
      "tax": 0.00,
      "exempt": true,
      "exempt_reason": "Clothing under $110 threshold in NY",
      "jurisdiction_breakdown": []
    },
    {
      "id": "item-2",
      "taxable_amount": 499.99,
      "tax": 44.37,
      "exempt": false,
      "exempt_reason": null,
      "jurisdiction_breakdown": [
        { "jurisdiction": "NY State",        "rate": 0.04,    "tax": 20.00 },
        { "jurisdiction": "New York County", "rate": 0.045,   "tax": 22.50 },
        { "jurisdiction": "MCTD District",   "rate": 0.00375, "tax": 1.87  }
      ]
    }
  ],
  "shipping": {
    "taxable_amount": 12.99,
    "tax": 1.15,
    "jurisdiction_breakdown": [
      { "jurisdiction": "NY State",        "rate": 0.04,    "tax": 0.52 },
      { "jurisdiction": "New York County", "rate": 0.045,   "tax": 0.58 },
      { "jurisdiction": "MCTD District",   "rate": 0.00375, "tax": 0.05 }
    ]
  },
  "total_tax": 45.52,
  "total_taxable": 512.98,
  "total_exempt": 190.00
}
```

Return appropriate HTTP error codes for malformed or invalid requests.

---

## Business Rules

### 1. Nexus
A seller only collects tax in states where they have **nexus**. The
`seller_nexus_states` field lists those. If the seller does not have nexus
in the destination state, the entire order is tax-free (no tax on items or
shipping). If the field is empty or missing, treat the seller as having
nexus everywhere.

### 2. Tax Rate Lookup
Rates are in `data/rates.json`. Each ZIP maps to a state rate plus county,
city, and district rates. Each jurisdiction's tax is calculated separately
(not as a single combined rate), then the per-jurisdiction values are summed
for the line-item total.

### 3. Sourcing
For this project, all rate lookups use the **destination** ZIP. (Origin-based
and modified-origin rules exist in real life, but they are out of scope here.)

### 4. Product Taxability
Whether a product is taxable depends on its category and the destination state.
See `data/taxability.json`. Most rules are simple booleans.

The notable exception: **clothing in NY is conditionally exempt** — items
priced below $110 per unit are exempt, items at $110 or above are taxable.
The threshold is per-unit, not per-line-item.

### 5. Shipping Taxability
Shipping is taxable when the order contains taxable items. If every item in
the order is exempt, shipping is also exempt. See `shipping_rules` in the
data file.

### 6. Resale Exemption Certificate
If `exemption_certificate` is `{ "type": "resale" }`, all sales tax is
removed — items and shipping all become exempt regardless of category.

### 7. Rounding
Tax must be rounded per jurisdiction per line item. Calculate
`line_value * jurisdiction_rate`, round to the nearest cent, then sum.
Floating-point arithmetic introduces subtle errors in financial math —
consider whether your language's default number type is appropriate for
currency.

### 8. Input Validation
Return HTTP 400 for invalid requests: unknown product categories, unknown
ZIP codes, negative prices, zero quantities, empty item lists, missing
required fields.

---

## Deliverables

1. A working API server with `POST /calculate-tax`
2. Your code pushed to your private copy of this repo
3. A short `NOTES.md` describing assumptions, decisions, and anything you
   chose to skip given the time limit. **NOTES.md matters** — it's how you
   show your reasoning when output is incomplete.

**Language/framework:** Your choice. We'll test by hitting your API over HTTP.

---

## Tips

- Read the data files before coding — they reveal more than the README alone
- Get the basics solid before reaching for edge cases
- If you don't have time for something, document it in `NOTES.md` rather
  than ship something half-broken
- Use AI however you'd like — we care about *how* you use it, not whether
  you do
