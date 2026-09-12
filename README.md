# watch-nv

Filesystem watching for novo-lang: an event model that can say what
actually happened, two backends behind one trait, and the run-on-change
loop a `--watch` mode is.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

| module | holds | rows |
| --- | --- | --- |
| `wtevent` | the event model, and rename pairing | `[]` |
| `wtfilter` | include, exclude, and the descend decision | `[]` |
| `wtdebounce` | coalescing, with the clock supplied | `[]` |
| `wtwatch` | **`WatchBackend[e]`**, the watch tree, the replay backend | `[e]`, `[]` |
| `wtinotify` | the Linux backend, and the four limits | `[io]`, `[fs]`, `[]` |
| `wtpoll` | the portable backend: snapshot, and compare | `[fs]`, `[]` |
| `wtloop` | `run_on_change`, and the same loop with the sleep taken out | `[e, io, time]`, `[e]`, `[]` |

## The load-bearing interface

```novo norun:pseudo
pub trait WatchBackend[e]
    fn watch(self, path: Str) -> Result<Int, WtFault> [e]
    fn unwatch(self, id: Int) -> Result<Unit, WtFault> [e]
    fn poll(self, timeout_seconds: Float) -> Result<WtPoll, WtFault> [e]
    fn close(self) -> Result<Unit, WtFault> [e]
```

**The effect parameter is what makes two backends worth having.**
inotify costs `[io]` — a watch is a file descriptor.  Polling costs
`[fs]` — it reads directories.  A recorded transcript costs **nothing**.
So `wtloop.step`, the whole turn of a watch loop, is written once and
charged what the caller's backend costs, and a test of a tool's watch
behaviour runs at `[]` with no filesystem and no clock.  An enum of
backends would charge every caller `[io, fs]`, the test included.

**`poll` never blocks past its deadline, and that is the contract.**  A
caller that also has a socket, a signal or a keypress to attend to keeps
its own scheduling; a caller with nothing else to do passes a long
deadline and gets blocking behaviour.  A backend with a blocking
`next_event` makes every tool that watches files *and* does something
else grow a thread.

## Three things a naive event model cannot say

**A rename is one event with two paths, and no backend sends it that
way.**  inotify delivers `MOVED_FROM` and `MOVED_TO` sharing a cookie,
possibly in separate reads, and possibly with no partner at all when the
file crossed the watched boundary.  A watcher that passed the raw halves
on makes every consumer pair them and every consumer pairs them wrongly —
a build tool sees a delete and an add and rebuilds from scratch where
nothing changed.  `WtRenamed(from, to)` is the paired event,
`WtMovedOut`/`WtMovedIn` are the honest halves, and `wtevent.pair` is the
state machine with a deadline.

**The queue overflows, and that is an event.**  inotify's queue is
bounded at 16384 by default and the kernel drops events and sends
`IN_Q_OVERFLOW` when it fills — which a `git checkout` of a large branch
does routinely.  Without a `WtOverflow` arm a watcher silently misses
changes, and the symptom is a build that is stale in a way nothing
explains.  The only correct response is a full rescan, and a consumer can
only make it if it is told: `wtevent.demands_rescan` is the predicate,
and `wtloop` acts on it.

**A directory event is not a file event.**  A recursive watcher must add
a watch when a directory is created, so `is_directory` is on every event
and `wtwatch.watch_new_directory` is published.

## What recursion costs

Recursion is not a kernel feature on Linux.  inotify watches **one**
directory per descriptor, so a recursive watch over a tree of ten
thousand directories is ten thousand watches — and
`fs.inotify.max_user_watches` is 8192 on a stock kernel.

That is why `wtfilter.may_descend` decides **before** a watch is placed
rather than after an event arrives: a watcher that filtered only events
would create every one of those watches and then throw the events away.
It is also why `wtinotify.limits_from_proc` and
`wtinotify.raise_limit_hint` exist — a tool that prints
`fs.inotify.max_user_watches=524288` beside its refusal has solved the
operator's problem, and one that prints "too many watches" has moved it.

And it leaves a real gap, named rather than hidden: between a `mkdir` and
the watch landing on the new directory, events inside it are lost.
`wtinotify.known_blind_spots` says so as data a tool can print, beside
the other two — a network filesystem's changes, and a bind mount's second
path.

## The one example that will work

```novo
use wtfilter
use wtinotify
use wtloop
use wtwatch

fn rebuild(events: [WtEvent]) -> Bool [io]
    println("${list.len(events)} changed")
    true

fn main() [io, time]
    match wtinotify.open()
        Err(e) => println(e.message())
        Ok(b)  =>
            let c = wtloop.config(wtfilter.filter(["**/*.nv"],
                                                  wtfilter.common_excludes()))
            match wtloop.run_on_change(b, ".", c, rebuild)
                Ok(turns) => println("${turns} turns")
                Err(e)    => println(e.message())
```

## Adding it, and checking it

```console
$ novo pkg add watch-nv
$ novo pkg build
$ novo test tests/wtdebounce_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: watch-nv.<module>.<fn>`.

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The rows, which are not the ones the plan guessed

The must-have plan's row says `[io, time]`.  What the package measures to
is finer, and the difference is worth reading:

- **Three modules declare nothing.**  The event model, the filter and
  the debouncer are arithmetic over values the caller already holds —
  which is what lets the rename pairing, the ignore decision and the
  coalescing window be tested in microseconds with no filesystem and no
  sleep anywhere.
- **`[io]` is inotify's**, because a watch is a descriptor.
- **`[fs]` is the polling backend's**, because it reads directories —
  and `wtinotify.limits_from_proc`'s, because it genuinely reads three
  files under `/proc`.
- **`[time]` appears in exactly one function**, `wtloop.now_unix`.
  Everything else takes the reading.
- **`[e, io, time]` on `run_on_change`** — and the `[io]` is the
  action's.  SPEC § 5.5 charges a function-typed parameter's row to
  whoever calls through it, so a loop that runs an `[io]` action is
  `[io]`; the compiler refused the first spelling of that signature,
  which is the effect system doing its job on an interface with no
  bodies in it.

## What the `novo` CLI would take

There is no `--watch` verb in `compiler/bin/novo.ml` today — which is why
the row exists.  Four places would take this package, and each needs a
different part of it:

| verb | what it needs |
| --- | --- |
| `novo build --watch` | `run_on_change` whole, with `wtfilter.common_excludes()` so `_novo/` and `.git/` are never watched |
| `novo test --watch` | the same, with the action running the suites the changed files belong to — which needs `WtEvent.path` relative, as it is |
| `novo doc --watch` | the same over `src/**` and the README |
| novols | `step` rather than `run_on_change`, because a language server owns its own event loop and cannot give it up — and `workspace/didChangeWatchedFiles` is exactly a `[WtEvent]` batch |

The two that would change most: `novo test --watch` is the reason
`WtEvent` carries both ends of a rename (a moved test file must not
re-run as a new one and a deleted one), and novols is the reason `step`
exists separately at `[e]` with the clock as an argument.

## What is deliberately absent

- **A thread.**  Nothing here spawns one.  `poll` takes a deadline and
  `wait_seconds` says how long to sleep; the caller's scheduling stays
  the caller's.
- **`kqueue`, `FSEvents`, `ReadDirectoryChangesW`.**  Three more
  backends behind the same trait, each worth a row of its own when a
  consumer on that platform is real.  `wtpoll` is what works everywhere
  meanwhile, and its cost is stated rather than hidden.
- **Rename detection in the polling backend.**  Nothing in a `stat`
  connects two paths, and a heuristic that paired a delete and a create
  by size would pair two unrelated empty files.
- **Content hashing.**  The polling backend compares `(size, mtime,
  mode)`, for the reason git's index does; `wtpoll.rehash_suspicious`
  names the paths that comparison cannot settle, which is the same shape
  as git's racy-clean rule.

## What widened, and what did not

- **`run_on_change` grew `[io]`**, from the action's own row — see
  above.
- **A wrapper returning another module's `?Float` does not compile**;
  the doc example for `wtdebounce.next_deadline` matches on the optional
  instead, and the defect is filed (it was found in git-nv the day
  before with `?Int`, so it is the representation and not the width).
- **No device claim.**  `Str`, `Result` and a filesystem are throughout.

## Reference

Rust's [`notify`](https://docs.rs/notify) and Python's
[`watchdog`](https://pythonhosted.org/watchdog/) are the reference
implementations; the inotify manual page is the specification for the
backend and for every limit named above.

## Licence

Apache-2.0.
