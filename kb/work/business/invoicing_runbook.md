# Invoicing Runbook — Fortnox

**Scenario:** Invoice AFRY (Sweden, domestic B2B).  
**Frequency:** Monthly (net 30 unless SOW differs).

## 1) Create the invoice (Fortnox)
- Customer card: AFRY; verify orgnr & VAT no. (momsnr).  
- Add line(s): hours × rate (SEK), project/PO ref if required.  
- Terms & footer: add **“Godkänd för F-skatt”** (required on every invoice).  
  Ref: https://verksamt.se/avtal-fakturering/offert-och-anbud/visa-godkand-for-f-skatt

## 2) Legal content (Momslagen/fakturering)
Must include: date, unique number, seller & buyer names/addresses, seller VAT no., description/quantity, taxable base per rate, VAT rate(s), VAT amount.  
Ref: https://www.skatteverket.se/foretag/moms/saljavarorochtjanster/momslagensregleromfakturering.4.58d555751259e4d66168000403.html

## 3) Send & archive
- Send e-invoice/PDF as required by AFRY.  
- Save the PDF to `company/invoices/YYYY-MM/AFRY-YYYYMM.pdf` (your repo or secure store).

## 4) Payment & reconciliation
- When paid, reconcile bank transaction in Fortnox.  
- If foreign customers later: validate EU VAT nos. in VIES via Skatteverket.  
  https://www.skatteverket.se/foretag/moms/saljavarorochtjanster/forutsattningarformoms/eu-och-internationell-handel/foretags-kopare-inom-eu/vies.4.6d02084411db6e252fe80009412.html

## Notes for non-domestic cases (for future)
- **EU B2B services (huvudregeln):** reverse charge in buyer’s state; your invoice zero-rates with “Reverse charge” note; report in **periodisk sammanställning** if applicable.  
- **Export (outside EU):** typically outside scope of Swedish VAT; keep evidence of place of supply.  
Check: https://skatteverket.se/foretag/moms
