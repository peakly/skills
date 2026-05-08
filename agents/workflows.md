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

# By CUIT (raw digits, no dashes)
curl "https://api.peakly.ar/v1/customers/lookup-cuit?cuit=30123456789" \
  -H "X-API-Key: pk_your_api_key_here"
```

**Customer response fields:**

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

`taxCategoryId` is a numeric ID for the customer's tax category (IVA condition). Use `GET /v1/tax-categories` to resolve names: if the category's `discriminates` is `true`, the customer is a **Responsable Inscripto** and expects a Factura A; otherwise use Factura B or C.

Use `id` when creating receipts.

---

## 2. Create a Factura B (consumidor final / monotributista)

Use when the customer's tax category has `discriminates: false` (Consumidor Final, Monotributista, Exento).

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

For Factura B, `unitPrice` is the **price including IVA** (precio con IVA). AFIP calculates the tax breakdown automatically.

**Successful response (receipt submitted to AFIP):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "number": "00002-00000047",
  "letter": "B",
  "status": "Pendiente_Afip",
  "total": 12100.00,
  "cae": null,
  "caeExpiration": null,
  ...
}
```

Once AFIP processes it (usually within seconds), `status` becomes `"Creada"` and `cae` is populated. Poll `GET /v1/sales/sales-receipts/{id}` if needed.

---

## 3. Create a Factura A (responsable inscripto — with IVA breakdown)

Use when the customer's tax category has `discriminates: true` (Responsable Inscripto).

For Factura A, each line item must include a `taxTypeId` referencing an IVA rate combo item. Fetch the available IVA types first:

```bash
curl "https://api.peakly.ar/v1/combos/9/items" \
  -H "X-API-Key: pk_your_api_key_here"
```

This returns combo items like:

```json
[
  { "id": 94, "data": "21%", "afipData": "5" },
  { "id": 93, "data": "10,5%", "afipData": "4" },
  { "id": 92, "data": "Sin IVA", "afipData": "3" },
  { "id": 91, "data": "Exento", "afipData": "2" },
  { "id": 90, "data": "No Gravado", "afipData": "1" }
]
```

Use the appropriate `id` in the `taxTypeId` field (e.g. `94` for 21%).

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
        "taxTypeId": 94
      }
    ]
  }'
```

`unitPrice` is the **net price** (before IVA). IVA is calculated and itemized by Peakly based on `taxTypeId`.

**Successful response:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "number": "00001-00000123",
  "letter": "A",
  "status": "Pendiente_Afip",
  "subtotal": 10000.00,
  "vat21": 2100.00,
  "total": 12100.00,
  "cae": null,
  "caeExpiration": null,
  ...
}
```

---

## 4. Void (anular) a receipt

Once authorized by AFIP, a receipt cannot be deleted — it can only be voided.

```bash
curl -X POST "https://api.peakly.ar/v1/sales/sales-receipts/{id}/void" \
  -H "X-API-Key: pk_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{}'
```

By default, Peakly automatically creates a compensating credit note (Nota de Crédito). To skip it, send `{"createCreditNote": false}`.

**Response:** the updated receipt with `"status": "Anulado"`.

**Notes:**
- Only same-period receipts can be voided in some AFIP configurations. If the receipt is from a prior period, issue a **Nota de Crédito** instead (create a new receipt with `"receiptType": "nota_credito"` and `"associatedReceiptId": "<original-id>"`).
- Only receipts with `status: "Creada"` or `status: "Cobrada"` can be voided.

---

## 5. Check outstanding receivables

Returns a paginated list of unpaid receipts with due dates and amounts across all customers.

```bash
curl "https://api.peakly.ar/v1/sales/reports/outstanding-receivables" \
  -H "X-API-Key: pk_your_api_key_here"
```

**Response shape:**

```json
{
  "data": [
    {
      "cliente": "ACME S.A.",
      "fecha": "2026-04-01",
      "fechaVencimiento": "2026-04-30",
      "comprobante": "00001-00000120",
      "tipoComprobante": "Factura A",
      "importe": 12100,
      "contactoCobro": "cobranzas@acme.com.ar",
      "observacion": null
    }
  ],
  "nextCursor": null,
  "hasMore": false
}
```

**Pagination:**

| Parameter | Description |
|---|---|
| `cursor` | Opaque cursor for the next page (from previous `nextCursor`) |
| `page_size` | Results per page (default 50, max 500) |

---

## 6. List today's sales

```bash
# Replace date with today's date
curl "https://api.peakly.ar/v1/sales/sales-receipts?date_from=2026-04-29&date_to=2026-04-29" \
  -H "X-API-Key: pk_your_api_key_here"
```

**Available filters:**

| Parameter | Description |
|---|---|
| `date_from` / `date_to` | Date range (`YYYY-MM-DD`) |
| `status` | Filter by status: `Borrador`, `Pendiente_Afip`, `Creada`, `Cobrada`, `Anulado` |
| `customer_id` | Filter by customer ID |
| `receipt_type` | Filter by type: `factura`, `nota_credito`, `nota_debito` |
| `search` | Free-text search by customer name, receipt number, or CAE. Supports prefixes: `cae:<value>`, `cuit:<value>`, `nro:<value>`, `id:<value>` |
| `receipt_book_id` | Filter by receipt book ID |
| `page_size` | Page size (default 20, max 100) |

**Response:**

```json
{
  "data": [ ... ],
  "nextCursor": null,
  "hasMore": false
}
```

---

## 7. Get organization dashboard summary

Requires `date_from` and `date_to` (both mandatory).

```bash
curl "https://api.peakly.ar/v1/sales/reports/dashboard-stats?date_from=2026-04-01&date_to=2026-04-29" \
  -H "X-API-Key: pk_your_api_key_here"
```

Optional `group_by` parameter: `day` (default), `week`, or `month`.

Returns `kpis`, `timeSeries`, `topCustomers`, `recentReceipts`, and `trailing12MonthsTotal` — useful for quick summaries.

---

## 8. Full end-to-end: invoice a customer

```typescript
import { PeaklyClient } from "@peakly/sdk";

const client = new PeaklyClient({ apiKey: process.env.PEAKLY_API_KEY! });

async function invoiceCustomer(customerSearch: string, description: string, amount: number) {
  // 1. Find customer
  const { data: page } = await client.sales.customers.list({ search: customerSearch });
  if (!page?.data?.length) throw new Error(`Customer not found: ${customerSearch}`);
  const customer = page.data[0];

  // 2. Resolve tax category to determine IVA behaviour
  const { data: taxCategories } = await client.$fetch.GET("/v1/tax-categories", {});
  const category = taxCategories?.find((c: any) => c.id === customer.taxCategoryId);
  const isRI = category?.discriminates === true;

  // 3. Pick receipt book
  const { data: books } = await client.$fetch.GET("/v1/receipt-books", {});
  if (!books?.length) throw new Error("No receipt books configured");
  const book = books[0]; // use first available; add your own selection logic

  // 4. Build detail line (RI needs taxTypeId for IVA breakdown)
  const detail: Record<string, unknown> = { description, quantity: 1, unitPrice: amount };
  if (isRI) {
    detail.taxTypeId = 94; // 21% — fetch /v1/combos/9/items for full list
  }

  // 5. Create receipt (auto-submits to AFIP)
  const { data: receipt, error } = await client.sales.receipts.create({
    customerId: customer.id,
    receiptBookId: book.id,
    date: new Date().toISOString().split("T")[0],
    saleConditionId: 1,
    details: [detail],
  });
  if (error) throw new Error(`Create failed: ${JSON.stringify(error)}`);

  return receipt; // poll receipt.status until "Creada" for confirmed CAE
}
```

---

## Error Handling Patterns

```typescript
const { data, error, response } = await client.sales.receipts.create(payload);

if (error) {
  if (response.status === 422 && error.afipError) {
    // AFIP rejected — log and retry with corrections
    console.error("AFIP error:", error.afipError.code, error.afipError.description);
  } else if (response.status === 400) {
    // Validation error
    console.error("Validation:", error.errors);
  } else {
    throw new Error(`Unexpected error ${response.status}: ${JSON.stringify(error)}`);
  }
}
```
