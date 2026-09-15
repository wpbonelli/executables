# Migrating off pymake with modflow-devtools

This document lays out a phased plan for replacing this repository's `releases.json` / `scripts/fetch_releases.py` hybrid (see [DEVELOPER.md](DEVELOPER.md#program-sources)) with the [modflow-devtools Programs API](https://github.com/MODFLOW-ORG/modflow-devtools/issues/263), and for shrinking pymake's role in this repo's release workflow over time.

**Status**: planning only. Nothing in this document has been implemented here yet. It depends on work currently on an unmerged branch in `modflow-devtools` (`programs-api-trim`) - not yet released, not yet on PyPI. Revisit once that lands.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Why](#why)
- [Ground truth as of this writing](#ground-truth-as-of-this-writing)
- [Phase 0: swap the mechanism, not the scope](#phase-0-swap-the-mechanism-not-the-scope)
- [Phase 1: add the confirmed-safe, currently-unlisted programs](#phase-1-add-the-confirmed-safe-currently-unlisted-programs)
- [Phase 2: re-attempt the reverted trio, cautiously](#phase-2-re-attempt-the-reverted-trio-cautiously)
- [Phase 3: programs that genuinely aren't ready](#phase-3-programs-that-genuinely-arent-ready)
- [Phase 4: shrink pymake's role, then reconsider this repo's purpose](#phase-4-shrink-pymakes-role-then-reconsider-this-repos-purpose)
- [Integration notes](#integration-notes)
- [References](#references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Why

`releases.json` + `scripts/fetch_releases.py` + `scripts/patch_code_json.py` are a hand-rolled ~250-line reimplementation of asset selection, download, and extraction that `modflow_devtools.programs` now does directly, generalized across *any* `MODFLOW-ORG` (or other) repo that publishes per-platform release assets - not just the three repos it ships out-of-the-box support for. `DEVELOPER.md` already anticipated this ("Future `modflow-devtools` migration"); this document makes the plan concrete now that the API exists and has been verified to work this way.

## Ground truth as of this writing

`DEVELOPER.md`'s "Program sources" tables are **stale** relative to `releases.json` and the actual state of each program's repo (verified live via the GitHub API, not assumed from the docs):

| Program(s) | `DEVELOPER.md` claims | Actual state |
|---|---|---|
| `mf6`, `zbud6`, `mf5to6`, `libmf6` | fetched, tag `6.7.0` | **Correct**, but the pinned tag is stale - current upstream release is `6.8.0` |
| `triangle`, `gridgen` | fetched | **Correct**, in `releases.json` today |
| `zonbud`, `zonbudusg`, `mfusg_gsi` (`mfusgt`) | fetched | Real per-platform releases exist, but were deliberately reverted out of `releases.json` in [#71](https://github.com/MODFLOW-ORG/executables/pull/71): *"keep building... with pymake until new releases are made, the current ones are dynamically linked to gfortran libs."* `mfusgt` has since moved to `v2.6.2` (past the flagged version) - worth re-verifying, not assuming fixed. |
| `mfnwt` | fetched, tag `v1.3.0` | **No releases at all** on `MODFLOW-ORG/mfnwt`. Still pymake-built. The doc is simply wrong here. |
| `swtv4`, `mflgr`, `mf2000` | fetched | **No releases.** Still pymake-built. |
| `mp6` (`modpath6`) | fetched | **No releases**, and not listed in the "built by pymake" table either - undocumented either way. |
| `vs2dt` | fetched | Real cross-platform releases exist, never actually added to `releases.json`. |
| `mt3d-usgs` | built by pymake | Real cross-platform releases exist (unlisted as a fetch candidate anywhere). |
| `mf2005`, `mt3dms`, `mfusg` (plain, not `mfusgt`) | built by pymake | Correct - each publishes a single non-platform-tagged archive, not usable by ostag-based asset selection as-is. |
| `mp7`, `crt`, `sutra` | built by pymake / USGS source | Correct. `crt`'s repo doesn't currently exist under that name at all. |

Fixing `DEVELOPER.md`'s tables to match reality is worth doing regardless of this migration, and should probably happen alongside Phase 0 below rather than separately.

## Phase 0: swap the mechanism, not the scope

Replace `scripts/fetch_releases.py` with calls to `modflow_devtools.programs.install_program`, covering exactly the same three programs `releases.json` covers today (`modflow6`, `triangle`, `gridgen`). Same behavior, different implementation - if anything breaks, it's obviously about the swap, not scope creep.

```python
# sketch - not the actual replacement script
import warnings
warnings.filterwarnings("ignore", message=".*modflow_devtools.programs.*experimental.*")
from pathlib import Path
from modflow_devtools.programs import install_program

MANIFEST = [
    {"repo": "modflow6", "tag": "6.8.0"},   # bump from the currently-stale 6.7.0
    {"repo": "triangle", "tag": "v1.6"},
    {"repo": "gridgen", "tag": "v1.0.02"},
]

def fetch(ostag, outdir):
    fetched = []
    for entry in MANIFEST:
        installations = install_program(
            repo=entry["repo"], version=entry["tag"], bindir=Path(outdir), platform=ostag,
        )
        fetched += [exe for inst in installations for exe in inst.executables]
    return fetched
```

Notes:

- The manifest itself gets simpler: `install_program` autodetects the platform asset name and archive shape (bundle vs. single-program), so the `"assets": {"linux": "...", "macarm": "...", ...}` block in today's `releases.json` becomes unnecessary - just `repo` and `tag`.
- `--list` mode (feeding pymake's `--exclude`) still needs a list of program names - that doesn't disappear, it just derives from `MANIFEST` (or a small parallel list) instead of `releases.json`.
- `patch_code_json.py` gets simpler too: it currently re-parses `releases.json` and does a separate `HEAD` request per asset just to get a `Last-Modified` date. Each `install_program()` call already returns `ProgramInstallation.version`/`.source`, so patch data can come from the return value directly. Note `code.json`'s date field means "when upstream source was last updated," while `installed_at` means "when this build ran" - a small semantic difference worth deciding explicitly rather than blurring.
- Consider `version="latest"` per entry instead of a pinned tag, to stop the tag from silently going stale (as `modflow6`'s `6.7.0` already has) - trades reproducibility for freshness. A call for whoever owns this manifest, not a mechanical part of the swap.

## Phase 1: add the confirmed-safe, currently-unlisted programs

`vs2dt` and `mt3d-usgs` both have real, cross-platform releases with no known portability issue (unlike the gfortran-linked trio in Phase 2). Same mechanism as Phase 0, just two more manifest entries.

**Before** removing either from pymake's build list, run the same check this workflow already does for macOS (`otool -L` on the fetched binary; `ldd` on Linux) on a clean runner with no dev toolchain preinstalled, so a missing shared lib actually surfaces. This is exactly the check that caught the Phase 2 problem below - there's no reason to assume this batch is exempt.

## Phase 2: re-attempt the reverted trio, cautiously

`zonbud`, `zonbudusg`, `mfusg_gsi` (`mfusgt`) were pulled from `releases.json` in #71 because their published binaries were dynamically linked against gfortran libs not guaranteed present on a user's machine. `mfusgt` has since released past the flagged version (`v2.6.2` vs. the `v2.6.0` that had the problem). Before re-adding any of the three:

1. Download the current release asset for each platform.
2. Run the same linkage check as Phase 1, on a clean runner.
3. Only re-add to the manifest (and remove from pymake's build list) once confirmed statically linked (or otherwise portable) on all three platforms.

## Phase 3: programs that genuinely aren't ready

`mfnwt`, `swtv4`, `mflgr`, `mf2000`, `modpath6` have no independent releases at all yet. `mf2005`, `mt3dms`, `mfusg` (plain) publish a single non-platform-tagged archive, not the per-ostag shape any installer (this one included) can select a platform asset from. These need their own repos to adopt per-platform release CI first - that's work in those repos, not here or in modflow-devtools. `crt` and `sutra` are furthest out: `crt`'s repo doesn't exist under that name, and `sutra` is USGS-source-only per current docs.

Tracking this phase mostly means periodically re-running the ground-truth check above against each repo, not writing new code.

## Phase 4: shrink pymake's role, then reconsider this repo's purpose

Once Phases 0-2 land, pymake's role in `release.yml` shrinks to whatever's left in Phase 3's list (`mf2005`, `mfusg`, `mt3dms`, `mp7`, `crt`, `sutra`, plus any Phase 3 program that hasn't migrated yet). At that point it's worth revisiting whether a combined `executables` distribution needs to keep existing at all, versus users/tools installing each program directly from its own repo via `install_program(repo=<name>)` - or whether the "one command installs everything" convenience is worth keeping via a composition step (fetch each program's own latest release and re-bundle, rather than building anything from source). Not a decision to make now; flagging it as the natural next question once the migrated set is large enough to make it concrete.

## Integration notes

- `release.yml`'s build job already runs `pixi run --manifest-path modflow6/pixi.toml python -c "from modflow_devtools.ostags import get_ostag; ..."` to compute `ostag` - `modflow-devtools` is already an importable dependency in that exact CI step via `modflow6`'s own `pixi.toml`. Swapping in `install_program` calls shouldn't need new dependency wiring, just likely bumping the pinned `modflow-devtools` version once the relevant work is released.
- `install_program` emits a `FutureWarning` (the Programs API is still experimental) - suppress it in CI the same way the snippet above does, or output will get noisy.
- Today's fetch step writes straight into a zip (`fetch_releases.py --zip $ostag.zip`); `install_program`'s `bindir` extracts to a plain directory. Replicate the zip step separately after extraction (same as pymake's own `--zip` flag already does in a later workflow step), rather than trying to make `install_program` write into a zip directly.

## References

- modflow-devtools Programs API docs (`docs/md/programs.md`, user-facing) and design doc (`docs/md/dev/programs.md`) - not yet live on readthedocs; currently only on the unmerged `programs-api-trim` branch in `modflow-devtools`. The versions currently published at [modflow-devtools.readthedocs.io](https://modflow-devtools.readthedocs.io/en/latest/md/programs.html) still describe the older registry-based design this plan does not depend on.
- [MODFLOW-ORG/modflow-devtools#263](https://github.com/MODFLOW-ORG/modflow-devtools/issues/263) - Programs API tracking issue
- [#71](https://github.com/MODFLOW-ORG/executables/pull/71) - the gfortran dynamic-linking revert this plan's Phase 2 responds to
- [DEVELOPER.md](DEVELOPER.md) - this repo's current build/release process
