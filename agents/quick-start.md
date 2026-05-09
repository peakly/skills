# Peakly Agent Quick Start

Get your AI agent integrated with Peakly in under 10 minutes.

## Prerequisites

- A Peakly account at [peakly.ar](https://peakly.ar)
- An organization configured with at least one receipt book (talonario)

## 1. Authentication

Peakly uses API key authentication. All requests must include the `X-API-Key` header.

```bash
X-API-Key: pk_your_api_key_here
```

**Create an API key:**

1. Log into Peakly → Settings → API Keys
2. Click "New API Key" and copy the key (shown only once)

The API key is scoped to your organization — no `org_id` parameter needed.

**Test your key:**

```bash
curl https://api.peakly.ar/v1/customers \
  -H "X-API-Key: pk_your_api_key_here"
```

A `200 OK` with `{ "data": [...], "nextCursor": null, "hasMore": false }` confirms you are authenticated.

## 2. Environment Variables

```bash
export PEAKLY_API_KEY=pk_your_api_key_here
```

With the TypeScript SDK:

```typescript
import { PeaklyClient } from "@peakly/sdk";

const client = new PeaklyClient({
  apiKey: process.env.PEAKLY_API_KEY!,
});
```

## 3. Your First Invoice

Creating a sales receipt (factura) is a three-step process: look up the customer → find the receipt book → create the receipt.

### Step 1: Find the customer

```bash
curl "https://api.peakly.ar/v1/customers?search=ACME" \
  -H "X-API-Key: pk_your_api_key_here"
```

```json
{
  "data": [
    {
      "id": 42,
      "businessName": "ACME S.A.",
      "taxId": "30123456789",
      "taxCategoryId": 1,
      "email": "pagos@acme.com.ar",
      "isActive": true
    }
  ],
  "nextCursor": null,
  "hasMore": false
}
```

The `taxCategoryId` tells you the customer's IVA condition. Fetch the full list of tax categories to know which IVA type applies:

```bash
curl "https://api.peakly.ar/v1/tax-categories" \
  -H "X-API-Key: pk_your_api_key_here"
```

Look at the `discriminates` flag: `true` means the customer is **Responsable Inscripto** (RI) and expects a Factura A with itemized IVA. `false` means Consumidor Final, Monotributista, or Exento — use Factura B or C.

### Step 2: Find a receipt book

```bash
curl "https://api.peakly.ar/v1/receipt-books" \
  -H "X-API-Key: pk_your_api_key_here"
```

A single receipt book supports Factura A, B, and C — the actual letter issued is determined by the customer's tax category at the time of creation. Pick a receipt book based on your organization's setup (e.g., by `description` or `pointOfSale` number).

### Step 3: Create the receipt

When `isDraft` is omitted (or `false`), the receipt is automatically confirmed and submitted to AFIP — no separate authorize call needed.

```bash
curl -X POST "https://api.peakly.ar/v1/sales/sales-receipts" \
  -H "X-API-Key: pk_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 42,
    "receiptBookId": 1,
    "date": "2026-04-29",
    "saleConditionId": 1,
    "details": [
      {
        "description": "Consulting services - April 2026",
        "quantity": 1,
        "unitPrice": 10000.00
      }
    ]
  }'
```

The response includes `status: "Pendiente_Afip"` immediately after creation. Once AFIP responds, the receipt moves to `status: "Creada"` and a `cae` value appears (14-digit AFIP authorization code). If AFIP is slow, poll `GET /v1/sales/sales-receipts/{id}` until `status` changes.

**Creating as a draft first** (optional): pass `"isDraft": true` to skip the AFIP step. The receipt stays at `status: "Borrador"`. When ready, call `POST /v1/sales/sales-receipts/{id}/confirm` to trigger AFIP authorization.

## 4. Error Handling

All errors follow this shape:

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "errors": ["receiptBookId is required"]
}
```

Common errors to handle:

| Code | Meaning | Action |
|---|---|---|
| `400` | Validation error | Check `errors` array for field-level messages |
| `401` | Invalid API key | Verify `X-API-Key` header |
| `404` | Resource not found | Check the ID in the path |
| `409` | Conflict (duplicate) | The receipt may already exist |
| `422` | AFIP rejection | See `afipError` field for AFIP error code |

### AFIP rejections

When AFIP rejects a receipt during authorization, the receipt remains at `status: "Pendiente_Afip"` and the authorize result includes an `error` field:

```json
{
  "success": false,
  "error": "El campo IVA no puede ser cero para este tipo de comprobante"
}
```

To retry AFIP authorization after fixing the issue, call:

```bash
curl -X POST "https://api.peakly.ar/v1/sales/sales-receipts/{id}/authorize" \
  -H "X-API-Key: pk_your_api_key_here"
```

Common AFIP error codes:
- `10016` — IVA amount is missing or zero for a Factura A
- `10043` — Receipt number is out of sequence (use `GET /v1/sales/sales-receipts/next-number?receipt_book_id={id}&customer_id={id}` to preview)
- `10048` — AFIP service temporarily unavailable, retry with exponential backoff

## 5. SDK Quick Reference

```typescript
import { PeaklyClient } from "@peakly/sdk";

const client = new PeaklyClient({ apiKey: process.env.PEAKLY_API_KEY! });

// Find customers
const { data: page } = await client.sales.customers.list({ search: "ACME" });
const customer = page.data[0];

// Create a receipt (auto-submits to AFIP)
const { data: receipt, error } = await client.sales.receipts.create({
  customerId: customer.id,
  receiptBookId: 1,
  date: "2026-04-29",
  saleConditionId: 1,
  details: [{ description: "Services", quantity: 1, unitPrice: 10000 }],
});

if (error) {
  console.error("Failed:", error);
} else {
  // Poll until AFIP responds (status changes from Pendiente_Afip to Creada)
  console.log("Status:", receipt.status, "CAE:", receipt.cae);
}
```

## Next Steps

- [Workflow Cookbook](workflows.md) — common agent patterns (search customer, create Factura A/B, void, check balance)
- [Concepts](concepts.md) — understand Argentine invoicing, AFIP, CAE, and factura types
- [Entity Reference](entities.md) — field-level guide for Customer, Product, and Sales Receipt
