# DemandLink Website Build — README

## Project Overview

This repository is the source of truth for building the DemandLink website. The site is hosted on GoHighLevel (GHL) as a landing page funnel to enable A/B testing. Code is written collaboratively with Claude Code and pasted into GHL's builder, block by block.

The initial build is a single funnel with four pages: Home, Thank You, Privacy Policy, and Terms of Service. Later phases will expand into a full multi-page site.

Brand personality, color tokens, typography, and tone guidance live in a separate brand guide file. This README focuses on **how the build is structured and how code is organized and delivered** — not visual identity.

---

## Platform

**GoHighLevel (GHL) Funnel/Landing Page builder** (not the Website builder — A/B testing requires the Funnel builder). All custom code is pasted into GHL's builder elements and runs inside GHL's page rendering environment.

---

## Page Structure

Four pages, each a step in the same GHL funnel:

1. **Home** — main landing page with hero, content sections, form popup
2. **Thank You** — shown after form submission
3. **Privacy Policy**
4. **Terms of Service**

---

## Code Block Architecture

This is the most important concept to understand before writing any code for this project. Read this section carefully.

### One page = multiple blocks

Every page is built out of **separate Custom HTML/Javascript blocks**, not a single monolithic block. At minimum, every page has:

- A **Header** block
- One or more **Content** blocks (sections of the page body)
- A **Footer** block

These are independent builder elements stacked as rows on the page. They do **not** share a `<style>` scope. CSS class names must be prefixed per section to prevent collisions across blocks.

### Shared elements across pages

| Element | Used on | How it's shared |
|---|---|---|
| **Global Footer** | All 4 pages | Set as a GHL Global Footer — one source of truth. Edit once, updates everywhere. |
| **Main Header** | Home + Thank You | Full header with nav and CTA. |
| **Legal Header** | Privacy Policy + Terms of Service | Stripped-down header shared by the two legal pages. **Distinct** from the main header. |
| **Shared Custom CSS + JS** | All 4 pages | **Manually synced** — not a GHL global section. Source of truth lives in the repo; pasted into every page's Custom CSS panel when edited. See *Shared Custom CSS + JS* below. |

Note the distinction: the Global Footer is shared by GHL automatically (edit once, propagates). The shared Custom CSS + JS is shared **by convention only** — the user must paste it into all four pages manually after any edit.

When Claude Code is asked to modify a header, it must first confirm **which header** is being edited (Main vs. Legal) — they live in separate blocks and changes to one do not affect the other.

### Home page — one extra block

The Home page has one block that no other page has: a **Hidden Popup Button** block. See the *Hidden Popup Button Pattern* section below for what it is and why it exists.

### Per-page block inventory (current state)

| Page | Blocks present |
|---|---|
| Home | Main Header · Content (1 or more) · Hidden Popup Button · Footer (global) |
| Thank You | Main Header · Content · Footer (global) |
| Privacy Policy | Legal Header · Content · Footer (global) |
| Terms of Service | Legal Header · Content · Footer (global) |

---

## Shared Custom CSS + JS

Every page in this funnel uses the **same** Custom CSS block. These are **manually synced** — not GHL global sections. When a change is made, it must be pasted into all four pages.

The canonical copy of this shared code lives in the repo (e.g. `/shared/page-custom-css.css`). Treat the repo file as the source of truth. Any edit must be (a) committed to the repo and (b) pasted into every page in GHL.

### What the shared code contains

- **Google Fonts import** — Figtree, Inter, JetBrains Mono
- **`:root` brand tokens** — all color and font variables referenced via `var(--token-name)` from every block
- **Base `html, body` background** — sets the page default to the dark brand color so there's no flash before blocks render
- **`window.dlOpenPopup()`** — a global helper that finds the hidden native popup trigger (by class `.dl-popup-trigger`) and clicks it. All custom-coded CTAs that need to open the form popup call this function rather than re-implementing the querySelector each time.

### Known issue to flag for Claude Code

The `window.dlOpenPopup()` function currently sits inside the **Custom CSS panel** alongside the CSS. This is likely wrong — CSS parsers don't execute JavaScript, so the function may not actually be defined at runtime from that location. It may also exist in a separate Custom HTML/Javascript block somewhere and be working from there instead. **This has not been confirmed.**

**First task for a new Claude Code session:** investigate where `window.dlOpenPopup` is actually being defined and called from in the live site. If the JS is only in the Custom CSS panel, it needs to be relocated to a proper JS injection point — either a dedicated Custom HTML/Javascript block with a `<script>` tag, or GHL's Footer Tracking Code under Settings → Tracking Code. Once the correct location is confirmed, update this README to document it.

### What this means for Claude Code

- Do **not** redefine `:root` tokens, re-import Google Fonts, or redeclare `window.dlOpenPopup` inside any section block. They are already available globally (pending the issue above).
- When editing brand tokens or the popup helper, provide the updated code as a replacement for the shared file, and instruct the user to paste it into all four pages and commit the updated file to the repo.
- When writing a custom CTA that opens the form popup, use `window.dlOpenPopup()` — do not re-implement the querySelector-and-click logic inline.

---

## Forms & Popups

### Native everything

All lead capture uses GHL **native forms** hosted inside GHL **native popups** — not coded forms. This keeps CRM integration, submission history, and workflow triggers working without webhooks.

### Where styling lives

This is the detail that trips people up most — form CSS and popup CSS live in different places.

| Element | Where its CSS lives | Why |
|---|---|---|
| **Form** (fields, labels, inputs, submit button, form layout) | **Form-level Custom CSS** — Form Builder → Advanced tab → Custom CSS | The form is a separate GHL entity with its own dedicated CSS field. |
| **Popup** (popup container, overlay, close button, outer spacing) | **Page-level Custom CSS** (i.e. the shared Custom CSS), scoped to the popup's unique CSS selector (e.g. `#hl_main_popup-xa1hhtz9Or`) | The popup has no dedicated Custom CSS field. It is styled from outside using its auto-generated selector or a custom class assigned in the Styles panel. |

When Claude Code needs to style the popup, it must be given the popup's CSS selector by the user. When styling the form, the code goes into the form's own Custom CSS field — a `<style>` tag inside a section block cannot reach the form.

---

## Hidden Popup Button Pattern (Home page only)

GHL's native popups can only be opened reliably by **GHL native buttons**. Custom-coded buttons inside a Custom HTML/Javascript block cannot directly trigger a popup through the GHL API. The project uses this workaround:

1. A **GHL native button** is placed on the page and configured (via GHL's native action settings) to open the form popup on click. This is the "real" trigger.
2. That native button has the CSS class **`dl-popup-trigger`** applied via the Styles panel's Custom Class field.
3. The native button is **hidden from view** via CSS and lives inside its own block labeled **Hidden Popup Button**.
4. The global helper **`window.dlOpenPopup()`** (defined in the shared global JS — see *Shared Custom CSS + JS* above) finds the element with class `.dl-popup-trigger` and calls `.click()` on it.
5. Custom-coded CTA buttons on the page simply call `window.dlOpenPopup()` on click.

When Claude Code is building or editing any CTA that needs to open the form popup:

- Use `window.dlOpenPopup()` — do not re-implement the querySelector-and-click logic inline
- Do **not** attempt to call any GHL popup API directly
- Do **not** attempt to open the popup through URL parameters or custom events

If the hidden native button's class, position, or popup configuration needs to change, that edit happens in the **Hidden Popup Button** block on the Home page — not in any CTA.

---

## Working With Claude Code

### The user is not a developer

The project owner pastes code into the right places when given clear instructions but cannot infer where code goes, debug silently broken behavior, or interpret ambiguous guidance. Every Claude Code response must be written with this in mind.

### Mandatory Placement Guide

For every piece of code Claude Code provides, it must include a Placement Guide covering:

1. **Exactly where this code goes.** Name the specific GHL builder location. Be precise about *which* block — headers, content, and footers are separate. Examples:
   - "Paste into the existing **Home Page Header** block (open its Code Editor and replace contents)."
   - "Paste into the page-level Custom CSS panel (Settings → Custom CSS) on the **Home** page. **Reminder:** this is the shared CSS — after saving, paste the same updated code into the Custom CSS panel of all four pages and update the shared file in the repo."
   - "Paste into the **form-level** Custom CSS field (Form Builder → Advanced tab → Custom CSS)."
   - "Paste into the **Hidden Popup Button** block on the Home page."
   - "Paste into the **Legal Header** block (shared by Privacy Policy and Terms of Service — confirm whether it is a true shared block or a duplicate before editing)."

2. **Step-by-step builder instructions** if the user needs to create or locate something first (add a row, drag a Custom HTML/Javascript element in, open the Code Editor, etc.).

3. **Selectors or IDs the user must provide first**, and how to find them in the Styles panel. The common ones:
   - Popup CSS selector (for popup styling or for configuring the hidden native button's action)
   - Hidden native popup button selector or class (for debugging, not for writing CTAs — CTAs should use `window.dlOpenPopup()`)
   - Form root selector (for form-scoped CSS overrides)

4. **What the user should see** after pasting, so they can verify it worked.

5. **What to do if it doesn't look right** — the most common failure mode and its fix.

### Code delivery format

When code spans multiple placement locations, label each piece and order them in paste order:

```
═══════════════════════════════════════════════════════
STEP 1 OF 3 — PASTE INTO: Shared Custom CSS (all 4 pages)
Location: Each page → Settings → Custom CSS
Purpose: Updates a brand token used across the site
═══════════════════════════════════════════════════════

[code]

What you should see: Nothing visible yet — foundation
for the steps that follow. Remember to paste this into
all four pages and update the repo file.

═══════════════════════════════════════════════════════
STEP 2 OF 3 — PASTE INTO: Home Page Content Block
Location: Existing Custom HTML/Javascript block on the
Home page, labeled "Home Page Content"
Purpose: Builds the hero section
═══════════════════════════════════════════════════════

[code]

What you should see: The hero with headline, subhead,
and two CTA buttons.
```

### Why code is split across blocks

Do not combine everything into one block, even if asked. Reasons:

- **Shared tokens, font imports, and `window.dlOpenPopup`** live in the shared Custom CSS + JS — never redeclared inside a block
- **Form CSS** must live in form-level Custom CSS — a `<style>` tag in a section block cannot reach the separately-placed GHL native form
- **Popup CSS** must be scoped by the popup's unique selector from page-level CSS — popups have no dedicated CSS field
- **Header, Content, Footer, and Hidden Popup Button** are each their own block — their HTML and CSS must stay inside their own block so the block remains portable and doesn't depend on invisible neighbors
- **Section-specific styles** stay inside each section's block to avoid class-name collisions between sections

---

## Code Standards

### Language
Plain HTML, CSS, and vanilla JavaScript only. No React, no npm, no build tools, no frameworks. External resources allowed: Google Fonts, hosted image URLs, and standard CDN-hosted libraries loaded via `<script src="...">` when genuinely necessary.

### Class naming
Prefix every CSS class with the section name to prevent collisions between blocks. Examples: `.hero-container`, `.hero-headline`, `.features-card`, `.footer-column`, `.legal-header-logo`. The class `.dl-popup-trigger` is reserved for the single hidden native popup button — do not reuse it.

### Responsive design
Every block must be fully responsive. Breakpoints: mobile `< 768px`, tablet `768–1024px`, desktop `> 1024px`. Mobile-first where reasonable.

### !important overrides
GHL applies many default styles with `!important`. When styling GHL native forms, popups, or buttons via Custom CSS, overrides will often need `!important` to take effect. This is expected — use it where required, but don't spray it across custom-coded sections that have no GHL defaults fighting them.

### CSS custom properties
Brand tokens (colors, fonts, spacing) are defined as CSS variables in the **shared Custom CSS** that is pasted into every page's Custom CSS panel. All blocks reference them via `var(--token-name)`. See *Shared Custom CSS + JS* for the source-of-truth file and the sync rule. Do not redeclare tokens inside individual blocks.

---

## GHL-Specific Notes

### Code entry points

| Entry point | Where | Use for |
|---|---|---|
| Section block | Custom HTML/Javascript element in a builder row | Individual blocks (Main Header, Legal Header, Content sections, Hidden Popup Button). Footer is shared globally. |
| Page-level Custom CSS | Page → Settings → Custom CSS | The shared CSS (tokens, font imports, popup styling, GHL native element overrides) — pasted identically into all 4 pages |
| Form-level Custom CSS | Form Builder → Advanced tab → Custom CSS | Form field/button/layout styling only |
| HTML inside form | Form Builder → HTML element | Custom-styled headings or text blocks within the form body |
| Footer Tracking Code (likely correct home for `window.dlOpenPopup`) | Settings → Tracking Code | Global JS — to be confirmed (see *Shared Custom CSS + JS* known issue) |

### Container spacing is zeroed out

Every Custom HTML/Javascript block and its wrapping row, column, and section containers have **margin and padding set to 0 on both mobile and desktop** in the GHL builder. This is deliberate — all spacing is controlled from within the block's own code so sections are self-contained and portable.

What this means for Claude Code:

- Every block must define its own outer padding (top/bottom for vertical rhythm, left/right for gutters) inside its own CSS. Do not assume a parent container is supplying any breathing room.
- Full-bleed backgrounds work by default — the block's root element can span edge-to-edge without fighting container padding.
- If a section looks cramped against the viewport edges, the fix is inside that block's CSS, not in the GHL builder's spacing controls.

### Auto-generated selectors

Custom code blocks, popups, and forms receive unique auto-generated IDs visible in the Styles panel. Examples:

- Custom code block: `#custom-code-wY2Jjh9fMU`
- Popup: `#hl_main_popup-xa1hhtz9Or`
- Form root: `#_builder-form`

Do not guess these. When JavaScript or CSS needs to target another block, the popup, or the form, **request the selector from the user** and place a clear `REPLACE_WITH_YOUR_SELECTOR` token in the code with instructions on how to find it.

### JavaScript timing

- Scripts that query the DOM **inside their own block**: wrap in `document.addEventListener('DOMContentLoaded', ...)` or place the `<script>` at the end of the block.
- Scripts that reference elements **in other blocks** (e.g. a hero CTA calling `window.dlOpenPopup()`): the global helper handles its own querying; CTAs can call it directly in an `onclick` handler. For other cross-block references, wrap in `window.addEventListener('load', ...)` to ensure the full page has rendered.

### Images
All images hosted in GHL's media library or an external host, referenced by URL. For missing assets, use `<!-- REPLACE: [asset-name] URL -->`. Inline SVG preferred for icons.

### Visibility controls
GHL's Styles panel provides native show/hide toggles per device (desktop / tablet / mobile). Use these for block-level visibility instead of CSS `display: none`. Use CSS media queries for element-level responsive behavior **within** a block.

---

## What the User Provides per Section

For every section to be built or edited, the user will provide:

1. **Which page and which block** the work belongs to (Main Header, Legal Header, a specific Content block, Hidden Popup Button, Global Footer, or a new block to be created)
2. Section purpose and layout description
3. Exact copy (headlines, subheads, body text, button labels)
4. Image references and hosted URLs
5. Any relevant auto-generated GHL selectors:
   - Popup selector (if styling the popup or configuring a popup trigger)
   - Form selector (if the work involves form styling or targeting)
6. Reference to the brand guide (separate file) for colors, fonts, and tone

---

## Starting a Claude Code Session

At the start of every Claude Code session, provide:

1. **This README**
2. **The shared Custom CSS file** from the repo
3. **The brand guide file** (tone and any design context not already captured in the shared CSS tokens)
4. **The current code from every block** on the page being worked on (Header + Content + Hidden Popup Button if Home + form-level Custom CSS if form work is involved)
5. **The section brief** for the specific piece being built or edited

Claude Code should acknowledge the Placement Guide requirement and confirm **which block** it is editing before producing any code.

A good first task for a fresh Claude Code session is to review the existing codebase and flag anything that looks wrong or out of place — including but not limited to the known issue with `window.dlOpenPopup` living inside the Custom CSS panel.
