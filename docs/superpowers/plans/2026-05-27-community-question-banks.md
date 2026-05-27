# Community Question Banks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add community question banks to the trivia-night template — stored as JSON files in the repo, shown in the host dashboard dropdown, and submittable via a "Submit to Community" button that opens a GitHub PR.

**Architecture:** Static JSON files in `question-banks/` are fetched at runtime via GitHub's raw content CDN and rendered as a "Community Banks" section in the existing template combo box. The "Submit to Community" button (shown after a Firebase template save) uses the GitHub API with a scoped PAT to create a branch, commit the bank JSON + updated index.json, and open a PR. Both the public template repo and the personal live repo get the same dashboard HTML changes.

**Tech Stack:** Vanilla JS (no build step), GitHub REST API v3, GitHub raw content CDN, Firebase Realtime Database (existing)

---

## Prerequisites (manual — complete before Task 8)

Create a fine-grained GitHub PAT at `https://github.com/settings/personal-access-tokens/new`:
- Resource owner: `Dehlia24`
- Repository access: Only `dehlia24/trivia-night`
- Permissions: `Contents: Read and write`, `Pull requests: Read and write`

Copy the token value — it goes into `COMMUNITY_PAT` in Task 8.

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `question-banks/index.json` | Create | Master list of community banks |
| `question-banks/general-knowledge.json` | Create | Starter bank (15 questions) |
| `CONTRIBUTING.md` | Create | How to contribute a bank manually via PR |
| `.github/PULL_REQUEST_TEMPLATE.md` | Create | PR checklist for contributors |
| `README.md` | Modify | Add link to CONTRIBUTING.md |
| `trivia_night_host_dashboard.html` | Modify | Community banks dropdown + Submit to Community flow |
| `trivia-night-live/trivia_night_host_dashboard.html` | Modify | Same dashboard changes synced to live repo |

---

## Task 1: Create question-banks directory

**Files:**
- Create: `question-banks/index.json`
- Create: `question-banks/general-knowledge.json`

- [ ] **Step 1: Clone the template repo**

```bash
git clone https://github.com/Dehlia24/trivia-night.git /tmp/trivia-night-work
cd /tmp/trivia-night-work
mkdir question-banks
```

- [ ] **Step 2: Create general-knowledge.json**

Write `/tmp/trivia-night-work/question-banks/general-knowledge.json`:

```json
{
  "name": "General Knowledge",
  "author": "dehlia24",
  "description": "A broad mix of trivia across science, history, geography, and pop culture",
  "questions": [
    { "question": "What is the chemical symbol for gold?", "options": ["Au", "Ag", "Fe", "Pb"], "correct": 0, "category": "Science" },
    { "question": "Which planet is known as the Red Planet?", "options": ["Venus", "Jupiter", "Mars", "Saturn"], "correct": 2, "category": "Science" },
    { "question": "How many sides does a hexagon have?", "options": ["5", "6", "7", "8"], "correct": 1, "category": "Math" },
    { "question": "What is the capital of France?", "options": ["London", "Berlin", "Madrid", "Paris"], "correct": 3, "category": "Geography" },
    { "question": "Who painted the Mona Lisa?", "options": ["Michelangelo", "Raphael", "Leonardo da Vinci", "Donatello"], "correct": 2, "category": "Art" },
    { "question": "What year did World War II end?", "options": ["1943", "1944", "1945", "1946"], "correct": 2, "category": "History" },
    { "question": "Which element has the atomic number 1?", "options": ["Helium", "Hydrogen", "Lithium", "Carbon"], "correct": 1, "category": "Science" },
    { "question": "What is the largest ocean on Earth?", "options": ["Atlantic", "Indian", "Arctic", "Pacific"], "correct": 3, "category": "Geography" },
    { "question": "How many strings does a standard guitar have?", "options": ["4", "5", "6", "7"], "correct": 2, "category": "Music" },
    { "question": "What is the fastest land animal?", "options": ["Lion", "Cheetah", "Leopard", "Horse"], "correct": 1, "category": "Nature" },
    { "question": "Which country invented pizza?", "options": ["Greece", "Spain", "France", "Italy"], "correct": 3, "category": "Food" },
    { "question": "How many bones are in the adult human body?", "options": ["186", "196", "206", "216"], "correct": 2, "category": "Science" },
    { "question": "What is the longest river in the world?", "options": ["Amazon", "Nile", "Mississippi", "Yangtze"], "correct": 1, "category": "Geography" },
    { "question": "In what year did the Titanic sink?", "options": ["1910", "1912", "1914", "1916"], "correct": 1, "category": "History" },
    { "question": "What gas do plants absorb from the atmosphere?", "options": ["Oxygen", "Nitrogen", "Carbon Dioxide", "Hydrogen"], "correct": 2, "category": "Science" }
  ]
}
```

- [ ] **Step 3: Create index.json**

Write `/tmp/trivia-night-work/question-banks/index.json`:

```json
[
  {
    "name": "General Knowledge",
    "author": "dehlia24",
    "description": "A broad mix of trivia across science, history, geography, and pop culture",
    "file": "general-knowledge.json",
    "questionCount": 15
  }
]
```

- [ ] **Step 4: Commit**

```bash
cd /tmp/trivia-night-work
git add question-banks/
git commit -m "feat: add question-banks directory with General Knowledge starter bank"
```

---

## Task 2: Add contribution infrastructure

**Files:**
- Create: `CONTRIBUTING.md`
- Create: `.github/PULL_REQUEST_TEMPLATE.md`
- Modify: `README.md`

- [ ] **Step 1: Create CONTRIBUTING.md**

Write `/tmp/trivia-night-work/CONTRIBUTING.md`:

```markdown
# Contributing a Question Bank

Anyone can contribute a question bank to the community library. Contributed banks appear in the **Question Bank Templates** dropdown for all users of this template.

## Quick way: Submit from the dashboard

Open your host dashboard, build a question bank, save it as a template, then click **Submit to Community**. This automatically opens a pull request with your bank.

## Manual way: Open a pull request

### 1. Create your bank file

Add a file to `question-banks/your-bank-name.json` using this format:

```json
{
  "name": "Your Bank Name",
  "author": "your-github-username",
  "description": "One sentence describing what this bank covers",
  "questions": [
    {
      "question": "Your question text?",
      "options": ["Option A", "Option B", "Option C", "Option D"],
      "correct": 0,
      "category": "Your Category"
    }
  ]
}
```

Rules:
- Every question must have **exactly 4 options**
- `correct` must be `0`, `1`, `2`, or `3` (the index of the correct option)
- All fields (`question`, `options`, `correct`, `category`) are required

### 2. Update index.json

Add an entry to `question-banks/index.json`:

```json
{
  "name": "Your Bank Name",
  "author": "your-github-username",
  "description": "One sentence describing what this bank covers",
  "file": "your-bank-name.json",
  "questionCount": 25
}
```

`questionCount` must match the actual number of questions in your file.

### 3. Open a pull request

Submit your PR — the checklist will appear automatically.
```

- [ ] **Step 2: Create .github/PULL_REQUEST_TEMPLATE.md**

```bash
mkdir -p /tmp/trivia-night-work/.github
```

Write `/tmp/trivia-night-work/.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Community Bank Submission

- [ ] Added `question-banks/<my-bank>.json` with the correct structure
- [ ] Updated `question-banks/index.json` with a matching entry
- [ ] `questionCount` in index.json matches the actual number of questions in the file
- [ ] Every question has exactly 4 options
- [ ] `correct` is `0`, `1`, `2`, or `3` for every question
- [ ] All questions have a non-empty `category` field
```

- [ ] **Step 3: Add CONTRIBUTING link to README**

In `/tmp/trivia-night-work/README.md`, find the line:

```markdown
## About
```

Replace it with:

```markdown
## Contributing

Have a great set of trivia questions? [Contribute a question bank](CONTRIBUTING.md) to the community library.

---

## About
```

- [ ] **Step 4: Commit**

```bash
cd /tmp/trivia-night-work
git add CONTRIBUTING.md .github/PULL_REQUEST_TEMPLATE.md README.md
git commit -m "feat: add CONTRIBUTING.md, PR template, and README link for community banks"
```

---

## Task 3: Add CSS and HTML to dashboard

**Files:**
- Modify: `trivia_night_host_dashboard.html`

- [ ] **Step 1: Add .tmpl-combo-header and .tmpl-submit-row CSS**

In `/tmp/trivia-night-work/trivia_night_host_dashboard.html`, find:

```css
  .tmpl-name-input:focus { border-color: var(--green); }
```

Replace with:

```css
  .tmpl-name-input:focus { border-color: var(--green); }
  .tmpl-combo-header { padding: 5px 12px; font-size: 10px; font-weight: 700; letter-spacing: 1.5px; text-transform: uppercase; color: var(--muted); background: var(--surface2); pointer-events: none; }
  .tmpl-submit-row { display: none; margin-top: 10px; gap: 8px; align-items: center; flex-wrap: wrap; }
  .tmpl-submit-row.open { display: flex; }
```

- [ ] **Step 2: Add tmpl-submit-row HTML**

In `/tmp/trivia-night-work/trivia_night_host_dashboard.html`, find:

```html
          <div class="tmpl-info" id="tmpl-info"></div>
        </div>
```

Replace with:

```html
          <div class="tmpl-submit-row" id="tmpl-submit-row">
            <span style="font-size:12px;color:var(--muted)">Template saved — want to share it with the community?</span>
            <button class="btn btn-secondary" onclick="submitToCommunity()" style="font-size:13px;padding:6px 12px;">Submit to Community</button>
            <button class="btn" onclick="closeSubmitRow()" style="font-size:13px;padding:6px 12px;background:transparent;border:1px solid var(--border);color:var(--muted)">Dismiss</button>
          </div>
          <div class="tmpl-info" id="tmpl-info"></div>
        </div>
```

- [ ] **Step 3: Commit**

```bash
cd /tmp/trivia-night-work
git add trivia_night_host_dashboard.html
git commit -m "feat: add community banks CSS and submit row HTML to dashboard"
```

---

## Task 4: Add community banks JS — state, fetch, dropdown

**Files:**
- Modify: `trivia_night_host_dashboard.html`

- [ ] **Step 1: Add constants and state variables**

In `/tmp/trivia-night-work/trivia_night_host_dashboard.html`, find:

```javascript
let templates = {};       // globally shared named banks, stored in /templates
```

Replace with:

```javascript
let templates = {};       // globally shared named banks, stored in /templates
let communityBanks = [];  // banks fetched from question-banks/index.json on GitHub
let lastSavedTemplateName = '';

const COMMUNITY_INDEX_URL = 'https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/index.json';
const COMMUNITY_FILE_BASE = 'https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/';
const COMMUNITY_REPO = 'dehlia24/trivia-night';
const COMMUNITY_PAT = 'YOUR_PAT_HERE'; // fine-grained PAT: contents+PRs write on dehlia24/trivia-night only
```

- [ ] **Step 2: Add loadCommunityBanks function**

In `/tmp/trivia-night-work/trivia_night_host_dashboard.html`, find:

```javascript
// ─── Templates combo box ──────────────────────────────────────────────────────
```

Insert the following **before** that comment line:

```javascript
// ─── Community banks (fetched from GitHub at load time) ───────────────────────
async function loadCommunityBanks() {
  try {
    const res = await fetch(COMMUNITY_INDEX_URL);
    if (!res.ok) throw new Error('fetch failed');
    communityBanks = await res.json();
  } catch {
    communityBanks = [];
  }
  populateTemplateDropdown();
}

```

- [ ] **Step 3: Call loadCommunityBanks on init**

In `/tmp/trivia-night-work/trivia_night_host_dashboard.html`, find:

```javascript
  db.ref('templates').on('value', snap => {
    templates = snap.val() || {};
```

Read the full `db.ref('templates').on(...)` callback to find its closing `});`, then add `loadCommunityBanks();` on the line immediately after that closing `});`.

The result should look like:

```javascript
  db.ref('templates').on('value', snap => {
    templates = snap.val() || {};
    // ... existing callback body ...
  });
  loadCommunityBanks();
```

- [ ] **Step 4: Replace tmplComboGetFiltered**

Find and replace the entire `tmplComboGetFiltered` function:

```javascript
function tmplComboGetFiltered() {
  const filter = (document.getElementById('tmpl-search')?.value || '').toLowerCase().trim();
  const sorted = Object.entries(templates).sort(([,a],[,b]) => (b.savedAt||0) - (a.savedAt||0));
  return filter ? sorted.filter(([, t]) => t.name.toLowerCase().includes(filter)) : sorted;
}
```

Replace with:

```javascript
function tmplComboGetFiltered() {
  const filter = (document.getElementById('tmpl-search')?.value || '').toLowerCase().trim();
  const sortedFirebase = Object.entries(templates).sort(([,a],[,b]) => (b.savedAt||0) - (a.savedAt||0));
  return {
    community: filter ? communityBanks.filter(b => b.name.toLowerCase().includes(filter)) : communityBanks,
    firebase:  filter ? sortedFirebase.filter(([, t]) => t.name.toLowerCase().includes(filter)) : sortedFirebase
  };
}
```

- [ ] **Step 5: Replace tmplComboRenderList**

Find and replace the entire `tmplComboRenderList` function:

```javascript
function tmplComboRenderList() {
  const list = document.getElementById('tmpl-combo-list');
  if (!list) return;
  const filtered = tmplComboGetFiltered();
  list.innerHTML = '';
  tmplComboActiveIdx = -1;
  if (filtered.length === 0) {
    const empty = document.createElement('div');
    empty.className = 'tmpl-combo-empty';
    empty.textContent = 'No templates match';
    list.appendChild(empty);
    return;
  }
  filtered.forEach(([key, tmpl], i) => {
    const qCount = Array.isArray(tmpl.questions) ? tmpl.questions.length : 0;
    const date = tmpl.savedAt ? new Date(tmpl.savedAt).toLocaleDateString() : '';
    const item = document.createElement('div');
    item.className = 'tmpl-combo-option' + (key === selectedTemplateKey ? ' selected' : '');
    item.dataset.key = key;
    item.textContent = `${tmpl.name}  (${qCount}q${date ? ' · ' + date : ''})`;
    item.addEventListener('mousedown', e => { e.preventDefault(); tmplComboSelect(key); });
    list.appendChild(item);
  });
}
```

Replace with:

```javascript
function tmplComboRenderList() {
  const list = document.getElementById('tmpl-combo-list');
  if (!list) return;
  const { community, firebase } = tmplComboGetFiltered();
  list.innerHTML = '';
  tmplComboActiveIdx = -1;

  if (community.length === 0 && firebase.length === 0) {
    const empty = document.createElement('div');
    empty.className = 'tmpl-combo-empty';
    empty.textContent = 'No templates match';
    list.appendChild(empty);
    return;
  }

  if (community.length > 0) {
    const hdr = document.createElement('div');
    hdr.className = 'tmpl-combo-header';
    hdr.textContent = 'Community Banks';
    list.appendChild(hdr);
    community.forEach(bank => {
      const key = 'community::' + bank.file;
      const item = document.createElement('div');
      item.className = 'tmpl-combo-option' + (key === selectedTemplateKey ? ' selected' : '');
      item.dataset.key = key;
      item.textContent = `${bank.name}  (${bank.questionCount}q · by ${bank.author || 'anonymous'})`;
      item.addEventListener('mousedown', e => { e.preventDefault(); tmplComboSelect(key); });
      list.appendChild(item);
    });
  }

  if (firebase.length > 0) {
    const hdr = document.createElement('div');
    hdr.className = 'tmpl-combo-header';
    hdr.textContent = 'My Saved Templates';
    list.appendChild(hdr);
    firebase.forEach(([key, tmpl]) => {
      const qCount = Array.isArray(tmpl.questions) ? tmpl.questions.length : 0;
      const date = tmpl.savedAt ? new Date(tmpl.savedAt).toLocaleDateString() : '';
      const item = document.createElement('div');
      item.className = 'tmpl-combo-option' + (key === selectedTemplateKey ? ' selected' : '');
      item.dataset.key = key;
      item.textContent = `${tmpl.name}  (${qCount}q${date ? ' · ' + date : ''})`;
      item.addEventListener('mousedown', e => { e.preventDefault(); tmplComboSelect(key); });
      list.appendChild(item);
    });
  }
}
```

- [ ] **Step 6: Replace tmplComboClose to handle community:: keys**

Find and replace the entire `tmplComboClose` function:

```javascript
function tmplComboClose() {
  const input = document.getElementById('tmpl-search');
  const list = document.getElementById('tmpl-combo-list');
  input?.classList.remove('open');
  list?.classList.remove('open');
  // Restore selected name in input
  input.value = selectedTemplateKey && templates[selectedTemplateKey] ? templates[selectedTemplateKey].name : '';
}
```

Replace with:

```javascript
function tmplComboClose() {
  const input = document.getElementById('tmpl-search');
  const list = document.getElementById('tmpl-combo-list');
  input?.classList.remove('open');
  list?.classList.remove('open');
  if (selectedTemplateKey.startsWith('community::')) {
    const bank = communityBanks.find(b => b.file === selectedTemplateKey.replace('community::', ''));
    input.value = bank ? bank.name : '';
  } else {
    input.value = selectedTemplateKey && templates[selectedTemplateKey] ? templates[selectedTemplateKey].name : '';
  }
}
```

- [ ] **Step 7: Replace onTemplateSelect to handle community:: keys**

Find and replace the entire `onTemplateSelect` function:

```javascript
function onTemplateSelect() {
  const info = document.getElementById('tmpl-info');
  const delBtn = document.getElementById('tmpl-delete-btn');
  if (!selectedTemplateKey || !templates[selectedTemplateKey]) {
    info.textContent = '';
    delBtn.style.display = 'none';
    return;
  }
  const tmpl = templates[selectedTemplateKey];
  const qCount = Array.isArray(tmpl.questions) ? tmpl.questions.length : 0;
  const date = tmpl.savedAt ? new Date(tmpl.savedAt).toLocaleString() : 'unknown';
  info.textContent = `${qCount} question${qCount !== 1 ? 's' : ''} · Saved ${date}`;
  delBtn.style.display = '';
}
```

Replace with:

```javascript
function onTemplateSelect() {
  const info = document.getElementById('tmpl-info');
  const delBtn = document.getElementById('tmpl-delete-btn');
  if (!selectedTemplateKey) { info.textContent = ''; delBtn.style.display = 'none'; return; }
  if (selectedTemplateKey.startsWith('community::')) {
    const bank = communityBanks.find(b => b.file === selectedTemplateKey.replace('community::', ''));
    if (!bank) { info.textContent = ''; delBtn.style.display = 'none'; return; }
    info.textContent = `${bank.questionCount} question${bank.questionCount !== 1 ? 's' : ''} · by ${bank.author || 'anonymous'}${bank.description ? ' · ' + bank.description : ''}`;
    delBtn.style.display = 'none';
    return;
  }
  if (!templates[selectedTemplateKey]) { info.textContent = ''; delBtn.style.display = 'none'; return; }
  const tmpl = templates[selectedTemplateKey];
  const qCount = Array.isArray(tmpl.questions) ? tmpl.questions.length : 0;
  const date = tmpl.savedAt ? new Date(tmpl.savedAt).toLocaleString() : 'unknown';
  info.textContent = `${qCount} question${qCount !== 1 ? 's' : ''} · Saved ${date}`;
  delBtn.style.display = '';
}
```

- [ ] **Step 8: Commit**

```bash
cd /tmp/trivia-night-work
git add trivia_night_host_dashboard.html
git commit -m "feat: add community banks dropdown sections and GitHub fetch logic"
```

---

## Task 5: Modify loadSelectedTemplate for community banks

**Files:**
- Modify: `trivia_night_host_dashboard.html`

- [ ] **Step 1: Replace loadSelectedTemplate**

Find and replace the entire `loadSelectedTemplate` function:

```javascript
async function loadSelectedTemplate() {
  if (!selectedTemplateKey || !templates[selectedTemplateKey]) { showNotif('Select a template first', 'error'); return; }
  const tmpl = templates[selectedTemplateKey];
  if (questionBank.length > 0 && !confirm(`Replace your current ${questionBank.length}-question bank with "${tmpl.name}"?`)) return;
  questionBank = Array.isArray(tmpl.questions) ? [...tmpl.questions] : Object.values(tmpl.questions);
  await gref('questionBank').set(questionBank);
  renderQuestionsEditor();
  showNotif(`Loaded template "${tmpl.name}"`);
}
```

Replace with:

```javascript
async function loadSelectedTemplate() {
  if (!selectedTemplateKey) { showNotif('Select a template first', 'error'); return; }
  if (selectedTemplateKey.startsWith('community::')) {
    const file = selectedTemplateKey.replace('community::', '');
    const bank = communityBanks.find(b => b.file === file);
    if (!bank) { showNotif('Community bank not found', 'error'); return; }
    if (questionBank.length > 0 && !confirm(`Replace your current ${questionBank.length}-question bank with "${bank.name}"?`)) return;
    try {
      const res = await fetch(COMMUNITY_FILE_BASE + file);
      if (!res.ok) throw new Error('fetch failed');
      const data = await res.json();
      questionBank = data.questions;
      await gref('questionBank').set(questionBank);
      renderQuestionsEditor();
      showNotif(`Loaded "${bank.name}"`);
    } catch {
      showNotif('Failed to load community bank — check your connection', 'error');
    }
    return;
  }
  if (!templates[selectedTemplateKey]) { showNotif('Select a template first', 'error'); return; }
  const tmpl = templates[selectedTemplateKey];
  if (questionBank.length > 0 && !confirm(`Replace your current ${questionBank.length}-question bank with "${tmpl.name}"?`)) return;
  questionBank = Array.isArray(tmpl.questions) ? [...tmpl.questions] : Object.values(tmpl.questions);
  await gref('questionBank').set(questionBank);
  renderQuestionsEditor();
  showNotif(`Loaded template "${tmpl.name}"`);
}
```

- [ ] **Step 2: Commit**

```bash
cd /tmp/trivia-night-work
git add trivia_night_host_dashboard.html
git commit -m "feat: lazy-load community bank JSON on template load"
```

---

## Task 6: Add Submit to Community flow

**Files:**
- Modify: `trivia_night_host_dashboard.html`

- [ ] **Step 1: Modify confirmSaveTemplate to show submit row**

Find and replace the entire `confirmSaveTemplate` function:

```javascript
async function confirmSaveTemplate() {
  const name = document.getElementById('tmpl-name-input').value.trim();
  if (!name) { document.getElementById('tmpl-name-input').focus(); return; }
  const key = 'tmpl_' + Date.now();
  await db.ref('templates/' + key).set({ name, questions: questionBank, savedAt: Date.now() });
  closeSaveTemplate();
  showNotif(`Template "${name}" saved!`);
  // Auto-select the newly saved template
  setTimeout(() => { selectedTemplateKey = key; tmplComboClose(); onTemplateSelect(); }, 300);
}
```

Replace with:

```javascript
async function confirmSaveTemplate() {
  const name = document.getElementById('tmpl-name-input').value.trim();
  if (!name) { document.getElementById('tmpl-name-input').focus(); return; }
  const key = 'tmpl_' + Date.now();
  await db.ref('templates/' + key).set({ name, questions: questionBank, savedAt: Date.now() });
  lastSavedTemplateName = name;
  closeSaveTemplate();
  showNotif(`Template "${name}" saved!`);
  document.getElementById('tmpl-submit-row').classList.add('open');
  setTimeout(() => { selectedTemplateKey = key; tmplComboClose(); onTemplateSelect(); }, 300);
}
```

- [ ] **Step 2: Add closeSubmitRow and submitToCommunity functions**

Find the line:

```javascript
async function deleteSelectedTemplate() {
```

Insert the following **before** that line:

```javascript
function closeSubmitRow() {
  document.getElementById('tmpl-submit-row').classList.remove('open');
}

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

- [ ] **Step 3: Commit**

```bash
cd /tmp/trivia-night-work
git add trivia_night_host_dashboard.html
git commit -m "feat: add Submit to Community button and GitHub PR flow"
```

---

## Task 7: Update README tech stack table

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update Hosting row in tech stack table**

In `/tmp/trivia-night-work/README.md`, find:

```markdown
| Hosting | GitHub Pages or any static file server |
```

Replace with:

```markdown
| Hosting | GitHub Pages, Firebase Hosting, or any static file server |
```

- [ ] **Step 2: Commit and push template repo**

```bash
cd /tmp/trivia-night-work
git add README.md
git commit -m "docs: update hosting options in tech stack table"
git push origin main
```

---

## Task 8: Fill in COMMUNITY_PAT (manual prerequisite)

**Files:**
- Modify: `trivia_night_host_dashboard.html` (in `dehlia24/trivia-night` on GitHub, after push)

> This task requires the fine-grained PAT from the Prerequisites section. Complete that step first.

- [ ] **Step 1: Pull latest from template repo**

```bash
cd /tmp/trivia-night-work
git pull origin main
```

- [ ] **Step 2: Replace YOUR_PAT_HERE with the real token**

In `/tmp/trivia-night-work/trivia_night_host_dashboard.html`, find:

```javascript
const COMMUNITY_PAT = 'YOUR_PAT_HERE'; // fine-grained PAT: contents+PRs write on dehlia24/trivia-night only
```

Replace `YOUR_PAT_HERE` with the actual token value (keep the surrounding quotes).

- [ ] **Step 3: Commit and push**

```bash
cd /tmp/trivia-night-work
git add trivia_night_host_dashboard.html
git commit -m "config: add community PAT for Submit to Community feature"
git push origin main
```

---

## Task 9: Sync dashboard changes to trivia-night-live

**Files:**
- Modify: `trivia-night-live/trivia_night_host_dashboard.html`

- [ ] **Step 1: Clone live repo**

```bash
git clone https://github.com/Dehlia24/trivia-night-live.git /tmp/trivia-night-live-work
```

- [ ] **Step 2: Copy the modified dashboard HTML from the template repo**

```bash
cp /tmp/trivia-night-work/trivia_night_host_dashboard.html /tmp/trivia-night-live-work/trivia_night_host_dashboard.html
```

- [ ] **Step 3: Verify the real Firebase config is still present in the copied file**

```bash
grep "trivia-127b9" /tmp/trivia-night-live-work/trivia_night_host_dashboard.html | head -5
```

Expected output: several lines containing `trivia-127b9` (the real project ID). If none appear, the template's placeholder config was copied over — stop and resolve before continuing.

- [ ] **Step 4: Commit and push**

```bash
cd /tmp/trivia-night-live-work
git add trivia_night_host_dashboard.html
git commit -m "feat: sync community question banks feature from template repo"
git push origin main
```

Expected: GitHub Actions auto-deploys to `https://trivia-127b9.web.app` within ~1 minute.

- [ ] **Step 5: Verify deployment**

Wait ~90 seconds, then check:

```bash
curl -s "https://trivia-127b9.web.app/trivia_night_host_dashboard.html" | grep -c "COMMUNITY_INDEX_URL"
```

Expected output: `1`

---

## Self-Review Notes

- **Spec coverage:** All sections covered — file structure (Tasks 1), contribution infrastructure (Task 2), dropdown sections (Tasks 3–4), lazy-load (Task 5), Submit to Community (Task 6), both repos updated (Tasks 7–9), PAT prerequisite documented.
- **No placeholders:** All code is complete.
- **Type consistency:** `communityBanks` array used consistently across `loadCommunityBanks`, `tmplComboGetFiltered`, `tmplComboRenderList`, `tmplComboClose`, `onTemplateSelect`, `loadSelectedTemplate`. `COMMUNITY_PAT`, `COMMUNITY_REPO`, `COMMUNITY_FILE_BASE`, `COMMUNITY_INDEX_URL` constants defined in Task 4 Step 1 and used in Tasks 4–6. `lastSavedTemplateName` defined in Task 4 Step 1, set in Task 6 Step 1, read in Task 6 Step 2.
