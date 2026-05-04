# Peakly Agent Quick Start

Get your AI agent integrated with Peakly in under 10 minutes.

## Prerequisites

- A Peakly account at [app.peakly.ar](https://app.peakly.ar)
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

A `200 OK` with a JSON array confirms you're authenticated.

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
[
  {
    "id": 42,
    "businessName": "ACME S.A.",
    "cuit": "30-12345678-9",
    "ivaCondition": "responsable_inscripto"
  }
]
```

### Step 2: Find a receipt book

```bash
curl "https://api.peakly.ar/v1/receipt-books" \
  -H "X-API-Key: pk_your_api_key_here"
```

Look for a receipt book whose `receiptType` matches the customer's IVA condition:
- Customer is `responsable_inscripto` → use a **Factura A** receipt book
- Customer is `consumidor_final` or `monotributista` → use a **Factura B** or **C** receipt book

### Step 3: Create the receipt

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

A successful response includes a `cae` field once AFIP authorizes the receipt. If `cae` is absent, the receipt is in draft (`isDraft: true`) — call `POST /v1/sales/sales-receipts/{id}/authorize` to submit to AFIP.

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

```json
{
  "statusCode": 422,
  "message": "AFIP rejected the receipt",
  "afipError": {
    "code": 10016,
    "description": "El campo IVA no puede ser cero para este tipo de comprobante"
  }
}
```

When you receive a `422`, read `afipError.description` — it contains the official AFIP error message in Spanish. Common fixes:
- `10016` — IVA amount is missing or zero for a Factura A
- `10043` — Receipt number is out of sequence (use `/v1/sales/sales-receipts/next-number`)
- `10048` — AFIP service temporarily unavailable, retry with exponential backoff

## 5. SDK Quick Reference

```typescript
import { PeaklyClient } from "@peakly/sdk";

const client = new PeaklyClient({ apiKey: process.env.PEAKLY_API_KEY! });

// Find customers
const { data: customers } = await client.sales.customers.list({ search: "ACME" });

// Create a receipt
const { data: receipt, error } = await client.sales.receipts.create({
  customerId: 42,
  receiptBookId: 1,
  date: "2026-04-29",
  saleConditionId: 1,
  details: [{ description: "Services", quantity: 1, unitPrice: 10000 }],
});

if (error) {
  console.error("Failed:", error);
} else {
  console.log("CAE:", receipt.cae);
}

// Authorize (submit to AFIP)
await client.sales.receipts.authorize(receipt.id);
```

## Next Steps

- [Workflow Cookbook](workflows.md) — common agent patterns (search customer, create Factura A/B, void, check balance)
- [Concepts](concepts.md) — understand Argentine invoicing, AFIP, CAE, and factura types
