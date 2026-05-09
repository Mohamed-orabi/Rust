# Five Rust Patterns — Complete Runnable Examples

This is the companion to `rust-systems-patterns.md`. Where that file explains the *concepts*, this file gives you **complete programs you can actually build and run**.

Each example is a self-contained Cargo project. You can either:

- **Option A:** Create five separate projects (`cargo new example1`, etc.) and copy each example into the corresponding project.
- **Option B:** Use one project and swap `src/main.rs` between examples.

I recommend Option A — having all five built side by side lets you go back and re-run any of them.

All examples target Linux (which is what we're using for the project). They will work on WSL2 with no modifications.

---

## Setup (do this once)

If you haven't already:

```bash
mkdir -p ~/projects/rust-patterns
cd ~/projects/rust-patterns
```

Then for each example below, create a new project:

```bash
cargo new --bin example1-safe-wrappers
cd example1-safe-wrappers
# Replace Cargo.toml and src/main.rs with the code below.
cargo run
```

---

## Example 1 — Safe Wrappers Around Unsafe Primitives

**What this program does:** It exposes a safe API for several common Unix syscalls (`getpid`, `getppid`, `getuid`, `kill`, `unlink`), then calls them from `main` to demonstrate the wrappers in action. It also intentionally triggers an error (sending a signal to a non-existent PID) to show how `errno` is captured into `io::Error`.

### `Cargo.toml`

```toml
[package]
name = "example1-safe-wrappers"
version = "0.1.0"
edition = "2021"

[dependencies]
libc = "0.2"
```

### `src/main.rs`

```rust
//! Example 1: Safe wrappers around unsafe primitives.
//!
//! Demonstrates how to wrap libc syscalls in safe Rust functions with
//! proper error handling and SAFETY comments.

use std::ffi::CString;
use std::io;

// ----- Safe wrapper: getpid -----

/// Returns the current process's PID.
///
/// This is a safe wrapper around getpid(2). The syscall has no preconditions
/// and never fails, so the wrapper is unconditionally sound.
fn getpid() -> i32 {
    // SAFETY: getpid never fails and has no side effects beyond returning
    // the calling process's PID.
    unsafe { libc::getpid() }
}

// ----- Safe wrapper: getppid -----

/// Returns the parent process's PID.
fn getppid() -> i32 {
    // SAFETY: getppid never fails.
    unsafe { libc::getppid() }
}

// ----- Safe wrapper: getuid -----

/// Returns the real user ID of the calling process.
fn getuid() -> u32 {
    // SAFETY: getuid never fails.
    unsafe { libc::getuid() }
}

// ----- Safe wrapper: kill (can fail) -----

/// Sends a signal to a process.
///
/// Use signal `0` to test whether a process exists without actually
/// signaling it.
fn kill(pid: i32, signal: i32) -> io::Result<()> {
    // SAFETY: kill(2) is sound for any (pid, signal) pair. Invalid arguments
    // produce error returns, not undefined behavior.
    let ret = unsafe { libc::kill(pid, signal) };
    if ret == -1 {
        Err(io::Error::last_os_error())
    } else {
        Ok(())
    }
}

// ----- Safe wrapper: unlink (takes a string argument) -----

/// Removes a file from the filesystem.
fn unlink(path: &str) -> io::Result<()> {
    // CString conversion can fail if the path contains an interior NUL byte,
    // which is invalid for any Unix path anyway.
    let c_path = CString::new(path).map_err(|_| {
        io::Error::new(io::ErrorKind::InvalidInput, "path contains NUL byte")
    })?;

    // SAFETY: unlink takes a NUL-terminated C string. CString guarantees
    // NUL termination; the pointer is valid for the duration of the call.
    let ret = unsafe { libc::unlink(c_path.as_ptr()) };
    if ret == -1 {
        Err(io::Error::last_os_error())
    } else {
        Ok(())
    }
}

// ----- main: exercise the wrappers -----

fn main() {
    println!("=== Pattern 1: Safe Wrappers Demo ===\n");

    // These three never fail.
    println!("My PID:        {}", getpid());
    println!("Parent PID:    {}", getppid());
    println!("My UID:        {}", getuid());

    // Test that we exist by sending ourselves the null signal.
    let me = getpid();
    match kill(me, 0) {
        Ok(()) => println!("\nI exist (kill(self, 0) succeeded)"),
        Err(e) => println!("\nUnexpected: kill(self, 0) failed: {}", e),
    }

    // Try to signal a process that almost certainly doesn't exist.
    // This demonstrates errno capture.
    println!("\nTrying to signal PID 999999...");
    match kill(999_999, 0) {
        Ok(()) => println!("  Surprisingly, that PID exists."),
        Err(e) => {
            println!("  Failed (as expected): {}", e);
            println!("  errno value: {:?}", e.raw_os_error());
            // ESRCH = "No such process" = errno 3 on Linux
            if e.raw_os_error() == Some(libc::ESRCH) {
                println!("  Confirmed: this is ESRCH (No such process)");
            }
        }
    }

    // Try to unlink a file that doesn't exist.
    println!("\nTrying to unlink /tmp/this-file-does-not-exist-12345...");
    match unlink("/tmp/this-file-does-not-exist-12345") {
        Ok(()) => println!("  Surprisingly, that file existed and was removed."),
        Err(e) => {
            println!("  Failed (as expected): {}", e);
            if e.raw_os_error() == Some(libc::ENOENT) {
                println!("  Confirmed: ENOENT (No such file or directory)");
            }
        }
    }

    println!("\n=== Demo complete ===");
}
```

### Run it

```bash
cargo run
```

### Expected output

```
=== Pattern 1: Safe Wrappers Demo ===

My PID:        12345
Parent PID:    9876
My UID:        1000

I exist (kill(self, 0) succeeded)

Trying to signal PID 999999...
  Failed (as expected): No such process (os error 3)
  errno value: Some(3)
  Confirmed: this is ESRCH (No such process)

Trying to unlink /tmp/this-file-does-not-exist-12345...
  Failed (as expected): No such file or directory (os error 2)
  Confirmed: ENOENT (No such file or directory)

=== Demo complete ===
```

(Your PIDs and UIDs will differ.)

### What to notice

1. The five `unsafe { ... }` blocks each contain one libc call and nothing else.
2. Every block has a SAFETY comment.
3. The wrappers are themselves safe — `main` never writes `unsafe`.
4. Errors are propagated as `io::Result`, not raw `-1` returns.
5. `e.raw_os_error()` lets you compare against specific errno constants like `libc::ESRCH` and `libc::ENOENT`.

---

## Example 2 — Newtypes Around Raw Resources

**What this program does:** It defines `Pid`, `Uid`, and `Signal` newtypes, gives them useful methods, and shows how the type system prevents mixing them up. It also demonstrates the *typestate* version: an `UncheckedPid` that you must verify before it becomes a `LivePid` you can actually signal.

### `Cargo.toml`

```toml
[package]
name = "example2-newtypes"
version = "0.1.0"
edition = "2021"

[dependencies]
libc = "0.2"
```

### `src/main.rs`

```rust
//! Example 2: Newtypes around raw resources.
//!
//! Demonstrates how wrapping integer IDs in distinct types prevents
//! accidental misuse and enables typestate-style API design.

use std::io;

// ----- Pid newtype -----

/// A process ID. Wraps a raw kernel pid_t.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Pid(i32);

impl Pid {
    pub fn from_raw(raw: i32) -> Self { Pid(raw) }
    pub fn as_raw(self) -> i32 { self.0 }

    /// The current process's PID.
    pub fn this() -> Self {
        // SAFETY: getpid never fails.
        Pid(unsafe { libc::getpid() })
    }

    /// The parent process's PID.
    pub fn parent() -> Self {
        // SAFETY: getppid never fails.
        Pid(unsafe { libc::getppid() })
    }
}

// Custom Display so println!("{}", pid) prints just the number.
impl std::fmt::Display for Pid {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

// ----- Uid newtype -----

/// A user ID. Wraps a raw kernel uid_t.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Uid(u32);

impl Uid {
    pub fn from_raw(raw: u32) -> Self { Uid(raw) }
    pub fn as_raw(self) -> u32 { self.0 }

    /// The current process's real UID.
    pub fn current() -> Self {
        // SAFETY: getuid never fails.
        Uid(unsafe { libc::getuid() })
    }

    /// True if this is the root user.
    pub fn is_root(self) -> bool { self.0 == 0 }
}

impl std::fmt::Display for Uid {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

// ----- Signal newtype -----

/// A signal number. Wraps a raw kernel signal value.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Signal(i32);

impl Signal {
    pub const NULL: Signal    = Signal(0);   // existence probe
    pub const SIGTERM: Signal = Signal(libc::SIGTERM);
    pub const SIGKILL: Signal = Signal(libc::SIGKILL);
    pub const SIGCONT: Signal = Signal(libc::SIGCONT);

    pub fn as_raw(self) -> i32 { self.0 }
}

// ----- Typestate: unchecked vs verified PIDs -----

/// A PID we have not yet verified exists.
#[derive(Debug, Clone, Copy)]
pub struct UncheckedPid(Pid);

impl UncheckedPid {
    pub fn new(pid: Pid) -> Self { UncheckedPid(pid) }

    /// Verify the process exists. Returns Some(LivePid) if it does.
    pub fn check(self) -> Option<LivePid> {
        // SAFETY: signal 0 is harmless — it only checks for existence.
        let ret = unsafe { libc::kill(self.0.as_raw(), 0) };
        if ret == 0 {
            Some(LivePid(self.0))
        } else {
            None
        }
    }
}

/// A PID we have verified is a running process.
///
/// Functions taking LivePid are guaranteed (at construction time) to
/// have been validated. The type system enforces the workflow.
#[derive(Debug, Clone, Copy)]
pub struct LivePid(Pid);

impl LivePid {
    pub fn pid(self) -> Pid { self.0 }
}

// ----- A function that requires a verified PID -----

/// Sends a signal to a verified process.
///
/// This function takes LivePid, not Pid — callers must check existence first.
fn signal_process(target: LivePid, signal: Signal) -> io::Result<()> {
    // SAFETY: kill is sound for any (pid, signal). LivePid construction
    // verified the PID existed, though it could have exited since.
    let ret = unsafe { libc::kill(target.pid().as_raw(), signal.as_raw()) };
    if ret == -1 {
        Err(io::Error::last_os_error())
    } else {
        Ok(())
    }
}

// ----- main: exercise the newtypes -----

fn main() {
    println!("=== Pattern 2: Newtypes Demo ===\n");

    // Construction via the type's own methods.
    let me = Pid::this();
    let parent = Pid::parent();
    let uid = Uid::current();

    println!("My PID:     {}", me);
    println!("Parent PID: {}", parent);
    println!("My UID:     {} (root: {})", uid, uid.is_root());

    // Demonstrate that you CAN'T accidentally mix types.
    // Uncomment any of these lines to see the compiler refuse:
    //
    //     signal_process(me, Signal::NULL);
    //     // ^ error: expected LivePid, found Pid
    //
    //     let _ = uid == me;
    //     // ^ error: cannot compare Uid with Pid

    // Typestate demo: must verify before signaling.
    println!("\nVerifying my own PID exists...");
    let unchecked = UncheckedPid::new(me);
    match unchecked.check() {
        Some(live) => {
            println!("  Verified! Got a LivePid: {:?}", live);
            // Now we can call functions that require a LivePid.
            signal_process(live, Signal::NULL).unwrap();
            println!("  Sent null signal successfully.");
        }
        None => {
            println!("  Verification failed (this shouldn't happen for self).");
        }
    }

    println!("\nVerifying PID 999999 exists...");
    let unchecked = UncheckedPid::new(Pid::from_raw(999_999));
    match unchecked.check() {
        Some(_) => println!("  Surprisingly, it does."),
        None => println!("  Doesn't exist — can't construct a LivePid."),
    }
    // Note: because check() returned None, there is NO way to call
    // signal_process with that PID. The type system makes the bug
    // unrepresentable.

    println!("\n=== Demo complete ===");
}
```

### Run it

```bash
cargo run
```

### Expected output

```
=== Pattern 2: Newtypes Demo ===

My PID:     12345
Parent PID: 9876
My UID:     1000 (root: false)

Verifying my own PID exists...
  Verified! Got a LivePid: LivePid(Pid(12345))
  Sent null signal successfully.

Verifying PID 999999 exists...
  Doesn't exist — can't construct a LivePid.

=== Demo complete ===
```

### What to notice

1. `Pid`, `Uid`, and `Signal` are all `Copy` — they behave like the integers they wrap, but the compiler treats them as distinct types.
2. The commented-out lines in `main` would not compile — uncomment them and see the compiler errors. That's the type system catching bugs.
3. `signal_process` takes `LivePid`, not `Pid`. There's no way to call it without first calling `.check()`. The typestate pattern makes the workflow mandatory.
4. Each newtype has methods that only make sense for its type (e.g., `Uid::is_root()`).

---

## Example 3 — RAII for Resource Management

**What this program does:** It defines an `OwnedFd` that wraps a raw file descriptor and closes it on drop. The program opens a file, reads it, and lets `Drop` handle cleanup. It also intentionally triggers different exit paths (early return on error) to prove `Drop` runs in all of them. We use `strace` afterward to verify that `close` actually happens.

### `Cargo.toml`

```toml
[package]
name = "example3-raii"
version = "0.1.0"
edition = "2021"

[dependencies]
libc = "0.2"
```

### `src/main.rs`

```rust
//! Example 3: RAII for resource management.
//!
//! Demonstrates how Drop guarantees cleanup, even on error paths.
//! Run under strace to verify close() is called.

use std::ffi::CString;
use std::io;
use std::os::unix::io::RawFd;

// ----- OwnedFd: an RAII file descriptor -----

/// A file descriptor that closes itself when dropped.
pub struct OwnedFd {
    fd: RawFd,
}

impl OwnedFd {
    /// Open a file read-only.
    pub fn open_readonly(path: &str) -> io::Result<Self> {
        let c_path = CString::new(path).map_err(|_| {
            io::Error::new(io::ErrorKind::InvalidInput, "NUL in path")
        })?;

        // SAFETY: open(2) takes a valid C string and integer flags.
        let fd = unsafe { libc::open(c_path.as_ptr(), libc::O_RDONLY) };
        if fd == -1 {
            Err(io::Error::last_os_error())
        } else {
            println!("  [OwnedFd] opened fd {} for {:?}", fd, path);
            Ok(OwnedFd { fd })
        }
    }

    pub fn as_raw(&self) -> RawFd { self.fd }

    /// Read up to buf.len() bytes from this fd.
    pub fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
        // SAFETY: self.fd is valid (constructor guaranteed); buf is a
        // valid Rust slice with buf.len() bytes of writable memory.
        let ret = unsafe {
            libc::read(self.fd, buf.as_mut_ptr() as *mut libc::c_void, buf.len())
        };
        if ret < 0 {
            Err(io::Error::last_os_error())
        } else {
            Ok(ret as usize)
        }
    }
}

impl Drop for OwnedFd {
    fn drop(&mut self) {
        // SAFETY: we own the fd; close is sound. We deliberately ignore
        // close errors because there's nothing useful to do with them
        // in a destructor.
        let ret = unsafe { libc::close(self.fd) };
        if ret == 0 {
            println!("  [Drop] closed fd {}", self.fd);
        } else {
            println!("  [Drop] close failed for fd {} (ignored)", self.fd);
        }
    }
}

// ----- Demonstrating Drop on the success path -----

fn read_first_bytes(path: &str, n: usize) -> io::Result<Vec<u8>> {
    let fd = OwnedFd::open_readonly(path)?;
    let mut buf = vec![0u8; n];
    let read = fd.read(&mut buf)?;
    buf.truncate(read);
    Ok(buf)
    // ⬆ fd's Drop runs here automatically.
}

// ----- Demonstrating Drop on an error path -----

fn read_then_fail(path: &str) -> io::Result<Vec<u8>> {
    let fd = OwnedFd::open_readonly(path)?;
    let mut buf = vec![0u8; 16];
    let _ = fd.read(&mut buf)?;

    // Simulate an error after we've successfully opened the fd.
    println!("  (about to return an error — watch Drop run)");
    Err(io::Error::new(io::ErrorKind::Other, "simulated failure"))
    // ⬆ fd's Drop still runs here, even though we're returning Err.
}

// ----- Demonstrating Drop on a panic path -----

fn read_then_panic(path: &str) {
    let _fd = OwnedFd::open_readonly(path).unwrap();
    println!("  (about to panic — watch Drop run anyway)");
    panic!("simulated panic");
    // ⬆ Drop runs during stack unwinding.
}

// ----- main -----

fn main() {
    println!("=== Pattern 3: RAII Demo ===\n");

    // ---- Path 1: success ----
    println!("Demo 1: success path (open, read, return Ok)");
    match read_first_bytes("/etc/hostname", 64) {
        Ok(bytes) => println!("  read {} bytes: {:?}", bytes.len(),
                              String::from_utf8_lossy(&bytes).trim()),
        Err(e) => println!("  error: {}", e),
    }

    // ---- Path 2: error after open ----
    println!("\nDemo 2: error path (open succeeds, then fail)");
    match read_then_fail("/etc/hostname") {
        Ok(_)  => println!("  unexpectedly succeeded"),
        Err(e) => println!("  got error (as expected): {}", e),
    }

    // ---- Path 3: panic after open ----
    println!("\nDemo 3: panic path (open succeeds, then panic)");
    let result = std::panic::catch_unwind(|| {
        read_then_panic("/etc/hostname")
    });
    match result {
        Ok(_)  => println!("  unexpectedly returned normally"),
        Err(_) => println!("  caught panic (as expected); fd was closed during unwind"),
    }

    // ---- Path 4: file doesn't exist ----
    println!("\nDemo 4: open fails (no Drop because no fd was created)");
    match read_first_bytes("/no/such/path", 16) {
        Ok(_)  => println!("  unexpectedly succeeded"),
        Err(e) => println!("  got error (as expected): {}", e),
    }

    println!("\n=== Demo complete ===");
}
```

### Run it

```bash
cargo run
```

### Expected output

```
=== Pattern 3: RAII Demo ===

Demo 1: success path (open, read, return Ok)
  [OwnedFd] opened fd 3 for "/etc/hostname"
  [Drop] closed fd 3
  read 12 bytes: "your-hostname"

Demo 2: error path (open succeeds, then fail)
  [OwnedFd] opened fd 3 for "/etc/hostname"
  (about to return an error — watch Drop run)
  [Drop] closed fd 3
  got error (as expected): simulated failure

Demo 3: panic path (open succeeds, then panic)
  [OwnedFd] opened fd 3 for "/etc/hostname"
  (about to panic — watch Drop run anyway)
  [Drop] closed fd 3
thread 'main' panicked at src/main.rs:NN:N:
simulated panic
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
  caught panic (as expected); fd was closed during unwind

Demo 4: open fails (no Drop because no fd was created)
  got error (as expected): No such file or directory (os error 2)

=== Demo complete ===
```

### Bonus: verify with strace

To prove the closes really happen at the kernel level, build and run under strace:

```bash
cargo build
strace -e trace=openat,close ./target/debug/example3-raii 2>&1 | grep -E '(openat|close).*hostname|close\([3-9]\)'
```

You'll see one `openat` and one `close` for each demo path that opened the file. The `close` call is the kernel-level proof that `Drop` actually ran.

### What to notice

1. In **Demo 1** (success), `Drop` runs at the end of the function.
2. In **Demo 2** (error path), `Drop` runs *before* the error is returned to the caller — Rust unwinds the local scope first.
3. In **Demo 3** (panic), `Drop` runs during stack unwinding.
4. In **Demo 4** (open fails), no `Drop` runs because no `OwnedFd` was ever constructed — `OwnedFd::open_readonly` returned `Err` before the struct was created.
5. You wrote zero cleanup code in `main`. The compiler generated all six `close` calls (three demos × two function calls each, except demo 4).

---

## Example 4 — errno and io::Error

**What this program does:** It calls several syscalls that can fail, captures `errno` via `io::Error::last_os_error()`, and demonstrates the patterns: capturing errno immediately, matching against specific error codes, and the `EINTR` retry idiom.

### `Cargo.toml`

```toml
[package]
name = "example4-errno"
version = "0.1.0"
edition = "2021"

[dependencies]
libc = "0.2"
```

### `src/main.rs`

```rust
//! Example 4: errno and io::Error.
//!
//! Demonstrates how to capture errno into io::Error, match specific error
//! codes, and handle EINTR with a retry loop.

use std::ffi::CString;
use std::io;

// ----- Wrapper for stat that captures errno -----

/// Returns true if a path exists.
fn path_exists(path: &str) -> io::Result<bool> {
    let c_path = CString::new(path).map_err(|_| {
        io::Error::new(io::ErrorKind::InvalidInput, "NUL in path")
    })?;

    // We use stat just to test existence — we don't care about the result.
    let mut statbuf: libc::stat = unsafe { std::mem::zeroed() };

    // SAFETY: stat takes a valid C string and a writable stat buffer.
    let ret = unsafe { libc::stat(c_path.as_ptr(), &mut statbuf) };

    if ret == 0 {
        return Ok(true);
    }

    // Capture errno IMMEDIATELY — before any other call that could clobber it.
    let err = io::Error::last_os_error();

    // Inspect the errno code.
    match err.raw_os_error() {
        Some(libc::ENOENT) => Ok(false),    // missing is not an error here
        Some(libc::EACCES) => Err(err),     // permission denied is an error
        _                  => Err(err),     // anything else is an error
    }
}

// ----- A wrapper that demonstrates the EINTR retry pattern -----

/// Sleep for `seconds` seconds, retrying on EINTR.
///
/// nanosleep can be interrupted by a signal, returning -1 with errno=EINTR
/// and the unslept time in `rem`. The standard idiom is to retry with the
/// remaining time until completion.
fn sleep_with_retry(seconds: u64) -> io::Result<()> {
    let mut req = libc::timespec {
        tv_sec: seconds as libc::time_t,
        tv_nsec: 0,
    };
    let mut rem = libc::timespec { tv_sec: 0, tv_nsec: 0 };

    loop {
        // SAFETY: req and rem are valid pointers to timespec values
        // we own; nanosleep is a standard syscall.
        let ret = unsafe { libc::nanosleep(&req, &mut rem) };

        if ret == 0 {
            return Ok(());
        }

        // Capture errno immediately.
        let err = io::Error::last_os_error();

        if err.raw_os_error() == Some(libc::EINTR) {
            println!("  (interrupted by a signal — retrying with {}s remaining)",
                     rem.tv_sec);
            // Use the remaining time as the new request.
            req = rem;
            continue;
        }

        return Err(err);
    }
}

// ----- A function that demonstrates the "capture errno first" rule -----

/// WRONG version: shows what happens if you let other code run before
/// capturing errno. We use this only to demonstrate the bug visually.
fn unlink_wrong_way(path: &str) -> io::Result<()> {
    let c_path = CString::new(path).unwrap();

    // SAFETY: standard wrapper.
    let ret = unsafe { libc::unlink(c_path.as_ptr()) };
    if ret == -1 {
        // ❌ BAD: between the failing syscall and last_os_error(), we
        // call eprintln!, which goes through libc's stdio. errno may
        // have been overwritten by the time we read it.
        eprintln!("  (unlink failed — but we're about to read errno LATE)");

        // The errno we get here might be wrong.
        let err = io::Error::last_os_error();
        return Err(err);
    }
    Ok(())
}

/// RIGHT version: capture errno immediately.
fn unlink_right_way(path: &str) -> io::Result<()> {
    let c_path = CString::new(path).unwrap();

    // SAFETY: standard wrapper.
    let ret = unsafe { libc::unlink(c_path.as_ptr()) };
    if ret == -1 {
        // ✅ Capture FIRST, then do anything else.
        let err = io::Error::last_os_error();
        eprintln!("  (unlink failed — errno captured before this print)");
        return Err(err);
    }
    Ok(())
}

// ----- main -----

fn main() {
    println!("=== Pattern 4: errno and io::Error Demo ===\n");

    // Demo 1: path_exists — distinguishing ENOENT from real errors.
    println!("Demo 1: path_exists with errno discrimination");
    for path in &["/etc/hostname", "/no/such/file", "/etc"] {
        match path_exists(path) {
            Ok(true)  => println!("  {:?}: exists", path),
            Ok(false) => println!("  {:?}: does not exist (errno=ENOENT)", path),
            Err(e)    => println!("  {:?}: error: {} (errno={:?})",
                                  path, e, e.raw_os_error()),
        }
    }

    // Demo 2: nanosleep with EINTR retry.
    // This will only show the retry message if a signal arrives during
    // the sleep. With nothing else going on, it just sleeps quietly.
    println!("\nDemo 2: nanosleep (EINTR-safe sleep)");
    println!("  sleeping for 1 second...");
    sleep_with_retry(1).unwrap();
    println!("  done.");

    // Demo 3: errno-timing. We can't reliably reproduce the bug in the
    // wrong version every time, but we show both side-by-side so you
    // can see what disciplined errno handling looks like.
    println!("\nDemo 3: capture errno IMMEDIATELY");
    println!("  Wrong way:");
    let _ = unlink_wrong_way("/no/such/file");
    println!("  Right way:");
    let _ = unlink_right_way("/no/such/file");

    // Demo 4: enumerate common errno codes by triggering them.
    println!("\nDemo 4: triggering specific errno codes");

    // ENOENT — no such file
    let r = unlink_right_way("/no/such/file/12345");
    print!("  ENOENT trigger: ");
    print_errno(&r);

    // EACCES — permission denied (try to remove a file in a read-only dir)
    // /proc files cannot be unlinked.
    let r = unlink_right_way("/proc/version");
    print!("  EACCES/EPERM trigger: ");
    print_errno(&r);

    println!("\n=== Demo complete ===");
}

fn print_errno(r: &io::Result<()>) {
    match r {
        Ok(())  => println!("(unexpected success)"),
        Err(e)  => {
            let code = e.raw_os_error();
            let name = match code {
                Some(libc::ENOENT) => "ENOENT",
                Some(libc::EACCES) => "EACCES",
                Some(libc::EPERM)  => "EPERM",
                Some(libc::EINTR)  => "EINTR",
                Some(libc::EBADF)  => "EBADF",
                Some(libc::EINVAL) => "EINVAL",
                _ => "other",
            };
            println!("errno={:?} ({}) — {}", code, name, e);
        }
    }
}
```

### Run it

```bash
cargo run
```

### Expected output

```
=== Pattern 4: errno and io::Error Demo ===

Demo 1: path_exists with errno discrimination
  "/etc/hostname": exists
  "/no/such/file": does not exist (errno=ENOENT)
  "/etc": exists

Demo 2: nanosleep (EINTR-safe sleep)
  sleeping for 1 second...
  done.

Demo 3: capture errno IMMEDIATELY
  Wrong way:
  (unlink failed — but we're about to read errno LATE)
  Right way:
  (unlink failed — errno captured before this print)

Demo 4: triggering specific errno codes
  ENOENT trigger:   (unlink failed — errno captured before this print)
errno=Some(2) (ENOENT) — No such file or directory (os error 2)
  EACCES/EPERM trigger:   (unlink failed — errno captured before this print)
errno=Some(1) (EPERM) — Operation not permitted (os error 1)

=== Demo complete ===
```

### What to notice

1. In Demo 1, `ENOENT` is treated as a normal "not found" rather than an error — knowing the specific errno code lets you make that distinction.
2. In Demo 2, `sleep_with_retry` would print the retry message if a signal arrived during the sleep. Try sending it `SIGCONT` from another terminal to see it: `kill -CONT <pid>`.
3. In Demo 3, both versions work in the simple case, but the "wrong way" pattern is a latent bug. In a complex codebase, the wrong-way version eventually fails when a logging call clobbers errno before you read it.
4. Demo 4 shows how to map `raw_os_error()` codes to readable names using `libc::ENOENT`, `libc::EPERM`, etc.

---

## Example 5 — Tour of nix

**What this program does:** It rewrites the operations from Examples 1-4 using `nix` instead of raw `libc`. Side-by-side, you'll see how much cleaner `nix` makes systems code, and which patterns from earlier examples are baked into the crate.

### `Cargo.toml`

```toml
[package]
name = "example5-nix-tour"
version = "0.1.0"
edition = "2021"

[dependencies]
nix = { version = "0.29", features = ["process", "signal", "fs", "user"] }
```

### `src/main.rs`

```rust
//! Example 5: A tour of the nix crate.
//!
//! Re-implements the previous examples using nix to show how the crate
//! applies all the previous patterns to the entire syscall surface.

use nix::sys::signal::{kill, Signal};
use nix::sys::stat::stat;
use nix::unistd::{getppid, getuid, Pid, Uid};
use std::path::Path;

fn main() -> nix::Result<()> {
    println!("=== Pattern 5: nix Tour ===\n");

    // ---- Process IDs and user IDs (newtypes provided by nix) ----
    println!("Process and user info (using nix newtypes):");
    let me = Pid::this();
    let parent = getppid();
    let uid: Uid = getuid();

    println!("  My PID:     {}", me);            // Pid is Display
    println!("  Parent PID: {}", parent);
    println!("  My UID:     {} (root: {})", uid, uid.is_root());

    // ---- Sending signals (typed Signal enum) ----
    println!("\nSignaling self with SIGCONT (harmless on a running process):");
    kill(me, Signal::SIGCONT)?;
    println!("  ok.");

    // Existence probe — nix takes Option<Signal>, where None means "null signal".
    println!("\nExistence probe of self (kill with None means null signal):");
    match kill(me, None) {
        Ok(())  => println!("  I exist."),
        Err(e)  => println!("  Surprising error: {}", e),
    }

    println!("\nExistence probe of PID 999999:");
    match kill(Pid::from_raw(999_999), None) {
        Ok(())  => println!("  Surprisingly, exists."),
        Err(e)  => {
            // nix returns nix::errno::Errno — a typed enum.
            // Compare directly against variants, not raw integers.
            use nix::errno::Errno;
            if e == Errno::ESRCH {
                println!("  Doesn't exist (Errno::ESRCH).");
            } else {
                println!("  Other error: {:?}", e);
            }
        }
    }

    // ---- Filesystem operations ----
    println!("\nFile metadata via nix::sys::stat::stat:");
    for path in &["/etc/hostname", "/no/such/file"] {
        match stat(Path::new(path)) {
            Ok(meta) => {
                println!("  {:?}: size={} bytes, mode={:o}",
                         path, meta.st_size, meta.st_mode & 0o777);
            }
            Err(e) => {
                println!("  {:?}: error — {:?}", path, e);
            }
        }
    }

    // ---- Demonstrating that nix newtypes prevent mixups ----
    // The following lines (commented out) demonstrate compiler errors:
    //
    //     let bad = me + parent;
    //     // ^ error: cannot add `Pid` to `Pid` (no Add impl)
    //
    //     kill(uid, Signal::SIGCONT);
    //     // ^ error: expected `Pid`, found `Uid`
    //
    //     kill(me, 18);
    //     // ^ error: expected `Option<Signal>`, found integer

    println!("\n=== Demo complete ===");
    Ok(())
}
```

### Run it

```bash
cargo run
```

### Expected output

```
=== Pattern 5: nix Tour ===

Process and user info (using nix newtypes):
  My PID:     12345
  Parent PID: 9876
  My UID:     1000 (root: false)

Signaling self with SIGCONT (harmless on a running process):
  ok.

Existence probe of self (kill with None means null signal):
  I exist.

Existence probe of PID 999999:
  Doesn't exist (Errno::ESRCH).

File metadata via nix::sys::stat::stat:
  "/etc/hostname": size=12 bytes, mode=644
  "/no/such/file": error — ENOENT

=== Demo complete ===
```

### What to notice

1. **No `unsafe` anywhere in `main`.** Every `unsafe` block is hidden inside `nix`. We're getting the same syscall functionality with none of the soundness obligations on us.
2. **Newtypes everywhere.** `Pid`, `Uid`, `Signal`, `Errno` — all the patterns from Examples 1-4 are built into the crate.
3. **Typed errors.** `nix::Result<T>` uses `Errno` as its error type — you compare directly against `Errno::ESRCH`, not against raw integers.
4. **`?` propagation works naturally.** `main` returns `nix::Result<()>` and uses `?` throughout.
5. **Compare to Example 1.** That file was 80 lines; this one is 50, with more functionality. `nix` is what scaling Patterns 1-4 across the syscall surface looks like.

The only caveat: `nix` covers most but not all syscalls. For exotic or recently-added syscalls, you may still need `libc` directly — and when you do, you fall back to applying Patterns 1-4 manually. The patterns aren't replaced by `nix`; they're the foundation `nix` is built on.

---

## How to use these examples

Now that you have all five running, the workflow that gets the most value out of them:

1. Build and run each one. Confirm the output matches.
2. Modify each one. Add a function, change behavior, break it, fix it. The compiler will teach you what you got wrong.
3. Try the deliberately-broken modifications I suggested:
   - Uncomment the type-mixing lines in Examples 2 and 5 to see compiler errors.
   - Run Example 3 under `strace` to see the kernel-level proof that `Drop` ran.
   - Send a signal to Example 4 mid-sleep to see the EINTR retry fire.
4. When you start writing real Phase 1 code, refer back to these. You'll need to re-apply each pattern, often within the same function.

The goal isn't to memorize the code. It's to develop the muscle memory: when you see a libc call, the wrapper structure happens automatically. When you see an integer ID, the newtype question comes up automatically. When you see a resource, the `Drop` impl gets sketched automatically. That muscle memory is the real deliverable of Step 3.
