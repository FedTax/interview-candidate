# Sales Tax Rate Engine - Technical Assessment

## Overview

Build a REST API that calculates sales tax for e-commerce transactions AND processes tax refunds for returns. Your API handles the full order lifecycle: calculating tax at checkout, and reversing tax when items are returned.

The API receives order details — items (potentially shipping from different warehouses), discounts (per-item and order-level), fee-type line items (excise taxes, delivery fees), destination address, seller nexus, marketplace flags, and optional exemption certificates — and returns a jurisdiction-level tax breakdown. A separate returns endpoint reverses tax for returned items using the original transaction's rates.

Sales tax in the US is complex. Rates vary by jurisdiction, product taxability varies by state, sellers only collect where they have nexus, some states have tax holidays, local jurisdictions cap tax on expensive items, discounts change what's actually taxable, some line items are government-mandated fees with their own special rules, and marketplace orders are handled differently from direct sales. Your implementation should handle these rules correctly using the provided data files.

**Time limit**: ~4 hours from when you start. Focus on correctness over polish.

**You may use AI tools freely** — we care about your process and how you use them, not just the final result.

---

## API Specification

### Endpoint 1: `POST /calculate-tax`

Calculates tax for a new order.

**Request body**:
```json
{
  "order_id": "ORD-12345",
  "transaction_date": "2026-06-15",
  "captured_date": null,
  "items": [
    {
      "id": "item-1",
      "category": "clothing",
      "price": 120.00,
      "quantity": 2,
      "discount": 25.00,
      "origin": { "state": "CA", "zip": "90001" }
    },
    {
      "id": "item-2",
      "category": "electronics",
      "price": 499.99,
      "quantity": 1,
      "discount": 0,
      "origin": { "state": "NY", "zip": "10001" }
    }
  ],
  "order_discount": 30.00,
  "shipping": {
    "cost": 12.99
  },
  "destination": {
    "state": "NY",
    "zip": "10001"
  },
  "seller_nexus_states": ["NY", "CA", "TX"],
  "is_marketplace_order": false,
  "exemption_certificate": null
}
```

**Response body**:
```json
{
  "order_id": "ORD-12345",
  "items": [
    {
      "id": "item-1",
      "original_amount": 240.00,
      "line_discount": 25.00,
      "allocated_order_discount": 9.35,
      "taxable_amount": 0.00,
      "tax": 0.00,
      "exempt": true,
      "exempt_reason": "Clothing under $110 threshold in NY (post-discount price: $107.50)",
      "jurisdiction_breakdown": []
    },
    {
      "id": "item-2",
      "original_amount": 499.99,
      "line_discount": 0.00,
      "allocated_order_discount": 20.65,
      "taxable_amount": 479.34,
      "tax": 42.54,
      "exempt": false,
      "exempt_reason": null,
      "jurisdiction_breakdown": [
        { "jurisdiction": "NY State", "rate": 0.04, "tax": 19.17 },
        { "jurisdiction": "New York County", "rate": 0.045, "tax": 21.57 },
        { "jurisdiction": "MCTD District", "rate": 0.00375, "tax": 1.80 }
      ]
    }
  ],
  "shipping": {
    "taxable_amount": 12.99,
    "tax": 1.15,
    "jurisdiction_breakdown": [
      { "jurisdiction": "NY State", "rate": 0.04, "tax": 0.52 },
      { "jurisdiction": "New York County", "rate": 0.045, "tax": 0.58 },
      { "jurisdiction": "MCTD District", "rate": 0.00375, "tax": 0.05 }
    ]
  },
  "total_tax": 43.69,
  "total_taxable": 492.33,
  "total_exempt": 205.65
}
```

### Endpoint 2: `POST /calculate-return`

Calculates the tax refund for returned items from a previously-calculated order.

**Request body**:
```json
{
  "original_order_id": "ORD-12345",
  "return_items": [
    { "id": "item-2", "quantity": 1 }
  ]
}
```

**Response body**:
```json
{
  "original_order_id": "ORD-12345",
  "returned_items": [
    {
      "id": "item-2",
      "refund_amount": 479.34,
      "tax_refund": 42.54,
      "jurisdiction_breakdown": [
        { "jurisdiction": "NY State", "rate": 0.04, "tax_refund": 19.17 },
        { "jurisdiction": "New York County", "rate": 0.045, "tax_refund": 21.57 },
        { "jurisdiction": "MCTD District", "rate": 0.00375, "tax_refund": 1.80 }
      ]
    }
  ],
  "shipping_tax_refund": 1.15,
  "shipping_tax_refund_reason": "100% of items returned, full shipping tax refund",
  "total_tax_refund": 43.69,
  "total_refund": 536.03
}
```

### Field Definitions

**Calculate-Tax Request**:
- `order_id` — Unique order identifier. Used to reference this order in returns.
- `transaction_date` — Date of the transaction (ISO 8601). Used for rate lookup and tax holiday determination.
- `captured_date` — Optional. When present, this date is used for rate lookups instead of `transaction_date`. Represents when the payment was actually captured (which may differ from when the order was placed).
- `items[].discount` — Per-item discount amount (total for the line, not per-unit). Reduces taxable amount. Cannot be applied to fee-type items (see data file).
- `order_discount` — Order-level discount distributed proportionally across eligible items only (see discount rules in data file).
- `is_marketplace_order` — When `true`, a marketplace (e.g., Amazon, eBay) has already collected tax. See marketplace rules.
- All other fields as described in Endpoint 1 example.

**Calculate-Return Request**:
- `original_order_id` — References a previously-calculated order
- `return_items[].id` — Item ID from the original order
- `return_items[].quantity` — Number of units being returned (must be <= original quantity)

**Calculate-Return Response**:
- Tax refund amounts use the rates from the ORIGINAL order calculation, not current rates
- `shipping_tax_refund` — Proportional: % of order value returned = % of shipping tax refunded
- Non-refundable items are rejected (see data file)

---

## Additional Examples

### Example 2: Discount pushes clothing below NY threshold

**Request**:
```json
{
  "order_id": "ORD-200",
  "transaction_date": "2026-06-15",
  "items": [
    { "id": "item-1", "category": "clothing", "price": 120.00, "quantity": 1, "discount": 15.00, "origin": { "state": "NY", "zip": "10001" } }
  ],
  "order_discount": 0,
  "shipping": { "cost": 0 },
  "destination": { "state": "NY", "zip": "10001" },
  "seller_nexus_states": ["NY"],
  "exemption_certificate": null
}
```

**Response**: The $120 jacket has a $15 line discount → effective price is $105. Since $105 < $110, it's exempt under NY's clothing threshold.
```json
{
  "order_id": "ORD-200",
  "items": [
    {
      "id": "item-1",
      "original_amount": 120.00,
      "line_discount": 15.00,
      "allocated_order_discount": 0.00,
      "taxable_amount": 0.00,
      "tax": 0.00,
      "exempt": true,
      "exempt_reason": "Clothing under $110 threshold in NY (post-discount price: $105.00)"
    }
  ],
  "shipping": { "taxable_amount": 0, "tax": 0.00 },
  "total_tax": 0.00,
  "total_taxable": 0.00,
  "total_exempt": 105.00
}
```

### Example 3: Partial return with proportional shipping refund

**Setup**: An order with 2 items totaling $500. Item-1 is $200, item-2 is $300. Full tax was calculated.

**Return Request**: Return only item-1 ($200 of $500 = 40% of order)
```json
{
  "original_order_id": "ORD-300",
  "return_items": [
    { "id": "item-1", "quantity": 1 }
  ]
}
```

**Key behavior**: Item-1's tax is fully refunded. Shipping tax refund = 40% of original shipping tax (since 40% of order value is being returned).

### Example 4: Rate date override — captured_date used instead of transaction_date

When `captured_date` is provided, all rate lookups use that date instead of `transaction_date`. This matters when rates change between order placement and payment capture. The API should use `captured_date` for rate lookups and tax holiday date checks if present, falling back to `transaction_date` if `captured_date` is null.

---

## Business Rules

### 1. Nexus

A seller is only required to collect sales tax in states where they have **nexus**. If the seller does not have nexus in the destination state, the entire order is tax-free. If `seller_nexus_states` is missing or empty, treat as nexus everywhere.

### 2. Tax Rate Lookup

Tax rates are in `data/rates.json`. Each ZIP maps to jurisdiction rates. Jurisdiction taxes are independent — calculated separately, not as a combined rate.

**Rate date**: Use `captured_date` if provided, otherwise `transaction_date`. This date determines which rates apply and whether tax holidays are active.

### 3. Sourcing

Sourcing determines which location's rates apply. Since each item has its own origin, sourcing is evaluated **per item**:
- **Destination-based**: rates from buyer's location
- **Origin-based**: rates from seller's location for intrastate; destination for interstate
- **Modified origin**: split by jurisdiction level (see data file); destination for interstate

Shipping always uses destination rates regardless of item origins.

### 4. Fee-Type Items and Excise Taxes

Some line items in an order aren't regular products — they're government-mandated fees, shipping charges, or gratuities. These are identified by `is_fee: true` in the data file and follow different rules than regular items:

- **Discount exclusion**: Items marked `excluded_from_discounts: true` must NEVER receive any portion of line-item or order-level discounts. If a request includes a non-zero line discount on such an item, return an error. Order-level discount allocation should skip these items entirely — they are excluded from both the numerator and denominator of the allocation calculation.
- **Taxability**: Fee items have their own taxability rules per state. Most fees are not subject to sales tax, but some are (e.g., core charges are taxable in all states). Check the per-state rules for each fee category.
- **Exemption certificates**: Excise tax items (`is_excise: true`) are NOT affected by exemption certificates. A resale certificate removes sales tax from regular products, but a taxable fee like a core charge remains taxable regardless.
- **Refunds**: Only items with `non_refundable: true` cannot be returned. Other fee items can be returned normally.
- **Shipping proportion**: Fee-type items are excluded from the CA proportional shipping calculation (only regular product items count toward the taxable/total ratio).

Read `data/taxability.json` carefully — the `fee_handling` section explains the general rules, and each fee category has its own specific flags.

### 5. Marketplace Orders

When `is_marketplace_order` is `true`, a marketplace facilitator (Amazon, eBay, etc.) has already collected and remitted sales tax on behalf of the seller. The seller does not need to collect again.

This is a simple short-circuit: if `is_marketplace_order` is `true`, the entire order should return zero tax, regardless of nexus, exemptions, or any other rules. Fee items still appear in the response but with zero tax. This flag overrides everything else.

### 6. Discounts

Two types of discounts, applied in this order:

**Line discounts** (`items[].discount`): Subtracted directly from the item's line value (`price * quantity - discount`). Cannot reduce below zero. Cannot be applied to items with `excluded_from_discounts: true` — return an error if attempted.

**Order discounts** (`order_discount`): Allocated proportionally across **eligible** items only. Eligible items are those that are: (1) not excluded_from_discounts, (2) not fee-type items, and (3) not categorically exempt in the destination state (e.g., groceries). If no items are eligible, the order discount is not allocated.

**Allocation formula**: For each eligible item:
```
item_share = order_discount * (item_pre_discount_value / sum_of_all_eligible_pre_discount_values)
```
Round each allocation to the nearest cent. If rounding causes the sum of allocations to differ from the order_discount, adjust the eligible item with the largest allocation to make it exact.

**After discounts**: The post-discount amount is used for:
- Threshold checks (NY clothing $110, TX holiday $100)
- Taxable amount calculation
- CA proportional shipping calculation
- Tax cap evaluation

### 5. Product Taxability

Whether a product is taxable depends on category + destination state. See `data/taxability.json`. Some categories have thresholds evaluated against the **post-discount per-unit price** (price minus per-unit discount allocation).

### 6. Tax Holidays

Some states temporarily exempt categories during date ranges. See `tax_holiday` fields in the data. Holiday threshold is evaluated against the post-discount per-unit selling price. Use the **rate date** (captured_date or transaction_date) to determine if a holiday is active.

### 7. Local Tax Caps

Some jurisdictions cap local (city + district) tax at a max per-unit value. State tax is uncapped. The cap is evaluated against the **post-discount per-unit price**. See `local_tax_cap` in rate data.

### 8. Shipping Taxability

Varies by state. One state uses proportional calculation based on post-discount taxable values. One state always taxes shipping even when items are exempt.

### 9. Exemption Certificates

- **Resale**: All tax removed
- **Nonprofit**: State-specific (see data). HI removes state tax but county still applies.

### 10. Rounding

Round tax per jurisdiction per line item to nearest cent. Floating-point arithmetic introduces errors in financial calculations — consider appropriate data types.

### 11. Returns

The `/calculate-return` endpoint reverses tax for returned items:

- **Original rates**: Always use the tax calculation from the original order, not current rates. Your API must store or reconstruct the original calculation.
- **Partial returns**: Only the returned items' tax is refunded. You can return fewer units than originally ordered.
- **Shipping tax refund**: Proportional to the value returned. If returning items worth 40% of the original order's value, refund 40% of original shipping tax.
- **Non-refundable items**: Some categories are marked `non_refundable` in the data. Attempts to return these should result in an error.
- **Multiple returns**: An order can be partially returned multiple times. The second return should account for what was already returned.

### 12. Input Validation

Return appropriate HTTP error codes for:
- Invalid or missing required fields
- Unknown product categories or ZIP codes
- Negative prices, zero quantities
- Return quantities exceeding original quantities
- Attempts to return non-refundable items
- Returns referencing non-existent orders

---

## Data Files

- `data/rates.json` — Tax rates by state/ZIP, sourcing method, local tax caps
- `data/taxability.json` — Taxability rules, tax holidays, shipping rules, discount rules, return rules, exemption certificates

---

## Deliverables

1. A working API server with both endpoints
2. Your code pushed to this repository
3. A brief `NOTES.md` describing assumptions, decisions, and how you handled ambiguous cases

**Language/framework**: Your choice. Tests hit your API over HTTP.

---

## Tips

- Read the requirements and data files thoroughly before coding
- The interactions between features are where the real complexity lives: discounts + thresholds, returns + partial quantities, tax caps + discounts, holidays + captured dates
- Consider: what happens when an order discount pushes an item below a threshold, then the customer returns the OTHER item — does the threshold exemption still apply to the original calculation?
- Write your own tests
- Your `NOTES.md` matters — it shows your reasoning on ambiguous cases
