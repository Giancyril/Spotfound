# Google Sheets Security & Access Control Guide

**Status:** Hardened & Implemented  
**Applies to:** Student Masterlist (`Copy of Master List`) & Activity Logs (`Lost Items`, `Found Items`)

---

## 1. Problem Statement & Risk Assessment

Historically, the student masterlist was retrieved directly via Google's public Gviz visualization endpoint (`/gviz/tq`). For this endpoint to function without interactive OAuth, the Google Sheet required permissions set to:
> *"Anyone with the link can view"*

### Risks Identified:
- **Student PII Leakage:** Anyone who discovered or scraped the Sheet ID (`1-uxgLmMS13UbC_BvcVjxeGjlJUgykvRIbb4D0y7zrPI`) could download the institution's complete student directory (School ID, Full Name, Institutional Email, Academic Program, and Year Level).
- **Compliance Non-Compliance:** Exposing student education records publicly violates statutory privacy requirements (e.g., GDPR Article 32, FERPA, and Philippine Data Privacy Act of 2012 / RA 10173).

---

## 2. Hardened Architecture (Zero Public Access)

To completely eliminate public readability of the student masterlist, the system utilizes a **Private Proxy Webhook Architecture** deployed via Google Apps Script:

```
┌─────────────────────────────────┐
│     Google Drive Permissions    │  ──► Set to "Restricted" (Private)
│  (Only Sheet Owner has access)  │      Public Gviz endpoint returns 401/Redirect
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ Google Apps Script (Webhook)    │  ──► Executes as "Me" (Sheet Owner)
│ - Validates API_SECRET_KEY      │      Blocks unauthenticated requests
│ - Queries Private Sheet Data    │
└────────────────┬────────────────┘
                 │ Authenticated HTTPS (JSON)
                 ▼
┌─────────────────────────────────┐
│ Backend Server (Node/Express)   │  ──► In-Memory Redis Cache (<5ms)
│ - `student.service.ts`          │      Syncs every 6h via `masterlist.sync.ts`
│ - `masterlist.sync.ts`          │
└─────────────────────────────────┘
```

### Key Security Controls:
1. **Spreadsheet Privacy:** The Google Sheet is shared exclusively with authorized administrator accounts and set to **Restricted**. Anonymous or public link sharing is disabled.
2. **Authenticated Proxy:** The Google Apps Script (`google-apps-script/SheetsLogger.gs`) executes with owner authority and strictly requires a pre-shared secret key (`API_SECRET_KEY`) passed in backend requests.
3. **No Credential Exposure:** Neither frontend clients nor unauthorized parties ever see or interact with the secret key or the underlying sheet. Lookups are mediated by the authenticated backend.
4. **Resilient Local Caching:** The backend caches verified masterlist records in Redis, minimizing outbound API calls to Google and eliminating real-time dependencies on Google Sheets during scanner usage.

---

## 3. Implementation & Setup Guide

### Step 1: Restrict Google Sheet Sharing Permissions
1. Open the Google Spreadsheet: `https://docs.google.com/spreadsheets/d/1-uxgLmMS13UbC_BvcVjxeGjlJUgykvRIbb4D0y7zrPI/edit`
2. Click **Share** (top right corner).
3. Under **General access**, change from *"Anyone with the link"* to **"Restricted"**.
4. Ensure only institution administrators have access.

### Step 2: Deploy Updated Google Apps Script
1. In the Spreadsheet, navigate to **Extensions** > **Apps Script**.
2. Replace or update the script with the contents of `google-apps-script/SheetsLogger.gs`.
3. Set your secret key:
   - Go to **Project Settings** (gear icon) > **Script Properties**.
   - Add property: `API_SECRET_KEY` with a strong cryptographic secret (or use the configured default).
4. Click **Deploy** > **Manage deployments** > **Edit** > select **New version** > **Deploy**.
5. Copy the Web app URL.

### Step 3: Configure Backend Environment Variables
In `server/.env`:
```dotenv
GOOGLE_SHEETS_WEBHOOK_URL="https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec"
GOOGLE_SHEETS_ACCESS_KEY="<YOUR_STRONG_SECRET_KEY>"
```

---

## 4. Verification & Testing

### Verification 1: Confirm Public Gviz Access is Blocked
Execute the following curl command without Google authentication:
```bash
curl -I "https://docs.google.com/spreadsheets/d/1-uxgLmMS13UbC_BvcVjxeGjlJUgykvRIbb4D0y7zrPI/gviz/tq?tqx=out:json"
```
**Expected Response:** HTTP 302 Found (Redirects to Google Accounts sign-in) or HTTP 401/403 Unauthorized. The public Gviz endpoint is blocked.

### Verification 2: Test Unauthorized Webhook Request
```bash
curl -s "https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec?action=masterlist"
```
**Expected Response:**
```json
{"success":false,"error":"Unauthorized: Invalid or missing security key."}
```

### Verification 3: Test Authorized Backend Retrieval
```bash
curl -s "https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec?action=masterlist&key=<YOUR_STRONG_SECRET_KEY>"
```
**Expected Response:**
```json
{"success":true,"count":1500,"data":[{"id":"20250122","email":"20250122@nbsc.edu.ph","name":"...","course":"...","department":"...","yearLevel":"..."}]}
```

---

## 5. Security Checklist Status

- [x] Public Gviz access deprecated in favor of authenticated proxy
- [x] Google Sheet permissions documented for "Restricted" mode
- [x] Pre-shared secret authentication enforced in Google Apps Script
- [x] Backend services (`student.service.ts`, `masterlist.sync.ts`) upgraded to prioritize authenticated webhook
- [x] Automatic fallback and error isolation in place without server crashing
