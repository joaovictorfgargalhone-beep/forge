# Forge
Forge - The Universal Linux Package Manager - lets you install any package from any package repo, from source tarball to well-known package managers like Pacman, APT and DNF.

> [!CAUTION]
> Do not run "sudo rm -rf /" LOL 😂 
> (seriously, don't try this, I even gave the wrong command purposely)

# Installation

Clone this repo with the CLI command **git**:

```bash
git clone https://github.com/johnfg-archbtw/forge
```

> [!NOTE]
> If you are quite advanced in Linux, you may notice that there aren't any Makefile nor any compile script, and that's the point, this project is 100% Bash!

Then cd to the forge/ dir, and run the automated installation script, as root:

```bash
cd forge/
sudo ./install-forge
```

# Usage

forge <command> [package] [flags]
forge <wares|patts|menu>

Sandbox mode: if Forge can't actually write to the real system (most
commonly: not running as root), it automatically redirects every path
(recipes, logs, checksums, work dir, lang files, lock file, repo cache)
to a sandbox under $HOME/.forge/, and "installed" packages land under
$HOME/.forge/root instead of the real filesystem. This is checked with
a real write test, not a platform sniff, so it covers Termux (no root,
no real /etc or /usr on Android), a regular user account without sudo,
a restricted container, or anything else. 

It's meant for trying recipes
and the whole smith/sketch/probe flow out away from the actual LFS
machine, not for a permanent non-root install. Set FORGE_SANDBOX=0 to
force real system paths regardless, or FORGE_SANDBOX=1 to force the
sandbox even when running as root. Every path is still an explicit env
var or forge.conf override away regardless (RECIPE_DIR, LOG_DIR,
CHECKSUM_DIR, WORKDIR, LANG_DIR, INSTALL_ROOT, REPO_DIR, FORGE_HOME sets
all of the above at once). 'forge menu' also accepts 'dialog' in place
of whiptail, since that's what Termux actually packages.

## Commands:
   
### smith:

   ```
   smith <pkg> [-y|--yes] [-f|--force] [--dry-run] [--clean|--fresh] [--no-cache] [--no-store] [--no-strip] [--keep-la] [--with-<opt>|--without-<opt> ...]
```
   
Downloads (or, for a GIT_URL recipe, clones), heats (configures),
     molds (builds), and smiths (installs) a package from its recipe (located at /etc/forge/recipes/).

* -y, --yes: skips confirmation prompts. On a fresh install or upgrade, it proceeds automatically; if the same version is already installed, it does nothing (no redundant reinstall). Also answers any OPTIONS prompt with that option's own default.
* -f, --force: forces a reinstall even if the package is already up to date.
* --dry-run: shows what would happen (fresh install / upgrade / already installed, checksum status, dependencies) without downloading, building or installing anything.
* --clean, --fresh: discards any previous partial build for this package/version and start over (download, extract, mold, smith all from scratch). Also forces a rebuild even if a matching pre-built package exists in the repo.
* --no-cache: never use a pre-built package from the repo for this run, even if one matches -- always build from source.
*  --no-store: don't save a freshly-built package into the repo afterward. Has no effect when a cached build was used instead of building.
* --no-strip: keeps debug symbols in installed binaries/libraries (see "Cleanup" below). Useful when you'll be debugging this specific build.
* --keep-la: keeps libtool .la files instead of deleting them (see "Cleanup" below).
* --with-<opt>, --without-<opt>: answers  one of the recipe's OPTIONS on the command line instead of being prompted (see OPTIONS below).


If heat()/mold() get interrupted (Ctrl+C, closed terminal, power loss) or fail partway through, the extracted source tree is kept as-is instead of being wiped -- running 'forge smith <pkg>' again picks up right where it left off. This is NOT line-by-line resumption inside heat()/mold()/smith() -- Forge only tracks three checkpoints: whether the source is already extracted (or, for a GIT_URL recipe, cloned), whether heat() already finished successfully once, and whether mold() already finished successfully once. Whichever of heat()/mold() didn't finish is rerun from its own start on the next attempt (relying on make/ninja/cargo/x.py and most build systems being incremental by nature to skip what's already done, since their own build directory is preserved too); whichever DID finish is skipped entirely. This only resumes for the SAME version -- if the recipe's VERSION changed since the interrupted attempt, it starts fresh automatically. 

Use --clean to discard an in-progress build on purpose (e.g. after editing heat()/mold()/smith(), or if a build got into a bad state) instead of resuming it.

A recipe may optionally define finish(), run after the package is copied onto the real filesystem (not the staging directory). Use it for steps that only make sense on the real installed path, such as setcap on the final binary.

A recipe defines GIT_URL instead of URL to be sourced via 'git clone' (shallow, --depth 1) rather than a tarball download --see 'sketch --source git' above for the easiest way to generate one. GIT_REF (a tag, branch, or commit) is optional; without it, the repo's default branch HEAD is cloned. Everything else about the package (heat()/mold()/smith(), OPTIONS, caching, the repo) works exactly the same either way -- caching in particular still applies, keyed the same way (recipe content + OPTIONS), so a git-sourced package that takes a while to build still only pays that cost once per config.

A recipe defines PKG_FORMAT ('pacman' or 'deb') alongside a normal
     URL to install an already-built binary package from another
     distro's format directly -- Arch's .pkg.tar.zst/.xz or Debian's
     .deb -- instead of building anything. URL still points at the
     actual package file (a repo mirror, a local path, wherever); Forge
     downloads it exactly like any other package, then extracts it
     itself and stages the real files into PKG_DIR, the same role it
     always has. heat()/mold()/smith() are NOT called for a PKG_FORMAT
     recipe -- there's nothing to configure or build, only to unpack --
     so don't define them (sketch generates a PKG_FORMAT recipe without
     them; assay doesn't require them either for this case). The
     package's own version (pacman's pkgver from .PKGINFO, or Debian's
     Version: from control) is used in place of the recipe's own
     VERSION whenever the two disagree, since that's the authoritative
     source. Maintainer scripts (pacman's .INSTALL post_install/
     post_upgrade, Debian's postinst) are deliberately NOT run --
     most of what they'd typically do (ldconfig, mandb, desktop/icon/
     mime caches) is already covered generically by the Hooks system
     below; 'smith' prints a note when a package had one, so you can
     check by hand if something package-specific seems to be missing.
     Everything else (caching, hooks, history, OPTIONS) works the same
     as any other recipe.

     A recipe may optionally define heat(), run before mold() to handle
     configuration (./configure, meson setup, cmake, autogen.sh) as its
     own tracked step -- see the resume paragraph above for why this is
     separate from mold() rather than lumped into it. Recipes generated
     by 'sketch' always define one, using
     echo 'No configuration for this package' as a no-op placeholder
     when the detected build system has no separate configure step
     (setup.py-based packages, mainly). heat() is optional for the sake
     of recipes written before this existed -- if a recipe has none,
     that step is just skipped.

     A recipe may optionally define OPTIONS, an array of
     "key:description:default" entries (default being 'y' or 'n'), for
     build-time choices mold()/smith() can branch on -- e.g.:
       OPTIONS=("doxygen:Build API documentation with Doxygen:n")
     mold()/smith() read the choice from $OPT_DOXYGEN (key upper-cased,
     - to _; always "y" or "n"). Each option is resolved, in order: from
     --with-<key>/--without-<key> on the command line; else the
     recipe's own default when running with -y/--yes; else an
     interactive prompt in the book's own "[y/N]"/"[Y/n]" style.

     Before building, smith checks the repo (REPO_DIR, see 'forge repo'
     below) for a package already built with the exact same recipe
     content and OPTIONS. On a match, it skips the download and mold()
     entirely and installs straight from that cached build. On a miss,
     it builds normally and -- unless --no-store was given -- caches
     the result afterward for next time. This applies only to packages
     with a real URL (tarball); self-contained meta-packages are never
     cached.

     Cleanup: as soon as smith() (make/ninja install, staged via DESTDIR
     into PKG_DIR) finishes a fresh build, the entire extracted
     source/build tree (SRC_DIR) is deleted right then -- not held onto
     until the whole run finishes. Nothing past that point needs it:
     manifest generation, caching, and the final copy to the real
     filesystem all work from PKG_DIR alone. This matters most for
     packages like LLVM or Firefox, whose build directory alone can run
     many times the size of what actually ends up installed -- with the
     old end-of-run cleanup, that whole tree sat on disk well past the
     point it was needed. (This never applied to a repo cache hit --
     there's no source/build tree to begin with when installing straight
     from a cached build.)

     Forge also strips debug symbols from ELF binaries/libraries and
     deletes libtool .la files in PKG_DIR right after, before it's
     installed onto the real system or cached in the repo. Both are
     standard LFS/BLFS practice and don't change runtime behavior --
     Linux binaries don't need .la files the way some other Unixes do,
     and stripping only removes symbols used for debugging, not
     anything the program needs to run. This is what actually shrinks
     what stays installed build over build. Skip with --no-strip /
     --keep-la, or turn either off permanently via STRIP_BINARIES=0 /
     REMOVE_LA_FILES=0 in forge.conf. The repo tarball itself is also
     compressed at xz -9e (max, slower) rather than the default level,
     since a cached build is written once but meant to be reused --
     size matters more there than build-time speed.

     Hooks: right after installing files onto the real system, 'smith'
     checks the manifest it just wrote against FORGE_HOOKS (a table of
     "pattern::command" pairs, editable in forge.conf) and runs whatever
     matches -- glib-compile-schemas for GSettings schemas,
     update-desktop-database for .desktop files, gtk-update-icon-cache
     for icons, mandb for man pages, ldconfig for shared libraries,
     update-mime-database for MIME data. Each matching command runs at
     most once per install, and a hook whose command isn't on PATH is
     skipped with a note rather than failing the install.

     History: every successful install/upgrade/removal (from 'smith' or
     'melt') appends one line to HISTORY_FILE
     (/etc/forge/log/history.log by default) -- timestamp, action, name,
     version. Purely a record for your own reference (nothing reads it
     back for rollback); it's meant to answer "when did I install this"
     without relying on memory.

     Dependency cascade: when installing a package with unmet DEPS,
     independent packages at the same point in the tree (neither
     depends on the other) build concurrently instead of one at a
     time -- capped at MAX_PARALLEL_JOBS (forge.conf; defaults to
     nproc) regardless of how wide that point in the tree is. A
     package still only starts once everything IT depends on has
     already finished. Set PARALLEL_DEPS=0 in forge.conf to force
     strictly sequential installs instead (easier to read the combined
     output of a big cascade, at the cost of speed).

   assay <pkg>
     Validates a recipe before you ever smith it: checks NAME/VERSION/URL are set, mold() (build method) or smith() (install method) functions exist, dependecies DEPS have matching recipes, and warns if no checksum SHA256SUM/MD5SUM or CRLF line endings are present.

   Checksums can also live outside the recipe, in /etc/forge/checksums/<pkg>.sha256sum
   or .md5sum (just the hash, one line). If present, this takes priority over
   SHA256SUM/MD5SUM set inside the recipe itself.

   probe <pkg|--all> [--update] [-y|--yes]
     Checks upstream for a newer version, without installing anything.
     Requires a recipe to define:
       VERSION_CHECK_URL="<page listing available versions/tarballs>"
       VERSION_REGEX='<grep -oP pattern that prints ONLY the version>'
     Use \K to drop everything matched before the version, and a (?=...)
     lookahead to drop what comes after, so the pattern prints just the
     version string. Example, for a directory listing full of links like
     "gdk-pixbuf-2.42.12.tar.xz":
       VERSION_REGEX='gdk-pixbuf-\K[0-9]+\.[0-9]+\.[0-9]+(?=\.tar\.xz)'
     Recipes without both fields are skipped (silently under --all, with
     a note otherwise). With --update, rewrites VERSION="..." in the
     recipe file (keeping a .bak), and refreshes the checksum via the
     recipe's existing CHECKSUM_URL. The URL field itself does not need
     to change, since it's already expected to reference \${VERSION}.
     Does not install anything -- follow up with 'forge temper <pkg>'.

   sketch <pkg> [--source lfs|blfs|slfs|glfs|mlfs|gnome|git|pacman|apt] [-s <source>] [--url <page, tarball, or git repo>] [--ref <tag/branch/commit>] [--category <dir>] [--force]
     Generates a DRAFT recipe by looking up <pkg> in the LFS, BLFS,
     SLFS, GLFS, or MLFS book and scraping its page: download link,
     version, checksum (if listed), any patch or additional-download
     links, and the book's own build/install shell commands, split
     into mold() (build steps), smith() (the step(s) run "as the root
     user" during Installation, i.e. install), and finish() (any
     step(s) found under a later "Configuring <pkg>" section/
     subsection -- these run against the real installed system, same
     as smith()'s own "$1", which is why they aren't rewritten with
     DESTDIR). --book is still accepted as an alias for --source.
     Matching against a book is EXACT (on the page's own URL slug, not
     its link text) -- "gtkmm" will never silently resolve to whatever
     "gtkmm3"/"gtkmm4" page happens to come first. On a total miss,
     sketch lists every slug that merely contains <pkg> as a
     "Did you mean" suggestion instead of guessing for you.
     Without --source, all six are tried in order (blfs, slfs, glfs,
     mlfs, lfs, gnome), stopping at the first match -- pass --source to
     search only one. "gnome" is different from the rest: instead of a
     book page, it looks up <pkg> directly under
     download.gnome.org/sources/<pkg>/, picks the newest series
     directory (alpha/beta included -- this can outrank the latest
     stable release, which is intentional) and the newest tarball
     inside it. It never has build commands to scrape (see "Generic
     build detection" below).
     LFS itself is a special case: its package pages (chapter 5-8/9)
     have the build commands but not the download link/checksum --
     those live on separate chapter 3 pages (Packages and Patches).
     When --source lfs matches (or is passed explicitly), sketch also
     fetches those two chapter 3 pages and uses them as a fallback
     for the download/checksum/patch info.
     Generic build detection: whenever no command blocks were found on
     the source page (or there was no page at all -- the "gnome"
     source, or a direct tarball --url), sketch downloads the tarball
     and looks at its top-level files to guess mold()/smith(), in this
     priority: meson.build > configure > autogen.sh > setup.py. These
     are generic, standard commands (plain meson/autotools/distutils,
     no project-specific flags) -- always review them before smithing.
     Requires python3 (used for the HTML parsing).

     Also best-effort: a build subdirectory (mkdir build && cd build,
     common with meson/cmake) detected in mold() gets a matching cd
     added at the start of smith(), since mold() and smith() run as
     separate pipeline stages and don't share a working directory.
     Additional download links found beside the main tarball (besides
     patches, which go into EXTRA_FILES) are added to mold() as a plain
     wget line -- commented out if the book marks that one optional.

     --url accepts EITHER a documentation page to scrape (same as any
     book match) OR a link straight to a tarball (e.g. a GitHub release
     asset, or any host with no page at all) -- recognized by its file
     extension (.tar.gz/.tar.xz/.tar.bz2/.tar.zst/.zip). A direct
     tarball skips page-scraping entirely: VERSION is parsed from the
     filename itself (expects "name-1.2.3.tar.xz"), and mold()/smith()
     come from Generic build detection above, since there's no page to
     scrape commands from. Use --url when the package isn't found
     automatically, to point at a book edition/mirror not covered by
     the defaults, or for any tarball with no book/GNOME entry at all.

     --source git --url <git repo URL> [--ref <tag/branch/commit>]:
     for a package whose released tarballs lag behind its actual git
     history (an upstream that's slow to cut releases, or effectively
     abandoned, while real fixes sit on the default branch). Shallow-
     clones the repo (no page to scrape, same as "gnome") and detects
     the build system straight from the checked-out tree -- same
     priority as everywhere else: meson.build > configure > autogen.sh
     > setup.py. Without --ref, picks the newest tag (sort -V,
     alpha/beta included); with no tags at all, falls back to the
     default branch's HEAD, and VERSION becomes that commit's short
     hash. The recipe gets GIT_URL/GIT_REF instead of URL/checksum --
     'smith' clones at that ref instead of downloading+extracting a
     tarball (see 'smith' below). GIT_REF is written out explicitly
     even when auto-detected: an unpinned branch means every future
     'smith' could silently pull something different, which defeats
     the point of a reproducible recipe.

     --source pacman|apt <pkg>: resolves <pkg> to a download URL,
     preferring the LOCAL package index your system already has --
     pacman's sync databases (/var/lib/pacman/sync/*.db, populated by
     'pacman -Sy') or apt's (/var/lib/apt/lists/*_Packages, populated
     by 'apt update') -- since that's instant and needs no network
     round-trip at all. When there's no usable local index (not on
     Arch/Debian, or the package just isn't in what's currently
     synced), automatically falls back to fetching the sync
     databases/Packages files directly from PACMAN_MIRROR/PACMAN_REPOS
     or APT_MIRROR/APT_SUITE/APT_COMPONENTS (forge.conf; sane defaults
     built in) -- this works even on a machine with no pacman/apt
     installed at all, since only wget/tar/awk are needed either way.
     On a hit, builds the real download URL (pacman: from
     /etc/pacman.d/mirrorlist when using the local index, or straight
     from PACMAN_MIRROR when fetched fresh, substituting $repo/$arch
     same as pacman itself; apt: from the Filename: field combined
     with the first "deb " line in /etc/apt/sources.list(.d), or
     APT_MIRROR directly) and falls straight into the PKG_FORMAT
     handling above -- same generated recipe you'd get from
     'sketch --url' pointed directly at that .pkg.tar.zst/.deb. Since
     "found nothing" here can still just mean the wrong repos/suite
     are configured rather than the package not existing, these stay
     opt-in only (via --source), never part of the default
     multi-source search.

     --category places the recipe under RECIPE_DIR/<dir>/ instead of the
     top level (matching subfolders like python-modules/).
     --force overwrites an existing recipe at that path.

     The book installs straight onto the real system, with no notion of
     staging -- so 'make install'/'ninja install' lines in smith() are
     rewritten to add DESTDIR="$1", matching how smith() is expected to
     install here. This, the mold()/smith()/finish() split, the build-
     directory detection, and the optional-download detection are all
     best-effort heuristics based on how the book's HTML is structured.
     The generated recipe is marked as a draft and always needs a look
     before you trust it -- run 'forge assay <pkg>' and read through
     mold()/smith()/finish() yourself before smithing.

     The book index URLs are in /etc/forge/forge.conf
     (LFS_BOOK_INDEX_URL / BLFS_BOOK_INDEX_URL / SLFS_BOOK_INDEX_URL),
     editable if a book's structure changes or you want to point at a
     different edition.

   temper <pkg|--all> [-y|--yes]
     Updates a single package to the version in its recipe, only if it differs from what's installed. With --all, checks every installed package, lists what's outdated, asks for one confirmation, then updates them all.

   melt <pkg> [-y|--yes] [--with-deps]
     Uninstalls a package: removes every file and directory it's known to have installed, then deletes its record. Warns first if another installed package depends on it.
                        --with-deps     After removing <pkg>, checks whether that left any of its own dependencies orphaned (installed only as a dependency, nothing left needs them) and offers to remove those too -- same idea as 'pacman -Rns'.

   orphans
     Lists installed packages that were pulled in only as a dependency
     and that nothing currently installed needs anymore (pacman's
     '-Qdt'). Chains cascade correctly -- if removing one orphan would
     orphan another, both show up. Doesn't remove anything by itself;
     see 'melt --with-deps' to actually clean them up as part of
     removing whatever pulled them in.

   search <term>
     Case-insensitive search over every recipe's NAME and DESCRIPTION
     in RECIPE_DIR (not just installed ones), showing install status
     for each match. For a package not in any local recipe yet, try
     'forge sketch <term>' instead, which looks it up in a book/GNOME/
     git rather than searching what you already have.

   wares
     Lists packages currently installed by forge, with their version.

   patts
     Short for "patterns", lists every recipe available in /etc/forge/recipes (whether installed or not), with version and install status (installed/not installed).

   gauge <pkg>
     Shows a package recipe's details (name, version, URL, checksum, description, OPTIONS, and how many pre-built configs exist for it in the repo) without installing anything.

   repo <init|pull|push|status|list> [pkg]
     Manages the git repo of pre-built packages that 'smith' reads from
     and writes to (see the OPTIONS section under 'smith' above).
                        init    Creates REPO_DIR as a git repo if it doesn't exist yet, and wires up REPO_REMOTE as its 'origin' if one is configured.
                        pull    Fetches newer pre-built packages from the remote (git pull --ff-only). Requires REPO_REMOTE to be set.
                        push    Pushes any local commits (builds cached with --no-store not used) to the remote. Also happens automatically after each build if REPO_AUTO_PUSH=1.
                        status  Shows the repo path, remote, auto-push setting, and any uncommitted changes.
                        list [pkg]      Lists every cached build, or just those for one package if given, with its version, config id, and the OPTIONS that produced it.
     REPO_DIR, REPO_REMOTE, and REPO_AUTO_PUSH are set in /etc/forge/forge.conf.
     Set REPO_GPG_KEY to GPG-sign every commit repo_store makes (a key
     ID or email gpg already knows about), and REPO_GPG_VERIFY=1 to
     refuse a cached build whose commit isn't verifiably signed --
     matters once REPO_REMOTE is shared with a machine/person you
     don't fully trust the write access to; irrelevant for a purely
     local, single-user repo.

# History of Forge

This idea originated from Linux From Scratch (LFS), when I succesfully followed the whole book and booted into the OS. 

![Linux From Scratch Logo](https://www.linuxfromscratch.org/images/linux-from-scratch.png)

After installing internet drivers then WPA Supplicant and DHCP, I realized that writing every command after another finished — to build and install the package — took way too much, so the idea of creating an automatic package installer popped in my mind.

> [!NOTE]
> Before you say about Automated Linux From Scratch (ALFS), that project wasn't updated not even a single bit since 2017. 😭
>
>  I don't even know how to use that since there's no updated documentation that explains how to build that, and any Makefile there just fails. 

Then I created a little recipe-based PM (like Gentoo's Portage, that relies on ebuild files to build the packages sources), that I would still have to write every command inside the recipe files, but that is WAY better than having to wait for the commands to finish (that REALLY take that long once you enter to packages like GCC, LLVM or QT 6).

Later on, I got working a new subcommands for Forge that would fetch a package directly from BLFS homepage, then get specific HTML patterns to write the recipe files AUTOMATICALLY (damn that's really good). This subcommand name is sketch.

Then I added other LFS' books support to sketch, SLFS and LFS (much later I added GLFS, MLFS

> [!TIP]
> I use Arch btw
> ![Arch Linux Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f9/Archlinux-logo-standard-version.svg/1280px-Archlinux-logo-standard-version.svg.png?utm_source=commons.wikimedia.org&utm_campaign=index&utm_content=thumbnail&_=20240926043220)



