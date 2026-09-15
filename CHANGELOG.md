# Changelog

All notable changes to watch-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
