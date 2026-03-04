# Naturalisation Revision Quiz

Interactive quiz app for French naturalisation interview preparation, built from the *Livret du Citoyen* (February 2022 edition).

## Project Structure

```
index.html                        # Self-contained quiz app (HTML + CSS + JS)
docs/
├── questions.md                  # 88 MCQ questions in markdown (source of truth)
├── livret-de-citoyen.pdf         # Original PDF source
├── livret-de-citoyen.md          # Raw extracted markdown
└── livret-de-citoyen/
    ├── README.md                 # Chapter index
    ├── 00-avant-propos.md        # Formatted chapters (00–09)
    ├── 01-republique-francaise.md
    ├── ...
    ├── 09-declaration-droits-homme.md
    └── images/                   # Extracted images (pageNN_NN.jpg/png)
```

## Pipeline

```
PDF ──Step 1──> raw markdown (.md)
    ──Step 2──> images (images/)
    ──Step 3──> formatted chapter files (00–09.md)  [manual]
    ──Step 4──> questions.md                        [manual, from chapters]
    ──Step 5──> index.html QUESTIONS array            [manual, from questions.md]
```

### Prerequisites

```bash
pip install pdfplumber pypdf
```

### Step 1 — Extract text to markdown

Uses **pdfplumber** for text extraction with layout preservation.

```bash
python3 -c "
import pdfplumber, sys
pdf_path = sys.argv[1]
out_path = pdf_path.rsplit('.', 1)[0] + '.md'
with pdfplumber.open(pdf_path) as pdf:
    pages = [p.extract_text() for p in pdf.pages if p.extract_text()]
    with open(out_path, 'w') as f:
        f.write('\n\n---\n\n'.join(pages))
    print(f'{out_path}: {len(pages)} pages, {sum(len(p) for p in pages)} chars')
" docs/<source>.pdf
```

### Step 2 — Extract images

Uses **pypdf** to extract embedded images.

```bash
python3 -c "
from pypdf import PdfReader
import os, sys
pdf_path = sys.argv[1]
out_dir = os.path.join(os.path.dirname(pdf_path), pdf_path.rsplit('.', 1)[0].split('/')[-1], 'images')
os.makedirs(out_dir, exist_ok=True)
reader = PdfReader(pdf_path)
count = 0
for i, page in enumerate(reader.pages):
    for j, img in enumerate(page.images):
        ext = os.path.splitext(img.name)[1]
        fname = f'page{i+1:02d}_{j+1:02d}{ext}'
        with open(os.path.join(out_dir, fname), 'wb') as f:
            f.write(img.data)
        count += 1
print(f'{count} images -> {out_dir}')
" docs/<source>.pdf
```

### Step 3 — Format chapter files (manual)

Split the raw markdown into individual chapter files (`00-avant-propos.md` through `09-declaration-droits-homme.md`). Clean up OCR artifacts, add proper headings, and fix formatting. Create a `README.md` with the table of contents.

### Step 4 — Write questions (manual)

Create `docs/questions.md` with MCQ questions based on the chapter content. Each question follows this format:

```markdown
## {N}. {Theme Name}

### Q{id}. {Question text}

- A) {option}
- B) {option}
- C) {option}
- D) {option}

<details><summary>Réponse</summary>

**{letter}) {correct answer text}**

</details>
```

**Rules:**
- 4 options per question (A/B/C/D)
- Exactly 1 correct answer
- Group questions by theme (matching chapter topics)
- Questions should test key facts from the source material

**Current themes (8):**

| # | Theme | Chapter |
|---|-------|---------|
| 1 | Principes et Valeurs de la République | 01 |
| 2 | Droits et Devoirs des Citoyens | 02 |
| 3 | Organisation Politique | 03 |
| 4 | Collectivités Locales | 04 |
| 5 | Repères Historiques | 05 |
| 6 | Apports Étrangers | 06 |
| 7 | La France en Europe et dans le Monde | 07 |
| 8 | Géographie et Territoire | 08 |

### Step 5 — Update index.html QUESTIONS array (manual)

The `QUESTIONS` constant in `index.html` is a JSON array. Each entry:

```javascript
{
  "id": 1,                          // Sequential, matches Q{id} in questions.md
  "theme": "Theme Name",            // Must match one of the 8 themes above
  "question": "Question text?",     // Same as questions.md
  "options": ["A", "B", "C", "D"],  // Same order as questions.md
  "correct": 0,                     // 0-based index (A=0, B=1, C=2, D=3)
  "explanation": "Why this answer"   // Optional — can be "" if none
}
```

**Mapping from questions.md:**
- Letter A → index 0, B → 1, C → 2, D → 3
- Theme name comes from the `## {N}. {Theme Name}` heading (without the number prefix)
- Explanation is optional enrichment not present in questions.md

### Quick run (Steps 1–2)

```bash
SOURCE=livret-de-citoyen
PDF=docs/${SOURCE}.pdf

python3 -c "
import pdfplumber, sys
pdf_path = sys.argv[1]
out_path = pdf_path.rsplit('.', 1)[0] + '.md'
with pdfplumber.open(pdf_path) as pdf:
    pages = [p.extract_text() for p in pdf.pages if p.extract_text()]
    with open(out_path, 'w') as f:
        f.write('\n\n---\n\n'.join(pages))
    print(f'{out_path}: {len(pages)} pages')
" "$PDF"

python3 -c "
from pypdf import PdfReader
import os, sys
pdf_path = sys.argv[1]
out_dir = os.path.join('docs', os.path.basename(pdf_path).rsplit('.', 1)[0], 'images')
os.makedirs(out_dir, exist_ok=True)
reader = PdfReader(pdf_path)
count = 0
for i, page in enumerate(reader.pages):
    for j, img in enumerate(page.images):
        ext = os.path.splitext(img.name)[1]
        with open(os.path.join(out_dir, f'page{i+1:02d}_{j+1:02d}{ext}'), 'wb') as f:
            f.write(img.data)
        count += 1
print(f'{count} images -> {out_dir}')
" "$PDF"
```

## Quiz Features

- **88 MCQ questions** across 8 themes with 3-level randomization
- **Theme filtering** — select specific topics on the start screen
- **Configurable quiz size** — presets (10/20/30/all) or custom number
- **Immediate feedback** — correct/wrong indicators with explanations
- **Session persistence** — resume interrupted quizzes via localStorage
- **Score history** — last 5 results shown on start screen
- **Results review** — per-theme breakdown, pass/fail verdict, full answer review
- **Retry wrong answers** — targeted practice on mistakes
- **Dark/light theme** — manual toggle + OS preference detection
- **Keyboard shortcuts** — 1-4 (answer), Enter (next), Left (back), Escape (quit)
- **Accessibility** — ARIA roles, live regions, focus management, screen reader support

## Hosting

Hosted on **GitHub Pages** from the `master` branch root (`/`).

- **URL:** `https://elkuno213.github.io/quiz-naturalisation/`
- **Setup:** Repo Settings → Pages → Source: Deploy from branch → `master` / `/ (root)`
- No build step needed — `index.html` is served directly as a static file

## Notes

- **pdfplumber** handles French accents and layout correctly; pages separated by `---`
- **pypdf** extracts embedded images as-is (no re-encoding)
- For a new PDF source: copy to `docs/`, rename to kebab-case, run Steps 1–2, then manually create chapter files and questions
- The current source is the *Livret du Citoyen* (February 2022 edition)
