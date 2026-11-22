# zotero-add

Safely add items, attachments, snapshots, and PDFs to a running Zotero instance from the shell — with built-in guards against duplicates, title mangling, displaced files, and bridge timeouts.

- **Author:** Muse Spark (idea and instigation: Pranesh Prakash)
- **Copyright:** © 2026 Pranesh Prakash
- **Contributors:** Pranesh Prakash, Muse Spark (OpenCode), Gemini 3.8 Flash (Antigravity)
- **License:** [GNU General Public License v3.0 or later](COPYING) (SPDX: `GPL-3.0-or-later`)

## Requirements

- Python 3.9+ (stdlib only; `pypdf` is optionally required for the `adopt` command: `pip install pypdf`)
- [`zotero-cli`](https://github.com/PiaoyangGuohai1/cli-anything-zotero) (`uv tool install cli-anything-zotero`)
- A **running Zotero desktop** instance with the `cli-bridge` add-on installed

## Installation

Symlink or copy `zotero-add` to a folder on your `PATH`:

```sh
ln -s /path/to/zotero-add ~/.local/bin/zotero-add
```

## Specifying the Library

Set your default library via `ZOTERO_LIBRARY` or pass `--library` / `-L` to any command:

```sh
export ZOTERO_LIBRARY=user        # your personal library
export ZOTERO_LIBRARY=my-group    # a group library by name
# Or pass directly:
zotero-add -L my-group dups --title "..."
```

## Usage

### 0. Direct Arguments (`zotero-add <doi|pdf|key>`)

A bare argument is routed to the command that takes it, so you don't have to remember
the taxonomy:

```sh
zotero-add https://doi.org/10.3324/haematol.2026.301753   # -> import-doi
zotero-add ~/papers/mystery-paper.pdf                     # -> adopt
zotero-add QPCABAHY                                       # -> check
zotero-add --dry-run 10.3324/haematol.2026.301753         # print the routed command only
```

Flags after the argument are passed through to the routed command
(`zotero-add 10.1234/abc --collection UFNUEIWT`). arXiv IDs and webpages are refused with
a pointer to `zotero-cli add arxiv` / the Zotero connector, because neither can be saved
faithfully without a translator.

Collections are **never inferred**: without `--collection` the item is saved unfiled
(Zotero's "Unfiled Items" view finds it later). You can ask for the *currently selected*
collection explicitly, which resolves to whatever is highlighted in Zotero's left pane —
the same target the browser connector would use:

```sh
zotero-add current                                  # library, collection, selected items
zotero-add 10.1234/abc --collection current         # file it where you're looking
```

If the selection lives in a different library from `--library`, or nothing is selected,
it stops and says so rather than guessing.

### 1. Check for Duplicates (`dups`)

Searches the library by title, URL, DOI, or citation key before adding (exits with code 1 if duplicates exist):

```sh
zotero-add dups --title "Designing Data Governance" --doi "10.1038/..."
```

### 1b. Ask Where It Would Go (`where`)

Advisory only — reads the library and never writes. Given a DOI, an item key, or just a
title, it reports which collections already hold items like this one, and which
collections are named after it (acronyms included, so "chronic myeloid leukemia" matches
a `CML` collection):

```sh
zotero-add where 10.3324/haematol.2026.301753
zotero-add where QPCABAHY
zotero-add where --title "State of DPI in India 2025" --journal "Report"
zotero-add where QPCABAHY --json
```

Ranking uses what people actually did — items with matching titles, and items in the
same journal — and only then collection names. A collection name that is the only
evidence is shown separately, as a hint.

Every creating command runs the same check and refuses to add on a hit (`--force` overrides):

| command | checked before creating | when |
|---|---|---|
| `item` | title, URL, DOI, citation key | before saving |
| `import-doi` | DOI, citation key; then title | before the Crossref call, and again after it (catches preprints vs published versions, or records with a different DOI) |
| `adopt` (DOI found) | same as `import-doi`, after resolving the DOI | before saving |
| `adopt` (arXiv only) | title, arXiv URL, citation key | before saving |

Title matching is deliberately conservative: it requires an exact normalized match, or
the shorter title to be ≥70% of the longer one and contained in it. Short titles
("Data Governance") therefore match only exactly — check with `dups` when in doubt.
The check looks only at the library you target, not at your other libraries.

### 2. Add Items

#### Manually with Specific Fields (`item`)
Creates an item (dup-checked), preserves title capitalization against linter plugins, and pins a Better BibTeX citation key:

```sh
zotero-add item report \
  --title "Designing Data Governance" \
  --person "Doe, Jane" \
  --date 2024-12 \
  --field institution=ExampleOrg \
  --key doeDesigningData2024 \
  --collection DEADB33F \
  --tag mytag \
  --file report.pdf
```
Use `--person "Last, First[@role]"` or `--org "Name[@role]"` (roles: `author`, `editor`, `contributor`, etc.). Use `--snapshot` to capture an HTML snapshot of `--url`.

#### From a DOI via Crossref (`import-doi`)
Fetches authoritative metadata from Crossref, unescapes entities, parses author/editor roles, pins a citation key, and triggers the PDF cascade:

```sh
zotero-add import-doi 10.1017/S0035869X00130837 --key burrowProtoIndoaryans1973
```

#### Adopt an Orphan PDF (`adopt`)
Takes a local PDF, extracts its DOI or arXiv ID from text (or matches title/author/year via OpenAlex), creates the parent item, and links or stores the file:

```sh
zotero-add adopt ~/papers/mystery-paper.pdf
zotero-add adopt ~/papers/mystery-paper.pdf --stored  # copy into Zotero storage instead of linking
```

### 3. Backfill Missing Metadata (`fill-metadata`)

Finds missing identifiers and bibliographic fields (`DOI`, `abstractNote`, `language`, `date`) via OpenAlex and Crossref using strict matching (title similarity $\ge 70\%$, publication year $\pm 1$, author surname overlap):

```sh
# Dry-run diff preview (default)
zotero-add fill-metadata ITEMKEY
zotero-add fill-metadata --collection DEADB33F

# Explicitly apply changes to Zotero
zotero-add fill-metadata ITEMKEY --apply
zotero-add fill-metadata --collection DEADB33F --apply

# Restrict to specific fields
zotero-add fill-metadata ITEMKEY --fields doi,abstract --apply
```

### 4. Fetch Full-Text PDFs (`fetch-pdf`)

Retrieves open-access or repository PDFs using an ordered cascade: **OpenAlex $\rightarrow$ Unpaywall $\rightarrow$ arXiv $\rightarrow$ Sci-Hub**:

```sh
zotero-add fetch-pdf ITEMKEY
UNPAYWALL_EMAIL=you@example.org zotero-add fetch-pdf ITEMKEY --force
```

### 5. Attach Files & Snapshots (`attach`)

Attaches local files or webpage snapshots to an existing item and verifies that the attachment was saved correctly:

```sh
zotero-add attach ITEMKEY --file document.pdf --snapshot
```

### 6. Verify and Repair Library Health (`check`, `audit`, `repair`)

```sh
# Inspect an individual item's attachments and verify file existence
zotero-add check ITEMKEY

# Audit whole library: reports missing files and unlinked disk orphans
zotero-add audit
zotero-add audit --since 2026-10-01

# Repair displaced files: restores files moved by auto-mover plugins back into storage/
zotero-add repair --since 2026-10-01
```

## Why `zotero-add` Instead of Bare `zotero-cli`?

Automating Zotero from the terminal or via AI agents — particularly when plugins like **Better BibTeX**, **Zotero Linter** (`zotero-format-metadata`), and **Zotero Attanger** are running — exposes several subtle edge cases:

1. **Accidental Duplicates**: Automated additions easily duplicate records if a call fails or is re-run. `zotero-add` checks titles (with fuzzy containment), URLs, DOIs, and citation keys before creating items.
2. **Title Mangling**: Metadata linter plugins often enforce sentence-casing on newly created items, lower-casing proper nouns (e.g. converting *"Data Governance: India"* into *"Data governance: india"*). `zotero-add` waits for the linter to run, then restores the original intended capitalization.
3. **Displaced Files in Group Libraries**: File organizer plugins (like Attanger) may automatically relocate newly added attachments into external folders as linked files. Group libraries in Zotero cannot sync linked files, leaving attachments broken for other collaborators. `zotero-add` checks file placement and `repair` moves displaced files back into syncable storage.
4. **Bridge Timeouts & Phantom Records**: Slow network or database operations can cause the CLI bridge to report a timeout even when Zotero continues processing in the background. Blind retries create duplicate items. `zotero-add` polls the running instance before declaring failure.
