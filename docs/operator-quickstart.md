# Operator quickstart — insatsu-actor

Four steps. Each one names the command, what it prints, and **how to make it fail on
purpose** — a check you have never seen refuse is a check you cannot rely on.

This repository plans records; it does not print or post anything. Nothing below opens a
socket or touches a PDS, so all four steps are safe to run on any checkout.

## Prerequisites

| Step | Needs |
|---|---|
| 1, 2 | [`nbb`](https://github.com/babashka/nbb) (`npx nbb` works) |
| 3, 4 | the [Clojure CLI](https://clojure.org/guides/install_clojure) (`clojure`), which starts a JVM |

Steps 1 and 2 cover the whole repository without a JVM. Reach for 3 and 4 when you want the
runner `deps.edn` declares, or the linter.

```bash
git clone git@github.com:cloud-itonami/insatsu-actor.git
cd insatsu-actor
```

## 1. Run the contract tests

```bash
cat > /tmp/insatsu-run-tests.cljs <<'EOF'
(ns insatsu-run-tests
  (:require [clojure.test :as t]
            [insatsu.murakumo-test]))
(t/run-tests 'insatsu.murakumo-test)
EOF
kbb --backend sci --classpath "src:test" /tmp/insatsu-run-tests.cljs
```

```
Testing insatsu.murakumo-test

Ran 9 tests containing 70 assertions.
0 failures, 0 errors.
```

**Make it fail.** Empty the body of `missing-gates` in `src/insatsu/murakumo.cljk` so it
always returns `[]` — i.e. claim every gate is satisfied:

```clojure
(defn missing-gates [_spec _attestations] [])
```

The run becomes `14 failures, 0 errors` — 2 in `missing-gates-computes-the-diff` and
3 per declared cell in `cell-plan-blocks-when-gates-missing`, which is 12 for the four cells
this manifest declares. (The 14 is derived from the cell count, not a fixed number: it moves
when the manifest does.) That is the fail-closed property of this actor: with no
attestations, every cell must plan nothing. Restore the file
(`git checkout src/insatsu/murakumo.cljk`) before continuing.

## 2. Check that `cell-specs` still agrees with the manifest

```bash
kbb --backend sci --classpath src scripts/check-manifest-cells.cljk
```

```
SCANNED	manifest-cells=4	cell-specs=4
OK manifest と cell-specs は一致している
```

**Why this is a separate step from step 1.** The contract suite introspects `cell-specs`,
so it is green whatever that map contains. Delete `:printmailjob` from `cell-specs` and
step 1 still reports `0 failures, 0 errors` — only the assertion count moves, 70 → 57, which
nobody reads. Step 2 is the only thing in this repository that notices.

**Make it fail — four ways, each with its own exit code and its own named reason:**

| Break | Exit | Says |
|---|---|---|
| delete `:printmailjob` from `cell-specs` | 1 | `FAIL declared in manifest, no cljc cell: ("com-etzhayyim-apps-insatsu-printMailJob")` |
| change `actor-did` to `did:web:WRONG.example.com` | 1 | `FAIL actor-did disagrees with manifest @id: cljc="did:web:WRONG.example.com" manifest="did:web:insatsu.etzhayyim.com"` |
| empty the manifest's `triggers.subscribeRepos.collections`, `requiredCollections` and `requiredLoops` | **2** | `REFUSED manifest が cell を 1 つも宣言していない` |
| corrupt `actor-manifest.jsonld` so it is not JSON | **2** | `REFUSED actor-manifest.jsonld を読めない: …` |

The exit code is three-valued on purpose. `2` is neither pass nor fail — it means the check
could not be performed. A checker that returned `0` for an unreadable or empty manifest
would make "measured nothing" indistinguishable from "measured, found nothing wrong", and
would then accumulate as a green tick forever.

Restore whatever you broke (`git checkout .`) before continuing.

## 3. Run the tests the way `deps.edn` declares

```bash
kbb -M:test
```

```
Running tests in #{"test"}

Testing insatsu.murakumo-test

Ran 9 tests containing 70 assertions.
0 failures, 0 errors.
```

Same counts as step 1 — the suite is `.cljc` and both runners see the same assertions. The
first run also downloads the test-runner and its transitive dependencies, so how long it
takes says more about your Maven cache and machine load than about this repository; do not
treat any single timing as expected output.

## 4. Lint

```bash
kbb -M:lint
```

```
src/insatsu/murakumo.cljk:82:14: warning: unused binding input
linting took …ms, errors: 0, warnings: 1
```

Exit `0`. The alias is `--fail-level error`, so the standing warning does not fail the run.
It is a real finding — `records-for` destructures `:as input` and never uses it — left in
place rather than silently fixed, because touching `src/` is a different piece of work from
documenting it. If you make it zero, update this line.

## Appendix — look at a plan

Neither a test nor a check; this is how you read what the actor would emit.

```bash
cat > /tmp/insatsu-probe.cljs <<'EOF'
(ns insatsu-probe (:require [insatsu.murakumo :as m] [clojure.pprint :as pp]))
(println "cells:" (sort (keys m/cell-specs)))
(println "--- no attestations ---")
(let [p (m/cell-plan :printmailjob {})]
  (println :status (:status p) :effects (:effects p) :missing (count (:missing-gates p))))
(println "--- all gates attested ---")
(let [att (into {} (map (fn [g] [g true]) m/common-gates))]
  (pp/pprint (:effects (m/cell-plan :printmailjob
                                    {:attestations att
                                     :request-id "req-demo"
                                     :computed-at "2026-09-01T00:00:00Z"}))))
EOF
kbb --backend sci --classpath src /tmp/insatsu-probe.cljs
```

```
cells: (:printmailjob :printpartner :shinka :shinkaevolution)
--- no attestations ---
:status :blocked :effects [] :missing 7
--- all gates attested ---
[{:op :mst/put-record,
  :actor "did:web:insatsu.etzhayyim.com",
  :collection "com.etzhayyim.insatsu.printmailjob",
  :rkey "req-demo",
  :record
  {:$type "com.etzhayyim.insatsu.printmailjob",
   :actorBoundary "cljc-migration-scaffold",
   :legacyCell "com-etzhayyim-apps-insatsu-printMailJob",
   …}}]
```

Two things to notice in that output, both explained in the [README](../README.md):

- the planned `:collection` is **not** the collection the manifest subscribes to. The
  manifest's name survives as `:legacyCell`.
- the `:actor` DID stamped into the record is `did:web:insatsu.etzhayyim.com`, which did not
  resolve in DNS when last measured. `.well-known/did.json` in this repository describes a
  *different* identifier, `did:web:etzhayyim.com:actor:insatsu`.

## What is not covered by any of this

- `actor-manifest.test.ts` does not run — there is no `package.json` or vitest dependency in
  this repository. Its assertions about `nanoid` and the four actor paths are checked by
  nothing.
- Steps 1–4 are run by hand. None of them is a fleet gate, so nothing runs them on push.
- Step 2 lives in `scripts/`, not in `test/`. Promoting it to a `deftest` would put it under
  both runners and inside step 1 — worth doing, and deliberately not done here.
