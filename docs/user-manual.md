# 📗 Stelae User Manual

---

## 1. What is Stelae

Stelae is a system for publishing, preserving, and authenticating legal documents online. It takes documents stored in Git repositories and serves them as a browsable, linkable website, while keeping a verifiable record of what changed and when.

A few ideas are worth knowing before going further:

- **Archive**: the top-level folder that holds everything Stelae needs to run: its configuration and all the repositories it serves.
- **Stele**: one organisation's set of legal documents within an archive, made up of a root repository and the content repositories it lists. An archive can contain more than one stele, and a stele can depend on other steles.
- **Root repository**: the main repository of a stele. It doesn't hold the documents themselves; it holds the configuration that tells Stelae which content repositories to serve and how.
- **Content repository**: a repository holding the actual documents, in a particular format (for example, web pages, structured data, or another format a partner chooses to publish in).

For how these pieces fit together internally, see the architecture documentation (forthcoming).

---

## 2. Recommended Use of `taf`

Repositories are not meant to be cloned by hand. Stelae uses `taf` (The Archive Framework) to set up and maintain an archive because it does two things a plain `git clone` does not:

1. **Discovery** — it reads a stele's configuration and automatically fetches every repository it depends on, so you don't have to find and clone each one yourself.
2. **Verification** — it checks that each repository is authentic and hasn't been tampered with.

Because of this, `taf` should be your default way of creating and updating an archive:

- `taf repo init` — sets up a new, empty archive.
- `taf repo clone <url>` — clones a stele and everything it depends on.

Cloning repositories manually with `git clone` is possible as a fallback, but it skips the verification step and requires you to track down every dependency yourself. Treat it as a last resort, not a routine choice.

---

## 3. File Structure

An archive follows a consistent layout. This section describes it in general terms; the [Quickstart Guide](quickstart.md) shows it filled in for a real example.

```
archive/
├── .taf/
│   └── config.toml         # Says which stele is the root of this archive
└── <organisation>/
    ├── <root repository>/
    │   └── targets/
    │       ├── repositories.json   # Lists this stele's content repositories
    │       ├── dependencies.json   # Lists other steles this one depends on (if any)
    │       └── protected/
    │           └── info.json       # Identifies this stele
    ├── <content repository>/
    └── ...
```

A few notes on what these pieces mean:

- **`.taf/config.toml`** points to the archive's root stele — which organisation and which repository within it.
- **`targets/repositories.json`**, inside the root repository, is the source of truth for what that stele serves: which content repositories exist, what kind of documents each holds, and where each is reachable on the site.
- **`targets/dependencies.json`** is only needed when a stele relies on another organisation's stele as well as its own. Most archives with a single organisation won't have one.
- **`targets/protected/info.json`** identifies the stele itself (its name and the organisation it belongs to).
- Each **organisation folder** can hold more than one repository: the root repository plus every content repository it lists.

You are not expected to write these files by hand as a matter of course — `taf` and the tools your partner provides will usually produce them. Understanding the layout mainly helps when something needs troubleshooting.

---

## 4. Commands and General Usage

Stelae is used from the command line. Every command is run from the archive's root folder (the one containing `.taf/`), unless you point it elsewhere with `--archive-path`.

| Command | What it does |
|---|---|
| `stelae serve` | Starts the web server, making the archive browsable and its API reachable. |
| `stelae check` | Looks over the archive and reports anything missing or misconfigured, without starting a server or changing anything. |
| `stelae update` | Updates stored historical information for the archive. |
| `stelae git` | Serves the archive's repositories directly over Git, so they can be cloned. |

A typical day-to-day flow is: clone or update an archive with `taf`, run `stelae check` to confirm it's in good shape, then run `stelae serve` to make it available.

### `stelae check`

Since an archive can depend on other archives, `stelae check` looks through the whole chain, not just the one you're standing in. Each thing it finds is reported against the specific stele and file it belongs to, so you know exactly where to look.

What it finds falls into two kinds:

- **Errors** — something is missing or wrong in a way that would likely cause problems once the archive is served. If there are any errors, the check fails.
- **Warnings** — worth knowing about, but not serious enough to fail the check on their own.

Because a failed check exits with a non-zero status, it fits naturally into an automated pipeline: run it as a step before deploying or serving an archive, and let it stop the process if something's wrong, rather than finding out after the fact.

---

## 5. Example

*(This section will be filled in with a complete walkthrough later. For now, the [Quickstart Guide](quickstart.md) covers the same ground with concrete commands and file contents for a generalised example archive.)*

---
