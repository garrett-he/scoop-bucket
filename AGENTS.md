# AGENTS.md

Personal Scoop bucket for Windows apps missing from official buckets. `bucket/`
is the source of truth; default branch is `master`.

## Layout

| Path | Purpose |
| --- | --- |
| `bucket/` | One `<app>.json` per app, from `app-name.json.template`. |
| `bin/` | Thin wrappers around Scoop's own scripts, pre-wired to `bucket/`. |
| `scripts/` | Helper files referenced by manifests (currently `.gitkeep`). |
| `deprecated/` | Retired manifests, never offered for install (currently `.gitkeep`). |
| `skills/scoop-bucket/` | Project skill: manifest authoring workflow and references. Read `SKILL.md` before adding, updating or reviewing a manifest. |

## Rules

- Never guess a payload or a hash: download and inspect the real file, compute
  SHA-256 locally or verify a published checksum.
- Every manifest needs `checkver` and `autoupdate` — Excavator depends on them.
- Record non-obvious decisions in the manifest's `##` field, not outside it.
- One manifest per commit; Conventional Commits with a body and the manifest
  name as scope (e.g. `feat(flclash): add flclash manifest`); native `git` (no
  commit-wrapping tool); no AI signatures or co-author lines.
- Keep the skill generic. Do not add app names, vendor URLs or version numbers
  to `skills/scoop-bucket/`; read `bucket/*.json` for concrete examples.
- Markdown and JSON: UTF-8 no BOM, one trailing newline. `.gitattributes`
  stores LF in Git and checks out CRLF on Windows.

## Common commands

Run from the repo root; `<app>` is the manifest name.

```pwsh
.\bin\formatjson.ps1 -App <app>
.\bin\checkver.ps1 -App <app> -Update
.\bin\checkhashes.ps1 -App <app> -Update
.\bin\checkurls.ps1 -App <app>
.\bin\test.ps1                       # Pester 5.2.0 + BuildHelpers 2.0.1
scoop download .\bucket\<app>.json -f -u
git diff --check
```

`bin/` wrappers resolve `SCOOP_HOME` from `scoop prefix scoop` unless already
set; a local Scoop install is required. CI runs `bin/test.ps1`; Excavator checks
versions every four hours. See `README.md` for the full guide and
`skills/scoop-bucket/references/` for validation traps and manifest patterns.
