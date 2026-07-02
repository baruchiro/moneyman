---
name: fork-maintenance
description: How this fork of daniel-hauser/moneyman is maintained — keep `main` equal to upstream, cut feature branches toward `release` (and toward upstream), and promote `main` → `release`. Use when syncing from upstream, bumping/patching dependencies, contributing a change back to upstream, resolving a `main`→`release` merge, or reasoning about why a change lives on `release` vs `main`.
---

# Fork maintenance flow

This repository is a **fork** of [`daniel-hauser/moneyman`](https://github.com/daniel-hauser/moneyman) (the "upstream"). It carries fork-only customizations (Actual Budget support, dependency pins, CI tweaks) on top of upstream, and ships them as a Docker image. This skill describes how the branches relate and the three flows that keep them in sync.

## Branch model

| Branch | Role | Rule |
| --- | --- | --- |
| `main` | Pristine mirror of `upstream/main`. | **Must stay byte-equal to upstream.** Never commit fork-specific changes here — a fork-only file on `main` breaks equality and leaks into any branch cut from `main` for an upstream PR. |
| `release` | **Default branch** + production. `= main` + fork customizations. | Every push to `release` publishes a Docker image + GitHub release via `.github/workflows/build.yml`. Nothing lands here except through a promotion/PR. |
| feature branches | Short-lived, one change each. | Base them on `main` or `release` depending on where the change belongs (see Flow 2). |

`release` is the **default branch** on purpose: scheduled/`workflow_dispatch` workflows only fire from the default branch, and keeping `release` (not `main`) as default lets the automation run while `main` stays a clean mirror.

## Flow 1 — upstream → main (keep `main` current)

Goal: `main` always equals the latest `upstream/main`, with no fork-only content.

- **Automated:** the `Weekly upstream sync` workflow (`.github/workflows/weekly-upstream-sync.yml`), job `update-main`, pulls `upstream/main` into `main`. Runs weekly (Mon 05:00 UTC) and on manual dispatch.
- **Manual:** GitHub's "Sync fork → Update branch" on `main`, which fast-forwards it to upstream.

Never merge fork customizations into `main`. If `main` ever diverges from upstream by anything other than upstream's own history, that's a bug.

## Flow 2 — feature branch → release and/or upstream

Decide the target **before** choosing the base branch:

- **Contribute back to upstream** (e.g. a dependency bump upstream should also take): branch off **`main`**, make **one clean commit**, and open a *cross-fork* PR (base `daniel-hauser/moneyman:main`, head `baruchiro/moneyman:<branch>`). Because `main` is a pristine mirror, the PR is exactly your single commit. Do **not** branch off `release` for this — `release` carries all the fork divergence, so the PR would show dozens of unrelated commits.
- **Fork-only change** (Actual Budget code, CI, dependency pins that upstream doesn't want): branch off **`release`** and open a PR into `release`.

A change can flow both ways: contribute it to upstream from a `main`-based branch, and it will reach `release` naturally through Flow 3 once upstream (and then `main`) has it.

## Flow 3 — main → release (promote upstream into production)

Goal: bring synced-upstream changes into `release` **through a reviewed PR** so nothing ships unreviewed.

- **Automated:** the `Weekly upstream sync` workflow, job `propose-release`, opens a `main → release` PR whenever `main` is ahead of `release`.
- **Conflict resolution:** resolve on the **`release` side** (in the PR / merge commit). Keep the fork's functional customizations; take upstream's newer dependency versions; regenerate `package-lock.json` from the merged `package.json` with `npm install --package-lock-only`. **`main` stays untouched.**
- Merging the PR is what triggers the Docker publish + GitHub release (`build.yml`). That merge is the single production gate.

## Conventions & gotchas

- **Default branch must be `release`** (Settings → Branches) or the sync workflow can't schedule/dispatch.
- **Enable** Settings → Actions → General → "Allow GitHub Actions to create and approve pull requests", or `propose-release` can't open its PR.
- Fork-only dependency pins (e.g. `@actual-app/api`) live on `release`, not `main`.
- Typical merge conflicts are in `package.json` / `package-lock.json` (dep versions) and the occasional Actual Budget source file — resolve toward newer deps + preserved fork behavior.
- Anything cut for an **upstream** PR must come from `main`, never `release`.
