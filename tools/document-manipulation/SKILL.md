---
name: document-manipulation
description: Techniques for extracting text, rendering images, and translating Chinese labels in PDF technical manuals using PyMuPDF (fitz) and PIL. Use this skill when asked to extract content from PDFs, render pages as images, replace or overlay text in images, or translate in-place labels in scanned/vector PDFs.
---

# Document Manipulation

Techniques learned while producing an English version of the Huananzhi H12D-8D manual.
Applies to any bilingual or Chinese-primary technical PDF.

---

## 1. Extracting Text and Spans from a PDF

```python
import fitz  # PyMuPDF

doc = fitz.open("manual.pdf")
page = doc[3]  # 0-indexed

# All text blocks with bounding boxes
blocks = page.get_text("dict")["blocks"]
for block in blocks:
    for line in block.get("lines", []):
        for span in line.get("spans", []):
            print(span["text"], span["bbox"], span["color"])
```

### Key span attributes
| Attribute | Type | Notes |
|-----------|------|-------|
| `text` | str | Raw string, may be Unicode (Chinese, special chars) |
| `bbox` | tuple(4) | `(x0, y0, x1, y1)` in **72-DPI PDF points** |
| `color` | int | Packed RGB: `0xRRGGBB` |
| `size` | float | Font size in points |
| `font` | str | Font name |

### Important: PDF points vs. pixel coordinates

When rendering at non-72 DPI you **must scale bboxes**:

```python
SCALE = 150 / 72  # 150 DPI render
px0 = int(span["bbox"][0] * SCALE)
py0 = int(span["bbox"][1] * SCALE)
px1 = int(span["bbox"][2] * SCALE)
py1 = int(span["bbox"][3] * SCALE)
```

---

## 2. Rendering Pages as PNG Images

```python
import fitz

SCALE = 150 / 72  # 150 DPI
mat = fitz.Matrix(SCALE, SCALE)

doc = fitz.open("manual.pdf")
page = doc[3]
pix = page.get_pixmap(matrix=mat, alpha=False)
pix.save("page3.png")
```

- Use `alpha=False` for RGB output without transparency channel.
- 150 DPI (`SCALE = 150/72`) is a good balance: readable text, reasonable file size.
- 300 DPI gives sharper results for printing but 4× larger files.

---

## 3. Translating Chinese Text In-Place with PIL Overlays

### Why not PyMuPDF redaction?

`page.add_redact_annot()` + `page.apply_redactions()` fails silently for background color:

- `sample_bg()` (internal PyMuPDF helper) samples a pixel **3px above** the bbox → often hits
  table border lines → returns `(0, 0, 0)` → text rendered invisible on black background.
- U+2212 "−" (MINUS SIGN) is not in the built-in `helv` font → renders as empty box.

**Always use PIL overlay on a pre-rendered PNG instead.**

### PIL overlay pipeline

```python
import fitz
import numpy as np
from PIL import Image, ImageDraw, ImageFont

SCALE = 150 / 72
FONT_PATH = "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf"

TRANSLATIONS = {
    "中文原文": "English translation",
    # ... full dict
}

doc = fitz.open("manual.pdf")
page = doc[3]
mat = fitz.Matrix(SCALE, SCALE)
pix = page.get_pixmap(matrix=mat, alpha=False)

# Convert to PIL
img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
draw = ImageDraw.Draw(img)

# Process all spans
blocks = page.get_text("dict")["blocks"]
for block in blocks:
    for line in block.get("lines", []):
        for span in line.get("spans", []):
            chinese = span["text"].strip()
            if chinese not in TRANSLATIONS:
                continue
            english = TRANSLATIONS[chinese]

            # Scale bbox to pixel space
            bx0 = int(span["bbox"][0] * SCALE)
            by0 = int(span["bbox"][1] * SCALE)
            bx1 = int(span["bbox"][2] * SCALE)
            by1 = int(span["bbox"][3] * SCALE)
            bw = bx1 - bx0
            bh = by1 - by0

            # Sample background color (2px inner margin to avoid border pixels)
            margin = 2
            region = np.array(img.crop((
                bx0 + margin, by0 + margin,
                max(bx1 - margin, bx0 + margin + 1),
                max(by1 - margin, by0 + margin + 1)
            )))
            
            # Choose percentile based on original text luminance
            r, g, b = (span["color"] >> 16) & 0xFF, (span["color"] >> 8) & 0xFF, span["color"] & 0xFF
            lum = 0.299 * r + 0.587 * g + 0.114 * b
            pct = 80 if lum < 128 else 20  # light bg for dark text; dark bg for light text
            bg = tuple(int(np.percentile(region[:, :, ch].flatten(), pct)) for ch in range(3))

            # Fill bbox with background color
            draw.rectangle([bx0, by0, bx1, by1], fill=bg)

            # Find largest font size that fits
            font_size_pts = span["size"]
            for scale_factor in [1.0, 0.85, 0.70, 0.60, 0.50, 0.40, 0.35]:
                fs = max(6, int(font_size_pts * SCALE * scale_factor))
                font = ImageFont.truetype(FONT_PATH, fs)
                bb = draw.textbbox((0, 0), english, font=font)
                tw, th = bb[2] - bb[0], bb[3] - bb[1]
                if tw <= bw and th <= bh:
                    break

            tx = bx0 + (bw - tw) // 2
            ty = by0 + (bh - th) // 2
            draw.text((tx, ty), english, fill=(r, g, b), font=font)

img.save("output.png")
```

### Known gotchas

| Problem | Root cause | Fix |
|---------|-----------|-----|
| Text invisible on black bg | `sample_bg()` hit table border 3px above bbox | Use PIL percentile approach above |
| U+2212 "−" renders as empty box | Not in helv/built-in fonts | Use DejaVuSans; replace "−" with ASCII "-" |
| Very small cells (≤25 px wide) | Table cells too narrow for English | Use scale_factor ≤ 0.35 or draw replacement table |
| Audio/diagram labels not found as spans | Labels are vector graphics or rasterized, not PDF text | Use `page.get_drawings()` / `page.get_images()` to locate; draw English directly at known coords |
| Span bbox in PDF points, image in pixels | Scale mismatch | Always multiply bbox by `SCALE = DPI/72` |

---

## 4. Identifying Non-Text Content (Diagrams, Images)

Some labels in PDFs are embedded as vector drawings or rasterised images, not text spans.
They won't appear in `get_text("dict")` at all.

```python
# Check vector drawings (lines, rects, curves with fill/stroke colors)
drawings = page.get_drawings()
for d in drawings:
    print(d["rect"], d["type"])

# Check embedded raster images
images = page.get_images(full=True)
for img in images:
    xref = img[0]
    base_image = doc.extract_image(xref)
    # base_image["image"] is the raw bytes; base_image["ext"] is "png"/"jpeg"/etc.

# Check all items on page (text + image + drawing)
for item in page.get_text("rawdict")["blocks"]:
    if item["type"] == 1:  # type 1 = image block
        print("Image block at", item["bbox"])
```

---

## 5. Finding a Page's Coordinate System

```python
page = doc[3]
print(page.rect)         # e.g. Rect(0, 0, 595, 842) — A4 in points
print(page.mediabox)     # Same as rect for most PDFs
```

---

## 6. Useful PyMuPDF Snippets

```python
# Count pages
print(len(doc))

# Extract all text as plain string
text = page.get_text("text")

# Search for a string and get its location
hits = page.search_for("Battery")
for rect in hits:
    print(rect)  # fitz.Rect with bbox

# Crop a region and save as image
clip = fitz.Rect(100, 200, 400, 500)  # in PDF points
pix = page.get_pixmap(matrix=mat, clip=clip, alpha=False)
pix.save("crop.png")
```

---

## References

- [PyMuPDF documentation](https://pymupdf.readthedocs.io/)
- [PIL/Pillow documentation](https://pillow.readthedocs.io/)
- [Huananzhi H12D-8D skill](../hardware/huananzhi-h12d-8d/SKILL.md) — real-world application of all techniques above
