# RhinoDB, a fork of SpacetimeDB

RhinoDB is a fork of [SpacetimeDB](https://github.com/clockworklabs/SpacetimeDB) by
Clockwork Laboratories, Inc. It is the server for an online soccer manager game, self-hosted
on **one node**. This folder (`docs/fork/`) describes every change the fork makes to upstream
and why, so the fork stays understandable and its changes stay separable from upstream.

SpacetimeDB's `LICENSE.txt` and copyright notices are untouched. Upstream code is not
relicensed. "SpacetimeDB" is Clockwork Labs' trademark; this fork is not affiliated with or
endorsed by them.

## Why this fork exists

Before this fork I built [RhinoDB in C#](https://github.com/Mehmovic/rhino-db) (AGPL-3.0): an
in-memory game-server database written bottom up to learn how databases work: storage,
indexes, transactions, WAL durability, crash recovery, two-phase commit, networking. It is
finished as a learning project.

While building it I found that SpacetimeDB already is the system RhinoDB was heading toward:
data in memory, one writer per database, transactional functions called by clients, and
subscriptions pushing changes to them. It is further along, maintained by a team, and running
real games. Rebuilding that (the subscription engine above all) would have cost months that
belong to the game.

So the name moves to this fork. It builds on SpacetimeDB and changes only what the game
needs and upstream doesn't offer.

## Base

| | |
|---|---|
| Upstream | `upstream` → https://github.com/clockworklabs/SpacetimeDB |
| Base tag | `v2.11.0` (commit `6b6e0ab19e4f8f52472aa4379552e9593c8f53bb`, dated 2026-10-05) |
| Toolchain | Rust `1.96.1` + `wasm32-unknown-unknown` (pinned by `rust-toolchain.toml`) |

The fork tracks the **latest upstream tag**, not master, and moves to newer tags to pick up
bug fixes. Every move is recorded here with the new tag and its date, because that date
decides when that release's code converts to AGPL (see below).

Note: on 2026-10-07, `v2.11.0` was tagged upstream but not yet published as a GitHub Release
(the latest published release was `v2.10.2`, 2026-09-29).

## Building on Windows

- Rust comes from `rust-toolchain.toml`; rustup installs it on the first `cargo` call.
- **A full Perl is required.** OpenSSL is built from source (`openssl` with `vendored` in the
  root `Cargo.toml`), and its configure step runs Perl. The Perl bundled with Git for
  Windows is missing modules (`Locale::Maketext::Simple`) and fails. Install Strawberry Perl
  (`winget install StrawberryPerl.StrawberryPerl`) and make sure it comes first on `PATH`.
- Build the binaries: `cargo build --release -p spacetimedb-cli -p spacetimedb-standalone`
  (about 17 minutes from clean on 8 cores).
- Upstream CI runs the test suite (`cargo ci test`) on Linux only; Windows CI builds only.
  `cargo test --all` doesn't compile on Windows: `spacetimedb-dst-lib` turns on the
  runtime's `simulation` feature (Linux/macOS only), and module crates such as
  `modules/environment-test` get linked as native test binaries whose WASM host imports
  `lld-link` can't resolve. Run the tests on Linux.

## Licensing

SpacetimeDB is under the **Business Source License 1.1**, not an open-source license. From
`LICENSE.txt` at v2.11.0:

- **Allowed:** copy, modify, create derivative works, redistribute, and non-production use.
- **Production use (Additional Use Grant):** "no more than one SpacetimeDB instance in
  production", and not as a "Database Service" (a commercial offering where third parties
  create tables whose schemas they control). A game doesn't do that.
- **Change Date:** the earlier of 2031-09-15 or the fourth anniversary of each version's
  first public distribution. That version then becomes AGPL-3.0 with a linking exception.
  For v2.11.0 that's about 2030-10-05 if the tag date counts as the first public distribution
  (not yet confirmed). Merging newer upstream code brings back the BSL limits for that code.
- **After conversion, AGPL's network clause applies:** players reach the modified server
  over the network, so its source must be offered to them. The linking exception is meant
  to keep game modules closed.
- **Trademark:** the BSL grants no trademark rights. Hence the rename: only user-facing names
  (product, binary, CLI, docs) change. Internal crate names stay, so upstream merges stay
  manageable.
- **Clustering and replication** live in the proprietary SpacetimeDB Cloud, not in
  open-source Standalone.
- Some subdirectories carry their own license (e.g. Apache-2.0 for some bindings/SDKs). The
  crate-by-crate list goes in the codebase map.

**Hard rule: never run a second production instance.**

**Open questions to Clockwork Labs** (unconfirmed until they answer):
- what counts as "one instance" (dev, staging, a cold standby);
- that a modified build is covered by the grant;
- that renaming the fork is acceptable;
- how per-release conversion works, and which date counts as "first public distribution";
- the exact linking-exception text;
- whether tiered storage reaches Standalone;
- commercial license terms.

## What the fork changes, in order

Each step gets its own document in this folder before any code is written.

0. **Set up.** Fork at the latest tag, build on Windows, run the test suite on Linux (WSL),
   run Standalone end to end with a C# and a Rust module, and map the codebase
   (`00-codebase-map.md`): crates and their roles, the path of one reducer call from
   WebSocket to commit to subscription update, where durability happens, and which crates
   are BSL vs Apache-2.0.
1. **Lock down a single-tenant node.** Standalone performs no server-level authentication:
   anyone who reaches the port can create or publish databases. Restrict publishing and
   database creation to the operator (a token, or bind to localhost and publish over SSH).
   Investigate energy/fuel metering (where it lives, what it costs, whether Standalone
   already disables it) and disable it if it's measurable overhead. **Keep** the WASM
   sandbox (a native module crash would take the database down) and client identity/auth
   (players need it).
2. **A transient (non-durable) table type** for short-lived match/room state: rows live in
   memory, are never written to the commitlog or snapshots, and are empty after a restart.
   Otherwise they behave like normal tables (indexes, queries, subscriptions). Event tables
   don't cover this: their inserts are still written to the commitlog. This is the C#
   RhinoDB's "Instant" table. Small, isolated diff, with measurements before and after.
3. **Tiered storage decision** (after upstream's announced 2026-10-31 release). If disk tiers
   ship in Standalone, use them. If they're Cloud-only, decide whether eviction to disk is
   worth building here. That's a big job, not a default.
4. **Only if measurements prove it:** a columnar / ECS store for match simulation. It means
   months of work in upstream's busiest code. Don't start without benchmark evidence that
   step 2 is too slow.

## How changes are made

- **Plan and tests first**, then the implementation.
- **Crash and fault tests** beat unit tests for storage and durability code.
- **Windows and Linux (WSL).** Some bugs only show up on Linux.
- **Every change documented here**, with what it touches upstream, so it can be re-applied
  or dropped on the next upstream move.
- **Readable names**; comments explain *why*, not *what*.

## Lessons carried over from the C# RhinoDB

- **Group commit only scales if the fsync happens outside the append lock.** That one change
  took two-phase commit from ~1.3k to ~12.5k transactions per second.
- **A draining/shutdown path must wait for the real in-flight work.** The C# RhinoDB had a
  bug where it awaited the outer task of `StartNew(async ...)` and disposed storage under a
  running operation.
- **Flaky native crashes can be runtime bugs, not ours** (dotnet/runtime#134931: a collectible
  `AssemblyLoadContext` plus thread statics corrupted the GC heap). Use crash reports, debug
  symbols and controlled one-factor experiments before blaming the engine.
- **Errors to clients:** a kind plus a custom code, never internal messages. Engine-health
  failures collapse to one "server unavailable" kind.
- **Settings in one config file**, enforced at compile time where possible.
