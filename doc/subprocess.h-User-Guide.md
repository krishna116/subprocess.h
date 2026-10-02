# subprocess.h User Guide

[subprocess.h](https://github.com/sheredom/subprocess.h) is a **single-header**, cross-platform library for launching external processes, interacting with their stdin/stdout/stderr, and waiting for them to finish.

| Item | Details |
|---|---|
| Language | C99; can also be used directly from C++ (the header declares its own `extern "C"`) |
| Platforms | Linux, macOS, Windows |
| Compilers | gcc, clang, MSVC `cl.exe`, `clang-cl.exe` |
| License | [Unlicense](https://unlicense.org/) (public domain, no restrictions) |
| Integration cost | One `.h` file — no implementation macro, nothing to build |

> Every C/C++ example in this document was compiled and run on **Linux x86-64 / glibc 2.39 / gcc 13.3**. Passages covering Windows-specific behavior are flagged as such; those conclusions come from reading the source and the official README.

---

## Contents

- [1. Integration](#1-integration)
- [2. Core Concepts](#2-core-concepts)
- [3. API Reference](#3-api-reference)
- [4. Examples](#4-examples)
- [5. Gotchas and Caveats](#5-gotchas-and-caveats)
- [6. Compile-Time Configuration Macros](#6-compile-time-configuration-macros)
- [7. FAQ](#7-faq)
- [8. Quick Reference](#8-quick-reference)

---

## 1. Integration

### 1.1 Basic usage

Drop `subprocess.h` into your project and include it:

```c
#include "subprocess.h"
```

You do **not** need to define an implementation macro such as `SUBPROCESS_IMPLEMENTATION`, the way many single-header libraries require. Every function in the library is defined as `__attribute__((weak))` under GCC/Clang and as `__inline` under MSVC, which means:

- Including it from a single source file produces no errors beyond "unused function" warnings;
- **Including it from multiple source files does not cause duplicate-definition errors** (verified: two `.c` files each including the header link cleanly).

### 1.2 You must enable the POSIX/GNU feature macros on Linux (important)

This is the step that trips up most people. Internally the library uses `pipe2`, `O_CLOEXEC`, `fdopen`, and `posix_spawn_*`, which means it **fails to compile outright in strict ISO C mode**:

```sh
# Fails: 'O_CLOEXEC' undeclared / implicit declaration of 'pipe2'
gcc -std=c11 -c main.c
```

Use either of these instead:

```sh
gcc -std=gnu11 -c main.c                 # Recommended: gnu mode enables the extensions by default
gcc -std=c11 -D_GNU_SOURCE -c main.c     # Or define the feature macro explicitly
```

Same for C++:

```sh
g++ -std=c++17 -D_GNU_SOURCE main.cpp
```

With CMake, the simplest approach is:

```cmake
add_compile_definitions(_GNU_SOURCE)   # or target_compile_definitions(...)
```

> macOS and Windows are unaffected — they expose the required interfaces by default.

---

## 2. Core Concepts

### 2.1 `struct subprocess_s`

The process handle. Declare it by value; no manual initialization is required (though zero-initializing is recommended):

```c
struct subprocess_s p;        // fine
struct subprocess_s p = {0};  // safer; helps catch "used before successful creation" bugs
```

The fields are readable but **should not be modified by hand**:

```c
struct subprocess_s {
  FILE *stdin_file;
  FILE *stdout_file;
  FILE *stderr_file;
#if defined(_WIN32)
  void *hProcess;
  void *hStdInput;
  void *hEventOutput;
  void *hEventError;
#else
  pid_t child;
  int   return_status;
#endif
  int alive;
  int no_wait;
};
```

### 2.2 Options — `enum subprocess_option_e`

A bitmask, combined with `|` and passed to `subprocess_create` / `subprocess_create_ex`.

| Option | Value | Description |
|---|---|---|
| `subprocess_option_combined_stdout_stderr` | `0x1` | Merges stdout and stderr into a single `FILE*`. In this mode `subprocess_stderr()` returns `NULL` |
| `subprocess_option_inherit_environment` | `0x2` | The child **inherits the parent's environment variables**. Without it, the child gets an empty environment |
| `subprocess_option_enable_async` | `0x4` | Enables asynchronous reading, which is required to safely call `subprocess_read_stdout/stderr` before `join` |
| `subprocess_option_no_window` | `0x8` | Launch without a visible window where the platform supports it (meaningful on Windows) |
| `subprocess_option_search_user_path` | `0x10` | Search `PATH` for the program name. **Always enabled on Windows.** Note that it uses the **launching process's own PATH**, not the PATH inside any custom `environment` you pass in |
| `subprocess_option_enable_async_no_wait` | `0x20` | Combined with `enable_async`, returns 0 immediately instead of blocking when no data is available |

### 2.3 Error codes — `enum subprocess_error_e`

Returned by `subprocess_create` / `subprocess_create_ex`. Success is always 0; failures are negative.

| Enumerator | Value | Meaning |
|---|---|---|
| `subprocess_error_success` | `0` | Success |
| `subprocess_error_unknown` | `-1` | Unknown error |
| `subprocess_error_invalid_options` | `-2` | Invalid option combination |
| `subprocess_error_invalid_environment` | `-3` | Environment conflicts with options (see below) |
| `subprocess_error_not_found` | `-4` | Program not found |
| `subprocess_error_permission_denied` | `-5` | Permission denied |
| `subprocess_error_no_memory` | `-6` | Out of memory |
| `subprocess_error_pipe` | `-7` | Pipe creation failed |
| `subprocess_error_spawn` | `-8` | Process creation failed |
| `subprocess_error_not_supported` | `-9` | Not supported on this platform (for example, specifying a working directory on old glibc) |

After you get an error code, dig into the platform-specific reason: check `errno` on POSIX, or `GetLastError()` on Windows.

---

## 3. API Reference

### 3.1 Creating a process

```c
subprocess_weak int subprocess_create(const char *const command_line[],
                                      int options,
                                      struct subprocess_s *const out_process);

subprocess_weak int
subprocess_create_ex(const char *const command_line[], int options,
                     const char *const environment[],
                     const char *const process_cwd,
                     struct subprocess_s *const out_process);
```

- `command_line`: a `NULL`-terminated array of strings; the first element is the program name or path. This memory only needs to stay valid **until the function returns**.
- `environment`: an array of `"FOO=BAR"` strings, `NULL`-terminated; passing `NULL` means an **empty environment** (unless `inherit_environment` is also set).
- `process_cwd`: the child's working directory; pass `NULL` to inherit the parent's current directory.
- Return value: 0 on success, otherwise a negative error code.

Constraints:

- If `options` includes `subprocess_option_inherit_environment`, then `environment` **must be `NULL`**, otherwise you get `-3 invalid_environment`. If you want "inherit plus extra variables", read the parent's environment yourself and append to `environment`.
- On Windows every string is interpreted as **UTF-8** and converted internally to wide characters for the Unicode APIs.

### 3.2 Getting the standard streams

```c
subprocess_pure FILE *subprocess_stdin (const struct subprocess_s *const process);
subprocess_pure FILE *subprocess_stdout(const struct subprocess_s *const process);
subprocess_pure FILE *subprocess_stderr(const struct subprocess_s *const process);
```

These return standard C `FILE*` handles, usable directly with `fputs` / `fgets` / `fread` and friends.

- The `FILE*` from `subprocess_stdin()` is **written to by the parent** to feed the child's stdin.
- If process creation failed (non-zero return), all three return `NULL`. **Using them without checking the return value segfaults immediately** — this is the single most common crash with this library.
- With `combined_stdout_stderr`, `subprocess_stderr()` always returns `NULL` (verified).

### 3.3 Waiting and cleanup

```c
subprocess_weak int subprocess_join(struct subprocess_s *const process,
                                    int *const out_return_code);
subprocess_weak int subprocess_destroy(struct subprocess_s *const process);
subprocess_weak int subprocess_terminate(struct subprocess_s *const process);
subprocess_weak int subprocess_alive(struct subprocess_s *const process);
```

| Function | Purpose |
|---|---|
| `subprocess_join` | Blocks until the process exits; the exit code is written to `out_return_code` (may be `NULL`). **Calling it closes the child's stdin pipe** |
| `subprocess_destroy` | Releases the handle and pipes. May be called before the process exits, in which case the child can outlive the parent |
| `subprocess_terminate` | Forcefully terminates a (possibly hung) process. You can still call `join` / `destroy` afterwards |
| `subprocess_alive` | Whether the process is still running; non-zero means alive |

Key points:

- After `subprocess_join`, **do not write to stdin**; after `subprocess_destroy`, **do not read or write any stream**.
- The exit code obtained from `join` after `terminate` is **guaranteed non-zero** (verified: terminating `sleep 30` yields an exit code of 1).
- If the child dies from an unhandled exception, the exit code is likewise non-zero.

### 3.4 Asynchronous reading

```c
subprocess_weak unsigned
subprocess_read_stdout(struct subprocess_s *const process, char *const buffer,
                       unsigned size);

subprocess_weak unsigned
subprocess_read_stderr(struct subprocess_s *const process, char *const buffer,
                       unsigned size);
```

Returns the number of bytes actually read. A return of `0` has two possible meanings: the process has finished, or `enable_async_no_wait` is set and no data is currently available.

**You must pass `subprocess_option_enable_async` at creation time** for this to be a supported use. Calling these without that option is undefined behavior — in testing, simple cases occasionally do return data, but you must never rely on it.

---

## 4. Examples

All of the following compile as-is (remember `-D_GNU_SOURCE` or `-std=gnu11`).

### 4.1 Minimal example: run a command and get its exit code

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"sh", "-c", "exit 42", NULL};
  struct subprocess_s p = {0};

  int rc = subprocess_create(cmd, subprocess_option_search_user_path, &p);
  if (0 != rc) {                       // always check!
    printf("create failed: %d\n", rc);
    return 1;
  }

  int code = -1;
  subprocess_join(&p, &code);
  printf("exit code = %d\n", code);    // 42

  subprocess_destroy(&p);
  return 0;
}
```

> Without `subprocess_option_search_user_path`, the first argument must be a path (e.g. `/bin/sh`, `./mytool`). A bare `sh` returns `-4 not_found`.

### 4.2 Capturing standard output

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"ls", "-l", NULL};
  struct subprocess_s p = {0};
  if (0 != subprocess_create(cmd, subprocess_option_search_user_path, &p))
    return 1;

  int code = -1;
  subprocess_join(&p, &code);          // wait for exit first

  char buf[1024];
  while (fgets(buf, sizeof(buf), subprocess_stdout(&p)))   // then read
    fputs(buf, stdout);

  subprocess_destroy(&p);
  return code;
}
```

### 4.3 Capturing stdout and stderr separately

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"sh", "-c", "echo to-out; echo to-err 1>&2", NULL};
  struct subprocess_s p = {0};
  if (0 != subprocess_create(cmd, subprocess_option_search_user_path, &p))
    return 1;

  int code = -1;
  subprocess_join(&p, &code);

  char buf[256];
  while (fgets(buf, sizeof(buf), subprocess_stdout(&p)))
    printf("OUT| %s", buf);
  while (fgets(buf, sizeof(buf), subprocess_stderr(&p)))
    printf("ERR| %s", buf);

  subprocess_destroy(&p);
  return 0;
}
```

### 4.4 Merging stdout and stderr

```c
const char *cmd[] = {"sh", "-c", "echo A; echo B 1>&2", NULL};
struct subprocess_s p = {0};
subprocess_create(cmd,
                  subprocess_option_search_user_path |
                      subprocess_option_combined_stdout_stderr,
                  &p);

int code = -1;
subprocess_join(&p, &code);

/* Note: subprocess_stderr(&p) == NULL here; everything comes through stdout */
char buf[256];
while (fgets(buf, sizeof(buf), subprocess_stdout(&p)))
  printf("%s", buf);          // prints A, then B

subprocess_destroy(&p);
```

### 4.5 Writing to standard input

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"cat", NULL};
  struct subprocess_s p = {0};
  if (0 != subprocess_create(cmd, subprocess_option_search_user_path, &p))
    return 1;

  FILE *in = subprocess_stdin(&p);
  fputs("hello from parent\n", in);
  fflush(in);                 // flush is required

  int code = -1;
  subprocess_join(&p, &code); // join closes the stdin pipe, which is what makes cat exit

  char buf[128] = {0};
  if (fgets(buf, sizeof(buf), subprocess_stdout(&p)))
    printf("echo back: %s", buf);

  subprocess_destroy(&p);
  return 0;
}
```

Key point: programs like `cat`, `grep`, and `sort` that read stdin until EOF only exit once you **`fflush` and then `subprocess_join`** (which closes the pipe).

### 4.6 Asynchronous reading (the correct way to handle large output)

This is the most important example. **When the child's output exceeds the pipe buffer (64 KiB by default on Linux), joining before reading deadlocks** — the child blocks writing to the pipe while the parent blocks waiting for it to exit. With roughly 820 KB of output, the program hangs forever (verified).

The fix: pass `enable_async` at creation and read as you wait.

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  /* produces roughly 820 KB of output */
  const char *cmd[] = {"sh", "-c",
                       "i=0; while [ $i -lt 20000 ]; do echo "
                       "\"0123456789012345678901234567890123456789\"; "
                       "i=$((i+1)); done",
                       NULL};

  struct subprocess_s p = {0};
  if (0 != subprocess_create(cmd, subprocess_option_search_user_path |
                                      subprocess_option_enable_async, &p))
    return 1;

  char buf[4096];
  unsigned total = 0;
  for (;;) {
    unsigned n = subprocess_read_stdout(&p, buf, sizeof(buf));
    if (0 == n) {
      if (!subprocess_alive(&p)) break;   // process exited and nothing is left
      continue;                            // still running, keep reading
    }
    total += n;
    /* process buf[0..n) here */
  }

  int code = -1;
  subprocess_join(&p, &code);
  printf("drained %u bytes, exit=%d\n", total, code);

  subprocess_destroy(&p);
  return 0;
}
```

That `continue` is a busy loop. In production code, either sleep briefly between iterations or use `enable_async_no_wait` with polling — see [5.2](#52-busy-waiting-and-cpu-usage).

### 4.7 Custom environment and working directory

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"sh", "-c", "echo FOO=$FOO", NULL};
  const char *env[] = {"FOO=BAR", "PATH=/usr/bin:/bin", NULL};
  struct subprocess_s p = {0};

  int rc = subprocess_create_ex(cmd,
                                subprocess_option_search_user_path,
                                env,       /* fully replaces the environment */
                                "/tmp",    /* child working directory; NULL means inherit */
                                &p);
  if (0 != rc) return 1;

  int code = -1;
  subprocess_join(&p, &code);

  char buf[256];
  while (fgets(buf, sizeof(buf), subprocess_stdout(&p)))
    printf("%s", buf);      // FOO=BAR

  subprocess_destroy(&p);
  return 0;
}
```

**`environment` fully replaces the environment; it does not append.** To inherit the parent's environment instead, use:

```c
subprocess_create(cmd, subprocess_option_inherit_environment |
                           subprocess_option_search_user_path, &p);
```

### 4.8 Running with a timeout

The library has no built-in timeout. Implement one by polling `subprocess_alive` and calling `subprocess_terminate`:

```c
#include "subprocess.h"
#include <time.h>

static void sleep_ms(int ms) {
  struct timespec ts = {ms / 1000, (long)(ms % 1000) * 1000000L};
  nanosleep(&ts, NULL);
}

/* 0 = exited normally; -1 = failed to create; -2 = timed out and was terminated */
static int run_with_timeout(const char *const cmd[], int timeout_ms, int *out_code) {
  struct subprocess_s p = {0};
  if (0 != subprocess_create(cmd, subprocess_option_search_user_path |
                                      subprocess_option_enable_async, &p))
    return -1;

  int waited = 0;
  while (subprocess_alive(&p) && waited < timeout_ms) {
    sleep_ms(10);
    waited += 10;
  }

  if (subprocess_alive(&p)) {
    subprocess_terminate(&p);
    subprocess_join(&p, out_code);
    subprocess_destroy(&p);
    return -2;
  }

  subprocess_join(&p, out_code);
  subprocess_destroy(&p);
  return 0;
}
```

Verified: `sleep 10` with a 500 ms timeout returns `-2`, and `join` reports an exit code of 1 (non-zero).

### 4.9 Using it from C++ (RAII wrapper)

The header declares `extern "C"` itself, so C++ can include it directly:

```cpp
#include "subprocess.h"
#include <stdexcept>
#include <string>
#include <vector>

class Process {
public:
  explicit Process(std::vector<std::string> args,
                   int options = subprocess_option_search_user_path |
                                 subprocess_option_enable_async)
      : args_(std::move(args)) {
    argv_.reserve(args_.size() + 1);
    for (auto &a : args_) argv_.push_back(a.c_str());
    argv_.push_back(nullptr);

    if (0 != subprocess_create(argv_.data(), options, &proc_))
      throw std::runtime_error("failed to launch: " + args_.front());
    started_ = true;
  }

  ~Process() { if (started_) subprocess_destroy(&proc_); }

  Process(const Process &) = delete;
  Process &operator=(const Process &) = delete;

  std::string readAllStdout() {
    std::string out;
    char buf[4096];
    for (;;) {
      unsigned n = subprocess_read_stdout(&proc_, buf, sizeof(buf));
      if (0 == n) {
        if (!subprocess_alive(&proc_)) break;
        continue;
      }
      out.append(buf, n);
    }
    return out;
  }

  int wait() {
    int code = -1;
    subprocess_join(&proc_, &code);
    return code;   // the destructor handles cleanup
  }

private:
  std::vector<std::string> args_;
  std::vector<const char *> argv_;
  subprocess_s proc_{};
  bool started_ = false;
};

// Usage:
//   Process p({"echo", "hi"});
//   std::cout << p.readAllStdout();
//   int code = p.wait();
```

Build it with: `g++ -std=c++17 -D_GNU_SOURCE main.cpp`.

---

## 5. Gotchas and Caveats

### 5.1 Always check the return value of `subprocess_create`

The library does **not** set the `FILE*` handles to some safe sentinel on failure — the three stream accessors return `NULL`. Calling `fgets`/`fputs` on `NULL` segfaults immediately.

This exact code crashed during testing:

```c
subprocess_create(cmd, 0, &p);            // no search_user_path; returns -4
fgets(buf, sizeof(buf), subprocess_stdout(&p));   // stdout_file == NULL → SIGSEGV
```

Make it a habit: **check the return value after every create**.

### 5.2 Busy-waiting and CPU usage

`subprocess_read_stdout` blocks when there is no data (in non-`no_wait` mode); under `enable_async_no_wait` it returns 0 immediately. The `continue` polling loop in section 4.6 is a busy loop that will peg one CPU core.

Better options:

- Insert a `nanosleep` in the loop (see `sleep_ms` in 4.8);
- Or use `subprocess_option_enable_async_no_wait` combined with a sleeping poll — verified to return 0 immediately without blocking when there is no data.

### 5.3 Deadlock: joining before reading large output

Worth repeating, because it is the easiest trap to fall into and the hardest to diagnose ("the program just hangs for no reason"):

| Scenario | Result (verified with ~820 KB of output) |
|---|---|
| `subprocess_join` first, then read stdout | **Hangs forever** (killed only by the timeout) |
| `enable_async` + read while waiting | Drains all 820000 bytes normally, exit code 0 |

Rule of thumb: **if the child's output could exceed the pipe buffer, you must use `enable_async` and read while waiting.** Conversely, if you know the output is small (a few KB), joining first is safe — examples 4.2 through 4.5 all use that pattern.

### 5.4 The default environment is empty, not inherited

This is deliberately the opposite of Windows `CreateProcessA`; upstream justifies it as picking the safest default:

```c
subprocess_create(cmd, 0, &p);                    // empty environment
subprocess_create(cmd, inherit_environment, &p);  // inherit the parent's environment
```

Running `/bin/sh -c 'echo HOME=[$HOME]'` under an empty environment prints `HOME=[]`, which confirms the environment really is empty.

But there is an **easy-to-misread trap**: under an empty environment, `echo $PATH` still prints a long list of directories (such as `/usr/local/sbin:/usr/local/bin:...`). That is the **shell's own built-in default PATH**, used when PATH is unset — it does not mean the parent's environment was inherited. Check a variable with no shell fallback, like `HOME`, to tell the difference.

Also, child processes that need network access generally need `inherit_environment`, since the environment implicitly carries the parent's credentials.

### 5.5 `search_user_path` uses the parent's PATH

Even if you pass a custom `environment` through `subprocess_create_ex` (one that contains `PATH=...`), `search_user_path` still resolves the program using the **launching process's own PATH**. If you want the program to be found via a custom PATH, build the absolute path yourself.

### 5.6 Process cleanup and zombies

- The normal path: `create` → (read/write) → `join` → `destroy`.
- `join` waits for the child to exit and reaps it, avoiding zombie processes.
- Calling `destroy` without `join` leaves the child running, potentially longer than the parent (upstream explicitly supports this "release an orphan" use case).
- Recommended termination order: `terminate` → `join` (yields a non-zero exit code) → `destroy`.

### 5.7 Windows notes

- On Windows, `search_user_path` is **always enabled**; you do not need to pass it.
- Command-line strings are interpreted as **UTF-8** and converted internally to the Unicode APIs — passing UTF-8 paths and arguments is safe, but make sure what you pass really is UTF-8.
- `subprocess_option_no_window` currently has a real effect only on Windows.
- `terminate` kills just the target process, not its descendants. Killing an entire process tree requires a Job Object, which this library does not provide.

### 5.8 Thread safety

A single `subprocess_s` handle **should not be operated on concurrently by multiple threads**. A common and safe division of labor is one thread reading stdout and another reading stderr, but never two threads calling `join` on the same process at once.

### 5.9 POSIX signals

Upstream recommends ignoring `SIGPIPE` in programs that use this library — writing to a closed pipe raises it, and the default action terminates your process.

```c
signal(SIGPIPE, SIG_IGN);
```

---

## 6. Compile-Time Configuration Macros

These can be defined before including the header to override the library's platform detection. You normally will not need to touch them.

| Macro | Purpose |
|---|---|
| `SUBPROCESS_HAVE_CWD` | Whether `subprocess_create_ex` can honor `process_cwd`. When unsupported it is 0, and passing a non-NULL working directory returns `-9 not_supported` with `errno` set to `ENOSYS`. Affected platforms: glibc < 2.29, macOS < 10.15, iOS/tvOS/watchOS. musl < 1.1.24 needs this defined manually to override detection |
| `SUBPROCESS_SPAWN_VIA_FORK` | Use `fork()+exec()` instead of `posix_spawn()`. Enabled automatically on AIX, OpenBSD, and NetBSD < 10 |
| `SUBPROCESS_ADDCHDIR_IS_POSIX` | Whether the platform provides the standard `posix_spawn_file_actions_addchdir` or the `_np` variant |
| `SUBPROCESS_SPAWN_REPORTS_EXEC_ERRORS` | Whether a failed `exec` is reported back to the caller |

---

## 7. FAQ

**Q: Why can't the child see the parent's environment variables?**
A: An empty environment is the default. Add `subprocess_option_inherit_environment`.

**Q: Why does it say the program can't be found?**
A: The program name isn't on PATH and you didn't pass `subprocess_option_search_user_path`. Either add that option or use an absolute path.

**Q: Why does combining `inherit_environment` with a custom `environment` fail?**
A: Upstream explicitly forbids it; you get `-3 invalid_environment`. For "inherit plus extras", read the parent's environment yourself and merge.

**Q: My program hangs. Why?**
A: 99% of the time it's the deadlock in [5.3](#53-deadlock-joining-before-reading-large-output). Switch to `enable_async` and read while waiting.

**Q: Can I get the child's PID?**
A: On POSIX read `process.child` (`pid_t`); on Windows read `process.hProcess`. These are internal fields, though, so future compatibility isn't guaranteed.

**Q: Will including from multiple source files cause duplicate definitions?**
A: No. GCC/Clang use weak symbols and MSVC uses inline. Verified: two translation units link cleanly.

---

## 8. Quick Reference

### Options for common needs

| Need | Option |
|---|---|
| Launch by program name (not an absolute path) | `search_user_path` |
| Child should inherit environment variables | `inherit_environment` |
| Output may be large / needs live reading | `enable_async` |
| Want non-blocking reads | `enable_async` \| `enable_async_no_wait` |
| Don't care about separation, see stdout+stderr together | `combined_stdout_stderr` |
| No window popup on Windows | `no_window` |

### Lifecycle order

```
subprocess_create[_ex]
        │
        ├─ (optional) subprocess_stdin  → fputs → fflush
        │
        ├─ small output: subprocess_join → fgets(stdout/stderr)
        │
        ├─ large output: subprocess_read_stdout loop → subprocess_join
        │
        └─ (optional) subprocess_alive polling → subprocess_terminate
        │
subprocess_join (if not already called)
        │
subprocess_destroy
```

### Return-value checklist

- [ ] Did `subprocess_create` / `create_ex` return 0?
- [ ] Did you confirm creation succeeded before using `subprocess_stdin/stdout/stderr`?
- [ ] With `combined_stdout_stderr`, did you avoid `subprocess_stderr` (it's NULL)?
- [ ] Are you reading or writing any stream after `join` / `destroy`?
- [ ] Does every successfully created process eventually reach `join` (or explicitly become an orphan) and `destroy`?

---

## Reference

- Repository: <https://github.com/sheredom/subprocess.h>
- License: Unlicense (public domain)
