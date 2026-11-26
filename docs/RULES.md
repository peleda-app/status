<!-- dojo-rules v4 -->
# Rules

The rules every repository of the peleda-app workspace follows. This file is normative: "must" and "never" are requirements, "should" is the default that needs a reason recorded in `docs/DECISIONS.md` to depart from. It is generic on purpose; facts about one repository (its crates, its threat model, its extra rules) live in that repository's `AGENTS.md` and `docs/`.

The source of this file is `docs/RULES.md` in the dojo repository. Every other repository holds a byte-identical copy written by `dojo sync`. A repository may add stricter rules in its own `AGENTS.md`; it never loosens these except through an exception (EXC-1).

Each rule has an id. When a rule is enforced automatically, the ids of the `dojo check` checks that enforce it follow the rule as **Checks:**. Rules without checks are covered by the review checklist in dojo's `docs/REVIEW.md`; being unchecked never makes a rule optional.

## AGT: how the rules reach every repository

- **AGT-1. One copy, synced.** `docs/RULES.md` is written only by `dojo sync`, never edited by hand. A change to the rules is a change to dojo, followed by `dojo sync` in each repository. **Checks:** `RULES-COPY`.
- **AGT-2. AGENTS.md links the rules first.** Every repository has an `AGENTS.md` whose first instruction is to read [docs/RULES.md](RULES.md), followed by what is specific to that repository: what it is, its layout, its status, its documents, its extra rules. **Checks:** `RULES-AGENTS`.
- **AGT-3. CLAUDE.md imports the rules.** When a repository has a `CLAUDE.md`, it contains the lines `@docs/RULES.md` and `@AGENTS.md`, so the rules are in context at the start of every session and in every clone; it may also import `@AGENTS.local.md`, which exists only on machines that have one. **Checks:** `RULES-AGENTS`.
- **AGT-4. Local notes stay local.** `AGENTS.local.md` and `CLAUDE.local.md` hold personal notes for one machine (language of conversation, standing permissions) and are listed in `.gitignore`.
- **AGT-5. Every repository declares itself.** The root holds a `dojo.toml` whose `[repo] name` exists in dojo's `registry.toml`. The tier comes from the registry; a `tier` in `dojo.toml` that disagrees is an error. **Checks:** `CFG-MISSING`, `CFG-INVALID`.
- **AGT-6. Shared scripts are copies.** `scripts/check-unsafe.sh` and `fuzz/run.sh`, at the root and in every nested layer that has them, are installed by `dojo sync` from dojo's `shared/` and never edited in place; improvements go to dojo. `deny.toml`, `mise.toml` and the CI workflow start from dojo's `templates/` and may grow repository-specific entries. **Checks:** `SHARED-SCRIPTS`, `SHARED-DENY`.
- **AGT-7. `dojo check` is part of the gate.** `mise run check` and CI run `dojo check` in every repository, through the `dojo` task of dojo's `templates/mise.toml` (an installed `dojo`, else the dojo checkout beside the repository). Every manifest and file dojo reads must be readable by it: parsable, within its size limits, not a symlink. **Checks:** `SHARED-GATE`, `MOD-MANIFEST`, `REPO-READ`.

## WORK: working in a repository

- **WORK-1. Done means green.** A change is done only when `mise run check` passes. Load, fuzz, container and network suites run through their own `mise` tasks (`load`, `fuzz`, `it`, `network`) when the change touches what they cover.
- **WORK-2. Pinned toolchains.** The Rust toolchain is pinned in `rust-toolchain.toml` and `mise.toml`; the fuzz workspace pins its nightly in `fuzz/rust-toolchain.toml`. Run cargo through `mise exec -- cargo ...` or `mise run`, never through a `cargo` that ignores the pin.
- **WORK-3. Stay inside the repository.** Tools never rewrite files of a sibling repository: no `cargo fmt --all` in a workspace with path dependencies into siblings (use `cargo fmt -p <crate>` or plain `cargo fmt`). A change a sibling needs goes to that sibling.
- **WORK-4. Shared build resources.** Never `cargo clean` or redirect a build directory that other repositories share. Check free disk before release builds or a second artifact set; prefer `cargo test --workspace` narrowed with `--test` or a name filter over `-p`, which unifies features differently and builds another artifact set. Run heavy builds one at a time on a constrained machine.
- **WORK-5. No commits or pushes without an explicit instruction.** Neither agents nor tools commit, push, tag, publish, rename or change the visibility of a repository unless the maintainer asked for that action at that time.
- **WORK-6. Decisions and questions are written down.** A decision taken without the maintainer goes to `docs/DECISIONS.md` with its reason; what only the maintainer can decide goes to `docs/QUESTIONS.md`.

## MOD: modules and contracts

- **MOD-1. Contracts.** Every capability is a contract: a crate or package holding only traits, the types in their signatures, its error type, its conformance suite, and the pure checks every caller must run on untrusted output or wire helpers both sides share. No I/O, no platform or third-party SDK types. Its manifest declares `kind = "contract"`. **Checks:** `MOD-KIND`, `MOD-CONTRACT-TESTING`.
- **MOD-2. Implementations.** An implementation is named `<contract>-<kind>` (`storage-s3`, `sandbox-linux`), declares `kind = "implementation"` and `implements = "<contract crate>"`, and depends on its contract and on libraries, never on another implementation (tests and composition roots excepted). **Checks:** `MOD-KIND`, `MOD-CONTRACT-TESTING`.
- **MOD-3. Libraries are pure.** A library that is neither contract nor implementation has no I/O and no global mutable state: no sockets, files, clocks, randomness or environment. Time and randomness arrive as arguments or through a contract; a lazily parsed embedded table is the one acceptable static. A shared I/O helper with no trait and no state beyond one call is allowed when it depends on no contract or implementation; a service crate holds the logic of one service and receives every dependency as a contract.
- **MOD-4. Composition roots pick.** Only composition roots (binaries in `apps/`, a node or application builder, test harnesses) depend on implementations and choose them. Everything else receives `Arc<dyn Trait>` and never asks which implementation it got.
- **MOD-5. Object-safe traits.** Contract traits are `Send + Sync`, usable as `dyn Trait`, with no generic parameters on their methods; async methods go through `#[async_trait]`. CPU-bound contracts stay synchronous.
- **MOD-6. Nothing leaks.** No type of a third-party library, SDK, driver or protocol crate crosses a contract boundary, including inside errors; implementations wrap such errors in the contract's own variant.
- **MOD-7. One error enum per contract.** Each contract has one `thiserror` enum. Its `Display` never carries content (SEC-3). Libraries never return `anyhow::Error`; `anyhow` is a dependency of binaries only. **Checks:** `MOD-ANYHOW`.
- **MOD-8. Guarantees, not mechanics.** Every public trait, method and type has a doc comment saying what it guarantees: ordering, idempotency, limits, deadlines, concurrency, what happens on missing data or failure. The workspace sets `missing_docs = "warn"` and the check denies warnings. **Checks:** `MOD-LINTS`.
- **MOD-9. Extension points are public contracts.** Implementations may live in other repositories. A contract another repository depends on changes under semver; keep such contracts small and free of I/O.
- **MOD-10. One implementation of each thing.** Before writing a function, check whether this or a sibling repository already owns it; use theirs by dependency, never by copy. A repository that owns something never sees a second implementation grow elsewhere.
- **MOD-11. Generic layers stay generic.** A library meant for several consumers names none of them and knows nothing of their domain; consumers plug in through its public contracts, and the library never depends on a consumer.
- **MOD-12. Small modules.** A crate or package has one job.
- **MOD-13. Bounded resources.** Every cache, queue, list, index and buffer has a configured cap; nothing grows with traffic or input size. Large inputs are parsed as streams. Work is bounded by input caps, never by wall time.
- **MOD-14. Deterministic outputs.** Code whose output is stored, compared, signed or shown as a decision gives equal output for equal input, in the same order: no hash-map iteration order, locale, clock or thread count reaches it.
- **MOD-15. Declared kind.** Every package declares `[package.metadata.dojo] kind` as one of `contract`, `implementation`, `library`, `app`, `test`, `fuzz`. **Checks:** `MOD-KIND`.

## DOC: documentation

- **DOC-1. A README per module.** Every crate, app and package has `docs/README.md`: what it does and its public API with guarantees; for a contract, what an implementation must do and how to run the conformance suite; for an implementation, its configuration, limits and platform or provider quirks. An npm package may keep it as `README.md` at its root, where package registries show it (and where a `docs/` folder may be site content); the Rust half of an app whose parent package is documented shares that README. **Checks:** `MOD-DOCS`.
- **DOC-2. Docs change with the code.** A change to behavior or public API updates its docs in the same change.
- **DOC-3. Repository documents.** Every repository has `README.md`, `AGENTS.md` (AGT-2) and, while under development, `docs/PLAN.md`, `docs/DECISIONS.md` and `docs/QUESTIONS.md`. The status line in `AGENTS.md` says what exists and what does not.
- **DOC-4. Claims are tested.** Every security or privacy property a document claims has a test that tries to break it (TEST-5); a claim without one is a bug.

## TEST: tests

- **TEST-1. No change without tests.** Every change ships its tests in the same change. **Checks:** `TEST-PRESENT`.
- **TEST-2. Tests sit next to the code.** Unit tests live in `#[cfg(test)]` modules; property tests (`proptest`) cover anything that parses, validates, orders, merges, plans or derives. Fixtures are hand-written files in the crate's `tests/fixtures/`.
- **TEST-3. No network, no wall clock.** Tests never reach beyond loopback; providers, DNS and HTTP are tested against in-process fakes. Tests never sleep on wall-clock time where a manual clock can be injected. The only exceptions are `#[ignore]`d tests against real third-party services, run by `mise run network`.
- **TEST-4. Conformance suites.** Every contract ships a suite behind its `testing` feature (`<crate>::testing::...`), and every implementation runs it through a dev-dependency on the contract with that feature. Security properties (refusing strangers, replay, rollback, fencing, hostile input, oversized input) are part of the suite, not left to each implementation. **Checks:** `MOD-CONTRACT-TESTING`.
- **TEST-5. Attack tests at every break-in point.** Wherever an attacker can push, a test writes the attack down. Typical break-in points: decoders of untrusted bytes; document models and their validation; IPC and worker protocols; wire frames and logs from other nodes; signature, capability, membership, session, expiry and sequence-number checks; key derivation and wrapping; leader election, leases and fencing; sandbox walls (files, network, processes, signals, memory, CPU, inherited descriptors); responses of external services and model endpoints; push messages, webhooks and commands; anything the user or a customer can type; storage tiers that are not trusted. Typical attacks: forged, replayed, reordered, expired or truncated messages; oversized, bit-flipped and malformed input; a non-member, revoked or wrong sender; a lower fence or stale head; wrong key or context; cross-origin requests and CSRF; brute force; every limit hit at once.
- **TEST-6. Regression test before the fix.** Every bug found gets a test written before the fix and shown to fail on the old code. A security bug's test is named after it and kept forever; when a fuzz target exists, the failing input also joins its corpus.
- **TEST-7. Fuzz targets for every decoder.** Every function that decodes untrusted input has a `cargo fuzz` target in `fuzz/` (its own workspace and pinned nightly, driven by `fuzz/run.sh`). `mise run check` compiles the targets on stable; CI runs them nightly. **Checks:** `TEST-FUZZ`, `SHARED-SCRIPTS`.
- **TEST-8. Load tests where it matters.** Where throughput, contention or timing matter, load tests live in `tests/load` (or `#[ignore]`d tests) and run with `mise run load`; each states the numbers it expects so a regression is visible. A small version may run in `cargo test` and checks invariants only.
- **TEST-9. Real servers only in local containers.** Integration tests against real servers run only in containers on the local machine (`tests/integration`, `mise run it`).
- **TEST-10. Platform code is tested on its platform.** Linux-specific code is checked and tested in a Linux container; Windows code is at least compile-checked for its target.
- **TEST-11. Golden snapshots for decisions.** Where output is a versioned decision (classification, extraction, scoring), a golden snapshot over the fixture corpus makes every output change visible in review and forces a version bump.

## SEC: security and data

- **SEC-1. No unsafe code.** The workspace sets `unsafe_code = "forbid"` and every crate inherits the workspace lints. The only exception is platform FFI with no safe binding: such a file is listed in `unsafe-allowlist.txt` (the workspace then uses `"deny"`), starts with `#![allow(unsafe_code)]`, keeps every unsafe block small with a `// SAFETY:` comment directly above it, and its crate docs explain why. `scripts/check-unsafe.sh` runs in `mise run check`. **Checks:** `SEC-UNSAFE`, `MOD-LINTS`, `SHARED-SCRIPTS`.
- **SEC-2. Untrusted message content is parsed only in rinse.** Untrusted content entering the Peleda client, node or server (message bodies, attachments, remote resources) is parsed only by rinse's sandboxed workers, the same way on servers and devices. Nothing else links an HTML, image, PDF, font, archive or office parser for such input; other repositories consume rinse's checked document model. A repository whose purpose is parsing one specific format (a spam filter reading raw messages, an importer reading published lists) declares that boundary and its own hardening in its `AGENTS.md`.
- **SEC-3. Never log content or secrets.** Logs, metrics, traces, error messages, explain output and test failure messages carry ids, counts, timings, rule names, positions and shape skeletons (`dddddd`). Never content, subjects, names, addresses, URLs, codes, amounts, passwords, keys or tokens. Names that are application data appear as pseudonymous ids.
- **SEC-4. No real data.** Tests, fixtures, corpora and docs use invented content, invented people and brands, addresses on `example.com`, `example.org` and `example.net`, and the reserved `.example` and `.test` names. Real mail, real names, real codes and real customer data never enter a repository; evaluations over real data read a cache outside it.
- **SEC-5. Dependencies are reviewed.** Every Rust repository has a `deny.toml` from dojo's template and runs `cargo deny check` (advisories, bans, licenses, sources). A new dependency is justified in the change. **Checks:** `SHARED-DENY`.
- **SEC-6. Check, then commit.** A replay counter, fence, head, sequence number or state change is recorded only after every check on the message passed, including who the caller expects the sender to be.
- **SEC-7. Versioned, signed wire formats.** Every wire format that crosses a trust boundary carries a version and a context-bound signature.
- **SEC-8. Trust before privilege.** Nothing an untrusted party writes freely (display names, claimed senders, logos, headers from untrusted servers, model output) raises a privilege, a grade or attention. Only authenticated signals do.
- **SEC-9. Precision over recall for security.** A security-relevant decision (a code shown on a lock screen, a link marked safe, an entitlement) is granted only when every rule holds; when in doubt the answer is a lower grade or nothing.
- **SEC-10. Hostile input never crashes.** Code that reads untrusted input never panics on it and never fails as a whole: unreadable input gives fewer results or a typed error. Runners that call several analyzers isolate each one.
- **SEC-11. Linear-time matching.** Text from untrusted sources is matched only with linear-time engines and hand-written scanners with explicit caps. Never a backtracking engine, never a pattern from data that was not reviewed and compiled at build time.
- **SEC-12. Model output never steers.** Output of a language model is untrusted display text. It never feeds an access, trust, notification or security decision and never produces a code, a link or a grade on its own. External models are off by default and need the person's explicit, revocable consent.
- **SEC-13. Least privilege at the edges.** A component that faces untrusted input or users (a UI webview, a customer portal, a worker) holds no secrets and no power beyond its job; it asks a checked core for everything else. Commands that reach privileged systems are closed vocabularies with parameters from catalogs, never free-form strings, and every intent is re-checked by the side that acts.
- **SEC-14. No secrets in repositories.** Keys, tokens and passwords never enter a repository, a fixture or a log, test values included when they could work anywhere real.

## LIC: licensing and dependency direction

Tiers, their meaning and the CLA policy are explained in dojo's `docs/LICENSING.md`.

| Tier | License | Publish | May depend on |
| --- | --- | --- | --- |
| open | `MIT OR Apache-2.0` (or the registry's override, such as `MIT`) | allowed | open only |
| noncommercial | `PolyForm-Noncommercial-1.0.0`, commercial license from the maintainer | `publish = false` | open, noncommercial |
| private | none | `publish = false`, `"private": true` | open, noncommercial, private |

- **LIC-1. The registry decides the tier.** Every repository has an entry in dojo's `registry.toml`; `dojo.toml` never contradicts it. **Checks:** `CFG-INVALID`.
- **LIC-2. License files match the tier.** Open: `LICENSE-MIT` and `LICENSE-APACHE` (or `LICENSE` with the MIT text when the registry says `MIT`). Noncommercial: `LICENSE` or `LICENSE.md` with the PolyForm Noncommercial License 1.0.0. Private: no license file. **Checks:** `LIC-FILES`.
- **LIC-3. Manifests match the tier.** Cargo `license` (on the package or `[workspace.package]`) and `package.json` `license` carry the tier's SPDX expression; private manifests carry none (or `UNLICENSED`). **Checks:** `LIC-MANIFEST`.
- **LIC-4. Only open code is published.** Noncommercial and private crates are `publish = false`; private `package.json` files are `"private": true`. **Checks:** `LIC-PUBLISH`, `LIC-MANIFEST`.
- **LIC-5. Dependencies point toward more open tiers.** A dependency on a sibling repository (a Cargo path or git dependency, `[patch]` or `[replace]` entry in any manifest of the repository; a `link:`, `file:`, `portal:`, path or git npm dependency) is allowed only as the table says. Unknown siblings are reported; a retired repository is never a dependency. **Checks:** `DEP-DIRECTION`, `DEP-UNKNOWN`.
- **LIC-6. Contributions.** Noncommercial repositories accept contributions only under the signed CLA in `CLA.md`, since the maintainer must be able to sell commercial licenses for all of the code. Open repositories need only a DCO sign-off. **Checks:** `LIC-CLA`.
- **LIC-7. Say what noncommercial means.** The README of a noncommercial repository has the section from dojo's `templates/NONCOMMERCIAL-NOTICE.md`, filled in.
- **LIC-8. Third-party code and data.** Code and data from elsewhere enter only under a license the repository's tier may carry, recorded where they are used, with attribution in `NOTICE`. Never read, copy or translate code under a license the tier cannot carry (for example AGPL in an open repository); a clean-room repository says so in its `AGENTS.md`. Open repositories never add a non-open license to the `allow` list of `deny.toml`.

## STYLE: style and layout

- **STYLE-1. English.** Every file is in English: code, comments, docs, commit messages, fixtures.
- **STYLE-2. Comments are rare.** Doc comments on the public API are required (MOD-8). Inside function bodies, comment only what a reader cannot infer from the code.
- **STYLE-3. Commit messages and PR titles.** One line in Conventional Commits form with the module as scope (`feat(storage-s3): ...`, `fix(sandbox-linux): ...`). No body, no footers: no `Co-Authored-By`, no generated-by lines, no `Closes`/`Fixes` references. A pull or merge request description stays empty; the title is the whole message.
- **STYLE-4. Names.** Crate and package directories drop the repository's package prefix, package names keep it: with the prefix `acme-`, `crates/storage-s3` is `acme-storage-s3`. Each repository declares its prefix in its `AGENTS.md`. TypeScript packages are published under the `@peleda-app/` scope with names chosen per repository.
- **STYLE-5. Layout.** `crates/*` for libraries, contracts and implementations; `apps/*` for binaries and other composition roots; `packages/*` for TypeScript packages; `tests/integration` and `tests/load` for cross-crate suites; `fuzz/` for fuzz targets; `docs/` for project-wide documents.
- **STYLE-6. Workspace dependencies.** Dependency versions live in `[workspace.dependencies]` of the root `Cargo.toml`; members write `dep.workspace = true`. **Checks:** `MOD-WORKSPACE-DEPS`.
- **STYLE-7. Workspace lints.** Every member has `[lints] workspace = true`; the root sets `unsafe_code`, `missing_docs` and clippy lints; no Cargo configuration lowers them with `rustflags`; `mise run check` runs `cargo fmt --check` and `cargo clippy --workspace --all-targets -- -D warnings`. **Checks:** `MOD-LINTS` (inheritance, `unsafe_code`, `missing_docs`, `rustflags`; the clippy table is reviewed).

## REL: releases and CI

- **REL-1. Signed and attested.** Every released artifact is signed and attested from CI: container images with BuildKit provenance (`mode=max`), an SBOM, a keyless `cosign sign` of the manifest digest and a build provenance attestation; binaries and archives with `SHA256SUMS`, a build provenance attestation and a `cosign sign-blob --bundle` per file; npm packages with `--provenance` through trusted publishing; crates through crates.io trusted publishing.
- **REL-2. Only CI publishes.** Publishing runs only in GitHub Actions (environment `release`, `id-token: write`), never from a laptop. Private repositories publish nothing to public registries.
- **REL-3. Hardened workflows.** Actions are pinned by commit SHA; workflows start from `permissions: {}` and grant each job only what it needs; checkouts use `persist-credentials: false`; siblings are checked out beside the repository, private ones with a dedicated read token.

## EXC: exceptions

- **EXC-1. Ask in dojo.toml.** An exception to an automated check is an `[[allow]]` entry in the repository's `dojo.toml` with `check` (the check id), `path` (repository-relative) and `reason`. It is reviewed like code. An entry that matches nothing is reported so stale exceptions do not pile up. **Checks:** `CFG-UNUSED-ALLOW`.
- **EXC-2. Unsafe has its own list.** Unsafe code is allowed only through `unsafe-allowlist.txt` (SEC-1), never through `[[allow]]`.
- **EXC-3. Unchecked rules.** A departure from a rule no check enforces is recorded with its reason in the repository's `docs/DECISIONS.md` and is visible in review.
- **EXC-4. Fuzzing opt-out.** A Rust repository with no decoder of untrusted input may set `[checks] fuzz = false` in `dojo.toml`, with the reason in `docs/DECISIONS.md`.
