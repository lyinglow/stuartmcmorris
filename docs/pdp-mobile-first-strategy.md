# DriverReviews.de — Mobile-First PDP Strategy (Tyres)

Reference implementation: [`prototype/pdp-tyre-product.html`](../prototype/pdp-tyre-product.html) — a standalone,
real HTML page (Continental AllSeasonContact 2, 205/55 R16) that every section below maps to 1:1. Open it directly
in a browser at a 375px viewport to test the mobile flow, and ≥1024px for the desktop layout.

Primary market: **Germany** (`driverreviews.de`, German copy, EUR, DE tyre-law FAQ content). Structure is built to
extend to AT/CH/UK via `hreflang` without a template rewrite — see [§5 Multi-market notes](#5-multi-market-notes).

---

## 1. Content hierarchy flow

The mobile flow is the source of truth. Desktop reuses the *same DOM order* and repositions visually with CSS
Grid — this matters for GEO (see §3): crawlers read source order, not visual order.

### Mobile (375px), top to bottom

| Zone | What | Why here |
|---|---|---|
| **A — Identity**<br>*above the fold* | Brand chip + hero image · H1 product name · size/season subtitle · star rating + review count (linked to reviews) · award badges | Confirms the user landed on the right product in under 2 seconds. Rating + review count is visible immediately — it's the trust signal that keeps a review-driven shopper scrolling instead of hitting back. |
| **B — Purchase decision**<br>*first scroll, ~1 screen down* | AI summary paragraph · key specs (size, season, noise, wet grip) · price · size selector · Add to Cart | Everything required to decide and buy, grouped together. No content that isn't decision-relevant is allowed in this zone. |
| **C — Deepening**<br>*second scroll* | Key themes (3–5 cards) · awards/certifications · video (product test + car test) · sibling season variants | For the user who wants *why*, not just *what*. Themes are the expanded, evidenced version of the one-line AI summary. |
| **D — Trust & detail**<br>*deep scroll, opt-in* | Full review list (paginated) · FAQ accordion · brand info · other models from this brand · related products (other brands) · footer | Long-tail content and cross-sell. Highest word count, lowest per-user attention — but *not* lowest crawl value (see §3), so it still ships as real HTML, not lazy-injected fragments. |

**Rule that governs the whole order:** every zone answers one question before the next zone is allowed to ask a
new one — *"what is this" → "should I buy it" → "why should I believe that" → "prove it / what else."* Nothing
from zone C or D is allowed to appear before zone B is complete, including on desktop.

### Desktop (≥1024px)

Two-column, same source order:

```
┌─────────────────────────────┬───────────────────────────┐
│                               │  H1 + rating + badges      │  Zone A/B, right column,
│      Image gallery           │  AI summary                │  sticky within the
│      (sticky-height,          │  Key specs                 │  viewport as the user
│       left column)            │  Price + size + Add to Cart│  scrolls the gallery
│                               │  (sticky buy box)          │
├───────────────────────────────┴───────────────────────────┤
│  Key themes (3-col grid) · Awards · Video · Sibling variants│  Zone C, full width
├──────────────────────────────────────────────────────────┤
│  Reviews · FAQ · Brand info · Other models · Related        │  Zone D, full width
└──────────────────────────────────────────────────────────┘
```

The buy box becomes `position: sticky` in the right column instead of a fixed bottom bar — desktop has the
horizontal room to keep it permanently visible without occluding content, so the mobile sticky-CTA *pattern* is
replaced by a different mechanism that serves the same job, not stacked on top of it.

---

## 2. UX pattern recommendations

| Pattern | Used for | Not used for | Reasoning |
|---|---|---|---|
| **Always-visible block** (no interaction to reveal) | AI summary, key themes, key specs, star rating, price | — | These are the E-E-A-T/USP content. Anything gated behind a click is a coin-flip on whether a GenAI crawler that doesn't execute JS ever sees it. It also loses human users who bounce before tapping. |
| **Native `<details>/<summary>` accordion** | FAQ, "all technical specs" (beyond the 4 key specs), individual long review bodies if truncated | AI summary, key themes | `<details>` content is *in the DOM* whether open or closed — Googlebot indexes it, and most GenAI crawlers that parse static HTML (they largely don't execute JS) see it too. This gets you the mobile-density benefit of an accordion without the GEO cost of `display:none`-via-JS. |
| **Tabs implemented as real links** (`role` styling, `<a href>` targets) | Season-variant switcher (Summer / Winter / All-Season) | — | These "variants" are different SKUs with different specs and reviews, so they must be different crawlable, indexable URLs. A JS panel-swap would silently merge three products' worth of AI-crawler visibility into one URL and orphan the other two. Visually it still reads as a tab strip. |
| **Radio-pill group** (native `<input type="radio">` + `<label>`) | Size selector within the buy box | — | Size only changes price/stock, not page content — a true in-page state change, not a content-hiding problem. Native radios stay keyboard- and screen-reader-accessible for free and degrade to a working form without JS. |
| **Progressive disclosure via pagination/"show more"** | Review list (show 3–5 server-rendered, "show all 3,842" loads/links to the rest) | AI summary, themes | Reviews are unbounded — you cannot ship 3,842 reviews in the initial payload. Server-render a real, representative sample (not the shortest/newest — pick ones that cover the theme spread) so both crawlers and users see substantive content even if they never click "show more." |
| **Sticky bottom CTA bar (mobile only)** | Add to Cart + price, triggered by `IntersectionObserver` once the primary buy button scrolls off-screen | — | Reviews pages get long; without this the buy action can be 10+ screens away. `IntersectionObserver` toggling a `transform: translateY()` is the cheap, non-layout-shifting way to do it — CSS-only fallback (`position: sticky` inside a wrapper) if you want zero-JS resilience. |
| **Lazy loading** (`loading="lazy"`) | Every image/video below the hero | Hero image (`loading="eager" fetchpriority="high"` — it's the LCP element) | Standard native lazy-loading needs no JS and doesn't affect crawlability — the `src`/`alt` are still in the HTML regardless of when the browser fetches the bytes. |

**Anti-pattern to explicitly avoid:** collapsing the AI summary or key themes into an accordion "to save space."
This is the single most common mistake teams make porting a desktop PDP to mobile, and it is exactly the content
this project is trying to make *more* visible, not less.

---

## 3. GEO-friendly schema and HTML structure

### Principles

1. **Server-rendered, not client-injected.** GPTBot, PerplexityBot, ClaudeBot and most GenAI crawlers do not
   execute JavaScript at crawl time. If the AI summary, specs, price, or FAQ answers only exist after a client-side
   fetch, they are invisible to these agents even though Googlebot (which does render JS, on a delay) might still
   see them. Render everything in §1's zones A–D in the initial HTML response.
2. **Source order = priority order.** Desktop's CSS Grid repositions visually but never reorders the DOM (see the
   prototype's `.pdp-grid` — `grid-column`/`grid-row` placement, never `order`). A crawler reading raw HTML gets
   the same priority sequence a mobile user scrolling gets.
3. **Visible text and schema must match.** `FAQPage` answers must be the literal, visible answer text, not a
   paraphrase — Google explicitly discounts FAQ rich results where they don't match, and an LLM cross-checking
   your JSON-LD against your visible copy for trustworthiness will do the same, informally.
4. **No content behind interaction for anything you want cited.** If a fact should be quotable by an AI answer
   engine ("DriverReviews.de says the noise rating is 71dB"), it cannot live only inside a collapsed accordion or
   an image (e.g. a spec sheet as a JPEG). Key specs are marked up as a real `<dl>`, not a rasterized image.

### Schema stack (all present in the prototype's `<head>`)

- **`Product`** — `name`, `brand`, `sku`, `gtin13`, `image[]`, `description`, `category`, and `additionalProperty[]`
  for the tyre-specific attributes (size, season, EU label noise/wet-grip/fuel-efficiency, 3PMSF).
- **`Offer`** (nested in `Product.offers`) — `price`, `priceCurrency`, `availability`, `priceValidUntil`,
  `shippingDetails`. For pages with several sizes at different prices, upgrade this to `ProductGroup` +
  `isVariantOf` with one `Product`/`Offer` per SKU rather than a single averaged price — more accurate and
  Google's preferred pattern for size/variant matrices.
- **`AggregateRating`** — `ratingValue`, `bestRating`, `reviewCount`. This is what powers the star+count shown in
  Zone A and is the fastest-parsed trust signal for both search rich results and AI answer engines.
- **`Review[]`** — a handful of representative reviews inline (matches what's rendered in Zone D), not all 3,842.
- **`FAQPage`** — one `Question`/`acceptedAnswer` pair per visible FAQ accordion item, text identical to the
  rendered `<summary>`/`<p>`.
- **`BreadcrumbList`** — supports both search breadcrumbs and gives crawlers an explicit category hierarchy
  (Reifen → Ganzjahresreifen → Continental → model) that reinforces topical authority signals.
- **Not included on this page, recommended at site level:** `Organization` schema for Continental as a brand
  entity (on `/reifen/continental/`), and `WebSite` + `SearchAction` on the homepage.

There is no official schema.org type for "AI-generated summary" — don't force one. Ship it as a plainly-labelled,
semantically-marked `<section aria-label="KI-Zusammenfassung der Bewertungen">` with a visible "based on N
reviews / last updated [date]" byline. That byline is doing real GEO work: it's an explicit
recency-and-provenance signal, which is what LLM answer engines weight when deciding whether to trust and cite a
number.

### HTML structural rules used throughout

- Semantic landmarks: `<header>`, `<nav aria-label="Breadcrumb">`, `<main>`, `<section aria-label="…">`,
  `<footer>` — every content block is a labelled section, not a `<div>` soup.
- One `<h1>` (product name), `<h2>` per major zone-C/D section, so both crawlers and screen readers get a real
  outline.
- Specs as `<dl>`/`<dt>`/`<dd>`, not a table screenshot or a comma-separated string in a `<div>`.
- `<time>`/plain ISO-ish dates for review and summary freshness, not relative strings like "2 weeks ago" that mean
  nothing to a crawler snapshotting the page once.
- `hreflang` + `canonical` in `<head>` for the multi-market variants (see §5).

---

## 4. Wireframe: above-fold vs. scroll zones

The prototype has a **"Content-Zonen anzeigen" toggle** at the very top of the page — check it to overlay a small
zone label (A/B/C/D) on every block, live, on the real markup. Below is the same map as a static reference.

### Mobile — 375×667 (iPhone SE, the tightest common baseline)

```
┌───────────────────────────────┐  ← viewport top
│ ≡  DriverReviews.de            │  sticky header, 44px
├───────────────────────────────┤
│ [Continental]                  │
│ ┌───────────────────────────┐ │
│ │                           │ │
│ │      HERO IMAGE            │ │  ZONE A
│ │      (4:3, eager-loaded)    │ │  above the fold
│ │                           │ │  on a 667px-tall
│ └───────────────────────────┘ │  viewport
│ ○ ○ ○ ○  (thumbnails)          │
│ Continental AllSeasonContact 2 │
│ Ganzjahresreifen · 205/55 R16  │
│ ★★★★★ 4,6 · 3.842 Bewertungen  │
╌╌╌╌╌╌╌╌╌╌╌╌ fold @ ~667px ╌╌╌╌╌╌  ← scroll starts here
│ 🏆 ADAC Empfehlenswert          │
│ ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━┓ │
│ ┃ ✦ KI-ZUSAMMENFASSUNG        ┃ │
│ ┃ "Der Continental Allseason  ┃ │  ZONE B
│ ┃ Contact 2 überzeugt..."     ┃ │  purchase decision
│ ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━┛ │  (1 scroll)
│ Größe 205/55R16  Saison AJ     │
│ Geräusch 71dB    Nasshaft. B   │
│ 94,90 €  · Auf Lager            │
│ [ In den Warenkorb ]            │
├───────────────────────────────┤
│ ZENTRALE THEMEN                │
│ ┌─────┐┌─────┐┌─────┐          │  ZONE C
│ │Nass-││Ruhe ││Preis│  ...     │  deepening
│ └─────┘└─────┘└─────┘          │  (2nd–3rd scroll)
│ Auszeichnungen · Video          │
│ Andere Ausführungen (Sommer/…)  │
├───────────────────────────────┤
│ BEWERTUNGEN (3.842)             │
│ [3 full reviews] [Alle anzeigen]│  ZONE D
│ HÄUFIGE FRAGEN (accordion)      │  trust & detail
│ Über Continental                 │  (opt-in scroll)
│ Weitere Modelle / Andere kaufen  │
└───────────────────────────────┘
  [ 94,90 €      In den Warenkorb ]  ← sticky bar, appears
                                       once buy box scrolls
                                       past top of viewport
```

### Desktop — ≥1024px, above the fold at ~900px tall

```
┌────────────────────────────────────────────────────────────────┐
│  DriverReviews.de                                                │
├────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────┐  Continental AllSeasonContact 2         │
│ │                        │  ★★★★★ 4,6 · 3.842 Bewertungen          │  ZONE A/B
│ │      HERO IMAGE         │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │  both visible
│ │      (sticky gallery)   │  ┃ ✦ KI-ZUSAMMENFASSUNG            ┃    │  without scrolling
│ │                        │  ┃ "...überzeugt vor allem..."      ┃    │  — the wide
│ └──────────────────────┘  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │  viewport buys
│  ○ ○ ○ ○                    Größe · Saison · Geräusch · Nasshaft.   │  back the room
│                              94,90 €  [In den Warenkorb] (sticky)   │  mobile can't
╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌ fold @ ~900px ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
│  ZENTRALE THEMEN (3-column)         Auszeichnungen · Video          │  ZONE C
├────────────────────────────────────────────────────────────────┤
│  Bewertungen · FAQ · Über Continental · Weitere Modelle · Related  │  ZONE D
└────────────────────────────────────────────────────────────────┘
```

The desktop fold is generous enough to fit all of Zone A *and* B — this is the payoff of the two-column layout,
and it's exactly why the brief's instruction to design mobile-first still matters: mobile has to make the A/B
split work in sequence, and desktop just gets to show the result of that discipline side-by-side.

---

## 5. Multi-market notes

- **Priority: Germany.** All copy, the FAQ content (situational winter-tyre law, `15.1`/`15.2` Fahrzeugschein
  reference), currency (EUR), and date formatting in the prototype are DE-specific by design — this is not a
  literal-translation template.
- **Structural reuse for AT/CH/UK:** the DOM, schema shape, and zone hierarchy do not change per market — only
  copy, currency, and jurisdiction-specific FAQ content (e.g. UK has no legal winter-tyre requirement, so that FAQ
  answer changes, not the FAQ slot). `hreflang` alternates + a market-specific `canonical` per locale prevent
  duplicate-content dilution across `driverreviews.de` / `.at` / `.ch` / `.co.uk`.
- **Currency and units are locale tokens, not hardcoded strings** in the real implementation — the prototype
  hardcodes `94,90 €` and metric units because it's DE-only, but the production template should source these from
  the market config so a UK build renders `£81.10` / mph-equivalent load index the same DOM otherwise.

---

## 6. What NOT to do (common failure modes this design avoids)

- Hiding the AI summary or key themes behind a "Read more" or accordion to save mobile vertical space — defeats
  the entire GEO purpose of having them.
- Rendering price, specs, or FAQ answers via client-side JS/hydration only — invisible to non-JS-executing GenAI
  crawlers.
- Using `display:none` + JS toggle for tab panels that contain unique per-variant content — orphans that content
  from indexing. (Native `<details>` is fine because it's DOM-present either way; `display:none`-until-JS-runs is
  not.)
- Infinite-scroll-only reviews with no server-rendered initial batch — a crawler that doesn't scroll/paginate sees
  zero reviews.
- A single blended `Product`/`Offer` price when the page actually sells several sizes at several prices — file
  under `ProductGroup` instead once the catalog needs it.
