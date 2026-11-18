# nodiMargo

**Know what a Salesforce change will break before you make it.**

nodiMargo is a fast splash-radius indexer for Salesforce codebases. It reads
your repo's Apex, Lightning Web Components, Aura, JavaScript and TypeScript,
and the metadata that wires them together (flows, Lightning pages,
Visualforce, permission sets, profiles, objects, fields and layouts). It then
answers one question: *if I change this, what else is affected?*

It is a single binary with no runtime dependencies. You can use it from the
command line, from CI, or from a coding agent such as Claude Code through its
built-in MCP server.

This repository holds the public releases. Download binaries from the
[Releases page](https://github.com/magia-practica/nodimargo-release/releases).

---

## Why use it

Salesforce dependencies are spread across code and metadata. Before you
rename a field or change a method signature, you need to know which classes
query it, which flows read it, which Lightning pages and layouts show it, and
which permission sets grant it. A text search finds some of these, and the
rest turn up as a failed deployment or a production incident.

nodiMargo builds a dependency graph of the whole repo and walks it for you:

```sh
nodimargo impact Account.Tier__c --direction upstream
```

The result lists every Apex method, trigger, flow, Lightning page, layout,
LWC and permission set that touches the field. Each entry has a depth, and
the result has a risk rating (`LOW` to `CRITICAL`). Apex SOQL, SOSL, typed
field reads and writes, `@salesforce/schema` imports, and flow merge fields
are all linked to the objects and fields they use.

**For you as a developer:**

- Check the blast radius before a refactor, rename or deletion, not after
  the deploy fails.
- Run `detect-changes` before you commit to see which symbols your diff
  touches and what depends on them.
- Give your coding agent the same view. Claude Code and other MCP clients
  can call `impact` and `detect_changes` themselves, and `agents-md` tells
  them when to do it.

**For your team in CI:**

- Every pull request gets a splash-radius report in the job summary, so
  reviewers can see what a change affects without reading every caller.
- It's fast enough to run on every PR. A full index of a 13,000-class repo
  takes a few seconds. An incremental update from a cached `main` snapshot
  takes about a second, and the snapshot is small enough for the Actions
  cache.

## What it understands

| Files | What is indexed |
|---|---|
| `.cls`, `.trigger` | Apex, including inline and literal dynamic SOQL/SOSL |
| `.js`, `.jsx`, `.mjs`, `.cjs`, `.ts`, `.tsx`, `.mts`, `.cts` | JavaScript and TypeScript, including `tsconfig`/`jsconfig` path aliases |
| `lwc/<bundle>/` | LWC modules and templates, `@salesforce/apex` and `@salesforce/schema` imports, `@wire` |
| `aura/<bundle>/` | Aura markup, controllers, helpers and the Apex controller |
| `.flow-meta.xml` | Flows: Apex actions, subflows, screen components, record variables and merge fields |
| `.flexipage-meta.xml` | Lightning pages |
| `.page`, `.component` | Visualforce pages and components, controllers and extensions |
| `.permissionset-meta.xml`, `.profile-meta.xml` | Permission sets and profiles: the classes, pages, objects and fields they grant |
| `objects/…` | Objects, fields, record types and relationship paths (source format) |
| `layouts/…` | Page layouts |
| Other metadata | Custom labels, applications, tabs, custom metadata and more, as named nodes |

Standard objects and fields you reference but don't define (`Account.Name`)
still get a node, so every reference to them meets in one place. When
nodiMargo can't be sure it found everything (a dynamic query built from
variables, an untyped receiver), the result says `lower-bound` rather than
claiming to be complete.

## Install

Each release has one archive per platform, each with a `.sha256` checksum:

| Platform | Archive |
|---|---|
| macOS, Apple silicon | `nodimargo-vX.Y.Z-aarch64-apple-darwin.tar.gz` |
| macOS, Intel | `nodimargo-vX.Y.Z-x86_64-apple-darwin.tar.gz` |
| Linux, x86-64 | `nodimargo-vX.Y.Z-x86_64-unknown-linux-gnu.tar.gz` |
| Linux, arm64 | `nodimargo-vX.Y.Z-aarch64-unknown-linux-gnu.tar.gz` |
| Windows, x64 | `nodimargo-vX.Y.Z-x86_64-pc-windows-msvc.zip` |

Replace `X.Y.Z` with the
[latest release](https://github.com/magia-practica/nodimargo-release/releases/latest)
and `<target>` with your platform in the commands below.

### macOS and Linux

With the [GitHub CLI](https://cli.github.com):

```sh
gh release download vX.Y.Z --repo magia-practica/nodimargo-release \
  --pattern 'nodimargo-vX.Y.Z-<target>.tar.gz*'
```

Or download the archive and its `.sha256` file from the Releases page. Then,
in the same directory:

```sh
shasum -a 256 -c nodimargo-vX.Y.Z-<target>.tar.gz.sha256    # Linux: sha256sum -c
tar -xzf nodimargo-vX.Y.Z-<target>.tar.gz
sudo cp nodimargo-vX.Y.Z-<target>/nodimargo /usr/local/bin/  # or any directory on your PATH
```

The binaries aren't signed. If macOS Gatekeeper blocks the binary, clear the
quarantine flag:

```sh
xattr -d com.apple.quarantine /usr/local/bin/nodimargo
```

### Windows

Download `nodimargo-vX.Y.Z-x86_64-pc-windows-msvc.zip` and its `.sha256`,
then in PowerShell, in the same directory:

```powershell
# Compare this hash with the first field of the .sha256 file
(Get-FileHash .\nodimargo-vX.Y.Z-x86_64-pc-windows-msvc.zip -Algorithm SHA256).Hash
Get-Content .\nodimargo-vX.Y.Z-x86_64-pc-windows-msvc.zip.sha256

Expand-Archive .\nodimargo-vX.Y.Z-x86_64-pc-windows-msvc.zip -DestinationPath .

# Add the unpacked folder to your user PATH
$u = [Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::SetEnvironmentVariable("Path", $(if ($u) { "$u;<dir>" } else { "<dir>" }), "User")
```

Replace `<dir>` with the full path of the unpacked folder, then open a new
terminal. SmartScreen may stop the unsigned binary the first time it runs:
choose **More info**, then **Run anyway**.

### Check it

```sh
nodimargo --version
nodimargo version     # also says whether a newer release exists
```

You also need `git` on your `PATH`, since nodiMargo uses it to find changed
files.

### Updating and uninstalling

To update, download the new archive and replace the binary. The first
`analyze` after an upgrade may rebuild the index in full once, then runs
incrementally again.

To uninstall, delete the binary, `~/.nodimargo` (the registry and any stored
license) and each repo's `.nodimargo/` folder. If you registered the MCP
server, also run `claude mcp remove nodimargo`.

---

## Walkthrough

### 1. Index your repo

```sh
cd /path/to/your/salesforce-repo
nodimargo analyze .
```

The first run indexes everything and writes `.nodimargo/index.nmi`. Later
runs update only the files git reports as changed and give the same result as
a full index. Add `.nodimargo/` to your `.gitignore`.

`analyze` prints a JSON report (trimmed):

```json
{ "mode": "full", "files": 1000, "parsed": 500,
  "stats": { "nodes": 12306, "edges": 115668 },
  "timings_ms": { "scan": 40, "parse": 57, "resolve": 16, "write": 31 } }
```

If `sfdx-project.json` sets a `namespace`, references with that prefix
resolve to your own classes. Re-run `analyze` whenever you want the index
current. It takes about a second.

### 2. Ask what a change affects: `impact`

```sh
nodimargo impact AccountService.save                    # who calls this? (upstream is the default)
nodimargo impact AccountService --direction downstream  # what does this depend on?
nodimargo impact Account.Tier__c                        # everything that uses a field
nodimargo impact Close_Opportunity_Flow                 # a flow by API name
nodimargo impact force-app/main/default/classes/Svc.cls # a whole file
```

The target can be a class, method, field (`Object.Field`), object, flow,
page, component or file path. Useful options:

| Option | Effect |
|---|---|
| `--direction upstream\|downstream` | Who depends on it, or what it depends on |
| `--depth N` | How many hops to walk |
| `--include-tests[=true\|false]` | Include test code. Left out by default and counted, except when the target or its class is `@isTest`: only test code can use it, so tests are followed automatically (`"testsIncluded": "auto: target is @isTest"`). `--include-tests=false` forces them off |
| `--kind Flow`, `--file-path P`, `--uid ID` | Narrow down an ambiguous name |
| `--summary-only` | Counts and risk only |

Output (trimmed):

```json
{ "target": { "name": "computeBatch", "type": "Method" },
  "direction": "upstream", "impactedCount": 5447, "risk": "CRITICAL",
  "summary": { "direct": 272, "processes_affected": 33, "modules_affected": 16 },
  "byDepth": { "1": [ ... ] } }
```

If a name matches several symbols, the result lists the candidates for you to
choose from. Exit code 2 means an error, such as target not found.

### 3. Check your diff before committing: `detect-changes`

```sh
nodimargo detect-changes                         # unstaged changes
nodimargo detect-changes --scope staged          # what you're about to commit
nodimargo detect-changes --scope compare --base-ref origin/main --format markdown
```

It maps the git diff onto indexed symbols and reports what changed and which
processes are affected, with a risk level. `--format markdown` gives a short
summary for a PR description or a CI job page.

### 4. Let your coding agent use it: `mcp` and `agents-md`

```sh
claude mcp add nodimargo -- nodimargo mcp   # register the MCP server with Claude Code
nodimargo agents-md .                       # add usage instructions to AGENTS.md / CLAUDE.md
```

The MCP server offers `impact`, `detect_changes` and `list_repos`, and picks
up a re-run of `analyze` without a restart. `agents-md` writes a marked block
into `AGENTS.md` and/or `CLAUDE.md`. The block tells agents to run `impact`
before modifying, renaming or deleting something, to report the risk and
agree on a plan when it is HIGH or CRITICAL, and to run `detect_changes`
before committing. Re-running it updates the block in place and leaves the
rest of the file alone.

### Other commands

| Command | What it does |
|---|---|
| `nodimargo list` | Registered repos, their stats and staleness |
| `nodimargo snapshot export OUT` | Copy the repo's index to a file, for example for a CI cache |
| `nodimargo ci cache-key` / `ci pick-base` | CI helpers for caching and restoring snapshots (see below) |
| `nodimargo version [--offline] [--json]` | Installed version, and whether a newer release exists |
| `nodimargo license …` | Activate, deactivate or check a license (see below) |

Run `nodimargo <command> --help` for every option.

---

## Licensing

**nodiMargo works fully without a license.** Every command and every feature
works the same. Without a license, `analyze`, `impact`, `detect-changes` and
the MCP server show a short reminder with a link to buy one. A license
removes the reminder.

There are two kinds of license, one for people and one for pipelines:

| | Developer license | CI license |
|---|---|---|
| Key prefix | `NODIMARGO-DEV-…` | `NODIMARGO-CI-…` |
| For | You, on your own machines | A CI pipeline |
| How it's used | Activated on up to 3 machines | Read from the `NODIMARGO_LICENSE_KEY` environment variable, never activated |

Buy one: [Developer license](https://buy.polar.sh/polar_cl_PUksSrlwPrChpehyZErZT4auxURIZLTch7L184Nv4e6) ·
[CI license](https://buy.polar.sh/polar_cl_FYuzz3kT7GpvOsspgvhloFYz9B2VwwmdgIeNV3M2kuY).
Your key arrives in the purchase email and in the
[customer portal](https://polar.sh/ignoti-et-quasi-occulti/portal).

### Developer license

```sh
nodimargo license activate NODIMARGO-DEV-…   # use one of your 3 machine slots
nodimargo license status                     # mode, masked key, last confirmed
nodimargo license deactivate                 # free this machine's slot
```

The license is stored in `~/.nodimargo/license.json`. To move to a new
machine, run `deactivate` on the old one first, or free the slot from the
customer portal.

### CI license

Store the key as a CI secret and expose it as `NODIMARGO_LICENSE_KEY` (see
the GitHub Actions section below). `license activate` refuses CI keys.
`nodimargo license status` in the pipeline reports whether the key is valid.
If the license server can't be reached from CI, the run counts as licensed
and prints a one-line warning. A flaky network never breaks your build.

### What the reminder looks like

The reminder goes to stderr with a QR code for the purchase link. Unlicensed
JSON output also gets a top-level `"license": {"status": "unlicensed", …}`
field, and `--format markdown` ends with one italic line. Licensed output has
neither.

### Network and privacy

- **When nodiMargo calls the license server:** with no stored license and no
  `NODIMARGO_LICENSE_KEY`, never. A stored developer license is checked at
  most once a day and stays valid through up to 14 days offline. A CI key is
  checked on each run.
- **Sent:** the license key, a salted hash of the machine ID (never the raw
  ID), the hostname as the activation label, the OS and the nodiMargo
  version.
- **Never sent:** your code, file contents or repository paths.

Besides licensing, the only network call nodiMargo makes is `nodimargo
version` checking for a newer release (skip it with `--offline`).

---

## GitHub Actions: a splash-radius report on every PR

This workflow indexes `main` on every push and caches a snapshot of the
index. On every pull request it restores the nearest snapshot from `main`,
updates the index incrementally, and writes a report to the job summary and
to a `detect.json` artifact. It downloads the Linux release binary, so it
needs no Rust toolchain.

### Setup

1. Save the workflow below as `.github/workflows/nodimargo.yml` in your
   Salesforce repo.
2. Set `NODIMARGO_REF` to the
   [latest release tag](https://github.com/magia-practica/nodimargo-release/releases/latest).
3. Optional, to remove the reminder: buy a CI license, then go to
   **Settings → Secrets and variables → Actions → New repository secret**
   and add it with the name `NODIMARGO_LICENSE_KEY`. An organization secret
   works too.
4. Keep the `permissions` block. `actions: read` lets `gh cache list` find
   the snapshots.

The first push to `main` creates the first snapshot. Until a snapshot
exists, or after the cache evicts it, a PR falls back to a full index. That
is slower but gives the same result.

```yaml
name: nodimargo

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  actions: read # lets `gh cache list` see the snapshots

env:
  NODIMARGO_REF: vX.Y.Z # set to the latest release tag
  NODIMARGO_REPO: magia-practica/nodimargo-release

jobs:
  splash-radius:
    runs-on: ubuntu-latest
    env:
      # Optional: a CI license key. Without it, runs still work but show the reminder.
      NODIMARGO_LICENSE_KEY: ${{ secrets.NODIMARGO_LICENSE_KEY }}
    steps:
      - name: Restore nodimargo
        id: bin
        uses: actions/cache@v4
        with:
          path: ~/.cargo/bin/nodimargo
          key: nodimargo-bin-${{ runner.os }}-${{ env.NODIMARGO_REF }}

      - name: Install nodimargo from a release
        if: steps.bin.outputs.cache-hit != 'true'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          set -euo pipefail
          target=x86_64-unknown-linux-gnu
          name="nodimargo-${NODIMARGO_REF}-${target}"
          gh release download "$NODIMARGO_REF" --repo "$NODIMARGO_REPO" \
            --pattern "${name}.tar.gz" --pattern "${name}.tar.gz.sha256"
          sha256sum -c "${name}.tar.gz.sha256"
          tar -xzf "${name}.tar.gz"
          mkdir -p ~/.cargo/bin
          cp "${name}/nodimargo" ~/.cargo/bin/

      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # the merge base and the first-parent walk need history

      - name: Keep index files out of the working tree diff
        run: |
          printf '%s\n' '.nodimargo/' '.nodimargo-cache/' 'keys.txt' 'detect.json' 'summary.md' >> .git/info/exclude

      # ---- main: index and cache a snapshot ----
      - name: Index main
        if: github.event_name == 'push'
        run: nodimargo analyze . --force

      - name: Compute the cache key
        id: key
        if: github.event_name == 'push'
        run: |
          set -euo pipefail
          KEY=$(nodimargo ci cache-key --sha "$GITHUB_SHA")
          echo "key=$KEY" >> "$GITHUB_OUTPUT"

      - name: Export the snapshot
        if: github.event_name == 'push'
        run: |
          set -euo pipefail
          mkdir -p .nodimargo-cache
          nodimargo snapshot export .nodimargo-cache/index.nmi

      - name: Save the snapshot
        if: github.event_name == 'push'
        uses: actions/cache/save@v4
        with:
          key: ${{ steps.key.outputs.key }}
          path: .nodimargo-cache

      # ---- pull request: incremental index and report ----
      - name: Find the merge base
        id: base
        if: github.event_name == 'pull_request'
        run: |
          set -euo pipefail
          BASE=$(git merge-base "origin/$GITHUB_BASE_REF" HEAD)
          echo "sha=$BASE" >> "$GITHUB_OUTPUT"

      - name: Pick a snapshot
        id: pick
        if: github.event_name == 'pull_request'
        env:
          GH_TOKEN: ${{ github.token }}
          BASE: ${{ steps.base.outputs.sha }}
        run: |
          set -euo pipefail
          # Only main's snapshots: a pull request cannot restore another PR's cache.
          gh cache list --ref refs/heads/main --key "$(nodimargo ci cache-key --prefix)" \
            --limit 100 --json key --jq '.[].key' > keys.txt || true
          PICK=$(nodimargo ci pick-base --keys keys.txt --from "$BASE")
          echo "key=$PICK" >> "$GITHUB_OUTPUT"

      - name: Restore the snapshot
        if: github.event_name == 'pull_request' && steps.pick.outputs.key != ''
        uses: actions/cache/restore@v4
        with:
          key: ${{ steps.pick.outputs.key }}
          path: .nodimargo-cache

      - name: Update the index
        if: github.event_name == 'pull_request'
        run: |
          set -euo pipefail
          if [ -f .nodimargo-cache/index.nmi ]; then
            nodimargo analyze . --base .nodimargo-cache/index.nmi
          else
            nodimargo analyze . --force # no snapshot: full index
          fi

      - name: Detect changes
        if: github.event_name == 'pull_request'
        env:
          BASE: ${{ steps.base.outputs.sha }}
        # Write the markdown and the JSON first, then fail with the saved
        # exit code, so the summary and the artifact are never lost.
        run: |
          set -euo pipefail
          rc=0
          nodimargo detect-changes --scope compare --base-ref "$BASE" --format markdown > summary.md || rc=$?
          json_rc=0
          nodimargo detect-changes --scope compare --base-ref "$BASE" > detect.json || json_rc=$?
          cat summary.md >> "$GITHUB_STEP_SUMMARY"
          if [ "$rc" -eq 0 ]; then rc=$json_rc; fi
          exit "$rc"

      - name: Upload the report
        if: always() && github.event_name == 'pull_request'
        uses: actions/upload-artifact@v7
        with:
          name: nodimargo-detect-changes
          path: detect.json
```

### Posting the report as a PR comment

To post the report as a PR comment, add `pull-requests: write` to
`permissions` and append this step:

```yaml
      - name: Comment on the PR
        if: github.event_name == 'pull_request'
        env:
          GH_TOKEN: ${{ github.token }}
          PR: ${{ github.event.pull_request.number }}
        run: gh pr comment "$PR" --body-file summary.md
```

Whether a large splash radius should fail the check is up to your team.
`detect.json` holds `summary.risk_level` and the full list of changed and
affected symbols, which you can gate on with `jq`.

### Other CI systems

The same flow works anywhere. On `main`, run `analyze`, then
`snapshot export`, and cache the result under `ci cache-key`. On a PR, list
your cached keys, run `ci pick-base --keys keys.txt --from <merge-base>`,
restore that snapshot, then run `analyze --base` and `detect-changes --scope
compare`. Snapshots are portable across clones, runners and operating
systems of the same repository.

---

## Environment variables

| Variable | Meaning |
|---|---|
| `NODIMARGO_LICENSE_KEY` | A CI license key (`NODIMARGO-CI-…`) |
| `NODIMARGO_HOME` | Registry and license directory (default `~/.nodimargo`) |
| `NODIMARGO_DEFAULT_REPO` | Repo used when `--repo` is omitted |
| `NODIMARGO_MCP_NAME` | MCP server name (default `nodimargo`) |
| `NODIMARGO_IMPACT_DEFAULT_LIMIT` | MCP `impact` rows per depth on a first page (default 30). A call with `offset > 0` or `summaryOnly` and no `limit` uses 100 instead, as the CLI always does; pass `limit` on every page for a fixed page size |

## Known limitations

- Dynamic SOQL is linked only through its string literals. A query built
  from variables marks affected results as `lower-bound`.
- Fields reached through a receiver typed only as `SObject`, field sets and
  describe-based reflection are not followed.
- Objects and fields are read in source format only, not metadata-API
  `.object` files.
- The binaries are unsigned. Windows is x64 only.
- On Windows, antivirus scanning can slow `analyze`. Excluding `.nodimargo\`
  helps.

## Third-party licenses

The Apex grammar is [`aheber/tree-sitter-sfapex`](https://github.com/aheber/tree-sitter-sfapex)
(MIT). The JavaScript and TypeScript grammars are the upstream tree-sitter
grammars (MIT). The metadata type registry is a snapshot of Salesforce's
`source-deploy-retrieve` (Apache-2.0).
