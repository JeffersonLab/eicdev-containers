---
name: sync-eic-spack
description: Sync C++ package versions in eicdev container Dockerfiles (ubuntu-root, eic-base, eic-full) with the official EIC spack environment at eic/containers. Trigger on "sync with eic spack", "sync with eic containers", "sync with eic", or "/sync-spack". Reports a diff, proposes edits for non-ROOT packages, and flags ROOT mismatches separately for explicit user approval.
---

# Sync eicdev containers with EIC spack environment

## Scope

Only these three Dockerfiles are in scope:

- `ubuntu-root/Dockerfile`
- `eic-base/Dockerfile`
- `eic-full/Dockerfile`

Only **C++ packages built from source** (those with `ARG VERSION_*=` or `ARG CERN_ROOT_VERSION=` followed by a `git clone --branch ${VERSION_*}` or a tarball download) are compared. Ignore everything installed via `apt-get` or `pip3`.

## Source of truth

Fetch live each run (do not cache):

```
https://raw.githubusercontent.com/eic/containers/master/spack-environment/packages.yaml
```

Use WebFetch. If the URL 404s, fall back to the GitHub blob URL the user mentioned and try again — do not silently skip.

## Procedure

### 1. Inventory Dockerfile packages

Grep the three Dockerfiles for `ARG VERSION_` and `ARG CERN_ROOT_VERSION`. Each entry yields:

- variable name (e.g. `VERSION_GEANT4`)
- current value (e.g. `v11.3.2`)
- file + line
- a hint comment usually present right above (e.g. `# spack: geant4 @11.3.2.east cxxstd=20 ...`) — useful for cross-check but **not authoritative**; the spack file is.

### 2. Parse spack `packages.yaml`

Look at each top-level package entry under `packages:` and extract its required version from `require:` (e.g. `require: "@11.3.2.east cxxstd=20"`) or `version:` lists. Only pull the version number — variants (`+foo ~bar cxxstd=20`) are not compared.

### 3. Match Dockerfile package → spack package

Match by name (case-insensitive, strip prefixes like `py-`). Common pairs:
`fmt`, `clhep`, `eigen` ↔ `VERSION_EIGEN3`, `catch2`, `fastjet`, `hepmc3`, `geant4`, `podio`, `edm4hep`, `edm4eic`, `dd4hep`, `actsvg` ↔ `VERSION_ACTSSVG`, `acts`, `jana2`, `irt`, `algorithms`, `npsim`, `spdlog`, `nlohmann-json`, `root` ↔ `CERN_ROOT_VERSION`.

If a Dockerfile package is **not** in spack, mark it "not in spack — skip" in the report. If a spack package is not in the Dockerfiles, ignore it.

### 4. Compare versions

Versions in the Dockerfile are usually git tags/branches; spack uses semver. Apply this rule:

- **Numbers look the same** → match (OK). Examples that all match `@11.3.2`:
  `v11.3.2`, `v11-03-02`, `11.3.2`, `CLHEP_2_4_7_1` ↔ `@2.4.7.1`.
- **Spack has an EIC suffix** like `@11.3.2.east`, `@1.6` vs `v01-06`, `@0.99.4` vs `v00-99-04` — the leading numeric part must match; treat the suffix as informational and OK.
- **Numbers differ** (e.g. spack `@45.0.0` vs Dockerfile `v44.4.0`) → mismatch. Before reporting, **verify with git** that the spack version actually exists as a tag/branch on the upstream repo:

  ```bash
  git ls-remote --tags --heads <upstream-url> | grep -i <version>
  ```

  Use the upstream URL from the `git clone` line in the Dockerfile. If the spack version doesn't exist upstream, flag the entry as "spack version not resolvable upstream — manual review" rather than proposing an edit.

- **Dockerfile uses a branch like `main` or `pr/...`** → never auto-edit. Report as "tracking branch — leave as-is unless user asks".

### 5. Report

Print one table grouped by file. Three buckets:

1. **OK** — version numbers match (one-line summary count is enough; only list if user asks).
2. **Mismatch (auto-proposable)** — spack has a clear newer version that exists upstream as a tag. Show: package, current, proposed, upstream-tag-form, file:line.
3. **ROOT** — always its own row, never auto-proposed. Even if numbers match, surface the spack value vs Dockerfile value so the user sees it.
4. **Manual review** — branches, missing-upstream-tag, name-mapping ambiguity.

### 6. Propose edits

For bucket 2 only, propose `Edit` calls (do not apply yet) one per package, then ask the user to approve all-at-once or pick a subset. Do not touch `CERN_ROOT_VERSION` unless the user explicitly says "update root" / "yes update root" / similar. After ROOT approval, apply the same Edit pattern.

### 7. Stop

After edits are applied, stop. Do not run `build_images.py`, do not rebuild, do not commit. The user drives those steps.

## Notes

- The `# spack: ...` comments above each ARG in the Dockerfiles are stale hints written by hand. Always trust the live `packages.yaml` over the comment.
- Tag-form translation when proposing a new value: keep the Dockerfile's existing tag style. If it currently uses `v11.3.2`, propose `v<new>`; if `v01-06`, propose `v<MM>-<mm>`; if `CLHEP_2_4_7_1`, propose the underscore form. Verify the chosen form exists with `git ls-remote` before writing it into the Edit.
- Do not invent version numbers. If parsing `packages.yaml` is ambiguous for some entry, list it under "Manual review" with the raw YAML snippet.
