---
name: styleguide-from-figma
description: "Given one or more Figma URLs, inspect the file with the Figma MCP server, extract design tokens (colors, typography, spacing, buttons, shadows, components), and write both a replicatable ./design.md styleguide and a ./styleguide.html visual preview (typography scale, color swatches, buttons, badges). Trigger when the user provides a figma.com URL and asks for a styleguide, style extraction, design tokens, to reverse-engineer a Figma file, or to 'make a design.md' / 'extract the style' / 'build a styleguide from this Figma'."
compatibility: Claude Code
metadata:
  author: Kasper Dolk
  version: "1.0"
---

# Styleguide From Figma

Turn any Figma design into a structured, replicatable `design.md` styleguide by inspecting the file with the Figma MCP server.

Sister skill of `styleguide-from-url`: same output shape, different data source. Running both against the same product (Figma source vs. shipped page) makes the design-vs-reality gap diffable.

## Input

The user supplies one or more Figma URLs. Optionally:
- Output paths (defaults: `./design.md` + `./styleguide.html` in the current working directory).
- A focus list (e.g. "just typography and colors").

Two artifacts are always produced from the same extraction pass:
1. `./design.md` — the token reference (markdown).
2. `./styleguide.html` — a visual preview page showing typography scale, color swatches, buttons, badges, spacing, radii, and shadows rendered from the extracted tokens.

If the user provides multiple Figma URLs, merge them into a **single** pair of outputs and dedupe tokens across files. Do not produce one file per link.

## Required MCP

This skill uses the official `figma` MCP server. The tools used are:
- `mcp__figma__get_variable_defs` — design tokens (variables) defined in the file; ground truth when present
- `mcp__figma__get_libraries` — linked design system libraries
- `mcp__figma__get_design_context` — primary reader; returns reference code + screenshot + tokens as CSS variables
- `mcp__figma__get_metadata` — node tree / structure
- `mcp__figma__get_screenshot` — visual reference image
- `mcp__figma__search_design_system` — search tokens/components by name (use if the file has a library and you need to resolve a specific token)
- `mcp__figma__get_figjam` — FigJam (`figma.com/board/...`) fallback

Rules:
- **NEVER** use `WebFetch` or `WebSearch` on `figma.com` URLs. Always go through the MCP tools.
- **NEVER** delegate Figma MCP calls to subagents (Explore, general-purpose, Plan, etc.). MCP tools are only available to the main loop. Call them yourself.
- If the `figma` MCP server is unavailable, stop and tell the user to enable it — do NOT guess styles.

## URL parsing

Extract `fileKey` and `nodeId` from the user's Figma URL before any MCP call:

- `figma.com/design/:fileKey/:fileName?node-id=:nodeId` → convert `-` to `:` in the `nodeId` (e.g. `1-23` → `1:23`).
- `figma.com/design/:fileKey/branch/:branchKey/:fileName` → use `branchKey` as `fileKey`.
- `figma.com/make/:makeFileKey/:makeFileName` → use `makeFileKey` as `fileKey`.
- `figma.com/board/:fileKey/:fileName` → FigJam file. FigJam boards don't have a styleguide in the normal sense; call `get_figjam` for context and report back to the user that the URL is a whiteboard, not a design file.

If `nodeId` is missing from the URL, call `get_metadata(fileKey)` to list top-level frames and pick a sensible root (e.g. the first page frame), or ask the user which node to start from.

## Process

### 1. Discover tokens first
For each Figma URL, in order:
1. Call `get_variable_defs(fileKey, nodeId)`. This returns design tokens explicitly defined in the file (colors, typography, spacing, radii, etc.). These are the **ground truth** — designer intent, not rendered approximations.
2. Call `get_libraries(fileKey)`. If the file uses a linked library, note it — some tokens may live upstream. If you need to resolve a specific token by name, use `search_design_system(fileKey, query)`.

If `get_variable_defs` returns an empty set, note "No variables defined in this file" in the output and fall back to extracting raw values from step 2.

### 2. Pull design context
Call `get_design_context(fileKey, nodeId)` for each URL. The response is React + Tailwind reference code enriched with contextual hints. Harvest from it:
- **CSS custom properties** — the designer-defined token layer (highest priority).
- **Colors** — hex values on backgrounds, text, borders.
- **Typography** — font-family, font-size, font-weight, line-height, letter-spacing per heading level and body.
- **Spacing** — padding/margin values on sections, cards, containers.
- **Radii / shadows** — border-radius and box-shadow values.
- **Button and link styles** — any element that looks like a CTA, link, or form control.

If the returned code is "loosely structured" (raw hex values + absolute positioning, no tokens), the file isn't using variables — lean harder on the screenshot and your own pattern recognition.

### 3. Capture visual reference
Call `get_screenshot(fileKey, nodeId)` for each URL. Note the returned image paths — they go into the final `design.md` as visual anchors.

### 4. Merge across links
When multiple Figma URLs are provided:
- Dedupe colors by normalized hex (case-insensitive, alpha split out).
- Dedupe fonts by (family, weight, size).
- Dedupe button styles by fingerprint: `(bg-color, text-color, border, border-radius, padding, font-size, font-weight)`.
- Dedupe spacing/radii/shadow values by exact string match.
- Merge variable definitions — if two files define the same variable name with different values, list both and flag the conflict.

### 5. Group colors
Use the same heuristic as the URL sibling:
- **Brand / Primary** — colors that appear on buttons, CTAs, primary links.
- **Neutrals** — grays, blacks, whites, near-whites used for text and surfaces.
- **Accents** — any remaining unique colors.
- **Borders** — colors found on borders/dividers.

Drive the grouping from where each color actually appears (buttons/links → brand; body/text → neutrals). Don't invent colors that aren't in the extracted set.

### 6. Write `./design.md`
Use the template below. Fill every section from the extracted data. Omit sections that are empty rather than leaving `TBD`. Keep values verbatim — if the file uses rem, write rem; if hex, write hex.

### 7. Render `./styleguide.html` (visual preview)
Write a single-file, self-contained HTML page that renders the extracted tokens visually — the companion artifact to `design.md`. Use the `Template: styleguide.html` section below as the scaffold.

Rules for the HTML:
- **Self-contained**: inline all CSS in a single `<style>` block. No external fonts unless the Figma file explicitly uses a Google Font — in that case add the matching `<link>` tag.
- **Use the extracted tokens as CSS custom properties** declared on `:root`. Every color swatch, font sample, button, and badge in the page should resolve from those variables, so the file doubles as a live token reference.
- **Same rule as `design.md`: never invent values.** If you didn't find badge styles in the Figma, omit the Badges section entirely rather than faking one. Same for shadows, border radii, etc.
- Section headings and layout stay neutral (system font, simple grid) — the page is a chrome to display the design system, not itself a branded artifact.

## Output file: `design.md` Template

```markdown
# Styleguide — {{file name or user-provided title}}

Source: {{figma URL(s), one per line}}
Captured: {{YYYY-MM-DD}}
Screenshots: {{relative paths to screenshots, one per link}}

## 1. Foundations

**Body / default surface**
- Background: `{{hex}}`
- Text color: `{{hex}}`
- Font family: `{{family}}`
- Base font size: `{{size}}`

## 2. Typography

| Element | Font family | Size | Weight | Line-height | Letter-spacing | Transform | Color |
|---------|-------------|------|--------|-------------|----------------|-----------|-------|
| H1 | ... | ... | ... | ... | ... | ... | ... |
| H2 | ... |
| H3 | ... |
| H4 | ... |
| H5 | ... |
| H6 | ... |
| Body | ... |
| Small | ... |
| Caption / overline | ... |
| Link | ... |

## 3. Color Palette

### Brand / Primary
- `#xxxxxx` — used on: buttons, CTAs
- ...

### Neutrals
- `#xxxxxx` — used on: body text
- ...

### Accents
- ...

### Borders
- ...

## 4. Buttons

For each distinct button style found:

### Primary
- Background: `...`
- Text color: `...`
- Border: `...`
- Border-radius: `...`
- Padding: `...`
- Font: `... / ... / ...` (family / size / weight)
- Letter-spacing: `...`
- Text-transform: `...`
- Box-shadow: `...`
- Example label: "..."

### Secondary
...

### Ghost / Tertiary
...

## 5. Links

### Default
- Color: `...`
- Text-decoration: `...`
- Font-weight: `...`

### Inline / Body links
...

## 6. Spacing & Layout

Observed from section/container/card nodes:
- Section padding (vertical): `...`
- Container max-width: `...`
- Card padding: `...`
- Card border-radius: `...`
- Card box-shadow: `...`
- Grid / column gap: `...`

## 7. Border Radius Scale

Unique radii observed, smallest → largest:
- `...`

## 8. Shadows

Unique shadows observed:
- `...`

## 9. Design Tokens (Figma Variables)

If the file defines variables via `get_variable_defs`, list them verbatim — grouped by collection if the file uses collections (e.g. color, spacing, typography). Otherwise: "No variables defined in this file."

```css
--token-name: value;
...
```

### Linked libraries
If `get_libraries` reports linked libraries, list them by name. Note any tokens that resolve through a library rather than locally.

## 10. Replication Notes

A short paragraph summarizing the overall aesthetic: minimal vs. dense, serif vs. sans, hard vs. soft corners, muted vs. saturated palette, etc. 3–5 sentences max. This is the "vibe" a developer needs to replicate the feel, not just the tokens.

## 11. Variant Overrides (if multiple links)

If multiple Figma URLs were merged, note anything that differs per node — e.g. the PDP uses a different primary button color from the homepage, or a section uses a tighter type scale. One bullet per delta.

## 12. Token conflicts (if any)

If two merged files defined the same variable with different values, list the conflict here so the team can reconcile upstream.
```

## Template: `styleguide.html`

Use this as the scaffold for the visual preview. Replace every `{{...}}` slot from the extracted data; drop entire `<section>` blocks whose data is empty.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Styleguide — {{file name}}</title>
<style>
  :root {
    /* Brand + neutrals (extracted) */
    {{--token-name: value; one per line, grouped by collection}}

    /* Layout chrome (neutral, fixed) */
    --chrome-bg: #fafafa;
    --chrome-fg: #111;
    --chrome-muted: #666;
    --chrome-border: #e5e5e5;
    --chrome-mono: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    --chrome-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
  }
  * { box-sizing: border-box; }
  html, body { margin: 0; background: var(--chrome-bg); color: var(--chrome-fg); font-family: var(--chrome-sans); }
  .wrap { max-width: 960px; margin: 0 auto; padding: 48px 24px 96px; }
  h1.doc-title { font-family: var(--chrome-sans); font-size: 28px; font-weight: 600; margin: 0 0 4px; }
  .doc-sub { color: var(--chrome-muted); font-size: 13px; margin-bottom: 48px; }
  section.block { margin-bottom: 56px; }
  section.block > h2 { font-family: var(--chrome-sans); font-size: 11px; font-weight: 600; letter-spacing: 0.1em; text-transform: uppercase; color: var(--chrome-muted); margin: 0 0 16px; padding-bottom: 8px; border-bottom: 1px solid var(--chrome-border); }
  .row { display: flex; flex-wrap: wrap; gap: 16px; }
  .grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr)); gap: 16px; }
  .mono { font-family: var(--chrome-mono); font-size: 12px; color: var(--chrome-muted); }

  /* Color swatches */
  .swatch { border: 1px solid var(--chrome-border); border-radius: 8px; overflow: hidden; background: #fff; }
  .swatch .chip { height: 88px; }
  .swatch .meta { padding: 10px 12px; display: flex; flex-direction: column; gap: 2px; }
  .swatch .hex { font-family: var(--chrome-mono); font-size: 12px; }
  .swatch .label { font-size: 12px; color: var(--chrome-muted); }

  /* Type samples */
  .type-row { display: grid; grid-template-columns: 80px 1fr; gap: 16px; align-items: baseline; padding: 12px 0; border-bottom: 1px dashed var(--chrome-border); }
  .type-row:last-child { border-bottom: 0; }
  .type-row .tag { font-family: var(--chrome-mono); font-size: 11px; color: var(--chrome-muted); text-transform: uppercase; }

  /* Token table */
  table.tokens { width: 100%; border-collapse: collapse; }
  table.tokens td { padding: 8px 12px; border-bottom: 1px solid var(--chrome-border); font-family: var(--chrome-mono); font-size: 12px; vertical-align: top; }
  table.tokens td:first-child { color: var(--chrome-muted); width: 40%; }

  /* Spacing + radius + shadow visual samples */
  .space-bar { background: var(--chrome-fg); height: 12px; border-radius: 2px; }
  .space-item, .radius-item, .shadow-item { background: #fff; border: 1px solid var(--chrome-border); border-radius: 8px; padding: 16px; display: flex; flex-direction: column; gap: 10px; }
  .radius-demo { width: 64px; height: 64px; background: var(--chrome-fg); }
  .shadow-demo { width: 100%; height: 64px; background: #fff; }
</style>
</head>
<body>
<div class="wrap">
  <h1 class="doc-title">Styleguide — {{file name or title}}</h1>
  <div class="doc-sub">Source: {{figma URL(s)}} · Captured {{YYYY-MM-DD}}</div>

  <!-- COLORS -->
  <section class="block">
    <h2>Colors — Brand / Primary</h2>
    <div class="grid">
      {{for each brand color}}
      <div class="swatch">
        <div class="chip" style="background: {{hex}};"></div>
        <div class="meta">
          <span class="hex">{{hex}}</span>
          <span class="label">{{usage, e.g. "Primary button bg"}}</span>
        </div>
      </div>
      {{/for}}
    </div>
  </section>

  <section class="block">
    <h2>Colors — Neutrals</h2>
    <div class="grid">{{same swatch pattern}}</div>
  </section>

  <section class="block">
    <h2>Colors — Accents</h2>
    <div class="grid">{{same swatch pattern}}</div>
  </section>

  <!-- TYPOGRAPHY -->
  <section class="block">
    <h2>Typography</h2>
    {{for each type style: H1..H6, Body, Small, Caption, Link — only those actually extracted}}
    <div class="type-row">
      <div class="tag">{{H1}}</div>
      <div style="font-family: {{family}}; font-size: {{size}}; font-weight: {{weight}}; line-height: {{line-height}}; letter-spacing: {{letter-spacing}}; color: {{color}};">
        {{Sample text, e.g. "The quick brown fox"}}
        <div class="mono" style="margin-top: 4px;">{{family}} · {{size}} / {{weight}} · lh {{line-height}}</div>
      </div>
    </div>
    {{/for}}
  </section>

  <!-- BUTTONS -->
  <section class="block">
    <h2>Buttons</h2>
    <div class="row">
      {{for each distinct button style}}
      <button style="
        background: {{bg}};
        color: {{text}};
        border: {{border}};
        border-radius: {{radius}};
        padding: {{padding}};
        font-family: {{family}};
        font-size: {{size}};
        font-weight: {{weight}};
        letter-spacing: {{tracking}};
        text-transform: {{transform}};
        box-shadow: {{shadow}};
        cursor: pointer;
      ">{{sample label, e.g. "Add to cart"}}</button>
      {{/for}}
    </div>
  </section>

  <!-- BADGES (omit entire section if no badge styles found in Figma) -->
  <section class="block">
    <h2>Badges</h2>
    <div class="row">
      {{for each badge style found}}
      <span style="
        display: inline-flex;
        align-items: center;
        background: {{bg}};
        color: {{text}};
        border: {{border}};
        border-radius: {{radius}};
        padding: {{padding}};
        font-family: {{family}};
        font-size: {{size}};
        font-weight: {{weight}};
        letter-spacing: {{tracking}};
        text-transform: {{transform}};
      ">{{sample label, e.g. "New"}}</span>
      {{/for}}
    </div>
  </section>

  <!-- LINKS -->
  <section class="block">
    <h2>Links</h2>
    <p style="font-family: {{body-family}}; font-size: {{body-size}}; color: {{body-color}};">
      This is body copy with an
      <a href="#" style="color: {{link-color}}; text-decoration: {{link-decoration}}; font-weight: {{link-weight}};">inline link</a>
      for reference.
    </p>
  </section>

  <!-- SPACING -->
  <section class="block">
    <h2>Spacing scale</h2>
    <div class="grid">
      {{for each unique spacing value, smallest → largest}}
      <div class="space-item">
        <div class="space-bar" style="width: {{value}};"></div>
        <span class="mono">{{value}}</span>
      </div>
      {{/for}}
    </div>
  </section>

  <!-- RADII -->
  <section class="block">
    <h2>Border radius</h2>
    <div class="grid">
      {{for each unique radius, smallest → largest}}
      <div class="radius-item">
        <div class="radius-demo" style="border-radius: {{value}};"></div>
        <span class="mono">{{value}}</span>
      </div>
      {{/for}}
    </div>
  </section>

  <!-- SHADOWS -->
  <section class="block">
    <h2>Shadows</h2>
    <div class="grid">
      {{for each unique shadow}}
      <div class="shadow-item">
        <div class="shadow-demo" style="box-shadow: {{value}};"></div>
        <span class="mono">{{value}}</span>
      </div>
      {{/for}}
    </div>
  </section>

  <!-- TOKENS (from get_variable_defs) -->
  <section class="block">
    <h2>Design tokens (Figma variables)</h2>
    <table class="tokens">
      {{for each variable}}
      <tr><td>{{--token-name}}</td><td>{{value}}</td></tr>
      {{/for}}
    </table>
  </section>
</div>
</body>
</html>
```

## Rules

- **Never invent values.** Every number/color in the output must come from Figma MCP output (variables, design context, or screenshot). If a section has no data, omit it.
- **Variables first.** When a token is defined as a Figma variable, cite the variable name and value — not the raw hex — in section 9. Raw hex goes in sections 3–8.
- **Deduplicate.** The same color in different casings or alpha notations counts once. Normalize hex to lowercase.
- **Prefer hex for colors** in sections 3–8. Note alpha separately when present.
- **Use only relative paths** for file writes (e.g. `./design.md`, `./screenshots/homepage.png`). Never write to absolute paths.
- **Respect file access.** If a Figma URL requires auth the MCP server doesn't have, stop and tell the user — do not attempt WebFetch as a workaround.
- **One styleguide per invocation.** Combine the provided links into one `./design.md`; don't crawl out to other Figma files referenced inside the design.
- **Don't exfiltrate copy.** This is a styleguide, not a content dump — sample labels on buttons/links stay short (≤40 chars) and serve only to name the style.

## Output to user

After writing both files:
1. Report both file paths: `./design.md` (token reference) and `./styleguide.html` (visual preview). Tell the user they can open the HTML in a browser to see typography, swatches, buttons, and badges rendered from the real tokens.
2. Give a 2–3 bullet summary: primary font, primary brand color, button style (pill/square/ghost).
3. List the screenshot paths captured, one per Figma URL processed.
