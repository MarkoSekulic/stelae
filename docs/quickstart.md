# 📘 Stelae Quickstart Guide

## Setting Up a Local Stelae Server

---

## 1. Introduction

**Stelae** is a system for distributing, preserving, and authenticating legal documents. It uses Git repositories as its storage backend and provides both a web interface and an API for accessing legal content.

This guide walks you through setting up a local Stelae server for **a partner's archive**. By the end, you will have a working local instance serving that partner's published legal documents.

Throughout this guide, placeholders are written in angle brackets:

| Placeholder | Meaning |
|-------------|---------|
| `<org>` | The partner's organisation name. This is also the name of the folder that holds their repositories. |
| `law` | The name of the partner's root repository. This is the usual name, but yours may differ. |

For a broader explanation of concepts and commands, see the [User Manual](user-manual.md).

---

## 2. System Requirements

### 2.1 Required Software

| Software | Purpose | Installation Link |
|----------|---------|-------------------|
| **Rust** | Compiles Stelae from source | [rustup.rs](https://rustup.rs/) |
| **Git** | Clones repositories | [git-scm.com](https://git-scm.com/) |
| **C++ Build Tools** | Required by Rust compiler | Windows: [C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)<br>Linux: `sudo apt-get install build-essential pkg-config libssl-dev` |
| **Python 3 + pip** | Installs `taf` | [python.org](https://python.org/) |
| **taf** | The Archive Framework – clones and verifies repositories | `pip install taf` |

### 2.2 Optional Tools

| Software | Purpose |
|----------|---------|
| **just** | Task runner used in Stelae development (install with `cargo install just`) |

---

## 3. Step-by-Step Installation

### 3.1 Build Stelae

First, clone the Stelae source code and compile it:

```bash
git clone https://github.com/openlawlibrary/stelae.git
cd stelae
cargo build --release
```

**Compilation Time**: This may take 5–15 minutes depending on your system.

After compilation, the executable is located at `target/release/stelae`. For convenience, add this directory to your system's `PATH`:

```bash
export PATH="$PATH:/path/to/stelae/target/release"  # Linux/macOS
```
or on Windows, add it to your system environment variables.

---

### 3.2 Create the Archive Directory

An **archive** is the root folder that Stelae serves. It contains your configuration and all Git repositories.

Create a new directory and navigate into it:

```bash
mkdir stelae_archive
cd stelae_archive
```

---

### 3.3 Initialise the Archive with `taf`

The `taf` (The Archive Framework) tool creates the required directory structure:

```bash
taf repo init
```

This creates a `.taf` folder containing a minimal `config.toml` file. The `.taf` folder marks this directory as a valid Stelae archive.

> **Note**: If `taf repo init` fails or doesn't create `.taf/config.toml`, you can create the folder manually:
> ```bash
> mkdir .taf
> touch .taf/config.toml
> ```

---

### 3.4 Configure the Archive Root

Edit `.taf/config.toml` and add the following content:

```toml
[root]
name = "law"
org = "<org>"
```

**Explanation of fields:**

| Field | Description |
|-------|-------------|
| `root.name` | The name of the authentication repository (the "root stele") |
| `root.org` | The organisation or namespace that owns the repository |

**Why this matters**: Stelae uses the `[root]` section to locate the authentication repository at `{org}/{name}` within your archive directory. With the values above, it looks for `<org>/law`.

> **Important**: Do not manually create `targets/repositories.json` in the archive root. The root repository (`law`) already contains this file, and Stelae will read it from there.

---

### 3.5 Clone the Repositories with `taf`

Use `taf repo clone` to clone the root repository and **automatically** fetch all dependent repositories:

```bash
taf repo clone https://github.com/<org>/law.git
```

**What this command does:**

1. Clones `<org>/law` into `<org>/law`
2. Reads the metadata inside `law` to discover all target repositories
3. Automatically clones every dependent repository (for example the partner's HTML, XML, and RDF content repositories)
4. Verifies the authenticity of each commit

**Alternative approach** (if `taf repo clone` fails):

If `taf` is not working correctly due to outdated documentation or version mismatches, you can manually clone the required repositories:

```bash
git clone https://github.com/<org>/law.git <org>/law
git clone https://github.com/<org>/law-html.git <org>/law-html
```

> **Note**: Cloning manually skips the authenticity checks that `taf` performs.

**At minimum**, you need:
- The root repository (`law`)
- At least one content repository (for example `law-html`)

---

### 3.6 Verify the Directory Structure

After cloning, your archive should have the following structure:

```
stelae_archive/
├── .taf/
│   └── config.toml                # Contains [root] name="law" org="<org>"
└── <org>/
    ├── law/                       # Root stele (authentication repository)
    │   ├── targets/
    │   │   ├── repositories.json  # Lists the content repositories and how they are served
    │   │   └── ...
    │   └── ...
    ├── law-html/                  # Content repository (auto-cloned by taf)
    ├── law-xml/                   # Content repository (auto-cloned by taf)
    └── ...                        # Other content repositories; these vary by partner
```

---

### 3.7 Check the Archive

Before starting the server, confirm that the archive is set up correctly. From the archive root (`stelae_archive`), run:

```bash
stelae check
```

Stelae looks through the archive, including any other archives it depends on, and reports anything that is missing or misconfigured. It only reads the archive; it does not start a server or change anything.

Findings come in two kinds:

| Kind | Meaning | Effect |
|------|---------|--------|
| **Error** | Something is wrong and is likely to cause problems when serving | The check fails |
| **Warning** | Worth a look, but not blocking | The check still succeeds |

Each finding names the stele and the file it relates to. Fix any errors and run the command again until none remain.

> **Tip**: When errors are found, `stelae check` exits with a non-zero code, so it works well in CI pipelines and other automated scripts.

---

### 3.8 Start the Stelae Server

From the archive root (`stelae_archive`), run:

```bash
stelae serve --port 8080
```

**Expected output**:

```
Running Publish Server on a Stelae archive at '.' on http://127.0.0.1:8080.
```

If you see this, the server is running successfully.

> **Troubleshooting**: If you get an error about `root` being missing, ensure your `.taf/config.toml` contains the `[root]` section as shown in Section 3.4.

---

### 3.9 Browse the Law Library

Open your web browser and navigate to:

```
http://127.0.0.1:8080/
```

You should see the partner's law library homepage, with links to the documents they publish.

All links should work because Stelae routes requests to the correct repositories based on the configuration in `<org>/law/targets/repositories.json`.

---

## 4. API Endpoints

Stelae provides several API endpoints for programmatic access:

| Endpoint | Description |
|----------|-------------|
| `/_api/versions/` | Lists available publication versions (requires `publications.json`) |
| `/_publication/{name}/` | Serves files from a specific publication |
| `/_date/YYYY-MM-DD/` | Serves files as they existed on a given date |
| `/_git/{org}/{repo}/` | Exposes the raw Git repository (can be cloned) |

**Example**: To view a document as it existed on January 1, 2024:

```
http://localhost:8080/_date/2024-01-01/<path-to-document>
```

---

## 5. Troubleshooting

### 5.1 Common Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `missing field 'root'` | `.taf/config.toml` is missing the `[root]` section | Add `[root]` with `name` and `org` as shown in Section 3.4 |
| `failed to resolve path ... The system cannot find the path specified` | A repository is missing | Re-run `taf repo clone` or manually clone the missing repository |
| `stelae check` reports errors | The archive has missing or misconfigured files | Read each message (it names the stele and file), fix the problem, and run the check again |
| `No publications found for stele: ...` | Root repository lacks `targets/publications.json` | **Optional** – not required for web browsing. Only needed for `/_api/versions/` (see Section 6) |
| `404` on `_reader/`, `_analytics/`, etc. | Frontend assets not present in the repository | These are external assets that may not be served locally. Ignore if the main page works. |
| `Connection refused` | Server not running | Ensure `stelae serve` is running and the port is correct |
| `taf: command not found` | `taf` not installed | Run `pip install taf` |

### 5.2 Debugging with Logs

To get detailed debug information, start the server with:

```bash
RUST_LOG="debug" stelae serve --port 8080
```

This will show you:
- Which repositories are being loaded
- Which Git objects are being requested
- Routing decisions

### 5.3 Common Pitfalls

1. **Wrong Working Directory**: Always run `stelae` commands from the archive root (the directory containing `.taf/`).
2. **Missing Repositories**: Verify that `<org>/law` exists and contains `targets/repositories.json`.
3. **Port Conflicts**: If port `8080` is already in use, choose a different port with `--port`:
   ```bash
   stelae serve --port 3000
   ```

---

## 6. Optional: Adding Publications (For API Support)

The `/_api/versions/` endpoint returns a list of named versions of the legal documents. To enable this:

1. Create `<org>/law/targets/publications.json`:

```json
{
  "versions": [
    {
      "name": "latest",
      "date": "2026-08-04",
      "refs": ["HEAD"]
    }
  ]
}
```

2. Restart the server:

```bash
stelae serve --port 8080
```

3. Visit `http://localhost:8080/_api/versions/` to see the published versions.

> **Note**: This is **not required** for the web interface to work. It only enables the version API.

---

## 7. Quick Reference

| Command | Description |
|---------|-------------|
| `taf repo init` | Initialises a new archive directory |
| `taf repo clone <url>` | Clones a repository and all its dependencies |
| `stelae check` | Checks the archive for missing or misconfigured files |
| `stelae serve --port <port>` | Starts the Stelae server |
| `RUST_LOG="debug" stelae serve` | Starts the server with debug logging |
| `git clone <url> <path>` | Manually clones a repository (fallback) |

---

## 8. Summary

| Step | What You Do | What Happens |
|------|-------------|--------------|
| 1 | Build Stelae from source | Compiles the `stelae` binary |
| 2 | Create archive directory | Creates a folder for your Stelae instance |
| 3 | Run `taf repo init` | Creates `.taf/` directory structure |
| 4 | Add `[root]` to `config.toml` | Tells Stelae where to find the root repository |
| 5 | Run `taf repo clone` | Clones the root and all dependent repositories |
| 6 | Run `stelae check` | Confirms the archive is set up correctly |
| 7 | Run `stelae serve` | Starts the web server |
| 8 | Browse `localhost:8080` | Access the law library |

---

## 9. Getting Help

If you encounter issues not covered in this guide:

1. **Run the check**: `stelae check`
2. **Enable debug logging**: `RUST_LOG="debug" stelae serve`
3. **Check the logs** for specific error messages
4. **Verify your directory structure** matches Section 3.6
5. **Ensure all repositories are cloned** (especially `<org>/law`)

For explanations of the concepts and commands used here, see the [User Manual](user-manual.md).

---
