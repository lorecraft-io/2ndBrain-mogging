# Claude Projects → Vault Sync Guide

How to route the contents of a single Claude.ai Project (or ChatGPT folder) into a matching project folder inside your mogged vault. The per-project subfolder pattern (`conversations/` + `knowledge/` + `assets/`) is the same one `/import-claude` uses when it writes; this doc is here so you can replicate the layout manually or understand what the skill is doing under the hood.

---

## Per-project layout

```
05-Projects/
  PROJECT-NAME/
    PROJECT-NAME.md      ← index note (knowledge base + conversation log + related links)
    conversations/       ← all conversation markdown files for this project
    knowledge/           ← all text-based knowledge (md, txt, jsx, html, sh) + converted docx/pptx/xlsx
    assets/              ← real binaries only (PDFs, zips — things that can't be markdown)
```

The index filename **must match the folder name exactly** — `PARZVL/PARZVL.md`, not `PARZVL/PARZVL-Index.md`. This is the vault-wide rule (see [`CLAUDE.md`](../CLAUDE.md)).

---

## File routing rules

| From the export | Extension | Goes to | Action |
|---|---|---|---|
| conversations | `.md` | `conversations/` | Copy as-is |
| knowledge | `.md`, `.txt`, `.jsx`, `.html`, `.sh` | `knowledge/` | Copy as-is |
| knowledge | `.docx` | `knowledge/` | Convert via `pandoc`, delete the `.docx` |
| knowledge | `.pptx` | `knowledge/` | Convert via `pandoc`, delete the `.pptx` |
| knowledge | `.xlsx` | `knowledge/` | Convert via `xlsx2csv` (or `openpyxl`), delete the `.xlsx` |
| knowledge | `.pdf` | `assets/` | Copy — but validate magic bytes first (see below) |
| knowledge | `.zip` | `assets/` | Copy as-is |
| system-prompt / project instructions | any | `knowledge/` | Copy as `system-prompt.md` |

## Validation rules

1. **Fake PDFs:** Claude.ai exports sometimes save plain text with a `.pdf` extension. Check that every `.pdf` starts with the magic bytes `%PDF` (hex `25504446`). If it's actually text, rename to `.md` and put in `knowledge/`.
2. **Fake DOCX / XLSX / PPTX:** These are zip archives under the hood — confirm they start with PK zip header (hex `504b0304`). If not, the file is corrupt or mislabeled.
3. **Empty files:** Flag any 0-byte files; don't copy.
4. **Dedup:** If the filename already exists in the destination, skip (don't overwrite without explicit user approval).

## Project index note format

Every `PROJECT-NAME.md` should have:

1. **Frontmatter** — `title`, `date`, `type: project`, `tags`, optional `status`.
2. **Description** — one-paragraph what-the-project-is.
3. **Knowledge Base** — `[[wikilinks]]` to every file in `knowledge/`. **Never** use backtick formatting for filenames that should be clickable.
4. **Conversation Log** — bulleted `[[YYYY-MM-DD-<slug>]] — short topic note` per conversation.
5. **Related** — bidirectional `[[wikilinks]]` to related projects, owners, organizations. `/tether` reads this section.

### Wikilink rules

- Use `[[filename]]` with **no extension** for `.md` files — Obsidian resolves by filename vault-wide.
- Use `[[filename.pdf]]` for PDFs — the extension is required for non-`.md` targets.
- Never put wikilinks inside a table cell — Obsidian's graph view doesn't see them there. Use bullet lists.
- Related links are **bidirectional**: if A links to B, B must link back to A. `/tether` fixes drift.

## Conversion commands

```bash
# .docx / .pptx → markdown
pandoc input.docx -t markdown -o output.md

# .xlsx → CSV (then wrap as a markdown table block)
xlsx2csv input.xlsx > output.csv

# Validate: is this actually a PDF?
xxd -l 4 file.pdf | grep "2550 4446"

# Validate: is this actually a zip-based format (.docx / .xlsx / .pptx)?
xxd -l 4 file.docx | grep "504b 0304"
```

## Scars from shipping this before

- Claude.ai exports occasionally save plain-text content with a `.pdf` extension. Validate every PDF.
- Obsidian renders PDFs natively, but **not** `.docx` / `.pptx` / `.xlsx` — always convert those to markdown so the vault stays readable.
- Knowledge Base sections must use `[[wikilinks]]`, never backticks — otherwise files aren't clickable in Obsidian.
- Related links must be bidirectional. If A links to B, B must link back to A.
- Tables with wikilinks inside them don't appear in Obsidian's graph view. Use bullet lists.
- `pandoc` handles `.docx` → markdown cleanly; the edge cases are mostly footnotes and tables-in-tables.

## Related

- [`PARSING-GUIDE.md`](PARSING-GUIDE.md) — folder routing rules for every skill that writes
- [`github-vault-guide.md`](github-vault-guide.md) — how to link project folders to GitHub repos
- `/import-claude` — the skill that runs this layout automatically
- `/tether` — fixes broken bidirectional links between project folders and related indexes
