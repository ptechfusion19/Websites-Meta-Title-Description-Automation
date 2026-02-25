## Setup

This document explains how to set up and run the **Meta Title & Description Automation** workflow and its HTML front‑end in a fresh environment (e.g. for a new GitHub clone).

---

## 1. Prerequisites

- **n8n instance**
  - Self‑hosted or cloud n8n where you can import workflows and configure credentials.
- **DataForSEO account**
  - Access to the **OnPage API**.
  - HTTP Basic Auth credentials (login + password) for API calls.
- **Google account**
  - Access to Google Sheets and Google Drive.
  - A target Google Sheet and a target Drive folder where results will be stored (you can reuse the ones from this repo or create your own).
- **SMTP email account**
  - An SMTP server and mailbox to send result emails from.
- **Static hosting for the front‑end (optional)**
  - Any static host (GitHub Pages, Netlify, Vercel, simple web server), or you can just open `index.html` directly in your browser for testing.

---

## 2. Files in this repo

- `Meta Title & Description Automation.json`  
  Exported n8n workflow that:
  - Receives a webhook request.
  - Calls DataForSEO OnPage to crawl a site.
  - Extracts meta titles and descriptions.
  - Saves rows into Google Sheets and a JSON file in Google Drive.
  - Sends an email with links plus DataForSEO cost/credit usage.

- `README.md`  
  Detailed functional description of each node in the workflow and the API calls.

- `index.html`  
  A small front‑end form that posts to the n8n webhook so non‑technical users can trigger the workflow.

---

## 3. Import the n8n workflow

1. Log in to your n8n instance.
2. Go to **Workflows → Import from file**.
3. Select `Meta Title & Description Automation.json` from this repo.
4. Once imported, open the workflow.
5. Confirm the **Webhook** node path is `MetaTitle-And-Description` (or adjust to your preference).
6. Click **Activate** to enable the workflow.

> The README in this folder explains what each node does in detail; consult it if you need to modify the behavior.

---

## 4. Configure credentials in n8n

Create or update the following credentials in n8n so they match the names used in the workflow:

- **HTTP Basic Auth – `DataForSeo`**
  - Used by:
    - `Start On-Page Crawl`
    - `Check Tasks Ready`
    - `Get On-Page Pages`
    - `DataForSEO User Data`
  - Set username and password from your DataForSEO account.

- **Google Sheets OAuth2 – `Google Sheets`**
  - Used by `Save Data`.
  - Connect to the Google account that owns the target sheet.

- **Google Drive OAuth2 – `Google Drive account 4`**
  - Used by `Upload file`.
  - Connect to the Google account that owns the target Drive folder.

- **SMTP – `SMTP account 2`**
  - Used by `Send JSON` (and `Send File` if you enable that email).
  - Configure SMTP host, port, username, password, and from‑address.

You can rename the credentials inside the workflow if your naming conventions differ; just make sure the nodes reference the updated credential names.

---

## 5. Update Google Sheet and Drive targets (if needed)

The workflow is currently configured with:

- **Google Sheet**
  - Document ID: `1B3MY4DeLz_pCEJeMG7_rJxk9kxIZG9pTUDuVYnvNJvo`
  - Sheet name: `Delta Solutions` (gid `0`)
  - Columns:
    - `Meta Title `
    - `Description`
    - `URL`

- **Google Drive folder**
  - Folder ID: `1S_R2qjWvxouW9Q2Riixo9wosZMG9IPTB`
  - JSON file name pattern:  
    `meta-<sanitised-start_url>-<ISO-timestamp>.json`

If you want to use different Sheet or Drive locations:

1. Open the **Save Data** node and replace the Sheet document and sheet name/gid with your own.
2. Open the **Upload file** node and replace the Drive folder with your own folder ID.
3. Save the workflow.

---

## 6. Configure the front‑end (`index.html`)

The HTML front‑end submits a POST request to your n8n webhook:

- Default webhook endpoint in the script is:  
  `https://n8n.programmx.com/webhook/MetaTitle-And-Description`

If your n8n instance uses a different base URL or path:

1. Open `index.html` in a text editor.
2. Search for the `fetch` call (near the bottom of the file).
3. Replace the URL with your own webhook, e.g.:
   - `https://YOUR-N8N-DOMAIN/webhook/MetaTitle-And-Description`
4. Save the file and redeploy to your static host (or just reopen it locally).

You can host this file anywhere that serves static HTML (GitHub Pages, Netlify, etc.). No build step is required.

---

## 7. How to run and test

### Option A: Using cURL / API client

1. Get your public webhook URL from the n8n Webhook node.  
   It will look like:
   - `https://your-n8n-domain/webhook/MetaTitle-And-Description`
2. Send a POST request with JSON body:

   ```bash
   curl -X POST "https://your-n8n-domain/webhook/MetaTitle-And-Description" \
     -H "Content-Type: application/json" \
     -d '{
       "website_link": "https://example.com/",
       "pages_to_crawl": 100,
       "email": "you@example.com"
     }'
   ```

3. Wait a few minutes (depending on crawl size).
4. Check:
   - The configured Google Sheet for new rows.
   - Google Drive for a new JSON file.
   - Your email inbox for the summary email.

### Option B: Using the HTML form

1. Open `index.html` in a browser (or visit the deployed URL if hosted).
2. Enter:
   - **Target Website** – the site you want to crawl.
   - **Email Address** – where the report should be sent.
   - **Number of Pages to Crawl** – e.g. `50` or `100`.
3. Click **Get Data**.
4. Wait for the success message; then check your email, Google Sheet, and Drive as above.

---

## 8. Deploying with GitHub

When using this repo on GitHub:

1. Push the code to your GitHub repository.
2. (Optional) Enable GitHub Pages for the branch containing `index.html` to get a public URL for the front‑end form.
3. Make sure the `index.html` webhook URL points to a publicly accessible n8n instance.
4. Keep `Meta Title & Description Automation.json`, `README.md`, and `SETUP.md` in version control so others can easily import and configure the workflow.

