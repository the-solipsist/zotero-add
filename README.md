# zotero-add

Safely add items, attachments, snapshots and PDFs to a running Zotero
instance from the shell — with guards against the failure modes that bite
in practice (duplicates, title-mangling plugins, displaced attachment
files, bridge timeouts that still complete).

- **Author:** Muse Spark (idea and instigation: Pranesh Prakash)
- **Copyright:** © 2026 Pranesh Prakash
- **Contributors:** Pranesh Prakash, Muse Spark
- **License:** [GNU General Public License v3.0 or later](COPYING) (SPDX: `GPL-3.0-or-later`)

## Requires

- Python 3 (stdlib only, except `pypdf` for the `adopt` command:
  `pip install pypdf`)
- [`zotero-cli`](https://github.com/PiaoyangGuohai1/cli-anything-zotero)
  (`uv tool install cli-anything-zotero`) and a **running Zotero** with its
  JS Bridge plugin

## Install

Copy `zotero-add` anywhere on your `PATH`, e.g. `~/.local/bin/zotero-add`.

## Use

```sh
export ZOTERO_LIBRARY=mylibrary   # or pass -L every time: user | group name | groupID | libraryID

# 1. Look for an existing record first (exit 1 = found)
zotero-add dups --title "…" --url "…" --doi "…"

# 2. Create, with pinned citation key, tags, collection, files
zotero-add item report --title "Data Governance Framework: India" \
  --key prakashIndia2024 --person "Prakash, Pranesh" --date 2024-12 \
  --field institution=LIRNEasia --collection DEADB33F --tag otr2026 \
  --file report.pdf

# 2b. Or create from a DOI via Crossref metadata (editors split out,
# entities unescaped, key pinned), then fetch its PDF automatically.
zotero-add import-doi 10.1017/S0035869X00130837 --key burrowProtoIndoaryans1973

# 2c. Or adopt an orphan PDF: finds its DOI/arXiv ID (else strict
# title match), creates the parent item, and links the file in place.
zotero-add adopt ~/papers/mystery-paper.pdf

# 3. Fetch a missing PDF by DOI (OpenAlex → Unpaywall → arXiv → Sci-Hub)
zotero-add fetch-pdf ITEMKEY
UNPAYWALL_EMAIL=you@example.org zotero-add fetch-pdf ITEMKEY --force

# 4. Fill missing metadata (DOI, abstract, language, date) via OpenAlex / Crossref
# Strict matching (title + year ±1 + author surname overlap); dry-run diff by default
zotero-add fill-metadata ITEMKEY
zotero-add fill-metadata ITEMKEY --apply
zotero-add fill-metadata --collection COLLKEY --apply

# 5. Inspect / repair
zotero-add check ITEMKEY   # attachments + whether their files exist
zotero-add audit           # whole library: missing files + disk orphans (exit 1 if issues)
zotero-add repair --since 2026-10-01   # put back files a mover plugin displaced
```

`--person "Last, First"` / `--org "Name"` keep their order; append
`@editor`, `@presenter` etc. for other roles. `--field name=value` sets any
Zotero field for that item type (wrong names are rejected by Zotero).

## Why not just `zotero-cli add`?

`zotero-add` wraps the sharp edges this workflow taught us:

- **Duplicates** are checked (title, URL, DOI, citation key) before creating;
  it refuses instead of duplicating.
- **Title mangling**: metadata-linting plugins rewrite new titles
  (sentence-casing that lower-cases proper nouns); the title is re-applied
  after they run.
- **Displaced attachments**: file-mover plugins can relocate a fresh
  download out of `storage/`; `verify`/`repair`/`audit` detect that and
  `repair` moves files back.
- **Bridge timeouts**: `zotero-cli js` can report "timed out" while Zotero
  carries on; results are polled, never blindly retried (which is how
  duplicates happen).
