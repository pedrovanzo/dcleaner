# dcleaner: First Run (AI assisted)

Log of the first, manual, AI-assisted sweep (Claude Code) of Node.js / npm / Yarn temporary and dependency files on a Windows 11 workstation. It records every scan, every cleanup and every disk usage snapshot, plus the lessons that should shape the future dcleaner agent.

- **Date:** 2026-11-08
- **OS:** Windows 11 (single drive, `C:`)
- **Tools found:** `node`, `npm`, `npx`, `yarn` (v1 Classic). Not installed: `pnpm`, `bun`.
- **Shells used:** Git Bash (POSIX) and Windows PowerShell 5.1
- **Result:** about **29.5 GB** freed. Free space on `C:` went from **111.31 GB** to **140.98 GB**.

## Redaction notes

This repo is public, so identifying details were replaced with placeholders:

| Placeholder | Meaning |
|---|---|
| `<user>` | Current Windows profile name (`C:\Users\<user>`) |
| `<old-user>` | Profile restored from a backup of an old PC (`C:\Users\<old-user>`) |
| `O01`–`O29` | Project folders inside the old-PC backup |
| `C01`–`C23` | Project folders inside the current workspace |
| `<machine>` | Computer name |

Project names were removed because they can reveal clients, people or private work. Only the location category is kept (Desktop, Documents, OneDrive, Downloads). Account SIDs and machine names were omitted. Public tool and app names (npm, Yarn, VS Code, nvm…) were kept.

---

## Phase 1: Discovery scan

### 1.1 Tools, cache locations and drives

```bash
ls /
for t in node npm yarn pnpm bun npx; do printf "%s: " $t; command -v $t >/dev/null && echo found || echo missing; done
npm config get cache      # -> C:\Users\<user>\AppData\Local\npm-cache
yarn cache dir            # -> C:\Users\<user>\AppData\Local\Yarn\Cache\v6
pnpm store path           # -> (pnpm not installed)
```

```powershell
Get-PSDrive -PSProvider FileSystem | Select-Object Name, @{n='UsedGB';e={[math]::Round($_.Used/1GB,1)}}, @{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}
```

| Drive | Used | Free |
|---|---|---|
| C: | 351.3 GB | 113.4 GB |

### 1.2 Finding every `node_modules` folder

Top-level `node_modules` folders only. Nested ones are pruned. `C:\Windows`, `$Recycle.Bin` and `C:\ProgramData\Microsoft` are skipped.

```bash
find /c -path /c/Windows -prune -o -path '/c/$Recycle.Bin' -prune -o -path /c/ProgramData/Microsoft -prune \
  -o -type d -name node_modules -print -prune 2>/dev/null > /tmp/dc_nm.txt
wc -l < /tmp/dc_nm.txt
awk -F/ '{ if ($0 ~ /AppData/) k="AppData"; else if ($0 ~ /Program Files/) k="ProgramFiles"; else k=$2"/"$3"/"$4; c[k]++ } END { for (k in c) print c[k], k }' /tmp/dc_nm.txt | sort -rn
```

| Area | `node_modules` folders |
|---|---|
| AppData (app-bundled, extensions, caches, nvm) | 2973 |
| `C:\Users\<old-user>` | 38 |
| `C:\Users\<user>` | 31 |
| Program Files | 5 |
| **Total** | **3047** |

**Token cost decision:** listing all 3047 paths in chat would cost about 90–100k tokens (about 30 tokens per path line). Instead, results were grouped and only the project folders were listed one by one.

### 1.3 First sizing attempt (failed)

Three `du`-based measurements were started in parallel:

```bash
# project node_modules
grep -v -e AppData -e 'Program Files' /tmp/dc_nm.txt | while read -r d; do du -sk "$d"; done | sort -rn
# AppData / Program Files node_modules grouped by app
grep -e AppData -e 'Program Files' /tmp/dc_nm.txt | while read -r d; do s=$(du -sk "$d" | cut -f1); k=$(echo "$d" | sed -E 's#^(/c/Program Files[^/]*/[^/]+).*#\1#; s#^(/c/Users/[^/]+/AppData/[^/]+/[^/]+(/[^/]+)?).*#\1#'); echo "$s|$k"; done \
  | awk -F'|' '{s[$2]+=$1; c[$2]++} END {for (k in s) printf "%8.1f MB  %4d dirs  %s\n", s[k]/1024, c[k], k}' | sort -rn
# known JS cache directories
for p in "$LOCALAPPDATA/npm-cache" "$LOCALAPPDATA/npm-cache/_npx" "$LOCALAPPDATA/npm-cache/_cacache" ... ; do [ -e "$p" ] && du -sk "$p"; done
```

All three ran past the 10-minute timeout. Claude Code then stopped them because the system was **critically low on memory**. Git Bash also hit process limits (`fork: Resource temporarily unavailable`), so some sizes came back as 0.

Partial results:

| Size | Path | Note |
|---|---|---|
| 8415.1 MB | `AppData\Local\npm-cache` | total |
| 7151.3 MB | `…\npm-cache\_cacache` | package tarballs |
| 1242.4 MB | `…\npm-cache\_npx` | npx packages |
| 0.1 MB | `…\npm-cache\_logs` | |
| 1849.1 MB | `AppData\Local\nvm\v20.10.0` | Node install, **not temp** |
| 243.6 MB | `AppData\Local\Microsoft\TypeScript` | VS Code TS cache (partial) |
| 132.3 MB | `AppData\Local\Programs\Microsoft VS Code` | app-bundled, don't touch |
| 40.4 MB | `C:\Program Files\Microsoft Visual Studio` | app-bundled, don't touch |
| 39.9 MB | `AppData\Local\Yarn\Cache` | partial |
| 29.5 / 17.9 / 13.1 MB | `AppData\Local\nvm\v16.9.0`, `v22.13.0`, `v18.0.0` | Node installs, **not temp** |

**Change of approach:** stop measuring everything at once. List the candidates first, then measure and clean one item at a time.

---

## Phase 2: Candidate list

```bash
grep -v -e AppData -e 'Program Files' /tmp/dc_nm.txt
for p in "$LOCALAPPDATA/Yarn/Cache" "$LOCALAPPDATA/Yarn/Berry" "$USERPROFILE/.yarn" "$LOCALAPPDATA/node-gyp/Cache" \
         "$LOCALAPPDATA/Temp/node-compile-cache" "$USERPROFILE/.npm" "$APPDATA/npm-cache"; do [ -e "$p" ] && echo "exists: $p"; done
```

**Global caches:**
- `C:\Users\<user>\AppData\Local\npm-cache`
- `C:\Users\<user>\AppData\Roaming\npm-cache`
- `C:\Users\<user>\AppData\Local\Yarn\Cache`
- `C:\Users\<user>\AppData\Local\node-gyp\Cache`
- `C:\Users\<user>\AppData\Local\Temp\node-compile-cache`

**Project `node_modules`:** 23 in the current workspace and 29 in the old-PC backup (see Phase 4).

**Excluded on purpose (not candidates):**
- `nvm\v*`: these are Node installs.
- `node_modules` inside extensions of VS Code, Trae, Antigravity and Visual Studio, and the one inside a Python `.venv` (gradio). The apps need them.
- The 9 extension `node_modules` under `C:\Users\<old-user>\.vscode\extensions` were also set aside. They are dead copies from the old PC.

---

## Phase 3: Global caches

Theory check before the first deletion. Deleting the npm cache does **not** affect installed projects, global packages (`AppData\Roaming\npm`), Node, nvm, `.npmrc` or login tokens. Costs: the next installs download everything again (slower, needs internet), `npx` tools download again, old logs are lost, and offline installs fail until the cache refills. Rare risk: a package unpublished from the registry can't be fetched again.

Disk usage was read with:

```powershell
$d = Get-PSDrive C; "Used: {0:N2} GB  Free: {1:N2} GB" -f ($d.Used/1GB), ($d.Free/1GB)
```

### 3.1 `AppData\Local\npm-cache` (cleaned)

Pre-checks: listed the contents and checked for running npm/npx processes (none).

```powershell
Get-ChildItem $p -Force
Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -match 'npm-cli|npx-cli|\bnpm\b|\bnpx\b' -and $_.Name -match 'node|npm|npx' }
```

Contents: `_cacache`, `_libvips`, `_logs`, `_npx`, `_prebuilds`, `_update-notifier-last-checked`. All are disposable.

Cleanup (contents removed, folder kept):

```powershell
Get-ChildItem $p -Force | ForEach-Object { Remove-Item $_.FullName -Recurse -Force -Confirm:$false -ErrorAction Stop }
```

Result: 0 errors, 0 items left.

| Drive C: | Used | Free |
|---|---|---|
| Before | 353.41 GB | 111.31 GB |
| After | 345.22 GB | 119.50 GB |
| **Change** | **−8.19 GB** | **+8.19 GB** |

### 3.2 `AppData\Roaming\npm-cache` (removed)

Measured per subfolder:

```powershell
Get-ChildItem $p -Force | ForEach-Object { (Get-ChildItem $_.FullName -Recurse -Force -File | Measure-Object Length -Sum).Sum }
```

| Size | Item | Modified |
|---|---|---|
| 14.86 MB | `_libvips` | 2025-08-29 |
| 0.61 MB | `_prebuilds` | 2025-08-29 |
| **15.47 MB** | **total (4 files)** | |

npm doesn't use this folder: its active cache is `AppData\Local\npm-cache`. Probably left behind by a one-off native or `sharp` install. The whole folder was removed:

```powershell
Remove-Item $p -Recurse -Force -Confirm:$false -ErrorAction Stop
```

| Drive C: | Used | Free |
|---|---|---|
| Before | 345.23 GB | 119.50 GB |
| After | 345.21 GB | 119.52 GB |
| **Change** | **−15.48 MB** | **+15.48 MB** |

The delta comes from free-bytes counts: 128,329,875,456 − 128,313,643,008 = 16,232,448 bytes. The script's first delta printed 0.00 MB because PowerShell re-reads a drive's free space every time it's accessed, which made the stored "before" value stale. Fix: store `.Free` in a variable right away.

### 3.3 `AppData\Local\Yarn\Cache` (cleaned)

Measured with an **8 GB ceiling**. The script walks the folder one file at a time and stops if the running total reaches 8 GB. It uses little memory and doesn't follow links (reparse points).

```powershell
$cap=8GB; $sum=0; $stack=New-Object System.Collections.Stack; $stack.Push((New-Object IO.DirectoryInfo $root))
while ($stack.Count -gt 0 -and -not $hit) { $d=$stack.Pop()
  foreach ($f in $d.EnumerateFiles()) { $sum+=$f.Length; if ($sum -ge $cap) { $hit=$true; break } }
  foreach ($s in $d.EnumerateDirectories()) { if (-not ($s.Attributes -band [IO.FileAttributes]::ReparsePoint)) { $stack.Push($s) } } }
```

Result: **4.42 GB** in 87,922 files, under the ceiling, in 5 seconds. One subfolder, `v6` (Yarn v1), last modified 2025-12-08.

Cleanup took **two passes**. The first `Remove-Item` failed on a path longer than Windows' 260-character limit and left 41,284 items. The retry with the long-path prefix `\\?\` finished the job:

```powershell
Get-ChildItem -LiteralPath "\\?\$p" -Force | ForEach-Object { Remove-Item -LiteralPath $_.FullName -Recurse -Force -Confirm:$false -ErrorAction Stop }
```

Result: 0 items left. The `Cache` folder was kept.

| Drive C: | Used | Free |
|---|---|---|
| Before | 345.21 GB | 119.52 GB |
| After | 340.63 GB | 124.10 GB |
| **Change** | **−4.58 GB** | **+4.58 GB** |

The first pass freed 3.96 GB and the retry freed 639.90 MB.

### 3.4 `AppData\Local\node-gyp\Cache` (cleaned)

| Size | Item | Modified |
|---|---|---|
| 27.30 MB | `16.9.0` | 2025-06-24 |
| 51.09 MB | `20.10.0` | 2025-06-02 |
| **78.38 MB** | **total (2,856 files)** | |

These are Node headers used to compile native modules. They download again on the next native build. Pre-check: no `node-gyp` processes running. Contents removed with the `\\?\` method from 3.3, 0 errors.

| Drive C: | Used | Free |
|---|---|---|
| Before | 340.63 GB | 124.10 GB |
| After | 340.55 GB | 124.18 GB |
| **Change** | **−83.84 MB** | **+83.84 MB** |

### 3.5 `AppData\Local\Temp\node-compile-cache` (skipped)

| Size | Item | Modified |
|---|---|---|
| 0.00 MB | `v24.21.0-x64-…` | 2026-11-01 |

The folder is empty. A tool that ships its own Node 24 is using it (none of the nvm versions is 24). Nothing to gain, so it was skipped.

---

## Phase 4: Old-PC backup project `node_modules` (`C:\Users\<old-user>`)

### 4.1 Last-modified dates (all 52 project folders, oldest first)

```bash
grep -v -e AppData -e 'Program Files' -e '/\.vscode/' -e '/\.trae/' -e '/\.antigravity/' -e '/\.venv/' /tmp/dc_nm.txt \
  | while IFS= read -r d; do printf '%s  %s\n' "$(stat -c '%y' "$d" | cut -c1-10)" "$d"; done | sort
```

| Date | Folders |
|---|---|
| 2024-08-05 | O01 |
| 2025-01-06 | O02–O29 (28 folders, same day: the date of the backup copy, not real activity) |
| 2025-01-19 → 2026-06-24 | C01–C23 (current workspace, see Phase 5) |

### 4.2 Risk check before deleting

Each project was checked for a manifest, lockfiles and markers that make a reinstall harder:

```bash
for f in package.json package-lock.json yarn.lock pnpm-lock.yaml; do [ -f "$p/$f" ] && l="$l $f"; done
grep -qE '"(file|link):' "$p/package.json" && extra="$extra LOCAL-DEP"
[ -d "$p/patches" ] && extra="$extra PATCHES"
grep -qE 'node-sass|"engines"' "$p/package.json" && extra="$extra ENGINE/NODE-SASS"
[ -d "$p/.git" ] && extra="$extra git"
```

| Project | Location | Files found | Markers |
|---|---|---|---|
| O01 | Desktop | package.json only (no lockfile) | |
| O02 | Desktop | pnpm-lock | |
| O03 | Desktop | package-lock | |
| O04 | Desktop | package-lock | |
| O05 | Desktop | package-lock | |
| O06 | Desktop | pnpm-lock | |
| O07 | Desktop | package-lock | |
| O08 | Desktop | pnpm-lock | |
| O09 | Desktop | package-lock | git |
| O10 | Desktop | package-lock | git |
| O11 | Desktop | package-lock | git |
| O12 | Desktop | package-lock | git |
| O13 | Desktop | none (test folder) | |
| O14 | Desktop | package-lock | git |
| O15 | Desktop | none (test folder) | |
| O16 | Desktop | package-lock | git |
| O17 | Desktop | package-lock | |
| O18 | Desktop | package-lock | git |
| O19 | Desktop | package-lock | |
| O20 | Desktop | package-lock + yarn.lock | |
| O21 | Desktop | package-lock | |
| O22 | Documents | yarn.lock | git |
| O23 | Documents | pnpm-lock | git |
| O24 | Documents | package-lock + yarn.lock | ENGINE/NODE-SASS, git |
| O25 | Documents | package-lock | git |
| O26 | Documents | package-lock | |
| O27 | Documents | package-lock | git |
| O28 | Documents | package-lock | |
| O29 | Documents | none (test folder) | |

Risks found:
- **O01:** no lockfile, so a reinstall picks newer versions.
- **O24:** `node-sass` or `engines` pin, so a reinstall probably needs an old Node version and build tools.
- **O02, O06, O08, O23:** pnpm projects, and pnpm isn't installed on this PC.
- **O20, O24:** mixed npm and yarn lockfiles.
- **O13, O15, O29:** test folders with no `package.json`. Each holds only a link to the user's own package in a sibling source folder (O12, O14 and O29's sibling), installed with a local `npm install ../package`.

Conclusion: low risk for an archive backup. No `patches/` folders were found.

### 4.3 Measurement (8 GB cumulative ceiling)

Same streaming method as 3.3, with the `\\?\` prefix. Links are skipped.

| Size | Project | Size | Project |
|---|---|---|---|
| 632.0 MB | O04 | 116.3 MB | O02 |
| 523.4 MB | O16 | 114.7 MB | O28 |
| 499.5 MB | O22 | 111.2 MB | O19 |
| 494.1 MB | O23 | 105.0 MB | O26 |
| 298.1 MB | O11 | 87.1 MB | O03 |
| 290.4 MB | O24 | 69.5 MB | O12 |
| 284.7 MB | O10 | 69.5 MB | O14 |
| 249.8 MB | O09 | 56.3 MB | O18 |
| 225.7 MB | O27 | 26.8 MB | O21 |
| 134.4 MB | O06 | 25.7 MB | O07 |
| 128.3 MB | O08 | 22.5 MB | O05 |
| 123.8 MB | O20 | 20.6 MB | O25 |
| | | 14.3 MB | O17 |
| | | 0.0 MB | O01, O13, O15, O29 |

**Total: 4.61 GB** in 437,087 files, under the ceiling, in 34 seconds. The measuring script silently skipped folders that raised errors, so this total is probably too low (see 4.5).

### 4.4 Cleanup

The first attempt with `cmd /c rd /s /q "\\?\$p"` was **blocked by Claude Code's safety check**: an empty variable would turn the target into `\\`. It was replaced with a .NET recursive delete that:
- validates the list first (exactly 29 paths, each matching `C:\Users\<old-user>\*\node_modules`, none under `.vscode`);
- removes links as links, without entering them;
- uses long paths (`\\?\`) and clears read-only attributes.

```powershell
function Remove-Tree([string]$dir) {
  $di = New-Object IO.DirectoryInfo $dir
  foreach ($e in $di.EnumerateFileSystemInfos()) {
    if ($e -is [IO.DirectoryInfo]) {
      if ($e.Attributes -band [IO.FileAttributes]::ReparsePoint) { [IO.Directory]::Delete($e.FullName, $false) }  # remove link only
      else { Remove-Tree $e.FullName }
    } else {
      if ($e.Attributes -band [IO.FileAttributes]::ReadOnly) { $e.Attributes = 'Normal' }
      $e.Delete()
    }
  }
  $di.Attributes = 'Directory'; [IO.Directory]::Delete($dir, $false)
}
foreach ($p in $list) { Remove-Tree "\\?\$p" }
```

Why not `Remove-Item -Recurse`: in PowerShell 5.1 it can follow directory links and delete the folder they point to.

Result: **29 of 29** folders removed.

| Drive C: | Used | Free |
|---|---|---|
| Before | 340.77 GB | 123.96 GB |
| After | 329.51 GB | 135.22 GB |
| **Change** | **−11.26 GB** | **+11.26 GB** |

### 4.5 Verification

The file counts of the 3 linked source folders were recorded before and after:

| Source folder | Before | After | Explanation |
|---|---|---|---|
| O29's sibling | 2 | 2 | untouched |
| O12 | 11,832 | 174 | O12's own `node_modules` was on the delete list |
| O14 | 11,706 | 48 | O14's own `node_modules` was on the delete list |

Git confirmed that no tracked file was deleted. It was run as a one-off, because the repos are owned by the old account, so the global git config wasn't changed.

```bash
git -c safe.directory='*' -C "$p" status --short
git -c safe.directory='*' -C "$p" ls-files | wc -l
```

The only change git reported was one modified `package-lock.json` in O14, dated 2025-04-06, which existed before this cleanup.

**Unexplained gap:** 11.26 GB was freed but only 4.61 GB was measured. Likely causes: (1) the measuring script silently skipped folders that raised errors, which undercounts; (2) other disk activity happened at the same time (the `Used` value had already moved between operations). No sign was found that anything outside the 29 folders was removed.

---

## Phase 5: Current workspace project `node_modules` (`C:\Users\<user>`)

### 5.1 Last-modified dates

| Date | Project | Location | Date | Project | Location |
|---|---|---|---|---|---|
| 2025-01-19 | C01 | Documents | 2025-12-08 | C12 | OneDrive |
| 2025-02-02 | C02 | Documents | 2025-12-14 | C13 | Desktop |
| 2025-04-06 | C03 | Desktop | 2026-02-28 | C14 | Desktop |
| 2025-04-06 | C04 | OneDrive | 2026-03-10 | C15 | Downloads |
| 2025-04-20 | C05 | OneDrive | 2026-03-26 | C16 | Desktop |
| 2025-08-30 | C06 | OneDrive | 2026-04-26 | C17 | Desktop |
| 2025-09-08 | C07 | OneDrive | 2026-04-26 | C18 | Desktop (C17's `.next` build output) |
| 2025-09-14 | C08 | OneDrive | 2026-06-06 | C19 | Desktop |
| 2025-10-05 | C09 | OneDrive | 2026-06-09 | C20 | Desktop |
| 2025-10-05 | C10 | OneDrive | 2026-06-10 | C21 | Desktop |
| 2025-12-06 | C11 | Desktop | 2026-06-12 | C22 | Desktop (C17's `.next\dev` build output) |
| | | | 2026-06-24 | C23 | Desktop |

C01–C16 were selected for this sweep. **C17–C23 were left alone.**

### 5.2 Measurement (C01–C16)

Improved script. It counts errors (skipped folders), links, and OneDrive cloud-only files separately. Cloud-only files are detected by the `RecallOnDataAccess` / `Offline` attributes. Reading only the metadata doesn't trigger downloads.

| Size | Files | Project |
|---|---|---|
| 580.4 MB | 25,843 | C03 |
| 571.2 MB | 21,066 | C08 |
| 518.5 MB | 27,602 | C16 |
| 476.0 MB | 19,551 | C07 |
| 474.7 MB | 19,315 | C10 |
| 407.5 MB | 19,960 | C13 |
| 384.0 MB | 15,145 | C15 |
| 355.8 MB | 35,882 | C01 |
| 345.2 MB | 34,789 | C02 |
| 343.3 MB | 9,874 | C12 |
| 339.5 MB | 9,534 | C05 |
| 336.9 MB | 17,697 | C04 |
| 240.3 MB | 4,355 | C14 |
| 186.7 MB | 6,079 | C06 |
| 109.8 MB | 6,467 | C11 |
| 105.6 MB | 2,663 | C09 |

**Total: 5.64 GB** in 275,822 files, in 17.6 seconds. Errors: 0. Links: 0. Cloud-only: 0 MB.

### 5.3 Cleanup

Pre-checks:
- exactly 16 paths, each matching `C:\Users\<user>\*\node_modules`;
- no running process with any of the projects in its command line.

Same `Remove-Tree` as 4.4, with free space read before and after **each** folder.

| Freed | Project | Freed | Project |
|---|---|---|---|
| 612.2 MB | C08 | 365.7 MB | C04 |
| 599.0 MB | C03 | 360.8 MB | C12 |
| 564.5 MB | C16 | 356.2 MB | C05 |
| 507.3 MB | C07 | 248.7 MB | C14 |
| 506.2 MB | C10 | 198.0 MB | C06 |
| 440.9 MB | C13 | 110.1 MB | C09 |
| 412.2 MB | C15 | −74.8 MB | C02 (*) |
| 401.7 MB | C01 | −102.0 MB | C11 (*) |

(*) Negative values mean another process wrote to the disk during that deletion. This confirms that other activity on the PC skews the numbers.

Result: **16 of 16** folders removed. All 16 projects still have their `package.json`. Run `npm install` / `yarn` before running them again.

| Drive C: | Used | Free |
|---|---|---|
| Before | 329.13 GB | 135.60 GB |
| After | 323.75 GB | 140.98 GB |
| **Change** | **−5.38 GB** | **+5.38 GB** |

---

## Summary

| # | Target | Action | Freed |
|---|---|---|---|
| 3.1 | `AppData\Local\npm-cache` | contents removed | 8.19 GB |
| 3.2 | `AppData\Roaming\npm-cache` | folder removed | 15.48 MB |
| 3.3 | `AppData\Local\Yarn\Cache` | contents removed (2 passes) | 4.58 GB |
| 3.4 | `AppData\Local\node-gyp\Cache` | contents removed | 83.84 MB |
| 3.5 | `AppData\Local\Temp\node-compile-cache` | skipped (empty) | 0 |
| 4 | 29 old-backup `node_modules` (O01–O29) | removed | 11.26 GB |
| 5 | 16 current-workspace `node_modules` (C01–C16) | removed | 5.38 GB |
| | **Total** | | **≈ 29.5 GB** |

| Drive C: | Used | Free |
|---|---|---|
| Start of run | 353.41 GB | 111.31 GB |
| End of run | 323.75 GB | 140.98 GB |

## Not yet covered

- Current workspace `node_modules` C17–C23 (recent projects and `.next` build output).
- `C:\Users\<old-user>\.vscode` (dead extensions from the old PC, 9 `node_modules`).
- Related JS caches: `AppData\Local\Microsoft\TypeScript` (~243.6 MB, partial), Cypress, Playwright (`ms-playwright`), electron / electron-builder, Puppeteer, and inside projects `.next`, `.turbo`, `.parcel-cache`, `node_modules\.cache`.
- Other ecosystems: pip, NuGet, Gradle, Maven, Cargo, Go module cache, Windows `%TEMP%`.

## Lessons for the dcleaner agent

1. **Scan first, act later.** List candidates, then measure, then delete one target at a time, with explicit human approval for each deletion.
2. **Bound the output, not just the work.** Grouped or summarized results keep token cost low. Full listings go to a log file, not the chat.
3. **Measure without running out of memory.** Walk files as a stream (one at a time, not the whole list at once), sequentially, with a size ceiling. Avoid parallel `du` jobs in Git Bash on Windows.
4. **Never swallow errors silently.** Count and log skipped folders, or the totals will undercount.
5. **Support long paths.** Use the `\\?\` prefix; paths over 260 characters broke a plain `Remove-Item`.
6. **Never follow links when deleting.** Remove reparse points as links. Avoid PowerShell 5.1 `Remove-Item -Recurse` on trees that may contain links.
7. **Validate target lists** (count and path pattern) before deleting. Refuse to run if any variable could be empty.
8. **Guard and verify.** Record file counts of protected folders before and after, and check `git status` on affected repos.
9. **Read disk usage right before and after each delete,** and store values in variables right away (PowerShell re-reads `PSDrive.Free` on every access). Other disk activity on the PC adds noise.
10. **Check for running processes** (npm, npx, yarn, node-gyp, dev servers) before cleaning their folders.
11. **Classify what must never be cleaned:** Node installs (nvm), app-bundled `node_modules`, editor extensions in use.
12. **Watch for OneDrive / cloud folders.** Deleting there also deletes from the cloud copy and syncs to other devices. Detect cloud-only placeholders without downloading them.
