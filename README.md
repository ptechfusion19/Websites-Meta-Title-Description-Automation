# Websites Meta Title and Description-Automation

Meta Title & Description automation built in n8n.  
This workflow takes a website URL and crawls it with the DataForSEO OnPage API, extracts meta titles and descriptions, saves the data to Google Sheets and Google Drive, and emails the user a JSON report plus cost/credit usage information.

---

## What the workflow does

1. **Trigger (Webhook)**
   - Endpoint path: `MetaTitle-And-Description`
   - Expects a JSON body like:
     ```json
     {
       "website_link": "https://example.com/",
       "pages_to_crawl": 100,
       "email": "user@example.com"
     }
     ```
   - `website_link` can also be sent as `url` or `website`.
   - `pages_to_crawl` can also be sent as `max_crawl_pages` (defaults to 100, min 1, max 2000).

2. **Parse Webhook Input (Code node)**
   - Normalises the input and produces:
     - `domain` – hostname without protocol or `www`
     - `start_url` – full URL with protocol
     - `max_crawl_pages` – clamped between 1 and 2000
     - `email` – passed through for use by the email nodes

3. **Start On-Page Crawl (HTTP Request → DataForSEO OnPage)**
   - POSTs to `https://api.dataforseo.com/v3/on_page/task_post`
   - Uses **HTTP Basic Auth** credentials named `DataForSeo` in n8n.
   - Sends:
     ```json
     [
       {
         "target": "<domain>",
         "max_crawl_pages": <max_crawl_pages>,
         "start_url": "<start_url>"
       }
     ]
     ```

4. **Wait for Crawl**
   - Wait node: currently **2 minutes**.

5. **Check Tasks Ready (HTTP Request)**
   - GET `https://api.dataforseo.com/v3/on_page/tasks_ready`
   - Same `DataForSeo` credentials.

6. **Get Task Id (Code node)**
   - Reads:
     - Task created in **Start On-Page Crawl**
     - Ready tasks from **Check Tasks Ready**
   - Matches the correct task either by `id` or `target`.
   - Outputs:
     ```json
     {
       "task_id": "<on_page_task_id>",
       "target": "<target_url>"
     }
     ```

7. **Get On-Page Pages (HTTP Request)**
   - POST `https://api.dataforseo.com/v3/on_page/pages`
   - Body:
     ```json
     [
       {
         "id": "<task_id>",
         "limit": 1000,
         "offset": 0,
         "filters": [["resource_type", "=", "html"]]
       }
     ]
     ```

8. **Extract Meta Title & Description (Code node)**
   - Reads the `tasks[0].result[0].items` array from `Get On-Page Pages`.
   - Filters to `resource_type === "html"` and existing `meta`.
   - Maps each page to:
     ```json
     {
       "url": "<page url>",
       "meta_title": "<meta title or title tag>",
       "meta_description": "<meta description>",
       "status_code": <http_status_code>,
       "onpage_score": <onpage_score>
     }
     ```

9. **Save Data (Google Sheets)**
   - Appends each extracted row to a Google Sheet:
     - Document: **MetaTitle and Description** (`1B3MY4DeLz_pCEJeMG7_rJxk9kxIZG9pTUDuVYnvNJvo`)
     - Sheet: **Delta Solutions** (gid `0`)
   - Columns:
     - `Meta Title `
     - `Description`
     - `URL`

10. **Convert To JSON (Code node)**
    - Collects all extracted rows into one array.
    - Serialises to JSON.
    - Exposes it as a binary file called `file` on the current item (base64-encoded JSON).

11. **Upload file (Google Drive)**
    - Uploads the JSON as a file to a Google Drive folder:
      - Folder: `1S_R2qjWvxouW9Q2Riixo9wosZMG9IPTB`
    - File name pattern:
      - `meta-<sanitised-start_url>-<ISO-timestamp>.json`
    - Output from this node includes:
      - `name`
      - `webViewLink`
      - `webContentLink`
      - plus standard Drive file metadata.

12. **DataForSEO User Data (HTTP Request)**
    - GET `https://api.dataforseo.com/v3/appendix/user_data`
    - Same `DataForSeo` HTTP Basic credentials.
    - Used to read account balance, total deposited, and per-endpoint costs.

13. **Parse User Data (Code node)**
    - Reads `money.total` and `money.balance` from user_data:
      - `totalDeposited` – total funds ever deposited.
      - `currentBalance` – current remaining balance.
      - `totalSpent` – `totalDeposited - currentBalance`.
    - Reads the `cost` field from:
      - `Start On-Page Crawl`
      - `Check Tasks Ready`
      - `Get On-Page Pages`
      - `DataForSEO User Data`
    - Produces:
      ```json
      {
        "totalDeposited": <number>,
        "currentBalance": <number>,
        "totalSpent": <number>,
        "costs": {
          "startOnPageCrawl": <number>,
          "checkTasksReady": <number>,
          "getOnPagePages": <number>,
          "getUserData": <number>
        },
        "totalSessionCost": <sum of all costs>,
        "balanceAfterSession": <currentBalance - totalSessionCost>
      }
      ```

14. **Merge → Combine File & Spend**
    - `Merge` (Append) combines:
      - Drive file output from **Upload file**
      - Spend/balance output from **Parse User Data**
    - `Combine File & Spend` (Code) merges them into a single item:
      - All Drive fields plus:
        - `totalDeposited`
        - `currentBalance`
        - `totalSpent`
        - `costs.*`
        - `totalSessionCost`
        - `balanceAfterSession`

15. **Send JSON (SMTP Email)**
    - From: `automation@programmx.com`
    - To: `Parse Webhook Input.email`
    - Subject: `Meta Title and Description - <start_url>`
    - HTML body includes:
      - Website
      - Generated time
      - JSON file name
      - DataForSEO account summary:
        - Total deposited
        - Current balance
        - Total spent
      - This session’s API costs:
        - Start OnPage Crawl
        - Check Tasks Ready
        - Get OnPage Pages
        - Get User Data
        - Total session cost
        - Balance after session
      - Buttons/links to:
        - **View in Google Drive** (`webViewLink`)
        - **Download JSON file** (`webContentLink`)

16. **Send File (optional email)**
    - Separate email that links directly to the MetaTitle & Description Google Sheet.

---

## How to call the workflow

### Webhook URL

Assuming your n8n instance is available at `https://n8n.programmx.com`, the webhook URL is:

```text
https://n8n.programmx.com/webhook/MetaTitle-And-Description
```

### Example request

```bash
curl -X POST "https://n8n.programmx.com/webhook/MetaTitle-And-Description" \
  -H "Content-Type: application/json" \
  -d '{
    "website_link": "https://delta-solution.co.uk/",
    "pages_to_crawl": 100,
    "email": "you@example.com"
  }'
```

After the crawl and processing complete, you will:

- Get the meta title & description data appended to the configured Google Sheet.
- Receive an email with:
  - Links to the JSON report in Google Drive.
  - DataForSEO account balance and spend information for this session.

---

## Credentials required

To run this workflow end‑to‑end you need the following credentials configured in n8n:

- **HTTP Basic Auth** – `DataForSeo`
  - Used by:
    - `Start On-Page Crawl`
    - `Check Tasks Ready`
    - `Get On-Page Pages`
    - `DataForSEO User Data`
- **Google Sheets OAuth2** – `Google Sheets`
  - Used by `Save Data`.
- **Google Drive OAuth2** – `Google Drive account 4`
  - Used by `Upload file`.
- **SMTP** – `SMTP account 2`
  - Used by `Send JSON` and `Send File`.

Update the sheet IDs, Drive folder, and email addresses in the nodes if you copy this workflow to a different environment.
