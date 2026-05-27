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
