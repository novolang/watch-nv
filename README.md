# watch-nv

A filesystem watcher tells a program that a file changed, so the program can act
without being asked. This package is a watcher for novo-lang: an event model, two
backends behind one interface, and the run-on-change loop that a `--watch` mode
is. On Linux the backend is
[inotify(7)](https://man7.org/linux/man-pages/man7/inotify.7.html), the kernel
interface for watching directories. Everywhere else it compares directory
listings taken at intervals. The reference implementations are Rust's
[notify](https://docs.rs/notify) and Python's
[watchdog](https://python-watchdog.readthedocs.io/).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **watch** is a request to be told when something under one directory changes.
An **event** says what happened to one path: it was created, its contents
changed, its metadata changed, it was removed, or it was renamed.

A **rename** is one thing that happened, and it has two paths. No backend
reports it that way. inotify sends a `MOVED_FROM` event and a `MOVED_TO` event
that share a number called a **cookie**, and the two may arrive in separate
reads. When a file moves out of the watched tree, or in from outside it, only
one half ever arrives. **Pairing** is putting the two halves back together, with
a deadline for the half whose partner never comes.

The kernel holds events in a **queue** until a program reads them. The queue is
bounded. When it fills the kernel drops events and reports an **overflow**. A
`git checkout` of a large branch does this routinely. The only correct response
is to rescan the tree, because the watcher cannot know what it missed.

**Recursion is not a kernel feature on Linux.** inotify watches one directory
per file descriptor. A recursive watch over a tree of ten thousand directories
is ten thousand watches, and the per-user limit is 8192 on a stock kernel.
Deciding not to descend into a directory has to happen before the watch is
placed, not after its events arrive.

One save from an editor produces four to six events. The editor writes a
temporary file, renames it over the original and removes a lock. **Debouncing**
turns that burst into the one change that happened, by waiting until nothing has
touched the path for a while.

Three of this package's seven modules perform no input or output. The event
model, the filter and the debouncer take values the caller already holds. The
clock is an argument, not a reading, so their behaviour is testable without
waiting.

## Install

```
novo pkg add watch-nv
```

## Example

```novo
use std.list
use wtfilter
use wtinotify
use wtloop

// What to do when something changed. It answers whether to keep watching.
fn rebuild(events: [WtEvent]) -> Bool [io]
    println("${list.len(events)} changed")
    true

fn main() [io, time]
    // Open an inotify watcher. Its watches are file descriptors.
    match wtinotify.open()
        Err(e) => println(e.message())
        Ok(b)  =>
            // Watch .nv files, and never descend into a build or .git directory.
            let f = wtfilter.filter(["**/*.nv"], wtfilter.common_excludes())

            // Poll, pair renames, filter, debounce, run the action, repeat.
            match wtloop.run_on_change(b, ".", wtloop.config(f), rebuild)
                Ok(turns) => println("${turns} turns")
                Err(e)    => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `wtevent` | The event model, the raw events a backend reports, and the state machine that pairs a rename's two halves. Performs no input or output. |
| `wtfilter` | Include and exclude patterns, whether a path is interesting, and whether a directory is worth watching at all. Performs no input or output. |
| `wtdebounce` | Coalescing a burst of events into one, with the clock supplied by the caller. Performs no input or output. |
| `wtwatch` | The `WatchBackend` interface both backends implement, the recursive watch tree, and a backend that replays a recorded transcript. |
| `wtinotify` | The Linux backend, the three kernel limits read from `/proc`, the `sysctl` line that raises one, and what the backend cannot see. |
| `wtpoll` | The portable backend: a directory snapshot as a value, and the comparison of two snapshots. |
| `wtloop` | `run_on_change`, the whole of a `--watch` mode, and `step`, the same turn with the sleep taken out. |

## How to choose an entry point

**A tool that wants a `--watch` flag calls `wtloop.run_on_change`.** It polls
the backend, pairs renames, filters, debounces, runs the action, adds watches
for directories that appear, and rescans on an overflow.

**A program that already owns an event loop calls `wtloop.step`.** A language
server or a program also serving HTTP cannot give up its own scheduling. `step`
takes the clock reading as an argument and runs no action, so it costs only what
the backend costs.

**On Linux, open the backend with `wtinotify.open`.** Elsewhere, and on a
network filesystem, build one with `wtpoll.poller`. `wtwatch.suggests_polling`
answers whether a fault means you should switch.

**A test of a tool's watch behaviour uses `wtwatch.replay_backend`.** It replays
a recorded transcript and performs no input or output, so the same loop runs
with no filesystem, no clock and no sleep.

The backend interface carries an effect parameter, so the loop is written once
and costs what the caller's backend costs. inotify costs `[io]`, because a watch
is a file descriptor. Polling costs `[fs]`, because it reads directories. The
replay backend costs nothing.

## The rules a user needs

1. **`poll` never blocks past its deadline.** It takes a timeout and answers
   what it has when the timeout passes. A caller with nothing else to do passes
   a long deadline and gets blocking behaviour.
2. **An overflow means rescan.** `WtOverflow` and `WtWatchLost` are the two
   events after which the watcher's view is incomplete. `wtevent.demands_rescan`
   is the predicate, and `wtloop` acts on it. inotify(7), "Limitations and
   caveats".
3. **inotify cannot say how many events it dropped.** `WtOverflow(dropped)`
   carries `0` when the backend cannot say, which on inotify is always.
4. **A rename is one event with two paths.** `WtRenamed(from, to)` is the paired
   event. `WtMovedOut` and `WtMovedIn` are the honest halves for a file that
   crossed the watched boundary. `wtevent.paths_of` answers both ends, and a
   consumer that read `WtEvent.path` alone invalidates the wrong one.
5. **Pairing needs a deadline and needs to be aged.** A `MOVED_FROM` whose
   partner never arrives has to become a `WtMovedOut` eventually. The default
   window is half a second. `wtevent.expire` releases the pending halves, and a
   quiet tree produces no other event to carry them out.
6. **Every event says whether the path is a directory.** A recursive watcher
   must place a watch when a directory is created.
   `wtwatch.watch_new_directory` is that call.
7. **Between a `mkdir` and the watch landing on the new directory, events inside
   it are lost.** That window is real and cannot be closed from user space.
   `wtinotify.known_blind_spots` lists it, along with a network filesystem's
   changes and a bind mount's second path, as text a tool can print.
8. **Decide not to descend before the watch is placed.** `wtfilter.may_descend`
   answers whether a directory can contain anything the patterns match. A
   watcher that filtered only events would place a watch on every directory and
   then throw the events away.
9. **Exclude beats include.** A filter with `include = ["**/*.nv"]` and
   `exclude = ["_novo/**"]` means the obvious thing, whichever was written
   first.
10. **An empty include list means everything.** The absence of a flag must not
    mean "watch nothing".
11. **Patterns match paths relative to the watch root**, and `WtEvent.path` is
    relative for the same reason. An absolute path would make an event's meaning
    depend on where the process was started.
12. **Debouncing needs two windows, not one.** `quiet_seconds` emits once the
    burst stops. `max_seconds` emits anyway once the first event of the burst is
    that old. Without the cap, a `git checkout` that touches files for thirty
    seconds never reaches quiet and the tool never runs.
13. **Coalescing keeps the more informative event.** A create followed by a
    modify stays a create. Anything followed by a remove becomes a remove.
    `wtdebounce.fold` is that rule.
14. **`wtdebounce.next_deadline` turns a spin into a sleep.** It answers when
    the debouncer next has something to emit, or nothing when it is empty.
15. **The polling backend compares size, mtime and mode.** It does not hash.
    `wtpoll.diff_snapshots` performs no input or output, so two snapshots built
    from literals are a complete test.
16. **An mtime that went backwards is a change.** A checkout, a restore from a
    backup and `rsync --times` all write files whose mtime is older than what
    was there. The comparison is inequality, not "newer".
17. **A file rewritten within the mtime's resolution to the same length is
    invisible to the polling backend.** `wtpoll.rehash_suspicious` names the
    paths that comparison cannot settle, for a caller that cannot accept that.
18. **Count your watches before you place them.** `wtinotify.fits` compares a
    wanted count against the kernel's limit, and `wtinotify.raise_limit_hint`
    answers the `sysctl` line that raises it. A tool that walks until the kernel
    refuses has already placed eight thousand watches it must now unwind.
19. **`wtinotify.limits_from_proc` reads three files under `/proc`.** It is the
    one function in that module that touches the filesystem rather than a
    descriptor.
20. **The action is a named function, not a lambda.** It takes the batch of
    events and answers whether to keep going, so a tool stops on a signal, a
    fatal error or a keypress without this package owning the decision.
21. **`run_on_change` costs the action's effects as well as its own.** A loop
    that runs an action performing output performs output. Its declared effects
    are the backend's, the clock, and whatever the action declares.
22. **Nothing here spawns a thread.** `poll` takes a deadline and
    `wtloop.wait_seconds` says how long to sleep. The caller's scheduling stays
    the caller's.

## What is not included

- **`kqueue`, `FSEvents` and `ReadDirectoryChangesW`.** The BSD, macOS and
  Windows kernel interfaces. Each is a backend behind the same interface when a
  consumer on that platform is real. `wtpoll` works everywhere meanwhile.
- **Rename detection in the polling backend.** Nothing in a `stat` result
  connects two paths, and a heuristic pairing a delete and a create by size
  would pair two unrelated empty files.
- **Content hashing.** Hashing a tree per interval would read every byte of a
  repository every second.
- **A thread, and an internal timer.** Both would take over the caller's
  scheduling and make the package untestable without waiting.
- **The `.gitignore` grammar.** The filter's patterns are globs.
  [git-core-nv](https://novo-lang.org/packages/git-core-nv)'s `gitignore` module
  is the package for git's own rules, which are not globs.
- **Running on a microcontroller.** `Str`, `Result` and a filesystem are used
  throughout. This package makes no claim.

## Related packages

- [glob-nv](https://novo-lang.org/packages/glob-nv) supplies the patterns. Its
  `may_descend` is the call that decides a whole subtree is not worth watching,
  which is the difference between twenty watches and two hundred thousand.
- [git-core-nv](https://novo-lang.org/packages/git-core-nv) has the `.gitignore`
  grammar, if you need git's exclusion rules rather than globs. Its index module
  compares `stat` data for the same reason `wtpoll` does.
- [progress-nv](https://novo-lang.org/packages/progress-nv) is what a long
  `--watch` action reports through.
- `std.fs` in the standard library reads and lists directories. It has no notion
  of being told when one changes.
- `std.time` in the standard library is where a clock reading comes from.
  `wtloop.now_unix` is the only function in this package that takes one.

## Tests

```bash
novo test tests                          # every suite
novo test tests/wtevent_tests.nv         # the rename pairing, and the overflow
novo test tests/wtfilter_tests.nv        # the descend decision
novo test tests/wtdebounce_tests.nv      # the timing, with no sleep anywhere
novo test tests/wtwatch_tests.nv         # the backend interface
novo test tests/wtpoll_tests.nv          # two snapshots, compared
novo test tests/wtloop_tests.nv          # one turn of the loop
novo test tests/wtinotify_tests.nv       # the limits and the sysctl line
```

| Suite | Tests |
| --- | --- |
| `wtfilter_tests.nv` | 13 |
| `wtevent_tests.nv` | 11 |
| `wtdebounce_tests.nv` | 10 |
| `wtpoll_tests.nv` | 10 |
| `wtwatch_tests.nv` | 10 |
| `wtloop_tests.nv` | 9 |
| `wtinotify_tests.nv` | 6 |

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
watch-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests are
the specification the implementation will have to satisfy.

The limits, the event kinds and the overflow behaviour come from the inotify(7)
manual page. The event model and the rename-pairing cases follow Rust's `notify`
and Python's `watchdog`, the two implementations that meet these problems in
production.

No suite sleeps and no suite touches a directory. The clock is an argument
everywhere, so a burst of events at 0.0, 0.01 and 0.02 that should emit at 0.32
is three calls and an assertion. `wtwatch_tests.nv` and `wtloop_tests.nv` drive
a whole watch session over the replay backend, and the compiler checking that
those functions declare no effects is itself the claim this package makes. The
inotify backend is not exercised, because a suite that opened a real watcher
would test the kernel.

## Implementation status

| Item | Implemented |
| --- | --- |
| `wtevent`'s six types, from `WtEventKind` to `WtRawKind` | declared |
| `wtevent`'s 12 functions, from `pairing` to `kind_name` | no |
| `wtfilter.WtFilter` | declared |
| `wtfilter`'s 11 functions, from `everything` to `is_editor_artifact` | no |
| `wtdebounce.WtDebounce`, `.WtPending` | declared |
| `wtdebounce`'s 10 functions, from `debounce` to `bypasses` | no |
| `wtwatch.WtFault` and `impl Error for WtFault` | declared |
| `wtwatch.WatchBackend`, the interface both backends implement | declared |
| `wtwatch.WtWatch`, `.WtPoll`, `.WtReplayBackend`, `.WtWatchTree` | declared |
| `wtwatch`'s eight functions, from `replay_backend` to `suggests_polling` | no |
| `wtwatch`'s `impl WatchBackend for WtReplayBackend`: four methods | no |
| `wtinotify.WtInotify`, `.WtInotifyLimits` | declared |
| `wtinotify`'s 10 functions, from `open` to `known_blind_spots` | no |
| `wtpoll.WtStatEntry`, `.WtSnapshot`, `.WtPoller` | declared |
| `wtpoll`'s 11 functions, from `poller` to `minimum_interval` | no |
| `wtloop.WtLoopConfig`, `.WtTurn`, `.WtLoopState` | declared |
| `wtloop`'s 13 functions, from `default_config` to `describe_turn` | no |

75 public functions, one interface with four methods, every body a `todo()`.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
