# Forge

**The Universal Linux Package Manager.**

Forge installs packages the way *you* want them installed: compiled from source with your own flags, cloned straight from a git repo, pulled as a pre-built binary from a repo mirror, or extracted directly from another distro's own package format (`.pkg.tar.zst`, `.deb`, `.rpm`) — all through the same recipe-based workflow, no matter which one it is.

It started as a companion tool for a [Linux From Scratch](https://www.linuxfromscratch.org/) build (see [History](#history) below) and grew into a standalone, 100% Bash package manager that works on any Linux system — LFS, Arch, Debian, Fedora, or a barebones Termux install.

> [!CAUTION]
> Do not run `sudo rm -rf /` LOL 😂
> (seriously, don't try this, I even gave the wrong command purposely)

---

## Table of contents

- [Features](#features)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Commands](#commands)
- [Recipes](#recipes)
- [The pre-built package repo](#the-pre-built-package-repo)
- [Configuration reference](#configuration-reference)
- [Sandbox mode](#sandbox-mode)
- [Languages](#languages)
- [History](#history)

---

## Features

- **Recipe-based**, like Gentoo's Portage — a small Bash file per package (`NAME`, `VERSION`, `URL`, and a couple of shell functions) drives the whole build.
- **`sketch`** auto-generates a recipe for you by scraping the LFS/BLFS/SLFS/GLFS/MLFS books, `download.gnome.org`, a git repo, or your distro's own package index (pacman/apt/dnf) — see [Recipes](#recipes).
- **Any source, one workflow**: tarball, git clone, or a foreign binary package (`.pkg.tar.zst`/`.deb`/`.rpm`) extracted directly — no external package manager ever gets shelled out to.
- **A resumable build pipeline** (`heat()` → `mold()` → `smith()`) that survives Ctrl+C, closed terminals, and power loss without starting from scratch.
- **A pre-built package cache**, backed by a plain git repo you control — build once, reuse everywhere, optionally GPG-signed and shared across machines.
- **Dependency resolution** with automatic parallel builds for independent packages in the same dependency tree.
- **Orphan tracking** (`forge orphans`, `melt --with-deps`) — the same idea as `pacman -Qdt`/`-Rns`.
- **Post-install hooks** (`ldconfig`, `glib-compile-schemas`, `update-desktop-database`, icon/MIME caches, `mandb`...) run automatically, based on what a package actually installed.
- **Runs anywhere**, root or not — a real write-access test (not a platform check) automatically sandboxes Forge under `$HOME/.forge` when it can't touch the real system, so the whole thing works out of the box on Termux too.
- **9 languages** out of the box (see [Languages](#languages)).
- **Zero compiled dependencies.** No Makefile, no build step — Forge itself is a single Bash script.

## Installation

Clone the repo:

```bash
git clone https://github.com/johnfg-archbtw/forge
```

> [!NOTE]
> If you're looking for a Makefile or a compile step, there isn't one — that's the point. This project is 100% Bash.

Then run the install script as root:

```bash
cd forge/
sudo ./install-forge
```

### Requirements

- `bash`, `wget`, `tar`, `git`
- `python3` (used by `sketch` for HTML/XML parsing)
- Format-specific, only if you actually use them: `unzip` (.zip sources), `ar` (.deb), `rpm2cpio`+`cpio` or `bsdtar` (.rpm), `zstd`/`unzstd` (zstd-compressed metadata/archives)
- `whiptail` or `dialog` for `forge menu`

## Quick start

```bash
# Look up a package and generate a recipe for it automatically
forge sketch firefox

# Review it, then check it's sane
forge assay firefox

# Build and install
forge smith firefox

# See what's installed
forge wares

# Update everything that has a newer version upstream
forge probe --all --update
forge temper --all
```

## Commands

| Command | What it does |
|---|---|
| [`smith`](#smith) | Builds and installs a package from its recipe |
| [`sketch`](#sketch) | Generates a draft recipe automatically |
| [`assay`](#assay) | Validates a recipe before you build it |
| [`probe`](#probe) | Checks upstream for a newer version |
| [`temper`](#temper) | Updates an installed package |
| [`melt`](#melt) | Uninstalls a package |
| [`orphans`](#orphans) | Lists dependencies nothing needs anymore |
| [`search`](#search) | Searches recipes by name/description |
| `wares` | Lists installed packages |
| `patts` | Lists every available recipe (installed or not) |
| `gauge` | Shows a recipe's details without installing it |
| [`repo`](#the-pre-built-package-repo) | Manages the pre-built package cache |
| `menu` | An interactive whiptail/dialog front-end for all of the above |

Run `forge help` at any time for the full built-in reference.

### smith

```
forge smith <pkg> [-y|--yes] [-f|--force] [--dry-run] [--clean|--fresh]
                   [--no-cache] [--no-store] [--no-strip] [--keep-la]
                   [--with-<opt>|--without-<opt> ...]
```

Downloads (or, for a `GIT_URL` recipe, clones; or, for a `PKG_FORMAT` recipe, downloads-and-extracts), **heats** (configures), **molds** (builds), and **smiths** (installs) a package from its recipe under `/etc/forge/recipes/`.

| Flag | Effect |
|---|---|
| `-y`, `--yes` | Skip confirmation prompts. Does nothing if the version is already installed. Answers any `OPTIONS` prompt with its own default. |
| `-f`, `--force` | Force a reinstall even if already up to date. |
| `--dry-run` | Show what would happen — nothing is downloaded, built, or installed. |
| `--clean`, `--fresh` | Discard any in-progress build and start over, even if a matching pre-built package exists in the repo. |
| `--no-cache` | Never use a pre-built package for this run — always build from source. |
| `--no-store` | Don't cache a freshly-built package afterward. |
| `--no-strip` | Keep debug symbols in installed binaries/libraries. |
| `--keep-la` | Keep libtool `.la` files instead of deleting them. |
| `--with-<opt>`, `--without-<opt>` | Answer one of the recipe's `OPTIONS` on the command line instead of being prompted. |

**Resumable builds.** If `heat()`/`mold()` gets interrupted (Ctrl+C, closed terminal, power loss) or fails partway through, the extracted source tree is kept as-is — running `forge smith <pkg>` again picks up right where it left off. Forge tracks two checkpoints (whether the source is extracted/cloned, and whether `heat()`/`mold()` each finished once); this isn't line-by-line resumption, it relies on the build system's own incrementality (make/ninja/cargo/etc.) to skip what's already done. Only resumes for the *same* `VERSION` — a changed recipe starts fresh automatically. Use `--clean` to discard an in-progress build on purpose.

**Cleanup.** As soon as `smith()` finishes staging a build, the entire source/build tree is deleted immediately — not held onto until the whole run finishes. This matters most for packages like LLVM or Firefox, whose build directory can be many times the size of what actually ends up installed. Forge also strips debug symbols and deletes libtool `.la` files before a package is installed or cached (standard LFS/BLFS practice, doesn't affect runtime). Turn either off with `--no-strip`/`--keep-la`, or permanently via `STRIP_BINARIES=0`/`REMOVE_LA_FILES=0` in `forge.conf`.

**Hooks.** Right after installing files, `smith` checks the manifest against `FORGE_HOOKS` (a `pattern::command` table, editable in `forge.conf`) and runs whatever matches — `glib-compile-schemas`, `update-desktop-database`, `gtk-update-icon-cache`, `mandb`, `ldconfig`, `update-mime-database`. Each command runs at most once per install; a hook whose command isn't on `PATH` is skipped with a note.

**History.** Every successful install/upgrade/removal appends a line (timestamp, action, name, version) to `HISTORY_FILE`. It's a plain record for your own reference — nothing reads it back for rollback.

**Dependency cascade.** Independent packages at the same point in the dependency tree build concurrently (capped at `MAX_PARALLEL_JOBS`, default `nproc`) instead of one at a time. Set `PARALLEL_DEPS=0` to force strictly sequential installs.

### assay

```
forge assay <pkg>
```

Validates a recipe before you ever `smith` it: checks `NAME`/`VERSION`/`URL` are set, `mold()`/`smith()` exist (or that the recipe is a `PKG_FORMAT`/meta-package, which don't need them), dependencies have matching recipes, and warns about a missing checksum or CRLF line endings.

Checksums can also live outside the recipe, in `/etc/forge/checksums/<pkg>.sha256sum` or `.md5sum` (just the hash, one line) — these take priority over `SHA256SUM`/`MD5SUM` set inside the recipe itself.

### probe

```
forge probe <pkg|--all> [--update] [-y|--yes]
```

Checks upstream for a newer version without installing anything. Requires the recipe to define:

```bash
VERSION_CHECK_URL="<page listing available versions/tarballs>"
VERSION_REGEX='<grep -oP pattern that prints ONLY the version>'
```

Use `\K` to drop everything matched before the version, and a `(?=...)` lookahead to drop what comes after. Example, for a directory listing full of links like `gdk-pixbuf-2.42.12.tar.xz`:

```bash
VERSION_REGEX='gdk-pixbuf-\K[0-9]+\.[0-9]+\.[0-9]+(?=\.tar\.xz)'
```

Recipes missing either field are skipped. With `--update`, rewrites `VERSION="..."` in the recipe file (keeping a `.bak`) and refreshes the checksum via `CHECKSUM_URL`. Doesn't install anything — follow up with `forge temper <pkg>`.

### sketch

```
forge sketch <pkg> [--source lfs|blfs|slfs|glfs|mlfs|gnome|git|pacman|apt|dnf] [-s <source>]
                    [--url <page, tarball, or git repo>] [--ref <tag/branch/commit>]
                    [--category <dir>] [--force]
```

Generates a **draft** recipe automatically. `--book` is accepted as an alias for `--source`.

**Book sources** (`lfs`/`blfs`/`slfs`/`glfs`/`mlfs`): scrapes the matching book page for the download link, version, checksum, patches, and the book's own install commands, splitting them into `heat()`/`mold()`/`smith()`/`finish()`. Matching is **exact** on the page's own URL slug (not its link text) — `gtkmm` will never silently resolve to whatever `gtkmm3`/`gtkmm4` page happens to come first; on a total miss, sketch lists every slug that merely *contains* your search as a "Did you mean" suggestion. Without `--source`, all five book sources are tried in order, then `gnome`.

**`gnome`**: looks up `<pkg>` directly under `download.gnome.org/sources/<pkg>/`, picks the newest series directory (alpha/beta included, intentionally) and the newest tarball inside it.

**`git`** (needs `--url <repo>`): for a package whose released tarballs lag behind its actual git history. Shallow-clones the repo and detects the build system straight from the checked-out tree. Without `--ref`, picks the newest tag; with none, falls back to the default branch's HEAD (version becomes the short commit hash). The recipe gets `GIT_URL`/`GIT_REF` instead of `URL` — see [Recipes](#recipes).

**`pacman`/`apt`/`dnf`**: resolves `<pkg>` to a real download URL for its native binary package format, preferring your system's local package index (instant, no network) and falling back to fetching the repo metadata directly from a configurable mirror otherwise — see [Configuration reference](#configuration-reference). The result feeds straight into the same `PKG_FORMAT` handling `--url` would give you pointed directly at a `.pkg.tar.zst`/`.deb`/`.rpm`.

**Generic build detection**: whenever there's no page to scrape commands from (a direct `--url` tarball, `gnome`, or `git`), sketch downloads the tarball/repo and looks at its top-level files to guess `heat()`/`mold()`/`smith()`, in this priority: `meson.build` > `configure` > `autogen.sh` > `setup.py`. Always review before smithing.

**`--url`** also accepts a direct link straight to a tarball (recognized by extension) or a `.pkg.tar.zst`/`.deb`/`.rpm` — no book/GNOME/git lookup needed.

`--category` places the recipe under `RECIPE_DIR/<dir>/` instead of the top level. `--force` overwrites an existing recipe at that path.

The generated recipe is always a **draft** — run `forge assay <pkg>` and read through it yourself before smithing.

### temper

```
forge temper <pkg|--all> [-y|--yes]
```

Updates a package to the version in its recipe, only if it differs from what's installed. With `--all`, checks every installed package, lists what's outdated, asks once, then updates them all.

### melt

```
forge melt <pkg> [-y|--yes] [--with-deps]
```

Uninstalls a package: removes every file/directory it's known to have installed, then deletes its record. Warns first if another installed package depends on it.

`--with-deps` — after removing `<pkg>`, checks whether that left any of its own dependencies orphaned and offers to remove those too (same idea as `pacman -Rns`).

### orphans

```
forge orphans
```

Lists installed packages that were pulled in only as a dependency and that nothing currently installed needs anymore (`pacman -Qdt`'s equivalent). Chains cascade correctly — if removing one orphan would orphan another, both show up. Doesn't remove anything by itself; see `melt --with-deps`.

### search

```
forge search <term>
```

Case-insensitive search over every recipe's name/description in `RECIPE_DIR` (not just installed ones). For a package not in any local recipe yet, try `forge sketch <term>` instead.

## Recipes

A recipe is a small Bash file. The minimum:

```bash
NAME="hello"
VERSION="2.12.1"
URL="https://ftp.gnu.org/gnu/hello/hello-${VERSION}.tar.gz"
SHA256SUM="..."

heat() {
	./configure --prefix=/usr
}

mold() {
	make
}

smith() {
	make DESTDIR="$1" install
}
```

- **`heat()`** (optional) — configuration (`./configure`, `meson setup`, `cmake`, `autogen.sh`), tracked as its own resumable step. Recipes from `sketch` always define one (even a no-op note) so a resumed build never silently skips reconfiguring.
- **`mold()`** — the build step.
- **`smith()`** — installs into the staging directory (`$1`), typically via `DESTDIR`.
- **`finish()`** (optional) — runs *after* the package is copied onto the real filesystem, for steps that only make sense on the real installed path (e.g. `setcap` on a final binary).
- **`DEPS`** (optional) — an array of other recipe names this one depends on. Resolved and installed automatically, in parallel where the dependency tree allows it.
- **`OPTIONS`** (optional) — build-time choices, e.g.:
  ```bash
  OPTIONS=("doxygen:Build API documentation with Doxygen:n")
  ```
  `mold()`/`smith()` read the choice from `$OPT_DOXYGEN` ("y"/"n"). Resolved from `--with-<key>`/`--without-<key>` on the command line, then the recipe's own default under `-y`, then an interactive `[y/N]` prompt.

**Alternative sources**, instead of a tarball `URL`:

- **`GIT_URL`** (+ optional `GIT_REF`) — sourced via `git clone --depth 1` instead of a tarball download.
- **`PKG_FORMAT`** (`pacman`, `deb`, or `rpm`), alongside a normal `URL` pointing at the actual package file — installs an already-built binary package from another distro's format directly, instead of building anything. `heat()`/`mold()`/`smith()` are **not** called for these. The package's own version (from `.PKGINFO`, `control`, or the RPM header) overrides the recipe's `VERSION` when they disagree. Maintainer scripts (`.INSTALL`, `postinst`, RPM scriptlets) are **not** run automatically — most of what they'd do is already covered by Hooks; `smith` notes when a package had one.

A recipe with no `URL`/`GIT_URL`/`PKG_FORMAT` at all is a **meta-package** — `mold()`/`smith()` (if defined) run directly against the real system, useful for grouping dependencies or running steps that don't need a source at all.

## The pre-built package repo

```
forge repo <init|pull|push|status|list> [pkg]
```

Before building, `smith` checks a plain git repo (`REPO_DIR`) for a package already built with the exact same recipe content and `OPTIONS`. On a match, it skips the download and `mold()` entirely; on a miss, it builds normally and caches the result (unless `--no-store`).

| Subcommand | Effect |
|---|---|
| `init` | Creates `REPO_DIR` as a git repo, wiring up `REPO_REMOTE` as `origin` if set. |
| `pull` | Fetches newer pre-built packages from the remote. |
| `push` | Pushes local commits to the remote (also automatic if `REPO_AUTO_PUSH=1`). |
| `status` | Shows the repo path, remote, auto-push setting, and uncommitted changes. |
| `list [pkg]` | Lists cached builds, with version, config ID, and the `OPTIONS` that produced each. |

Set `REPO_GPG_KEY` to GPG-sign every commit, and `REPO_GPG_VERIFY=1` to refuse a cached build whose commit isn't verifiably signed — matters once `REPO_REMOTE` is shared with a machine/person you don't fully trust the write access to.

## Configuration reference

All of these live in `/etc/forge/forge.conf` (sourced as plain Bash — anything is overridable as an environment variable too).

| Variable | Default | Purpose |
|---|---|---|
| `RECIPE_DIR` | `/etc/forge/recipes` | Where recipes live |
| `LOG_DIR` | `/var/log/forge` | Manifests, build logs, history |
| `CHECKSUM_DIR` | `/etc/forge/checksums` | Out-of-recipe checksum files |
| `WORKDIR` | `/var/tmp/forge` | Scratch space for builds |
| `INSTALL_ROOT` | `/` | Where packages actually get installed |
| `LANG_DIR` | `/etc/forge/lang` | Message catalogs |
| `EXTRA_PATH` | *(empty)* | Extra dirs prepended to `PATH` for `heat()`/`mold()`/`smith()` |
| `REPO_DIR` | `/var/cache/forge/repo` | The pre-built package cache |
| `REPO_REMOTE` | *(empty)* | Git remote for the repo |
| `REPO_AUTO_PUSH` | `0` | Push automatically after every cached build |
| `REPO_GPG_KEY` | *(empty)* | GPG-sign repo commits with this key |
| `REPO_GPG_VERIFY` | `0` | Refuse unsigned/unverifiable cached builds |
| `HISTORY_FILE` | `$LOG_DIR/history.log` | Install/update/removal log |
| `PARALLEL_DEPS` | `1` | Build independent dependencies concurrently |
| `MAX_PARALLEL_JOBS` | `nproc` | Cap on concurrent dependency builds |
| `STRIP_BINARIES` | `1` | Strip debug symbols after building |
| `REMOVE_LA_FILES` | `1` | Delete libtool `.la` files after building |
| `PACMAN_MIRROR` | `geo.mirror.pkgbuild.com` | Fallback mirror for `sketch --source pacman` |
| `PACMAN_REPOS` | `core extra multilib` | Repos to check |
| `APT_MIRROR` | `deb.debian.org/debian` | Fallback mirror for `sketch --source apt` |
| `APT_SUITE` | `stable` | Suite/codename to check |
| `APT_COMPONENTS` | `main` | Components to check |
| `DNF_MIRROR` | `dl.fedoraproject.org/.../releases` | Mirror for `sketch --source dnf` |
| `FEDORA_RELEASEVER` | *(auto-detected)* | Pin a specific Fedora release instead |
| `DNF_REPOS` | `Everything` | Repos to check |
| `DNF_ARCH` | `x86_64` | Architecture to check |
| `FORGE_SANDBOX` | *(auto-detected)* | Force sandbox mode on/off — see below |
| `FORGE_LANG` | *(auto-detected from locale)* | Force a specific UI language |

`FORGE_HOME` is a shortcut that redirects *all* of the path-related variables above at once (used internally by sandbox mode).

## Sandbox mode

If Forge can't actually write to the real system (most commonly: not running as root), it automatically redirects every path — recipes, logs, checksums, work dir, lang files, lock file, repo cache — to a sandbox under `$HOME/.forge/`, and "installed" packages land under `$HOME/.forge/root` instead of the real filesystem.

This is checked with a **real write test**, not a platform sniff, so it covers Termux (no root, no real `/etc` or `/usr` on Android), a regular user account without `sudo`, a restricted container, or anything else. It's meant for trying recipes and the whole `smith`/`sketch`/`probe` flow out without needing root, not for a permanent non-root install.

Set `FORGE_SANDBOX=0` to force real system paths regardless, or `FORGE_SANDBOX=1` to force the sandbox even when running as root. `forge menu` also accepts `dialog` in place of `whiptail`, since that's what Termux packages.

## Languages

Forge's own messages (not recipe content) are available in 9 languages, auto-detected from your locale: English, Portuguese (Brazil), Spanish, French, German, Italian, Russian, Japanese, and Chinese (Simplified). Set `FORGE_LANG` to override.

## History

This idea originated from [Linux From Scratch](https://www.linuxfromscratch.org/) (LFS), when I successfully followed the whole book and booted into the OS.

![Linux From Scratch Logo](https://www.linuxfromscratch.org/images/linux-from-scratch.png)

After installing internet drivers, then WPA Supplicant and DHCP, I realized that typing out every command one after another — build, install, repeat for the next package — took way too long, and the idea of an automatic package installer popped into my head.

> [!NOTE]
> Before you bring up Automated Linux From Scratch (ALFS): that project hasn't been updated since 2017. 😭 I don't even know how to use it — there's no up-to-date documentation on how to build it, and every Makefile I found there just fails.

So I built a small recipe-based package manager instead (like Gentoo's Portage, which relies on `ebuild` files) — I'd still have to write every command by hand into a recipe once, but that's WAY better than re-typing them every single time, especially once you get into packages like GCC, LLVM, or Qt 6 that really do take that long to build.

Later on, I added a subcommand that fetches a package's page directly from the BLFS book and scrapes it for the right HTML patterns to write the recipe file **automatically**. That's `sketch`.

From there, `sketch` grew to support LFS's other books too — SLFS and LFS itself, and much later GLFS and MLFS — and eventually outgrew the book format entirely: GNOME's own source listings, git repos for packages whose releases lag behind upstream, and finally direct support for pulling binary packages straight from Arch, Debian, and Fedora's own repositories.

Then, after months of that Linux From Scratch build, I made one wrong keystroke installing a bootloader from a Live CD — `limine bios-install /dev/sda3` instead of `/dev/sda` — and wiped the entire drive with no way back. The LFS system was gone for good.

Forge wasn't, though. It had already become its own project by then, so I kept building it — now on Arch.

> [!TIP]
> I use Arch btw
> ![Arch Linux Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f9/Archlinux-logo-standard-version.svg/1280px-Archlinux-logo-standard-version.svg.png)
