# Troubleshooting and known traps

Reference material for `scoop-bucket`. Load when a validation tool disagrees
with an otherwise working download, or when a download/signing step hangs.

This file holds general traps only. Do not hard-code app names, vendor URLs or
version numbers here; read the live manifests in `bucket/` for concrete cases.

## Referer-based 403 from a CDN

Some CDNs return 403 when the request carries a same-origin Referer, and only
accept an empty Referer or the vendor's own page.

Both Scoop download paths set a Referer:

- `lib/download.ps1` → `Start-Download`: `$wreq.Referer = strip_filename $url`
- `Invoke-CachedAria2Download`: writes `referer=$(strip_filename $url)` into the
  aria2 input file

Installing `aria2` therefore does **not** remove the header. The global
`--referer` option is overridden by the input file; only
`--header=Referer:` can override it, but that is a global setting with side
effects and should not be used.

Scoop already skips the Referer when the URL contains `sourceforge.net`:
`Start-Download` excludes `sourceforge\.net|portableapps\.com`, and the aria2
path uses `$url.Contains('sourceforge.net')`. Encode the marker as a fragment:

```json
"url": "https://cdn.example.com/<file>.exe#/sourceforge.net/dl.7z"
```

`url_filename` becomes `dl.7z` (which also forces 7z unpacking), and
`strip_fragment` restores the vendor's exact URL for the HTTP request — no
extra query parameters. Do **not** put the marker in a query string
(`?noreferer=sourceforge.net#/dl.7z`): `url_remote_filename` treats it as the
filename and the CDN receives an altered query. Add a `##` comment explaining
the marker.

`bin/checkurls.ps1` sets a Referer unconditionally and may still report 403 for
such URLs. CI runs `bin/test.ps1`, not `checkurls`, so trust the end-to-end
`scoop download .\bucket\<app>.json -f -u` result instead of rewriting a
working manifest to silence that tool.

## 7z self-extracting installers

A PE file with an embedded 7z archive is not detected by extension. Append
`#/dl.7z` so Scoop hands it to 7-Zip. Inspect the extracted layout anyway:
some of these bundles are flat, while others are NSIS payloads full of
`$PLUGINSDIR`, `$TEMP`, `$_22_\` and updater stubs that need `pre_install` /
`installer.script` cleanup. Some NSIS bundles ship both architectures in one
installer (for example `app64.7z` and `app32.7z`). Expand only the payload
matching the manifest's architecture, delete the unused one plus `$PLUGINSDIR`,
and do not add an `architecture` block when both payloads come from the same
URL — state the 64-bit-only choice in `##`. Do **not** defer the expansion to
`post_install`: Scoop creates shims and Start Menu shortcuts first, so it fails
with `Can't shim '<main>.exe': File doesn't exist`. See
`manifest-patterns.md` section 2.1 for the exact hook order.

## Hash check failed on an unchanged version (silent rebuild)

Before assuming a corrupt download, prove which side changed:

1. `7z t <file>` — a valid EOCD and "Everything is Ok" rules out truncation.
2. `sha256sum` the cached/downloaded file, then re-download independently with
   `curl -L -o <temp>` and hash again. Two matching hashes = upstream changed.
3. Read `7z l <file>` timestamps. If the entries are dated *after* the manifest
   was authored but `checkver` still reports the same version, the vendor
   re-published the archive in place without bumping the version (common with
   install4j/Electron builds).
4. Cross-check against the vendor's published checksum list, if one exists. An
   exact match confirms authenticity, not just stability.

Then update the top-level `hash` to the locally computed SHA-256 and record the
reason in `##`.

Prevention is limited: `checkver`/Excavator only fire on a version change, so a
same-version rebuild is invisible to scheduled autoupdate and will break
installs until someone runs `bin/checkhashes.ps1 -App <app> -Update`. Say so in
`##` rather than implying autoupdate will catch it. Adding `checkhashes` to CI
is usually not worth it — it re-downloads every payload in the bucket.

What *can* be improved is the version-bump path. If the vendor publishes a
checksum list, wire it into `autoupdate.hash` so Excavator does not download a
hundreds-of-MiB installer just to hash it:

```json
"hash": {
    "mode": "extract",
    "url": "https://vendor.example/download/checksums",
    "find": "$md5 \\*$basename"
}
```

`$md5`/`$sha1`/`$sha256` become capture groups and `$basename` is
`Regex.Escape`d by `substitute`, so anchoring on it avoids grabbing the
checksum of a sibling `.exe`/`.msi` or another platform from the same file.
`format_hash` turns a 32-char result into `md5:<hex>`; a probe should print
`Found: md5:... using Extract Mode`. Keep the top-level `hash` as SHA-256 —
both formats are valid and they need not match.

## "Can't shim '<exe>': File doesn't exist" on a fresh install

Scoop creates shims (`lib/install.ps1:59`) and Start Menu shortcuts (`:60`)
before `post_install` (`:69`). If the manifest expands a second-stage archive
in `post_install`, the app dir at shim time still holds only NSIS leftovers
(`$PLUGINSDIR`, `$R0`, `uninstallerIcon.ico`) and `Get-Command <exe>` at
`lib/install.ps1:187` throws before the hook ever runs.

Move that `Expand-7zipArchive` (and any layout fixups) into
`installer: { "script": [...] }` or `pre_install`. Confirm the target exe name
from the inner archive with `7z l "$PLUGINSDIR\app-64.7z"` — do not assume it
matches the uninstaller or the installer filename. The `bin` name itself is
usually correct; only the hook ordering is wrong.

Note: 7-Zip prints CJK filenames in the console OEM codepage (GBK on zh-CN
Windows). Pipe the listing through `iconv -f GB18030 -t UTF-8` before matching.

## Redirects and URL stability

Scoop's `WebClient` downloader does not reliably follow 308 redirects. Use the
final URL, or a page that does not redirect: when a download endpoint returns
308, keep `checkver.url` on a stable HTML page while the download URL itself
stays on the original path.

## checkver regex groups

`bin/checkver.ps1` reads the version from `$matchesHashtable['1']` and falls
back to the `version` group when that is empty. A regex built only from named
groups returns `GetGroupNames()` = `0, xid, version`, so it reliably uses the
named `version` group. Avoid mixing numbered and named groups. Use `[\s\S]*?`
to span lines; the page is matched as a single string.

## Version sources inside inline page state

SPA download pages often embed a serialized state object (Nuxt/Next) instead of
a visible version label. A bare `[\d.]+` regex will latch onto a gray release,
a library version or an unrelated product version. Anchor on the exact key and
bound the match:

```json
"checkver": {
  "url": "https://vendor.example/download.html",
  "regex": "<exact-key>:.{0,120}?<AppName>_(?<version>[\\d.]+)\\.exe"
}
```

Verify with .NET regex that `Matches()` returns exactly one hit. State URLs are
escaped as `https:\u002F\u002F...`, so do not hard-code that prefix in the
regex.

Before reusing a "lite"/portable binary, check its PE version: a stripped-down
single-file build can report a lower `FileVersion` than the main installer.
Pairing a fixed unversioned Lite URL with the main-line version would mislabel
the package.

## GitHub endpoints and proxies

- `checkver.github` accepts `api.github.com/repos/<owner>/<repo>/releases/latest`
  and reads `$.tag_name`; a repository HTML page is scraped instead and is more
  fragile.
- A local proxy may fail `CONNECT github.com` with 502 while
  `api.github.com` works. Query the API and, when a direct asset download is
  needed, use
  `api.github.com/repos/<owner>/<repo>/releases/assets/<id>` with
  `Accept: application/octet-stream` (it 302s to the release CDN). Keep the
  standard `github.com/.../releases/download/...` URL in the manifest.
- Omitting `autoupdate.hash` activates "Github Mode", which reads the asset
  `digest` from the API. Confirm with a probe that prints `using Github Mode`.

## Inno Setup

Recent Inno Setup installers may not open in 7-Zip (`Cannot open the file as
archive`); `innounp` handles them. Set `"innosetup": true` and let
`lib/depends.ps1` add `innounp` as a helper — no manual `depends` entry.
Confirm the installer type from the PE version resource before falling back to
`installer`.

## Validation toolchain limits

- `bin/test.ps1` requires Pester 5.2.0 and BuildHelpers 2.0.1. A machine with
  only Pester 3.4.0 and no BuildHelpers cannot run it; CI provides both.
- The official schema validator's `Add-Type` call may be blocked by a local
  security policy. Fall back to a field-level check against Scoop's
  `schema.json` (required fields, types, no empty placeholders) and state the
  limitation honestly.
- `markdownlint-cli` may be absent and `npx` may be offline; check headings,
  fenced-language tags, blank lines and bare URLs with a script instead.
- `aria2` occasionally drops the first connection; rerun the download once
  before treating it as a URL problem.
- `scoop download -f -u` can fail when the machine's configured aria2 proxy is
  unusable: aria2 reports `Download aborted` / `0B/s`, Scoop falls back to
  `Start-Download`, and hash checking then fails because the expected cache file
  was never written. Treat this as a local download-path problem, not a manifest
  defect, when `checkurls` passes and both `bin/checkhashes.ps1 -Update` and an
  autoupdate probe independently download the file and compute the same hash.
  Do not change global proxy or aria2 settings to work around it; report the
  blocker and cross-check with a direct `curl` download plus `sha256sum`.
  For a clean end-to-end check without touching the global config, point
  `XDG_CONFIG_HOME` at a throwaway directory holding `scoop/config.json` with
  `{ "aria2-enabled": false }`. `lib/core.ps1` loads the config from that path,
  so only that invocation skips aria2 and ignores the stale `proxy` key:

  ```bash
  XDG_CONFIG_HOME='C:\Users\<user>\AppData\Local\Temp\cfg' \
    scoop download .\bucket\<app>.json -f -u
  ```

  Confirm with `scoop config` afterwards that the original `proxy` value is
  still present, and tell the user that `scoop install` will hit the same aria2
  failure until the configured proxy is fixed.

## CDN User-Agent blacklists

Some CDNs reject curl's default UA with 403 `denied by UA ACL = blacklist`,
while browser, empty and Scoop UAs pass. Do not design a manifest workaround
around the UA of the inspection tool: confirm with `bin/checkurls.ps1` and
`scoop download -f -u`, which use Scoop's own UA. Compare with
`curl -A <browser>` and `curl -A ""`. An OSS multipart ETag ending in `-N` is not
an MD5 and cannot be reused as a hash source.

## UTF-8 manifests and Windows PowerShell 5.1

`Get-Content -Raw` decodes UTF-8 as ANSI, turning CJK text into mojibake that can
even break `ConvertFrom-Json`. Read with
`[System.IO.File]::ReadAllText($path, [System.Text.Encoding]::UTF8)` or verify
with Python. Scoop's `parse_json` reads UTF-8 correctly, so `formatjson.ps1` and
`checkver.ps1` are unaffected; `shim` writes `.shim` files as UTF-8, so a CJK
main executable name is safe in `bin` and `shortcuts`.

## Validating hook scripts without installing

When a hook cannot be executed end to end, validate statically: parse it with
`[System.Management.Automation.Language.Parser]::ParseInput`, resolve escaped
paths with `Test-Path -LiteralPath`, and confirm helper auto-detection with
`$script -like '*Expand-7zipArchive *'`. A literal `$` in a hook must be escaped
with a backtick (`` `$ ``), otherwise PowerShell expands `$PLUGINSDIR` to an
empty variable. Report that the hook was statically verified, not executed.

## File hygiene

`.gitattributes` stores LF in Git and checks out CRLF on Windows. Manifests
should be UTF-8 without BOM, formatted by `bin/formatjson.ps1` (4-space
indent), with exactly one trailing newline. Always run `git diff --check` and
inspect `git diff` before committing. Delete temporary `*-probe.json` files.

## GPG signing hangs

Two GnuPG installations can coexist on Windows. Edit `gpg-agent.conf` in the
homedir of the binary that `git config gpg.program` points to; the other one has
no effect on commits. Reload with `gpgconf --reload gpg-agent` and verify with
`gpgconf --list-options gpg-agent`. `default-cache-ttl` and `max-cache-ttl` must
both be raised (for example to `86400`), because the latter caps the former at
2 hours by default. The cache starts at the first passphrase entry and is
cleared when the agent restarts.

If `git commit` hangs with no output while signing, the GUI pinentry
(`pinentry-w32`) dialog is invisible to a background shell. With an unset
`pinentry-timeout` the wait is unlimited. Add `pinentry-timeout 60` to fail
fast. Do not retry the commit repeatedly and do not disable signing: the staged
files and `.git/COMMIT_EDITMSG` are intact, so report the state and let the user
finish in an interactive terminal with `git commit -F .git/COMMIT_EDITMSG`.

## Windows download quirks

Native `curl -o` can resolve relative output paths unexpectedly, and raw API
requests can time out behind a proxy. Prefer shell redirection or an absolute
path resolved in Python. Use absolute paths for all tool arguments.
