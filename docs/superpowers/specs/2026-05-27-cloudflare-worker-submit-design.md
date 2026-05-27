# Cloudflare Worker — Community Submit — Design Spec
**Date:** 2026-05-27
**Replaces:** PAT-in-HTML approach from community-question-banks spec

---

## Overview

Replace the client-side GitHub API calls in `submitToCommunity()` with a Cloudflare Worker proxy. The PAT lives as a Cloudflare secret env var — never in any repo or HTML file. The dashboard sends bank data to the Worker endpoint; the Worker creates the branch, commits the files, and opens the PR server-side.

---

## Architecture

```
Browser (submitToCommunity)
    │  POST { name, questions }
    ▼
Cloudflare Worker (trivia-community-submit)  ← PAT in env secret
    │  GitHub API: create branch, commit bank JSON, update index.json, open PR
    ▼
dehlia24/trivia-night-self-host-template
    └── PR created → URL returned to browser → opened in new tab
```

---

## Worker Spec

**Endpoint:** `POST /submit`
**Hosted at:** `https://trivia-community-submit.<subdomain>.workers.dev/submit`

### Request body
```json
{ "name": "My Bank Name", "questions": [...] }
```

### Validation (returns 400 on failure)
- `name`: non-empty string, max 60 chars
- `questions`: array, 1–200 items
- Each question: `question` (non-empty string), `options` (array of exactly 4 non-empty strings), `correct` (integer 0–3), `category` (non-empty string)

### Rate limiting
- 3 submissions per IP per hour
- Tracked in Cloudflare KV store (`RATE_LIMIT_KV` binding)
- Exceeds limit → 429 `{ ok: false, error: "Too many submissions — try again later" }`

### CORS
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```
OPTIONS preflight returns 204.

### GitHub API steps (server-side, using `GITHUB_PAT` env secret)
1. GET `repos/dehlia24/trivia-night-self-host-template/git/ref/heads/main` → get SHA
2. POST `repos/.../git/refs` → create branch `community/bank-{timestamp}`
3. PUT `repos/.../contents/question-banks/{file}` → commit bank JSON
4. GET `repos/.../contents/question-banks/index.json` → fetch current index
5. PUT `repos/.../contents/question-banks/index.json` → append entry + commit
6. POST `repos/.../pulls` → open PR

### Success response
```json
{ "ok": true, "prUrl": "https://github.com/Dehlia24/trivia-night-self-host-template/pull/N" }
```

### Error response
```json
{ "ok": false, "error": "human-readable message" }
```

---

## Dashboard Changes (`trivia_night_host_dashboard.html`)

### Remove
- `COMMUNITY_PAT` constant
- `COMMUNITY_REPO` constant
- All inline GitHub API fetch calls inside `submitToCommunity()`

### Add
```javascript
const COMMUNITY_WORKER_URL = 'https://trivia-community-submit.<subdomain>.workers.dev/submit';
// <subdomain> is your Cloudflare workers.dev subdomain — filled in after deploying the Worker
```

### Replace `submitToCommunity()` body
```javascript
async function submitToCommunity() {
  if (!lastSavedTemplateName) return;
  try {
    const res = await fetch(COMMUNITY_WORKER_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name: lastSavedTemplateName, questions: questionBank })
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error);
    closeSubmitRow();
    showNotif('PR submitted! Opening in new tab…');
    setTimeout(() => window.open(data.prUrl, '_blank'), 600);
  } catch (e) {
    closeSubmitRow();
    showNotif(e.message || 'Submit failed — open a PR manually at github.com/dehlia24/trivia-night-self-host-template', 'error');
  }
}
```

`COMMUNITY_INDEX_URL` and `COMMUNITY_FILE_BASE` stay unchanged (read-only raw content fetches, no secret needed).

### Revert
The local commit `eeb242e` (which added the PAT to HTML) must be reverted before pushing.

---

## Files

| File | Action | Repo |
|------|--------|------|
| `worker.js` | Create | Deployed to Cloudflare (not in any GitHub repo) |
| `trivia_night_host_dashboard.html` | Modify | `trivia-night-self-host-template` + `trivia-night-live` |

---

## Prerequisites (manual steps before implementation)

1. Create free Cloudflare account at cloudflare.com
2. Note your Cloudflare account subdomain (shown in Workers dashboard as `<subdomain>.workers.dev`)
3. Install wrangler CLI: `npm install -g wrangler`
4. Login: `wrangler login`
5. Create KV namespace for rate limiting: `wrangler kv:namespace create RATE_LIMIT_KV`
6. Add `GITHUB_PAT` secret to Worker: `wrangler secret put GITHUB_PAT` → paste token

---

## Out of Scope
- Worker analytics or logging beyond basic Cloudflare metrics
- Author name collection (submitted banks have empty author field; PR submitter can edit)
- Duplicate submission detection
