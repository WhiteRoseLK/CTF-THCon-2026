# P4t4t0rz at the Library

- **Category:** Cryptography / Misc
- **Points:** 136

```
Knowledge is relative
```

*(submitted without the `THC{}` wrapper)*

## TL;DR

- The challenge ships a PDF of Aristotle's collected works and a single ciphertext: `30:7/260:22/27:5`.
- It's a **book cipher** in `page:word` form. Each token names a page (1-indexed) and a word position on that page (also 1-indexed).
- One gotcha: page 27 starts with a `Part 7` section header, which inflates the word count if you don't strip it. Filter "Part N" lines before splitting.
- The three words spell out a coherent Aristotelian aphorism: *Knowledge is relative*.

## 1. Reading the cipher

The cipher is the smoking gun:

```
30:7/260:22/27:5
```

Three `N:M` pairs separated by `/`. The most natural reading — and the only one that makes sense given a book attachment — is **page : word**. No other format produces interpretable output.

| Token | Page | Word |
|---|---|---|
| `30:7` | 30 | 7 |
| `260:22` | 260 | 22 |
| `27:5` | 27 | 5 |

## 2. Extracting words from the PDF

`pdfplumber` is the right tool: it preserves page boundaries reliably, unlike `pdftotext` which can shift content. The recipe per token is:

1. Open the page (1-indexed → 0-indexed in the API).
2. Strip section headers (more on that below).
3. `split()` on whitespace.
4. Take index `word − 1`.

```python
import pdfplumber

def word_on_page(pdf, page_1based: int, word_1based: int) -> str:
    text = pdf.pages[page_1based - 1].extract_text()
    # Strip "Part N" lines so they don't pollute the word count
    cleaned = " ".join(
        line for line in text.splitlines()
        if not (line.strip().startswith("Part ")
                and len(line.strip().split()) == 2
                and line.strip().split()[1].isdigit())
    )
    return cleaned.split()[word_1based - 1]
```

## 3. The page-27 trap

Run the naive version (no header filter) and you get garbage on token `27:5`. Inspecting the page top reveals why:

```
Part 7
[body text starts here]
```

If `Part 7` is counted as two words, the 5th "word" lands inside or just past the heading and yields a meaningless token. The body text is what the cipher refers to, so any `Part N` line must be excluded before the split.

The other two tokens (`30:7`, `260:22`) land on body pages with no header — they decode the same way regardless of filtering, but the filter is harmless on those.

## 4. Decoding

```python
import pdfplumber

CIPHER = "30:7/260:22/27:5"
PDF = "aristotle.pdf"

def word_on_page(pdf, page, word):
    text = pdf.pages[page - 1].extract_text()
    cleaned = " ".join(
        line for line in text.splitlines()
        if not (line.strip().startswith("Part ")
                and len(line.strip().split()) == 2
                and line.strip().split()[1].isdigit())
    )
    return cleaned.split()[word - 1]

with pdfplumber.open(PDF) as pdf:
    out = []
    for tok in CIPHER.split("/"):
        p, w = (int(x) for x in tok.split(":"))
        word = word_on_page(pdf, p, w)
        print(f"  {tok} → {word}")
        out.append(word)
    print(" ".join(out))
```

```
  30:7   → Knowledge
  260:22 → is
  27:5   → relative
Knowledge is relative
```

The submission is the bare sentence — no `THC{}` envelope.

## 5. Methodology

- **`N:M / N:M / ...` over a book attachment is almost always `page:word`.** Try the obvious format before reaching for anything more elaborate.
- **Inspect the top of each referenced page.** Section headers, page numbers, and running titles routinely break the natural word count. A 30-second sanity print of `page.extract_text()[:200]` saves an hour of debugging.
- **Use `pdfplumber` rather than `pdftotext` for per-page word extraction.** PDF reflow can shift text across pages in `pdftotext`, breaking the page-1 alignment that book ciphers depend on.
- **Don't assume the flag wraps in `THC{}`.** When the cipher decodes to a coherent English sentence, that *is* the flag.
