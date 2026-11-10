# garrett-he's Scoop Bucket

[![Tests](https://github.com/garrett-he/scoop-bucket/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/garrett-he/scoop-bucket/actions/workflows/ci.yml)
[![Excavator](https://github.com/garrett-he/scoop-bucket/actions/workflows/excavator.yml/badge.svg?branch=master)](https://github.com/garrett-he/scoop-bucket/actions/workflows/excavator.yml)

A personal [Scoop](https://scoop.sh/) bucket for Windows tools that are missing
from, or lag behind, the official buckets. It exists to keep my own machines
reproducible; `bucket/` is the source of truth for what is available.

## Installation

Add the bucket once, then install apps by manifest name:

```pwsh
scoop bucket add garrett-he https://github.com/garrett-he/scoop-bucket
scoop install garrett-he/<app>
```

If you prefer an SSH remote:

```pwsh
scoop bucket add garrett-he git@github.com:garrett-he/scoop-bucket.git
```

`scoop update` refreshes the bucket itself, `scoop update <app>` upgrades one
app, and `scoop bucket rm garrett-he` removes the bucket entirely.

## Repository layout

| Path | Purpose |
| --- | --- |
| `bucket/` | One `<app>.json` manifest per app. `app-name.json.template` is the starting point. |
| `scripts/` | Helper scripts, registry entries and other files referenced by manifests. |
| `deprecated/` | Retired manifests, kept for reference and never offered for install. |
| `bin/` | Thin wrappers around Scoop's own maintenance scripts, pre-wired to `bucket/`. |
| `.github/workflows/` | CI, scheduled updates and issue/PR handlers. |

Both `scripts/` and `deprecated/` currently hold only a `.gitkeep`; delete it
once the directory contains real files.

## Adding a manifest

1. Start from the template and fill in the fields:

   ```pwsh
   Copy-Item .\bucket\app-name.json.template .\bucket\<app>.json
   ```

   Drop the leading `##` comment and every field the app does not need. Set
   `checkver` and `autoupdate` as well — without them Excavator cannot keep the
   app fresh. See [App Manifests](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests)
   for the field reference; VS Code already gets schema completion for
   `bucket/**/*.json` from `.vscode/settings.json`.

2. Compute hashes and normalize the JSON:

   ```pwsh
   .\bin\checkhashes.ps1 -App <app> -Update
   .\bin\formatjson.ps1 -App <app>
   ```

3. Verify every download URL is reachable:

   ```pwsh
   .\bin\checkurls.ps1 -App <app>
   ```

4. Run the test suite (see [Local tests](#local-tests)), then commit:

   ```pwsh
   git add bucket/<app>.json
   ```

   Use a [Conventional Commits](https://www.conventionalcommits.org/) message
   such as `feat: add <app>`, and keep one manifest change per commit.

## Keeping manifests up to date

The Excavator workflow runs every four hours and commits version bumps on its
own, so manual work is only needed when an upstream URL or hash changes shape.

```pwsh
.\bin\missing-checkver.ps1                  # manifests without checkver/autoupdate
.\bin\checkver.ps1 -App <app> -Update      # bump version, url and hash
.\bin\checkhashes.ps1 -App <app> -Update   # recompute hashes after a manual url edit
.\bin\auto-pr.ps1 -Push                    # update every app and push to origin
```

`bin/auto-pr.ps1` defaults `-Upstream` to `garrett-he/scoop-bucket:master` and
uses `master` as its origin branch. `-Push` writes straight to the origin
branch; `-Request` opens a pull request instead and requires
[hub](https://hub.github.com/) to be installed. Run it without either switch
to print its help.

## Local tests

`bin/test.ps1` invokes the Pester suite bundled with Scoop core. Prerequisites:

```pwsh
Install-Module BuildHelpers -Scope CurrentUser -MinimumVersion 2.0.1
Install-Module Pester -Scope CurrentUser -MinimumVersion 5.2.0
.\bin\test.ps1
```

The wrappers in `bin/` resolve `SCOOP_HOME` from `scoop prefix scoop` unless the
variable is already set, so a local Scoop installation is required. CI instead
points `SCOOP_HOME` at a checkout of `ScoopInstaller/Scoop`.

## Automation

| Workflow | Trigger | What it does |
| --- | --- | --- |
| `ci.yml` | push to `master`/`main`, PR, manual | Runs `bin/test.ps1` on `windows-latest` for both `powershell` and `pwsh`. |
| `excavator.yml` | every 4 hours, manual | Checks versions and commits updates; `SKIP_UPDATED=1` skips current manifests. |
| `issues.yml` | issue opened / labeled `verify` | Runs the ScoopInstaller issue handler, including automatic hash fixes. |
| `pull_request.yml` | PR opened, `/verify` comment | Runs the ScoopInstaller PR handler. |

If GitHub Actions are disabled on the repository, enable them under
`Settings > Actions > General`: allow all actions and reusable workflows, and
make sure workflow permissions are not restricted to read-only —
`excavator.yml` declares `contents: write`.

## Conventions

- Commits use Conventional Commits; keep one manifest change per commit.
- `.gitattributes` stores LF in Git and checks out CRLF on Windows.
- `.markdownlint.json` disables the line-length rule and scopes duplicate-heading
  checks to siblings.
- The default branch is `master`; update the badge URLs and `bin/auto-pr.ps1`
  together if it is ever renamed.

## Credits and license

Scaffolded from [ScoopInstaller/BucketTemplate](https://github.com/ScoopInstaller/BucketTemplate).
Released under the [Unlicense](LICENSE).
