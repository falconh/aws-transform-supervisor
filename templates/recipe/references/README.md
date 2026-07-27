# references/

Reference material for this Recipe. ATX reads these files during the transformation, so
what goes here directly shapes output quality.

Replace this file with your own material.

## Highest value first

1. **Before/after example code** — the single most effective reference. Show a real snippet
   in its old form and its migrated form. Cover the edge cases, not just the happy path.
2. **Human-readable migration guides** — the upstream project's own upgrade guide, or your
   team's internal one.
3. **API documentation** — for the libraries or frameworks involved.

## Service limits

- **Text only**: `.md`, `.html`, `.txt`, and source files. Binary files, images and rich text
  (`.pdf`, `.png`, `.docx`) are **not** supported. Extract the text and use that instead.
- **10 MB total** across everything in this directory.
- Prefer a few descriptively named files over many small ones — concatenate related material.

## Naming

Name files for what they contain, not their origin: `sdk-v2-migration-guide.md` beats
`doc1.md`. The file names are part of what ATX uses to decide what to read.
