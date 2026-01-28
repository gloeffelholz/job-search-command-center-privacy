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

## Quick start (5 minutes)
This template starts empty. Most people can be up and running in about 5 minutes using Google Sheets.

Before you begin, you’ll need:
- A Google account
- Access to Google Sheets
- The Job Search Command Center GPT link

There are two setup paths:

- **Path A (recommended): Google Sheets + Apps Script**
  - Fastest
  - No servers
  - No GitHub or Vercel required

- **Path B (advanced): GitHub + Vercel**
  - More control and extensibility
  - More setup
  - Optional

If you’re not sure which to choose, start with **Path A**. You can switch later.

High-level steps:
1. Create a Google Sheet to store your job pipeline
2. Set up a lightweight backend
3. Paste your resume and configuration into the GPT
4. Start using the Job Search Command Center

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

Use this path if you want maximum reliability and control, or if Path A gives you repeated authorization/deployment issues.  Apps Script Web Apps can be fast to set up, but they sometimes fail in confusing ways (authorization prompts, deployment versions, account mismatches). Path B uses a standard backend on Vercel with explicit configuration, which is usually more predictable once set up. 

IMPORTANT NOTE: Google, Github and Vercel requirements here are met with FREE versions. Paid accounts are not required in any software to build or use the AI Agent.

## Connect the GPT


## Security & privacy

For privacy details, see [PRIVACY.md](./PRIVACY.md).

## Troubleshooting

This is a public template provided as-is. There is no guaranteed support, but feedback and issue reports are welcome.

If you run into problems or have suggestions:
1. Try using ChatGPT to help diagnose and resolve the issue.  Just tell it what errors you are getting, share the code, and it can usually walk you through the right fix.
2. If that doesn't work or you need more help open a GitHub issue in this repository (preferred)
3. You can also use the support link provided inside the GPT
4. If you encounter repeated authorization or deployment issues with Path A (Apps Script), consider switching to Path B. Those issues are often caused by Google account, permission, or deployment behavior rather than the template itself.

