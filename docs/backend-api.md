# Backend HTTP contract

The n8n workflow talks to a small Flask service (default `http://host.docker.internal:8080`, because n8n itself runs in a container). The service is not part of this repository; this document specifies what the workflow expects from it.

All endpoints accept and return JSON, and are called with `POST` unless stated otherwise.

## Common types

**Offer**

| Field | Type | Notes |
|---|---|---|
| `titre` | string | Job title |
| `entreprise` | string | Hiring company |
| `remuneration` | string | Free text, e.g. `"45 000 - 55 000 €"`; `"A négocier"` when absent |
| `lieu` | string | Location |
| `description` | string | Short description (the workflow truncates it before the LLM call) |
| `url` | string | Link to the original ad |

**Region number** — the workflow sends the region as a string `"1"` … `"12"` (12 French regions). The backend maps it to a folder name such as `"2. BRETAGNE"`.

## Scraping

| Endpoint | Body | Response |
|---|---|---|
| `/scrape` (APEC) | `{ "url": "<search url>" }` | `{ "success": true, "total": 42, "offers": [Offer] }` |
| `/scrape-hellowork` | `{ "url", "region_num" }` | same |
| `/scrape-ouestfrance` | `{ "url", "region_num" }` | same |
| `/scrape-meteojob` | `{ "url", "region_num" }` | same |
| `/scrape-cadreemploi` | `{ "url", "region_num" }` | same |

Errors: `400 {"error": "URL manquante"}` when the URL is missing, `500 {"error": "...", "success": false}` on scraping failure.

Implementation notes: HelloWork, Ouest-France Emploi, Meteojob and Cadremploi are rendered with Playwright (JavaScript-driven pages) and parsed with BeautifulSoup; APEC uses its JSON search API.

## State & checks

### `POST /check-keep`

Monthly memory per region, to avoid surfacing the same company twice.

```json
// request
{ "company": "ACME", "salary": "45 000 €", "region": "2" }

// response
{ "keep": true,  "reason": "new company this month" }
{ "keep": true,  "reason": "salary changed" }
{ "keep": false, "reason": "already kept this month" }
```

The state is one JSON file per region and month. Writes are protected by a per-region lock and are atomic (temporary file + rename, retried), so parallel runs cannot corrupt it.

### `POST /check-client` *(optional)*

```json
// request
{ "company": "ACME" }
// response
{ "is_client": false }
{ "is_client": true, "matched": "ACME SAS" }
```

Looks the company up in a spreadsheet of existing clients, using accent-, case- and whitespace-insensitive matching.

## Output

### `POST /download-pdf-batch`

```json
// request
{
  "region_num": "2",
  "annonces": [
    { "entreprise": "ACME", "titre": "Responsable RH", "url": "https://...", "remuneration": "50 000 €" }
  ]
}

// response
{ "success": true, "total": 12, "pdfs_ok": 11, "errors": 1, "details": [ ... ] }
```

Renders each ad to an A4 PDF (first 2 pages) using Playwright with **3 parallel workers**, and files it under `Annonces_PDF/<region>/<YYYY-MM>/<company> - <salary> - <title>.pdf`.

> The workflow node uses *Execute Once* and sends the whole list in a single call.

### `POST /create-report` *(optional)*

```json
{ "region": "2. BRETAGNE", "analysed": 120, "kept": 14, "pdfs": 14 }
```

Stores the latest run summary shown on the report page.

## Progress & UI

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | Liveness check |
| `GET` | `/progress` | Current step, number of offers analysed, PDFs done |
| `POST` | `/progress/update` | Called by the workflow to update the progress state |
| `GET` | `/progression` | Live progress page (HTML) |
| `GET` | `/report` | Latest run report page (HTML) |

## Host helper (optional)

A tiny service on the **host** (port `9999`) lets the containerised workflow open the progress page in the user's browser: `GET /open/progression`, `GET /open/report`.
