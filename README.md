# Job Search Command Center — Public Template

## What this is
This repository contains the setup guide for the **Job Search Command Center – Public Template** 

Public GPT Template: https://www.futureinsites.com/job-search-command-center.html

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
- The Job Search Command Center GPT link https://chatgpt.com/g/g-697a162c9eb08191a5f24fb6c6f74980-job-search-command-center-public-template

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
