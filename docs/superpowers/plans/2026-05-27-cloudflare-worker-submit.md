# Cloudflare Worker Community Submit — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the client-side GitHub PAT in `submitToCommunity()` with a Cloudflare Worker proxy that holds the PAT server-side as an env secret.

**Architecture:** A Cloudflare Worker receives `POST { name, questions }` from the browser, validates the payload, enforces per-IP rate limiting via Cloudflare KV, then makes all 6 GitHub API calls server-side using `GITHUB_PAT` stored as a Cloudflare secret. The browser only ever talks to the Worker's public HTTPS endpoint — no secret is exposed. The template repo's dashboard HTML is updated to use a single fetch to `COMMUNITY_WORKER_URL` instead of the 60-line inline GitHub API flow.

**Tech Stack:** Cloudflare Workers (serverless JS), Cloudflare KV (rate limiting), Wrangler CLI (deployment), GitHub Contents + Refs + Pulls APIs

---

## Context

- Template repo: `/tmp/trivia-night-work` → GitHub remote `dehlia24/trivia-night-self-host-template`
- Live site repo: `/tmp/trivia-night-live` → GitHub remote `Dehlia24/trivia-night-live`
- Local commit `eeb242e` in the template repo added the real PAT to the HTML. **It was never pushed.** It must be reverted before any push.
- After revert, the file will have `COMMUNITY_PAT = 'YOUR_PAT_HERE'` and `COMMUNITY_REPO = 'dehlia24/trivia-night'` (old name) — both get replaced in Task 4.
- The Worker is **never committed to any GitHub repo** — it lives only on Cloudflare.
- Worker files are created at `~/trivia-worker/` on the local machine.

---

## Prerequisites (manual — complete before Task 2)

The following steps require the user to act. They cannot be automated:

1. Create a free Cloudflare account at cloudflare.com
2. Note your **workers.dev subdomain** — visible in the Workers dashboard as `<subdomain>.workers.dev`
3. Install Wrangler CLI: `npm install -g wrangler`
4. Login: `wrangler login` (opens browser)
5. Create KV namespace:
   ```bash
   wrangler kv:namespace create RATE_LIMIT_KV
   ```
   Note the `id` value printed in the output — needed for `wrangler.toml` in Task 2.
6. Add GitHub PAT secret (use the fine-grained PAT scoped to `contents:write` + `pull_requests:write` on `dehlia24/trivia-night-self-host-template`):
   ```bash
   wrangler secret put GITHUB_PAT
   ```
   Paste the token when prompted.

---

## File Structure

| File | Location | Action |
|------|----------|--------|
| `worker.js` | `~/trivia-worker/worker.js` | Create — Cloudflare Worker entry point |
| `wrangler.toml` | `~/trivia-worker/wrangler.toml` | Create — Wrangler deploy config |
| `trivia_night_host_dashboard.html` | `/tmp/trivia-night-work/` | Modify — replace submitToCommunity |
| `trivia_night_host_dashboard.html` | `/tmp/trivia-night-live/` | Modify — sync from template |

---

## Task 1: Revert the PAT commit

**Files:**
- Modify: `/tmp/trivia-night-work/trivia_night_host_dashboard.html`

- [ ] **Step 1: Revert commit eeb242e**

```bash
cd /tmp/trivia-night-work
git revert eeb242e --no-edit
```

Expected output includes a new commit message like `Revert "config: fix repo name references and add community PAT"`.

- [ ] **Step 2: Verify the PAT is gone**

```bash
grep -n "COMMUNITY_PAT\|COMMUNITY_REPO\|github_pat" /tmp/trivia-night-work/trivia_night_host_dashboard.html
```

Expected output:
```
739:const COMMUNITY_INDEX_URL = 'https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/index.json';
740:const COMMUNITY_FILE_BASE = 'https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/';
741:const COMMUNITY_REPO = 'dehlia24/trivia-night';
742:const COMMUNITY_PAT = 'YOUR_PAT_HERE'; // fine-grained PAT: contents+PRs write on dehlia24/trivia-night only
```

No `github_pat_` string should appear anywhere. (The `COMMUNITY_PAT` line with `YOUR_PAT_HERE` is the placeholder from before — fine to leave for now, Task 4 removes it entirely.)

---

## Task 2: Create Cloudflare Worker files

**Files:**
- Create: `~/trivia-worker/worker.js`
- Create: `~/trivia-worker/wrangler.toml`

> **Prerequisite:** Complete all manual steps listed above before this task. You need the KV namespace ID from step 5.

- [ ] **Step 1: Create the worker directory**

```bash
mkdir -p ~/trivia-worker
```

- [ ] **Step 2: Create wrangler.toml**

Create `~/trivia-worker/wrangler.toml` with the following content. Replace `REPLACE_WITH_YOUR_KV_NAMESPACE_ID` with the `id` value from `wrangler kv:namespace create RATE_LIMIT_KV`:

```toml
name = "trivia-community-submit"
main = "worker.js"
compatibility_date = "2026-01-01"

[[kv_namespaces]]
binding = "RATE_LIMIT_KV"
id = "REPLACE_WITH_YOUR_KV_NAMESPACE_ID"
```

- [ ] **Step 3: Create worker.js**

Create `~/trivia-worker/worker.js`:

```javascript
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS') {
      return new Response(null, { status: 204, headers: corsHeaders() });
    }

    if (request.method !== 'POST') {
      return jsonResponse({ ok: false, error: 'Method not allowed' }, 405);
    }

    const url = new URL(request.url);
    if (url.pathname !== '/submit') {
      return jsonResponse({ ok: false, error: 'Not found' }, 404);
    }

    const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
    const rateLimitKey = `ratelimit:${ip}`;
    const now = Date.now();
    const windowMs = 60 * 60 * 1000;

    const stored = await env.RATE_LIMIT_KV.get(rateLimitKey, 'json');
    const window = stored && (now - stored.start < windowMs)
      ? stored
      : { start: now, count: 0 };

    if (window.count >= 3) {
      return jsonResponse({ ok: false, error: 'Too many submissions — try again later' }, 429);
    }

    let body;
    try {
      body = await request.json();
    } catch {
      return jsonResponse({ ok: false, error: 'Invalid JSON' }, 400);
    }

    const validationError = validate(body);
    if (validationError) {
      return jsonResponse({ ok: false, error: validationError }, 400);
    }

    const { name, questions } = body;

    await env.RATE_LIMIT_KV.put(
      rateLimitKey,
      JSON.stringify({ start: window.start, count: window.count + 1 }),
      { expirationTtl: 3600 }
    );

    const file = name.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '') + '.json';
    const branch = `community/bank-${Date.now()}`;
    const repo = 'dehlia24/trivia-night-self-host-template';
    const headers = {
      'Authorization': `token ${env.GITHUB_PAT}`,
      'Accept': 'application/vnd.github.v3+json',
      'Content-Type': 'application/json'
    };

    try {
      const refRes = await fetch(`https://api.github.com/repos/${repo}/git/ref/heads/main`, { headers });
      if (!refRes.ok) throw new Error('Failed to get main branch ref');
      const mainSha = (await refRes.json()).object.sha;

      const branchRes = await fetch(`https://api.github.com/repos/${repo}/git/refs`, {
        method: 'POST', headers,
        body: JSON.stringify({ ref: `refs/heads/${branch}`, sha: mainSha })
      });
      if (!branchRes.ok) throw new Error('Failed to create branch');

      const bankJson = JSON.stringify({ name, author: '', description: '', questions }, null, 2);
      const fileRes = await fetch(`https://api.github.com/repos/${repo}/contents/question-banks/${file}`, {
        method: 'PUT', headers,
        body: JSON.stringify({
          message: `Community bank: ${name}`,
          content: btoa(unescape(encodeURIComponent(bankJson))),
          branch
        })
      });
      if (!fileRes.ok) throw new Error('Failed to commit bank file');

      const idxRes = await fetch(`https://api.github.com/repos/${repo}/contents/question-banks/index.json`, { headers });
      if (!idxRes.ok) throw new Error('Failed to fetch index.json');
      const idxData = await idxRes.json();
      const currentIndex = JSON.parse(atob(idxData.content.replace(/\s/g, '')));

      currentIndex.push({ name, author: '', description: '', file, questionCount: questions.length });
      const idxUpdateRes = await fetch(`https://api.github.com/repos/${repo}/contents/question-banks/index.json`, {
        method: 'PUT', headers,
        body: JSON.stringify({
          message: `Community bank: ${name} — update index`,
          content: btoa(unescape(encodeURIComponent(JSON.stringify(currentIndex, null, 2)))),
          sha: idxData.sha,
          branch
        })
      });
      if (!idxUpdateRes.ok) throw new Error('Failed to update index.json');

      const prRes = await fetch(`https://api.github.com/repos/${repo}/pulls`, {
        method: 'POST', headers,
        body: JSON.stringify({
          title: `Community bank: ${name}`,
          head: branch, base: 'main',
          body: `Community question bank submitted from the trivia night host dashboard.\n\n**Bank:** ${name}\n**Questions:** ${questions.length}`
        })
      });
      if (!prRes.ok) throw new Error('Failed to open PR');
      const prData = await prRes.json();

      return jsonResponse({ ok: true, prUrl: prData.html_url });
    } catch (e) {
      return jsonResponse({ ok: false, error: e.message || 'GitHub API error' }, 500);
    }
  }
};

function validate({ name, questions } = {}) {
  if (!name || typeof name !== 'string' || name.trim().length === 0) return 'name is required';
  if (name.length > 60) return 'name must be 60 characters or fewer';
  if (!Array.isArray(questions) || questions.length < 1) return 'questions must be a non-empty array';
  if (questions.length > 200) return 'questions array must not exceed 200 items';
  for (let i = 0; i < questions.length; i++) {
    const q = questions[i];
    if (!q.question || typeof q.question !== 'string' || q.question.trim().length === 0)
      return `question[${i}].question is required`;
    if (!Array.isArray(q.options) || q.options.length !== 4)
      return `question[${i}].options must be an array of exactly 4 items`;
    if (q.options.some(o => !o || typeof o !== 'string' || o.trim().length === 0))
      return `question[${i}].options must all be non-empty strings`;
    if (!Number.isInteger(q.correct) || q.correct < 0 || q.correct > 3)
      return `question[${i}].correct must be an integer 0–3`;
    if (!q.category || typeof q.category !== 'string' || q.category.trim().length === 0)
      return `question[${i}].category is required`;
  }
  return null;
}

function corsHeaders() {
  return {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type'
  };
}

function jsonResponse(data, status = 200) {
  return new Response(JSON.stringify(data), {
    status,
    headers: { 'Content-Type': 'application/json', ...corsHeaders() }
  });
}
```

- [ ] **Step 4: Verify files exist**

```bash
ls -la ~/trivia-worker/
```

Expected: `worker.js` and `wrangler.toml` both present.

---

## Task 3: Deploy and smoke-test the Worker

> **Prerequisite:** All manual steps from Prerequisites section must be completed, and `wrangler.toml` must have the real KV namespace ID from Task 2.

- [ ] **Step 1: Deploy the Worker**

```bash
cd ~/trivia-worker
wrangler deploy
```

Expected output includes a line like:
```
Published trivia-community-submit (X.XXs)
  https://trivia-community-submit.<your-subdomain>.workers.dev
```

Note the full URL — you will need it for Task 4.

- [ ] **Step 2: Test CORS preflight**

```bash
curl -s -o /dev/null -w "%{http_code}" -X OPTIONS \
  https://trivia-community-submit.<your-subdomain>.workers.dev/submit \
  -H "Origin: https://example.com" \
  -H "Access-Control-Request-Method: POST"
```

Expected: `204`

- [ ] **Step 3: Test validation (expect 400)**

```bash
curl -s -X POST \
  https://trivia-community-submit.<your-subdomain>.workers.dev/submit \
  -H "Content-Type: application/json" \
  -d '{"name":"","questions":[]}'
```

Expected response:
```json
{"ok":false,"error":"name is required"}
```

- [ ] **Step 4: Test rate limit path (expect 400 on bad question, not 429 yet)**

```bash
curl -s -X POST \
  https://trivia-community-submit.<your-subdomain>.workers.dev/submit \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Bank","questions":[{"question":"Q?","options":["A","B","C"],"correct":0,"category":"Test"}]}'
```

Expected response:
```json
{"ok":false,"error":"question[0].options must be an array of exactly 4 items"}
```

(A valid 4-option question would proceed to the GitHub API calls, which requires the real PAT secret to be set.)

---

## Task 4: Update dashboard HTML in template repo

**Files:**
- Modify: `/tmp/trivia-night-work/trivia_night_host_dashboard.html` (lines ~739–742 and ~1610–1681)

After completing Task 3 you know the Worker URL: `https://trivia-community-submit.<your-subdomain>.workers.dev/submit`. Use it in Step 1.

- [ ] **Step 1: Replace the four constants block**

Find this exact block in the file (lines ~739–742 after the Task 1 revert):

```javascript
const COMMUNITY_INDEX_URL = 'https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/index.json';
const COMMUNITY_FILE_BASE = 'https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/';
const COMMUNITY_REPO = 'dehlia24/trivia-night';
const COMMUNITY_PAT = 'YOUR_PAT_HERE'; // fine-grained PAT: contents+PRs write on dehlia24/trivia-night only
```

Replace with (substituting your actual subdomain in `COMMUNITY_WORKER_URL`):

```javascript
const COMMUNITY_INDEX_URL = 'https://raw.githubusercontent.com/dehlia24/trivia-night-self-host-template/main/question-banks/index.json';
const COMMUNITY_FILE_BASE = 'https://raw.githubusercontent.com/dehlia24/trivia-night-self-host-template/main/question-banks/';
const COMMUNITY_WORKER_URL = 'https://trivia-community-submit.<your-subdomain>.workers.dev/submit';
```

- [ ] **Step 2: Replace the submitToCommunity function**

Find this entire function (lines ~1610–1681):

```javascript
async function submitToCommunity() {
  if (!lastSavedTemplateName) return;
  const name = lastSavedTemplateName;
  const file = name.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '') + '.json';
  const branch = `community/bank-${Date.now()}`;
  const headers = {
    'Authorization': `token ${COMMUNITY_PAT}`,
    'Accept': 'application/vnd.github.v3+json',
    'Content-Type': 'application/json'
  };
  try {
    // Get SHA of main branch HEAD
    const refRes = await fetch(`https://api.github.com/repos/${COMMUNITY_REPO}/git/ref/heads/main`, { headers });
    if (!refRes.ok) throw new Error('ref');
    const mainSha = (await refRes.json()).object.sha;

    // Create branch
    const branchRes = await fetch(`https://api.github.com/repos/${COMMUNITY_REPO}/git/refs`, {
      method: 'POST', headers,
      body: JSON.stringify({ ref: `refs/heads/${branch}`, sha: mainSha })
    });
    if (!branchRes.ok) throw new Error('branch');

    // Commit bank JSON file
    const bankJson = JSON.stringify({ name, author: '', description: '', questions: questionBank }, null, 2);
    const fileRes = await fetch(`https://api.github.com/repos/${COMMUNITY_REPO}/contents/question-banks/${file}`, {
      method: 'PUT', headers,
      body: JSON.stringify({
        message: `Community bank: ${name}`,
        content: btoa(unescape(encodeURIComponent(bankJson))),
        branch
      })
    });
    if (!fileRes.ok) throw new Error('file');

    // Fetch current index.json and append new entry
    const idxRes = await fetch(`https://api.github.com/repos/${COMMUNITY_REPO}/contents/question-banks/index.json`, { headers });
    if (!idxRes.ok) throw new Error('index fetch');
    const idxData = await idxRes.json();
    const currentIndex = JSON.parse(atob(idxData.content.replace(/\s/g, '')));
    currentIndex.push({ name, author: '', description: '', file, questionCount: questionBank.length });
    const idxUpdateRes = await fetch(`https://api.github.com/repos/${COMMUNITY_REPO}/contents/question-banks/index.json`, {
      method: 'PUT', headers,
      body: JSON.stringify({
        message: `Community bank: ${name} — update index`,
        content: btoa(unescape(encodeURIComponent(JSON.stringify(currentIndex, null, 2)))),
        sha: idxData.sha,
        branch
      })
    });
    if (!idxUpdateRes.ok) throw new Error('index update');

    // Open pull request
    const prRes = await fetch(`https://api.github.com/repos/${COMMUNITY_REPO}/pulls`, {
      method: 'POST', headers,
      body: JSON.stringify({
        title: `Community bank: ${name}`,
        head: branch, base: 'main',
        body: `Community question bank submitted from the trivia night host dashboard.\n\n**Bank:** ${name}\n**Questions:** ${questionBank.length}`
      })
    });
    if (!prRes.ok) throw new Error('PR');
    const prData = await prRes.json();

    closeSubmitRow();
    showNotif('PR submitted! Opening in new tab…');
    setTimeout(() => window.open(prData.html_url, '_blank'), 600);
  } catch {
    closeSubmitRow();
    showNotif('Submit failed — open a PR manually at github.com/dehlia24/trivia-night', 'error');
  }
}
```

Replace with:

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

- [ ] **Step 3: Verify no PAT or old repo references remain**

```bash
grep -n "COMMUNITY_PAT\|COMMUNITY_REPO\|github_pat\|dehlia24/trivia-night'" /tmp/trivia-night-work/trivia_night_host_dashboard.html
```

Expected: no output (zero matches).

- [ ] **Step 4: Verify the new constants are present**

```bash
grep -n "COMMUNITY_INDEX_URL\|COMMUNITY_FILE_BASE\|COMMUNITY_WORKER_URL" /tmp/trivia-night-work/trivia_night_host_dashboard.html
```

Expected output (line numbers may vary slightly):
```
739:const COMMUNITY_INDEX_URL = 'https://raw.githubusercontent.com/dehlia24/trivia-night-self-host-template/main/question-banks/index.json';
740:const COMMUNITY_FILE_BASE = 'https://raw.githubusercontent.com/dehlia24/trivia-night-self-host-template/main/question-banks/';
741:const COMMUNITY_WORKER_URL = 'https://trivia-community-submit.<your-subdomain>.workers.dev/submit';
```

- [ ] **Step 5: Verify submitToCommunity is the short version**

```bash
grep -c "fetch(COMMUNITY_WORKER_URL" /tmp/trivia-night-work/trivia_night_host_dashboard.html
```

Expected: `1`

```bash
grep -c "api.github.com" /tmp/trivia-night-work/trivia_night_host_dashboard.html
```

Expected: `0`

- [ ] **Step 6: Commit and push**

```bash
cd /tmp/trivia-night-work
git add trivia_night_host_dashboard.html
git commit -m "feat: replace inline GitHub API flow with Cloudflare Worker proxy"
git push
```

Expected: push succeeds with no GitHub push protection warnings.

---

## Task 5: Sync dashboard to trivia-night-live

**Files:**
- Modify: `/tmp/trivia-night-live/trivia_night_host_dashboard.html`

- [ ] **Step 1: Copy the updated dashboard**

```bash
cp /tmp/trivia-night-work/trivia_night_host_dashboard.html /tmp/trivia-night-live/trivia_night_host_dashboard.html
```

- [ ] **Step 2: Verify community code is present in live repo**

```bash
grep -n "COMMUNITY_WORKER_URL\|loadCommunityBanks\|tmpl-submit-row" /tmp/trivia-night-live/trivia_night_host_dashboard.html | head -10
```

Expected: 3+ matching lines showing the Worker URL constant, `loadCommunityBanks` function, and the submit row element.

- [ ] **Step 3: Verify no PAT in live file**

```bash
grep "github_pat\|COMMUNITY_PAT" /tmp/trivia-night-live/trivia_night_host_dashboard.html
```

Expected: no output.

- [ ] **Step 4: Commit and push (triggers Firebase auto-deploy)**

```bash
cd /tmp/trivia-night-live
git add trivia_night_host_dashboard.html
git commit -m "feat: add community question banks and Cloudflare Worker submit"
git push
```

Expected: push succeeds. GitHub Actions auto-deploys to `trivia-127b9.web.app` within ~1 minute.

- [ ] **Step 5: Verify deployment**

Watch the Actions tab at `https://github.com/Dehlia24/trivia-night-live/actions` — the deploy job should complete green. Then open `https://trivia-127b9.web.app/trivia_night_host_dashboard.html` and confirm the template dropdown shows a "Community Banks" section with "General Knowledge" listed.
