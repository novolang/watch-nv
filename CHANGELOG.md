# Changelog

All notable changes to watch-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Changed

- The errno mappings and the inotify mask dispatch are `match` expressions (the errno constants are now upper-case `const` items, which a pattern compares rather than binds), and the debounce lookups (`index_of`, `next_deadline`) answer an optional rather than `-1`; no public signature or behaviour changes.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-11

Every body implemented, published over the 0.0.3 interface.  Nothing was
added, removed or renamed: the signatures, the effect rows and the type
shapes are the ones the interface release declared, and the suites that
were written against them are green on novo 0.20.2 with no leak line.
What follows is what the bodies turned out to be and what a consumer
should know before depending on them.

### Added

- `wtevent` pairs a rename's two halves into one `WtRenamed(from, to)`
  and ages a half whose partner never comes into `WtMovedOut` or
  `WtMovedIn` at the pairing's deadline.  `paths_of` answers both ends,
  so a consumer never has to know which event kind it holds.  The
  overflow predicate is one function the loop and the caller share.
- `wtfilter` calls `glob.may_descend` for the descend decision, so a
  directory no pattern can match is never watched and never walked.
  Exclusion is applied before inclusion, and an empty include list means
  everything rather than nothing.
- `wtdebounce` is two windows over a value: a quiet window that emits
  once the burst stops and a cap that emits anyway, so a checkout that
  writes for thirty seconds still reaches the consumer.  `fold` keeps the
  more informative event, and `next_deadline` answers when the caller
  should look again.
- `wtwatch` walks the tree through the filter, places a watch when a
  directory appears, and replays a recorded transcript at `[]`, which is
  what lets the loop be tested with no filesystem, no clock and no sleep.
- `wtpoll` compares two directory snapshots by size, mtime and mode.  An
  mtime that went backwards is a change, because a checkout and a restore
  both produce one, and `rehash_suspicious` names the paths a `stat`
  comparison cannot settle.
- `wtinotify` places one watch per directory, reads the three kernel
  limits from `/proc`, and answers the `sysctl` line that raises one.
  The record parser reads the 16-byte header and the name field of every
  event in one read, pairs the `MOVED_FROM`/`MOVED_TO` cookie, and
  reports the overflow pseudo-descriptor as an overflow.
- `wtloop.step` is one turn — poll, pair, expire, filter, debounce,
  watch new directories, rescan on demand — with the clock reading
  supplied, so a caller with its own event loop runs it inside that loop.
  `run_on_change` is that turn with the sleep and the action attached.

### Changed

- **Twenty `== false` comparisons across the seven committed suites are
  spelled `not`.**  The 0.20.2 linter raises `redundant-bool` on each and
  suggests exactly that rewrite, which is the same predicate.  All twenty
  are pre-existing — `wtfilter` 10, `wtevent` 3, `wtwatch` 3, and one
  each in `wtdebounce`, `wtinotify`, `wtloop` and `wtpoll`, the same
  counts at the same lines at `HEAD` — and no assertion's meaning
  changed.  Every `.nv` file in this package now lints with no warnings.

### Known

- **Coverage is the merged per-suite line count over `src/`.**  The
  eight suites run once each with `--cov`, the per-line maximum is taken
  across them, and the result is 98% of 626 instrumented lines.  Seven of
  those suites are values only; `tests/watch_edge_tests.nv` is the one
  that opens a descriptor and a temporary directory, driving a real
  inotify watcher through a write, a rename, a `mkdir` and an `rmdir`,
  walking a real tree with the poller, and parsing an inotify buffer of
  the kind the kernel writes after a queue overflow.
- **Eight regions under `src/` are excused, each with its reason at its
  `cov: skip` marker.**  `wtfilter`'s `base_name` guards an empty
  `str.split` answer that cannot occur, and `wtpoll`'s `walk` skips a `.`
  or `..` that `fs.read_dir` never answers.  `wtinotify`'s `failed_read`
  and its call site in `poll` are the read that failed after the
  readiness call said ready, which leaks one object in the runtime;
  `unwatch` and `close` fall back to a failure whose errno neither call
  answers on a descriptor the process still holds; `open_with` covers an
  `inotify_init1` refused for want of descriptors or instances; and
  `is_available` covers a kernel without inotify.
- **A raw inotify event's path is in the watched directory's frame.**
  The kernel answers the name relative to the directory the watch was
  placed on, and `WtRawEvent.watch_id` is what resolves it through
  `wtwatch.watch_of`.  The polling and replay backends answer in the
  root's frame directly.  `wtloop` does not join the two, so an event
  from a nested directory carries a path with no root prefix and a
  root-relative filter pattern will not match it.  A consumer that wants
  one frame either uses `wtwatch`'s tree itself or watches a flat
  directory.
- **`WtInotify.watches_held` answers `0`.**  The instance keeps no
  registry of its own, so the count the kernel holds cannot be read from
  it; the number a session actually has is `wtwatch.watch_count` over the
  tree the caller holds.
- **`WtInotify.poll` asks the descriptor whether it is ready before it
  reads.**  A drain-first read answers `None` when the queue is empty —
  the ordinary idle case — and `io.read_fd` leaks one object on every
  failed read (a `std.io` defect, filed against the toolchain and fixed
  in 0.20.3, above this package's 0.20.2 floor).  Asking
  `io.wait_either` first costs one syscall more per poll and leaks
  nothing; the read happens only when the kernel says there is something
  to read.  `non_blocking` is still recorded on the instance, and `poll`
  honours its deadline either way.
- **The record parser scans the name field for its first zero byte
  before it makes a `Str`.**  The kernel pads that field with NULs to the
  record's alignment and `bytes.to_str` keeps every byte it is given,
  though the `std.bytes` page says otherwise (a documentation defect,
  filed against the toolchain and fixed in 0.20.3, above this package's
  0.20.2 floor).  A `Str` built from the padded field
  compares unequal to the name it looks like.
- **The libc boundary is private.**  `wtinotify` declares its five libc
  symbols behind private `@ffi` shims and discharges `[ffi]` inside them,
  so no published signature carries an effect the 0.0.3 interface did not
  declare.  The module links against libc and is a host-layer module:
  the README's claim about microcontrollers stands unchanged.
- **The polling backend reports the whole tree on every `poll`.**  A
  poller keeps no snapshot between calls, so each `poll` compares what it
  scans against an empty snapshot and answers every admitted path as a
  creation.  A caller that wants the changes compares two snapshots of
  its own with `wtpoll.scan` and `wtpoll.diff_snapshots`, which performs
  no input or output.  The `slot` cell is the caller's, and this backend
  neither reads nor writes it.
- **A rename the polling backend cannot see is still invisible.**  No
  `stat` result connects two paths, so the poller reports a removal and a
  creation rather than a rename.
- **One dependency**, glob-nv, by registry range.

## 0.0.4 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

## 0.0.3 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves glob-nv 0.1.1 to 0.1.4.  glob-nv 0.1.1 writes into
  lists through names that are not declared `var`, which novo 0.11
  refuses (E2038), so this package did not build with novo 0.11 against
  it.  No requirement in the manifest changed.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `wtevent` — a rename as one event with two paths, the honest halves
  for the cases with no partner, the overflow and lost-watch arms a
  consumer must handle, and the pairing state machine with its deadline.
- `wtfilter` — include and exclude over glob-nv, the descend decision
  that keeps a watch count at twenty rather than two hundred thousand,
  and the editor-artifact test.
- `wtdebounce` — two windows rather than one, so a continuous burst does
  not starve the consumer; the coalescing rule as its own function; and
  the next-deadline call that turns a spin into a sleep.
- `wtwatch` — `WatchBackend[e]`, the tree walk that descends only where
  the filter allows, the new-directory call recursion needs, and the
  replay backend at `[]`.
- `wtinotify` — the Linux backend at `[io]`, the three kernel limits
  read from `/proc`, the `sysctl` line an operator needs, and the blind
  spots as data.
- `wtpoll` — a snapshot as a value and the comparison as `[]`, with the
  mtime compared for inequality rather than for being newer, and the
  paths a `stat` comparison cannot settle reported rather than hidden.
- `wtloop` — `run_on_change` whole, and `step` with the sleep taken out
  for a caller that owns its own event loop.

### Known

- **`WatchBackend[e]` is the load-bearing interface**: inotify is
  `[io]`, polling is `[fs]`, a replayed transcript is nothing, and the
  loop is written once.
- **The rows are finer than the plan's row guessed.** `[io, time]` was
  the intention; three modules declare nothing, `[time]` appears in one
  function, and `run_on_change` grew `[io]` from the action's own row —
  which the compiler insisted on, on a package with no bodies in it.
- **Recursion is not a kernel feature**, and the README says what that
  costs: a watch per directory, a limit an operator has to raise, and a
  window between a `mkdir` and the watch reaching it.
- **The polling backend detects no renames**, because nothing in a
  `stat` connects two paths.
- **A toolchain defect was hit again**: a wrapper returning another
  module's `?Float` does not compile, which is the `?Int` case filed the
  day before.
- **No device claim.** `Str`, `Result` and a filesystem are throughout.
- **One dependency**, glob-nv, by registry range.

### Design notes

- There is no `--watch` verb in the compiler today. Four places would
  take this package: `novo build --watch` and `novo doc --watch` want
  `run_on_change` whole with `wtfilter.common_excludes()`; `novo test
  --watch` wants the same with an action that runs the suites the
  changed files belong to, which is why `WtEvent.path` is relative; and
  a language server wants `step` rather than `run_on_change`, because
  it owns its own event loop and `workspace/didChangeWatchedFiles` is
  exactly a `[WtEvent]` batch.
- `novo test --watch` is the reason `WtEvent` carries both ends of a
  rename. A moved test file must not re-run as a new one and a deleted
  one.
- An enum of backends instead of the effect-parameterised interface
  would charge every caller the union of both backends' effects, the
  tests included.
