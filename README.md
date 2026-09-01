# insatsu-actor — the cljc actor boundary for a cloud print & mail network

`cloud-itonami/insatsu-actor` holds the **actor identity and the pure `.cljc` planning
boundary** for 印刷 (*insatsu*, "printing"): who this actor is
(`actor-manifest.jsonld`, `.well-known/did.json`) and, given a set of gate attestations,
**what records it would write** (`src/insatsu/murakumo.cljc`).

It plans. **It does not print, address, frank, or post anything.** `cell-plan` returns a
`:blocked` or `:ready` value containing `:mst/put-record` effect *descriptions*; nothing in
this repository opens a socket, holds a key, or writes to a PDS. That separation is the
point — the decision to publish a print-and-mail job, which carries a recipient's postal
address, is reviewable as a value before anything leaves the process.

## Boundary against the nearest repository

| Repository | What it owns |
|---|---|
| **`cloud-itonami/insatsu`** | The application code — the `kotoba/` record layer and the appview. Its own README states plainly that the identity is *not* there |
| **`cloud-itonami/insatsu-actor`** (here) | The actor's DID composition, its manifest, and the `.cljc` cell-plan boundary derived from it |

If you are looking for the print-job records, the shop routing, or the deployed surface, it
is in `cloud-itonami/insatsu`. If you are looking for *what this actor is allowed to emit
and under which attestations*, it is here. The two repos have no build or dependency
relationship; the only thing that crosses is the DID string.

Downstream, `actor-manifest.jsonld` names one entity: `did:web:yuubin.etzhayyim.com`, used
when the destination country is JPN and the route is compatible with Japan Post automation.
Nothing in this repository calls it.

## What is actually in this repository

There is no build output, no appview, and no runtime here — the whole repository is one
manifest, one namespace, its tests, and the documents you are reading.

| Path | What it is |
|---|---|
| `actor-manifest.jsonld` | Actor definition: 4-way path-based Multi-DID composition (`network:broker`, `partner`, `lane:postal`, `lane:hybrid`), 4 capabilities, the subscribed collections, the governance block, the convo system prompt |
| `.well-known/did.json` | A DID document for `did:web:etzhayyim.com:actor:insatsu` — see *Two identifiers* below before trusting it |
| `src/insatsu/murakumo.cljc` | The planning boundary: `cell-specs`, `missing-gates`, `records-for`, `cell-plan`, `all-cell-plans` |
| `test/insatsu/murakumo_test.cljc` | 9 contract tests / 70 assertions. Introspects `cell-specs` rather than hardcoding cell names, so it holds as the manifest changes — and see *What the tests do not see* |
| `scripts/check-manifest-cells.cljs` | The drift check the contract tests structurally cannot do: manifest ↔ `cell-specs` |
| `actor-manifest.test.ts` | A vitest suite that **cannot run in this repository** — see *What does not run here* |
| `storage-profile.edn` | Declares the `:kotoba/local-agent-kagi-chunks-v1` storage profile: local query, append-only transactions, kagi-chunked remote payload |
| `deps.edn` | `:test` (cognitect test-runner) and `:lint` (clj-kondo) aliases |
| `NOTICE` | Apache-2.0 + etzhayyim Charter Compliance Rider v3.1 |
| `.gitignore` | `.cpcache/` |

## The shape of the boundary

`cell-specs` names one planning cell per manifest declaration. Every cell carries the same
seven `common-gates`. `cell-plan` is total on that map:

- **any gate unattested** → `{:status :blocked :effects []}` with the missing gates named.
  It does not partially proceed. With no attestations at all, every cell reports all seven.
- **all seven attested** → `{:status :ready :records [...] :effects [...]}` with one
  `:mst/put-record` per declared collection.
- **unknown cell** → `ex-info`, not `nil`. A typo in a cell name cannot silently plan nothing.

## The collection an effect names is not the collection the manifest names

This surprises people, so it is stated here rather than left to be discovered. The manifest
subscribes to `com.etzhayyim.apps.insatsu.printMailJob`. The effect that `cell-plan`
produces names `com.etzhayyim.insatsu.printmailjob` — a different string: no `apps`
segment, and lower-cased. The manifest's own name survives inside the record, as
`:legacyCell "com-etzhayyim-apps-insatsu-printMailJob"`.

Both names are load-bearing and neither is a typo, but a consumer that greps for the
manifest's collection name will not find the planned record. `scripts/check-manifest-cells.cljs`
pins the pairing so it cannot drift silently.

## What the tests do not see

`test/insatsu/murakumo_test.cljc` introspects `cell-specs`. That is the right design for a
contract suite — it holds whatever the manifest declares — but it means the suite is
**green when a cell is deleted**: delete `:printmailjob` and the run goes from
9 tests / 70 assertions to 9 tests / 57 assertions, still `0 failures, 0 errors`.

The suite checks that `cell-specs` is internally consistent. It never checks that
`cell-specs` still agrees with `actor-manifest.jsonld`. That is what
`scripts/check-manifest-cells.cljs` is for, and step 2 of the
[operator quickstart](docs/operator-quickstart.md) walks it.

## Two identifiers, one actor — and only one of them resolves

This repository carries the actor's name in two different DID forms:

| Where | Identifier |
|---|---|
| `actor-manifest.jsonld` `@id`, and `insatsu.murakumo/actor-did` — stamped into every planned record | `did:web:insatsu.etzhayyim.com` |
| `.well-known/did.json` `id` | `did:web:etzhayyim.com:actor:insatsu` |

Nothing in this repository asserts that these two are the same actor: the `alsoKnownAs`
list in `.well-known/did.json` does not mention the subdomain form.

Measured 2026-09-01 — **re-measure rather than quoting these results**, they are about the
network, not about this tree:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://etzhayyim.com/actor/insatsu/did.json
curl -s -o /dev/null -w '%{http_code}\n' https://insatsu.etzhayyim.com/.well-known/did.json
```

On that date the path form returned **200** and the subdomain form failed to resolve in DNS
(`curl: (6) Could not resolve host`). The sibling repo `cloud-itonami/insatsu` reports the
same for the subdomain form, independently. So **the DID that this actor writes into every
record it plans does not currently resolve.**

Two further facts about the 200, both worth knowing before treating the checked-in file as
authoritative: the served document is **not** the copy in this repository — it advertises
`https://pds.aozora.app` and an `xrpc-libp2p` service and carries an empty `alsoKnownAs`,
where the checked-in copy advertises `https://pds.etzhayyim.com` and `https://aozora.app`
and lists four `alsoKnownAs` entries.

Neither of these is fixed here. This repository plans records; it does not own DNS or the
published DID document. They are recorded so that an operator reading a `:blocked` plan does
not spend the afternoon looking for the fault in `cell-specs`.

## What does not run here

`actor-manifest.test.ts` is a vitest suite asserting the manifest's `@id`, `nanoid`, and the
four actor paths. There is **no `package.json`, no lockfile, and no vitest dependency** in
this repository, so `npx vitest` has nothing to resolve. The file documents an intent; it is
not a check anything currently performs. The assertions it makes about `@id` are covered by
`scripts/check-manifest-cells.cljs`; the ones about `nanoid` and the four actor paths are
not covered anywhere.

## Running it

See **[`docs/operator-quickstart.md`](docs/operator-quickstart.md)** — four steps, each with
the command, what it prints, and how to make it fail on purpose.
