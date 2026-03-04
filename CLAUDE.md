# Naturalisation Revision - PDF to Markdown Pipeline

## Project Structure

```
docs/
├── <source>.pdf              # Original PDF
├── <source>.md               # Extracted raw markdown
└── <source>/
    └── images/               # Extracted images (pageNN_NN.jpg/png)
```

## Pipeline

### Prerequisites

```bash
pip install pdfplumber pypdf
```

### Step 1 — Extract text to markdown

Uses **pdfplumber** for best text extraction with layout preservation.

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

Uses **pypdf** to extract embedded images, named by page and index.

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

### Quick run (both steps)

Replace `<source>` with the PDF filename (without extension):

```bash
SOURCE=livret-de-citoyen
PDF=docs/${SOURCE}.pdf

# Text
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

# Images
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

## Notes

- **pdfplumber** handles French accents and layout correctly; pages are separated by `---`
- **pypdf** extracts embedded images as-is (no re-encoding); filenames encode page and position
- For a new PDF source, copy it to `docs/`, rename to kebab-case, and run the pipeline
- The current source is the *Livret du Citoyen* (February 2022 edition) used for naturalisation revision
