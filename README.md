# gigspaid.com Lighthouse audit (mobile)

Lighthouse (mobile) reports for **https://gigspaid.com/** and **https://gigspaid.com/#/work**, plus the five changes that would raise the Performance and Accessibility scores most, each tied to an exact element or file.

## TL;DR

- **Today:** Performance **77** (home) / **71** (work). Accessibility **88** / **91**. SEO 100 on both.
- **The five fixes**, ranked by expected gain (details, evidence and code snippets below):
  1. Remove the render-blocking Google Fonts stylesheet and `registerSW.js` from `index.html`.
  2. Fix the three failing accessibility audits: `aria-label` on `span`s, two colour-contrast pairs, and a skipped heading level.
  3. Stop the job grid shifting the layout on `/#/work` (CLS 0.219).
  4. Shorten the JS-then-API request chain in front of the LCP text.
  5. Allow the HTML document to be cached (`Cache-Control: no-store` on `/`).
- **Caveat:** every run carried Lighthouse's "page loaded too slowly" warning, so Performance is good to about ±5 points, and the gains are estimates from Lighthouse's scoring curve, not measurements.

**Contents:** [Summary](#summary) · [The 5 changes](#the-5-changes-ranked-by-expected-score-gain) · [Considered but not in the top 5](#considered-but-not-in-the-top-5) · [How to reproduce](#how-to-reproduce)

**Files**

```text
reports/home.report.html|json     median of 3 runs, https://gigspaid.com/
reports/work.report.html|json     median of 3 runs, https://gigspaid.com/#/work
reports/all-runs/                 all six raw runs (home-1..3, work-1..3)
package.json                      npm run audit | audit:home | audit:work
```

## Method

Lighthouse 12.8.2, Chrome 157 (headless), default mobile emulation and default simulated throttling (RTT 150 ms, 1,638 Kbps, 4x CPU slowdown), run on 2026-11-10 against the live site. Every measured number below comes from the JSON reports in `reports/`, except where a line says "my curl" or "my calculation".

## Summary

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

Reports (open the `.html` files in a browser):
- Home: `reports/home.report.html` and `reports/home.report.json` (run 2 of 3, the median)
- Work: `reports/work.report.html` and `reports/work.report.json` (run 3 of 3, the median)
- All six raw runs: `reports/all-runs/`

Performance across the 3 runs: home 54 / 77 / 78, work 73 / 68 / 71. The 54 run had TBT 690 ms and also carried Lighthouse's "device has a slower CPU than expected" warning, so it is an outlier. Accessibility, Best Practices and SEO were identical in all runs. Every run carried "The page loaded too slowly to finish within the time limit. Results may be incomplete", so treat performance numbers as approximate (about +/- 5 points).

## The 5 changes, ranked by expected score gain

| # | Change | Exact element / file | Expected gain (estimate) |
|---|---|---|---|
| 1 | Remove render-blocking Google Fonts CSS and `registerSW.js` | `index.html`: the `fonts.googleapis.com` `<link>`, `<script id="vite-plugin-pwa:register-sw" src="./registerSW.js">` | Performance +14 to +17 (home 77 to ~94, work 71 to ~85), upper bound |
| 2 | Fix the 3 failing accessibility audits | `aria-label` on `span.fx-*` / `span.scramble`; `span.gs-word` and `span.m` colours; footer `nav.sf-cols h4` | Accessibility 88 to 100 (home), 91 to 100 (work) |
| 3 | Stop the job list shifting the layout | `main#main div.grid-cards` (work page) | Performance +8 to +10 on work only (71 to ~79-81) |
| 4 | Shorten the JS-then-API chain before the LCP text appears | `assets/Landing-BN31nLoo.js`, `GET /api/landing`, `GET /api/work?kind=job`, `p.gs-a`, work `h1` | Performance about +10 home, +6 work (Lighthouse's own LCP figure) |
| 5 | Let the HTML be cached (it sends `Cache-Control: no-store`) | Response headers of `/` | Performance +6 on both pages |

How the gains were computed: Lighthouse's savings (ms) were subtracted from the measured FCP/LCP and re-scored with Lighthouse's own scoring function (`computeLogNormalScore` from the installed package, mobile p10/median constants, weights FCP 10 / SI 10 / LCP 25 / TBT 30 / CLS 25). The baseline reproduces the reported scores exactly (77 and 71). These are estimates, not measurements.

Overlap: fixes 1, 4 and 5 all shorten the same FCP/LCP path, so their gains are not additive. Fix 3 (CLS) and fix 2 (accessibility) are independent of them and of each other. If 1 and 5 were fully additive, Lighthouse's model would give about 97 (home) and 87 (work); plan on less.

### 1. Remove the render-blocking font stylesheet and `registerSW.js`

- Evidence: `render-blocking-insight` / `render-blocking-resources` lists the Google Fonts CSS (1,246 B, wasted 847 ms home / 842 ms work), `registerSW.js` and `assets/index-BTAc3Lop.css`, with estimated FCP and LCP savings of 1,850 ms (home) and 1,750 ms (work). The woff2 files are only requested after that CSS arrives (home: Archivo 90 KB + 86 KB, JetBrains Mono 32 KB). A second Fonts CSS request (`Archivo:wght@100..900&family=JetBrains+Mono:wght@400;500;700`) starts after the lazy `assets/Landing-BN31nLoo.js` loads, so there are two font stylesheets.
- Fix, step 1: self-host the two latin woff2 files (variable fonts, verified file names in the npm packages):
  ```
  npm i -D @fontsource-variable/archivo @fontsource-variable/jetbrains-mono
  mkdir -p public/fonts
  cp node_modules/@fontsource-variable/archivo/files/archivo-latin-wdth-normal.woff2 public/fonts/
  cp node_modules/@fontsource-variable/jetbrains-mono/files/jetbrains-mono-latin-wght-normal.woff2 public/fonts/
  ```
- Step 2, `index.html`: delete the two `preconnect` links and the `fonts.googleapis.com` stylesheet link, and add:
  ```html
  <link rel="preload" href="/fonts/archivo-latin-wdth-normal.woff2" as="font" type="font/woff2" crossorigin>
  ```
- Step 3, top of the main stylesheet (it ends up in `assets/index-*.css`), keeping the existing family names so no other CSS changes:
  ```css
  @font-face{font-family:'Archivo';font-style:normal;font-weight:100 900;font-stretch:62% 125%;font-display:swap;src:url(/fonts/archivo-latin-wdth-normal.woff2) format('woff2')}
  @font-face{font-family:'JetBrains Mono';font-style:normal;font-weight:100 800;font-display:swap;src:url(/fonts/jetbrains-mono-latin-wght-normal.woff2) format('woff2')}
  ```
  Also remove whatever code in the Landing chunk injects the second Google Fonts stylesheet (search the source for `fonts.googleapis.com`). Latin only: non-latin characters fall back to a system font.
- Step 4, `vite.config.js`: `VitePWA({ injectRegister: 'script-defer', /* keep the rest */ })` so `registerSW.js` is deferred.
- Expected gain: up to 1.85 s / 1.75 s off FCP and LCP (FCP score 0.23 to ~0.79, LCP 0.49 to ~0.95 on home), Performance +17 home, +14 work. Upper bound: the LCP text also waits for the JS and API chain in fix 4.

### 2. Fix the three failing accessibility audits (88 to 100 home, 91 to 100 work)

Failing audits and weights: `aria-prohibited-attr` 7, `color-contrast` 7, `heading-order` 3. That is 17 weighted points of 145 on home (score 0.88) and of 193 on work (0.91).

- `aria-prohibited-attr`: `aria-label` is not allowed on a plain `span`. Home: `h1 > span.gs-l > span > span.fx-slot[aria-label="wins."]`, `h2.gs-u > span.fx-decode`, `span.fx-mag`, `span.fx-grav` (2 nodes), `span.fx-type`, `span.fx-melt` (7 nodes). Work: `div.page-head > h1 > span.scramble[aria-label="Paid work. First claim wins."]`. Move the text out of `aria-label` into real text for screen readers and hide the animated copy:
  ```html
  <h1>
    <span class="scramble" aria-hidden="true">Paid work. First claim wins.</span>
    <span class="sr-only">Paid work. First claim wins.</span>
  </h1>
  ```
  ```css
  .sr-only{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0 0 0 0);white-space:nowrap}
  ```
  Same pattern for each `span.fx-*` listed above.
- `color-contrast` (background `#0c2a1b` in both cases):
  - `div.gs-band div.gs-marq span.gs-word` (3 nodes, home only, 30px bold, needs 3:1): `#2f6448` measured 2.22 by Lighthouse. Change to `#4a8a66`.
  - `footer.site-foot div.sf-base span.m` (both pages, 13px, needs 4.5:1): `#6f8f7b` measured 4.32 by Lighthouse. Change to `#7a9a86`.
  - Replacement ratios, my calculation with the WCAG 2.x relative-luminance formula (sRGB linearised, ratio = (L1+0.05)/(L2+0.05)) against `#0c2a1b`: `#4a8a66` = 3.76:1 (passes 3:1), `#7a9a86` = 4.99:1 (passes 4.5:1). The same formula gives 2.23 and 4.32 for the old colours, matching Lighthouse (2.22 / 4.32) to rounding. `#2f6448` is set as an inline style on the span, so change it where the JS sets it.
- `heading-order`: `footer.site-foot nav.sf-cols > div > h4` jumps more than one level below the preceding heading (Lighthouse reports "Heading order invalid" on both pages). Change the `<h4>` to `<p>` and keep its class, so the same markup is valid on both pages.
- Expected gain: Accessibility to 100 on both pages, assuming no new failures appear (+12 home, +9 work).

### 3. Stop the job list shifting the layout on `/#/work` (CLS 0.219)

- Where: `main#main div.grid-cards` (89 `a.card` children, about 295 px each: its height is 26,284 px; 2,873 DOM elements on the page, versus 869 on home).
- Evidence: `layout-shifts` reports 8 shifts of 0.027 each (total 0.219, CLS audit score 0.57); home is 0.002. The list comes from `GET /api/work?kind=job` (27 KB), finished at about 2,725 ms. Caveat: the report does not name the shifting nodes, so confirm in the DevTools Performance panel (Layout Shifts track). Linking the shifts to the late API response is an inference.
- Fix: render skeleton cards of the real card height while loading, and skip rendering off-screen cards:
  ```css
  .grid-cards{min-height:100vh}
  .grid-cards > .card{content-visibility:auto;contain-intrinsic-size:auto 295px}
  ```
  Rendering only the first 12 or so cards and paging the rest also fixes the 2,873-element `dom-size` warning.
- Expected gain: CLS under 0.1 gives an audit score of 0.9 (Performance 79); near 0.05 gives about 0.99 (Performance 81). So +8 to +10 on work, none on home.

### 4. Shorten the JS-then-API chain in front of the LCP text

- Where: LCP element is `section#top > div > div > p.gs-a` (inline `animation-delay: 0.38s`) on home and `div.page-head > div > h1` ("Paid work. First claim wins.") on work. Files: `assets/index-BoboNSB7.js` (62 KB transferred), lazy chunk `assets/Landing-BN31nLoo.js` (15 KB), `GET /api/landing`, `GET /api/work?kind=job`.
- Evidence: the LCP breakdown shows load delay 0 and load time 0, but render delay 2,842 ms (71% of LCP) on home and 2,373 ms (67%) on work; the served HTML is only a spinner `<img src="logo.svg">`. Home timeline (simulated): index JS finished at ~1,061 ms, `Landing-BN31nLoo.js` requested at ~1,615 ms, `/api/landing` 1,674 to 2,426 ms. Work: `/api/work?kind=job` requested at 1,375 ms, finished at 2,725 ms. `forced-reflow-insight` attributes 229 ms of forced reflow to `Landing-BN31nLoo.js` on home.
- Fix:
  - Home: import `Landing` statically in the entry instead of via `import('./.../Landing')`, so the chunk is not requested after the entry JS.
  - Start the data requests from `index.html`, before the JS runs:
    ```html
    <script>
    (function(){var u=location.hash.indexOf('#/work')===0?'/api/work?kind=job':(location.hash===''||location.hash==='#/'?'/api/landing':null);
    if(u){var l=document.createElement('link');l.rel='preload';l.as='fetch';l.href=u;l.crossOrigin='anonymous';document.head.appendChild(l)}})();
    </script>
    ```
    The app's own `fetch()` for the same URL must be a plain same-origin GET so the preload is reused.
  - Remove the inline `animation-delay: 0.38s` from `p.gs-a` and the scramble/decode effect from the work `h1`: render the final text at once and animate other elements.
- Expected gain: Lighthouse's `largest-contentful-paint-element` audit reports LCP savings of 1,500 ms (home) and 1,050 ms (work). By the scoring curve that is about +10 (home) and +6 (work). Overlaps with fix 1.

### 5. Allow the HTML to be cached (`Cache-Control: no-store` on `/`)

- Evidence: `document-latency-insight` flags the root document ("Server responded slowly, observed 792 ms home / 721 ms work") with estimated FCP/LCP savings of 700 ms / 600 ms. The same reports' `server-response-time` audit passes with 10 ms, so Lighthouse's own data is mixed; my curl measured time-to-first-byte of 0.56 to 0.68 s in 4 tries (TCP connect only 15 ms), which supports the slow-document finding. `curl -I` shows `Cache-Control: no-store` and `X-Vercel-Cache: MISS` on `/` and `registerSW.js`; `manifest.webmanifest`, `logo.svg` and the `/api/*` responses are `no-store` too. The hashed `assets/index-*.js` / `.css` are fine (`public, max-age=31536000, immutable`, gzip). `bf-cache` fails with `MainResourceHasCacheControlNoStore`.
- Fix: in `vercel.json` (or wherever the `no-store` is added; responses also carry `X-Railway-*` headers, so an origin server behind Vercel may be setting it, which I could not see):
  ```json
  {
    "headers": [
      { "source": "/", "headers": [{ "key": "Cache-Control", "value": "public, max-age=0, s-maxage=300, stale-while-revalidate=86400" }] },
      { "source": "/registerSW.js", "headers": [{ "key": "Cache-Control", "value": "public, max-age=0, must-revalidate" }] }
    ]
  }
  ```
  `s-maxage` lets the edge serve the HTML without going to the origin. Keep the previous deploy's hashed assets available for a few minutes, because cached HTML can point at the older bundle names.
- Expected gain: about 0.6 to 0.7 s off FCP/LCP per Lighthouse, Performance +6 on both pages. Back/forward cache will not be fixed by this alone: the second `bf-cache` failure (`JsNetworkRequestReceivedCacheControlNoStoreResource`) comes from the `no-store` API responses.

## Considered but not in the top 5
- `unused-javascript`: 29 KiB wasted in `assets/index-BoboNSB7.js` (50% on home, 48% on work). Lighthouse puts the FCP/LCP saving at 0 ms and TBT is already 0 ms.
- Best Practices 96: `errors-in-console` for a 401 from `https://gigspaid.com/api/me` on an anonymous visit. Returning 200 with an empty user would clear it. Outside the performance/accessibility brief.

## How to reproduce

Requires Node 18+ and Chrome installed.

```
npm install
npm run audit
```

`npm run audit` runs Lighthouse mobile once for each URL and overwrites `reports/home.report.{html,json}` and `reports/work.report.{html,json}`. Run `npm run audit:home` or `npm run audit:work` for one page. Each script is a plain `lighthouse` call:

```
npx lighthouse "https://gigspaid.com/#/work" --form-factor=mobile --only-categories=performance,accessibility,best-practices,seo --output=html --output=json --output-path=reports/work --chrome-flags=--headless=new --quiet
```

The committed reports are the median performance run of 3 per URL. To repeat that (Git Bash), then copy the median run's files to `reports/`:

```
for i in 1 2 3; do
  npx lighthouse "https://gigspaid.com/" --form-factor=mobile --only-categories=performance,accessibility,best-practices,seo --output=html --output=json --output-path=reports/all-runs/home-$i --chrome-flags=--headless=new --quiet
  npx lighthouse "https://gigspaid.com/#/work" --form-factor=mobile --only-categories=performance,accessibility,best-practices,seo --output=html --output=json --output-path=reports/all-runs/work-$i --chrome-flags=--headless=new --quiet
done
```

If Chrome is not found, set `CHROME_PATH` (for example `export CHROME_PATH="/c/Program Files/Google/Chrome/Application/chrome.exe"`). On Linux CI add `--no-sandbox` to `--chrome-flags`. Scores vary by a few points between runs and over time as the live site changes.
