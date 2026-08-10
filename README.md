# Building a Real-Time Renderer with Vulkan

A **teaching book** on designing and implementing a real-time forward renderer with Vulkan — built alongside the **Spark** reference engine. Each chapter explains theory, design decisions, and step-by-step implementation with real Spark source citations.

**Read the book online:** [https://hoihky.github.io/vulkan-rendering-notes/site/](https://hoihky.github.io/vulkan-rendering-notes/site/)

Use the HTML site for the best reading experience — navigation, search, syntax highlighting, and chapter-to-chapter links are all built in. The markdown files in this repository are the source; the site is the recommended way to follow along.

## Build the site locally

Requires a sibling [MDWeb](https://github.com/hoihky/MDWeb) checkout (override with `MDWEB_DIR`).

```bash
./build-site.sh
npx serve site
```

Output lands in `site/`. Set `OUT_DIR` to change the destination. Then open [http://localhost:3000](http://localhost:3000) (or the port `serve` prints).
