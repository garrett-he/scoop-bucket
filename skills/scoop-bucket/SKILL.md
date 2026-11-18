---
name: scoop-bucket
description: Author, update and validate Windows app manifests in this personal Scoop bucket. Use when adding a package from a vendor download page, GitHub release, installer or portable archive; when deciding how Scoop should unpack NSIS, Inno Setup, MSI or 7z SFX payloads; when wiring checkver, autoupdate and hash sources; or when verifying a bucket/*.json manifest with this repo's bin/ wrappers.
---

# Scoop bucket manifest authoring

## Overview

Turn an upstream download into a reproducible `bucket/<app>.json`: identify the
real payload, choose how Scoop unpacks it, wire automated version and hash
updates, then verify end to end before committing. This is a project skill for
this bucket only — keep it generic, and read the actual `bucket/*.json` files for
current examples rather than hard-coding them here.

## When to use

- Adding an app: "add <app>", "new manifest for <url>".
- Updating a manifest: version bump, moved URL, changed hash shape.
- Reviewing a manifest for correctness before a commit.

## Repo contract

- `bucket/` holds one `<app>.json` per app plus `app-name.json.template`.
- `bin/` wrappers are pre-wired to `bucket/`; run them from the repo root. They
  resolve `SCOOP_HOME` from `scoop prefix scoop` unless already set.
- `scripts/` holds helper files referenced by manifests; `deprecated/` holds
  retired manifests. Both currently contain only `.gitkeep`.
- `Scoop-Bucket.Tests.ps1` loads Scoop core's bucket tests via `bin/test.ps1`.
- CI runs `bin/test.ps1`; the Excavator workflow checks versions every four
  hours, so every manifest needs `checkver` and `autoupdate`.
- Commands are PowerShell on Windows. 7-Zip is available; `innounp` and `aria2`
  are optional helpers Scoop pulls in on demand.

## Non-negotiable rules

- **Never guess the payload.** Download the installer and inspect its real
  layout before writing the manifest. Guesswork about "portable", the main
  executable, or a nested folder is the main source of broken apps.
- **Never invent a hash.** Compute it locally (SHA-256), or take a published
  checksum from the vendor and verify it against the downloaded file.
- Ship `checkver` and `autoupdate` with every manifest; without them Excavator
  cannot keep the app fresh.
- Record non-obvious decisions in the `##` field (string or array), not in prose
  outside the manifest.
- Change one manifest per commit. Use native `git` and Conventional Commits;
  do not add AI signatures or co-author lines.

## Workflow

### 1. Establish upstream facts

Collect the exact version, direct download URL pattern, checksum, architecture
and license. Prefer a vendor JSON API or the GitHub releases API over scraping
HTML: APIs are stable and often publish the checksum directly.

- Record how the URL is derived from the version. If a token cannot be derived
  (a dated filename, an MD5 in the path, a short release id), capture it with a
  named regex group so `autoupdate` can reuse it.
- Note redirect behaviour. Scoop's `WebClient` downloader does not reliably
  follow 308 redirects; use the final URL or a page that does not redirect.
- Check the license identifier and URL, and whether the vendor publishes
  checksums (MD5, SHA-256, or a GitHub asset `digest`).

### 2. Download and inspect

Download the payload once into a scratch directory and examine it:

- Determine the container: ZIP/7z/tar, 7z SFX (`Type = PE` with an embedded
  `Type = 7z`), NSIS, Inno Setup, MSI, or a plain single-file executable.
- List the archive contents. Note whether files are flat at the root or nested
  under a version directory, and locate the main `.exe`.
- Identify installer artifacts to remove (`$PLUGINSDIR`, `$TEMP`, `$_22_\`,
  updater stubs) and any version-named directories that need renaming.
- Decide `persist` from where the app actually writes user data. Data under
  `%APPDATA%`, `%LOCALAPPDATA%` or `~/.<app>` needs no persistence; only files
  written inside the app directory do.
- Do not add `depends` when the package already bundles its runtime.

### 3. Classify and write the manifest

Start from `bucket/app-name.json.template`, then delete every unneeded field.
See `references/manifest-patterns.md` for the package-type decision tree, the
field reference and hook ordering.

Typical outcomes:

| Payload | Manifest shape |
| --- | --- |
| Flat portable archive | `url`, `hash`, `bin`, `shortcuts`; no hooks |
| Archive with a single top folder | add `extract_dir` |
| 7z self-extracting installer | append `#/dl.7z` to the URL to force 7-Zip |
| NSIS installer extractable by 7-Zip | `#/dl.7z` plus `pre_install`/`post_install` cleanup |
| Inno Setup installer | `"innosetup": true` (Scoop adds `innounp` as a helper) |
| Real GUI installer that cannot be unpacked | `installer` with silent args, or `pre_install`/`post_install` |

Use `bin: [[exe, alias]]` and `shortcuts: [[exe, "Name"]]`; keep the alias equal
to the manifest name so the app is callable from the shell.

### 4. Wire version detection and hash updates

- `checkver`: use `github` with an **API** URL for GitHub releases, otherwise
  `url` + `regex` with named groups. A fully named-group regex has no numbered
  group 1; `checkver` then falls back to the `version` group, so always include
  a `version` group.
- `autoupdate.url`: substitute `$version`, `$underscoreVersion`, `$dashVersion`,
  `$cleanVersion`, or `$match<GroupName>` for captured tokens.
- `autoupdate.hash`: prefer a zero-download source (vendor JSONPath, extract
  mode, GitHub asset digest; see `references/manifest-patterns.md`). Omit
  `autoupdate.hash` entirely for GitHub release assets so Scoop reads the API
  `digest`.
- Keep the top-level `hash` as the locally computed SHA-256 of the exact file
  described by `url`, including any `#/...` fragment (the fragment is stripped
  before downloading).

### 5. Validate

Run from the repo root, replacing `<app>` with the manifest name:

```pwsh
.\bin\formatjson.ps1 -App <app>
.\bin\checkver.ps1 -App <app>
.\bin\checkurls.ps1 -App <app>
.\bin\checkhashes.ps1 -App <app> -Update
.\bin\test.ps1
scoop download .\bucket\<app>.json -f -u
```

`bin/test.ps1` requires Pester 5.2.0 and BuildHelpers 2.0.1; run it with
`scoop prefix scoop` resolvable. Verify the autoupdate path on a throwaway copy
so the real manifest is never mutated by a probe:

```pwsh
Copy-Item .\bucket\<app>.json .\bucket\<app>-probe.json
.\bin\checkver.ps1 -App <app>-probe -Update -ForceUpdate
# inspect the expanded URL and hash in the probe, then
Remove-Item .\bucket\<app>-probe.json
```

Finish with file hygiene and a diff check: no BOM, a single trailing newline,
CRLF checkout consistent with `.gitattributes`, and `git diff --check`.

`scoop download -f -u` is the end-to-end proof that the URL resolves and the
hash matches. Run `scoop install` only when explicitly requested; downloaded
archives remain in the Scoop cache and can be inspected in place.

Known false alarms and workarounds live in `references/troubleshooting.md` —
consult it before changing a working manifest to satisfy a tool warning.

### 6. Commit

```pwsh
git add bucket/<app>.json
git diff --cached --check
git commit -m "feat: add <app>" -m "- Describe what was done."
```

Use native `git` for this bucket. Follow Conventional Commits with a body; keep
the prefix set `feat | fix | ci | refactor | chore | docs | test`, and never add
AI signatures. If signing hangs with no output in a non-interactive shell, do
not retry the commit and do not change the signing configuration: report the
state and let the user finish in an interactive terminal with
`git commit -F .git/COMMIT_EDITMSG`. The staged files and the commit message
file are preserved.

## Resources

- `references/manifest-patterns.md` — package-type decision tree, field
  reference, checkver/autoupdate variables, hash strategies, anti-patterns.
- `references/troubleshooting.md` — known traps (Referer 403, 7z SFX,
  redirects, GitHub API vs HTML, Inno Setup, checkurls, Pester prerequisites,
  GPG pinentry) with the Scoop source files that explain each behaviour.
