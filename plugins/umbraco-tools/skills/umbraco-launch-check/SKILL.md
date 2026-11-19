---
name: umbraco-launch-check
description: Pre-launch / post-launch verification checklist for an Umbraco website (one Umbraco project in a repo, given by its .csproj). Checks Umbraco/ASP.NET config per environment (production runtime mode, debug, logging, IIS web.config), Umbraco health check, redirects (https/www, hops), caching and security headers, indexing safety (noindex on stage, none on prod), robots.txt, sitemap, on-page SEO, broken links, Lighthouse/PageSpeed (LCP, CLS, INP), responsiveness, schema.org structured data. Writes a tickable Markdown checklist to launch-checks/ in the repo root and proposes code fixes for approval. Use via /umbraco-launch-check or when the user asks to verify an Umbraco site before or after a production launch, or to check dev/stage/prod environments.
argument-hint: <path/to/Project.csproj> [--stage[=url]] [--prod[=url]] [--pages=url1,url2]
---

Run a launch-readiness check of one Umbraco project and record the results as a Markdown checklist.

Arguments: $ARGUMENTS

The detailed checks live in [checks.md](checks.md) next to this file. Read it before starting the checks.

## 1. Parse arguments

- First positional argument: path to the Umbraco `.csproj` (required). If missing or ambiguous, list the `*.csproj` files in the repo that reference `Umbraco.Cms` and ask which one.
- `--stage` / `--stage=<url>`: check the stage environment (live) + stage config in code.
- `--prod` / `--prod=<url>`: check the production environment (live) + production config in code.
- Both may be given together; run the checks for each and compare the environments where relevant (e.g. differing headers, robots, schema).
- Neither given: **dev mode** - config/code checks for dev plus live checks against the local site.
- `--pages=<url,...>`: extra pages to include in the page sample.

Target environment names map to appsettings/transform names: dev = `Development`, stage = `Staging` (also try `Stage`, `Test`, `UAT` - use whatever file actually exists), prod = `Production`.

## 2. Gather project context

1. Read the `.csproj`: Umbraco version (`Umbraco.Cms` PackageReference), target framework, other Umbraco packages (Forms, Deploy, Workflow, uSync, SEO packages, ModelsBuilder settings like `ModelsMode`/`ModelsBuilder` props).
2. Find the docs for that version: on docs.umbraco.com, pick the project's major version (version selector / the version listed in the docs navigation) and use the URL it leads to - do not guess the URL format. Fetch the relevant pages for **that** major version whenever a check depends on Umbraco behavior (runtime modes, health checks, settings keys, load balancing, hosting on IIS). Cite the doc URL in the checklist item. Never assume a setting from memory if the docs for that version say otherwise.
3. Collect config: `appsettings.json`, `appsettings.{Env}.json`, `Properties/launchSettings.json`, `web.config` and `web.{Env}.config` transforms, `Properties/PublishProfiles/*.pubxml`, uSync config, CI/CD yaml files (`azure-pipelines*.yml`, `.github/workflows`, etc.), `Program.cs`/`Startup.cs`/composers for middleware (headers, redirects, rewrites, robots/sitemap handling).
4. Resolve URLs for each requested environment (when no URL value was passed), in this order:
   - `appsettings.{Env}.json` -> `Umbraco:CMS:WebRouting:UmbracoApplicationUrl`
   - publish profile for that env -> `SiteUrlToLaunchAfterPublish`, `DeployIisAppPath` (on IIS this is often the hostname), `MSDeployServiceURL` (deploy server, usually *not* the site URL)
   - `web.{Env}.config` rewrite rules referencing a canonical host
   - CI/CD yaml, uSync domain files (`uSync/v9/Domains/*.config`), README
   - dev: `launchSettings.json` `applicationUrl` (prefer the https one)
   
   If several candidates or none are found, ask the user (show the candidates and where each came from). Record the resolved URL and its source at the top of the checklist.
5. Dev mode only: check whether the local site responds. If not, ask the user whether to start it (`dotnet run --project <csproj>` in the background) or to skip live checks.

## 3. Umbraco MCP (health check and backoffice data)

Look for an Umbraco MCP server configured for this project: `.mcp.json` in the repo root/project folder, the project entry in `~/.claude.json`, `.claude/settings*.json`, and tools currently available whose names contain `umbraco`.

- **Dev mode + Umbraco MCP available:** use it to run all health check groups and collect results; also use it for backoffice data the checks need (Culture & Hostnames / domains, redirect URL management, document types/templates for page sampling, published 404 page).
- **Stage/prod, or no MCP:** do not sign in to the backoffice and never enter credentials. Add a 🔍 manual item: "Run Settings -> Health Check -> Check all groups in the backoffice of <env URL>/umbraco and paste the failed/warning items here." When the user pastes results, fold them into the checklist and propose fixes.
- An MCP connected to a **non-local** Umbraco instance must not be used for stage/prod checks unless the user explicitly says so in chat.

## 4. Run the checks

Follow [checks.md](checks.md). Each section states which environments it applies to (D/S/P). Guidelines:

- Use `curl` (via Bash) for headers, redirect chains, status codes, robots.txt, sitemap, raw HTML. Use `curl -sS -o /dev/null -w` style output and `-I`/`-L --max-redirs 10 -w '%{url_effective} %{num_redirects}'` for chains. Report every hop.
- Use the built-in browser (`mcp__Claude_Browser__*`) for rendered DOM, responsiveness screenshots, and web tools without APIs.
- **Page sample:** homepage + one page per document type/template (derive from sitemap URLs, MCP, or Views/ folder names) + `--pages`. Cap at ~15 pages; state the sample in the checklist.
- **Crawl for broken links:** internal pages up to ~200 URLs, ~2 requests/second, follow only same-host links, check external links with HEAD (fallback GET) once each. Never submit forms during crawling.
- **Lighthouse:** S/P -> PageSpeed Insights API v5 (no key; if rate-limited, wait and retry once, then fall back to local Lighthouse). D (or fallback) -> local `npx lighthouse`. Run mobile and desktop for the homepage and 2-3 key templates. Save raw reports (Lighthouse JSON + HTML via `--output=json --output=html`, PSI JSON) to `<repo root>/launch-checks/raw/<project>-<yyyy-mm-dd>-<env>/` and link them; for PSI also link `https://pagespeed.web.dev/analysis?url=<url>`.
- Do not perform anything state-changing on the sites (form submissions, logins, purges). If a check would need it, make it a 🔍 manual item or ask first.
- If a tool/API is unreachable (validator down, bot detection on Rich Results Test), do not try to bypass it - mark the item 🔍 manual with a direct link the user can open.

## 5. Write the checklist

File: `<repo root>/launch-checks/<project>-<yyyy-mm-dd>-<env>.md` (`<project>` = the `.csproj` file name without extension), where `<env>` is `dev`, `stage`, `prod` or `stage+prod`. If the file already exists (re-run the same day), update it in place: keep the user's ticks on items that still fail, update statuses/evidence, and add a "Changes since previous run" note at the top.

Make sure `launch-checks/` is git-ignored: if `git check-ignore` says the file is not ignored, add `/launch-checks/` to the repo root `.gitignore` and mention it.

If the repo root is also the folder of a web project (`.csproj` in the repo root), files in `launch-checks/` would be published by the Web SDK (`.json` and `.config` are Content by default) - in that case propose `<Content Remove="launch-checks\**" />` for that `.csproj` as a fix.

Structure:

```markdown
# Umbraco launch check - <Project> - <env> - <yyyy-mm-dd>

- Project: `<path to csproj>` - Umbraco <version> (docs: <versioned docs URL>)
- Environment URL(s): <url> (source: <file/key or "passed as argument">)
- Page sample: <list>
- Summary: ✅ n  ❌ n  ⚠️ n  🔍 n

## Critical (fix before/immediately after launch)
- [ ] ❌ <title> - <one-line finding>
  - Evidence: <header/value/url/file:line>
  - Fix: <what to change> <(auto-fixable: F1)>

## <Section from checks.md>
- [x] ✅ <title> - <finding>
- [ ] ⚠️ ...
- [ ] 🔍 <manual step for the user>

## Proposed code fixes
| ID | File | Change | Fixes item |
|----|------|--------|-----------|
| F1 | ... | ... | ... |

## Not checked / limitations
- <what could not be verified and why>
```

Rules: passing items are pre-ticked `[x]`; failures, warnings and manual items unticked. Every non-pass item has evidence and a concrete fix. Put the most severe items in "Critical" (indexing mistakes, debug/dev mode on prod, broken https/redirects, exposed errors, 5xx pages) and keep them also referenced in their section.

## 6. Propose and apply fixes

1. After writing the checklist, show the user the "Proposed code fixes" table in chat (only fixes that are safe code/config changes in this repo: appsettings, web.config/transforms, robots/sitemap code, header middleware, Razor templates for meta/alt/h1/schema, etc.). Content fixes done in the backoffice and server/IIS changes are listed as manual items instead.
2. Ask which fixes to apply (all / selected IDs / none). Do not edit anything before approval.
3. Apply approved fixes matching the project's code style, build the project (`dotnet build <csproj>`), and report the result. Update the checklist: mark fixed items as "fixed in code - needs deploy to verify" (they stay unticked until verified live).
4. Never commit. Never change files outside this Umbraco project, except the checklist and raw reports in `<repo root>/launch-checks/` and the repo root `.gitignore`.

## 7. Report in chat

Short summary: counts per status, the critical items, link to the checklist file, and what needs the user's manual action.
