# `sast` — Claude Code Plugin (SAST)

A self-contained Claude Code plugin (`sast`) that brings **SAST** directly into Claude Code via slash-command skills, subagents, and lifecycle hooks.

## Plugin Structure

```text
sast/
├── .claude-plugin/
│   ├── plugin.json                  # Plugin manifest (name: "sast")
│   ├── marketplace.json             # Marketplace manifest (name: "sast")
│   └── icon.png                     # 1024x1024 square PNG plugin icon
├── skills/
│   ├── scan/
│   │   └── SKILL.md                 # /sast:scan    — Full SAST + taint + secrets + IaC + SCA scan
│   ├── diff/
│   │   └── SKILL.md                 # /sast:diff    — Fast incremental scan on git changed files
│   ├── secrets/
│   │   └── SKILL.md                 # /sast:secrets — Secret detection + live key validation + git history
│   ├── sca/
│   │   └── SKILL.md                 # /sast:sca     — Dependency CVEs (OSV), dep confusion, CMS, licenses
│   ├── triage/
│   │   └── SKILL.md                 # /sast:triage  — Threat model, FP gate inspection & exploitability triage
│   └── fix/
│       └── SKILL.md                 # /sast:fix     — Auto-remediation diffs & post-patch verification
├── agents/
│   ├── security-auditor.md          # @agent-sast:security-auditor
│   └── remediation-engineer.md      # @agent-sast:remediation-engineer
├── hooks/
│   └── hooks.json                   # PostToolUse (Write|Edit) secret tripwire & optional single-file scan
├── scripts/
│   ├── sast_runner.py               # Bundled self-contained SAST runner & Claude summary formatter
│   └── post_edit_check.py           # Fast PostToolUse hook handler
└── README.md
```

## Quick Start

### Install from GitHub Marketplace (Recommended)
```bash
claude plugin marketplace add vulnz/sast
claude plugin install sast@sast
```

Or inside an interactive Claude Code session:
```text
/plugin marketplace add vulnz/sast
/plugin install sast@sast
```

### Validate the Plugin Locally
```bash
claude plugin validate .
```

## Slash Commands (Skills)

| Command | Description |
| :--- | :--- |
| `/sast:scan [path]` | Full SAST + AST/taint (16+ languages) + secrets + IaC + SCA scan |
| `/sast:diff [path]` | Fast incremental SAST scan of git modified, staged, and untracked files (`--changed`) |
| `/sast:secrets [path]` | Scan working tree and `--git-history` for hardcoded credentials with live provider key validation |
| `/sast:sca [path]` | Audit dependencies (OSV CVEs), dependency confusion, bundled JS libs, CMS plugins, and `--licenses` |
| `/sast:triage [path]` | Build STRIDE `--threat-model` and triage findings against the 0-FP real-only gate |
| `/sast:fix [path]` | Remediate vulnerabilities (`--remediation`) and re-verify with `--changed` |

## Subagents

- **`@agent-sast:security-auditor`** — Runs the bundled SAST scanner, traces source-to-sink dataflows in source files, and produces prioritized vulnerability reports.
- **`@agent-sast:remediation-engineer`** — Crafts minimal security patches and re-runs the scanner to verify each fix.

## Plugin Options (`userConfig`)

Configured through Claude Code's plugin configuration dialog (`/plugin` → Configure options):

- `scan_on_edit` (`boolean`, default `false`) — Enable automatic single-file SAST scanning inside the `PostToolUse` (`Write|Edit`) hook.
- `provider_api_key` (`string`, `sensitive: true`, optional) — Optional API key stored in Claude Code's secure credential storage for standalone AI triage/remediation diffs.

## The SAST Engine (real binary vs. bundled fallback)

This plugin is the Claude Code front-end for the **Insomnia SAST engine** — the same
engine behind [insom.ai](https://insom.ai) and [scangit.org](https://scangit.org).
It resolves its scanner in this order:

1. **Real engine (preferred).** If the `sast` binary is on `PATH` (or `python -m sast`
   resolves), every skill shells out to it — full cross-file AST/taint analysis across
   16+ languages, 230+ secret rule packs with live key validation, OSV-backed SCA, IaC
   checks, CMS/web-shell detection, and the calibrated triage classifier.
2. **Bundled fallback.** If the `sast` binary is not installed, `scripts/sast_runner.py`
   runs a small, self-contained regex/AST ruleset so a scan still works out of the box
   with zero setup. The fallback is intentionally lighter than the full engine.

To get the full engine, install the launcher (it fetches the OS-matched, checksum-verified
engine binary from insom.ai on first run and auto-updates):

```bash
pip install sast          # or: npm i -g @insom/insom
sast self-version         # confirms the real engine is resolved
```

Once `sast` is on `PATH`, the plugin automatically uses it — no plugin reconfiguration needed.

## Data Handling & Network Behavior

This plugin runs locally and scans files you point it at. Two behaviors reach the network,
both opt-out:

- **Live secret validation (secrets scan).** By default the secrets scan tests each detected
  credential against its own provider's API (e.g. a found GitHub token is checked against the
  GitHub API) to label it `VALIDATED` / `invalid` / `not validated`. This sends the detected
  secret **only to that secret's own vendor**, never to insom.ai or any third party, and only
  for keys already found in your code. Disable it with `--noapivalidate` (or `/sast:secrets … --noapivalidate`).
- **Engine download (real binary only).** If you install the `sast` launcher, it downloads the
  OS-matched engine binary from insom.ai on first run (checksum-verified, cached). The plugin's
  bundled fallback makes **no** network calls.

The plugin does not collect telemetry, does not send your source code anywhere, and stores no
personal data. `provider_api_key` (optional) is held in Claude Code's secure credential storage
and used only for AI-assisted triage/remediation when you enable it.

## License

The plugin/launcher code in this repository is **MIT** © CQR Cybersecurity LLC
(see `LICENSE`). The Insomnia SAST **engine binary** the launcher downloads is
proprietary CQR software under separate terms.
