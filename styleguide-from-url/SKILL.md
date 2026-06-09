---
name: styleguide-from-url
description: "Given a URL, open it in a browser via chrome-devtools MCP, extract typography, colors, buttons, links, spacing, and other design tokens, then write a replicatable design.md styleguide. Trigger when the user provides a URL and asks for a styleguide, style extraction, design tokens, to reverse-engineer a page's style, or to 'make a design.md' / 'extract the style' / 'build a styleguide from this URL'."
compatibility: Claude Code
metadata:
  author: Kasper Dolk
  version: "1.0"
---

# Styleguide From URL

Turn any public URL into a structured, replicatable `design.md` styleguide by inspecting the live page with chrome-devtools MCP.

## Input

The user supplies a URL. Optionally, they may specify:
- Output path for the styleguide (default: `./design.md` in the current working directory).
- A viewport width (default: `1440x900` desktop; also capture `390x844` mobile if time permits).
- A focus list (e.g. "just typography and colors").

If the user provides multiple URLs, process them one at a time and write one `design.md` per URL (suffix the filename: `style-<slug>.md`).

## Required MCP

This skill uses the `chrome-devtools` MCP server. The tools used are:
- `mcp__chrome-devtools__new_page` (or `navigate_page` if a page already exists)
- `mcp__chrome-devtools__resize_page`
- `mcp__chrome-devtools__evaluate_script`
- `mcp__chrome-devtools__take_screenshot`
- `mcp__chrome-devtools__list_pages` / `select_page`
- `mcp__chrome-devtools__list_network_requests` (for font discovery)
- `mcp__chrome-devtools__close_page` (at the end)

It also shells out to `curl` via the Bash tool to download font files locally.

If `chrome-devtools` MCP is unavailable, stop and tell the user to install/enable it — do NOT guess styles.

## Process

### 1. Open the page
1. Call `list_pages`. If no active page, call `new_page` with the URL. Otherwise `navigate_page` to the URL.
2. `resize_page` to `1440x900`.
3. Wait for load — use `wait_for` on `body` if needed.
4. `take_screenshot` (full page, saved as reference — note the path).

### 2. Extract design tokens via `evaluate_script`
Run a single large script that returns a JSON object with everything we need. The script below is the canonical extractor — run it verbatim via `evaluate_script`.

```js
() => {
  const uniq = (arr) => [...new Set(arr)];
  const rgbToHex = (rgb) => {
    if (!rgb || rgb === 'rgba(0, 0, 0, 0)' || rgb === 'transparent') return null;
    const m = rgb.match(/\d+(\.\d+)?/g);
    if (!m) return null;
    const [r,g,b,a] = m.map(Number);
    const hex = '#' + [r,g,b].map(n => n.toString(16).padStart(2,'0')).join('');
    return a !== undefined && a < 1 ? `${hex} (alpha ${a})` : hex;
  };
  const pick = (el, props) => {
    const s = getComputedStyle(el);
    const out = {};
    for (const p of props) out[p] = s.getPropertyValue(p).trim();
    return out;
  };
  const typoProps = ['font-family','font-size','font-weight','line-height','letter-spacing','text-transform','color'];

  // Typography — one sample per heading level + body
  const typography = {};
  for (const tag of ['h1','h2','h3','h4','h5','h6','p','body','a','small','blockquote']) {
    const el = document.querySelector(tag);
    if (el) typography[tag] = pick(el, typoProps);
  }

  // Buttons — sample up to 6 distinct button styles
  const btnSel = 'button, a.button, a.btn, .btn, [role="button"], input[type="submit"], input[type="button"]';
  const buttons = [];
  const btnSeen = new Set();
  document.querySelectorAll(btnSel).forEach(el => {
    const s = getComputedStyle(el);
    const key = [s.backgroundColor, s.color, s.border, s.borderRadius, s.padding, s.fontSize, s.fontWeight, s.textTransform].join('|');
    if (btnSeen.has(key) || buttons.length >= 6) return;
    btnSeen.add(key);
    buttons.push({
      sample_text: (el.innerText || el.value || '').trim().slice(0, 40),
      tag: el.tagName.toLowerCase(),
      class: el.className || null,
      ...pick(el, ['background-color','color','border','border-radius','padding','font-family','font-size','font-weight','letter-spacing','text-transform','box-shadow','transition'])
    });
  });

  // Links — default + attempt :visited/:hover via inline style comparison
  const links = [];
  const linkSeen = new Set();
  document.querySelectorAll('a').forEach(el => {
    const s = getComputedStyle(el);
    const key = [s.color, s.textDecoration, s.fontWeight].join('|');
    if (linkSeen.has(key) || links.length >= 4) return;
    linkSeen.add(key);
    links.push({
      sample_text: (el.innerText || '').trim().slice(0, 40),
      ...pick(el, ['color','text-decoration','text-decoration-color','font-weight','text-underline-offset'])
    });
  });

  // Color palette — scan all elements, collect unique bg/text/border colors
  const colors = { background: new Set(), text: new Set(), border: new Set() };
  const all = document.querySelectorAll('*');
  const CAP = 5000;
  for (let i = 0; i < Math.min(all.length, CAP); i++) {
    const s = getComputedStyle(all[i]);
    const bg = rgbToHex(s.backgroundColor);
    const fg = rgbToHex(s.color);
    const bd = rgbToHex(s.borderColor);
    if (bg) colors.background.add(bg);
    if (fg) colors.text.add(fg);
    if (bd) colors.border.add(bd);
  }

  // CSS custom properties on :root (design tokens if the site uses them)
  const rootStyle = getComputedStyle(document.documentElement);
  const cssVars = {};
  for (let i = 0; i < rootStyle.length; i++) {
    const name = rootStyle[i];
    if (name.startsWith('--')) cssVars[name] = rootStyle.getPropertyValue(name).trim();
  }

  // Spacing / radii / shadows — sample from sections, cards, containers
  const containerSel = 'section, .container, .card, .wrapper, main, article, header, footer';
  const containers = [];
  const contSeen = new Set();
  document.querySelectorAll(containerSel).forEach(el => {
    const s = getComputedStyle(el);
    const key = [s.padding, s.margin, s.borderRadius, s.boxShadow, s.maxWidth].join('|');
    if (contSeen.has(key) || containers.length >= 8) return;
    contSeen.add(key);
    containers.push({
      tag: el.tagName.toLowerCase(),
      class: (el.className || '').toString().slice(0, 80),
      ...pick(el, ['padding','margin','border-radius','box-shadow','max-width','background-color','border'])
    });
  });

  // Page-level info
  const body = getComputedStyle(document.body);
  const page = {
    title: document.title,
    url: location.href,
    viewport: `${window.innerWidth}x${window.innerHeight}`,
    body_background: rgbToHex(body.backgroundColor),
    body_font: body.fontFamily,
    body_font_size: body.fontSize,
    body_color: rgbToHex(body.color),
  };

  return {
    page,
    typography,
    buttons,
    links,
    colors: {
      background: [...colors.background],
      text: [...colors.text],
      border: [...colors.border],
    },
    cssVars,
    containers,
  };
}
```

### 3. Discover and download fonts

The page just loaded the actual font files in its network tab. Capture them and save local copies so the styleguide is self-contained.

**3a. List font network requests**

Call `mcp__chrome-devtools__list_network_requests` with `resourceTypes: ["font"]`. This returns every `.woff2` / `.woff` / `.ttf` / `.otf` file the page actually fetched. If the list is empty, the page uses only system fonts — skip the rest of this step and note that in `design.md`.

**3b. Map font URLs to family + weight + style via `evaluate_script`**

The network request alone doesn't tell you which family/weight a file is for. Run this script to walk every stylesheet (cross-origin sheets included via inline `<style>` scan) and extract `@font-face` declarations:

```js
() => {
  const faces = [];
  const seen = new Set();

  // Same-origin stylesheet rules
  for (const sheet of document.styleSheets) {
    try {
      const rules = sheet.cssRules || sheet.rules;
      if (!rules) continue;
      for (const rule of rules) {
        if (rule.constructor && rule.constructor.name === 'CSSFontFaceRule') {
          const t = rule.cssText;
          if (!seen.has(t)) { seen.add(t); faces.push(t); }
        }
      }
    } catch (e) { /* cross-origin */ }
  }

  // Inline <style> tags (catch Framer / Next / etc. injected blocks)
  for (const styleEl of document.querySelectorAll('style')) {
    const matches = (styleEl.textContent || '').match(/@font-face\s*\{[^}]+\}/g);
    if (matches) for (const m of matches) {
      if (!seen.has(m)) { seen.add(m); faces.push(m); }
    }
  }

  // Parse each face into structured data
  const parsed = faces.map(t => {
    const family = (t.match(/font-family:\s*["']?([^;"']+)["']?/i) || [])[1]?.trim();
    const weight = (t.match(/font-weight:\s*([^;]+)/i) || [])[1]?.trim() || '400';
    const style = (t.match(/font-style:\s*([^;]+)/i) || [])[1]?.trim() || 'normal';
    // Pull every url(...) from the src — there can be more than one (woff2, woff, ttf)
    const urls = [...t.matchAll(/url\(["']?([^"')]+)["']?\)/g)].map(m => m[1]);
    return { family, weight, style, urls, raw: t };
  }).filter(f => f.family && f.urls.length > 0);

  return parsed;
}
```

**3c. Decide which faces to download**

You now have two lists:
- **Network fonts** (URLs the page actually fetched — these are the real ones in use)
- **Declared @font-face faces** (everything the CSS knows about — may include weights that weren't loaded yet)

Intersect them: keep `@font-face` entries whose first `url(...)` matches a URL from the network list. That's your download set. Skip "Placeholder" pseudo-families (Framer's `local()` fallbacks like `"Alliance No.1 SemiBold Placeholder"` — they have `src: local("Arial")` and no remote URL, so they'll filter out automatically).

If the network list contains fonts that don't appear in any `@font-face` (rare — usually JS-injected), download them with a generic name (`font-<hash>.woff2`) and note the family is unknown.

**3d. Create the `downloaded-fonts/` folder and download**

The output folder is **`downloaded-fonts/`** in the same directory as `design.md`. Create it if it doesn't exist (`mkdir -p downloaded-fonts`).

Pick a sensible filename for each font. Use the slug pattern: `<family-slug>-<weight>[-italic].woff2`.
- "Alliance No.1" / 600 / normal → `alliance-no1-600.woff2`
- "Familjen Grotesk" / 400 / italic → `familjen-grotesk-400-italic.woff2`
- Lowercase the family, replace spaces and dots with `-`, strip quotes.

Download each unique URL via `curl -sS -o <path> "<url>"`. Batch them in a single Bash command with `&&` chaining (or run in parallel with `&` and `wait` if there are many) so it finishes fast.

If any download fails (4xx/5xx), note the URL in the output and continue — do not abort the whole skill over one missing font.

**3e. Hold the font manifest for the template**

Keep an in-memory list like:

```
[
  { family: "Alliance No.1", weight: 600, style: "normal", file: "downloaded-fonts/alliance-no1-600.woff2", source_url: "https://..." },
  ...
]
```

You'll write this into `design.md` in §12 (Local Fonts) and emit a ready-to-paste `@font-face` block.

### 4. (Optional) Mobile pass
If the user asked for responsive tokens, `resize_page` to `390x844` and re-run the token-extraction script (step 2) — merge mobile-specific typography into the report under a "Mobile Overrides" section. The font set from step 3 doesn't need to be re-run; mobile loads the same files.

### 5. Write `design.md`

Use the **Template** below. Fill every section from the extracted JSON. Omit sections that are empty rather than leaving `TBD`. Round pixel values as-is (don't convert to rem unless the source uses rem).

For colors, deduplicate visually-identical values and group into:
- **Brand / Primary** — colors that appear on buttons, links, CTAs
- **Neutrals** — grays, blacks, whites, and near-whites used for text and surfaces
- **Accents** — any remaining unique colors

Make that grouping judgment from the extracted data (buttons/links drive "brand"; body/text colors drive "neutrals"). Don't invent colors that aren't in the extracted set.

### 6. Clean up
`close_page` the tab you opened (only if you opened it — don't close a pre-existing tab the user was using).

## Output file: `design.md` Template

```markdown
# Styleguide — {{page.title}}

Source: {{page.url}}
Captured: {{YYYY-MM-DD}} at {{viewport}}
Screenshot: {{relative path to screenshot}}

## 1. Foundations

**Body**
- Background: `{{body_background}}`
- Text color: `{{body_color}}`
- Font family: `{{body_font}}`
- Base font size: `{{body_font_size}}`

## 2. Typography

| Element | Font family | Size | Weight | Line-height | Letter-spacing | Transform | Color |
|---------|-------------|------|--------|-------------|----------------|-----------|-------|
| H1 | ... | ... | ... | ... | ... | ... | ... |
| H2 | ... |
| H3 | ... |
| H4 | ... |
| H5 | ... |
| H6 | ... |
| Body (p) | ... |
| Small | ... |
| Blockquote | ... |

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
- Transition: `...`
- Example label: "..."

### Secondary
...

## 5. Links

### Default
- Color: `...`
- Text-decoration: `...`
- Font-weight: `...`
- Underline offset: `...`

### Inline / Body links
...

## 6. Spacing & Layout

Observed from section/container/card elements:
- Section padding (vertical): `...`
- Container max-width: `...`
- Card padding: `...`
- Card border-radius: `...`
- Card box-shadow: `...`

## 7. Border Radius Scale

Unique radii observed, smallest → largest:
- `...`

## 8. Shadows

Unique shadows observed:
- `...`

## 9. CSS Custom Properties (Design Tokens)

If the source defines `:root` variables, list them verbatim. Otherwise: "Not exposed as CSS custom properties."

```css
--token-name: value;
...
```

## 10. Replication Notes

A short paragraph summarizing the overall aesthetic: minimal vs. dense, serif vs. sans, hard vs. soft corners, muted vs. saturated palette, etc. 3-5 sentences max. This is the "vibe" a developer needs to replicate the feel, not just the tokens.

## 11. Mobile Overrides (if captured)

Anything that changes at 390px wide — typography scale, stacked layouts, button sizing.

## 12. Local Fonts

Downloaded into `./downloaded-fonts/` for offline / self-hosted use.

| Family | Weight | Style | File | Source URL |
|--------|--------|-------|------|------------|
| ... | ... | ... | `downloaded-fonts/...` | `https://...` |

Drop-in `@font-face` block:

```css
@font-face {
  font-family: "{{family}}";
  src: url("./downloaded-fonts/{{file}}") format("woff2");
  font-weight: {{weight}};
  font-style: {{style}};
  font-display: swap;
}
... (one block per file)
```

If no web fonts were detected (system fonts only), state that here and skip the table.
```

## Rules

- **Never invent values.** Every number/color in the output must come from the extracted JSON. If a section has no data, omit it.
- **Deduplicate.** The same color in rgb/hex/rgba variants counts once.
- **Prefer hex for colors** in the final markdown. Note alpha separately when present.
- **Don't exfiltrate copy.** This is a styleguide, not a content dump — keep `sample_text` snippets short (≤40 chars) and use them only as button/link labels.
- **Respect robots.txt / auth walls.** If the page requires login or blocks automation, stop and report it — do not try to bypass.
- **One page at a time.** Don't crawl links.

## Output to user

After writing `design.md`:
1. Report the file path.
2. Give a 2-3 bullet summary: primary font, primary brand color, button style (pill/square/ghost).
3. Note the screenshot path for visual reference.
4. Report the count of fonts saved into `./downloaded-fonts/` (e.g. "8 font files saved"), or that none were detected if the page used only system fonts.
