# Manifest patterns

Reference material for `scoop-bucket`. Load when classifying a payload,
choosing manifest fields, or designing `checkver`/`autoupdate`.

This file holds general patterns only. For concrete examples, read the live
manifests in `bucket/` directly — they are the current source of truth and may
have evolved since this skill was written.

## 1. Classify the payload

Run `7z l <file>` (and `7z l -slt <file>` for technical info) against the
downloaded file before writing anything.

| Observation | Type | Manifest approach |
| --- | --- | --- |
| `Type = zip`, `7z`, `tar`, `rar`, `iso`, `nupkg` | Archive | Scoop unpacks it by default. Add `extract_dir` only when a single top-level folder wraps everything. |
| `Type = PE` with an embedded `Type = 7z` | 7z self-extracting installer | Append `#/dl.7z` to `url`; Scoop then treats the file as a 7z archive. Usually flat, no hooks. |
| PE, 7-Zip lists NSIS folders (`$PLUGINSDIR`, `$_22_\`) | NSIS | `#/dl.7z` fragment + `pre_install`/`post_install` to rename the version directory and delete temp folders. |
| NSIS with an embedded second-stage archive such as `$PLUGINSDIR\app-64.7z` | NSIS + 7z | `#/dl.7z` for the outer layer, then `installer.script` (or `pre_install`) calls `Expand-7zipArchive` to expand the inner archive into `$dir` and removes `$PLUGINSDIR`/`$R0`. `lib/depends.ps1` adds the 7zip helper automatically when the hook text contains `Expand-7zipArchive `. **Never use `post_install` for this** — see section 2. |
| PE, version resource mentions Inno Setup | Inno Setup | `"innosetup": true`. `lib/depends.ps1` adds `innounp` automatically; Scoop runs `innounp -x -c{app}`. No manual `depends`. |
| `.msi` | Windows Installer | Prefer `msiexec /a` or `lessmsi` to unpack into the app directory; otherwise use `installer` for a real silent install. |
| WiX/Burn, InstallShield, Squirrel, unknown stub | Real installer | Use `installer` with silent `args` (for NSIS: `["/S", "/D=$dir"]`), or `pre_install`/`post_install` for custom logic. |
| `.appx` / `.msix` | Packaged app | Requires `pre_install`/`post_install`; rarely worth it for a personal bucket. |
| Plain `.exe` / `.ps1`, no archive | Single file | No fragment, no hooks; point `bin` at the file. |

If 7-Zip cannot open a PE file, do not assume it is a real GUI installer: newer
Inno Setup versions need `innounp`. Confirm with the PE version resource before
falling back to `installer`.

## 2. Field reference

| Field | Use when | Notes |
| --- | --- | --- |
| `version`, `description`, `homepage`, `license` | Always | `license` is an SPDX string, `"Proprietary"`, or `{ "identifier": ..., "url": ... }`. |
| `url`, `hash` | Always (unless `architecture`) | `hash` is a SHA-256 hex string, or `md5:`/`sha1:`/`sha512:` prefixed. |
| `architecture` | Multiple builds | Nested `64bit` / `32bit` / `arm64` objects, each with its own `url`/`hash`; mirror the structure inside `autoupdate`. |
| `extract_dir` | Single wrapping folder inside an archive | Name the folder, not a path. |
| `extract_to` | Multiple roots must be merged | Rare; prefer `pre_install`. |
| `bin` | Always for apps with a CLI entry | String, array, or `[[exe, alias]]`. Keep the alias equal to the manifest name. |
| `shortcuts` | Desktop/Start Menu entry | `[[exe, "Name"]]`; use the vendor's display name, including CJK characters. |
| `innosetup` | Inno Setup payload | Boolean `true`. |
| `installer` / `uninstaller` | Real silent installers | `file`, `args`, `script`, `keep` as needed. |
| `pre_install` / `post_install` | Layout fixups | PowerShell string or array. `$dir`, `$version`, `$global` are available; escape a literal `$` for PowerShell with a backtick (`` `$ ``). |
| `persist` | User data written inside the app dir | String or array of files/dirs, or `[[source, target]]`. Omit for `%APPDATA%`-style storage. |
| `env_add_path` | CLI tools need PATH | Point at a directory inside the app dir. |
| `depends` | Missing runtime | Do not add when the archive bundles its own runtime. |
| `notes` | Licensing / caveats | String or array; shown after install. |
| `##` | Any non-obvious decision | String or array of comments; ignored by Scoop. |
| `checkver`, `autoupdate` | Always | See sections 3 and 4. |

Delete every unused field from the template. Empty strings and empty arrays
are not harmless: they are validated by `schema.json`.

### 2.1 Hook order matters (shims are created before `post_install`)

Scoop's `lib/install.ps1` runs `install_app` in this order:

```
Invoke-Extraction            (line 53)  unpack the outer archive
pre_install                  (line 54)
Invoke-Installer             (line 56)  runs installer.script
link_current                 (line 58)
create_shims                 (line 59)  <- bin entries must exist by now
create_startmenu_shortcuts   (line 60)  <- shortcut targets must exist by now
post_install                 (line 69)  <- too late for layout changes
```

So any hook that **creates the main executable or changes the layout `bin`/
`shortcuts` point at** must live in `pre_install` or `installer.script`.
`post_install` is only safe for work that does not affect shims (writing
`notes`, post-install config, cleanup of files nothing references).

Getting this wrong fails the very first install with:

```
Get-Command: ... lib/install.ps1:187
The term '<main>.exe' is not recognized ...
Can't shim '<main>.exe': File doesn't exist.
```

This is the classic failure for NSIS installers whose real app is inside an
embedded archive: after the outer `#/dl.7z` extraction the app dir contains only
NSIS leftovers, so `bin` points at a file that does not exist yet. Prefer
`installer: { "script": [...] }`, which is the pattern used for these payloads.

`lib/depends.ps1` includes `$pre_install + $installer.script + $post_install`
in its helper detection, so `Expand-7zipArchive ` in `installer.script` still
pulls in the 7zip helper automatically.

## 3. checkver and autoupdate variables

`checkver` variants:

- `{ "github": "https://api.github.com/repos/<owner>/<repo>/releases/latest" }` —
  uses `$.tag_name` with the default regex `(?:v|V)?([\d.-]+)`. Use the **API**
  URL, not the `github.com/.../releases` HTML page.
- `{ "url": <page or API>, "regex": <pattern> }` — named groups become
  `$match<GroupName>` in `autoupdate`. Always include a `(?<version>...)` group.
- `{ "url": ..., "jsonpath": ..., "reverse": true, "replace": ... }` — for
  JSON endpoints that need shaping.
- `{ "url": ..., "xpath": ... }` — for XML feeds.

`bin/checkver.ps1` prefers the numbered group `1`, and falls back to the
`version` group when it is empty. A regex made entirely of named groups returns
`GetGroupNames()` = `0, xid, version` (no `1`), which is safe. Do not rely on a
numbered group and named groups at the same time.

Substitution tokens available in `autoupdate.url` / `autoupdate.hash`:

| Token | Meaning |
| --- | --- |
| `$version` | New version string |
| `$underscoreVersion` | Dots replaced by underscores (for example `10_6_0`) |
| `$dashVersion`, `$cleanVersion` | Dash-separated / cleaned variants |
| `$match<GroupName>` | Captured named group, capitalized (`(?<xid>...)` → `$matchXid`) |
| `$matchMd5`, `$matchFilename`, `$matchXid` | Common real-world captures |
| `$basename`, `$url`, `$sha256` | Used inside `hash` extract rules |

Every variable used in `autoupdate` must be produced by `checkver`. Verify by
running the probe described in `SKILL.md`; a typo silently produces a literal
`$matchFoo` in the URL.

Multi-line matching: `bin/checkver.ps1` reads the page as one string, so
`[\s\S]*?` is the correct way to span lines. Scoop's autoupdate helper escapes
the substituted filename with `Regex.Escape` before use.

## 4. Hash strategies

Goal: keep `autoupdate` able to produce a valid hash without downloading a
500 MB installer.

| Strategy | Manifest | When to use |
| --- | --- | --- |
| Vendor checksum via JSONPath | `"hash": { "url": <api>, "jp": "$.versions.Windows_x64.hash" }` | API publishes MD5/SHA-256. `format_hash` adds `md5:` automatically for 32 hex chars. |
| Extract from a page | `"hash": { "mode": "extract", "url": <page>, "find": "$matchFilename[\\s\\S]*?$sha256" }` | Vendor prints checksums in HTML. `$sha256` becomes a capture group. |
| GitHub asset digest | Omit `autoupdate.hash` entirely | `url` is a `github.com/.../releases/download/...` asset and no `hashfile_url` is set. `get_hash_for_app` enters "Github Mode", reads the asset `digest` (`sha256:...`) from the API, and strips the prefix. |
| Recompute locally | Omit `autoupdate.hash` for non-GitHub, or use `bin/checkhashes.ps1 -Update` | Last resort; Excavator downloads the file to hash it. |

The top-level `hash` is always the SHA-256 of the current file. When `url` ends
with a `#/...` fragment, Scoop strips the fragment before downloading, so the
hash still covers the original binary.

For a plain `md5sums`-style text file, use `mode: extract` with
`"find": "$md5 \\*$basename"`. `$md5`, `$sha1`, `$sha256`, `$sha512` and
`$checksum` expand into hex capture groups; `$basename` is regex-escaped, so it
is the safe way to anchor on one line of a multi-platform checksum list.
`format_hash` prefixes a 32-char result with `md5:`.

See `troubleshooting.md` for what to do when the vendor rebuilds an archive
under an unchanged version number — `checkver` cannot detect that case.

## 5. Anti-patterns

- Writing `bin` as a flat `["App.exe", "app"]` array when an alias is wanted.
  Scoop iterates the array and treats **each string as a separate shim target**,
  so it creates `App` and `app` shims; the second target is resolved by
  `Get-Command` against the shim that was just created, so the shim ends up
  pointing at itself. Symptoms: `An object at the specified path ...app.shim.
  does not exist`, `WARN Overwriting shim ('app.exe' -> 'app.exe')`, and an
  `app.shim` whose `path` is inside `~\scoop\shims`. The correct form is
  nested: `[[ "App.exe", "app" ]]`. Verify after install with `scoop which app`
  and by reading `~\scoop\shims\app.shim`.
- Adding `persist` "just in case" — it changes upgrade semantics and can pin
  stale configuration.
- Adding `depends` for a runtime that is already bundled.
- Pointing `bin` at an updater stub or launcher instead of the real main exe.
- Using a GitHub HTML releases page for `checkver.github` instead of the API.
- Hard-coding a version-derived token (date, md5, short hash) instead of
  capturing it with a named group.
- Leaving `autoupdate.hash` configured when the vendor API or GitHub digest can
  provide it for free.
- Trusting `checkurls` failures without checking whether the tool itself sets a
  Referer header (see `troubleshooting.md`).
