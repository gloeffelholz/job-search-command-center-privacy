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

This is the fastest way to get started. It uses Google Sheets as your database and Google Apps Script as a lightweight API.

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

## Setup path B (advanced): GitHub + Vercel
## Connect the GPT
## Security & privacy
## Troubleshooting

For privacy details, see [PRIVACY.md](./PRIVACY.md).
