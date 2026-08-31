# byakugan

A content-inspection companion to [sharingan-go](https://github.com/Mohamed-Newish/sharingan-go)
for authorized bug-bounty/pentest recon — not a rerun of it.

sharingan-go owns everything network-facing: subdomain enumeration,
probing, port scanning, JS crawl/intel, WAF fingerprinting, origin-IP
discovery, block isolation — all gated behind its stealth engine. Look at
its own `internal/output/layout.go` and you'll find it already reserves
filenames for `screenshots/`, `raw_responses/`, `paths.txt`, `params.txt`,
`auth_urls`, `sqli_candidates`, and `xss_candidates` — but the phases that
would populate several of them (`screenshots`, `wordlist`, `sqli`, `xss`)
are literal `TODO` stubs in the code, and nothing writes `raw_responses/`
or `auth_urls` at all.

**byakugan fills exactly those gaps**, using the proven tomnomnom-toolchain
commands a bash recon pipeline ran before sharingan-go existed (`meg`,
`aquatone`, `tok`, `unfurl`, `dsss`, `kxss`, `dalfox`), plus one thing
sharingan-go never reserved space for at all: `cariddi`-style
secrets/juicy-endpoint crawling. It reads the *same* target directory
sharingan-go writes to (`hosts`, `urls`, `domains`, `inscope.txt`) and
writes into the *same* filenames sharingan-go's layout already promises —
so the two tools share one coherent per-target output without ever
duplicating each other's work.

Named for the Byakugan — the "all-seeing" dojutsu — as the thematic
complement to Sharingan (the copying one): sharingan-go automates the
mechanical recon, byakugan is the deep-look pass over what it surfaced —
raw page bodies, screenshots, secrets sitting in page source, marker-based
SQLi/XSS leads.

## Division of labor

| | sharingan-go | byakugan |
|---|---|---|
| Subdomain enum, probing, ports | ✅ | — |
| JS crawl/intel (endpoints, secrets, sourcemaps) | ✅ | — |
| WAF fingerprinting, origin-IP, block isolation | ✅ | — |
| Root-page fetch (raw headers/body) | reserves `raw_responses/`, doesn't write it | ✅ `meg` |
| Screenshots | stub | ✅ `aquatone` |
| Path/param wordlists | stub | ✅ `tok` + `unfurl` |
| Auth-flow URL triage | reserves `auth_urls`, doesn't write it | ✅ zero-network grep |
| Marker-based SQLi scan | stub | ✅ `dsss` |
| Marker-based XSS scan | stub | ✅ `kxss` + `dalfox` |
| Secrets/juicy-extension crawl (whole site, not just JS) | no equivalent | ✅ `cariddi` |

Always run sharingan-go first, byakugan second, against the same target
directory. byakugan has nothing to do on an empty one.

## Usage

```bash
byakugan <target-dir> [flags]
```

```bash
byakugan targets/example.com --dry-run -v --blind-xss https://your-collector   # preview
byakugan targets/example.com --blind-xss https://your-collector                # run for real
```

### Flags

| Flag | Meaning |
|---|---|
| `--only <phases>` | comma list — run just these phases |
| `--skip <phases>` | comma list — skip these phases |
| `--resume` | skip a phase whose output file/dir already has content |
| `--dry-run` | print what would run, execute nothing |
| `--profile <name>` | `ninja` \| `normal` (default) \| `loud` — see below |
| `--blind-xss <url>` | your collector — required for the `xss` phase, no hardcoded default |
| `--no-scope-check` | proceed even without an `inscope.txt` (loud warning) — off by default |
| `-v` | verbose |
| `-h`, `--help` | usage |

### Phases (run order)

```
rawfetch → screenshots → wordlist → authgrep → secrets → sqli → xss
```

| Phase | Tool | Writes |
|---|---|---|
| `rawfetch` | `meg` | `raw_responses/` |
| `screenshots` | `aquatone` | `screenshots/` |
| `wordlist` | `tok` + `unfurl` (+ `uro` if present) | `paths.txt`, `params.txt` |
| `authgrep` | plain `grep` — zero extra requests | `auth_urls` |
| `secrets` | `cariddi` | `secrets_juicy/{secrets,juicy}` |
| `sqli` | `qsreplace` + `dsss` | `sqli_candidates` |
| `xss` | `qsreplace` + `kxss` (optional) + `dalfox` | `xss_candidates` |

Every artifact is appended with `anew` (dedupe, never overwrite) wherever
the tool chain supports piping through it. A missing tool skips its own
phase with a clear message — nothing here hard-crashes the run.

### Stealth profiles

Same three-tier vocabulary as sharingan-go, mapped onto the tools byakugan
actually shells out to:

| Profile | meg delay | dalfox workers | cariddi concurrency |
|---|---|---|---|
| `ninja` | 1000s | 3 | 3 |
| `normal` (default) | 200s | 10 | 10 |
| `loud` | 10s | 30 | 30 |

Verify the exact flags against your installed tool versions (`meg -h`,
`dalfox -h`, `cariddi -h`) if a version drift breaks a phase — CLI
surfaces can shift between releases, same caveat sharingan-go's own README
notes for `katana`.

### Scope enforcement

byakugan reads `<target-dir>/inscope.txt` (same allow-list format as
sharingan-go's `--scope`: bare roots or `*.root` wildcards, one per line,
`#` comments OK) and filters every host/URL it's about to touch against
it before running a network-facing phase. This is defense-in-depth, not
the primary guarantee — sharingan-go's own output should already be
scope-safe by construction, since it enforces `--scope` on every request
it sends. The filter here protects against a hand-edited or stale
artifact file being fed in directly. Refuses to run without an
`inscope.txt` unless `--no-scope-check` is passed explicitly.

## Install

Requires bash. No build step — it's a single script.

```bash
git clone https://github.com/Mohamed-Newish/byakugan.git
chmod +x byakugan/byakugan
ln -s "$(pwd)/byakugan/byakugan" ~/.local/bin/byakugan   # or copy it anywhere on PATH
```

### External tools it shells out to

| Tool | Used by | Notes |
|---|---|---|
| [meg](https://github.com/tomnomnom/meg) | `rawfetch` | |
| [aquatone](https://github.com/michenriksen/aquatone) | `screenshots` | |
| [tok](https://github.com/tomnomnom/hacks/tree/master/tok), [unfurl](https://github.com/tomnomnom/unfurl) | `wordlist` | |
| [uro](https://github.com/s0md3v/uro) | `wordlist` | optional — falls back to a less-deduped pass without it |
| [qsreplace](https://github.com/tomnomnom/qsreplace), [anew](https://github.com/tomnomnom/anew) | `sqli`, `xss` (and everywhere else for deduping) | |
| [dsss](https://github.com/stamparm/DSSS) | `sqli` | |
| [kxss](https://github.com/Emoe/kxss) | `xss` | optional — falls back to piping straight to `dalfox` without it |
| [dalfox](https://github.com/hahwul/dalfox) | `xss` | |
| [cariddi](https://github.com/edoardottt/cariddi) | `secrets` | |

A missing tool doesn't stop the run — its phase just logs that it was
skipped and moves on.

## Disclaimer

For **authorized security testing and educational use only**. Run it
exclusively against assets you own or are explicitly permitted (in scope)
to test.
