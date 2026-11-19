# Umbraco launch check - detailed checks

Environment tags: **D** = dev, **S** = stage, **P** = prod. "Code" checks read the config for the target environment (base `appsettings.json` merged with `appsettings.{Env}.json`, `web.config` + `web.{Env}.config` transform, env vars set in `web.config` `<aspNetCore><environmentVariables>` or the publish profile). "Live" checks hit the URL.

All projects are hosted on **IIS** (ASP.NET Core Module, in-process or out-of-process). Prefer IIS-level fixes in `web.config` (URL Rewrite rules, `staticContent`, `httpProtocol`, `requestFiltering`) unless the project already handles the concern in middleware - then fix it where it is already handled.

Verify every Umbraco-specific expectation against docs.umbraco.com for the project's major version and link the page.

---

## 1. Umbraco and ASP.NET configuration (code; D/S/P, expectations differ per env)

- **Environment name**: `ASPNETCORE_ENVIRONMENT` resolves to the expected value for the target (from `web.config` transform, publish profile `EnvironmentName`, or IIS - if it is set only on the server, make it a 🔍 manual item). A missing value on IIS means `Production`, which silently loads prod config on stage - flag that.
- **Runtime mode** (S/P): prod must use `Umbraco:CMS:Runtime:Mode = Production` (v10.1+). Check the version's prerequisites from the docs ("Runtime modes" page), typically: ModelsBuilder `ModelsMode` = `Nothing` (models built at compile time, `SourceCodeManual`/`SourceCodeAuto` on dev), `Umbraco:CMS:Hosting:Debug = false`, runtime minification settings (v10-v13: `RuntimeMinification:UseInMemoryCache = true`, `CacheBuster = Version`/`AppDomain`), `Umbraco:CMS:WebRouting:UmbracoApplicationUrl` set. Stage: recommend `Production` too so stage matches prod (⚠️ if not). Dev: `Development` or `BackofficeDevelopment` expected.
- **Debug off** (S/P): `Umbraco:CMS:Hosting:Debug` false; `<compilation debug>` not relevant on Core but check for leftover `UseDeveloperExceptionPage()` outside an `IsDevelopment()` guard; `stdoutLogEnabled="false"` in `web.config` (or it fills the disk).
- **HTTPS** (S/P): `Umbraco:CMS:Global:UseHttps = true`.
- **Install/upgrade**: `Umbraco:CMS:Unattended:InstallUnattended` and `UpgradeUnattended` are as intended for that env; no install credentials (`UnattendedUserPassword`) committed in appsettings - ❌ if a real-looking password is in the repo.
- **Secrets in repo**: connection strings with passwords, SMTP passwords, API keys in committed appsettings for S/P - ⚠️ with recommendation (server-side env vars / user secrets). Do not print the secret values in the checklist; reference file:line only.
- **Logging** (S/P): Serilog `MinimumLevel.Default` not `Debug`/`Verbose`; log path not inside a publicly served folder.
- **Umbraco settings worth confirming** (per version docs): `Umbraco:CMS:Global:Smtp` configured (forms/notifications need it); `Umbraco:CMS:Content:Notifications:Email`; `Umbraco:CMS:WebRouting` `UseStrictDomainMatching`, `DisableRedirectUrlTracking = false` (keeps automatic 301s when nodes are renamed); `Umbraco:CMS:Imaging` cache `BrowserMaxAge`/`CacheMaxAge`; `Umbraco:CMS:Examine` settings; MainDom lock / load balancing settings if more than one instance.
- **Delivery API**: if enabled, `PublicAccess` and `ApiKey` intended; if not used, disabled.
- **uSync** (if installed): on prod `ImportAtStartup` = `None` (or as agreed), `ExportOnSave` off; uSync dashboard not left open for non-admins.
- **ModelsBuilder / compilation**: project builds in Release; `RazorCompileOnBuild` default; no `ModelsMode=InMemoryAuto` on S/P.
- **Hardcoded hosts**: grep the project (Views, appsettings, JS, CSS) for dev/stage hostnames and `localhost` that would end up on prod.
- **Packages**: `dotnet list <csproj> package --vulnerable --include-transitive` and `--outdated`; check the Umbraco major version's support/EOL status on the Umbraco product release calendar - ⚠️ if EOL within 6 months.
- **Licenses** (S/P): Umbraco Forms / Deploy / Workflow / Engage etc. - licenses (in `umbraco/Licenses` or `Umbraco:Licenses` config) cover the prod hostname. 🔍 manual if not determinable from code.

## 2. Umbraco health check (D via Umbraco MCP; S/P and D-without-MCP manual)

- D with Umbraco MCP: run all health check groups (Configuration, Data Integrity, Live Environment, Permissions, Security, Services). Report each warning/error as an item with the docs link for that check.
- Otherwise: 🔍 manual item asking the user to run Settings -> Health Check -> "Check all groups" on `<env>/umbraco` and paste failures.
- Independently of the backoffice, reproduce what can be reproduced from outside: HTTPS, security headers, debug mode, SMTP config, `X-Powered-By`, click-jacking protection.

## 3. Hosting, redirects, caching (live; S/P, D partial)

- **Redirect chains**: request each of `http://host`, `http://www.host`, `https://host`, `https://www.host` (whichever is non-canonical) plus a deep URL in each variant. Expect **exactly one 301** to the canonical `https://` host (308 also acceptable). ❌ on 302/307 for canonicalisation, ⚠️ on 2+ hops. Report every hop.
- **Trailing slash / case**: `/page` vs `/page/` and `/Page/` - one of them canonical, the other redirects (301) or both serve with a correct canonical tag. Check Umbraco `AddTrailingSlash` setting vs. actual sitemap/canonical URLs.
- **Canonical host consistency**: canonical tags, sitemap, `og:url`, schema `url`/`@id` all use the same final host+scheme.
- **TLS**: certificate valid, matches host (incl. www variant), expiry date (⚠️ < 30 days), full chain served. HSTS header on https responses (`max-age` >= 15552000 recommended; `includeSubDomains`/`preload` only as suggestion). HTTP/2 enabled (`curl --http2 -sI -o /dev/null -w '%{http_version}'`).
- **Compression**: HTML/CSS/JS/SVG/JSON served with `br` or `gzip` (`curl -H 'Accept-Encoding: br, gzip'`). On IIS: `urlCompression doStaticCompression/doDynamicCompression` (and IIS Brotli module if available).
- **Static asset caching**: sample CSS, JS, fonts, images from `wwwroot` and `/media/...` (incl. ImageSharp-processed URLs with `?width=`): `Cache-Control: public, max-age >= 2592000` (30 d); `immutable` + 1 year when files are fingerprinted (`asp-append-version`, hashed bundles, Smidge cache buster). ⚠️ if assets are not fingerprinted but cached long (stale after deploy). HTML pages must not be cached publicly for long. On IIS: `<staticContent><clientCache cacheControlMode="UseMaxAge" .../>` or `StaticFileOptions.OnPrepareResponse`.
- **Security headers** on HTML responses: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, clickjacking protection (`Content-Security-Policy: frame-ancestors 'self'` or `X-Frame-Options: SAMEORIGIN`), `Content-Security-Policy` (⚠️ if missing, propose a starting policy - mention it must be tested with GTM/analytics/embeds). Headers removed: `Server` (IIS: `requestFiltering removeServerHeader="true"`), `X-Powered-By` (IIS: `<remove name="X-Powered-By"/>`), `X-AspNet-Version`.
- **Error handling**: request a random non-existent URL -> must return HTTP **404** (not 200 = soft 404, not 500) with the custom 404 page (Umbraco `Error404Collection` / content finder). A malformed request should not reveal a stack trace or the yellow/dev error page. Check IIS `httpErrors` does not swallow the app's 404 page (`existingResponse="PassThrough"` when the app renders its own).
- **Mixed content**: no `http://` resources on `https://` pages.
- **Backoffice exposure** (P): `/umbraco` reachable publicly - ⚠️ suggest IIS IP restriction or `Umbraco:CMS:Security` / custom middleware if feasible for the team; check the login page sends `X-Robots-Tag: noindex` / is disallowed in robots.txt. `/umbraco/api`, `/install` not exposing anything unexpected.
- **Uptime/health endpoint**: is there an ASP.NET Core health endpoint (`MapHealthChecks`) for monitoring? ⚠️ suggestion only.

## 4. Indexing safety (live + code; D/S/P)

Stage is public (not IP-restricted), so stage must be protected from indexing by headers/meta/robots alone.

- **Stage (S)**:
  - Every HTML response has `<meta name="robots" content="noindex, nofollow">` **and** an `X-Robots-Tag: noindex, nofollow` header. The header also covers PDFs, images and other media; ❌ if only the meta tag is present (media/PDF files from stage can get indexed). Check a media file (`/media/...pdf` or image) for the header too.
  - robots.txt returns `Disallow: /` for `*`. Note in the item: robots disallow alone does not stop indexing of linked URLs, which is why `noindex` is also required - and that crawlers only see `noindex` on pages they are allowed to crawl, so the header is the dependable part.
  - Canonical tags on stage pointing to prod are acceptable; canonical pointing to stage is fine too as long as noindex is present.
  - Search Google for `site:<stage host>` is not possible automatically - add a 🔍 manual item: "Search `site:<stage host>` in Google; if results appear, request removal in the stage GSC property or keep noindex until they drop."
- **Prod (P)** - the inverse failure:
  - No `noindex` in meta robots or `X-Robots-Tag` on any sampled page or the homepage (❌ Critical).
  - robots.txt is not `Disallow: /`; does not block CSS/JS/media needed for rendering; references the sitemap with an absolute prod URL.
  - No `noindex`/`nofollow` leftover from stage config in `appsettings.Production.json`, transforms, or layout views.
- **Where it's decided (code)**: find how the robots meta/header and robots.txt differ per environment (appsettings flag, environment check in layout, SEO package setting, IIS rewrite). ⚠️ if the noindex logic depends on something easy to forget (manually edited content node, setting in backoffice per environment) - propose making it config-driven per environment so prod indexes by default and every non-prod env noindexes by default.
- **Sitemap** (S/P): `/sitemap.xml` (or as referenced in robots.txt) exists, is valid XML, uses the env's own canonical host (prod sitemap must not contain stage/dev hosts), every URL (sample up to 200) returns 200 without redirect, is not noindexed, and matches its own canonical. Pages with "hide from navigation/sitemap" or noindex properties are excluded. `lastmod` present and plausible. For multilingual: `xhtml:link` alternates or separate sitemaps per culture.
- **Dev (D)**: same code-level checks; live check that local responses are noindex is informational only.

## 5. On-page SEO and semantics (live; page sample)

For each page in the sample:

- `<title>` present, unique across the sample, ~30-60 characters; meta description present, unique, ~70-160 characters.
- Exactly one `<h1>`; heading levels do not skip (h2 -> h4) in main content.
- `<html lang>` set and matches the culture; `<meta charset>`, `<meta name="viewport" content="width=device-width, initial-scale=1">`.
- Landmarks: `<header>`, `<nav>`, `<main>` (exactly one), `<footer>`; lists/buttons/links used semantically (no clickable `div`s without role).
- Images: `alt` present on every `<img>`; empty `alt=""` only for decorative images; flag alts that are file names or duplicated generic text. `width`/`height` (or aspect-ratio) set to prevent CLS; `loading="lazy"` below the fold but **not** on the LCP image.
- Canonical tag present, absolute, self-referencing (unless intentionally pointing elsewhere), one per page.
- `hreflang` (multilingual): reciprocal, includes self, `x-default`, matches Umbraco Culture & Hostnames.
- Open Graph (`og:title`, `og:description`, `og:image` absolute + reachable + >= 1200x630, `og:url`, `og:type`) and `twitter:card`.
- Favicon set (`favicon.ico`, PNG/SVG icons, `apple-touch-icon`), web manifest if used, `theme-color`.
- **Broken links**: crawl (see SKILL.md limits). Report 4xx/5xx with the page(s) linking to them; ⚠️ internal links that redirect (link directly to final URL); external links that time out as ⚠️, not ❌.
- **Redirect URL management**: if this is a relaunch of an existing site, 🔍 manual/ask for the old URL list (old sitemap, analytics export) or use the Wayback Machine CDX API (`http://web.archive.org/cdx/search/cdx?url=<host>/*&output=json&fl=original&collapse=urlkey&limit=500`) to sample old URLs and check they 301 to relevant new pages (not all to the homepage).
- **Analytics/tag manager**: GTM/GA4 present on prod; on stage absent or using a different container/measurement ID; cookie consent banner present and blocks non-essential tags until consent (check the network requests in the browser before accepting).
- **Accessibility**: Lighthouse a11y score plus the main failing audits (contrast, form labels, link names, button names). Note that automated checks cover only part of WCAG.

## 6. Performance and responsiveness (live)

- **Lighthouse** for homepage + 2-3 key templates, mobile and desktop:
  - S/P: `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=<url>&strategy=mobile|desktop&category=performance&category=accessibility&category=best-practices&category=seo`
  - D (or fallback): `npx --yes lighthouse <url> --quiet --chrome-flags="--headless=new" --output=json --output=html --output-path=<path> [--preset=desktop]`. For a local https dev cert add `--chrome-flags="--headless=new --ignore-certificate-errors"`.
  - Report: the 4 category scores, LCP, CLS, TBT (lab proxy for INP), FCP, Speed Index, TTFB (server response time). Thresholds: LCP <= 2.5 s, CLS <= 0.1, INP <= 200 ms, TTFB <= 0.8 s; performance score >= 90 desktop, >= 70 mobile as ⚠️ thresholds (not hard fails).
  - List the top opportunities with the specific resources: render-blocking CSS/JS, unused JS/CSS, images not in modern formats / oversized (suggest ImageSharp `?width=&format=webp` + `srcset`), LCP element and what delays it (preload hint, `fetchpriority="high"`), third-party weight (GTM, chat widgets, video embeds -> facade), font loading (`font-display: swap`, preload, self-host), long main-thread tasks.
- **Field data without Search Console** (P): the PSI response `loadingExperience` / `originLoadingExperience` contains CrUX real-user Core Web Vitals (p75 LCP, CLS, INP) when the site has enough traffic. Report them separately from lab data; "no field data" is informational (new/low-traffic site). Note it takes ~28 days after launch to populate.
- **TTFB / server**: measure `curl -w '%{time_starttransfer}'` 3x on homepage and a deep page; ⚠️ if consistently > 0.8 s (check output caching, Examine queries in views, missing `IPublishedContent` caching, app pool cold start - on IIS suggest `startMode="AlwaysRunning"` + `preloadEnabled="true"` + Application Initialization to avoid cold starts after recycle).
- **Responsiveness**: in the built-in browser, load the sample pages at 375x812, 768x1024 and desktop; check for horizontal scroll (`document.documentElement.scrollWidth > clientWidth`), overlapping/cut-off elements, readable font sizes, tap targets; attach notable screenshots to the chat (describe them in the checklist).
- **Page weight**: total transfer size and request count for the homepage; ⚠️ > 3 MB or > 100 requests.

## 7. Schema.org structured data (live; page sample)

- Extract all JSON-LD (`<script type="application/ld+json">`) and microdata/RDFa from each sampled page. ❌ on invalid JSON.
- **Validate**:
  - Schema Markup Validator: try `curl -sS -X POST https://validator.schema.org/validate --data-urlencode "url=<url>"` (response is JSON prefixed with `)]}'`; parse `numErrors`, `numWarnings`, and per-triple errors). For dev/localhost (not reachable by the validator) post the HTML instead with `--data-urlencode "html=<page html>"`. If the endpoint does not respond as expected, use the browser on https://validator.schema.org/ (code snippet tab for dev).
  - Google Rich Results Test (https://search.google.com/test/rich-results?url=<url>): no API. Try it in the built-in browser for S/P; if it shows a CAPTCHA or bot check, do not attempt to solve it - mark 🔍 manual with the direct link. Not usable for localhost (use the "code" input manually, 🔍).
  - Also check locally against Google's documented required/recommended properties for each rich-result type found (Organization, LocalBusiness, WebSite, BreadcrumbList, Article/BlogPosting, Product + Offer, FAQPage, Event, JobPosting, Recipe, VideoObject...), using developers.google.com/search/docs/appearance/structured-data as reference.
- **Consistency**: structured data matches visible content (name, price, dates, breadcrumbs, FAQ text); URLs absolute and on the env's canonical host (prod schema must not contain stage URLs); images reachable; dates ISO 8601 with timezone.
- **Graph linking - propose improvements**:
  - One `Organization` (or `LocalBusiness`) node per site with a stable `@id` (e.g. `https://www.example.com/#organization`), `name`, `url`, `logo` (ImageObject), `sameAs` (social profiles), `contactPoint`/`address` where relevant.
  - One `WebSite` node (`@id` `/#website`) with `publisher` -> `{"@id": ".../#organization"}`, `inLanguage`; `SearchAction` only if site search exists.
  - Each page: `WebPage` (or subtype `AboutPage`, `ContactPage`, `CollectionPage`, `FAQPage`...) with `@id` = `<page url>#webpage`, `isPartOf` -> website, `breadcrumb` -> BreadcrumbList `@id`, `primaryImageOfPage`, `inLanguage`, `datePublished`/`dateModified`.
  - Content entities (`Article`, `Product`, `Event`...) with `mainEntityOfPage` -> webpage `@id`, `publisher`/`author`/`brand` referencing the Organization `@id` instead of repeating the object.
  - Prefer a single `@graph` array per page over several unrelated scripts; avoid duplicate conflicting nodes (e.g. two Organizations from a package and from the template).
  - Suggest additional types that fit the content found but are missing (BreadcrumbList on deep pages, `FAQPage` where FAQ blocks exist - noting Google shows FAQ rich results only for limited sites, `Product`/`Offer`, `Event`, `JobPosting`, `LocalBusiness` with `openingHoursSpecification`).
  - Point to the Razor partial / view component that generates the schema so fixes can be made in code.

## 8. Other checks

- **Forms** (Umbraco Forms or custom): spam protection present (reCAPTCHA/hCaptcha/honeypot); SMTP configured; workflow recipients are not test addresses on prod. Test submission only after asking the user (it sends real emails).
- **Cookies**: session/auth cookies `Secure`, `HttpOnly`, `SameSite`; consent banner as in section 5.
- **robots.txt / sitemap / 404 / 500 for each culture domain** if the site has multiple hostnames.
- **Media**: very large originals in `/media` served unresized on pages (> 500 KB images) - list them with the page they appear on.
- **Backups / rollback**: 🔍 manual reminder (P pre-launch only): DB + media backup taken, rollback plan agreed.
- **Monitoring**: 🔍 manual reminder (P): uptime monitor on homepage + health endpoint, certificate expiry alert, log/error alerting.
