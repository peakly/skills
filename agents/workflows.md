# Peakly Workflow Cookbook

Step-by-step recipes for common invoicing workflows. All examples use `curl`; see the TypeScript equivalents in the SDK quick start.

Base URL: `https://api.peakly.ar`  
Auth header: `X-API-Key: pk_your_api_key_here`

---

## 1. Search for a customer by name or CUIT

```bash
# By name
curl "https://api.peakly.ar/v1/customers?search=ACME" \
  -H "X-API-Key: pk_your_api_key_here"

# By CUIT (use the raw number, no dashes)
curl "https://api.peakly.ar/v1/customers/lookup-cuit?cuit=30123456789" \
  -H "X-API-Key: pk_your_api_key_here"
```

**Response fields to capture:**

```json
{
  "id": 42,
  "businessName": "ACME S.A.",
  "cuit": "30-12345678-9",
  "ivaCondition": "responsable_inscripto",
  "email": "pagos@acme.com.ar"
}
```

Use `id` when creating receipts. Use `ivaCondition` to pick the correct receipt type (see [Concepts](concepts.md)).

---

## 2. Create a Factura B (consumidor final / monotributista)

Use when your customer is a **consumer** (`consumidor_final`) or **monotributista**.

```bash
curl -X POST "https://api.peakly.ar/v1/sales/sales-receipts" \
  -H "X-API-Key: pk_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 42,
    "receiptBookId": 2,
    "date": "2026-04-29",
    "saleConditionId": 1,
    "details": [
      {
        "description": "Web development services",
        "quantity": 1,
        "unitPrice": 12100.00
      }
    ]
  }'
```

For Factura B, the price is typically expressed **including IVA** (precio con IVA). AFIP will calculate the tax breakdown automatically.

---

## 3. Create a Factura A (responsable inscripto — with IVA breakdown)

Use when your customer is a **responsable inscripto** (company registered for IVA).

For Factura A, each line item must include a `taxTypeId` referencing an IVA rate. Fetch available IVA types first:

```bash
curl "https://api.peakly.ar/v1/combos/9/items" \
  -H "X-API-Key: pk_your_api_key_here"
```

This returns the IVA combo items with their IDs (e.g. `taxTypeId: 3` for 21%, `taxTypeId: 2` for 10.5%, `taxTypeId: 1` for 0%). Use the appropriate `id` in the `taxTypeId` field.

```bash
curl -X POST "https://api.peakly.ar/v1/sales/sales-receipts" \
  -H "X-API-Key: pk_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 99,
    "receiptBookId": 1,
    "date": "2026-04-29",
    "saleConditionId": 1,
    "details": [
      {
        "description": "Consulting - April 2026",
        "quantity": 1,
        "unitPrice": 10000.00,
        "taxTypeId": 3
      }
    ]
  }'
```

The net price goes in `unitPrice`; IVA is calculated separately by Peakly based on the `taxTypeId`.

**After creation, authorize with AFIP:**

```bash
curl -X POST "https://api.peakly.ar/v1/sales/sales-receipts/{id}/authorize" \
  -H "X-API-Key: pk_your_api_key_here"
```

A `cae` field appears in the response once AFIP approves. Without a CAE, the receipt is not fiscally valid.

---

## 4. Void (anular) a receipt

Once a receipt has been authorized by AFIP, it cannot be deleted — it can only be voided (anulado).

```bash
curl -X POST "https://api.peakly.ar/v1/sales/sales-receipts/{id}/void" \
  -H "X-API-Key: pk_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{}'
```

By default, Peakly automatically creates a compensating credit note (Nota de Crédito). To skip it, send `{"createCreditNote": false}`.

**Notes:**
- Only same-period receipts can be voided in some AFIP configurations. If the receipt is from a prior period, you must issue a **Nota de Crédito** (credit note) instead.
- A voided receipt returns `status: "anulado"`.

---

## 5. Check outstanding receivables

```bash
curl "https://api.peakly.ar/v1/sales/reports/outstanding-receivables" \
  -H "X-API-Key: pk_your_api_key_here"
```

Returns a paginated list of unpaid receipts with their due dates and amounts across all customers.

**Pagination:**

| Parameter | Description |
|---|---|
| `cursor` | Opaque cursor for the next page (from previous response) |
| `page_size` | Results per page (default 50, max 100) |

---

## 6. List today's sales

```bash
# Replace date with today's date
curl "https://api.peakly.ar/v1/sales/sales-receipts?date_from=2026-04-29&date_to=2026-04-29" \
  -H "X-API-Key: pk_your_api_key_here"
```

**Useful filters:**

| Parameter | Description |
|---|---|
| `date_from` / `date_to` | Date range (`YYYY-MM-DD`) |
| `status` | Filter by status: `draft`, `confirmed`, `authorized`, `anulado` |
| `customer_id` | Filter by customer ID |
| `receipt_type` | Filter by factura type: `factura`, `nota_credito`, `nota_debito` |

---

## 7. Get organization dashboard summary

```bash
curl "https://api.peakly.ar/v1/sales/reports/dashboard-stats" \
  -H "X-API-Key: pk_your_api_key_here"
```

Returns totals for today, this month, and this year — useful for quick summaries.

---

## 8. Full end-to-end: invoice a customer

```typescript
import { PeaklyClient } from "@peakly/sdk";

const client = new PeaklyClient({ apiKey: process.env.PEAKLY_API_KEY! });

async function invoiceCustomer(customerSearch: string, description: string, amount: number) {
  // 1. Find customer
  const { data: customers } = await client.sales.customers.list({ search: customerSearch });
  if (!customers?.length) throw new Error(`Customer not found: ${customerSearch}`);
  const customer = customers[0];

  // 2. Pick receipt book based on IVA condition
  const { data: books } = await client.$fetch.GET("/v1/receipt-books", {});
  const isResponsable = customer.ivaCondition === "responsable_inscripto";
  const book = books?.find((b: any) => b.receiptType === (isResponsable ? "A" : "B"));
  if (!book) throw new Error("No matching receipt book found");

  // 3. Create receipt
  const { data: receipt, error } = await client.sales.receipts.create({
    customerId: customer.id,
    receiptBookId: book.id,
    date: new Date().toISOString().split("T")[0],
    saleConditionId: 1,
    details: [{ description, quantity: 1, unitPrice: amount }],
  });
  if (error) throw new Error(`Create failed: ${JSON.stringify(error)}`);

  // 4. Authorize with AFIP
  const { data: authorized } = await client.sales.receipts.authorize(receipt.id);
  return authorized;
}
```

---

## Error Handling Patterns

```typescript
const { data, error, response } = await client.sales.receipts.create(payload);

if (error) {
  if (response.status === 422 && error.afipError) {
    // AFIP rejected — log the code and retry with corrections
    console.error("AFIP error:", error.afipError.code, error.afipError.description);
  } else if (response.status === 400) {
    // Validation error
    console.error("Validation:", error.errors);
  } else {
    throw new Error(`Unexpected error ${response.status}: ${JSON.stringify(error)}`);
  }
}
```
