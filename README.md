# pty-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A pseudoterminal as a pair of values, and a child running on it.

Open a pair.  Start a program on the slave with an argv, an
environment, a `$TERM` and a size that is already right before the
first draw.  Read and write the master in chunks.  Resize, and the
kernel raises SIGWINCH in the child's process group for you.  Signal
that group rather than its leader.  Wait, and get an exit status that
is still there after the descriptor has closed.  And wait on many
ptys at once, with no cap on how many, which is what a multiplexer is.

It is not a terminal emulator — parsing what the child writes is
novo-vte's and drawing it is novoterm's.  It does not own your own
terminal; that is termios-nv, which this package depends on for the
attributes and the window size.

## Adding it, and checking it

```bash
novo pkg add pty-nv         # into your novo.toml
novo pkg build              # type- and effect-check the package
novo test --isolate tests/ptymodel_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: pty-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use ptychild
use ptyio
use ptymodel
use ptyopen
use ptypoll
use std.list

// A multiplexer's pump, with nothing in it that is not about ptys.
fn pump(panes: [PtyMaster], children: [PtyChild], buf: [u8]) -> Result<Int, PtyError> [io]
    var set = ptypoll.poll_set(panes)
    var kids = children
    var scratch = buf
    while ptypoll.set_len(set) > 0
        let ready = ptypoll.wait_ready(set, 100)!
        var i = 0
        while i < ptypoll.set_len(set)
            if ptypoll.is_readable(ready, i)
                let m = ptypoll.master_at(set, i)!
                let r = ptyio.read(m, scratch, 4096)!
                scratch = r.bytes
                // …feed `scratch` to the pane's parser…
            if ptypoll.is_hungup(ready, i)
                // The child is gone.  Reap it, then drop the pane —
                // a hung-up descriptor is permanently ready, and a
                // loop that did not drop it would spin.
                let _ = ptychild.try_wait(kids[i])!
                set = ptypoll.drop_at(set, i)!
            i = i + 1
    Ok(0)

fn main() [io]
    match ptyopen.spawn(ptyopen.spawn_request(ptyopen.login_argv(ptyopen.default_shell())))
        Err(e) => println(e.message())
        Ok(s)  =>
            match pump([s.master], [s.child], [])
                Ok(_)  => println("done")
                Err(e) => println(e.message())
```

## The layer, and why

`host`, and `[io]` is the whole of it.

| module | row | why |
| --- | --- | --- |
| `ptymodel` — every type, `shell_status`, `exit_is_final`, `default_spawn_size` | `[]` | the values, and arithmetic over them |
| `ptyopen.spawn_request` and every `with_*`, `login_argv` | `[]` | a request is a value until somebody spawns it |
| `ptyopen.open_pair`, `.spawn`, `.spawn_on`, `.close_session`, `.default_shell`, `.environment_of` | `[io]` | `posix_openpt`, `fork`, `execvp`, the environment |
| `ptyio.read`, `.write`, `.write_all`, `.set_nonblocking`, `.is_nonblocking`, `.resize`, `.size_of`, `.slave_attrs`, `.set_slave_attrs`, `.master_terminal`, `.slave_terminal`, `.close_master`, `.close_slave` | `[io]` | `read`, `write`, `fcntl`, two ioctls |
| `ptyio.master_fd`, `.slave_fd`, `.slave_name` | `[]` | accessors |
| `ptychild.try_wait`, `.wait`, `.is_running`, `.signal`, `.signal_group`, `.hangup`, `.terminate`, `.foreground_group` | `[io]` | `waitpid`, `killpg`, `tcgetpgrp` |
| `ptychild.pid_of`, `.pgid_of` | `[]` | accessors |
| `ptypoll.poll_set`, `.with_extra`, `.set_len`, `.drop_at`, `.master_at`, `.is_readable`, `.is_hungup`, `.anything_ready` | `[]` | the set and the answer are values |
| `ptypoll.wait_ready` | `[io]` | one `poll` |

**The plan's row for this package reads `[io, proc]`, and `proc` is not
an effect.**  The labels an effect clause accepts are the eleven in the
`host` budget — `novo pkg layers` prints them — and `docs/publishing.md`
§ Design says why processes are among them rather than beside them:
"`[io]` covers pipes, processes and the environment and not only the
console, so dotenv, git plumbing, keyrings and pseudo-terminals all
need it".  `std.proc`'s own rows are `[io]` for the same reason.  So
forking, exec'ing, signalling and reaping are all `[io]` here, and
nothing was widened to say so.

**No `[time]` either**, and it would have been easy to reach for: the
poll has a timeout and `terminate` has a grace period.  Both are
arguments to a syscall that waits, not readings of a clock, and a
package that declared `[time]` for them would be telling an embedded
consumer it needs a clock it does not.

## The load-bearing interface

**The child is a value, and its exit status outlives the descriptor.**

The alternative — and it is what this package is replacing — is to key
everything off the master descriptor through a process-global table:
`kill(fd, sig)`, `child_exited(fd)`, `child_status(fd)`.  It reads
well and it has one rule that cannot be expressed in the types:

> Read the status BETWEEN the `child_exited` that returned 1 and the
> `close` for the same fd — close releases the slot the status lives
> in.

A caller who closes first has lost the status permanently, and nothing
says so: the answer is `0`, indistinguishable from a child that
succeeded.  A pump whose teardown path closes descriptors before it
reports is a pump that reports success for every crash, and that is a
bug found by a user rather than by a test.

So `PtyChild` carries the process id and the process group id;
`try_wait` and `wait` answer a `PtyExit`; and a `PtyExit` is an
ordinary value that a caller keeps for as long as it likes, in
whichever order its own code reads best.

Three things follow from it, and each of them was impossible in the
other shape:

- **`PtyExit` has three cases and not an integer.**  A program that
  exited 130 chose to; a program killed by SIGINT did not.
  `shell_status` folds them into the conventional one number, at the
  end, for a caller that has to answer with one.
- **`PtyStopped` is not an exit.**  A child stopped by SIGTSTP, or by
  SIGTTIN because it read the terminal from the background, is waiting
  to be continued.  A pump that folded it into "exited" closes a pane
  whose program the user meant to come back to.
- **`pgid` is carried separately from `pid`**, which is what makes
  `signal_group` expressible — see below.

## The signal goes to the group

A pane's child is a shell.  The program the user is looking at is the
shell's child: `less`, `vim`, a build.  A SIGHUP to the shell alone
leaves that grandchild holding the terminal, and what the user sees is
a pane that will not close.

`ptychild.signal_group` targets the foreground process group — the one
Ctrl-C reaches, which the kernel already maintains because a pty child
is a session leader.  `ptychild.signal` is there for the caller who
genuinely means one process, and `hangup` is the named form of the one
almost everybody wants.

`ptychild.terminate` is the escalation everybody writes by hand:
SIGTERM to the group, a grace period, then SIGKILL to whatever is
still there.  Written by hand it is usually wrong in one of two
directions — killing at once, which loses a shell's history file and
an editor's swap file, or waiting forever for a program that caught
SIGTERM and ignored it.

`ptychild.foreground_group` is the read side, and it is more useful
than it looks: it is different from the child's own group as soon as
the shell runs anything, which is how a status bar shows the running
command rather than the word "bash", and how a pane can tell whether
closing it would interrupt something.

## Hangup is not readable

A descriptor whose child has exited is **permanently ready**.  Every
poll returns immediately, forever.  A loop that tested only "readable",
read zero bytes and went round again spins at a hundred per cent of a
core — the single most common bug in a program built on a poll loop,
and one that looks like a performance problem rather than a logic
error.

So `PtyReady` reports `hungup` separately from `readable`, and the
pump's answer is: read what is left, reap the child, drop the index.

The set has **no ceiling**, which is the other half of the same
concern.  A poll that caps at sixteen descriptors and silently
truncates is a multiplexer whose seventeenth pane never updates — and
nothing reports it, because a truncated poll is indistinguishable from
a quiet pane.  Seventeen panes is a person with a large monitor.

And `with_extra` exists because everything that is not a pty has to
wake the same loop: the host terminal a multiplexer is displayed on, a
control socket, a window system's connection.  A loop polling only its
ptys answers a keystroke on the next pty's timeout, and a user feels
every millisecond of that.

## What `std.pty` keeps, and what this replaces

`std.pty` is the standard library's `@ffi`-bound pty surface — about
forty externs, conditionally linked, Linux-only through `forkpty`.  It
is what novoterm and novomux are built on today.  This package **does
not replace it and does not wrap it**; the two coexist, with a clear
line between them.

**What `std.pty` keeps, and should:** it is the runtime boundary.  The
externs, the conditional link (`bin/novo_rt_pty.c` is linked only for a
program whose source mentions `novo_pty_`), and the platform's
`forkpty` are all exactly where they belong — in the standard library,
next to the C that implements them.  A package cannot declare an
extern at all without becoming a `sys` package on the bindings shelf,
and a pty binding is not a C library a consumer chose.

**What this package is instead:** the surface a program should be
written against, over that boundary.  Six things change, and each of
them is a bug somebody has already hit:

| `std.pty` today | pty-nv |
| --- | --- |
| `spawn_shell(cmd: Str, …)` splits on whitespace with no quoting | `argv: [Str]`; a caller who wants shell parsing passes `["/bin/sh", "-c", line]` |
| a failed exec is `_exit(127)` with a diagnostic on stderr | `PtyExecFailed(program, errno)` over a close-on-exec pipe |
| `read_byte` / `write_byte`, one byte per call | `read` and `write` over a chunk, into the caller's buffer |
| `(fd → pid)` in a process-global table; the status dies with the close | `PtyChild` and `PtyExit` are values |
| `kill_signal(fd, sig)` reaches the child alone | `signal_group`, `hangup`, `terminate` |
| `poll_many` caps at 16 fds and truncates silently | the set is a list, indexed answers, hangup reported apart |

**And what this package deliberately leaves behind.**  Rather more
than half of `std.pty`'s surface is not about pseudoterminals at all:
`novo_unix_*` (an AF_UNIX transport), the mirror clients, the
`\x1cNMUX1` resize frame, `\x1cKILL`, `\x1cCAPT`, `\x1cLSES`,
`\x1cSWSE`, the reattach counter, the headless flag, the primary-size
negotiation.  Those are **novomux's own client-server protocol**,
riding the pty module because the conditional link already pulled it
in — its own comment says as much: *"Riding the `std.pty`
conditional-link cascade… Hoist into `std.unix.*` when a second
non-pty caller shows up."*

That second caller has now shown up, in the shape of this package
declining to carry it.  **Two missing rows**, and neither is on the
grid:

- **`unixsock-nv`** (`host`, `networking`): AF_UNIX stream sockets —
  listen, connect, accept, non-blocking read and write, and passing a
  descriptor over one.  Wanted by a multiplexer's daemon, by an
  editor's language-server transport, and by any program with a
  control socket.
- **`muxproto-nv`** (`core`, `terminal`): the attach protocol — the
  size prologue, detach, kill, capture-pane, list-sessions, the
  read-only mirror. Sans-IO, a codec, and the thing a second
  multiplexer implementation would need in order to speak to the
  first.

Until they exist, a program that needs that machinery keeps calling
`std.pty` for it, alongside this package for the ptys.  Nothing here
prevents that: they are different functions over the same descriptors.

## Four things a spawn has to get right

In the order the child notices them, and none of them is a caller's to
remember, because they are all inside `ptyopen.spawn`:

1. **The size is baked in before the exec**, so the child's first
   `TIOCGWINSZ` is already right.  A full-screen program that had to
   be resized afterwards draws one visible frame wrong.
2. **The slave becomes the controlling terminal** — a `setsid` and a
   `TIOCSCTTY`.  Without it the kernel raises no SIGINT on Ctrl-C and
   job control does not work; the symptom a user reports is "Ctrl-C
   does nothing in this pane".
3. **The signal dispositions are reset.**  `SIG_IGN` survives both
   fork *and* exec, so a child spawned by a daemon that ignores SIGHUP
   inherits "ignore SIGHUP" — and then a kill-pane delivers the signal
   successfully, the shell discards it, and the pane never dies.  It
   reproduces only through the daemon path, which is how a whole test
   suite can miss it.
4. **The parent's copy of the slave is closed.**  A slave the parent
   still holds never reports EOF when the child exits, and a pump
   waiting for that EOF waits forever.

## What this does not do, on purpose

- **It does not parse the child's output.**  That is novo-vte, and
  ansi-nv under it.
- **It does not own your terminal.**  termios-nv does; this package
  depends on it rather than re-spelling `TioWinSize` and `TioAttrs`,
  because a program that used both and had two `WinSize` types could
  not compile at all.
- **It does not multiplex.**  It gives a multiplexer the poll; the
  layout, the status bar and the key bindings are novomux's.
- **It does not speak an attach protocol.**  That is the
  `muxproto-nv` row above.
- **It is not a general subprocess API.**  A program that wants a pipe
  rather than a terminal wants `std.proc`; the whole reason to pay for
  a pty is that the child must believe it is on a terminal.
- **It is POSIX.**  Windows has ConPTY, which is a different design
  with a different lifecycle, and it is a separate package rather than
  a branch inside this one.
- **No device claim.**  The package is `host`.

## The reference implementation

Rust's `portable-pty` for the shape of the pair and the child, and
Python's `pty` and `os.forkpty` for the minimum.  `tmux`'s `spawn.c`
and `job.c` are where the four spawn rules come from, and `script(1)`
is the smallest correct program that does all of this.

Three things change in the port.  `portable-pty` hides the platform
behind a trait object with a `PtySystem` per operating system; here
there is one implementation and a second platform would be a second
package, because a ConPTY child does not have a process group and half
this surface would be a runtime error on it.  Its `Child` is a trait
with `wait` and `kill` and no notion of a process group at all, so
`signal_group` has no equivalent — which is exactly the gap that makes
a pane refuse to close.  And its reader is a `Read` implementation that
blocks, with a separate thread per pane as the intended shape; here the
poll is the interface, because a multiplexer with one thread and a
poll is simpler than one with a thread per pane and a channel.

## Status

| item | implemented |
| --- | --- |
| `ptymodel` — `PtyMaster`, `PtySlave`, `PtyPair`, `PtyEnvVar`, `PtySpawn`, `PtyChild`, `PtySession`, `PtyExit`, `PtyRead`, `PtyError` | types only |
| `ptymodel.PTY_SIGHUP`, `.PTY_SIGINT`, `.PTY_SIGQUIT`, `.PTY_SIGKILL`, `.PTY_SIGTERM`, `.PTY_SIGCONT`, `.PTY_SIGTSTP`, `.PTY_SIGWINCH` | yes — they are constants |
| `ptymodel.shell_status`, `.exit_is_final`, `.default_spawn_size`, the `message` impl | no |
| `ptyopen.open_pair`, `.spawn`, `.spawn_on`, `.close_session` | no |
| `ptyopen.spawn_request`, `.with_size`, `.with_env`, `.with_term`, `.with_cwd`, `.with_attrs`, `.without_parent_env` | no |
| `ptyopen.default_shell`, `.login_argv`, `.environment_of` | no |
| `ptyio.read`, `.write`, `.write_all`, `.set_nonblocking`, `.is_nonblocking` | no |
| `ptyio.resize`, `.size_of`, `.slave_attrs`, `.set_slave_attrs` | no |
| `ptyio.master_terminal`, `.slave_terminal`, `.master_fd`, `.slave_fd`, `.slave_name`, `.close_master`, `.close_slave` | no |
| `ptychild.try_wait`, `.wait`, `.is_running`, `.signal`, `.signal_group`, `.hangup`, `.terminate` | no |
| `ptychild.pid_of`, `.pgid_of`, `.foreground_group` | no |
| `ptypoll.poll_set`, `.with_extra`, `.set_len`, `.drop_at`, `.master_at`, `.wait_ready` | no |
| `ptypoll.is_readable`, `.is_hungup`, `.anything_ready` | no |

One dependency: termios-nv, `host`, for the terminal the slave is and
the size a spawn starts at.
