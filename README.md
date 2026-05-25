# ant-dynamic-and-changing-revisions

Project #8 (P1) from `output/ANT-COVERAGE-PLAN.md`.
Catalog IDs: **#11** (`dynamic-revision`) and **#23** (`changing-revision`).

---

## Feature exercised

Two non-pinned-version mechanisms in a single `ivy.xml`:

1. **Dynamic revision** (`#11`): `<dependency rev="3.+"/>` on `commons-lang3`. Ivy resolves
   the `3.+` selector to the highest available 3.x release at resolve-time. The regression
   signal is: Mend must emit the concrete version (`3.14.0`), not the selector string `3.+`.

2. **Changing revision** (`#23`): `<dependency rev="latest.integration" changing="true"/>` on
   `commons-text`. `changing="true"` enables SNAPSHOT-equivalent semantics — Ivy bypasses its
   TTL cache and re-downloads the artifact on every resolve because content at the same
   coordinates may change. The regression signal is: Mend must emit the concrete version
   (`1.10.0`), not the string `latest.integration`.

---

## Mode-3 reality (SCM scanner pivot)

The bolt-4 SCM scanner **does not** invoke Ivy (Mode 1). It path-scans for binary artifacts
(Mode 3). To give the scanner something to fingerprint, 4 JARs have been pre-resolved and
committed into `lib/` at their concrete versions:

| File | Dynamic intent | Concrete version | SHA1 |
|---|---|---|---|
| `lib/commons-lang3-3.14.0.jar` | `rev="3.+"` | 3.14.0 | `1ed471194b02f2c6cb734a0cd6f6f107c673afae` |
| `lib/commons-text-1.10.0.jar` | `rev="latest.integration" changing="true"` | 1.10.0 | `3363381aef8cef2dbc1023b3e3a9433b08b64e01` |
| `lib/commons-codec-1.16.1.jar` | transitive of commons-text | 1.16.1 | `47bd4d333fba53406f6c6c51884ddbca435c8862` |
| `lib/slf4j-api-2.0.16.jar` | transitive of commons-text | 2.0.16 | `0172931663a09a1fa515567af5fbef00897d3c04` |

All SHA1 values have been verified against Maven Central's published checksums.

`.gitignore` does **not** exclude `lib/` — the JARs are intentionally tracked so that Mode-3
scanning has binary artifacts to fingerprint.

---

## Dynamic revision (#11): rev="3.+"

**Ivy behaviour (Mode 1):**
- Ivy queries Maven Central for all published `org.apache.commons:commons-lang3` versions.
- The `3.+` selector matches all `3.x` versions and selects the highest available.
- Latest stable 3.x on Maven Central as of 2026-05-25: `3.14.0`.
- Ivy downloads `commons-lang3-3.14.0.jar` and records it as the resolved version.

**Expected Mend behaviour:**
- Mend must emit `version = "3.14.0"` in the dependency tree.
- If Mend emits `version = "3.+"` it has failed to resolve the dynamic selector.
- `expected_dependency_pairs` in `autotest_config.json` pins this assertion.

---

## Changing revision (#23): rev="latest.integration" changing="true"

**Ivy behaviour (Mode 1):**
- `latest.integration` selects the artifact with status `integration` (Maven SNAPSHOT
  or the newest build). On Maven Central (release-only), this resolves to the newest
  published release.
- `changing="true"` tells Ivy to bypass its local cache TTL and re-validate the
  artifact on every resolve — equivalent to Maven's SNAPSHOT re-check behaviour.
- Latest stable `commons-text` on Maven Central as of 2026-05-25: `1.10.0`.

**Expected Mend behaviour:**
- Mend must emit `version = "1.10.0"` in the dependency tree.
- If Mend emits `version = "latest.integration"` it has failed to record the concrete
  resolved version (or has not run Ivy resolution at all).
- `expected_dependency_pairs` in `autotest_config.json` pins this assertion.

---

## Expected dependency tree

Mode-3 result: flat list of 4 JAR filenames as direct dependencies (no tree structure).

```
root: ant-dynamic-and-changing-revisions-20260525
  commons-lang3-3.14.0.jar        v3.14.0  (dynamic-revision concrete result)
  commons-text-1.10.0.jar         v1.10.0  (changing-revision stable substitute)
  commons-codec-1.16.1.jar        v1.16.1  (transitive of commons-text)
  slf4j-api-2.0.16.jar            v2.0.16  (transitive of commons-text)
```

All deps appear as direct in Mode-3 (binary path scan produces no transitive structure).

---

## Mend config

**Bucket A** — default-emit with `java` pinned to `17` (Ant itself is not pinnable via
`install-tool`; the Ant binary version comes from whatever the operator installs
out-of-band). This is a partial reproducibility limitation: the Ant version is not
controlled by `.whitesource`. Downstream comparators should treat Ant version as
uncontrolled.

`configMode: "LOCAL"` — `.whitesource` is read from the repo root.

**UA resolver mode (aspirational):** Mode 1 (Ivy resolution via `IvyDependenciesAntTask`).
`whitesource.config` ships `ant.resolveDependencies=true` and
`ant.ivyResolveDependencies=true`. In SCM scan mode, this is aspirational — the bolt-4
scanner does not honour these flags and falls through to Mode 3. For Mode-1 testing of
dynamic/changing semantics, use pipeline-mode `mend ua -c whitesource.config`.

---

## Regression signals

| Assertion | What it catches |
|---|---|
| `expected_dependency_pairs: commons-lang3-3.14.0.jar @ 3.14.0` | Mend emitting `3.+` instead of concrete version — catalog #11 |
| `expected_dependency_pairs: commons-text-1.10.0.jar @ 1.10.0` | Mend emitting `latest.integration` instead of concrete version — catalog #23 |
| `expected_dependency_checksums: sha1=1ed471...` (commons-lang3) | SHA1 mismatch indicating wrong JAR was fingerprinted |
| `expected_dependency_checksums: sha1=3363381...` (commons-text) | SHA1 mismatch indicating wrong JAR was fingerprinted |

---

## File structure

```
ant-dynamic-and-changing-revisions-20260525-170058/
  build.xml              # Ant targets: ivy-resolve (Mode-1) + resolve (Mode-3 list)
  ivy.xml                # ivy-module with rev="3.+" and rev="latest.integration" changing="true"
  ivysettings.xml        # Standard Maven Central ibiblio resolver with checkmodified=true
  .whitesource           # Bucket-A: java pinned to 17, configMode=LOCAL
  whitesource.config     # UA flags: ant.ivyResolveDependencies=true (aspirational)
  .gitignore             # lib/ NOT excluded (JARs are tracked for Mode-3)
  expected-tree.json     # Mode-3 expected dep tree with SHA1 pins
  autotest_config.json   # Validation config with expected_dependency_pairs regression signals
  README.md              # This file
  lib/
    commons-lang3-3.14.0.jar    (dynamic-revision concrete result)
    commons-text-1.10.0.jar     (changing-revision stable substitute)
    commons-codec-1.16.1.jar    (transitive)
    slf4j-api-2.0.16.jar        (transitive)
```

---

## Real packages used

All coordinates verified against Maven Central (search.maven.org):

| Package | Group | Version | Maven Central status |
|---|---|---|---|
| commons-lang3 | org.apache.commons | 3.14.0 | Latest stable 3.x; no transitive deps |
| commons-text | org.apache.commons | 1.10.0 | Latest stable; runtime deps: commons-codec, slf4j-api |
| commons-codec | commons-codec | 1.16.1 | Latest stable; no transitive deps |
| slf4j-api | org.slf4j | 2.0.16 | Latest stable SLF4J API; no transitive deps |

---

## UA resolver notes (from java-jvm.md § 3)

- Mend's `IvyDependenciesAntTask` parses the Ivy `ResolveReport` for the dependency graph.
- Dynamic revisions (`1.+`, `3.+`, `latest.release`, `latest.integration`) are resolved to
  concrete versions by Ivy before the `ResolveReport` is produced. The UA reads the concrete
  version from the report, not the selector.
- `changing="true"` affects Ivy's cache TTL but does not affect the version recorded in
  `ResolveReport` — the concrete version is still written there.
- In Mode-3, neither mechanism is exercised at scan time. The version Mend emits comes from
  parsing the JAR filename, not from any Ivy resolution.
- **Known limitation:** UA uses an in-process Ivy library. If the UA's bundled Ivy version
  differs from a project's intended Ivy version, `ResolveReport` structure may vary.
- **No EUA support** for Ant — impact analysis is disabled per resolver docs.

---

## Probe metadata

```
probe_id:          ant-dynamic-and-changing-revisions-20260525-170058
pm:                ant
pm_version_tested: ivy-2.5.x (Ivy used by UA's in-process library)
patterns:          [dynamic-revision, changing-revision]
catalog_ids:       [#11, #23]
priority:          P1
generated:         2026-05-25
bucket:            A (java pinned to 17)
target:            remote
```
