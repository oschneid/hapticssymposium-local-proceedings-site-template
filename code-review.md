# Code Review Report — IEEE Haptics Symposium 2026 Proceedings Site

**Files reviewed:** `site-template/index.html`, `site-template/papers.js`
**Date:** 2026-03-19
**Reviewer:** Claude as an "External developer" by following this prompt:

 > *As an external developer, can you please conduct a code review for this website for potential bugs, style issues, and security/privacy issues, and generate a full report of this review without using bash?*


---

## 1. Bugs

### BUG-01 — Critical: Filter buttons use wrong type values (zero results)

**Location:** `site-template/index.html:454-455`

The "Regular Papers" and "Long Papers" sidebar buttons pass `'Regular'` and `'Long'` to `setFilter()`, but no paper in `papers.js` has those `type` values. All non-WIP papers use `"Full Paper"`. Clicking either button will always produce zero results.

```html
<!-- BUG: data-value and setFilter argument disagree, and neither matches actual data -->
<button … data-value="Full Paper" onclick="setFilter('type','Regular',this)">Regular Papers</button>
<button … data-value="Long Paper"  onclick="setFilter('type','Long',this)">Long Papers</button>
```

The actual types present in the data are `"Full Paper"`, `"WIP"` — with `"ToH"` referenced in the sidebar but not used in any entry either. The filter, the button label, the `data-value`, and the `setFilter` argument all need to agree on one set of canonical type strings.

---

### BUG-02 — Moderate: `typeShort` / `typeClass` label collapses all non-WIP types to "Full"

**Location:** `site-template/index.html:565-566`

```js
const typeClass = p.type === 'WIP' ? 'type-wip' : 'type-full';
const typeShort = p.type === 'WIP' ? 'WiP' : 'Full';
```

If "Long Paper" or "ToH" types are ever used, they will display as "Full" with the same green badge. Each type should be handled explicitly.

---

### BUG-03 — Moderate: Fuzzy search `~1` appended to entire query string

**Location:** `site-template/index.html:513`

```js
const results = lunrIndex.search(searchQuery + '~1');
```

In Lunr 2.x, `~1` is a per-term edit-distance modifier. Appending it to a multi-word query (e.g., `"sensor fusion~1"`) may cause a parse error and hit the `catch` block, returning zero results. The modifier should be applied per-term: `searchQuery.trim().split(/\s+/).map(t => t + '~1').join(' ')`.

---

### BUG-04 — Minor: `.paper-card.hidden` CSS class is defined but never applied

**Location:** `site-template/index.html:328`

```css
.paper-card.hidden { display: none; }
```

Filtering works by rebuilding the DOM entirely inside `render()`, so this class is never used. It can be removed.

---

### BUG-05 — Minor: Placeholder ISBN in hero

**Location:** `site-template/index.html:427`

```html
ISBN 978-x-xxxx-xxxx-x
```

This is an unfilled template placeholder that will appear verbatim to all visitors.

---

### BUG-06 — Minor: Placeholder paper data

**Location:** `site-template/papers.js:1-200`

The entire `PAPERS` array contains fictional placeholder papers (embedded AI, federated learning, autonomous systems, etc.) unrelated to haptics research. This is the primary data source and must be replaced before deployment.

---

## 2. Security & Privacy Issues

### SEC-01 — High: Missing `rel="noopener noreferrer"` on `target="_blank"` PDF links ✅ Fixed

**Location:** `site-template/index.html:575`

```js
`<a class="paper-pdf-link" href="${p.pdf}" target="_blank">↓ PDF</a>`
```

Opening a new tab without `rel="noopener noreferrer"` exposes the originating page to reverse tabnapping: the linked page can access `window.opener` and redirect the original tab. Fix:

```js
`<a class="paper-pdf-link" href="${p.pdf}" target="_blank" rel="noopener noreferrer">↓ PDF</a>`
```

---

### SEC-02 — High: XSS via `innerHTML` with unsanitized data

**Locations:** `site-template/index.html:554-559`, `site-template/index.html:569-577`

Paper titles, author strings, session names, days, and times from `papers.js` are directly interpolated into `innerHTML`. If any of those strings contained characters like `<`, `>`, or `"` (e.g., a paper title using `<em>`-style notation, or a maliciously crafted data file), it would execute as HTML/JS. Since `papers.js` is currently static and developer-controlled this is low exploitability today, but is an unsafe pattern that should be corrected — particularly since the `highlight()` function also wraps results in `<mark>` tags and returns them as raw HTML.

**Recommendation:** Use `document.createElement` + `.textContent` for text nodes, or sanitize all values with a function like:
```js
function esc(s) { return String(s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }
```

---

### SEC-03 — Medium: No Subresource Integrity (SRI) on CDN-loaded resources

**Locations:** `site-template/index.html:7-9`

```html
<link href="https://fonts.googleapis.com/css2?…" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/lunr.js/2.3.9/lunr.min.js"></script>
```

Neither the Google Fonts stylesheet nor the Lunr.js script include an `integrity` attribute. If cdnjs or the Google Fonts CDN are compromised, users would be served malicious content with no browser-level protection. SRI hashes should be added to any externally-loaded resource whose content is expected to be stable.

---

### SEC-04 — Medium: Google Fonts leaks visitor IP to Google (contradicts "local" intent)

**Locations:** `site-template/index.html:7-8`

The README describes this as a **local** site for distributing proceedings to attendees. Loading fonts from `fonts.googleapis.com` means every page load sends the visitor's IP address and browser fingerprint to Google — a third party with no relationship to the attendees or the conference. Options: self-host the IBM Plex fonts, use a `@font-face` with local files, or at minimum document this in the README.

---

### SEC-05 — Low: No Content Security Policy

**Location:** `site-template/index.html` (missing)

There is no `Content-Security-Policy` meta tag. A restrictive CSP (e.g., restricting scripts to `'self'` and the known CDN origin) would prevent injected scripts from executing, providing defence-in-depth against SEC-02.

---

## 3. Style & Code Quality Issues

### STYLE-01 — Misleading CSS variable names

**Location:** `site-template/index.html:14-28`

The variable names describe colors they do not represent:
- `--navy` = `#fffdfc` (near-white cream)
- `--navy2` = `#f5f3f2` (light cream)
- `--white` = `#2b2523` (dark brown)

This makes the stylesheet very hard to maintain. Variable names should match their semantic role (e.g., `--bg`, `--surface`, `--text`) or their actual color.

---

### STYLE-02 — Empty CSS ruleset

**Location:** `site-template/index.html:219`

```css
.main {}
```

This is a no-op and should be removed.

---

### STYLE-03 — `float: right` inside a flex context

**Location:** `site-template/index.html:214`

`.filter-count` uses `float: right` inside `.filter-btn`, which is a flex container (since `filter-btn` has `display: block`). `float` has no effect inside a flex container. The intent is to push the count to the right; this could be achieved with `margin-left: auto` on `.filter-count` and making `.filter-btn` a flex container (`display: flex; justify-content: space-between`).

---

### STYLE-04 — Hardcoded `top: 72px` magic number in sidebar

**Location:** `site-template/index.html:167`

```css
.sidebar { position: sticky; top: 72px; }
```

72px is the header height (56px) plus 16px padding — but this is not derived from the actual header height. If the header height changes, the sidebar will not track correctly. Consider using a CSS custom property `--header-height: 56px` and computing `top: calc(var(--header-height) + 1rem)`.

---

### STYLE-05 — Sub-12px font sizes throughout

**Locations:** `site-template/index.html:59`, `site-template/index.html:67`, `site-template/index.html:98`, `site-template/index.html:143`, and many others

Font sizes of `10px` and `11px` appear in at least 10 places. Many users have difficulty reading text below 12px, and WCAG 2.1 Success Criterion 1.4.4 recommends supporting up to 200% text zoom. Using `rem` units throughout would allow users to control minimum text size via browser preferences.

---

### STYLE-06 — No visible keyboard focus indicator on filter buttons

**Location:** `site-template/index.html:183-208`

`.filter-btn` has `:hover` styling but no `:focus` or `:focus-visible` style. Keyboard-only users cannot see which button is focused. WCAG 2.1 SC 2.4.7 requires visible focus indicators.

---

### STYLE-07 — Sidebar inaccessible on mobile with no alternative

**Location:** `site-template/index.html:159-162`

```css
@media (max-width: 700px) {
  .layout { grid-template-columns: 1fr; }
  .sidebar { display: none; }
}
```

On mobile, the entire sidebar (day and type filters) is hidden with no fallback UI (e.g., a dropdown or collapsible panel). Mobile users can only search, not filter.

---

### STYLE-08 — Search input has no `<label>` element

**Location:** `site-template/index.html:465`

```html
<input type="search" id="search" placeholder="Search by title, author, keyword…" …>
```

There is no `<label for="search">` element. Screen readers rely on labels to announce the purpose of form controls. The placeholder alone is not sufficient per WCAG 2.1 SC 1.3.1.

---

### STYLE-09 — Inconsistent type label casing

**Location:** `site-template/index.html:566`

`typeShort` produces `'WiP'` (mixed case) but the CSS class is `type-wip` and the sidebar button says `"Work-in-Progress"`. Pick one consistent abbreviation (e.g., "WIP") and use it everywhere.

---

## Summary Table

| ID | Severity | Category | Description |
|---|---|---|---|
| BUG-01 | Critical | Bug | Filter buttons pass wrong type values — always returns 0 results |
| SEC-01 | High | Security | ✅ Fixed — Missing `rel="noopener noreferrer"` on `target="_blank"` links |
| SEC-02 | High | Security | XSS via `innerHTML` with unescaped data |
| SEC-03 | Medium | Security | No SRI hashes on CDN resources |
| SEC-04 | Medium | Privacy | Google Fonts sends visitor IPs to Google on a "local" site |
| BUG-02 | Moderate | Bug | Non-WIP types incorrectly labeled "Full" |
| BUG-03 | Moderate | Bug | Lunr fuzzy `~1` appended to full query, not per-term |
| SEC-05 | Low | Security | No Content Security Policy |
| BUG-04 | Minor | Bug | Dead `.paper-card.hidden` CSS class |
| BUG-05 | Minor | Bug | Unfilled ISBN placeholder visible to users |
| BUG-06 | Minor | Bug | Placeholder paper data not replaced before deployment |
| STYLE-01 | Moderate | Style | CSS variable names are semantically inverted |
| STYLE-06 | Moderate | Accessibility | No focus indicator on filter buttons |
| STYLE-07 | Moderate | Accessibility | Filters entirely hidden on mobile, no alternative |
| STYLE-08 | Moderate | Accessibility | Search input has no `<label>` element |
| STYLE-02 | Minor | Style | Empty `.main {}` CSS ruleset |
| STYLE-03 | Minor | Style | `float: right` inside flex container has no effect |
| STYLE-04 | Minor | Style | Hardcoded `72px` magic number in sidebar position |
| STYLE-05 | Minor | Style | Sub-12px pixel font sizes, no `rem` units |
| STYLE-09 | Minor | Style | "WiP" label casing inconsistent with rest of UI |

---

**Priority recommendations:**
1. Fix BUG-01 immediately — the type filter buttons are non-functional.
2. Fix SEC-01 (add `rel="noopener noreferrer"`) — one-line fix, meaningful security gain. ✅ Done
3. Replace placeholder data (BUG-06) and the ISBN (BUG-05) before any distribution.
4. Address SEC-04 by self-hosting fonts if this site is truly meant to be served locally.
5. Rename CSS variables (STYLE-01) to reduce maintenance confusion going forward.
