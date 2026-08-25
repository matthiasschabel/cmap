# MCSLAB cmap integration workflow

**Status:** Active
**Last updated:** 2026-08-25
**Scope:** the `matthiasschabel/cmap` fork and its use by `python-mcslab`

## Context

MCSLAB needs to exercise several prospective cmap improvements together while continuing to
support the released package. Upstream must receive small, independent pull requests; a combined
development build must therefore be assembled without making any feature branch depend on the
others.

This follows the same branch topology as the MCSLAB napari fork, but the downstream capability
boundary is smaller because MCS uses cmap's public API rather than private implementation details.

## Current Decision

### Branch roles

| Branch | Role | Rule |
|---|---|---|
| `upstream/main` | pyapp-kit source of truth | Fetch only; never commit here. |
| `origin/main` | clean mirror in the personal fork | Keep aligned with `upstream/main`. |
| `feature/<topic>` or `fix/<topic>` | one upstream pull request | Cut from `upstream/main`; contain one behavior and its narrow regression test. |
| `integration` | combined MCSLAB development build | Merge feature branches with `--no-ff`; never submit or use as a feature-branch base. |

The main checkout at `~/GitHub/cmap` stays on `integration` so an editable install exposes every
active feature. Work on individual upstream changes in sibling worktrees:

```text
~/GitHub/cmap                         integration checkout
~/GitHub/cmap-feat/high-color         one upstream bug fix
~/GitHub/cmap-feat/signed-infinity    one upstream feature
```

```bash
git -C ~/GitHub/cmap fetch upstream
git -C ~/GitHub/cmap worktree add \
    ~/GitHub/cmap-feat/high-color -b fix/napari-high-color upstream/main
```

Merge completed feature branches into the development build without rebasing the integration
branch into them:

```bash
git -C ~/GitHub/cmap switch integration
git -C ~/GitHub/cmap merge --no-ff fix/napari-high-color
```

`git log --merges integration` is the integration manifest. Removing an unmerged feature means
reverting its merge commit, not rebuilding or rewriting published history.

### Downstream compatibility

MCSLAB supports two configurations throughout upstream review:

1. **Stock:** the released `cmap>=0.7` dependency. Existing MCS behavior and fallbacks remain
   available.
2. **Patched:** an editable `~/GitHub/cmap` on `integration`, or a retained integration commit for
   reproducible testing. MCS detects the new public capability and delegates to it.

Capability detection follows the API, not version strings. Planned probes include constructor or
property support for signed infinity colors and the presence of an exact-variant selector. The
compatibility layer must fail closed: an absent capability selects the existing MCS implementation,
never a partial approximation.

Daily local development installs the integration checkout after the normal environment sync:

```bash
uv pip install --editable ~/GitHub/cmap
```

A normal `uv sync` restores the locked released cmap. Once the first fork capability exists,
python-mcslab should add mutually exclusive stock and fork verification commands and a retained
integration commit, rather than relying on whichever editable checkout happens to be installed.

### Upstream contribution shape

Every upstream PR should contain:

- one observable correction or capability;
- the smallest public regression test that fails before and passes after;
- only documentation required to state the changed public contract;
- a concise description of symptom, cause, and fix.

Do not include MCSLAB architecture notes, the complete migration sequence, drive-by cleanup, broad
test matrices, or speculative public abstractions. Those belong in this directory.

No agent opens an issue or pull request, pushes an upstream branch, or posts a comment without
specific authorization for that submission.

### Promotion and retirement

1. Cut a clean feature branch from `upstream/main`.
2. Implement and test the single change in its worktree.
3. Merge the branch into `integration` and verify both stock and patched MCS configurations.
4. Prepare the minimal upstream contribution from the feature branch only.
5. Keep the MCS capability fallback while the PR is reviewed and until the change ships in the
   minimum supported cmap release.
6. Raise the cmap floor and delete the fallback in one downstream change.
7. Stop merging the feature branch after `integration` contains an upstream release that includes
   it.

## Alternatives Considered

- **Base every feature on `integration`.** Rejected because every PR would contain unrelated
  changes and become coupled to every earlier review.
- **Require the patched cmap build in MCSLAB.** Rejected because it would make MCS releases depend
  on upstream review timing.
- **Gate on cmap version numbers.** Rejected because a fork commit and a release can expose the same
  capability under incomparable versions.
- **Keep the development notes only in python-mcslab.** Rejected because the notes describe cmap
  branches, defects, and state transitions and should evolve with that repository.

## Deferred Work

- Add the reproducible fork dependency axis after the first feature is merged into `integration`;
  there is no useful distinct fork build to pin yet.
- Decide whether a dedicated private `_cmap_caps.py` is warranted after two capabilities exist.
  Until then, keep probes at the existing catalog and interop boundaries rather than creating an
  abstraction for one call site.

### What the first six PRs taught

Practices that were not obvious from the plan and are worth repeating:

- **Write the test first and watch it fail, before touching the source.** A run of the existing
  suite on an unmodified tree proves nothing, because nothing in it asserts the defect. Twice
  the first red run failed for the wrong reason (a pydantic deprecation surfacing through
  `filterwarnings = ["error"]`, and a warning cache already primed by an earlier test), which
  only showed up because the red run was real.
- **Every branch cuts from `upstream/main`, so test files collide.** Two of six integration
  merges conflicted where separate branches added a test at the same point in
  `tests/test_colormap.py`. Production hunks have never conflicted. Resolve by keeping both;
  the upstream PRs are unaffected.
- **Check the tracker for the specific defect, not the subsystem.** Searching "napari" once at
  the start does not cover a stops-aliasing bug found three PRs later.
- **Run the project's pinned tools.** `uv run --group dev ruff`, not `uvx ruff`, which resolves
  an unrelated version. Take a mypy baseline before editing, since `src/cmap/_colormap.py`
  already reports four missing-stub errors and a fifth is easy to miss.
- **A stacked PR is sometimes the honest option.** #149 documents behavior that only becomes
  true once #148 lands, so it is branched from #148 and says so. Do not describe future
  behavior as current in rendered API docs.
- **Outward-facing prose follows `agents-conventions/core/AGENTS_upstream.md`**, including the
  punctuation rule: em dashes are the loudest tell of model-drafted text, so prefer periods,
  commas, and semicolons.
- **A merge upstream stales everything else immediately.** When five PRs landed at once,
  `integration` fell three upstream commits behind and the one remaining PR started reporting a
  conflict, both from the same cause: every branch was cut from the same older base. Merge
  `upstream/main` into `integration` and rebase the open branch in the same pass. See
  `AGENTS_upstream.md` "After it merges" and `AGENTS_version_control.md` "Deleting a branch
  after it merges" for the verification procedure; squash merges mean ancestry checks report a
  landed branch as unmerged.

## Next Steps

1. Keep #151 and #155 synchronized while both remain open. #155 is stacked on #151 and must be
   rebased whenever #151 changes.
2. Keep merging revised tips into `integration` with `--no-ff`; the merge list is the manifest.
3. Keep the local napari forwarding branch unpushed until napari accepts the infinity fields.
4. Retain and pin the first integration commit consumed outside the local checkout.
5. Remove a feature worktree once its PR is merged or closed; each carries its own `.venv`.
