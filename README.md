# Update and improve Procurements Scraper — n8n + Firecrawl

As the main goal of the workflow, this page contains explaination of the improvments. It auto-discovers the number of pages, batches the listing pages in small chunks to avoid LLM truncation, polls Firecrawl for completion with retries.

> **Scope**: Listing pages only (cards) as in the old workflow extracts: `title`, `detail_page_url`, `published_date`, `status`, `deadline`.
Note: Description is removed to avoid the token turncated issue
---

> **Another solution**: I have built another n8n automation without **Firecrawl** works very well with the native API of the website [Mercell] using their API URL:
(`https://search-service-api.discover.app.mercell.com/public/api/v1/search?filter=delivery_place_code%3ASE&keywords=it&page={n}`).
The workflow is in the file [Procurements scraper without firecrawl v1.2.12.json]
![alt text](<Daigram Procurements scraper without firecrawl v1.2.12.png>)
---

## Notice that counts can vary across batches

1. **token isseue**  
   The LLM is overloaded with too much text to output at once. it silently drops items at the end of the  “tail”, causing incomplete results. Firecrawl's LLM has a maximum number of “tokens” (chunks of text) it can handle in its output window. When send two long listing pages in the same job, the combined data might be too big.

2. **cards vary / language**  
   Variation in markup and Swedish labels (e.g., **Publicerad**, **Sista anbudsdag**) can cause missed fields if the prompt/schema doesn’t normalize synonyms.

3. **slow content**  
   Times out while Firecrawl renders → fewer cards present at extract time.

---

## Workflow outlines

- Fetches `totalItems` and `pageSize` from Mercell’s search API to compute **`totalPages`**.
- Builds page URLs like `https://app.mercell.com/search?keywords=it&filter=delivery_place_code%3ASE&page={n}`.
- **Batches by 2 pages/job** posts to **Firecrawl Extract** with a lean schema.
- **Polls** `GET /v1/extract/{id}` until `status: "completed"`; retries on transient issues.
- **Flattens** all Firecrawl result shapes, **remove duplication** by `detail_page_url|title`, and **upserts** to Google Sheets.

---

## Improvements implemented

### 1) Dynamic job id & JSON response guarantee
**Node:** `Firecrawl – GET /extract/:id`  
- **URL expression:** `=https://api.firecrawl.dev/v1/extract/{{$json.id}}` (the current item carries `id`).
- **Header:** `Accept: application/json` for consistent JSON envelopes (even on errors).
- **Options:** *Ignore response code* = **ON** so retries can branch on failures instead of hard-stopping.

### 2) Auto pagination
New helper nodes:
- **Get First Page [to get the total pages]** → reads `numRes` & `pageSize`.
- **Compute Total Pages** → outputs `{ totalPages }` consumed by URL builder.
  
### 3) Smaller batches
**Node:** `Build URL list`  
- **Issue:** Larger chunks increase truncation risk.  
- **Fix:** **2 pages/job** by default.

### 4)  list flattener
**Node:** `Get list of procurements`  
**Problem:** Only looked at `payload.data[0]`, missing other pages or shapes.  
**Fix:** Flatten **all** root records and **all** entries in `data[]` (one per URL), then dedupe by `detail_page_url|title`.

### 5) Completion guard + retries
New/updated nodes:
- **Carry job id (Merge by position)** → merges the GET response with `{ id }` from `Pick job id`, ensuring retries still have `id`.
- **Complete and has itms** → IF condition:
  ```
  {{ $json.status === 'completed'
     && Array.isArray($json.data?.procurements)
     && $json.data.procurements.some(p => p && Object.keys(p).length > 0) }}
  ```
  - **True** → map + write to Sheets → signal next batch
  - **False** → wait (20s) → **retry GET** with same `id`

---

### 6) Firecrawl POST body

Remove `status` **optional** not required and **remove** field description.

```jsonc
 prompt: 'Extract EVERY procurement card visible on the page(s). Include the title, deadline, published date, URL to detail page. Do not skip any card. If a field is missing, return an empty string.'
```

---

## Nodes map diagram
![alt text](<Daigram Procurements Scraper with firecrawl v2.0.14.png>)
---

## How TO RUN the workflow
- Fetch the workflow you want to import, in n8n improt the workflow and make sure to replace the credential to Firecrawl and Google spreedshets.
- Make sure to replace the link '[YOUR_SPREADSHEET_ID]' with your spreadsheet at the node "write to sheets"
---

## Check the resuls under this link:
[spreedsheet result](https://docs.google.com/spreadsheets/d/1GK0M4PQMgcdkiA4h4SykACH_MFdg-VoHWsdUrM-uh5M/edit?usp=sharing)
## Future enhancements
- Add a **max-tries counter** and a fallback to **single-page jobs** if a 2-page batch returns suspiciously few items (e.g., `< 36`).
- Second-pass `description`, consider fetching it from each `detail_page_url` in a **second pass** to avoid list truncation.

