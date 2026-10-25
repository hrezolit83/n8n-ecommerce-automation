# n8n E-commerce Automation Portfolio — Vitaliy Dyedukh

12 production n8n workflows I built for a German e-commerce company that sells on 12+ European marketplaces (Otto, Leroy Merlin / Adeo, Conforama, Worten, Altex, Empik, CDON and more), with **Odoo** as the central ERP.

All credentials, company names, hosts and IDs are removed. Workflows are importable into n8n (Community Edition is enough): **Workflows → Import from File**, then connect your own credentials.

| # | Workflow | Nodes | Key techniques |
|---|---|---|---|
| 01 | Marketplace Autoposting (main): Odoo → Marketplace | 76 | Orchestrator + sub-workflows, attribute matching, image pipeline, compliance filter, stage routing |
| 02 | Marketplace Orders → Odoo Sales Orders | 35 | Customer deduplication, SKU matching, order total check, sale order via JSON-RPC |
| 03 | Stock & Price Sync: Odoo → Mirakl | 31 | Batch loops, diff-only updates, multi-channel pricing, rate-limit waits |
| 04 | Tracking Numbers → Marketplace (order fulfillment) | 14 | Sub-workflow triggered by Odoo, delayed read of shipment data, website and carrier validation, fulfillment via REST API |
| 05 | AI Category Mapper | 19 | LLM classification against a category tree, structured output parser |
| 06 | AI Required Attributes Filler | 18 | LLM + marketplace value lists, strict prompt rules, fallbacks |
| 07 | Auto-Reconciliation: Bank Payments ↔ Odoo Invoices | 29 | Odoo accounting API, matching logic, Bitrix24 checklist |
| 08 | Lost Orders Monitoring (all marketplaces) | 22 | Health checks per channel, alert deduplication |
| 09 | Product Listing via XML API (Hood.de) | 61 | XML request building/parsing, pagination, taxes, images |
| 10 | AI Category Selection with Fallback | 20 | Two-step LLM (OpenAI + Grok), Google Sheets manual fallback |
| 11 | Multi-language Listing: Odoo → CDON (Nordic) | 59 | Main orchestrator + sub-workflows, per-locale validation (DK/SE/FI), GPSR and chemicals checks, currency conversion, stage routing |
| 12 | Pending → Active Offer Activation | 51 | Offer import, status polling, stage updates in Odoo |

---

## Case 1 — Main autoposting pipeline: Odoo → marketplace listing

**Problem.** Each marketplace has its own category tree, required attributes and content rules. Listing hundreds of products by hand took days, and every product was rejected for a different reason.

**Solution.** The main workflow orchestrates the whole listing process. It takes a product from the Odoo task queue and:
1. Checks whether the product is already listed and whether its offer is LIVE, so nothing is posted twice.
2. Loads product data, properties, material and prices from Odoo and groups content per language.
3. Calls sub-workflows for category selection and required-attribute matching, routing unexpected categories and unresolved attributes to their own Odoo stages.
4. Replaces words forbidden by marketplace rules and builds the image set through an image-upload sub-workflow.
5. Imports the product with its offer, polls the import status in batches and runs two levels of error checks.
6. Moves the Odoo task to Pending, Active, Error or Unexpected Category with a readable reason.

**Result.** Listing became fully automatic: products move from "New task" to "Pending"/"Active" without manual work; every rejection has a readable reason.

## Case 2 — Marketplace orders into the ERP

**Problem.** Orders from a Nordic marketplace (DK/SE/FI) had to be entered into Odoo manually, including customers, addresses and prices.

**Solution.** Every 30 minutes the workflow fetches new orders and, for each one:
1. Skips orders that already exist in Odoo (by source order number).
2. Finds the country and the right pricelist, then searches for the customer by email or phone, address and name; creates a new contact only if no exact match exists.
3. Matches order lines to Odoo products by SKU or legacy ID, and checks that the sum of lines equals the order total before creating anything.
4. Creates and confirms the sales order on the right website. (The invoice steps at the end are disconnected: invoicing was later moved to Odoo's own automatic invoicing settings.)
5. Stops with a clear error if products are missing or totals do not match, instead of creating broken records.

**Result.** Orders appear in Odoo automatically, without duplicate customers or wrong totals.

## Case 3 — Stock & price sync across 4 country channels

**Problem.** Prices and stock for thousands of offers on a Mirakl marketplace (FR/ES/PT/IT channels) had to stay in sync with Odoo pricelists.

**Solution.** The workflow loads offers from the marketplace in pages, loads matching Odoo products, compares them per channel and sends only changed offers in batches, with waits to respect rate limits.

**Result.** Thousands of offers are updated in minutes, and only real changes hit the API.

## Case 4 — Tracking numbers to the marketplace

**Problem.** After shipping, every order had to be confirmed on the marketplace with the carrier and tracking number, otherwise the seller rating dropped.

**Solution.** When a delivery is validated in Odoo, a router calls this sub-workflow with the shipment ID. It:
1. Waits briefly, because the label-creation process fills in the tracking number and carrier a moment later.
2. Reads the shipment from Odoo (JSON-RPC): tracking number, carrier, marketplace order number and website.
3. Checks that the shipment belongs to one of the marketplace's country stores (DK/SE/FI).
4. Maps the Odoo carrier name to the marketplace's list of supported carriers.
5. Sends the fulfillment with tracking information to the marketplace API, or stops with a clear error if the website or carrier is wrong.

**Result.** Orders are confirmed with tracking automatically, a minute after shipping, without manual work.

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

## Case 11 — Multi-language listing on a Nordic marketplace

**Problem.** Products had to be listed on a Nordic marketplace in several languages (DK/SE/FI) with its own API, prices in local currencies and strict content and safety rules.

**Solution.** Listing is split into several connected workflows. This is the main one: it receives one product at a time, collects all data and calls sub-workflows for categories, attributes and images. It:
1. Checks whether the product already exists on the marketplace and in which status (active, paused, inaccessible).
2. Collects product data for every locale from Odoo and validates each one: title and description length, barcode, price, brand allow-list.
3. Runs compliance checks: GPSR data, main image, stock, delivery tags, and blocks chemical products without a safety PDF.
4. Converts prices to local currencies and builds the article payload with images.
5. Sends the product to the marketplace and moves the Odoo task to the right stage with a readable reason.

**Result.** Products are listed in all Nordic locales automatically; every rejection has a clear reason in Odoo.

## Case 12 — Pending → Active activation

For products that became LIVE on the marketplace, the workflow creates offers in bulk, polls the import status until it is complete and moves the Odoo tasks to Active.

---

**Stack:** n8n, Odoo JSON-RPC, Mirakl API, Otto API, Bitrix24, OpenRouter / xAI Grok / OpenAI, JavaScript (Code nodes).
**Contact:** [LinkedIn](https://www.linkedin.com/in/vitaliy-dyedukh-11419b2a9/) · [Telegram](https://t.me/hermit83) · deduh83@gmail.com
