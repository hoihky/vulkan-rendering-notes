# Building a Real-Time Renderer with Vulkan

A **teaching book** on designing and implementing a real-time forward renderer with Vulkan — built alongside the **Spark** reference engine. Each chapter explains theory, design decisions, and step-by-step implementation with real Spark source citations.

## Build the site

Requires a sibling [MDWeb](https://github.com/hoihky/MDWeb) checkout (override with `MDWEB_DIR`).

```bash
./build-site.sh
npx serve site
```

Output lands in `site/`. Set `OUT_DIR` to change the destination.

## Regenerate markdown from scripts (optional)

Use this only when bulk-updating chapters from the Python content modules:

```bash
python3 generate_book.py   # overwrites part-*/ and appendix/ markdown
./build-site.sh
```

Set `SPARK_ROOT=/path/to/spark` so Spark source citations resolve correctly during generation.

## Repository layout

| Path | Purpose |
|------|---------|
| `part-*/`, `appendix/` | Canonical chapter markdown |
| `index.md` | Book landing page |
| `build-site.sh` | MDWeb static site build |
| `generate_book.py` | Regenerate markdown from `scripts/parts/content/` |
| `scripts/book_core.py` | Shared file-writing helpers |
| `scripts/parts/content/` | Chapter bodies as Python strings |
| `scripts/parts/part*.py` | Part generators wired by `generate_book.py` |
| `site/` | Generated HTML (not edited by hand) |

## Chapter template (Part I style)

1. Build-Along Milestone
2. The problem we are solving
3. Core concepts (subsections with tables and diagrams)
4. Design decisions (concept → decision → why → Spark location)
5. Implementation walkthrough (numbered steps + Spark code excerpts)
6. How this connects to the rest of your engine
7. Pitfalls and debugging
8. Exercises
9. Summary

## Preview

```bash
npx serve site
```
