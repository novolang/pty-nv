# pty-nv

A **pseudoterminal** is a pair of character devices the kernel provides
so that a program can talk to another program as if it were a terminal.
One end, the **master**, is held by the program that is pretending to be
the terminal. The other, the **slave**, is handed to a child, which
cannot tell it from a serial line. It is specified by POSIX as
`posix_openpt`, `grantpt`, `unlockpt` and `ptsname`, and described in
Linux's [`pty(7)`](https://man7.org/linux/man-pages/man7/pty.7.html).
This package brings pseudoterminals to novo-lang: the pair, the child
running on the slave, the reads and writes, the signals, the exit
status, and the wait that a program with many pseudoterminals runs. It
is built on [termios-nv](https://novo-lang.org/packages/termios-nv) for
the terminal attributes and the window size.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

Anything written to the master arrives as the child's input; anything
the child writes comes out of the master. That much is a pipe. What
makes it a pseudoterminal is everything the slave carries besides the
bytes. It has terminal attributes, so a shell can turn echo off to read
a password. It has a window size, so a full-screen program knows how
large to draw. It has a **controlling terminal** and a **foreground
process group**, which is what makes Ctrl-C raise SIGINT and Ctrl-Z
raise SIGTSTP in the right processes. A shell on a pseudoterminal
behaves exactly as it does on a real terminal, because as far as it can
tell it is on one.

A **process group** is a set of processes the kernel can signal
together. A terminal has one of them in the foreground at a time: the
one the user is typing at. A child spawned on a pseudoterminal is a
**session leader**, so its own process group is the session's. As soon
as its shell runs a command, the foreground group becomes that
command's, not the shell's. `tcgetpgrp` on the master is how a program
reads which group that is.

Changing the window size is not a message to the child. Setting the
size on the master with the `TIOCSWINSZ` request makes the kernel raise
SIGWINCH in the foreground process group, and programs such as `vim`,
`less` and `htop` redraw at the new geometry by themselves. There is no
second notification to send.

A child ends in one of three ways, and they are different facts. It can
run and return a status. It can be killed by a signal, possibly writing
a core file. Or it can be *stopped* — by SIGTSTP from Ctrl-Z, or by
SIGTTIN because it read the terminal from the background — which is not
an ending at all but a child waiting to be continued. `waitpid` reports
all three, and `PtyExit` keeps them apart. The shell convention folds
them into one number, the exit code for a normal exit and 128 plus the
signal number for a killed one, and `ptymodel.shell_status` is that
folding.

Every function in this package that touches the machine performs input
or output and nothing else. No function reads a clock: the wait's
timeout and the termination grace period are arguments to a system call
that waits, not readings of a clock.

| Quantity | Value |
| --- | --- |
| Default spawn size | 24 rows by 80 columns, the fallback `screen` and `tmux` use |
| Descriptors one wait may cover | no limit but the operating system's |
| Signals named as constants | SIGHUP 1, SIGINT 2, SIGQUIT 3, SIGKILL 9, SIGTERM 15, SIGCONT 18, SIGTSTP 20, SIGWINCH 28 |
| Status for a child killed by signal `n`, by the shell convention | 128 + n |

## Install

```
novo pkg add pty-nv
```

## Example

A loop over several pseudoterminals: read whatever is ready, and reap
and drop a child that has gone.

```novo ignore
use ptychild
use ptyio
use ptymodel
use ptyopen
use ptypoll

fn pump(panes: [PtyMaster], children: [PtyChild], buf: [u8]) -> Result<Int, PtyError> [io]
    var set = ptypoll.poll_set(panes)
    var kids = children
    var scratch = buf
    while ptypoll.set_len(set) > 0
        // Wait up to 100 milliseconds for any of them to have bytes.
        let ready = ptypoll.wait_ready(set, 100)!
        var i = 0
        while i < ptypoll.set_len(set)
            if ptypoll.is_readable(ready, i)
                let m = ptypoll.master_at(set, i)!
                // Append up to 4096 bytes to the caller's buffer.
                let r = ptyio.read(m, scratch, 4096)!
                scratch = r.bytes
            if ptypoll.is_hungup(ready, i)
                // Collect the child's exit status, then stop waiting
                // on a descriptor that is ready forever.
                let _ = ptychild.try_wait(kids[i])!
                set = ptypoll.drop_at(set, i)!
            i = i + 1
    Ok(0)

fn main() [io]
    // Start the user's login shell on a fresh pair.
    match ptyopen.spawn(ptyopen.spawn_request(ptyopen.login_argv(ptyopen.default_shell())))
        Err(e) => println(e.message())
        Ok(s)  =>
            match pump([s.master], [s.child], [])
                Ok(_)  => println("done")
                Err(e) => println(e.message())
```

This program is marked `ignore` because it needs the bodies this
release does not have: running it reaches a `todo()` and panics.

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: pty-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ptymodel` | The values: the two ends, the pair, a spawn request, a child, how a child ended, what a read came back with, the errors, the signal numbers, and the shell's one-number convention. |
| `ptyopen` | Opening a pair, building a spawn request, finding the user's login shell, and starting a child on the slave. |
| `ptyio` | Reading and writing the master in chunks, non-blocking mode, the window size in both directions, the slave's terminal attributes, the raw descriptors, and closing either end. |
| `ptychild` | Waiting for a child with and without blocking, signalling one process or its whole group, hangup, the SIGTERM-then-SIGKILL escalation, and the foreground process group. |
| `ptypoll` | A set of masters to wait on plus one descriptor that is not a pseudoterminal, the wait itself, and the answer as indices into the set. |

`ptymodel`, the request builders in `ptyopen`, the accessors in
`ptyio`, `ptychild.pid_of`, `ptychild.pgid_of` and every function in
`ptypoll` except the wait perform no input or output. Everything else
does.

## How to choose an entry point

**`ptyopen.spawn` opens a pair and starts a child on it.** It is the
one call almost every program makes, and the four things a spawn has to
get right are all inside it (rule 1).

**`ptyopen.open_pair` and `ptyopen.spawn_on` are the two halves.** They
are for a caller that wants the master before there is a child — one
that registers the descriptor somewhere first, or that opens the pair
and decides what to run afterwards.

**`ptyio.write` reports how many bytes went, and `ptyio.write_all`
blocks until they all have.** A program running a loop over several
pseudoterminals wants the first: the second waits for as long as the
child stays away from its input, and while it waits nothing else is
served.

**`ptychild.signal_group` reaches the whole foreground group, and
`ptychild.signal` reaches one process.** The group is almost always
what a program means (rule 4).

**`ptypoll.wait_ready` waits on many at once.** `ptypoll.with_extra`
adds one descriptor that is not a pseudoterminal, so a keystroke on the
program's own terminal wakes the same wait.

## The rules a user needs

1. **A spawn has four obligations, and `ptyopen.spawn` meets all
   four.** The window size is set before the exec, so the child's first
   `TIOCGWINSZ` is already right and a full-screen program does not
   draw one frame at the wrong size. The slave is made the controlling
   terminal with `setsid` and `TIOCSCTTY`, without which the kernel
   raises no SIGINT on Ctrl-C and job control does not work. The signal
   dispositions are reset, because an ignored signal survives both fork
   and exec, so a child spawned by a program that ignores SIGHUP would
   inherit that and never die when the terminal closes. And the
   parent's copy of the slave is closed, because a slave the parent
   still holds never reports end of file when the child exits.
2. **A spawn takes an argument list, not a command string.**
   `PtySpawn.argv` is a `[Str]`. A caller who wants a shell to parse a
   line passes `["/bin/sh", "-c", line]` and has said so.
3. **The exit status is a value that outlives the descriptor.**
   `ptychild.try_wait` and `ptychild.wait` answer a `PtyExit`, which a
   caller keeps for as long as it likes. A program may close the master
   first and report what the child exited with afterwards.
4. **Signal the group, not the process.** The child on a pane is
   usually a shell, and the program the user is looking at is the
   shell's own child. A SIGHUP to the shell alone leaves that
   grandchild holding the terminal, and the user sees a pane that will
   not close. `ptychild.signal_group` targets the group,
   `ptychild.hangup` is the named form of the common case, and
   `ptychild.signal` is for a caller that genuinely means one process.
5. **`PtyStopped` is not an exit.** `ptymodel.exit_is_final` is the
   check. A program that tore a pane down on a stopped child would
   close a program the user meant to continue.
6. **A hung-up descriptor is permanently ready.** Every wait returns
   immediately, forever, so a loop that tests only for readable bytes,
   reads zero and goes round again spins at full speed on one core.
   `PtyReady` reports `hungup` apart from `readable`. The answer is:
   read what is left, reap the child, drop the index.
7. **Readiness is answered by index into the set that was passed in.**
   A program holding its children in a list in the same order reads the
   answer straight off, and drops a dead one from both lists at once.
   `ptypoll.drop_at` and `ptypoll.master_at` answer
   `PtyIndexOutOfRange` rather than panicking.
8. **A short write on a master is normal, and not an error.** The
   master's buffer fills whenever the child is not reading, which is
   every time a program pauses. `ptyio.write` answers how many bytes
   went; the caller writes the rest on its next turn.
9. **Resizing is the notification.** `ptyio.resize` sets the size with
   `TIOCSWINSZ`, and the kernel raises SIGWINCH in the child's
   foreground process group. A caller does not also send a signal.
10. **A read has two answers that are not errors.** `PtyRead.would_block`
    means nothing was available on a non-blocking master, and
    `PtyRead.at_eof` means the child's last copy of the slave closed,
    which is when to reap.
11. **A failed exec is an error, not an exit status.**
    `PtyExecFailed(program, errno)` comes back over a pipe that is
    closed on a successful exec. The alternative, a child that exits
    127, cannot be told from a program that ran and exited 127.
12. **`$TERM` is set by the spawner and not inherited.** The child is
    talking to whatever reads the master, not to the terminal that
    launched the parent, so an inherited value describes the wrong
    terminal. `ptyopen.with_term` sets it. A shell whose terminfo
    lookup fails re-echoes whole lines instead of moving the cursor.
13. **The window size and the terminal attributes are termios-nv's
    types.** A spawn request takes a `TioWinSize` and an optional
    `TioAttrs`. A program using both packages has one of each type
    rather than two that cannot meet.
14. **`ptychild.foreground_group` is not `ptychild.pgid_of`.** They
    differ as soon as the shell runs anything. The first is what the
    user is typing at, which is how a status line shows the running
    command rather than the word "bash", and how a program can tell
    whether closing a pane would interrupt something.
15. **`ptychild.terminate` is SIGTERM, then a grace period, then
    SIGKILL.** Killing at once loses a shell's history file and an
    editor's swap file. Waiting indefinitely hangs on a program that
    caught SIGTERM and ignored it.

## What is not included

- **A terminal emulator.** This package moves bytes; deciding what the
  child's output means is a parser's work.
  [ansi-nv](https://novo-lang.org/packages/ansi-nv) reads the escape
  sequences and [novo-vte](https://novo-lang.org/packages/novo-vte)
  keeps the grid of cells they change.
- **Your own terminal.** termios-nv opens the controlling terminal,
  puts it in raw mode and puts it back. This package depends on it
  rather than re-spelling its types.
- **Layout, a status bar and key bindings.** This package gives a
  multiplexer the wait; the rest is the multiplexer's.
- **An attach protocol.** Detaching, reattaching, capturing a pane and
  listing sessions are a client-server protocol over a socket. The
  standard library's `std.pty` carries one today, and a program that
  needs it keeps calling `std.pty` for that alongside this package for
  the pseudoterminals.
- **Pipes.** A program that wants a subprocess on a pipe rather than a
  terminal wants `std.proc`. The reason to pay for a pseudoterminal is
  that the child must believe it is on a terminal.
- **Windows.** ConPTY is a different design with a different lifecycle
  and no process groups, so half of this surface would fail at run
  time on it.
- **Running on a microcontroller.** Every function here is a system
  call.

## Related packages

- [termios-nv](https://novo-lang.org/packages/termios-nv) owns the
  terminal a program was started on: its attributes, its raw mode, its
  size and its restoration. A pseudoterminal slave is a terminal too,
  which is why the two packages share `TioWinSize` and `TioAttrs`.
- [ansi-nv](https://novo-lang.org/packages/ansi-nv) parses and builds
  escape sequences and owns no descriptor. It is what a program feeds
  the bytes read off a master.
- [novo-vte](https://novo-lang.org/packages/novo-vte) keeps a grid of
  cells with a cursor and a scrollback, which is what those bytes
  change.
- `std.pty` in the standard library is the runtime binding: about forty
  external declarations over the platform's `forkpty`, linked only into
  a program that uses them. It is where the system calls are declared,
  and a package cannot declare an external function without becoming a
  binding itself. This package is the surface a program is written
  against, over that boundary. The two coexist and it differs in six
  ways: an argument list rather than a command string split on
  whitespace; a failed exec as an error rather than a child exiting
  127; chunked reads and writes rather than one byte per call; a child
  and its exit status as values rather than a table keyed by the master
  descriptor, where closing first loses the status; signals to the
  process group rather than to the child alone; and a wait with no cap
  rather than one that stops at sixteen descriptors and truncates.
- `std.proc` in the standard library runs a subprocess on pipes.

## Reference implementations

Rust's `portable-pty` is the reference for the shape of the pair and
the child, and Python's `pty` module and `os.forkpty` for the smallest
version. tmux's `spawn.c` and `job.c` are where the four obligations in
rule 1 come from, and `script(1)` is the smallest correct program that
does all of this.

`portable-pty` puts each operating system behind a trait object; this
package is POSIX only, and a second platform would be a second package.
Its child has no notion of a process group, so it has no equivalent of
`signal_group`. Its reader blocks, with one thread per pseudoterminal
as the intended shape; here the wait is the interface.

## Tests

```bash
novo test --isolate tests/ptymodel_tests.nv   #  7 tests: the values and the three exits
novo test --isolate tests/ptyopen_tests.nv    # 10 tests: the spawn request
novo test --isolate tests/ptypoll_tests.nv    #  9 tests: the set and the readiness answer
novo test --isolate tests/ptyhost_tests.nv    # 14 tests: the pair, the child and the signals
```

The behaviour asserted comes from POSIX for `posix_openpt` and
`waitpid`, from Linux's `pty(7)`, `tty_ioctl(4)` and `credentials(7)`
for the size request and the process groups, and from tmux for the
order a spawn does its work in.

`ptymodel_tests.nv` checks that the shell convention gives 128 plus the
signal number for a killed child, that exiting 130 and being killed by
SIGINT stay different facts, and that a stopped child has not ended.
`ptypoll_tests.nv` builds a set of seventeen, checks that a hung-up
descriptor is not reported as readable, that dropping an index shortens
the set by one, and that an index past the end is refused with both
numbers in the message. `ptyopen_tests.nv` builds requests and checks
that the argument list is not re-split, that `$TERM` is set by the
spawner, and that a login shell's first argument carries a leading
dash. `ptyhost_tests.nv` is the half that opens a pair and starts a
child.

The tests compile today and fail at run, each on the
`not implemented: pty-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `ptymodel.PTY_SIGHUP`, `.PTY_SIGINT`, `.PTY_SIGQUIT`, `.PTY_SIGKILL`, `.PTY_SIGTERM`, `.PTY_SIGCONT`, `.PTY_SIGTSTP`, `.PTY_SIGWINCH` | yes (they are constants) |
| `ptymodel.shell_status`, `.exit_is_final`, `.default_spawn_size`, and `PtyError`'s `message` | no |
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

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
