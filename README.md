# n8n E-commerce Automation Portfolio — Vitaliy Dyedukh

12 production n8n workflows I built for a German e-commerce company that sells on 12+ European marketplaces (Otto, Leroy Merlin / Adeo, Conforama, Worten, Altex, Empik, CDON and more), with **Odoo** as the central ERP.

All credentials, company names, hosts and IDs are removed. Workflows are importable into n8n (Community Edition is enough): **Workflows → Import from File**, then connect your own credentials.

| # | Workflow | Nodes | Key techniques |
|---|---|---|---|
| 01 | AI Product Listing: Odoo → Marketplace | 90 | LLM agents, structured output, HMAC-signed API, pagination, image pipeline |
| 02 | Marketplace Orders → Odoo Sales Orders | 56 | OAuth2, pagination, customer deduplication, sub-workflows |
| 03 | Stock & Price Sync: Odoo → Mirakl | 31 | Batch loops, diff-only updates, multi-channel pricing, rate-limit waits |
| 04 | Tracking Numbers Sync + Alerts | 42 | Shipment aggregation, carrier validation, Bitrix24 error tasks |
| 05 | AI Category Mapper | 19 | LLM classification against a category tree, structured output parser |
| 06 | AI Required Attributes Filler | 18 | LLM + marketplace value lists, strict prompt rules, fallbacks |
| 07 | Auto-Reconciliation: Bank Payments ↔ Odoo Invoices | 29 | Odoo accounting API, matching logic, Bitrix24 checklist |
| 08 | Lost Orders Monitoring (all marketplaces) | 22 | Health checks per channel, alert deduplication |
| 09 | Product Listing via XML API (Hood.de) | 65 | XML request building/parsing, pagination, taxes, images |
| 10 | AI Category Selection with Fallback | 20 | Two-step LLM (OpenAI + Grok), Google Sheets manual fallback |
| 11 | Multi-language Listing with Compliance Filter | 91 | PT/ES content, forbidden-words filter, offer status checks |
| 12 | Pending → Active Offer Activation | 51 | Offer import, status polling, stage updates in Odoo |

---

## Case 1 — AI-powered product listing on a new marketplace

**Problem.** Each marketplace has its own category tree and required attributes. Listing hundreds of products by hand took days, and every product was rejected for a different reason.

**Solution.** A workflow that takes a product from an Odoo task queue and:
1. Validates mandatory fields (barcode, price, GPSR, brand, delivery tag) and routes failures to clear error reasons back in Odoo.
2. Uses an LLM agent with structured output to pick the right marketplace category and attribute set from the API.
3. Pulls the full attribute schema (with pagination) and lets the LLM fill required attributes using only allowed values.
4. Builds the image set, signs the request (HMAC-SHA512) and imports the product with its offer.
5. Handles duplicate / invalid-field / API errors separately and updates the task stage.

**Result.** Listing became fully automatic: products move from "New task" to "Pending" without manual work; every rejection has a readable reason.

## Case 2 — Marketplace orders into the ERP

**Problem.** Orders from several marketplaces had to be entered into Odoo manually, including customers and addresses.

**Solution.** A scheduled workflow fetches new orders (OAuth2 token, paginated API), checks if the order already exists, finds or creates the buyer and delivery contacts without duplicates, maps products by SKU and creates a sales order. Errors stop the order and are logged instead of creating broken records.

**Result.** Orders appear in Odoo automatically several times a day; no duplicate customers.

## Case 3 — Stock & price sync across 4 country channels

**Problem.** Prices and stock for thousands of offers on a Mirakl marketplace (FR/ES/PT/IT channels) had to stay in sync with Odoo pricelists.

**Solution.** The workflow loads offers from the marketplace in pages, loads matching Odoo products, compares them per channel and sends only changed offers in batches, with waits to respect rate limits.

**Result.** Thousands of offers are updated in minutes, and only real changes hit the API.

## Case 4 — Tracking numbers and error alerts

**Problem.** Shipped orders had to get tracking numbers on the marketplace, otherwise the seller rating dropped.

**Solution.** The workflow collects completed pickings in Odoo, matches them with marketplace orders, validates carrier tracking numbers (including return labels), sends shipping data and calls sub-workflows for dropshipping and freight. Any error creates or updates a task in Bitrix24 for the responsible developer, without duplicating tasks.

**Result.** Tracking is transferred automatically; problems reach a person within minutes as a task, not as a customer complaint.

## Cases 5–6 — AI enrichment of product data

**Problem.** Marketplaces rejected products for a missing category or required attributes (color family, product type, etc.) that the ERP did not have.

**Solution.** Two sub-workflows: one maps a product to the marketplace category tree with an LLM and a structured output parser; the other finds required attributes, loads their allowed value lists and asks the LLM to choose an exact value (no invented values).

**Result.** Used for ~1,900 products stuck in "Pending"; combined with other fixes, the rejected backlog was cleared in bulk.

## Case 7 — Automatic payment reconciliation

**Problem.** Accountants matched incoming bank payments to open invoices by hand.

**Solution.** On a schedule the workflow loads unreconciled bank statement lines and unpaid invoices from Odoo, matches them, reconciles matched pairs through the Odoo API and posts the invoices. Anything that cannot be matched goes to a Bitrix24 task as a checklist, with only new items added on each run.

**Result.** Routine payments are reconciled without a person; accountants only see the exceptions.

## Case 8 — Lost orders monitoring

**Problem.** If an order sync silently stopped on one marketplace, nobody noticed until customers complained.

**Solution.** A monitoring workflow checks every active marketplace channel in Odoo, flags channels with no recent orders and opens a Bitrix24 task for the responsible developer, adding a comment instead of a duplicate task if the problem is already known.

**Result.** Broken channels are detected within hours, not days.

## Case 9 — Listing on an XML-based marketplace API

**Problem.** Hood.de uses an XML API, unlike the REST/JSON APIs of other marketplaces.

**Solution.** The workflow builds XML requests from Odoo product data, reads the full category tree and existing inventory with pagination, checks if the product already exists, prepares images and taxes and posts the item. Errors move the Odoo task to an error stage with a readable message.

## Case 10 — AI category selection with human fallback

Two-step LLM classification (main category, then subcategory) with structured output. If the model is not confident, the workflow takes the category from a Google Sheet that a manager maintains, so the process never stops.

## Case 11 — Multi-language listing with compliance filter

Builds Portuguese and Spanish product content from Odoo, replaces words forbidden by marketplace rules, checks if an offer already exists and whether it is active, and routes every product to the right Odoo stage: Pending, Active, Error or Unexpected Category.

## Case 12 — Pending → Active activation

For products that became LIVE on the marketplace, the workflow creates offers in bulk, polls the import status until it is complete and moves the Odoo tasks to Active. Later this logic was rewritten as a Python service, cutting the run time from ~4 hours to 6 seconds.

---

**Stack:** n8n, Odoo JSON-RPC, Mirakl API, Otto API, Bitrix24, OpenRouter / xAI Grok / OpenAI, JavaScript (Code nodes).
**Contact:** [LinkedIn](https://www.linkedin.com/in/vitaliy-dyedukh-11419b2a9/) · [Telegram](https://t.me/hermit83) · deduh83@gmail.com
