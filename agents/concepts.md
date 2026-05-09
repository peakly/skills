# Argentine Invoicing Concepts

This guide explains the Argentine fiscal and invoicing system so your agent can make correct decisions when creating and managing receipts through the Peakly API.

---

## AFIP / ARCA

**AFIP** (Administración Federal de Ingresos Públicos) is Argentina's federal tax authority — the equivalent of the IRS (USA) or HMRC (UK). It has been rebranded as **ARCA** (Agencia de Recaudación y Control Aduanero) but both names are still used interchangeably.

Every fiscal invoice issued in Argentina must be authorized by AFIP before it is legally valid. This authorization happens in real time via AFIP's web service and results in a **CAE**.

When creating a sales receipt through Peakly, the authorization step calls AFIP's servers on your behalf. If AFIP is unavailable, the receipt stays in `Pendiente_Afip` status until it can be resubmitted.

---

## CAE (Código de Autorización Electrónico)

The **CAE** is a numeric code AFIP returns when it approves a fiscal document. A receipt without a CAE is **not a valid fiscal invoice**.

- Format: 14-digit number, e.g. `71827364918273`
- Expiry: each CAE has a `caeExpiration` date — typically 10 days after issuance
- Printed on invoices: the CAE and its expiration date appear on the PDF

In the Peakly API, a receipt has `cae: null` until authorized. After AFIP processes the receipt, `cae` and `caeExpiration` are populated.

---

## Factura Types

The type of invoice to issue depends on the **tax category** of both parties.

| Type | When to use | Notes |
|---|---|---|
| **Factura A** | Seller and buyer are both Responsable Inscripto (`discriminates: true`) | Net + IVA itemized separately per line |
| **Factura B** | Seller is RI, buyer is Consumidor Final or Monotributista | Price includes IVA |
| **Factura C** | Seller is Monotributista | No IVA charged |
| **Factura E** | Export invoices | Special rules, currency may be USD |
| **Nota de Crédito** | Reversal of a prior invoice | Same letter as original (A/B/C/E) |
| **Nota de Débito** | Upward adjustment to an original invoice | Less common |

**Decision rule for agents:**

```
taxCategories = GET /v1/tax-categories
category = taxCategories.find(c => c.id == customer.taxCategoryId)

if category.discriminates == true:
    use Factura A — include taxTypeId per detail line
elif category.isExport:
    use Factura E
else:
    use Factura B or C
```

A single receipt book (talonario) covers all letter types (A, B, C). The actual letter is assigned automatically when the receipt is created, based on the customer's tax category.

---

## IVA (Impuesto al Valor Agregado)

IVA is Argentina's value-added tax — equivalent to VAT (EU) or GST (Australia).

Standard rates (fetch current IDs from `GET /v1/combos/9/items`):

| Combo item ID | Rate | Notes |
|---|---|---|
| `94` | 21% | General rate (most services and goods) |
| `93` | 10.5% | Reduced rate (some food, medicine, construction) |
| `91` | Exento (0%) | Exempt goods |
| `92` | Sin IVA | Not applicable (Monotributistas, exports) |
| `90` | No Gravado | Non-taxable items |
| `95` | 27% | Utilities and telecoms |

For **Factura A**, each detail line specifies its IVA type via `taxTypeId`. For **Factura B**, IVA is embedded in the total price and AFIP extracts it automatically — no `taxTypeId` needed.

---

## Tax Categories

Customers have a `taxCategoryId` (numeric ID) that maps to their fiscal condition. Use `GET /v1/tax-categories` to resolve IDs to names.

Key fields in the response:
- `discriminates: true` — customer is Responsable Inscripto → use Factura A
- `discriminates: false` — customer is CF, Monotributista, or Exento → use Factura B or C
- `abbreviation` — short code, e.g. `"RI"`, `"CF"`, `"MO"`

---

## CUIT / CUIL

**CUIT** (Clave Única de Identificación Tributaria) is the Argentine tax ID for companies and self-employed individuals.  
**CUIL** (Clave Única de Identificación Laboral) is the equivalent for employees.

Format: `XX-XXXXXXXX-X` (2 + 8 + 1 digits), e.g. `30-12345678-9`. The API stores it as 11 raw digits (no dashes) in `taxId`.

- Starts with `20`, `23`, or `27` — individual (persona física)
- Starts with `30` or `33` — company (persona jurídica)

CUIT is required on all Factura A receipts. The Peakly API validates CUIT format and can auto-fill customer data from AFIP using `GET /v1/customers/lookup-cuit?cuit=30123456789`.

Alternatively, pass `customerTaxId` instead of `customerId` when creating a receipt — Peakly will match or create the customer automatically.

---

## Receipt Book (Talonario / Punto de Venta)

A **receipt book** (talonario) is a pre-registered sequence of invoice numbers associated with a **point of sale** (punto de venta, PdV) registered with AFIP.

Key properties:
- `pointOfSale` — number registered with AFIP (e.g. `1`, `3`)
- `prefix` — formatted PdV prefix (e.g. `"0001"`)
- `lastInvoiceA` / `lastInvoiceB` / `lastInvoiceC` — last issued number per letter type

Invoice numbers are sequential per letter type and cannot be skipped or reused. Peakly assigns the next number automatically when a receipt is confirmed. Preview the next number before creating:

```bash
GET /v1/sales/sales-receipts/next-number?receipt_book_id={id}&customer_id={id}
```

Response:
```json
{ "letter": "A", "number": "00001-00000124", "nextSequential": 124 }
```

Note: both `receipt_book_id` and `customer_id` are required — the letter depends on the customer's tax category.

---

## Organization vs. Branch (Sucursal)

A Peakly **organization** corresponds to a single Argentine tax entity (a CUIT). It can have multiple **branches** (sucursales), each with its own address and potentially its own points of sale.

Your API key is scoped to one organization. If you need to issue receipts from a specific branch, pass `branchId` when creating receipts (optional — defaults to the main branch).

---

## Sale Condition (Condición de Venta)

The **sale condition** specifies payment terms. Fetch the full list:

```bash
GET /v1/combos/6/items
```

Common values:

| Typical ID | Description |
|---|---|
| `1` | Contado (cash / immediate) |
| `2` | 30 days |
| `3` | 60 days |
| `4` | 90 days |

The `id` field goes into `saleConditionId` on the receipt.

---

## Receipt Lifecycle

```
Borrador (draft)
    ↓  POST /sales-receipts/{id}/confirm
Pendiente_Afip (waiting for AFIP)
    ↓  AFIP authorization (automatic)
Creada (authorized — has CAE, fiscally valid)
    ↓  optional: mark as collected
Cobrada (collected / paid)
    ↓  POST /sales-receipts/{id}/void
Anulado (voided)
```

- `Borrador` — created with `isDraft: true`; not yet sent to AFIP. Call `confirm` to trigger authorization.
- `Pendiente_Afip` — submitted to AFIP; awaiting response (usually seconds). Created receipts without `isDraft` start here.
- `Creada` — AFIP returned a CAE; receipt is fiscally valid.
- `Cobrada` — marked as fully collected (optional status, used for accounting reconciliation).
- `Anulado` — voided. Only `Creada` or `Cobrada` receipts can be voided.

**Shortcut for non-drafts:** creating a receipt without `isDraft: true` automatically confirms and submits to AFIP. No separate `confirm` or `authorize` call is needed.

**Retrying failed authorization:** if a receipt gets stuck at `Pendiente_Afip` due to an AFIP error, fix the underlying issue and call:

```bash
POST /v1/sales/sales-receipts/{id}/authorize
```

This returns `{ "success": true, "cae": "...", "caeExpiration": "..." }` on success.
