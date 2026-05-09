# API Entities Reference

Field-level reference for the three core entities agents interact with most: **Customer**, **Product**, and **Sales Receipt**.

---

## Customer

A customer (cliente) is the buyer that appears on sales receipts. Each customer belongs to one organization and is identified by their business name and tax ID.

### Key fields

| Field | Type | Description |
|---|---|---|
| `id` | `integer` | Numeric ID — use this when creating receipts |
| `businessName` | `string` | Legal/business name |
| `taxId` | `string \| null` | CUIT/CUIL/DNI as raw digits (e.g. `"30123456789"`) |
| `taxCategoryId` | `integer` | IVA condition ID → resolve via `GET /v1/tax-categories`. `discriminates: true` = Responsable Inscripto (Factura A); `discriminates: false` = CF/Monotributista (Factura B/C) |
| `documentTypeId` | `integer` | Document type ID (combo 2). Inferred from `taxId` if omitted |
| `email` | `string \| null` | Default email for sending receipts |
| `phone` | `string \| null` | Phone number |
| `address` | `string \| null` | Street address |
| `city` | `string \| null` | City |
| `postalCode` | `string \| null` | Postal code |
| `saleConditionId` | `integer \| null` | Default sale condition for this customer (overrides org default) |
| `isActive` | `boolean` | Inactive customers cannot be used on new receipts |
| `providerId` | `string \| null` | External identifier for idempotent sync from your system |

### Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/customers` | List customers (paginated). Query: `search` (partial name/CUIT), `page_size`, `cursor` |
| `GET` | `/v1/customers/{id}` | Get customer by ID |
| `GET` | `/v1/customers/lookup-cuit?cuit={cuit}` | Look up CUIT in AFIP/ARCA to auto-fill name, tax category, and address |
| `POST` | `/v1/customers` | Create customer |
| `PATCH` | `/v1/customers/{id}` | Update customer |
| `DELETE` | `/v1/customers/{id}` | Soft-delete customer |

### Creating a customer

```bash
curl -X POST "https://api.peakly.ar/v1/customers" \
  -H "X-API-Key: pk_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "businessName": "Distribuidora Norte SRL",
    "taxId": "30345678901",
    "taxCategoryId": 1,
    "email": "pagos@dnorte.com.ar"
  }'
```

**Auto-fill from CUIT:** if you pass only `taxId` (a valid 11-digit CUIT), Peakly fetches the business name and tax category from AFIP automatically — no need to pass those fields manually.

```bash
curl -X POST "https://api.peakly.ar/v1/customers" \
  -H "X-API-Key: pk_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"taxId": "30345678901"}'
```

**Shortcut on receipt creation:** pass `customerTaxId` instead of `customerId` on `POST /v1/sales/sales-receipts` and Peakly will match or create the customer in one step.

### Search vs. lookup

- `GET /v1/customers?search=ACME` — searches existing customers in the organization by name or CUIT (partial match).
- `GET /v1/customers/lookup-cuit?cuit=30123456789` — queries AFIP directly for a CUIT even if the customer doesn't exist yet; returns canonical name and tax category.

---

## Product

A product (producto) is a reusable catalog entry that can be linked to receipt detail lines. Products are optional — detail lines can use `description` instead of a `productId` — but using products enables barcode lookup, VAT defaults, and reporting by product.

### Key fields

| Field | Type | Description |
|---|---|---|
| `id` | `integer` | Numeric ID — pass as `productId` in receipt detail lines |
| `description` | `string` | Product name / description (max 500 chars) |
| `barcode` | `string \| null` | EAN or other barcode |
| `unitPrice` | `number \| null` | Default unit price; overridden per line at receipt time |
| `vatRateId` | `integer \| null` | Default IVA rate combo item ID (combo 9). Overridden by `taxTypeId` on the receipt line if provided |
| `unitOfMeasureId` | `integer` | Unit of measure ID (combo 14, e.g. "Unidad", "Kg", "Hora") |
| `categoryId` | `integer \| null` | Product category ID (`GET /v1/product-categories`) |
| `isActive` | `boolean` | Inactive products cannot be added to new receipts |
| `isFavorite` | `boolean` | Marked as favorite for quick access |
| `quantity` | `number \| null` | Current stock quantity (when stock tracking is enabled) |
| `providerId` | `string \| null` | External identifier for idempotent sync |

### Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/products` | List products (paginated). Query: `name` (partial match), `page_size`, `cursor` |
| `GET` | `/v1/products/search` | Smart search with favorites, recents, and fuzzy matching. Query: `q`, `category_id`, `limit` |
| `GET` | `/v1/products/{id}` | Get product by ID |
| `POST` | `/v1/products` | Create product |
| `PATCH` | `/v1/products/{id}` | Update product |
| `DELETE` | `/v1/products/{id}` | Soft-delete product |

### Searching products

For agent use, prefer `GET /v1/products/search` — it returns three ranked lists in one call:

```bash
curl "https://api.peakly.ar/v1/products/search?q=consulting&limit=10" \
  -H "X-API-Key: pk_your_api_key_here"
```

```json
{
  "favorites": [ ... ],
  "recents": [ ... ],
  "results": [ ... ]
}
```

### Using a product on a receipt

Pass `productId` on a detail line to link the catalog entry. If `unitPrice` is omitted, the product's default price is used; `taxTypeId` falls back to the product's `vatRateId`.

```json
{
  "details": [
    {
      "productId": 17,
      "quantity": 2,
      "unitPrice": 5000.00
    }
  ]
}
```

If you pass `description` without `productId`, Peakly matches an existing product by exact description (case-insensitive) or creates a new one on the fly.

---

## Sales Receipt (Comprobante de Venta)

A sales receipt is the central fiscal document created when invoicing a customer. It maps to an Argentine fiscal comprobante (Factura A/B/C/E, Nota de Crédito, Nota de Débito).

### Key fields

| Field | Type | Description |
|---|---|---|
| `id` | `string` (UUID) | Unique receipt identifier — use this in all path params |
| `number` | `string` | Formatted receipt number, e.g. `"00001-00000123"` (assigned on confirm) |
| `letter` | `string` | Factura letter: `"A"`, `"B"`, `"C"`, `"E"`, `"M"`, etc. |
| `status` | `string` | Lifecycle status — see table below |
| `date` | `string` (ISO 8601) | Invoice date |
| `dueDate` | `string` (ISO 8601) | Payment due date |
| `customerId` | `integer` | Customer ID |
| `receiptBookId` | `integer` | Receipt book used |
| `typeId` | `integer` | Receipt type combo item ID (combo 5: Factura, NC, ND) |
| `receiptType` | `string` | Group: `"factura"`, `"nota_credito"`, `"nota_debito"` |
| `saleConditionId` | `integer` | Sale condition (payment terms) |
| `subtotal` | `number` | Net amount before IVA |
| `vat21` | `number` | IVA at 21% |
| `vat105` | `number` | IVA at 10.5% |
| `vat27` | `number` | IVA at 27% |
| `exemptAmount` | `number` | Exempt (Exento) amount |
| `nonTaxableAmount` | `number` | Non-taxable (No Gravado) amount |
| `total` | `number` | Total amount including IVA |
| `totalCollected` | `number` | Amount already collected against this receipt |
| `cae` | `string \| null` | 14-digit AFIP authorization code (`null` until authorized) |
| `caeExpiration` | `string \| null` | CAE expiration date (ISO 8601) |
| `observation` | `string \| null` | Free-text note (appears on PDF) |
| `serviceStartDate` | `string \| null` | Service period start (required for some organizations) |
| `serviceEndDate` | `string \| null` | Service period end |
| `currencyId` | `integer \| null` | Currency ID (`null` = ARS) |
| `exchangeRate` | `number \| null` | Exchange rate to ARS |
| `providerId` | `string \| null` | External identifier for idempotent creation |
| `errorMessage` | `string \| null` | Last AFIP error message (populated when authorization fails) |
| `details` | `array` | Line items — see detail fields below |
| `customer` | `object` | Embedded customer snapshot (name, taxId, address, etc.) |

### Status values

| Status | Meaning |
|---|---|
| `Borrador` | Draft — not yet submitted to AFIP. Call `POST /v1/sales/sales-receipts/{id}/confirm` to proceed. |
| `Pendiente_Afip` | Submitted to AFIP, waiting for response (usually seconds). |
| `Creada` | Authorized — AFIP returned a `cae`. Fiscally valid. |
| `Cobrada` | Collected — marked as fully paid (used for accounting reconciliation). |
| `Anulado` | Voided. |

### Detail line fields

| Field | Type | Description |
|---|---|---|
| `id` | `integer` | Line ID |
| `description` | `string` | Line description |
| `quantity` | `number` | Quantity |
| `unitPrice` | `number` | Unit price |
| `total` | `number` | Line total (quantity × unitPrice before IVA) |
| `totalWithVat` | `number \| null` | Line total including IVA |
| `taxTypeId` | `integer \| null` | IVA combo item ID (combo 9) — required on Factura A |
| `productId` | `integer \| null` | Linked product ID |
| `discount` | `integer \| null` | Line discount % (0–100) |
| `discountAmount` | `number \| null` | Line discount as absolute ARS amount |

### Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/sales/sales-receipts` | List receipts (paginated). Filters: `date_from`, `date_to`, `status`, `customer_id`, `receipt_type`, `search`, `receipt_book_id` |
| `GET` | `/v1/sales/sales-receipts/{id}` | Get receipt by ID |
| `GET` | `/v1/sales/sales-receipts/next-number` | Preview next number. Requires: `receipt_book_id`, `customer_id` |
| `POST` | `/v1/sales/sales-receipts` | Create receipt (auto-submits to AFIP unless `isDraft: true`) |
| `PATCH` | `/v1/sales/sales-receipts/{id}` | Update draft receipt (only `Borrador` receipts) |
| `DELETE` | `/v1/sales/sales-receipts/{id}` | Delete draft receipt (only `Borrador`) |
| `POST` | `/v1/sales/sales-receipts/{id}/confirm` | Confirm draft → triggers AFIP submission |
| `POST` | `/v1/sales/sales-receipts/{id}/authorize` | Retry AFIP authorization for a stuck `Pendiente_Afip` receipt |
| `POST` | `/v1/sales/sales-receipts/{id}/void` | Void an authorized receipt |
| `POST` | `/v1/sales/sales-receipts/{id}/duplicate` | Create a copy of a receipt as a new draft |
| `POST` | `/v1/sales/sales-receipts/{id}/send-email` | Send receipt PDF to customer by email |
| `GET` | `/v1/sales/sales-receipts/{id}/pdf` | Get PDF download URL |

### Minimal create payload

```json
{
  "customerId": 42,
  "receiptBookId": 1,
  "saleConditionId": 1,
  "details": [
    {
      "description": "Service fee",
      "quantity": 1,
      "unitPrice": 10000.00
    }
  ]
}
```

`date` defaults to today. `isDraft` defaults to `false` (auto-authorize).

### Idempotent creation

Pass `providerId` (your own external ID) to safely retry creation without creating duplicates:

```json
{
  "customerId": 42,
  "receiptBookId": 1,
  "saleConditionId": 1,
  "providerId": "order-2026-00423",
  "details": [...]
}
```

A duplicate `providerId` returns `PROVIDER_ID_DUPLICATE` (409). Alternatively, use the `Idempotency-Key` request header for one-time deduplication.
