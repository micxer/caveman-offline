# caveman-offline

Compress Claude Code's output. Runs entirely on your machine — no network calls,
no gateway, no telemetry, no CLI to install.

A stripped derivative of [JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman),
packaged as a Claude Code plugin.

```text
Before:  Sure! I'd be happy to help you with that. The issue you're
         experiencing is likely caused by an off-by-one error in the
         token expiry comparison within your authentication middleware...

After:   Bug in auth middleware. Token expiry check use `<` not `<=`. Fix:
```

Code blocks, error strings, API names, commands and numbers are never touched.
Compression applies to prose only.

## Why this exists

Upstream started as a prose-compression skill and has since grown a hosted
commercial product around it: an LLM proxy gateway, a telemetry CLI with
login/billing, downloadable binaries, and a Playwright browse driver. As of
upstream commit `df2ccd8` the Claude Code plugin ships **21 skills**, six of
which exist solely to drive that remote service.

That matters more than it sounds, because of how Claude Code loads plugins:

- Skill **bodies** are lazy — read only when a skill activates.
- Skill **descriptions** are eager — injected into every session's system
  prompt, forever, whether you use the skill or not.

So a plugin's baseline cost scales with how many skills it registers, not how
big they are. `claude plugin details` reports it directly:

|  | Skills | Agents | Always-on cost |
|---|---|---|---|
| upstream `caveman` | 21 | 3 | **~1,244 tok** |
| `caveman-offline` | 4 | 0 | **~177 tok** |

About 1,067 tokens back in every session, for features you did not ask for.

The second reason is scope. One upstream skill, `caveman-setup`, instructs the
agent to rewrite your repository's LLM callsites so traffic is proxied through
`https://gateway.caveman.so`, and to write a vendor API key into your project's
env file. Another, `caveman-evidence-review`, reads trace payloads back out of
that service. Both are documented upstream features and neither is hidden — but
if you installed the plugin to get shorter answers, they are a surprise, and
they activate on description match like any other skill.

This build keeps the compression and drops the product.

## What it does not do

Verified by reading every executable path, not by trusting the description:

- No `http`, `https`, `fetch`, `net`, `dgram`, `axios`, `undici`, `WebSocket`,
  or `urllib` anywhere in the shipped code. The only `require`s are `fs`,
  `path`, `os`, and one `child_process`.
- That single `child_process` call is `caveman-mode-tracker.js` spawning
  `caveman-stats.js` locally on `/caveman-stats`, which reads your own session
  transcripts and a hardcoded pricing table.
- No MCP server, no statusline hijack, no installer, no auto-updater, no
  `npx`, no binary download.

Everything the plugin does is: read a Markdown file, print it to stdout, and
write a mode string to a file under your Claude config directory.

## Install

Requires Node.js and Claude Code.

### From this repository

No clone needed — Claude Code resolves `owner/repo` as a GitHub marketplace:

```text
/plugin marketplace add micxer/caveman-offline
/plugin install caveman@caveman-offline
```

Same thing from a shell, if you prefer:

```sh
claude plugin marketplace add micxer/caveman-offline
claude plugin install caveman@caveman-offline
```

To pick up later commits:

```sh
claude plugin marketplace update caveman-offline
claude plugin update caveman@caveman-offline
```

### From a local clone

Choose this if you plan to edit the ruleset — see [Customising](#customising).
It also keeps you off the network entirely after the initial clone.

```sh
git clone https://github.com/micxer/caveman-offline
```

Then point the marketplace at the checkout, using an absolute path:

```text
/plugin marketplace add /absolute/path/to/caveman-offline
/plugin install caveman@caveman-offline
```

### Either way

Start a new session. You should see compressed output immediately — the
SessionStart hook injects the ruleset before your first prompt.

If you also have upstream's plugin installed, disable it first
(`/plugin disable caveman@caveman`). Both register a `caveman` skill namespace.

## Usage

```text
/caveman              # configured default (full)
/caveman lite         # lighter compression
/caveman ultra        # maximum
/caveman off          # stop; survives /compact for the rest of the session
/caveman-commit       # Conventional Commits, compressed to intent
/caveman-review       # code review, one line per finding
/caveman-stats        # session token usage, read from local logs
```

Plain prose works too — "stop caveman" or "normal mode" are matched by the
UserPromptSubmit hook.

Compression automatically drops back to normal prose for security warnings,
irreversible-action confirmations, multi-step sequences where dropped
conjunctions would create ambiguity, and any time you ask for clarification. It
also stays off for anything persisted outside the chat: code, comments, commit
messages, PR descriptions, docs.

### Default mode

Resolved in this order:

1. `CAVEMAN_DEFAULT_MODE` environment variable
2. `.caveman/config.json` or `.caveman.json`, walking up from the working
   directory — lets a team check a project default into version control
3. `~/.config/caveman/config.json` (or `$XDG_CONFIG_HOME`, or `%APPDATA%` on
   Windows)
4. `full`

```json
{ "defaultMode": "lite" }
```

Set `"defaultMode": "off"` in a repo to opt that project out entirely.

## What's in here

```text
.claude-plugin/plugin.json        SessionStart + UserPromptSubmit hook wiring
.claude-plugin/marketplace.json   marketplace manifest
skills/caveman/                   the compression ruleset — source of truth
skills/caveman-commit/            /caveman-commit
skills/caveman-review/            /caveman-review
skills/caveman-stats/             /caveman-stats
src/hooks/caveman-activate.js     SessionStart: read SKILL.md, inject ruleset
src/hooks/caveman-mode-tracker.js UserPromptSubmit: parse /caveman, reinforce
src/hooks/caveman-config.js       mode state, per-session files
src/hooks/caveman-parse.js        shared mode-change parser
src/hooks/caveman-stats.js        session token accounting, local only
src/hooks/cavecrew-model-overrides.js  kept byte-identical; inert here
```

21 files, 150 KB. Upstream is 1,422 files, 20 MB.

Verify the cost claim yourself after installing:

```sh
claude plugin details caveman@caveman-offline
```

`src/hooks/package.json` pins that directory to CommonJS. It is load-bearing:
without it the `.js` hooks die with `require is not defined in ES module scope`
whenever an ancestor `package.json` declares `"type": "module"` — which happens
in practice, because other plugins write one into the Claude config directory.

Mode is stored per session under `<claude-config>/.caveman-sessions/<id>.mode`,
with `.caveman-active` kept as a last-write-wins compatibility mirror. That is
what makes `/caveman off` survive an auto-compaction: SessionStart re-fires on
`compact`, `resume` and `fork`, and on those it reads the stored mode instead of
re-deriving the default.

## What was left out

| Removed | Why |
|---|---|
| `caveman-setup`, `caveman-discover`, `caveman-manage`, `caveman-evidence-review`, `caveman-optimize`, `caveman-learn` | Drive the Caveman Cloud service. `caveman-setup` rewrites your app's LLM callsites to route through a third-party gateway and writes a vendor API key into your env file. |
| `cavecrew` + `cavecrew-investigator` / `-builder` / `-reviewer` | A subagent delegation framework. Changes how work gets done, not how output reads. |
| `investigate-first`, `lean-build`, `surgical-patch`, `safe-refactor`, `migration`, `verify-and-stop` | Workflow prescriptions, not compression. Their descriptions carry no "caveman" marker (deliberate upstream), so activation is hard to attribute. |
| `caveman-explore` | Redundant with Claude Code's built-in `Explore` agent. |
| `caveman-compress` | Rewrites Markdown files in place. Local-only and harmless, but out of scope. |
| `install.sh`, `install.ps1`, `bin/install.js`, `src/tools/caveman-init.js` | All fetch from the network (`curl \| bash`, `npx -y github:...`). |
| `caveman-statusline.sh` / `.ps1` | Optional badge. Omitted to avoid competing with an existing statusline; re-add from upstream if you want it. |
| `engine/`, `proxy/`, `cacheengine/`, `rewriter/`, `browse/`, `mcp/`, `shrink/`, `packages/`, `extension/`, `node_modules/@caveman-ai/cli` | Binary downloaders, Playwright driver, login/billing/telemetry. Also outside upstream's MIT grant — see `NOTICE`. |

If you want any of these, install upstream instead. They are real features of a
different product; this build simply has a narrower purpose.

## Customising

`skills/caveman/SKILL.md` is the only file that defines behavior. The
SessionStart hook reads it at runtime rather than embedding a copy, so there is
no build step.

There is still a sync step, though, and it has two sharp edges.

**Edge one: install takes a snapshot.** `/plugin install` copies the plugin into
`<claude-config>/plugins/cache/caveman-offline/caveman/<version>/`, and the
running plugin loads from that snapshot — never from your working tree.

**Edge two: update is version-gated.** `claude plugin update` compares
`plugin.json`'s `version` against the installed one and reports "already at the
latest version" when they match, *even if the files differ*. Editing `SKILL.md`
and running update is therefore a no-op. The version has to be bumped as well.

What that means depends on how you installed:

| Installed from | To apply an edit |
|---|---|
| a local clone | bump `version`, then `claude plugin update caveman@caveman-offline` |
| this repo (GitHub) | commit and push, then `claude plugin marketplace update caveman-offline` before `claude plugin update` — update reads the marketplace checkout, not your working tree |

Either way, restart afterwards, then verify it actually took:

```sh
diff -rq ~/.claude/plugins/cache/caveman-offline/caveman/<version>/ . \
  --exclude=.git --exclude=.in_use
```

`.in_use` is an empty marker directory the plugin loader creates inside the
snapshot; excluding it is what stops the diff reporting a phantom difference.

If you iterate on the ruleset often, this ceremony gets old fast — use
`claude plugin init` instead, which installs to `~/.claude/skills/<name>/` and
loads directly with no snapshot and no version bump.

## Re-syncing from upstream

Copied files are byte-identical to upstream so that `diff` stays useful:

```sh
git clone --depth 1 https://github.com/JuliusBrussee/caveman /tmp/caveman-upstream
diff -ru /tmp/caveman-upstream/skills/caveman skills/caveman
diff -ru /tmp/caveman-upstream/src/hooks src/hooks
```

Review each hunk before applying, and never copy a directory wholesale — that is
how the removed surface comes back.

## Credit

All compression design, the ruleset, and the hook implementation are Julius
Brussee's work. This repository contributes only subsetting, packaging, and
documentation. Upstream is worth your stars:
[JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman).

## License

MIT. See `LICENSE` and `NOTICE`.
