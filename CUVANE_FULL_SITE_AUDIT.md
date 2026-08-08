# Cuvane Full Site Audit

Audit date: 2026-08-07
Auditor: opencode technical audit (read-only — no files modified, no commits made)
Target: `cuvane.com`, `www.cuvane.com`, `mikepflanagan.github.io/cuvane/`, and the local repo `/Users/home/Cuvane`

---

## 1. Executive Summary

**Overall health score: 74/100**

The site is a small, hand-written, 3-page static HTML site served by GitHub Pages under a custom domain. The content and HTML quality are solid, performance is excellent (no JS, no images, no external requests), and WCAG color contrast passes everywhere. The single critical problem is **HTTPS is not yet working on the custom domain** — every `https://` request currently fails TLS verification because GitHub has not yet provisioned the Let's Encrypt certificate for `cuvane.com`. The likely cause was a bad `www` CNAME record (accented `ï` → punycode `xn--mikepflanagan-o20i.github.io`) that has **now been corrected** to `mikepflanagan.github.io`; the certificate should provision automatically once DNS settles. Secondary issues: missing canonical/OG/robots/sitemap, one external `github.io` link, non-heading section labels, a publicly-exposed Runbook, and incomplete A-record set.

| Category | Health | Score |
|---|---|---|
| Deployment health | Good — clean static deploy, commit `5edcb34` live | 9/10 |
| DNS/domain health | Fair — apex has 1 of 4 A records; www CNAME just fixed | 6/10 |
| HTTPS health | **Broken** — TLS handshake fails, cert not provisioned | 2/10 |
| HTML health | Good — valid HTML5, only style-rule nits | 8/10 |
| Link health | Good — all internal links resolve; 1 external link | 8/10 |
| Accessibility health | Fair — contrast excellent, heading order & semantics need work | 6/10 |
| SEO health | Fair — good titles/descriptions, missing canonical/robots/sitemap/OG | 6/10 |
| Performance health | Excellent — 11–16 KB pages, zero requests | 10/10 |
| Security health | Fair — no secrets, but Runbook/source files publicly served | 6/10 |

---

## 2. Critical Findings

| # | Severity | Finding | Evidence | Files / URLs | Root cause | Fix | Fix type |
|---|---|---|---|---|---|---|---|
| C1 | **Critical** | HTTPS broken on `cuvane.com` and `www.cuvane.com` | `curl https://cuvane.com/` → SSL error 60 "no alternative certificate subject name matches cuvane.com"; cert served is `CN=*.github.io` (Let's Encrypt YR2). Pages API reports `https_enforced: false` | `cuvane.com`, `www.cuvane.com` | GitHub Pages auto-provisions a Let's Encrypt cert only after DNS validation succeeds. A bad `www` CNAME (`xn--mikepflanagan-o20i.github.io`, i.e. `mïkepflanagan…`) blocked validation. Record is **now correct** (`mikepflanagan.github.io`) but cert has not yet been issued | Wait for automatic issuance (up to ~24 h). If still failing, confirm all 4 GitHub A records exist on the apex, then re-check. **Do not manually install a cert** — GitHub Pages manages it | Manual (Porkbun DNS + GitHub Pages auto-provisioning) |
| C2 | **High** | HTTP is served directly; no HTTP→HTTPS redirect | `http://cuvane.com/` returns 200 with content (not a redirect to https) | `cuvane.com` | GitHub only enforces HTTPS after the custom-domain cert is provisioned | Once C1 resolves, GitHub auto-serves HTTPS and 301s HTTP→HTTPS. No code change | Manual (automatic after C1) |
| C3 | **High** | `www.cuvane.com` has only a working CNAME but apex has only **1 of 4** GitHub A records | `dig +short A cuvane.com` → `185.199.108.153` only (109/110/111 missing); `dig +short CNAME www.cuvane.com` → `mikepflanagan.github.io` ✓ | `cuvane.com` DNS at Porkbun | Only the first A record was added at Porkbun | Add `185.199.109.153`, `185.199.110.153`, `185.199.111.153` at Porkbun | Manual (Porkbun) |
| C4 | **High** | Internal source/business files are publicly served | `http://cuvane.com/Runbook/Cuvane-Launch-Runbook.md` → 200; `.html` → 200; `.pdf` → 200; `.docx` → 200; `README.md` → 200; `CNAME` → 200; `Ventures/*/.gitkeep` → 200 | `http://cuvane.com/Runbook/*`, `http://cuvane.com/README.md`, etc. | The entire repo root is the Pages document root (`source: main /`); every committed file is web-served | Move `Runbook/` and `Ventures/` out of the Pages-served path (e.g. into `docs/` excluded from Pages, or a private repo), or set Pages source to a subdirectory containing only `index.html`, `aura/`, `personalai/`, `.nojekyll`, `CNAME` | Code/repo-structure change |

---

## 3. Domain / DNS / HTTPS Findings

- **Apex A records** (Porkbun, TTL 30 s): `185.199.108.153` present; **109/110/111 missing** → partial GitHub Pages config (C3).
- **`www` CNAME**: `www.cuvane.com → mikepflanagan.github.io` (TTL 535 s). **Was previously broken** (`xn--mikepflanagan-o20i.github.io` = `mïkepflanagan.github.io`, accented `ï` encoded as punycode) — **fixed and confirmed** during this audit.
- **Nameservers**: `curitiba/salvador/fortaleza/maceio.ns.porkbun.com` — correct Porkbun NS.
- **MX**: `10 mx.zoho.com`, `20 mx2.zoho.com` — correct for Zoho Mail.
- **TXT**: `v=spf1 include:_spf.porkbun.com include:zoho.com ~all` and `zoho-verification=zb45722735.zmverify.zoho.com` — correct.
- **HTTPS apex**: FAILS (`curl` error 60; GitHub fallback `*.github.io` cert served) (C1).
- **HTTPS www**: FAILS (same fallback cert; hostname `www.cuvane.com` not in SAN) (C1).
- **HTTP apex**: 200, serves content directly (no redirect to https) (C2).
- **HTTP www**: 301 → `http://cuvane.com/` ✓ (GitHub's www→apex redirect, works over HTTP).
- **github.io canonical**: `https://mikepflanagan.github.io/cuvane/` → 301 → `http://cuvane.com/` ✓ (redirects to custom domain).
- **Redirect loop risk**: none observed. Apex↔www consistency: www 301s to apex — correct direction.
- **Canonical host consistency**: not yet enforceable because HTTPS is down; once C1 resolves, apex will be canonical and www will redirect.
- **Mixed content**: none (no subresources of any kind).
- **CNAME file** in repo: `cuvane.com` ✓ matches Pages API `cname: cuvane.com`.
- **Conclusion on the three hosts**: `cuvane.com` / `www.cuvane.com` / `mikepflanagan.github.io` are **configured correctly together at the DNS layer** (verified live). Only the TLS certificate provisioning is pending.

---

## 4. GitHub Pages / Deployment Findings

- **Remote**: `origin https://github.com/MikePFlanagan/cuvane.git` (HTTPS, no SSH).
- **Repo**: `MikePFlanagan/cuvane`, PUBLIC, default branch `main`.
- **Serving branch**: `main`, source path `/` (repo root). Pages API: `build_type: legacy`, `status: built`, `public: true`.
- **Build**: GitHub Pages static deploy (the auto `pages-build-deployment` workflow, `state: active`). No `_config.yml`, no Jekyll (`build_type: legacy` + `.nojekyll` = plain static serve).
- **Latest build**: commit `5edcb34`, duration 18 s, status `built`, no error.
- **CNAME handling**: file `CNAME` in repo root contains `cuvane.com`; Pages API confirms `cname: cuvane.com`.
- **`https_enforced`: false** — GitHub will flip this automatically once the custom-domain cert is issued (C1).
- **custom_404: false** — GitHub's default 404 page is used (fine).
- **Paths/URLs**: internal links are relative (`aura/`, `personalai/`) or absolute `https://cuvane.com/…`; no `_site` build output, no asset path rewrite issues.
- **Nothing** in the deployment prevents static asset serving or link resolution. The only deployment-level blocker is the pending cert (C1) and the A-record set (C3).

---

## 5. Complete URL Inventory

| URL | Status | Final URL | Title | H1 | Issues |
|---|---|---|---|---|---|
| `http://cuvane.com/` | 200 | self | Cuvane — Under pressure, we make diamonds. | Under pressure, we make diamonds. | No HTTPS; no canonical/OG |
| `http://cuvane.com/aura/` | 200 | self | Cuvane Aura — the machine that makes diamonds | Coal goes in. Brilliance comes out. | No HTTPS; no canonical/OG |
| `http://cuvane.com/personalai/` | 200 | self | PersonalAI — The operating system for your everyday | The operating system for your everyday. | No HTTPS; no canonical/OG |
| `http://cuvane.com/personalai` | 301 | → `…/personalai/` | — | — | OK (trailing slash added) |
| `http://cuvane.com/aura` | 301 | → `…/aura/` | — | — | OK |
| `http://cuvane.com/aura/index.html` | 200 | self | (same as aura/) | — | Duplicate route of `/aura/` |
| `http://cuvane.com/index.html` | 200 | self | (same as root) | — | Duplicate route of `/` (benign) |
| `https://cuvane.com/` | 000 / SSL err | — | — | — | **Critical C1** |
| `https://www.cuvane.com/` | 000 / SSL err | — | — | — | **Critical C1** |
| `http://www.cuvane.com/` | 301 | → `http://cuvane.com/` | — | — | OK |
| `https://mikepflanagan.github.io/cuvane/` | 301 | → `http://cuvane.com/` | — | — | OK (canonical redirect) |
| `http://cuvane.com/sitemap.xml` | 404 | GitHub 404 page | Page not found · GitHub Pages | 404 | **Missing sitemap** |
| `http://cuvane.com/robots.txt` | 404 | GitHub 404 page | — | — | **Missing robots.txt** |
| `http://cuvane.com/404.html` | 404 | GitHub 404 page | — | — | Default 404 (fine) |
| `http://cuvane.com/INDEX.html` | 404 | — | — | — | Case-sensitive (expected) |
| `http://cuvane.com/Aura/` | 404 | — | — | — | Case-sensitive (expected) |

---

## 6. Internal Link Graph

| Source | Anchor | Destination | Status | Issue |
|---|---|---|---|---|
| `/` | CUVANE (brand) | `#top` | OK | No `id="top"` on `<header>`; it exists on `<main id="top">` — works |
| `/` | Ventures / Story / Contact | `#ventures` `#story` `#contact` | OK | All IDs exist |
| `/` | DealDiamondCRM (card) | `https://mikepflanagan.github.io/dealdiamondcrm-site/` | OK (200) | External github.io link — see F1 |
| `/` | PersonalAI (card) | `personalai/` (relative) | OK (200) | Relative while subpages use absolute — inconsistent (F2) |
| `/` | Aura theme (footer) | `aura/` (relative) | OK (200) | Relative in footer — F2 |
| `/aura/` | CUVANE · AURA (brand) | `https://cuvane.com/` | OK over HTTP | HTTPS pending |
| `/aura/` | Palette / Icons / The Flow / Tokens | `#palette` `#icons` `#flow` `#tokens` | OK | All IDs exist |
| `/aura/` | Back to Cuvane | `https://cuvane.com/` | OK over HTTP | HTTPS pending |
| `/personalai/` | PersonalAI (brand) | `https://cuvane.com/` | OK over HTTP | HTTPS pending |
| `/personalai/` | What it is / Pillars / Status | `#what` `#pillars` `#status` | OK | All IDs exist |
| `/personalai/` | Back to Cuvane | `https://cuvane.com/` | OK over HTTP | HTTPS pending |
| `/personalai/` | Aura theme (footer) | `https://cuvane.com/aura/` | OK over HTTP | HTTPS pending |

No 404s, no 403s, no 500s, no malformed URLs, no localhost URLs, no `javascript:` links, no empty links, no broken fragments. No `target="_blank"` anywhere (so no new-tab issues).

---

## 7. Broken Links

- **No broken internal links** across all reachable pages (all resolve to 200 or valid 301→200).
- External link `https://mikepflanagan.github.io/dealdiamondcrm-site/` resolves 200 (validated only, not crawled — per scope).
- `http://cuvane.com/dealdiamondcrm-site/` → 404 (this was **never linked**; tested only as a probe).

---

## 8. HTML Findings

Validation tool: `html-validate` (HTML5-aware) and `tidy` (legacy HTML4 — its `<svg>` complaints are false positives for HTML5).

| # | Severity | Finding | Files | Detail |
|---|---|---|---|---|
| H1 | Low | `no-inline-style` rule violations | `index.html:242`, `aura/index.html:205`, `personalai/index.html:116` | `<svg width="0" height="0" style="position:absolute" aria-hidden="true">` — inline style on the sprite container |
| H2 | Low | Inline styles on swatch chips | `aura/index.html:317–322` | `style="background:#…"` on 6 `.chip` divs |
| H3 | Low | `prefer-tbody` (Runbook only) | `Runbook/Cuvane-Launch-Runbook.html:237,249,307` | `<tr>` directly in `<table>` without `<tbody>` |
| H4 | Medium | Section labels are `<div>`, not headings | `index.html` (`#ventures`, `#story`, `#contact`), `personalai/index.html` (`#what`, `#pillars`, `#status`) | `.section-head` is a `<div>` — no semantic `<h2>`. Headings jump H1→H3 |
| H5 | Medium | Heading hierarchy skips levels | `index.html`, `personalai/index.html` | H1 → H3 (no H2). `aura/` has correct H1→H2→H3 |
| H6 | Low | Duplicate-route pages without canonical | `index.html` ↔ `/index.html`; `aura/index.html` ↔ `/aura/` | Same content reachable two ways; no `<link rel="canonical">` to disambiguate |
| H7 | Low | Decorative SVGs not all `aria-hidden` | `index.html` machine-stage coal/machine/diamond SVGs | Some decorative `<svg>` have no `aria-hidden="true"` |
| H8 | Info | No DOCTYPE/lang/charset/viewport problems | all 3 pages | `<!DOCTYPE html>`, `<html lang="en">`, UTF-8, viewport present on every page — GOOD |
| H9 | Info | No duplicate IDs | all 3 pages | Verified all IDs unique — GOOD |
| H10 | Info | No malformed HTML / bad nesting | all 3 pages | html-validate clean apart from style rules — GOOD |

---

## 9. CSS / Responsive Findings

| # | Severity | Finding | Detail |
|---|---|---|---|
| R1 | Low | CSS is 100% inline `<style>` in each page | ~7 KB duplicated across 3 pages; no shared stylesheet. Fine for 3 pages; note for maintenance |
| R2 | Low | Three different `--max` widths | `index.html` 64rem, `aura` 68rem, `personalai` 60rem — inconsistent gutter widths between pages |
| R3 | Low | `@media (max-width: 40rem)` identical in all 3 pages | Only one mobile breakpoint; no tablet breakpoint. Grids use `auto-fit minmax` so they degrade gracefully — no horizontal scroll expected (NOT VERIFIED in a real browser) |
| R4 | Info | No fixed-width elements, no layout shifts | Pure CSS, no fonts/images/JS — CLS risk ~zero |
| R5 | Info | Contrast passes WCAG AA on all text/background pairs | Computed: 7.13–17.7:1 vs 4.5:1 requirement (see Commands section) — GOOD |
| R6 | Info | No unused/conflicting CSS detected | Small, hand-written, coherent — GOOD |

---

## 10. JavaScript Findings

- **Zero JavaScript** on any page (0 `<script>` tags, 0 external scripts). Consequently: no console errors, no uncaught exceptions, no CORS issues, no broken handlers, no dead buttons (only CSS animations), no DOM-dependent behavior. This is a **strength** for performance and security.
- Nothing to fix in this category.

---

## 11. Asset Findings

- **Images**: none (all graphics are inline SVG symbols). No scaling/optimization concerns.
- **Favicon**: inline `data:` URI SVG on all 3 pages — valid, no 404. No `.png`/`.ico`/`.webmanifest` (minor).
- **Fonts**: none loaded — system font stack (`-apple-system, …`) — no font-loading cost.
- **CSS**: inline only (see R1).
- **PDFs/DOCX/MD** served as files: Runbook (see C4) — served but not linked.
- **Missing assets**: none.
- **Mixed content**: none.

---

## 12. SEO Findings

| # | Severity | Finding | Detail |
|---|---|---|---|
| S1 | Medium | **No canonical tags** | No `<link rel="canonical">` on any page; apex/www/github.io and `/` vs `/index.html` create duplicate-URL ambiguity |
| S2 | Medium | **No `robots.txt`** (404) | Search engines get GitHub's 404 default; cannot control crawl |
| S3 | Medium | **No `sitemap.xml`** (404) | No index of the 3 pages |
| S4 | Medium | **No Open Graph / Twitter Card meta** | No `og:title`, `og:description`, `og:url`, `og:image`, `twitter:*` — poor link sharing |
| S5 | Low | No structured data / schema | No JSON-LD (Organization could be added) |
| S6 | Info | Titles & descriptions present and unique on all 3 pages | GOOD |
| S7 | Info | Clean URLs, proper H1 on each page | GOOD |
| S8 | Info | No duplicate titles/descriptions across pages | GOOD |
| S9 | Info | Image alt text: N/A (no raster images); inline SVG is decorative | GOOD |
| S10 | Info | Redirect consistency once HTTPS lives | www→apex 301 correct; github.io→cuvane.com correct |

---

## 13. Accessibility Findings (WCAG 2.2 AA)

| # | Severity | Finding | WCAG ref |
|---|---|---|---|
| A1 | Medium | **No skip-to-content link** | 2.4.1 Bypass Blocks |
| A2 | Medium | Heading order skips levels (H1→H3) and section labels are `<div>` not `<h2>` | 1.3.1 Info & Relationships, 2.4.6 Headings |
| A3 | Low | Decorative machine-stage SVGs not `aria-hidden` | 1.1.1, 4.1.2 |
| A4 | Info | **Color contrast PASSES** (all text ≥7.1:1) | 1.4.3 AA |
| A5 | Info | **Keyboard navigation**: default focus outlines intact (no `outline:none` anywhere); all interactive elements are real `<a>` | 2.1.1, 2.4.7 |
| A6 | Info | No `target="_blank"` — no focus-trap/new-tab concern | 2.5.4 |
| A7 | Info | No forms on any page — no label issues | — |
| A8 | Info | Interactive target sizes: cards/nav links are comfortably > 24 px | 2.5.8 |
| A9 | Info | Link text is descriptive ("Ventures", "Back to Cuvane", "DealDiamondCRM") | 2.4.4 |

---

## 14. Performance Findings

- **Total page weight**: `/` 14.2 KB HTML; `/aura/` 16.3 KB; `/personalai/` 11.5 KB. Total across all pages ≈ 42 KB. **Excellent.**
- **Requests per page**: 1 (the HTML document itself). No CSS/JS/image/font requests.
- **LCP impact**: LCP is the H1 text — paints near-instantly. Estimated LCP < 1 s on all connections.
- **CLS impact**: zero layout-shift sources (fixed sizes, no images/fonts). CLS ≈ 0.
- **INP impact**: no JS handlers → INP ≈ N/A (maximally good).
- **Render-blocking**: only the inline `<style>` block (required, tiny).
- **Caching**: GitHub Pages sets `Cache-Control: max-age=600`; fine for a static brochure site.
- **Lazy-loading**: nothing to lazy-load (no images). No changes needed.

---

## 15. Security / Privacy Findings

| # | Severity | Finding | Detail |
|---|---|---|---|
| P1 | **High** | **Business/strategy files publicly served** | `http://cuvane.com/Runbook/Cuvane-Launch-Runbook.{md,html,pdf,docx}` and `README.md` are fetchable by anyone. The Runbook discloses entity name (`MICHAELFLANAGANLLC`), trademark/budget plans, and internal checklists. No credentials, but sensitive business info. (C4) |
| P2 | Low | `.gitkeep` / `CNAME` / `.nojekyll` served as 200 | Harmless but sloppy exposure of repo internals |
| P3 | Info | **No secrets/API keys** | Grep for `secret|token|api[_-]?key|.env` over the repo found nothing; `.env` and `*.local` are in `.gitignore` — GOOD |
| P4 | Info | **No external scripts/embeds** | No third-party attack surface — GOOD |
| P5 | Info | No `target="_blank"` (no `rel` issues) | GOOD |
| P6 | Info | Email `hello@cuvane.com` shown as plain text (not a `mailto:`) | Not scrapable into a prefilled form; acceptable for a static site |
| P7 | Info | No CSP/HSTS/security headers | GitHub Pages does not allow custom headers; HSTS will be added by GitHub after HTTPS is enforced (C1). NOT VERIFIABLE on platform — informational |
| P8 | Info | No forms sending data insecurely | There are no forms — GOOD |

---

## 16. Content / UX Findings

| # | Severity | Finding | Detail |
|---|---|---|---|
| U1 | Medium | **Dead-end cards**: Jarvis and DriveHelp cards are `<div class="card">` (not links) while DealDiamondCRM and PersonalAI cards are links | Inconsistent affordance — some cards feel clickable, some aren't |
| U2 | Low | Footer lists "Cuvane.com · Cuvane.ai · Cuvane.io" as plain text | `.ai` is **not purchased** and `.io` is not yet live on Pages — readers may try them and get parked/error pages |
| U3 | Low | Contact section: "Coming soon — hello@cuvane.com once the mailboxes are live." | Accurate (Zoho MX is now set, mailbox still provisioning) — OK but temporary messaging |
| U4 | Low | README.md still says the site is "live at `https://mikepflanagan.github.io/cuvane/`" | That URL 301s to `cuvane.com`; README should reference the real domain |
| U5 | Low | Navigation label "PersonalAI" vs product framing "Personal OS" | Header/brand says "PersonalAI"; homepage card and copy say "PersonalAI"; user refers to it as "Personal OS" — pick one naming for consistency |
| U6 | Info | Copy quality | No grammar/spelling issues detected; tone consistent and strong |

---

## 17. Local vs Production Differences

- **Local HEAD**: `5edcb345da1ed4905570c8f358e976dd8873a3bf` — matches **deployed commit** (Pages latest build `5edcb345…`). No drift.
- **Working tree**: clean (`git status` empty).
- **Local files present but NOT in repo** (untracked, `.gitignore`d): `/Users/home/Cuvane/.DS_Store`. Not served.
- **Deployed files NOT represented locally**: none.
- **Deployed content vs local**: identical (same commit).
- **No build step**: what's in the repo root is exactly what's served (`source: main /`). No stale build, no generated assets, no `_site/`.

---

## 18. Dead / Legacy / Orphaned Files

| # | File | Status | Notes |
|---|---|---|---|
| D1 | `Runbook/Cuvane-Launch-Runbook.docx` | Live, unlinked, publicly served | Orphaned from a user standpoint; served as `application/vnd…wordprocessingml` |
| D2 | `Runbook/Cuvane-Launch-Runbook.pdf` | Live, unlinked, publicly served | Same |
| D3 | `Runbook/Cuvane-Launch-Runbook.md` | Live, unlinked, publicly served | Same |
| D4 | `Runbook/Cuvane-Launch-Runbook.html` | Live, unlinked, publicly served | Same |
| D5 | `Ventures/*/.gitkeep` (×4) | Live, unlinked, served as 200/octet-stream | Placeholder files exposed |
| D6 | `README.md` | Live, unlinked, served | Redundant with the site content |
| D7 | `CNAME`, `.nojekyll` | Required — keep | Not orphaned |
| D8 | `Ventures/` + `Runbook/` dirs in the served root | — | Cause of C4/D1–D6; consider moving out of Pages root |

---

## 19. Recommended Repair Order

### Phase 1 — Site-breaking
1. **HTTPS provisioning (C1)** — *Manual*: nothing to change now; the `www` CNAME is fixed. Verify daily; GitHub auto-issues the Let's Encrypt cert for `cuvane.com` + `www.cuvane.com` once DNS validates (up to ~24 h). If it hasn't appeared after 24 h, add the 3 missing A records (Phase 2) and confirm with GitHub's Pages settings screen.
2. **Add missing A records (C3)** — *Manual (Porkbun)*: add `185.199.109.153`, `185.199.110.153`, `185.199.111.153` for host `@`. Expected: full GitHub Pages IP coverage, faster cert validation. Risk: low (additive).

### Phase 2 — Domain / HTTPS / deployment
3. Confirm `https://cuvane.com/` and `https://www.cuvane.com/` return 200 with the correct cert; confirm `http://` now 301s to `https://`. — *Manual/verify*. If not, open GitHub → repo → Settings → Pages and check the "Custom domain" status; if it shows "unverified," re-save `cuvane.com`.

### Phase 3 — Broken pages / links
4. **No broken links exist.** Optionally repoint the DealDiamondCRM card from github.io to a branded URL (see F1 below) once HTTPS is up.

### Phase 4 — UX / responsive / accessibility
5. **Make Jarvis & DriveHelp cards links** — *Code (index.html:333–338, 344–348)*: wrap in `<a class="card" href="…">` (to future pages) or add a `aria-disabled`/visual affordance if intentionally placeholder. Expected: consistent clickability.
6. **Add skip link** — *Code (all 3 pages)*: `<a class="skip" href="#main">Skip to content</a>` + CSS. Expected: WCAG 2.4.1 pass.
7. **Fix heading structure** — *Code (index.html, personalai/index.html)*: change `.section-head` divs to `<h2 class="section-head">`; keep H1→H2→H3 order. Expected: proper outline; aligns with A2.

### Phase 5 — SEO / performance
8. **Add `robots.txt`** — *Code (new file, repo root)*: `User-agent: *` + `Allow: /` + `Sitemap: https://cuvane.com/sitemap.xml`. Expected: crawl control restored.
9. **Add `sitemap.xml`** — *Code (new file)*: URLs for `/`, `/aura/`, `/personalai/`. Expected: better indexing.
10. **Add canonical tags** — *Code (all 3 pages)*: `<link rel="canonical" href="https://cuvane.com/…">`. Expected: kills duplicate-URL ambiguity.
11. **Add Open Graph/Twitter meta** — *Code (all 3 pages)*: `og:title/og:description/og:url/og:type` (+ `twitter:card`). Expected: rich link previews.

### Phase 6 — cleanup
12. **Remove Runbook/README/Ventures from the served root (C4/D1–D6)** — *Code/repo-structure + manual GitHub setting*: either (a) set Pages source to a subfolder containing only the site files, or (b) move `Runbook/` and `Ventures/` into a `docs/`-style folder excluded from Pages, or into a private repo. Expected: `cuvane.com/Runbook/*` returns 404; sensitive strategy no longer public. **Requires Pages source reconfiguration in GitHub Settings.**
13. **Unify footer/relative links (F2)** — *Code (index.html:339,349,379)*: change `personalai/` and `aura/` to absolute `https://cuvane.com/…` (do **not** do this until Phase 1 completes, since HTTPS may be down). Expected: consistent absolute links.
14. **Tidy nits** — *Code*: remove the `style="position:absolute"` inline style on the sprite `<svg>` (move to CSS class), move swatch colors to classes (aura). Expected: html-validate clean.
15. **README update (U4)** — *Code (README.md:34–35)*: point to `https://cuvane.com` not github.io.

---

## 20. Code-Fixable vs Manual-Fix Matrix

| Issue | Code Fix Available? | Exact File | Manual Platform Step Required? | Platform | Recommended Action |
|---|---|---|---|---|---|
| HTTPS cert not provisioned (C1) | No | — | Yes (wait/verify) | GitHub Pages + Porkbun | Verify `www` CNAME correct (done); wait for auto-issuance; re-save custom domain if unverified |
| Missing 3 apex A records (C3) | No | — | Yes | Porkbun DNS | Add 185.199.109/.110/.111.153 |
| HTTP→HTTPS redirect (C2) | No | — | Yes (automatic) | GitHub Pages | Follows cert issuance automatically |
| Runbook/source files exposed (C4) | Yes + manual | repo restructure + Settings→Pages source | Yes | GitHub Pages settings | Move files / change Pages source dir |
| No canonical tags (S1) | Yes | `index.html`, `aura/index.html`, `personalai/index.html` | No | — | Add `<link rel="canonical">` |
| No robots.txt / sitemap (S2/S3) | Yes | new `robots.txt`, `sitemap.xml` | No | — | Add files |
| No OG/Twitter meta (S4) | Yes | all 3 pages | No | — | Add meta tags |
| Dead-end Jarvis/DriveHelp cards (U1) | Yes | `index.html` | No | — | Make links or add affordance |
| Skip link + heading order (A1/A2) | Yes | all 3 pages | No | — | Add skip link; div→h2 |
| External github.io link (F1) | Yes | `index.html:339` | Optional | — | Repoint after HTTPS live |
| Relative footer links (F2) | Yes | `index.html:349,379` | No | — | Absolutize after HTTPS live |
| Inline-style nits (H1/H2) | Yes | `aura/index.html`, sprite `<svg>` in all 3 | No | — | Move to CSS |
| README stale URL (U4) | Yes | `README.md` | No | — | Update to cuvane.com |

---

## 21. Commands and Evidence

| Command | Result summary |
|---|---|
| `git -C /Users/home/Cuvane remote -v` | origin = `https://github.com/MikePFlanagan/cuvane.git` |
| `git -C /Users/home/Cuvane log --oneline -15` | 10 commits; HEAD `5edcb34` |
| `git -C /Users/home/Cuvane status --short` | clean |
| `git -C /Users/home/Cuvane ls-files` | 17 tracked files (site + Runbook + Ventures `.gitkeep` + CNAME/.nojekyll) |
| `gh repo view MikePFlanagan/cuvane` | PUBLIC, default branch main, language HTML |
| `gh api …/pages` | `status: built`, `cname: cuvane.com`, `https_enforced: false`, `build_type: legacy`, source main `/`, `public: true` |
| `gh api …/pages/builds/latest` | `status: built`, commit `5edcb345…`, duration 18161 ms, no error |
| `gh api …/actions/workflows` | only auto `pages-build-deployment`, active |
| `dig +short A cuvane.com` | `185.199.108.153` only (3 missing) |
| `dig +short CNAME www.cuvane.com` | `mikepflanagan.github.io` (was `xn--mikepflanagan-o20i…` → fixed) |
| `dig +short NS/MX/TXT cuvane.com` | Porkbun NS; Zoho MX (mx.zoho.com, mx2.zoho.com); SPF incl zoho; zoho verification TXT |
| `curl` on 9 URL variants | apex/www HTTPS fail (SSL err 60); HTTP apex 200; HTTP www 301→apex; github.io 301→apex; trailing-slash 301s OK |
| `openssl s_client` to `cuvane.com:443` | serving `CN=*.github.io` fallback — no cuvane cert yet |
| `html-validate` on 4 files | only `no-inline-style` (7×) and `prefer-tbody` (3×, Runbook) |
| `tidy -errors` on index.html | HTML4-validator false positives on `<svg>`/`<defs>`; no real structural errors |
| Custom Python crawler | extracted titles, H1/H2/H3, hrefs, IDs, scripts, imgs, og for all 3 pages; verified all fragments resolve |
| Contrast computation script | all text pairs 7.13–17.7:1 (≥4.5:1 required) |
| `curl` on Runbook/README/Ventures paths | all served 200 (C4/D1–D6) |
| Grep for secrets | `secret|token|api[_-]?key|.env` → none |

---

## 22. Final Recommended Action Plan

The simplest path to production-ready:

1. **Now (0 action required):** The bad `www` CNAME was already corrected to `mikepflanagan.github.io` — confirm it still reads that, then let GitHub auto-issue the HTTPS cert (typically <24 h). This single fix unblocks HTTPS on both apex and www and enables HTTP→HTTPS enforcement automatically.
2. **While waiting (5 min, manual):** At Porkbun add the three missing GitHub A records (`185.199.109/110/111.153`) for `@` to match the existing `185.199.108.153`.
3. **Verify:** Once the cert is live, confirm `https://cuvane.com/` and `https://www.cuvane.com/` both return 200 with the cuvane.com certificate, and that `http://` 301s to `https://`.
4. **Code pass (one commit):** Add `robots.txt` + `sitemap.xml` + canonical tags + OG/Twitter meta; fix heading order and section labels to `<h2>`; add a skip link; make Jarvis/DriveHelp cards links; move the `style="position:absolute"` inline style to a class; update README URL; (after HTTPS) absolutize the remaining relative links.
5. **Housekeeping (manual + repo):** Move `Runbook/` and `Ventures/` out of the Pages-served root (change Pages source directory or relocate the files) so `cuvane.com/Runbook/*` and `README.md` are no longer public, and delete the leftover `.gitkeep` exposure if files move.
6. Re-run the crawl + `html-validate` to confirm a clean state.

Everything in steps 4–5 is low-risk and additive. The only thing with real risk is changing the Pages **source directory** (step 5) — if misconfigured the site goes 404, so verify the new path still contains `CNAME`, `.nojekyll`, `index.html`.

---

## COPY/PASTE HANDOFF TO CHATGPT

**1. What is broken**
- HTTPS does not work on `cuvane.com` or `www.cuvane.com` — TLS handshake fails ("no alternative certificate subject name matches cuvane.com"); GitHub is serving its fallback `*.github.io` cert. GitHub Pages reports `https_enforced: false`. HTTP still serves content directly (no redirect to HTTPS).
- Apex DNS has only 1 of 4 GitHub Pages A records (`185.199.108.153`); 109/110/111 are missing.
- No `robots.txt`, no `sitemap.xml`, no canonical tags, no Open Graph/Twitter meta on any page.
- Business files are publicly exposed: `cuvane.com/Runbook/*` (md/html/pdf/docx), `cuvane.com/README.md`, `cuvane.com/Ventures/*/.gitkeep`, `CNAME`, `.nojekyll` all return 200.
- Homepage cards for Jarvis and DriveHelp are not links (dead ends); section labels are `<div>` not `<h2>`; no skip link; some decorative SVGs lack `aria-hidden`.

**2. What is working**
- All 3 HTML pages are valid (html-validate), with unique titles/descriptions, proper viewport/charset/lang, no duplicate IDs, no malformed fragments, no broken internal links.
- All internal anchors resolve; trailing-slash 301s work; `www.cuvane.com` 301s to the apex; `mikepflanagan.github.io/cuvane/` 301s to the custom domain.
- The bad `www` CNAME (`xn--mikepflanagan-o20i.github.io` = accented `mïkepflanagan`) has already been fixed to `mikepflanagan.github.io` — the blocker for cert issuance is likely gone.
- Zoho MX + SPF + verification TXT are correct.
- No JS, no images, no external requests; total ~42 KB for the whole site; WCAG contrast passes (7.1–17.7:1); zero secrets in the repo.
- Local HEAD == deployed commit (`5edcb34`), working tree clean.

**3. What code can fix**
- Add `robots.txt`, `sitemap.xml`, canonical tags, OG/Twitter meta, skip link.
- Change `.section-head` divs to `<h2>`; make Jarvis/DriveHelp cards links.
- Move sprite inline `style="position:absolute"` to a CSS class; move aura swatch inline colors to classes.
- Absolutize `personalai/` and `aura/` footer links to `https://cuvane.com/…` (only after HTTPS is up).
- Update README URL to `https://cuvane.com`.
- Restructure repo so `Runbook/`, `Ventures/`, `README.md` are not under the Pages root (with a GitHub Pages source-directory change).

**4. What requires manual configuration**
- Porkbun DNS: add `185.199.109.153`, `185.199.110.153`, `185.199.111.153` A records for `@` (keep `185.199.108.153`).
- GitHub Pages: verify `cuvane.com` custom domain shows "verified"; if it stays unverified, re-save it in Settings → Pages. Confirm Pages source directory change if the repo is restructured.
- HTTPS: automatic once DNS is correct; no manual cert install.

**5. Exact files involved**
- `/Users/home/Cuvane/index.html` (root), `/Users/home/Cuvane/aura/index.html`, `/Users/home/Cuvane/personalai/index.html`
- `/Users/home/Cuvane/CNAME`, `/Users/home/Cuvane/.nojekyll`
- `/Users/home/Cuvane/README.md`, `/Users/home/Cuvane/Runbook/*` (4 formats), `/Users/home/Cuvane/Ventures/*/.gitkeep`
- New files to create: `robots.txt`, `sitemap.xml` at repo root
- Repo: `MikePFlanagan/cuvane` (PUBLIC, branch `main`, Pages source `main /`)

**6. Exact URLs involved**
- `https://cuvane.com/` (BROKEN → fix), `https://www.cuvane.com/` (BROKEN → fix), `http://cuvane.com/` (200, served directly), `http://www.cuvane.com/` (301→apex), `https://mikepflanagan.github.io/cuvane/` (301→apex)
- `http://cuvane.com/aura/`, `http://cuvane.com/personalai/` (both 200)
- Exposed: `http://cuvane.com/Runbook/Cuvane-Launch-Runbook.md`, `…/.html`, `…/.pdf`, `…/.docx`, `http://cuvane.com/README.md`
- External link on homepage: `https://mikepflanagan.github.io/dealdiamondcrm-site/` (200)

**7. Recommended repair sequence**
1. Verify `www` CNAME → `mikepflanagan.github.io` (done during audit; re-confirm).
2. Add the 3 missing Porkbun A records.
3. Wait ≤24 h for GitHub to issue the Let's Encrypt cert; then confirm both `https://` hosts work and HTTP redirects to HTTPS.
4. Make the code fixes (robots, sitemap, canonical, OG, headings, skip link, links, CSS cleanup, README) in one commit.
5. Move `Runbook/` + `Ventures/` out of the Pages root and re-point the Pages source directory; verify the site still serves.
6. Re-run crawl + html-validate.

**8. Commands to ask the user to run**
- `dig +short CNAME www.cuvane.com` (expect `mikepflanagan.github.io.`)
- `dig +short A cuvane.com` (expect 4 × 185.199.10X.153)
- `curl -sI https://cuvane.com/` and `curl -sI https://www.cuvane.com/` (expect 200, not SSL error)
- `curl -sI http://cuvane.com/` (expect 301 → https once enforced)
- `npx html-validate@latest /Users/home/Cuvane/index.html /Users/home/Cuvane/aura/index.html /Users/home/Cuvane/personalai/index.html`
- `git -C /Users/home/Cuvane status` (should be clean)

**9. Screenshots to ask for**
- GitHub → repo → Settings → Pages: the custom-domain box should show `cuvane.com` with a green "verified" badge (or "DNS check in progress").
- If the cert doesn't appear within 24 h: the DNS records page at Porkbun for `cuvane.com` showing the A records and the `www` CNAME record.

**10. Dangerous / uncertain — do NOT change yet**
- Do **not** delete or move `CNAME`/`.nojekyll` — Pages depends on them.
- Do **not** repoint the footer/card relative links to `https://cuvane.com` until HTTPS is confirmed working (otherwise the site would break on subpages served over HTTPS).
- Do **not** change the Pages source directory before backing up the plan — a wrong path takes the whole site offline; verify `CNAME`+`.nojekyll`+`index.html` are present in the new path.
- Do **not** edit DNS MX/SPF records — they are correct for Zoho.
- Do **not** add any custom security headers expecting them to work — GitHub Pages does not support custom headers; HSTS comes automatically with enforced HTTPS.
