# Reverse Charge — Non-EU SaaS (e.g., Claude Code)

**Goal:** Book US SaaS in Fortnox so VAT boxes auto-fill correctly.

## What Swedish law expects (Skatteverket)
- You self-assess VAT on services bought from outside the EU (omvänd betalningsskyldighet).  
- Report **tax base** in **Box 22**; report **25% output VAT** in **Box 30** (if 25% rate applies); usually deduct the same as **input VAT** in **Box 48**.  
Refs:  
- Buying services outside EU: https://skatteverket.se/foretag/moms/kopavarorochtjanster/inkopfranlanderutanforeu/kopatjanserfranlanderutanforeu.4.361dc8c15312eff6fd33a46.html  
- VAT return fields: https://www.skatteverket.se/foretag/moms/deklareramoms/fyllaimomsdeklarationen.4.3a2a542410ab40a421c80004214.html  
- General: https://skatteverket.se/foretag/moms/kopavarorochtjanster/inkopfranlanderutanforeu.4.18e1b10334ebe8bc80001159.html

## Fortnox — one-time supplier setup
1) Create supplier **Anthropic** (country: US).  
2) On the supplier card, set **Momstyp** = **Omvänd skattskyldighet** (reverse charge).  
   Ref: https://support.fortnox.se/produkthjalp/fakturering/momstyp
3) If billing currency = USD, enable the currency in **Settings → Accounting/Banking/Fakturering → Valuta**.  
   Refs:  
   - Supplier invoice in foreign currency: https://support.fortnox.se/produkthjalp/bokforing/leverantorsfaktura-med-utlandsk-valuta  
   - Currency settings: https://support.fortnox.se/produkthjalp/fakturering/installningar-valuta

## Posting pattern (BAS — common setup)
- **Debit 4531** Inköp av tjänster från land utanför EU, 25% (tax base, SEK)  
- **Credit 2440** Leverantörsskulder (total)  
- **Credit 2614** Utgående moms omvänd skattskyldighet 25% (calculated)  
- **Debit 2645** Ingående moms på förvärv från utlandet (same amount if full deduction)
Examples/refs:  
- Fortnox guide (web services outside EU): https://www.fortnox.se/fortnox-foretagsguide/bokforingstips/webbhotell

## Currency conversion
- Convert to SEK using a reasonable spot rate per Skatteverket **“Omräkning av valuta.”**  
  https://www.skatteverket.se/foretag/drivaforetag/omrakningavvaluta.4.906b37c10bd295ff4880002832.html

## Claude Code — set privacy before use
- In **Privacy Settings**, **opt-out of training** to keep the 30-day retention; opt-in = up to **5 years** retention for new/resumed chats & coding sessions.  
  https://www.anthropic.com/news/updates-to-our-consumer-terms  
  https://privacy.claude.com/en/articles/10023548-how-long-do-you-store-my-data  
  https://docs.claude.com/en/docs/claude-code/data-usage
