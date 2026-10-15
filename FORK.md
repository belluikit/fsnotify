Bellui fork of fsnotify
=======================

Fork of [fsnotify/fsnotify], module path rewritten to
`github.com/belluikit/fsnotify` (the same convention as [belluikit/glfw] and
[belluikit/glfw-js] — imports are swapped at the call site, no `replace`
directive).

Everything outside `backend_windows.go` is upstream, untouched.

The fork started from upstream **v1.9.0** (tagged here as v1.9.1). Since then it
has merged upstream `main` at `20b1e15` (v1.10.1 plus 8 commits); upstream is
a git remote and later syncs are plain merges of its `main`.

[fsnotify/fsnotify]: https://github.com/fsnotify/fsnotify
[belluikit/glfw]: https://github.com/belluikit/glfw
[belluikit/glfw-js]: https://github.com/belluikit/glfw-js

Why the fork exists
-------------------

Two defects in the Windows `ReadDirectoryChangesW` backend, both still present
on upstream `main` (checked at v1.10.1). They are reported upstream; when they
land there this fork should rebase onto the fixed release and, ideally, go away.

Found while chasing belluikit/bellui#68 — intermittent Go **runtime fatals**
(`runtime.semawakeup` / `runtime.semasleep` / `netpoll failed`, all
`errno=6` = `ERROR_INVALID_HANDLE`) in bellui's `internal/repository` tests on
windows/arm64.

### 1. Double `CloseHandle` on a torn-down watch (the runtime fatal)

`readEvents` re-arms a watch unconditionally at the end of the
event-processing loop:

```go
if err := w.startRead(watch); err != nil {
```

but `watch` is recovered from the completion packet's `OVERLAPPED` pointer, and
that packet may have been queued *before* a `remWatch` tore the watch down. By
then `startRead` has already closed `watch.ino.handle` and dropped the watch
from `w.watches`, so the re-arm closes the same handle a second time.

Windows is free to reissue a closed handle's numeric value to the next caller.
In a Go program the most frequent caller is the runtime itself — `CreateEvent`
in `semacreate`, once per new M — so the stray close destroys a handle the
runtime believes it owns, and the next `WaitForSingleObject`/`SetEvent` on it
fails with `ERROR_INVALID_HANDLE`. The runtime treats that as unrecoverable and
aborts the process, with a stack that points nowhere near fsnotify.

**Fix:** drop completion packets whose watch is no longer registered, before the
`switch qErr`.

Measured on the windows/arm64 runner, 150 rounds of Add/Remove churn against a
directory under continuous write pressure (an instrumented build counting every
`CloseHandle`): **2404 double closes before, 0 after**. Driving bellui's own
watch tests, the runtime fatal reproduced within 3 iterations before the fix and
did not appear in 160 runs after it.

### 2. `sendError` swallows Close's handshake token (the hang)

`w.done` (capacity 1) carries the reply channel that is `Close`'s only handshake
with the reader goroutine. `sendEvent` receives it and puts it back;
`sendError` received it and dropped it:

```go
select {
case <-w.done:      // consumed, never returned
	return false
```

If an error is reported while a `Close` is in flight, the token is lost: the
reader falls through to `default`, parks in `GetQueuedCompletionStatus` having
already consumed the wakeup, and `Close` blocks forever on a reply that can
never come. Its watches, its reader thread and its handles leak.

This is the hang bellui's `internal/repository/file_watch.go` works around with
a bounded 5s wait, described there as "the completion packet was posted but
never woke the thread". The two defects are causally linked: the double close
above makes `CancelIo` fail, which calls `sendError`, which is what loses the
token — so #1 is a frequent *cause* of #2.

**Fix:** put the token back, exactly as `sendEvent` does.

Measured: 4 hung `Close`s per 200 watcher rounds before, **0 after**, with the
process handle count going from a slow upward drift to flat.

Versioning
----------

The fork carries its own tag line starting at **v1.9.1** — "upstream v1.9.0 plus
the two fixes above" — the same way [belluikit/glfw] runs its own versions
independent of GLFW's. The module path is ours, so this can never collide with
an upstream v1.9.1.

When rebasing onto a newer upstream release, keep the base version legible: a
fork of upstream v1.10.1 should be tagged v1.10.2, not v1.9.2.
