# Brad Coles — Personal Brand Kit (v1)

Internal reference for video, presentation, and demo assets. Extends the visual identity already established on bradcoles.dev — not a public site page.

## 1. Brand Essence

- **Positioning:** Senior Data Engineering Consultant specialising in Microsoft Fabric & Azure — pragmatic, hands-on, results-driven.
- **Voice:** Technical and competent, clear and concise, professional but approachable, focused on business impact, data-driven.
- **On video:** Same voice as the blog — lead with the business/operational outcome, explain *why* before *how*, keep language plain even when the topic is deep.

## 2. Logo / Mark

- **Primary mark:** stacked `bc` / `dev_` lockup in JetBrains Mono Bold on deep navy — `bc` in light text, `dev_` in accent blue with the trailing underscore styled as a blinking terminal cursor (SVG `<animate>`, degrades gracefully to a static underscore in raster exports).
  - File: `brand/logo-bc-mark.svg` (square, 512×512 — suitable as channel avatar / favicon / watermark)
- **Wordmark (secondary):** `bradcoles.dev` set in JetBrains Mono — for end screens / lower-thirds.
- **Clear space:** pad at least 0.5× the mark's height on all sides.
- **Don't:** place on busy photo backgrounds, recolour outside the palette, or stretch/distort.

## 3. Colour Palette

| Name | Hex | Use |
|---|---|---|
| Background (deep navy) | `#0a0e1a` | Slide & video backgrounds |
| Card / panel | `#111827` | Lower-thirds, code panels, callout boxes |
| Text primary | `#e5e7eb` | Body text on dark |
| Text secondary | `#9ca3af` | Captions, muted labels |
| Accent | `#3b82f6` | Highlights, underlines, buttons |
| Accent bright | `#60a5fa` | Hover/glow, emphasis text |
| Accent dim | `#1e40af` | Borders, secondary fills |
| Success | `#10b981` | "Before/after" wins, positive metrics |
| Border | `#1f2937` | Dividers, grid lines |
| Code background | `#0d1117` | Terminal/code demo windows |

## 4. Typography

| Role | Font | Use | Fallback (tools without the font) |
|---|---|---|---|
| Display | Press Start 2P | Big intro/title card only — use sparingly | Any bold pixel/monospace |
| Headings | Syne (Bold/ExtraBold) | Slide titles, lower-third name, section headers | Poppins / Montserrat Bold |
| Body & code | JetBrains Mono | Body text, captions, terminal/code demos | Cascadia Code / Consolas |

## 5. Imagery & Motifs

- **Headshot:** `images/Head_shot-removebg-preview.png` (transparent background) for talking-head overlays / "about" cards.
- **Background motif:** subtle blue grid (as on the site header) for intros/title cards — keep opacity low (~5–8%) so it doesn't compete with foreground content.
- **Code/terminal framing:** present demos in a dark terminal-style window (`#0d1117` background, `#1f2937` border, JetBrains Mono) to match the site's existing code blocks.
- **Icon style:** simple line icons / text tags consistent with the site's skill tags — avoid stock photography.

## 6. Video & Slide Specs

- **Slide deck:** 16:9, 1920×1080. Background `#0a0e1a`; headings in Syne; body in JetBrains Mono.
- **YouTube thumbnail:** 1280×720. High-contrast accent-blue text on navy, headshot + 3–5 word hook.
- **Lower-third:** `#111827` panel at ~80% opacity; name in Syne Bold (`#e5e7eb`); role/title in JetBrains Mono (`#9ca3af`); thin accent-blue (`#3b82f6`) top border.
- **Safe area:** keep key text within the centre 90% of frame (10% margin) to avoid platform UI overlap.

## 7. File Naming & Location

- `brand/` — kit source files (this guide, logo, templates)
- `brand/logo-bc-mark.svg` — primary mark (v1 draft)
- `brand/title-card-template.svg` — 1920×1080 video intro/section title slide
- `brand/youtube-thumbnail-template.svg` — 1280×720 YouTube thumbnail
- Suggested additions as the kit grows: `brand/lower-third-template.svg`

**Editing the templates:** open the `.svg` files directly in a browser to preview, or import into Figma/Inkscape for layout tweaks. Text, colours, and the code-snippet content can be edited directly in the SVG markup. Headshot is referenced from `../images/Head_shot-removebg-preview.png`.

---
*v1 — drafted for the Fabric Delta table maintenance YouTube appearance. Iterate as needed.*
