# Job Search Command Center — Public Template

## What this is
This repository contains the setup guide for the **Job Search Command Center – Public Template**.

The Job Search Command Center is a configurable ChatGPT-based tool that helps you:
- Triage and prioritize job opportunities
- Track applications and status changes
- Plan follow-ups and next actions
- Apply a consistent scoring framework to roles

This is a **template**, not a hosted service.

You will run everything in your own accounts:
- Your job data lives in your own Google Sheet
- Any backend runs in your own environment (Google Apps Script or Vercel)
- You control all credentials and access

No job data, resumes, or private information are included or stored by this project.
## Quick start (≈30 minutes)

This template starts empty. Most people can be up and running in about **30 minutes** using Google Sheets.

Working through Vercel and GitHub will take longer, but provides more control and a production-style setup.

Before you begin, you’ll need:
- A Google account
- Access to Google Sheets
- The Job Search Command Center GPT link

High-level steps:
1. Create a Google Sheet to store your job pipeline
2. Set up a lightweight backend
3. Paste your resume and configuration into the GPT
4. Start using the Job Search Command Center
   
## Which setup path should I choose?

**Start with Path A unless you have a clear reason not to.**

| If you want… | Choose… |
|-------------|---------|
| Fastest setup (≈30 minutes) | **Path A — Google Sheets + Apps Script** |
| No servers or GitHub | **Path A** |
| Minimal configuration | **Path A** |
| Maximum reliability & control | **Path B — GitHub + Vercel** |
| Explicit credentials & auth | **Path B** |
| A production-style backend | **Path B** |

You can always start with **Path A** and switch to **Path B** later.  
Both paths use the same data model and GPT behavior.

## Setup path A (recommended): Google Sheets + Apps Script

This is the fastest way to get started. It uses Google Sheets as your database and Google Apps Script as a lightweight API. If you run into repeated authorization or deployment issues, switch to Path B. Path B is more reliable because it uses a standard API and explicit credentials.

### What you’ll create
- One Google Sheet to store your job pipeline
- One Apps Script project attached to that sheet

### Create the Google Sheet
1. Create a new Google Sheet
2. Name it something like:  
   Job Search – Command Center
3. Create a tab named:  
   Opportunities
4. Add these columns in row 1 (exact names):  
   Company | Role | Location | Status | Date Applied | Source | Comp Range | Priority | Notes

You can add more columns later, but do not rename these.

### Next
Once the sheet exists, you’ll add a small Apps Script backend to let the GPT read and write data.

### Add the Apps Script backend

1. In your Google Sheet, click:
   Extensions → Apps Script
2. Delete any existing code in the editor.
3. This script will expose a simple HTTP endpoint that the GPT can call to read and update your sheet.

### Apps Script code (copy/paste)
4. In Apps Script, create a new file named `Code.gs` (or use the default file) and paste the code below.
5. Replace the value of `SHEET_NAME` if you used a different tab name than `Opportunities`.
6. Click **Deploy → New deployment**:
   - Type: **Web app**
   - Execute as: **Me**
   - Who has access: **Anyone**
   - Click **Deploy**
   - Copy the **Web app URL** (you’ll use it in the GPT later)
7. The first time you deploy, Google will prompt you to authorize access. Approve the permissions.

```js
/**
 * Job Search Command Center — Apps Script backend (minimal)
 * Provides a simple JSON API for reading and updating the "Opportunities" sheet.
 */

const SHEET_NAME = 'Opportunities';

function doGet(e) {
  return handleRequest_(e);
}

function doPost(e) {
  return handleRequest_(e);
}

function handleRequest_(e) {
  try {
    const params = (e && e.parameter) ? e.parameter : {};
    const action = (params.action || '').toLowerCase();

    if (!action) {
      return json_({ ok: false, error: 'Missing action' }, 400);
    }

    if (action === 'health') {
      return json_({ ok: true, message: 'ok' });
    }

    if (action === 'list') {
      // Returns all rows as array of objects keyed by header name
      const sheet = getSheet_();
      const values = sheet.getDataRange().getValues();
      if (values.length < 2) return json_({ ok: true, rows: [] });

      const headers = values[0].map(h => String(h).trim());
      const rows = values.slice(1).filter(r => r.some(c => String(c).trim() !== ''));

      const objects = rows.map(r => {
        const obj = {};
        headers.forEach((h, i) => { obj[h] = r[i]; });
        return obj;
      });

      return json_({ ok: true, rows: objects });
    }

    if (action === 'append') {
      // Append a row. Expects JSON body: { row: { "Company": "...", ... } }
      const body = parseJsonBody_(e);
      const rowObj = body.row;
      if (!rowObj || typeof rowObj !== 'object') {
        return json_({ ok: false, error: 'Missing row object' }, 400);
      }

      const sheet = getSheet_();
      const headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0].map(h => String(h).trim());

      const row = headers.map(h => (h in rowObj ? rowObj[h] : ''));
      sheet.appendRow(row);

      return json_({ ok: true, appended: true });
    }

    if (action === 'update') {
      /**
       * Update rows by matching Company + Role (simple key).
       * Expects JSON body: { match: { Company: "...", Role: "..." }, updates: { Status: "...", Notes: "..." } }
       */
      const body = parseJsonBody_(e);
      const match = body.match || {};
      const updates = body.updates || {};

      if (!match.Company || !match.Role) {
        return json_({ ok: false, error: 'match.Company and match.Role are required' }, 400);
      }

      const sheet = getSheet_();
      const range = sheet.getDataRange();
      const values = range.getValues();
      if (values.length < 2) return json_({ ok: false, error: 'No rows to update' }, 400);

      const headers = values[0].map(h => String(h).trim());
      const headerIndex = {};
      headers.forEach((h, i) => { headerIndex[h] = i; });

      const companyIdx = headerIndex['Company'];
      const roleIdx = headerIndex['Role'];

      if (companyIdx === undefined || roleIdx === undefined) {
        return json_({ ok: false, error: 'Sheet must include Company and Role columns' }, 400);
      }

      let updatedCount = 0;

      for (let r = 1; r < values.length; r++) {
        const rowCompany = String(values[r][companyIdx] || '').trim();
        const rowRole = String(values[r][roleIdx] || '').trim();

        if (rowCompany === String(match.Company).trim() && rowRole === String(match.Role).trim()) {
          Object.keys(updates).forEach(k => {
            if (headerIndex[k] !== undefined) {
              values[r][headerIndex[k]] = updates[k];
            }
          });
          updatedCount++;
        }
      }

      if (updatedCount > 0) {
        range.setValues(values);
      }

      return json_({ ok: true, updatedCount });
    }

    return json_({ ok: false, error: `Unknown action: ${action}` }, 400);

  } catch (err) {
    return json_({ ok: false, error: String(err && err.message ? err.message : err) }, 500);
  }
}

function getSheet_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName(SHEET_NAME);
  if (!sheet) throw new Error(`Sheet "${SHEET_NAME}" not found`);
  return sheet;
}

function parseJsonBody_(e) {
  const raw = e && e.postData && e.postData.contents ? e.postData.contents : '';
  if (!raw) return {};
  return JSON.parse(raw);
}

function json_(obj, statusCode) {
  // Apps Script doesn't let you set HTTP status directly in ContentService,
  // but including it in the payload helps debugging.
  if (statusCode) obj.status = statusCode;

  const output = ContentService.createTextOutput(JSON.stringify(obj));
  output.setMimeType(ContentService.MimeType.JSON);
  return output;
}
```

## Test the backend (Path A)

These tests confirm your Apps Script Web App is reachable and can read and write your `Opportunities` sheet.

### Health check (GET)

Paste this into a browser, replacing `YOUR_WEB_APP_URL` with your deployed Apps Script Web App URL:
```
YOUR_WEB_APP_URL?action=health
```

Expected response:
```
{"ok":true,"message":"ok"}
```

### List rows (GET)

Paste this into a browser:
```
YOUR_WEB_APP_URL?action=list
```
Expected result:
- If your sheet only has headers, `rows` will be empty
- Otherwise, `rows` will contain your job data keyed by column name

### Append a test row (POST)

This should add a new row to your sheet.
```
curl -X POST "YOUR_WEB_APP_URL?action=append" \
  -H "Content-Type: application/json" \
  -d '{
    "row": {
      "Company": "Test Company",
      "Role": "Test Role",
      "Location": "Remote",
      "Status": "Applied",
      "Date Applied": "2026-01-28",
      "Source": "Backend test",
      "Comp Range": "",
      "Priority": "",
      "Notes": "Created during backend test"
    }
  }'
```

Expected response:
```
{"ok":true,"appended":true}
```
Refresh the Google Sheet and confirm the row appears.

### Update the test row (POST)

This updates rows by matching Company + Role.
```
curl -X POST "YOUR_WEB_APP_URL?action=update" \
  -H "Content-Type: application/json" \
  -d '{
    "match": { "Company": "Test Company", "Role": "Test Role" },
    "updates": { "Status": "Interviewing", "Notes": "Updated during backend test" }
  }'
```
Expected response:
```
{"ok":true,"updatedCount":1}
```

### Common errors

- Missing `action` parameter in the URL
- Sheet tab name does not match `SHEET_NAME`
- `Company` and `Role` columns are required for updates

## Setup path B (advanced): GitHub + Vercel

Use this path if you want maximum reliability and control, or if Path A gives you repeated authorization/deployment issues. Apps Script Web Apps can be fast to set up, but they sometimes fail in confusing ways (authorization prompts, deployment versions, account mismatches). Path B uses a standard backend on Vercel with explicit configuration, which is usually more predictable once set up.

> **IMPORTANT NOTE:** Google, GitHub, and Vercel requirements here are met with **free** versions. Paid accounts are not required.

### What you’ll create

- A GitHub repo (fork of this template)
- A Vercel project that deploys a small API
- A Google Sheet (still your “database”)
- A Google Cloud **Service Account** for Sheets API access (stored as env vars in Vercel)
- A simple API key (“shared secret”) so your GPT can securely call your backend

### Architecture (how it works)

- ChatGPT (your GPT) calls your Vercel API (HTTPS).
- Vercel API reads/writes your Google Sheet using the Google Sheets API (service account credentials).
- Your data stays in your Google account + your Vercel project.

### Step 1 — Fork the repo

1. In GitHub, click **Fork** (top-right).
2. Name it something like: `job-search-command-center`
3. Keep it public or private (either works).

### Step 2 — Create / prepare your Google Sheet

You can reuse the same Sheet format as Path A.

1. Create a new Google Sheet.
2. Name it: `Job Search – Command Center`
3. Create a tab named: `Opportunities`
4. Add these columns in row 1 (exact names):

   `Company | Role | Location | Status | Date Applied | Source | Comp Range | Priority | Notes`

5. Copy these two values (you’ll need them later):
   - **Spreadsheet ID** (from the URL):  
     `https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit...`
   - **Sheet/Tab name**: `Opportunities`

### Step 3 — Create a Google Cloud Project + Service Account (Sheets API)

This is the “credential” your Vercel backend will use to access the sheet.

1. Go to Google Cloud Console → create a **new project** (any name).
2. Enable **Google Sheets API** for that project.
3. Create a **Service Account**:
   - IAM & Admin → Service Accounts → Create
4. Create a **JSON key** for that service account:
   - Service account → Keys → Add key → Create new key → JSON  
     Download the JSON file (keep it private).

#### Share the Sheet with the service account

1. Open your Google Sheet.
2. Click **Share**.
3. Add the **service account email** (ends with `...iam.gserviceaccount.com`).
4. Give it **Editor** access.

This is critical — without it, Vercel will get “permission denied”.

### Step 4 — Deploy to Vercel

1. Go to Vercel → **Add New Project**.
2. Import your forked GitHub repo.
3. Deploy with defaults.

At this point the deployment will likely work, but the API won’t function until env vars are set.

### Step 5 — Set required environment variables in Vercel

In Vercel: Project → Settings → Environment Variables.

Create these variables:

#### A) Backend auth (shared secret)

- `JSC_API_KEY`  
  Value: generate a long random string (example: 32–64 chars)

Your GPT will send this on every request (details below).

#### B) Google Sheets connection

- `GOOGLE_SHEETS_SPREADSHEET_ID`  
  Value: your Spreadsheet ID
- `GOOGLE_SHEETS_SHEET_NAME`  
  Value: `Opportunities`

#### C) Service account credentials

You have two common options. Pick one and implement accordingly in the template backend:

**Option 1 (recommended): store the whole service account JSON**

- `GOOGLE_SERVICE_ACCOUNT_JSON`  
  Value: paste the entire JSON contents (as a single line)

**Option 2: store fields separately**

- `GOOGLE_CLIENT_EMAIL`
- `GOOGLE_PRIVATE_KEY` *(be careful with newlines; see troubleshooting)*

After adding env vars, redeploy (Vercel usually triggers automatically, but you can redeploy manually).

### Step 6 — Backend endpoints (expected behavior)

Your Vercel backend should expose endpoints that mirror Path A functionality:

- `GET /api/health` → `{ ok: true }`
- `GET /api/opportunities` → list rows
- `POST /api/opportunities` → append row
- `PATCH /api/opportunities` → update existing row (by match key)

**Auth rule (required):** every request must include:

- Header: `x-api-key: <your JSC_API_KEY>`

If the key is missing or wrong, return `401 Unauthorized`.

### Step 7 — Test your Vercel backend (before touching the GPT)

Assume your Vercel URL is:

`https://YOUR_PROJECT.vercel.app`

#### Health check

```bash
curl -i "https://YOUR_PROJECT.vercel.app/api/health" \
  -H "x-api-key: YOUR_JSC_API_KEY"
```

Expected:
- HTTP 200
- Body includes `{ "ok": true }`

#### List rows

```bash
curl -i "https://YOUR_PROJECT.vercel.app/api/opportunities" \
  -H "x-api-key: YOUR_JSC_API_KEY"
```

Expected:
- HTTP 200
- `{ ok: true, rows: [...] }`

#### Append a test row

```bash
curl -i -X POST "https://YOUR_PROJECT.vercel.app/api/opportunities" \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_JSC_API_KEY" \
  -d '{
    "row": {
      "Company": "Test Company",
      "Role": "Test Role",
      "Location": "Remote",
      "Status": "Applied",
      "Date Applied": "2026-01-28",
      "Source": "Vercel backend test",
      "Comp Range": "",
      "Priority": "",
      "Notes": "Created during Path B test"
    }
  }'
```

#### Update the test row

```bash
curl -i -X PATCH "https://YOUR_PROJECT.vercel.app/api/opportunities" \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_JSC_API_KEY" \
  -d '{
    "match": { "Company": "Test Company", "Role": "Test Role" },
    "updates": { "Status": "Interviewing", "Notes": "Updated during Path B test" }
  }'
```

Refresh the Google Sheet to confirm changes.

---

## Connect the GPT

This is the part that must be crystal clear because Path A vs Path B auth differs.

### Path A (Apps Script) — GPT Action Authentication

- Apps Script web app is deployed with **Who has access: Anyone**
- Your GPT calls the endpoint without a secret
- **GPT Action authentication setting:** `None`

### Path B (Vercel) — GPT Action Authentication (required)

Your Vercel API must be protected. The simplest production pattern is an API key header.

**Recommended:**

- **GPT Action authentication setting:** `API Key`
- Header name: `x-api-key`
- Value: your `JSC_API_KEY` (stored inside GPT as a secret)

### What you configure in the GPT

1. Open your GPT → **Configure**
2. Add an **Action**
3. Set the server URL to your Vercel base:  
   `https://YOUR_PROJECT.vercel.app`
4. Authentication:
   - Type: **API Key**
   - Location: **Header**
   - Name: `x-api-key`
   - Value: paste your `JSC_API_KEY`

Now your GPT can call:
- `GET https://YOUR_PROJECT.vercel.app/api/opportunities`
- `POST https://YOUR_PROJECT.vercel.app/api/opportunities`
- etc.

**Hard rule:** Do not use “None” auth for Path B.

### Path A - Google Sheets + Apps Script

This is a public template provided as-is. There is no guaranteed support, but feedback and issue reports are welcome.

If you run into problems or have suggestions:
1. Try using ChatGPT to help diagnose and resolve the issue.  Just tell it what errors you are getting, share the code, and it can usually walk you through the right fix.
2. If that doesn't work or you need more help open a GitHub issue in this repository (preferred)
3. You can also use the support link provided inside the GPT
4. If you encounter repeated authorization or deployment issues with Path A (Apps Script), consider switching to Path B. Those issues are often caused by Google account, permission, or deployment behavior rather than the template itself.

### Path B - Github + Vercel

### 401 Unauthorized

- Missing header `x-api-key`
- Wrong `JSC_API_KEY` in GPT vs Vercel env vars

### 403 / permission denied from Google

- You forgot to share the Google Sheet with the service account email
- Wrong Spreadsheet ID or sheet name

### Private key formatting problems

If using `GOOGLE_PRIVATE_KEY`, newlines often break.
- Ensure the value includes proper newlines (or replace `\n` escapes in code).

### Vercel deployment works but API fails at runtime

- Env vars not set for the correct environments (Production vs Preview)
- Redeploy after setting env vars

### “Sheet not found”

- `GOOGLE_SHEETS_SHEET_NAME` doesn’t match the tab name exactly (`Opportunities`)


## Security & privacy

For privacy details, see [PRIVACY.md](./PRIVACY.md).

- Your job data remains in **your Google Sheet**.
- Your backend runs in **your Vercel project** under **your account**.
- Secrets live only in:
  - Vercel Environment Variables
  - GPT Action secret storage (for `x-api-key`)
- Do not commit credentials to GitHub:
  - Never commit service account JSON
  - Never commit private keys
  - Never commit your `JSC_API_KEY`

Minimum recommended protections for Path B:
- Require `x-api-key` on every request
- Return `401` if missing/invalid
- Consider adding basic rate limiting (optional, but recommended)

