# 🏗️ Taf Server Architecture

> **Naming note**: this project was renamed from **Stelae** to **taf-server**. Source history, some file paths, older documentation, and even some symbol names (`StelaeGraph`, `Global` for `Stele`) still say "Stelae" — treat the two names as the same project.

> **Confidence note**: most of this document is drawn directly from reading the source. A few files weren't available to check directly (listed in §7); those sections are marked and kept intentionally short rather than guessed at length.

---

## 1. How to Read This Document

This is a **physical map**, organized the way the repository is organized: for each directory, what lives in it and what it's responsible for. Cross-cutting behavior that only makes sense once you've seen several files together (redirects, multi-tenancy, the update/rebuild logic) is pulled into its own section at the end rather than repeated in each file's entry.

---

## 2. Top-Level Layout

```
taf-server/
├── src/                  # The crate itself
├── tests/                # Integration tests, fixtures, and test helpers
├── migrations/sqlite/    # Database schema migrations
├── benches/              # Performance benchmarks
├── docs/                 # Documentation (this file lives here)
├── build.rs              # Build-time script
├── Cargo.toml            # Crate manifest and dependencies
├── justfile              # Task runner commands (lint, test, bench, ci)
└── dockerfile            # Container build
```

`src/` is a library crate with a thin binary wrapper: `src/main.rs` is the executable entry point, `src/lib.rs` declares the modules below.

---

## 3. `src/utils/` — CLI and Cross-Cutting Helpers

| File | Contents |
|---|---|
| `cli.rs` | The `stelae` CLI: `Cli`/`Subcommands` clap structs, the `CliProvider` trait, `execute_command` (dispatches a parsed subcommand), tracing/logging setup, and `run()` (the actual `main()` body). Each subcommand's real logic is a one-line delegation from here into its own module. |
| `check.rs` | `stelae check`'s implementation: recursively validates an archive (`repositories.json`, `dependencies.json`, `info.json`, target metadata files) and returns a `Report` of errors/warnings. |
| `archive.rs` | Small path-resolution helpers (`find_archive_path`, `get_name_parts`) used when locating an archive on disk. Not to be confused with `stelae/archive.rs`. |
| `git.rs` | The `Repo` type and git-blob lookup logic (`Repo::new`, `Repo::find_blob`, `Repo::iter_commits`, the `GIT_REQUEST_NOT_FOUND` sentinel). Every place in the server or `history/` that reads a file out of a repository's Git tree goes through here. |
| `http.rs` | `get_contenttype` — maps a file path to the `Content-Type` header used when serving it. |
| `paths.rs` | `clean_path` — path normalization used before every blob lookup and redirect match. |
| `md5.rs` | `md5::compute` — used by `history/changes.rs` to derive stable hash IDs for publications and versions. |

---

## 4. `src/stelae/` — Core Domain Types

| File | Contents |
|---|---|
| `archive.rs` | `Archive`: a path plus a map of qualified name → `Stele`. `Archive::parse` resolves the root stele and recursively discovers dependent steles via `traverse_children`, reading each stele's `dependencies.json`. Also holds `Archive::init` and the `Config`/`Headers` structs matching `.taf/config.toml` (`Headers.current_documents_guard` is the private-archive header name, referenced again in §8). |
| `stele.rs` | `Stele`: one organization's authentication repository plus its parsed `repositories.json`. Accessors read files out of the auth repo's Git tree at a given commit: `get_bytes_at_path`, `get_dependencies`, `get_repositories`, `get_repositories_for_commitish`, `get_targets_metadata_at_commit_and_filename`, `is_private_stelae`. |
| `types/repositories.rs` | `Repositories`/`Repository`/`Custom` — the schema of `targets/repositories.json`: a stele's content repositories, each with a type, a way of being routed (`serve-prefix` and/or `routes`), fallback/archived flags, and an optional preview URL. |
| `types/dependencies.rs` | `Dependencies`/`Dependency` — the schema of `targets/dependencies.json`. |
| `types/targets_metadata.rs` | `TargetsMetadata` — the schema of a data repository's metadata file at `targets/<org>/<data-repo-name>` (branch, commit, build-date, codified-date). |

---

## 5. `src/server/` — The Web Server

### 5.1 Top level

| File | Contents |
|---|---|
| `app.rs` | `serve_archive` — the `stelae serve` entrypoint: connects the database, parses the archive, starts Actix. `init` assembles the `App` and calls `api::routes::register_app`. |
| `errors.rs` | `CliError` (CLI exit codes), `HTTPError`/`StelaeError` (web-facing errors implementing Actix's `ResponseError`). |
| `git.rs` | `serve_git` — the `stelae git` entrypoint, serving the archive's repositories over the Git smart HTTP protocol so they can be cloned directly, independent of the document-serving API below. |
| `headers.rs` | Custom header constants and helpers — `HTTP_E_TAG`, `HTTP_X_FILE_PATH`, `etag_matches_if_none_match`. |
| `tracing.rs` | The Actix tracing middleware (`StelaeRootSpanBuilder`) used by `TracingLogger` in `app.rs`. |

### 5.2 `server/api/` — The actual endpoints

```rust
// api/mod.rs
pub mod archive;
pub mod date;
pub mod doc_transform;
pub mod routes;
pub mod serve;
pub mod state;
pub mod utils;
pub mod versions;
```

**`routes.rs`** is the composition root: every URL the server answers is registered here, in `register_app`. It's organized into two layers:

- **Static scopes**, registered up front regardless of the archive's content:
  - `/_api/versions/...` — a long list of path patterns (with/without a named publication, with/without a comparison date, with/without a trailing document path) all routed to the single `versions` handler in `api/versions`.
  - `/_date/{version_date}` and `/_date/{version_date}/{path}` — routed to `date` in `api/date`.
  - `/_publication/{pub_name}`, optionally with `/_date/{version_date}` and/or `/{path}` — also routed to `date`. (`date` handles both the plain and publication-scoped historical views; which one it's serving is just which path params are populated.)

- **Dynamic scopes**, built by walking the parsed `Archive`, via `register_guarded_and_unguarded_routes`:
  - `_archive/{namespace}/{name}` → `get_blob` (`api/archive`) — always registered, either openly (`initialize_archive_route`) or behind a header guard (`initialize_guarded_archive_route`), depending on whether `.taf/config.toml`'s `[headers]` section sets `current_documents_guard`.
  - The "current documents" routes → `serve` (`api/serve`), built per-stele by `register_root_routes` (root: mounts each repository's `routes` glob patterns at `/{tail:...}`, and its `serve-prefix`, if any, at `/{prefix}/{tail:.*}`) and `register_dependent_routes` (dependent steles: the same, but nested under whichever path prefixes the stele lists in `repositories.json`'s `scopes`, and skipping any `routes` entry starting with `_` — those are reserved for the root stele to handle). These are also either open (`initialize_dynamic_routes`) or, for multi-stele archives with a guard configured, registered once per stele behind a header matching that stele's qualified name (`initialize_guarded_dynamic_routes`).

**`state.rs`** holds what request handlers are given to work with:
- `Global` (trait) / `App` (struct) — `archive()`, `db()`, and `has_redirects(stele, repo)`, the last backed by a `HashSet<(String, String)>` computed once at startup so most requests never need a database round-trip just to check whether redirects are even possible.
- `RepoData` — per-route state for `serve`: which archive/stele/org/repo/serve-type a given route resolves to. Its `get_redirects()` reads a repository's own `redirects.json` file straight out of its Git blob — this is the *source* half of the redirect system described in §8, not what's queried live at request time.
- `Shared` — per-stele state holding the stele's fallback repository (if any), used when nothing else matches a request path.
- `init_repo`/`init_shared` — build the above from a `Repository`/`Stele`.

**`archive/`** implements `_archive`: `get_blob` resolves `{namespace}/{name}` plus `path`/`commitish` query params to a specific Git blob, after confirming the stele isn't private and that the repository is actually listed in its `repositories.json`. Supports `ETag`/`If-None-Match`. Its stele lookup is `format!("{namespace}/law")` — it assumes the root/auth repository is literally named `law`, which is narrower than the rest of the codebase (elsewhere the auth repo name comes from `.taf/config.toml`'s `root.name` and can be anything).

**`date/`** implements the historical/versioned document view (both `/_date/...` and `/_publication/...`, per the routing above): resolves which stele is being asked for, finds the relevant publication (by name, or the latest), finds which HTML repository and commit correspond to the requested date via the database, fetches that blob, and — for HTML or JSON documents — rewrites it via `doc_transform` so internal links point back into the same historical context, injecting an "outdated version"/"outdated publication" notification banner where relevant. `notifications.rs` alongside it builds those banner strings; `request/` holds the path-parameter struct.

**`doc_transform.rs`** is the HTML/JSON rewriting used by `date`: building the `/_date`/`/_publication` URL prefixes, formatting dates, computing the version window (previous/next/current) around a requested date, string-splicing a notification banner into `<main id="area__content">`, and — via `libxml` — walking an HTML document's `<a>`, `<span>`, heading, `<object>` (PDF), and `<meta>` elements to prefix local links, plus a parallel JSON walk for `manifest.json` and the site's TOC-index JSON format.

**`versions/`** implements `/_api/versions`: resolves the stele and the requested publication, looks up whether the requested URL is a document or a library/collection (via the `document_element`/`library` tables) and fetches its version history accordingly, and returns a JSON payload built in `response/` (with `messages.rs` supplying the user-facing status text). `get_stele_from_request` here — shared with `date` — is the multi-tenancy mechanism described in §8.

**`serve/`** implements the "current documents" routes: checks the startup-computed redirect flag before doing anything else, resolves the blob at `HEAD` (falling back to the stele's fallback repository if the primary one doesn't have the path), and serves it with `ETag` support.

**`utils.rs`** (this level, not `crate::utils`) is one function: `convert_vec_u8_to_hashmap`, turning a `redirects.json`'s array-of-pairs format into a map — used by `RepoData::get_redirects`.

---

## 6. `src/history/` and `src/redirects/`

**`history/changes.rs`** is `stelae update`'s implementation. Per included stele (respecting `--include`/`--exclude`), inside its own database transaction:
1. Refreshes that stele's redirects in the database (see §8) — every `stelae update` run, not just history import, keeps redirects current.
2. Finds the stele's RDF repository (`custom.type == "rdf"`); a stele without one is skipped for history purposes (its redirects are still processed).
3. Decides whether the stele's existing database rows are trustworthy or need a full rebuild (`is_stele_inconsistent` — three checks, detailed in §8), or rebuilds unconditionally if `--force` was passed.
4. Walks the RDF repository's `_publication/` directory, parsing each publication's RDF graph (via `sophia`) to extract publication metadata, per-document and per-library change objects, and revoking any earlier publication that shares a date with a newer one.
5. Records the auth repository's commit history against each historical HTML data repository, matching each commit's target-metadata `branch` field to a `publication/<date>` naming convention.

`history/rdf/graph.rs` and `namespaces.rs` weren't available to read directly, but their usage in `changes.rs` is clear enough to describe: `graph.rs` wraps `sophia`'s RDF graph with convenience lookups (`literal_from_triple_matching`, `iri_from_triple_matching`, and their `all_*` plural forms) plus an RDF-list `Bag` abstraction; `namespaces.rs` defines the predicate vocabulary used throughout (`oll::codifiedDate`, `oll::hasChanges`, `oll::archivedHtml`, `oll::status`, `dcterms::available`, and others).

**`redirects/mod.rs`** is small: `insert_redirects_for_stele` walks every repository a stele has, reads each one's committed `redirects.json` (via `RepoData::get_redirects`), and bulk-inserts the pairs into the database through `insert_redirect_pairs`. This is the write side of the mechanism `serve/` reads from at request time.

---

## 7. `src/db/` — Persistence

Not read directly this pass; described from `db::models::*` names, the migration filenames, and how handlers above use it.

`db/models/` has one folder per table, each pairing a struct (`mod.rs`) with its queries (`manager.rs`, generally split into a plain `Manager` for read-only handler code and a `TxManager` for transactional writes during `update`): `changed_library_document`, `data_repo_commits`, `document`, `document_change`, `document_element`, `library`, `library_change`, `publication`, `publication_has_publication_versions`, `publication_version`, `redirects`, `status`, `stele`, `version`.

The migration filenames give a rough history: an initial schema, then `add_commits` (`data_repo_commits`), `add_target_dates` (matching `TargetsMetadata`'s build/codified dates), `add_redirects_table` (the DB-backed redirect mechanism replacing static route registration), and `add_html_data_repo_name` (the column `changes.rs` uses to track which HTML repository a publication's content lives in, including backfilling it across a repository migration boundary).

`db::DatabaseConnection`, `db::DatabaseTransaction`, and the `db::Tx` trait (used as `conn.pool.begin()` / `tx.commit()` / `tx.rollback()`) live in `db/mod.rs`/`db/init.rs`, not read directly.

---

## 8. Notable Cross-Cutting Mechanisms

A few things only make sense once several files are read together:

- **Redirects have a source and a cache.** A `redirects.json` committed inside a data repository is the source of truth. `stelae update` (`redirects::insert_redirects_for_stele`) reads it out of Git and writes it into the `redirects` table. `serve` reads only the database, guarded by a startup-computed `has_redirects` flag so repositories with no redirects never pay a lookup cost. Nothing at serve time reads `redirects.json` directly.

- **Multi-tenancy is header-based, not path-based.** `get_stele_from_request` (shared by `date` and `versions`) reads an `X-Stelae` header to decide which stele's data to answer with, defaulting to the archive's root stele if absent. This is separate from the private-archive guard below — one picks *which* stele, the other decides *whether you're allowed to see it at all*.

- **The private-archive guard is a different, config-driven mechanism.** If `.taf/config.toml`'s `[headers]` section sets `current_documents_guard`, both `_archive` and the current-documents (`serve`) routes are registered once per stele behind a required header whose value must match that stele's qualified name. Without that config, the same routes are open.

- **Underscore-prefixed routes are reserved for the root.** A dependent stele's `repositories.json` can list `routes` starting with `_`, but `register_dependent_routes` explicitly skips them — they're expected to be handled once, by the root stele (this is the `_xml`, `_rdf`, `_archive`, `_date`, `_api` convention seen throughout).

- **`stelae update` self-heals rather than requiring manual intervention.** `is_stele_inconsistent` checks three things before trusting existing rows: whether the RDF repository has fewer publications at `HEAD` than the database has non-revoked ones (a sign of a force-push or rollback), whether any non-revoked publication is missing `html_data_repo_name` while the stele currently has an HTML repository (a column added after some rows existed, which can't be backfilled incrementally), and whether the most recently recorded auth-repo commit for each historical data repository still exists in that repository. Any of the three triggers a full delete-and-rebuild for that stele, same as passing `--force`.

---

## 9. `tests/` Layout

| Path | Purpose |
|---|---|
| `archive_testtools/` | Shared fixture-building helpers: constructing temporary archives (`mod.rs`), pre-built data-repository configurations (`config.rs`), misc test utilities (`utils.rs`). |
| `api/` | Integration tests against a running Actix test service, roughly one file per feature area. |
| `basic/` | Lower-level tests that don't go through the HTTP layer. |
| `common/` | Shared test setup (`initialize_archive`, `initialize_app`, `TestAppState`); `db_data/` seeds database fixtures for tests needing historical data. |
| `fixtures/` | Committed static test data: a full pre-built archive, and raw static files used when constructing data repositories on the fly. |

---

## 10. Still Unconfirmed

`main.rs`, `lib.rs`, `db/mod.rs`, `db/init.rs`, `server/headers.rs`, `server/tracing.rs`, `history/rdf/graph.rs`, `history/rdf/namespaces.rs`, and the `request`/`response` submodules under `api/versions`, `api/date`, `api/archive`. Everything above involving these was inferred from how they're *used* elsewhere rather than read directly — solid enough to navigate by, but worth a pass once those files are available.
