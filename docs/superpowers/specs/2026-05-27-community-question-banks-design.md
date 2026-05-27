# Community Question Banks — Design Spec
**Date:** 2026-05-27  
**Repo:** `dehlia24/trivia-night` (public template) + `dehlia24/trivia-night-live` (personal site)

---

## Overview

Add a community question bank system to the public trivia-night template. Community banks live as static JSON files in the GitHub repo. Anyone using the template sees them in the host dashboard's template dropdown. Anyone can submit their own bank via the "Submit to Community" button, which opens a GitHub PR for review. Manual PR contributions are also supported.

---

## File Structure

```
question-banks/
├── index.json                  ← master list fetched at dashboard load
├── general-knowledge.json      ← starter bank (ships with template)
└── <contributor-bank>.json     ← one file per community contribution
```

### `index.json` format
```json
[
  {
    "name": "General Knowledge",
    "author": "dehlia24",
    "description": "A broad mix across science, history, and pop culture",
    "file": "general-knowledge.json",
    "questionCount": 15
  }
]
```

### Individual bank file format
```json
{
  "name": "General Knowledge",
  "author": "dehlia24",
  "description": "A broad mix across science, history, and pop culture",
  "questions": [
    {
      "question": "What is the chemical symbol for gold?",
      "options": ["Au", "Ag", "Fe", "Pb"],
      "correct": 0,
      "category": "Science"
    }
  ]
}
```

Each question must have exactly 4 options and `correct` must be 0–3.

---

## Dashboard Changes (`trivia_night_host_dashboard.html`)

### Dropdown — two sections
The existing template combo box gains two labeled sections:

```
┌─ Search or select a template ──────────────────────┐
│ ── Community Banks ─────────────────────────────── │
│   General Knowledge  (15q · by dehlia24)            │
│   80s Pop Music      (30q · by someone)             │
│ ── My Saved Templates ──────────────────────────── │
│   Game Night #4      (20q · Jan 3)                  │
└─────────────────────────────────────────────────────┘
```

### Load time
1. Fetch `https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/index.json`
2. Populate "Community Banks" section in the dropdown
3. If fetch fails, show "Community banks unavailable" in that section — personal templates continue working normally

### Loading a community bank
When the user selects a community bank and clicks **Load**:
1. Fetch `https://raw.githubusercontent.com/dehlia24/trivia-night/main/question-banks/<file>`
2. Load `questions` array into `questionBank` and sync to Firebase (same as existing template load)

---

## "Submit to Community" Flow

After a user saves a template to Firebase via the existing **Save as template** flow, the save confirmation row gains a **"Submit to Community"** button that appears inline next to the existing "✓ saved" feedback text.

### Steps on click
1. Sanitize template name to a filename: `"80s Pop Music"` → `80s-pop-music.json`
2. Create branch `community/bank-{timestamp}` on `dehlia24/trivia-night` via GitHub API
3. Commit bank JSON file to that branch
4. Fetch current `index.json` from `main`, append new entry, commit updated `index.json` to the branch
5. Open PR titled `"Community bank: {template name}"` targeting `main`
6. Show success notification with direct link to the PR

### On failure
If any API call fails, show a notification with a fallback link to open a PR manually on GitHub.

### GitHub PAT
A fine-grained PAT scoped to `contents:write` + `pull_requests:write` on `dehlia24/trivia-night` only is stored as a constant in the template HTML (`COMMUNITY_PAT`). It is intentionally public — worst-case abuse is spam PRs that the repo owner never merges. The token can be revoked instantly. The README documents its purpose and minimal scope.

---

## Contribution Infrastructure

### `CONTRIBUTING.md`
- Explains the JSON format for bank files
- Two-step manual contribution process: add `<bank>.json` + update `index.json`
- Validation rules: exactly 4 options per question, `correct` is 0–3, all fields required
- Linked from the main README

### `.github/PULL_REQUEST_TEMPLATE.md`
Auto-populates on every PR with a checklist:
- [ ] Added `question-banks/<my-bank>.json`
- [ ] Updated `question-banks/index.json` with correct metadata
- [ ] All questions have exactly 4 options
- [ ] `correct` field is 0–3 for every question
- [ ] `questionCount` in `index.json` matches actual question count

### `question-banks/general-knowledge.json`
Starter bank of 15 questions shipped with the template. Ensures the community section is never empty on a fresh install and serves as a concrete format example.

---

## Repos Affected

| Repo | Changes |
|------|---------|
| `dehlia24/trivia-night` (template) | `question-banks/` directory, `CONTRIBUTING.md`, PR template, dashboard HTML changes |
| `dehlia24/trivia-night-live` (personal) | Dashboard HTML changes only (community banks point to the template repo) |

The community banks always point to `dehlia24/trivia-night` regardless of which site is loaded — submissions from the live site create PRs on the public template repo.

---

## Prerequisites (manual, before implementation)

1. Create a GitHub fine-grained PAT at `github.com/settings/personal-access-tokens/new`:
   - Resource owner: `Dehlia24`
   - Repository access: `dehlia24/trivia-night` only
   - Permissions: `Contents: Read and write`, `Pull requests: Read and write`
2. Copy the token value — it goes into the `COMMUNITY_PAT` constant in the HTML

---

## Out of Scope
- Validation of submitted banks beyond the PR checklist (no CI linting)
- User accounts or attribution beyond the `author` field in the JSON
- Deleting or editing community banks from the dashboard (managed via GitHub)
