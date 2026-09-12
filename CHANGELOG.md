# Changelog

All notable changes to pty-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `ptymodel` — the values: `PtyMaster`, `PtySlave`, `PtyPair`,
  `PtySpawn` with an argv rather than a command string, `PtyChild`
  carrying a pid and a process group, `PtyExit` in three cases, and
  `PtyRead` with the two answers that are not errors.  The signals a
  terminal program actually sends, named.
- `ptyopen` — `open_pair`, the request builders, `default_shell`,
  `login_argv`, `environment_of`, and `spawn` with the four things a
  spawn has to get right inside it.
- `ptyio` — chunked reads into the caller's buffer, writes that report
  a short write rather than failing on one, `resize` as the
  notification it already is, the slave's attributes as termios-nv's
  value, and both ends as `TioFd`.
- `ptychild` — `try_wait` and `wait` answering a value, `signal_group`
  and `hangup` targeting the group rather than its leader, `terminate`
  as the SIGTERM-then-SIGKILL escalation, and `foreground_group`.
- `ptypoll` — a caller-owned set with no ceiling, readiness by index,
  and hangup reported apart from readable.

### Known

- **The child is a value and its exit status outlives the
  descriptor.**  The shape being replaced keys the status off the
  master descriptor in a process-global table, where a caller who
  closes before reading has lost it for good and the answer is 0 —
  indistinguishable from success.
- **`PtyStopped` is not an exit.**  A child stopped by SIGTSTP or
  SIGTTIN is waiting to be continued, and a pump that folded it into
  "exited" closes a pane the user meant to come back to.
- **Hangup is not readable.**  A hung-up descriptor is permanently
  ready, so a loop that does not distinguish them spins at a hundred
  per cent of a core.
- **The poll set has no ceiling.**  A cap that truncates silently is a
  multiplexer whose seventeenth pane never updates.
- **`[io, proc]` in the plan is not an effect row.**  `proc` is not one
  of the eleven labels; processes are `[io]`, as `std.proc` already
  declares.  Nothing widened.
- **No `[time]`.**  The poll's timeout and `terminate`'s grace period
  are arguments to a syscall that waits, not readings of a clock.
- **`std.pty` coexists rather than being wrapped or replaced.**  It is
  the runtime boundary and belongs there; this is the surface a program
  should be written against, over it.
- **Two missing rows found**, both currently living inside `std.pty`
  because a conditional link already pulled them in:
  `unixsock-nv` (`host`, `networking`) for AF_UNIX stream sockets, and
  `muxproto-nv` (`core`, `terminal`) for the attach protocol — the size
  prologue, detach, kill, capture-pane, list-sessions and the read-only
  mirror.
- **POSIX only.**  Windows ConPTY is a different lifecycle with no
  process groups, and a separate package.
- **One dependency**: termios-nv, at the same layer.
- **No device claim.**  The package is `host`.
