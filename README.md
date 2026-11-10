# gigspaid.com Lighthouse audit (mobile)

Audited 2026-11-10 with Lighthouse 12.8.2, Chrome 157 (system Chrome, headless), default mobile emulation and default simulated throttling (RTT 150 ms, 1,638 Kbps, 4x CPU slowdown). All measured numbers below come from the JSON reports in `reports/`.

## Scores (median-performance run of 3)

| | Home `https://gigspaid.com/` | Work `https://gigspaid.com/#/work` |
|---|---|---|
| Performance | **77** | **71** |
| Accessibility | **88** | **91** |
| Best Practices | 96 | 96 |
| SEO | 100 | 100 |
| FCP | 4.0 s | 3.5 s |
| LCP | 4.0 s | 3.5 s |
| TBT | 0 ms | 0 ms |
| CLS | 0.002 | 0.219 |
| Speed Index | 4.5 s | 4.4 s |

Reports: `reports/home.report.html|json` (run 2 of 3) and `reports/work.report.html|json` (run 3 of 3). All six raw runs are in `reports/all-runs/`.

Spread over 3 runs (performance): home 54 / 77 / 78 (the 54 run had TBT 690 ms and LCP 4.4 s; the other two TBT 0 ms), work 73 / 68 / 71. Accessibility, Best Practices and SEO were identical across runs. Every run carried the Lighthouse warning "The page loaded too slowly to finish within the time limit. Results may be incomplete", so treat performance numbers as approximate. Runs were made from a Windows desktop on a normal connection against the live site (observed HTML server response 792 ms home / 721 ms work from this location).

## How it was run

```
export CHROME_PATH="/c/Program Files/Google/Chrome/Application/chrome.exe"
npx lighthouse "<URL>" --form-factor=mobile --only-categories=performance,accessibility,best-practices,seo \
  --output=html --output=json --output-path=<name>-<1..3> --chrome-flags="--headless=new --no-sandbox" --quiet
```
Run 3 times each for `https://gigspaid.com` and `https://gigspaid.com/#/work` (loaded as a full page load of that URL); the median performance run is kept. Lighthouse installed locally with `npm i -D lighthouse`.

## The 5 changes with the biggest impact

Gain estimates start from Lighthouse's own savings figures. Score deltas were computed by re-applying Lighthouse's log-normal scoring curve to the reported FCP/LCP and the audit weights (the curve reproduced the reported FCP/LCP scores to within 0.01). They are estimates, not measurements, and fixes 1 to 3 overlap, so do not add them.

### 1. Stop the Google Fonts stylesheet (and registerSW.js) from blocking first paint
- Where: `index.html`, `<link href="https://fonts.googleapis.com/css2?family=Archivo:wdth,wght@62..125,400..900&family=JetBrains+Mono:wght@400;700&display=swap" rel="stylesheet">`, and `<script id="vite-plugin-pwa:register-sw" src="./registerSW.js">` in `<head>`. A second Fonts CSS request (`...family=Archivo:wght@100..900&family=JetBrains+Mono:wght@400;500;700`) is fired later, after the lazy chunk `assets/Landing-BN31nLoo.js` loads.
- Evidence: `render-blocking-insight` lists the fonts CSS (1,246 B, wasted 847 ms home / 842 ms work), `registerSW.js` and `assets/index-BTAc3Lop.css`; Lighthouse's estimated savings are 1,860 ms (home) / 1,750 ms (work). The woff2 files are fetched only after that CSS (Archivo 90 KB + 86 KB, JetBrains 32 KB on home). FCP is 4.0 s / 3.5 s (FCP scores 0.23 / 0.33).
- Fix: self-host the three woff2 files under `/fonts/`, inline the `@font-face` rules in `assets/index-*.css` with `font-display: swap`, add `<link rel="preload" as="font" type="font/woff2" crossorigin>` for the Archivo file, delete the Google link and the two preconnects, and add `defer` to `registerSW.js` (or register on `load`). Use one font stylesheet instead of two.
- Expected gain: up to about 1.8 s off FCP and LCP. By the curve: home FCP 0.23 to about 0.8 and LCP 0.49 to about 0.95, roughly +15 to +17 Performance points on both pages. This is an upper bound; the real saving depends on how much of that 1.8 s the JS chain in fix 2 still occupies.

### 2. Remove the JS-then-data waterfall that delays the LCP element
- Where: LCP element is `section#top > div > div > p.gs-a` (inline `animation-delay: 0.38s`) on home, and `div.page-head > div > h1` ("Paid work. First claim wins.") on work. Files: `assets/index-BoboNSB7.js` (62 KB transfer), lazy chunk `assets/Landing-BN31nLoo.js` (15 KB), `GET /api/landing`, `GET /api/work?kind=job` (27 KB).
- Evidence: LCP breakdown shows 0 ms load delay and load time but render delay 2,842 ms (71%) on home and 2,373 ms (67%) on work; the served HTML contains only a spinner `<img src="logo.svg">`. Home timeline: index JS done at about 1,061 ms, `Landing-BN31nLoo.js` not requested until about 1,615 ms, `/api/landing` runs 1,674 to 2,426 ms. Work: `/api/work?kind=job` requested at 1,375 ms, finished at 2,725 ms. `forced-reflow-insight` attributes 229 ms of reflow to `Landing-BN31nLoo.js` on home.
- Fix: add `<link rel="modulepreload" href="./assets/Landing-BN31nLoo.js">` for the home route (or import Landing statically in the entry), preload the data (`<link rel="preload" as="fetch" crossorigin href="/api/landing">`, and the same for `/api/work?kind=job` when the hash is `#/work`), and make sure `p.gs-a` and the work `h1` are not hidden by an entrance animation (drop the 0.38 s delay and the scramble/decode effect on the LCP text, or render the final text and animate afterwards).
- Expected gain: Lighthouse gives no single savings figure for this; the ceiling is the 2.4 to 2.8 s render delay. Shaving about 1 s off LCP is plausible, worth several Performance points on both pages (LCP is 25% of the score). Estimate only.

### 3. Make the HTML cacheable and cut TTFB (`Cache-Control: no-store` on `/`)
- Where: response headers of `https://gigspaid.com/` (and `registerSW.js`, also `no-store`). The hashed `assets/index-BoboNSB7.js` and `assets/index-BTAc3Lop.css` already send `public, max-age=31536000, immutable` and Brotli, so they are fine.
- Evidence: `document-latency-insight` says "Server responded slowly (observed 792 ms home / 721 ms work)", estimated savings 690 ms / 620 ms. TTFB in the LCP breakdown is 1,171 ms / 1,164 ms (29% / 33% of LCP). My curl got `X-Vercel-Cache: MISS`. The `bf-cache` audit fails with `MainResourceHasCacheControlNoStore`.
- Fix: in the host config (Vercel headers / `vercel.json`) set `Cache-Control: public, max-age=0, must-revalidate` on `/index.html` (it already sends an ETag) so the edge can serve it, and stop sending `no-store` on `registerSW.js`.
- Expected gain: about 0.6 to 0.7 s off FCP/LCP per Lighthouse; by the curve roughly +6 to +7 Performance points; also lets the page enter the back/forward cache.

### 4. Fix CLS 0.219 on `/#/work` by reserving space for the job cards
- Where: `main#main div.grid-cards` (89 `a.card` children; 2,873 DOM elements on the page), filled after `GET /api/work?kind=job` resolves.
- Evidence: CLS audit score 0.57 (value 0.219, 8 layout shifts of 0.027 each; home is 0.002). CLS carries 25% of the Performance score.
- Fix: render fixed-height skeleton `.card` placeholders (same height as a real card) and give `.grid-cards` a `min-height`; render only the first dozen or so cards and page or virtualize the rest (this also addresses the `dom-size` warning of 2,873 elements). Caveat: this report does not name the shifting node, so confirm it in the DevTools Performance panel; linking the shifts to the late API response is an inference.
- Expected gain: getting CLS under 0.1 moves the audit from 0.57 to 1.0, i.e. 0.25 x 0.43 = about +10.8 Performance points on `/#/work` (none on home).

### 5. Fix the three failing accessibility audits (88 to about 100 on home, 91 to about 100 on work)
Failing audits and weights: `aria-prohibited-attr` 7, `color-contrast` 7, `heading-order` 3 (17 weighted points of 145 on home, of 193 on work).
- `aria-label` on non-semantic spans (7 nodes on home, 1 on work): `h1 > span.gs-l > span > span.fx-slot[aria-label="wins."]`, `h2.gs-u > span.fx-decode`, `span.fx-mag`, `span.fx-grav` (x2), `span.fx-type`; on work `div.page-head > h1 > span.scramble[aria-label="Paid work. First claim wins."]`. Fix: remove `aria-label` from those spans and put it on the heading itself, or keep the real text in a visually hidden span and mark the animated copy `aria-hidden="true"`.
- Color contrast: `div.gs-band div.gs-marq span.gs-word` is `#2f6448` on `#0c2a1b`, ratio 2.22 (needs 3:1 for large text; 3 nodes on home). Change to `#4a8a66` (3.76:1 by my own calculation). `footer.site-foot div.sf-base span.m` is `#6f8f7b` on `#0c2a1b`, ratio 4.32 (needs 4.5:1; both pages). Change to `#7a9a86` (4.99:1 by my own calculation). Replacement ratios are my calculation, not Lighthouse output.
- Heading order: `footer.site-foot nav.sf-cols > div > h4` skips a level. Change the `<h4>` to the level that follows the previous heading, or to a styled `<p>`.
- Expected gain: Accessibility to about 100 on both pages (+12 home, +9 work), assuming no new failures appear.

## Considered but not in the top 5
- `unused-javascript`: 29 KiB wasted in `assets/index-BoboNSB7.js` (about 49 to 50%). Small impact since TBT is already 0 ms in the median runs.
- Best Practices 96: `errors-in-console` for a 401 from `https://gigspaid.com/api/me` on an anonymous visit. Returning 200 with an empty user would clear it.
