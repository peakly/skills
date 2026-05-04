# Argentine Invoicing Concepts

This guide explains the Argentine fiscal and invoicing system so your agent can make correct decisions when creating and managing receipts through the Peakly API.

---

## AFIP

**AFIP** (Administración Federal de Ingresos Públicos) is Argentina's federal tax authority — the equivalent of the IRS (USA) or HMRC (UK).

Every fiscal invoice issued in Argentina must be authorized by AFIP before it is legally valid. This authorization happens in real time via AFIP's web service and results in a **CAE**.

When creating a sales receipt through Peakly, the authorization step calls AFIP's servers on your behalf. If AFIP is unavailable, the receipt stays in `pending` status until it can be resubmitted.

---

## CAE (Código de Autorización Electrónico)

The **CAE** is a numeric code AFIP returns when it approves a fiscal document. A receipt without a CAE is **not a valid fiscal invoice** — it is only a draft.

- Format: 14-digit number, e.g. `71827364918273`
- Expiry: each CAE has a `caeDueDate` — typically 10 days after issuance
- Printed on invoices: the CAE and its due date appear on the PDF

In the Peakly API, a receipt has `cae: null` until authorized. After calling `/v1/sales/sales-receipts/{id}/authorize`, the response includes `cae` and `caeDueDate`.

---

## Factura Types

The type of invoice to issue depends on the **IVA condition** of both parties.

| Type | When to use | Notes |
|---|---|---|
| **Factura A** | Seller is `responsable_inscripto`, buyer is also `responsable_inscripto` | Net + IVA itemized separately |
| **Factura B** | Seller is `responsable_inscripto`, buyer is `consumidor_final` or `monotributista` | Price includes IVA |
| **Factura C** | Seller is `monotributista` | No IVA charged |
| **Factura E** | Export invoices | Special rules, currency in USD or other |
| **Nota de Crédito** | Reversal of a prior invoice | Same type as original (A/B/C/E) |
| **Nota de Débito** | Adjustment increasing the original amount | Less common |

**Decision rule for agents:**

```
if customer.ivaCondition == "responsable_inscripto":
    use Factura A receipt book
elif customer.ivaCondition in ["consumidor_final", "monotributista"]:
    use Factura B or C receipt book
elif isExport:
    use Factura E receipt book
```

The `receiptType` field on the receipt book (talonario) tells you which factura type it produces.

---

## IVA (Impuesto al Valor Agregado)

IVA is Argentina's value-added tax — equivalent to VAT (EU) or GST (Australia).

Standard rates:
- **21%** — general rate (most services and goods)
- **10.5%** — reduced rate (some food, medicine, construction)
- **0%** — exempt (some agricultural goods, exports)

For **Factura A**, each line item specifies its IVA type via `taxTypeId` (a combo item ID from `GET /v1/combos/9/items`). For **Factura B**, IVA is embedded in the total price and AFIP extracts it automatically.

---

## CUIT / CUIL

**CUIT** (Clave Única de Identificación Tributaria) is the Argentine tax ID for companies and self-employed individuals.  
**CUIL** (Clave Única de Identificación Laboral) is the equivalent for employees.

Format: `XX-XXXXXXXX-X` (2 + 8 + 1 digits), e.g. `30-12345678-9`.

- Starts with `20` or `23` — individual (persona física)
- Starts with `27` — married woman (historical)
- Starts with `30` or `33` — company (persona jurídica)

CUIT is required on all Factura A receipts. The Peakly API validates CUIT format and can look up customer data from AFIP using `/v1/customers/lookup-cuit`.

---

## Receipt Book (Talonario / Punto de Venta)

A **receipt book** (talonario) is a pre-registered sequence of invoice numbers associated with a **point of sale** (punto de venta, PdV).

Key properties:
- `pointOfSale` — number registered with AFIP (e.g., `0001`, `0003`)
- `receiptType` — the factura type (A, B, C, E)
- `lastNumber` — the last issued invoice number

Invoice numbers are sequential and registered with AFIP — they cannot be skipped or reused. When you create a receipt, Peakly assigns the next sequential number automatically. Use `GET /v1/sales/sales-receipts/next-number?receiptBookId={id}` to preview the next number before creating.

Most organizations have separate talonarios for Factura A and Factura B.

---

## Organization vs. Branch (Sucursal)

A Peakly **organization** corresponds to a single Argentine tax entity (a CUIT). It can have multiple **branches** (sucursales), each with its own address and potentially its own points of sale.

Your API key is scoped to one organization. If you need to issue receipts from a specific branch, pass `branchId` when creating receipts (optional — defaults to the main branch).

---

## Sale Condition (Condición de Venta)

The **sale condition** specifies the payment terms:

| Typical ID | Description |
|---|---|
| `1` | Contado (cash/immediate) |
| `2` | 30 days |
| `3` | 60 days |
| `4` | 90 days |

Fetch the list: `GET /v1/combos/6/items`. The `id` field goes into `saleConditionId` on the receipt.

---

## Receipt Lifecycle

```
draft  →  authorized (CAE received)  →  confirmed
                                           ↓
                                        anulado (voided)
                                        or
                                     nota de crédito (credit note)
```

- `draft` — created but not yet sent to AFIP
- `authorized` — AFIP returned a CAE; fiscally valid
- `confirmed` — marked as sent/delivered to customer
- `anulado` — voided (only same-period receipts; cross-period requires a credit note)

Agents should check `status` and `cae` to know whether a receipt is legally valid before referencing it.
