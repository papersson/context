# Fortnox Checklist — Persson Tech AB (Lean, Solo-AB)

> Keep this to one page. Tick through it monthly; expand only if your setup changes.

## 0) One-time Setup
- [ ] **Company settings**: Fiscal year, VAT period, org no., VAT no. (SE[orgnr]01)
- [ ] **Bank**: Connect business account; enable bank feeds (if available)
- [ ] **Number series**: Define series for customer/supplier invoices
- [ ] **VAT period & codes**: Period = Monthly; confirm reverse-charge mappings (Box 22/30/48)
- [ ] **Currencies**: Enable USD (for Anthropic/other US suppliers)
- [ ] **User**: Only you; ensure two-factor auth
- [ ] **Document templates**: Invoice footer includes “Godkänd för F-skatt”
- [ ] **Products/Articles** (optional): Create “Consulting hour” w/ price 750 SEK/h

## 1) Customer Invoicing (AFRY, Sweden)
- [ ] **Create customer** “AFRY” with org/VAT no., billing email/portal ID if needed
- [ ] **New invoice**: Hours × rate, PO/reference if required
- [ ] **Footer**: “Godkänd för F-skatt” present
- [ ] **Terms**: Net 30 (or as per SOW)
- [ ] **Send**: E-invoice or PDF as required
- [ ] **Archive**: Save PDF to `company/invoices/YYYY-MM/AFRY-YYYYMM.pdf`

## 2) Supplier Invoices — Non-EU SaaS (Claude Code)
*(Reverse charge / Omvänd moms)*
- [ ] **Supplier** “Anthropic (US)”
- [ ] **Momstyp** = Reverse charge / “Omvänd skattskyldighet”
- [ ] **Currency** = USD; check SEK conversion
- [ ] **Accounts** (typical BAS; verify your mapping):
      - Debit **4531** Inköp av tjänster utanför EU, 25% (base)
      - Credit **2440** Leverantörsskulder
      - Credit **2614** Utg. moms omvänd skattskyldighet 25%
      - Debit **2645** Ing. moms på förvärv från utlandet
- [ ] **Attachment**: Upload the receipt/invoice PDF
- [ ] **Post** and verify VAT report shows base in **Box 22** and VAT in **Box 30**; input VAT in **Box 48**

## 3) Bank & Reconciliation (weekly or monthly)
- [ ] **Import/match** bank transactions (bank feed or CSV)
- [ ] **Match** customer payments ↔ invoices
- [ ] **Match** supplier payments ↔ supplier invoices
- [ ] **Unmatched** items investigated and posted (fees, FX differences)
- [ ] **Balance** 1930 (bank) equals statement

## 4) Expenses You Paid Privately (if any)
- [ ] **Create expense claim**: One per month
- [ ] **Attach** receipts; state purpose clearly
- [ ] **Post**: Debit expense (e.g., 6230/5410), Credit 2893 (liabilities to owner)
- [ ] **Reimburse**: Bank transfer → Debit 2893, Credit 1930

## 5) VAT Return (Momsdeklaration)
- [ ] **Review VAT report** for period
- [ ] **Reverse charge** totals present (Box 22/30/48)
- [ ] **Submit** VAT return at Skatteverket
- [ ] **Book payment/refund** on tax account when settled

## 6) Preliminary Tax (Debiterad F-skatt)
- [ ] **Calendar**: Pay by the 12th (moves on holidays)
- [ ] **Book** payment from bank to 1630 (Tax account) per your accounting policy

## 7) Month-End Close (lightweight)
- [ ] **Timesheet check** vs invoices sent
- [ ] **All receipts attached** in Fortnox
- [ ] **Bank reconciled** to statement date
- [ ] **Tag** period as closed; optional repo tag `close-YYYY-MM`

## 8) Year-End (overview)
- [ ] **AGM minutes** (within 6 months)
- [ ] **Annual report** filed to Bolagsverket (within 7 months)
- [ ] **Corporate tax return (INK2)** submitted per deadline

## Privacy & LLM Use (Claude Code)
- [ ] **Opt-out of training** in Claude Privacy settings (aim for 30-day retention)
- [ ] **No client secrets** or proprietary payloads in prompts
- [ ] **Redact** internal identifiers; use fixtures in examples

## Useful Fortnox Spots (where you’ll click)
- **Settings → Organization/Company**: Fiscal year, VAT period, VAT no.
- **Settings → Accounting → VAT**: VAT codes/mappings
- **Settings → Invoicing → Templates/Texts**: Footer “Godkänd för F-skatt”
- **Register → Customers/Suppliers**: AFRY, Anthropic
- **Invoicing → Customer Invoices**: Create/send/credit
- **Purchases → Supplier Invoices**: Register USD, reverse charge
- **Accounting → VAT Report**: Check boxes 22/30/48
- **Banking → Reconciliation**: Match transactions

### Notes
- Keep this file lean; if a step repeatedly confuses you, add a single bullet here instead of writing a new document.
