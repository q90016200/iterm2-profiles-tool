# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Python3 CLI (`iterm2-profiles`, stdlib only, no dependencies) that backs up, exports, and imports iTerm2 profiles. It operates on the **Preferences plist** (`com.googlecode.iterm2` domain, `New Bookmarks` array) via `defaults export`/`defaults import` — not iTerm2's Dynamic Profiles JSON files. Profiles that reference an auto-login script via `Initial Text` (e.g. `~/auto_login_ssh/gf1prod/xxx.sh`) get that script bundled alongside the profile data, since those scripts contain plaintext SSH passwords and the profile is useless without them.

## Commands

```bash
./iterm2-profiles --selftest              # run the self-check; must pass before considering any change done
./iterm2-profiles backup                  # backs up ALL profiles + scripts to backup/<timestamp>.zip
./iterm2-profiles list                    # list profiles, flags missing/untracked scripts and duplicate names
./iterm2-profiles export <name...> -o dir # backs up SELECTED profiles + scripts to dir.zip
./iterm2-profiles import [bundle]         # merge/overwrite from a bundle (dir or .zip); omit bundle to pick from backup/
```

There is no separate test suite or framework — `--selftest` (bottom of the file) is the test suite. It runs entirely against in-memory fakes and a temp directory; it never calls the real `defaults` command or touches the user's actual 82+ live profiles. Any change to the core logic must add/update a case there.

## Architecture

The file is split into two layers, and that split is load-bearing for testability:

- **Pure logic** (`find_bookmark`, `merge_bookmark`, `script_source_path`, `bundle_script_key`, `rewrite_initial_text`, `diff_profiles`, `diff_script`, `duplicate_names`) — no I/O, fully unit-testable.
- **I/O boundary** (`read_bookmarks`, `write_bookmarks`, `iterm_is_running`/`warn_if_iterm_running`) — the only functions that shell out to `defaults`/`pgrep`. `--selftest` monkeypatches the module-level `read_bookmarks`/`write_bookmarks` names to in-memory fakes rather than mocking at a framework level — preserve this pattern rather than introducing a mocking library.

**Bundle format** (`profiles.json` + `scripts/<relpath>`) is shared identically between `backup` and `export` (`write_bundle`). Both then go through `write_bundle_zip`, which zips the bundle dir and deletes the plaintext dir — the scripts contain plaintext SSH passwords, so nothing keeps a second unzipped copy on disk after the run. `backup` is just "export of every profile" to a timestamped `<BACKUP_ROOT>/<timestamp>.zip`. `import` accepts either a plain bundle dir or a `.zip` — a zip is extracted to a temp dir (`tempfile.mkdtemp`) that's removed in a `finally` regardless of outcome. Since bundles are sometimes zipped by hand (e.g. Finder/`zip -r`, which wraps everything in a top-level folder and adds `__MACOSX/` junk), the zip path is resolved via `rglob("profiles.json")` rather than assuming it sits at the archive root — this means `import` only ever needs to understand one format, and moving profiles to another machine is just: run `backup`, copy the zip over, run `import` on it there.

**Picking a bundle without typing a path**: `list_backup_bundles(BACKUP_ROOT)` scans `BACKUP_ROOT` for direct children that are either a dir containing `profiles.json` or a `.zip` file, sorted newest-name-first. `choose_backup_bundle` prints that as a numbered list and reads a choice, `sys.exit`ing if `BACKUP_ROOT` has nothing usable. Both the CLI `import` (when the `bundle` positional is omitted — it's optional via `nargs="?"`) and the interactive menu's "4) 匯入" go through this same helper, so behavior is identical between the two entry points.

**Script tracking has two paths**, both computed by `bundle_script_key(profile, script_dir)`:
- *Tracked*: the script resolves under `script_dir` (default `~/auto_login_ssh`) → stored in the bundle at a clean relative path, and on import its `Initial Text` is rewritten to match a different `--script-dir` if one was passed (since `~` already expands per-user, no rewrite is needed unless the base directory itself changes).
- *Untracked*: the script exists somewhere else entirely → still backed up (flattened under `scripts/_other/`), but restored on import to its original absolute path verbatim, with no `--script-dir` rewriting (there's no "base directory" concept to remap).

**Import is additive/overwrite-only, never destructive**: it iterates only the profiles present in the incoming bundle. A profile matched by `Name` gets diffed (profile fields via `diff_profiles`, script bytes via `diff_script` — either can trigger the confirm prompt even if the other shows no change) and, on confirmation, overwritten while preserving the old `Guid`. An unmatched name is appended with a freshly generated uppercase `Guid`. Anything present locally but absent from the bundle is left untouched — there is no delete path by design.

Before writing back, `import` checks `iterm_is_running()` and warns that `defaults import` replaces the entire plist domain — if iTerm2 is running and the user changed something else in the meantime, that domain-wide write would clobber it. It also skips the `defaults import` call entirely if nothing actually changed, to avoid a no-op write to live prefs.
