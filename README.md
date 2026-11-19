# claude-skills

Shared [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skills and agents, published as a plugin marketplace.

## Plugins

| Plugin | Contents | Usage |
|---|---|---|
| `dev-pipeline` | Skills `feature`, `bugfix`; agents `context-gatherer`, `diagnoser`, `planner`, `implementer`, `reviewer` | `/dev-pipeline:feature <task> [-test]`, `/dev-pipeline:bugfix <bug report>` |
| `umbraco-tools` | Skill `umbraco-launch-check` | `/umbraco-tools:umbraco-launch-check <path/to/Project.csproj> [--stage[=url]] [--prod[=url]] [--pages=url1,url2]` |

### dev-pipeline

Runs work through subagents pinned to different models:

- **feature**: research (Haiku) -> plan for approval (Opus) -> implement (Sonnet) -> review (Opus). Add `-test` to have tests written or updated.
- **bugfix**: gather context (Haiku) -> diagnose root cause or ask for evidence (Opus) -> approve -> implement with a regression test (Sonnet) -> verify.

Nothing is committed by the pipeline.

### umbraco-tools

`umbraco-launch-check` verifies an Umbraco site before or after a production launch: Umbraco/ASP.NET config per environment (production runtime mode, debug, logging, IIS `web.config`), Umbraco health check, https/www redirects, caching and security headers, indexing safety (noindex on stage, none on prod), robots.txt, sitemap, on-page SEO, broken links, Lighthouse/PageSpeed, responsiveness and schema.org structured data.

- No flag: dev checks. `--stage` / `--prod`: live checks of that environment; the URL is taken from the flag value or found in the project's config/publish profiles.
- Writes a tickable checklist to `launch-checks/<project>-<date>-<env>.md` in the repo root (git-ignored) and proposes code fixes, applied only after approval.
- Requirements: Node.js and Chrome (for local Lighthouse via `npx lighthouse`), `curl`, .NET SDK.

## Install

Requires access to this repository (Claude Code clones it with your git credentials).

```
/plugin marketplace add 20alex02/claude-skills
/plugin install dev-pipeline@claude-skills
/plugin install umbraco-tools@claude-skills
```

Or run `/plugin` and pick from the `claude-skills` marketplace.

Get updates:

```
/plugin marketplace update claude-skills
```

To have a project offer these plugins automatically to everyone who opens it, add to that project's committed `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-skills": {
      "source": { "source": "github", "repo": "20alex02/claude-skills" }
    }
  },
  "enabledPlugins": {
    "umbraco-tools@claude-skills": true
  }
}
```

## Repository layout

```
.claude-plugin/marketplace.json      marketplace definition (list of plugins)
plugins/<plugin>/
  .claude-plugin/plugin.json         plugin name, version, description
  skills/<skill>/SKILL.md            skills (plus supporting files)
  agents/<agent>.md                  subagents
```

## Contributing

1. Add or edit a skill under `plugins/<plugin>/skills/<skill>/` (or an agent under `plugins/<plugin>/agents/`). For a new plugin, add its folder with `.claude-plugin/plugin.json` and register it in `.claude-plugin/marketplace.json`.
2. Bump `version` in that plugin's `plugin.json` so installed copies pick up the change.
3. Open a pull request.

Skills must not reference files in a personal `~/.claude` folder; refer to agents by name, since installed plugin agents may be listed with the plugin prefix (e.g. `dev-pipeline:reviewer`).

### Maintainer setup (editing in place)

The maintainer uses the skills straight from this repo instead of installing the plugins, by linking them into the global Claude config with Windows directory junctions (no admin rights needed):

```powershell
$repo = "C:\projects\claude-skills\plugins"
New-Item -ItemType Junction -Path "$HOME\.claude\skills\bugfix" -Target "$repo\dev-pipeline\skills\bugfix"
New-Item -ItemType Junction -Path "$HOME\.claude\skills\feature" -Target "$repo\dev-pipeline\skills\feature"
New-Item -ItemType Junction -Path "$HOME\.claude\skills\umbraco-launch-check" -Target "$repo\umbraco-tools\skills\umbraco-launch-check"
New-Item -ItemType Junction -Path "$HOME\.claude\agents" -Target "$repo\dev-pipeline\agents"
```

Edits made through either path are the same files, so there is nothing to sync. Do not also install the plugins on that machine, or the skills appear twice. Note that `~/.claude/agents` points into the `dev-pipeline` plugin, so any new personal agent created there lands in this repo.
