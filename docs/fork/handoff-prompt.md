# Handoff prompt: building a game server on a SpacetimeDB fork

> Paste this whole file as the first message to a new agent, started in the root of the
> fork. The fork is named **RhinoDB** and is based on upstream tag `v2.11.0`. The reader-facing version of this file, with
> the current state, is [`docs/fork/README.md`](README.md); read it first.

---

## Who I am and what I want

I'm building an **online soccer manager game**. For the past weeks I built my own in-memory
game-server database in C# (RhinoDB, now finished and published at
https://github.com/Mehmovic/rhino-db, AGPL-3.0) to learn how databases work: storage, indexes,
transactions, WAL durability, crash recovery, two-phase commit, networking.

I've decided to stop and build on **SpacetimeDB** instead, which already is what RhinoDB was
heading toward. I'll work on a **fork of SpacetimeDB** (Rust), self-hosted on **one node**.
I'm **learning Rust** as I go: explain Rust concepts when they come up (ownership, lifetimes,
traits, async, unsafe), briefly and in context, not as lectures.

Game modules may be written in C# (SpacetimeDB supports C# modules), where I'm already
fluent. Rust is needed only for changes to the engine itself.

## How I like to work

- **Plan + tests first, then build.** For anything non-trivial: write the plan, write the
  failing tests, then implement. When I say "plan" or "tests", don't write production code.
  When I say "build", "implement" or "fix", do.
- **Never commit or push unless I ask.** I sometimes edit files directly while you work;
  treat changes on disk as intentional.
- **Verify, don't assume.** Read the actual code before claiming how something works, and
  say so when something is unverified. Be honest about failures: show test output, don't
  hide flakiness.
- **Crash and fault tests beat unit tests** for storage and durability code. Test on Linux
  early, not just Windows: in RhinoDB a whole class of bugs only showed up on Linux. I have
  WSL (Ubuntu 24.04).
- **Readable names that say what they mean.** Comments are lean and explain *why*, not *what*.
- **User-facing docs:** keep a `docs/fork/` folder in the fork describing every change we make
  and why, so the fork stays understandable and its changes stay separable from upstream.

## Licensing: what we verified (2026-10-07) and what's still open

SpacetimeDB is under **BSL 1.1**, not MIT. From `LICENSE.txt` (v2.11.0):

- **Allowed:** "copy, modify, create derivative works, redistribute, and make non-production
  use".
- **Production use (Additional Use Grant):** "no more than one SpacetimeDB instance in
  production" and not as a "Database Service". A Database Service lets third parties create
  tables whose schemas they control; a game doesn't.
- **Change Date:** the earlier of 2031-09-15 or 4 years after each release's first public
  distribution. Each release then becomes AGPL-3.0 with a linking exception. After
  conversion, multiple nodes are allowed for *that* release. Merging newer upstream code
  brings back the BSL limit for those parts.
- **After conversion, AGPL's network clause applies to the modified server.** Players reach
  it over the network, so the fork's server source must be offered to them. The linking
  exception is meant to keep game modules closed.
- **Trademark:** the BSL grants no trademark rights. Renaming the fork is allowed and
  advisable. Present it as "X, a fork of SpacetimeDB", keep their `LICENSE.txt` and
  copyright notices intact, and never relicense their code. If renaming, rename only
  user-facing things (product name, binary, CLI, docs), not internal crate names, so
  upstream merges stay manageable.
- **Clustering and replication** are in the proprietary SpacetimeDB Cloud, not in
  open-source Standalone. Multiple nodes means building it ourselves, or a commercial license.

**Open:** I'm emailing Clockwork Labs to confirm:
- what counts as "one instance" (dev, staging, a cold standby);
- that a modified build is covered by the grant;
- that renaming is acceptable;
- how conversion works per release;
- the exact linking-exception text;
- whether tiered storage reaches Standalone;
- commercial license terms.

Treat these as open until I report their answer. **Hard rule: never set up a second
production instance.**

## Technical facts we found (verify against the code before relying on them)

- **All table data lives in RAM today** on the leader node.
- **Tiered storage is announced for 2026-10-31** ("Spacetime Continuum"): disk tables and
  object-storage tables, as a cache hierarchy (memory, NVMe, object storage). A transaction
  touching non-resident data is aborted, the data fetched, and the transaction retried.
  Unknown: per-table vs automatic, and whether it ships in Standalone or only Cloud.
- **Event tables** exist: rows are deleted at the end of each transaction, but inserts *are
  still written to the commitlog*. So there is no truly non-durable table today.
- **Modules run in a WebAssembly sandbox** (Wasmtime; V8 for TypeScript), with
  fuel/energy metering.
- **Standalone performs no server-level authentication:** anyone who reaches the port can
  create or publish databases. It must be firewalled or put behind a gateway.

## What I want to change, in order

### 0. Set up (first session)
1. Clone SpacetimeDB at the **latest release tag**, not master, and record the tag and its
   release date (the date decides when that release converts to AGPL). Set up my fork
   remote; keep `upstream` pointing at clockworklabs/SpacetimeDB.
2. Install the Rust toolchain the repo pins. Build, run the test suite on Windows **and** in
   WSL, and report what passes, fails or is skipped before changing anything.
3. Run Standalone locally (`spacetime start`), publish a tiny C# module and a tiny Rust
   module, and call a reducer from a client, end to end.
4. **Map the codebase** into `docs/fork/00-codebase-map.md`:
   - crates and their roles (datastore, table storage, commitlog, durability, snapshots,
     module host/wasm, subscriptions, client API, standalone, bindings, SDKs);
   - the path of one reducer call, from WebSocket message to commit to subscription update;
   - where durability happens.

   Note which crates are BSL and which are Apache-2.0 (check each `Cargo.toml` `license`
   field).

### 1. Lock down a single-tenant node (small, early)
- **Publishing and database creation** restricted to me (a token, or bound to localhost and
  published over SSH).
- **Investigate energy/fuel metering:** where it lives, what it costs, and whether
  Standalone already disables it. Disable it if it's measurable overhead.
- **Keep the WASM sandbox.** Removing it is a rewrite, and a native module crash would take
  the database down.
- **Keep client identity/auth.** Players need it.

### 2. A transient (non-durable) table type
For short-lived match/room state: rows live in memory, are never written to the commitlog
or snapshots, and are empty after a restart. They behave like normal tables otherwise
(indexes, queries, subscriptions).
- Plan and tests first: commit path, snapshot exclusion, replay and recovery, what
  subscribers see after a restart, schema/migration behavior.
- This is the RhinoDB "Instant table" idea. Keep the diff small and isolated.
- Measure before and after; that decides whether step 4 is needed at all.

### 3. Tiered storage decision (after 2026-10-31)
Check what shipped and whether it's in Standalone. If it's Cloud-only, we decide together
whether eviction to disk is worth building in the fork. That's a big job, not a default.

### 4. Only if measurements prove it: a columnar / ECS store for match simulation
A separate storage engine is months of work and touches upstream's busiest code. Don't
start without benchmark evidence that the transient row store from step 2 is too slow.

## Lessons from RhinoDB to carry over
- **Group commit only scales if the fsync happens outside the append lock.** In RhinoDB that
  one change took two-phase commit from ~1.3k to ~12.5k transactions per second.
- **A draining/shutdown path must wait for the *real* in-flight work.** RhinoDB had a bug
  where it returned the outer task of `StartNew(async ...)`, and disposed storage under a
  running operation.
- **Flaky native crashes in test runs** can be runtime bugs, not ours. A .NET collectible
  `AssemblyLoadContext` plus thread statics corrupted the GC heap
  (dotnet/runtime#134931). Use crash reports and debug symbols (lldb), and controlled
  experiments that toggle one factor at a time, before blaming the engine.
- **Errors to clients:** a kind plus a custom code, never internal messages. Engine-health
  failures collapse to one "server unavailable" kind.
- **Settings in one config file**, enforced at compile time where possible.

## First message from you, please

Confirm you've read this. Then start step 0.1-0.2: clone at the latest release tag, build,
run tests on Windows and WSL. Report the results before changing anything. Ask me for the
fork's GitHub remote URL and the fork's name when you need them.
