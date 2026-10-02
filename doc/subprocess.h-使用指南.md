# subprocess.h 使用指南

[subprocess.h](https://github.com/sheredom/subprocess.h) 是一个**单头文件**的跨平台子进程库，用于启动外部进程、与其 stdin/stdout/stderr 交互、并等待其结束。

| 项目 | 说明 |
|---|---|
| 语言 | C99，同时可直接在 C++ 中使用（头文件自带 `extern "C"`） |
| 平台 | Linux、macOS、Windows |
| 编译器 | gcc、clang、MSVC `cl.exe`、`clang-cl.exe` |
| 许可 | [Unlicense](https://unlicense.org/)（公共领域，无任何限制） |
| 集成成本 | 一个 `.h` 文件，**无需实现宏、无需编译成库** |

> 本文档中所有 C/C++ 示例均在 **Linux x86-64 / glibc 2.39 / gcc 13.3** 下实测编译运行通过。涉及 Windows 特有行为的段落已单独标注，其结论来自源码与官方 README。

---

## 目录

- [1. 集成方式](#1-集成方式)
- [2. 核心概念](#2-核心概念)
- [3. API 参考](#3-api-参考)
- [4. 使用样例](#4-使用样例)
- [5. 注意事项](#5-注意事项)
- [6. 编译期配置宏](#6-编译期配置宏)
- [7. 常见问题](#7-常见问题)
- [8. 速查表](#8-速查表)

---

## 1. 集成方式

### 1.1 基本用法

把 `subprocess.h` 放进工程，直接包含即可：

```c
#include "subprocess.h"
```

**不需要**像很多单头库那样定义 `SUBPROCESS_IMPLEMENTATION` 之类的宏。库中所有函数在 GCC/Clang 下以 `__attribute__((weak))` 定义，在 MSVC 下以 `__inline` 定义，因此：

- 单个源文件包含不会报"未使用函数"之外的错误；
- **多个源文件同时包含也不会产生重复定义错误**（已实测：两个 `.c` 均 include 该头文件，链接通过）。

### 1.2 Linux 下必须开启 POSIX/GNU 特性宏（重要）

这是最容易卡住的一步。库内部使用了 `pipe2`、`O_CLOEXEC`、`fdopen`、`posix_spawn_*` 等接口，在**严格 ISO C 模式下会直接编译失败**：

```sh
# 会失败：'O_CLOEXEC' undeclared / implicit declaration of 'pipe2'
gcc -std=c11 -c main.c
```

正确的两种写法，任选其一：

```sh
gcc -std=gnu11 -c main.c                 # 推荐：gnu 模式默认开启扩展
gcc -std=c11 -D_GNU_SOURCE -c main.c     # 或显式定义特性宏
```

C++ 同理：

```sh
g++ -std=c++17 -D_GNU_SOURCE main.cpp
```

如果用 CMake，最省事的做法：

```cmake
add_compile_definitions(_GNU_SOURCE)   # 或 target_compile_definitions(...)
```

> macOS / Windows 无此问题，它们默认暴露所需接口。

---

## 2. 核心概念

### 2.1 `struct subprocess_s`

进程句柄。按值声明即可，不需要手动初始化（但建议零初始化）：

```c
struct subprocess_s p;        // 足够
struct subprocess_s p = {0};  // 更保险，便于排查"未创建成功就使用"的问题
```

结构体字段可读取，但**不要手动修改**：

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

### 2.2 选项 `enum subprocess_option_e`

位掩码，通过 `|` 组合传给 `subprocess_create` / `subprocess_create_ex`。

| 选项 | 值 | 说明 |
|---|---|---|
| `subprocess_option_combined_stdout_stderr` | `0x1` | stdout 与 stderr 合并为同一个 `FILE*`。此时 `subprocess_stderr()` 返回 `NULL` |
| `subprocess_option_inherit_environment` | `0x2` | 子进程**继承父进程环境变量**。不加则使用空环境 |
| `subprocess_option_enable_async` | `0x4` | 启用异步读取，才能在 `join` 之前安全调用 `subprocess_read_stdout/stderr` |
| `subprocess_option_no_window` | `0x8` | 尽可能无窗口启动（Windows 上有意义） |
| `subprocess_option_search_user_path` | `0x10` | 在 `PATH` 中搜索程序名。**Windows 上始终启用**。注意：它用的是**启动进程自己的 PATH**，而不是你传入的自定义 `environment` 里的 PATH |
| `subprocess_option_enable_async_no_wait` | `0x20` | 配合 `enable_async`，读不到数据时立即返回 0 而不阻塞 |

### 2.3 错误码 `enum subprocess_error_e`

`subprocess_create` / `subprocess_create_ex` 的返回值；成功恒为 0，失败为负数。

| 枚举值 | 数值 | 含义 |
|---|---|---|
| `subprocess_error_success` | `0` | 成功 |
| `subprocess_error_unknown` | `-1` | 未知错误 |
| `subprocess_error_invalid_options` | `-2` | 选项组合非法 |
| `subprocess_error_invalid_environment` | `-3` | 环境与选项冲突（见下） |
| `subprocess_error_not_found` | `-4` | 找不到程序 |
| `subprocess_error_permission_denied` | `-5` | 权限不足 |
| `subprocess_error_no_memory` | `-6` | 内存不足 |
| `subprocess_error_pipe` | `-7` | 管道创建失败 |
| `subprocess_error_spawn` | `-8` | 进程创建失败 |
| `subprocess_error_not_supported` | `-9` | 平台不支持（例如老 glibc 上指定工作目录） |

拿到错误码后，进一步查看平台原因：POSIX 读 `errno`，Windows 读 `GetLastError()`。

---

## 3. API 参考

### 3.1 创建进程

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

- `command_line`：以 `NULL` 结尾的字符串数组，第一个元素是程序名或路径。这块内存只需在**函数返回前**保持有效。
- `environment`：`"FOO=BAR"` 形式的数组，以 `NULL` 结尾；传 `NULL` 表示**空环境**（除非同时给了 `inherit_environment`）。
- `process_cwd`：子进程工作目录；传 `NULL` 表示继承父进程当前目录。
- 返回值：0 成功，否则为负的错误码。

约束：

- 若 `options` 含 `subprocess_option_inherit_environment`，则 `environment` **必须为 `NULL`**，否则返回 `-3 invalid_environment`。想"继承 + 追加"，得自己读取父进程环境变量再拼进 `environment`。
- Windows 上所有字符串按 **UTF-8** 解释，库内部转为宽字符调用 Unicode API。

### 3.2 获取标准流

```c
subprocess_pure FILE *subprocess_stdin (const struct subprocess_s *const process);
subprocess_pure FILE *subprocess_stdout(const struct subprocess_s *const process);
subprocess_pure FILE *subprocess_stderr(const struct subprocess_s *const process);
```

返回标准 C 的 `FILE*`，可直接用 `fputs` / `fgets` / `fread` 等。

- `subprocess_stdin()` 返回的 `FILE*` 供**父进程写入**以喂给子进程。
- 若进程创建失败（返回值非 0），这三个函数返回 `NULL`。**未检查返回值就使用会直接段错误**——这是本库最常见的崩溃原因。
- 使用 `combined_stdout_stderr` 时，`subprocess_stderr()` 恒返回 `NULL`（已实测）。

### 3.3 等待与销毁

```c
subprocess_weak int subprocess_join(struct subprocess_s *const process,
                                    int *const out_return_code);
subprocess_weak int subprocess_destroy(struct subprocess_s *const process);
subprocess_weak int subprocess_terminate(struct subprocess_s *const process);
subprocess_weak int subprocess_alive(struct subprocess_s *const process);
```

| 函数 | 作用 |
|---|---|
| `subprocess_join` | 阻塞等待进程结束，退出码写入 `out_return_code`（可为 `NULL`）。**调用它会关闭子进程的 stdin 管道** |
| `subprocess_destroy` | 释放句柄与管道。可在进程结束前调用，此时子进程可能比父进程活得更久 |
| `subprocess_terminate` | 强制终止（可能挂起的）进程。之后仍可调用 `join` / `destroy` |
| `subprocess_alive` | 进程是否仍在运行，非 0 表示存活 |

要点：

- `subprocess_join` 之后**不要再写 stdin**；`subprocess_destroy` 之后**不要再读写任何流**。
- `terminate` 之后 `join` 得到的退出码**保证非 0**（实测 `sleep 30` 被终止后退出码为 1）。
- 子进程若因未处理异常终止，退出码同样为非 0。

### 3.4 异步读取

```c
subprocess_weak unsigned
subprocess_read_stdout(struct subprocess_s *const process, char *const buffer,
                       unsigned size);

subprocess_weak unsigned
subprocess_read_stderr(struct subprocess_s *const process, char *const buffer,
                       unsigned size);
```

返回实际读入的字节数。返回 `0` 有两种可能：进程已结束，或启用了 `enable_async_no_wait` 但当前无数据。

**必须在创建时带上 `subprocess_option_enable_async`** 才是受支持的用法。不带该选项时调用属于未定义行为——实测在简单场景下偶尔能读到数据，但绝不能依赖。

---

## 4. 使用样例

以下示例均可直接编译（记得加 `-D_GNU_SOURCE` 或用 `-std=gnu11`）。

### 4.1 最小示例：运行命令并取退出码

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"sh", "-c", "exit 42", NULL};
  struct subprocess_s p = {0};

  int rc = subprocess_create(cmd, subprocess_option_search_user_path, &p);
  if (0 != rc) {                       // 一定要检查！
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

> 不传 `subprocess_option_search_user_path` 时，第一个参数必须是带路径的程序（如 `/bin/sh`、`./mytool`），仅写 `sh` 会返回 `-4 not_found`。

### 4.2 捕获标准输出

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"ls", "-l", NULL};
  struct subprocess_s p = {0};
  if (0 != subprocess_create(cmd, subprocess_option_search_user_path, &p))
    return 1;

  int code = -1;
  subprocess_join(&p, &code);          // 先等结束

  char buf[1024];
  while (fgets(buf, sizeof(buf), subprocess_stdout(&p)))   // 再读
    fputs(buf, stdout);

  subprocess_destroy(&p);
  return code;
}
```

### 4.3 分别捕获 stdout 与 stderr

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

### 4.4 合并 stdout 与 stderr

```c
const char *cmd[] = {"sh", "-c", "echo A; echo B 1>&2", NULL};
struct subprocess_s p = {0};
subprocess_create(cmd,
                  subprocess_option_search_user_path |
                      subprocess_option_combined_stdout_stderr,
                  &p);

int code = -1;
subprocess_join(&p, &code);

/* 注意：此时 subprocess_stderr(&p) == NULL，所有输出都在 stdout 里 */
char buf[256];
while (fgets(buf, sizeof(buf), subprocess_stdout(&p)))
  printf("%s", buf);          // 依次输出 A 和 B

subprocess_destroy(&p);
```

### 4.5 写入标准输入

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
  fflush(in);                 // 必须 flush

  int code = -1;
  subprocess_join(&p, &code); // join 会关闭 stdin 管道，cat 才会结束

  char buf[128] = {0};
  if (fgets(buf, sizeof(buf), subprocess_stdout(&p)))
    printf("echo back: %s", buf);

  subprocess_destroy(&p);
  return 0;
}
```

关键点：像 `cat`、`grep`、`sort` 这类读取 stdin 直到 EOF 才退出的程序，**必须靠 `fflush` + `subprocess_join`（关闭管道）** 才能让它结束。

### 4.6 异步读取（处理大量输出的正确姿势）

这是最重要的一个样例。**当子进程输出超过管道缓冲区（Linux 默认 64 KiB）时，如果先 `join` 再读，会死锁**——子进程阻塞在写管道，父进程阻塞在等它退出。实测输出约 820 KB 时程序永久挂住。

正确做法：创建时加 `enable_async`，边读边等。

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  /* 产生约 820KB 输出 */
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
      if (!subprocess_alive(&p)) break;   // 进程已结束且无残留数据
      continue;                            // 还活着，继续读
    }
    total += n;
    /* 这里可以处理 buf[0..n) */
  }

  int code = -1;
  subprocess_join(&p, &code);
  printf("drained %u bytes, exit=%d\n", total, code);

  subprocess_destroy(&p);
  return 0;
}
```

上面的 `continue` 是忙等（busy loop）。生产代码建议改成短暂休眠或使用 `enable_async_no_wait` 配合轮询，见 [5.2](#52-忙等与-cpu-占用)。

### 4.7 自定义环境变量与工作目录

```c
#include "subprocess.h"
#include <stdio.h>

int main(void) {
  const char *cmd[] = {"sh", "-c", "echo FOO=$FOO", NULL};
  const char *env[] = {"FOO=BAR", "PATH=/usr/bin:/bin", NULL};
  struct subprocess_s p = {0};

  int rc = subprocess_create_ex(cmd,
                                subprocess_option_search_user_path,
                                env,       /* 完全替换环境 */
                                "/tmp",    /* 子进程工作目录，NULL 表示继承 */
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

**`environment` 是完全替换，不是追加。** 想继承父进程环境，改用：

```c
subprocess_create(cmd, subprocess_option_inherit_environment |
                           subprocess_option_search_user_path, &p);
```

### 4.8 带超时的运行

库没有内置超时，用 `subprocess_alive` 轮询 + `subprocess_terminate` 实现：

```c
#include "subprocess.h"
#include <time.h>

static void sleep_ms(int ms) {
  struct timespec ts = {ms / 1000, (long)(ms % 1000) * 1000000L};
  nanosleep(&ts, NULL);
}

/* 0 = 正常退出；-1 = 创建失败；-2 = 超时被终止 */
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

实测：对 `sleep 10` 设 500 ms 超时，返回 `-2`，`join` 得到的退出码为 1（非 0）。

### 4.9 C++ 中使用（RAII 封装）

头文件自带 `extern "C"`，C++ 可直接 include：

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
    return code;   // 句柄回收交给析构函数
  }

private:
  std::vector<std::string> args_;
  std::vector<const char *> argv_;
  subprocess_s proc_{};
  bool started_ = false;
};

// 用法：
//   Process p({"echo", "hi"});
//   std::cout << p.readAllStdout();
//   int code = p.wait();
```

编译：`g++ -std=c++17 -D_GNU_SOURCE main.cpp`。

---

## 5. 注意事项

### 5.1 必须检查 `subprocess_create` 的返回值

库**不会**在失败时把 `FILE*` 置为某个安全值——失败时三个流访问函数返回 `NULL`。对 `NULL` 调用 `fgets`/`fputs` 会立刻段错误。

本项目实测中，下面这行代码就直接崩了：

```c
subprocess_create(cmd, 0, &p);            // 没加 search_user_path，返回 -4
fgets(buf, sizeof(buf), subprocess_stdout(&p));   // stdout_file == NULL → SIGSEGV
```

养成习惯：**每次 create 后先判返回值**。

### 5.2 忙等与 CPU 占用

`subprocess_read_stdout` 在没数据时（非 `no_wait` 模式）会阻塞；在 `enable_async_no_wait` 模式下立即返回 0。4.6 节那种 `continue` 轮询属于忙等，会打满一个 CPU 核。

改进方式：

- 循环中插入 `nanosleep`（见 4.8 的 `sleep_ms`）；
- 或者用 `subprocess_option_enable_async_no_wait` 配合休眠轮询，实测该模式下无数据立即返回 0、不阻塞。

### 5.3 死锁：先 join 后读大输出

再次强调，这是本库最容易踩、且症状最隐蔽（"程序莫名其妙卡住"）的问题：

| 场景 | 结果（实测，输出约 820 KB） |
|---|---|
| 先 `subprocess_join`，再读 stdout | **永久挂起**（超时后被 kill） |
| `enable_async` + 边读边等 | 正常读完 820000 字节，退出码 0 |

规律：**只要子进程输出量可能超过管道缓冲区，就必须用 `enable_async` 边读边等。** 反之，如果确认输出很小（几 KB），先 join 再读是安全的，4.2～4.5 的样例都是这个模式。

### 5.4 环境变量默认是"空"，不是"继承"

这是与 Windows `CreateProcessA` 行为相反的设计，官方解释是"用最安全的默认值"：

```c
subprocess_create(cmd, 0, &p);                    // 空环境
subprocess_create(cmd, inherit_environment, &p);  // 继承父进程环境
```

实测在空环境下运行 `/bin/sh -c 'echo HOME=[$HOME]'` 得到 `HOME=[]`，证明确实是空环境。

但有个**容易误判的陷阱**：空环境下 `echo $PATH` 仍会输出一长串路径（如 `/usr/local/sbin:/usr/local/bin:...`）。那是 **shell 自身在 PATH 未设置时启用的默认路径**，不代表继承了父进程环境。判断依据应看 `HOME` 这类非 shell 兜底的变量。

另外，需要访问网络的子进程通常要加 `inherit_environment`，因为环境里隐含了父进程的凭据。

### 5.5 `search_user_path` 用的是父进程的 PATH

即使你通过 `subprocess_create_ex` 传了自定义 `environment`（里面含 `PATH=...`），`search_user_path` 查找程序时用的仍是**创建者进程自己的 PATH**。想让程序从自定义 PATH 里被找到，得自己拼绝对路径。

### 5.6 进程清理与僵尸进程

- 正常路径：`create` → （读写）→ `join` → `destroy`。
- `join` 会等待子进程退出并回收，避免僵尸进程。
- 只调用 `destroy` 而不 `join`，子进程可能继续运行甚至比父进程活得久（官方明确支持这种"放出孤儿进程"的用法）。
- 终止顺序建议：`terminate` → `join`（拿到非 0 退出码）→ `destroy`。

### 5.7 Windows 相关

- Windows 上 `search_user_path` **始终启用**，无需显式指定。
- 命令行字符串按 **UTF-8** 解释，内部转 Unicode API 调用——传入 UTF-8 路径/参数是安全的，但要保证传入的确实是 UTF-8。
- `subprocess_option_no_window` 目前只在 Windows 上有实际效果。
- `terminate` 只杀目标进程本身，不会连带孙进程；要杀整个进程树需要额外的 Job Object 机制，本库不提供。

### 5.8 线程安全

单个 `subprocess_s` 句柄**不应被多个线程并发操作**。一个常见且安全的分工是：一个线程读 stdout、另一个线程读 stderr，但不要两个线程同时 `join` 同一个进程。

### 5.9 POSIX 信号

官方建议在使用本库的程序中忽略 `SIGPIPE`——向已关闭的管道写入会触发它，默认行为是直接终止你的进程。

```c
signal(SIGPIPE, SIG_IGN);
```

---

## 6. 编译期配置宏

这些宏可在 include 之前自行定义，用来覆盖库的平台探测结果。一般不需要动。

| 宏 | 作用 |
|---|---|
| `SUBPROCESS_HAVE_CWD` | `subprocess_create_ex` 能否支持 `process_cwd`。不支持时为 0，此时传非 NULL 工作目录返回 `-9 not_supported`、`errno` 为 `ENOSYS`。受影响平台：glibc < 2.29、macOS < 10.15、iOS/tvOS/watchOS。musl < 1.1.24 需要手动定义覆盖 |
| `SUBPROCESS_SPAWN_VIA_FORK` | 用 `fork()+exec()` 代替 `posix_spawn()`。AIX、OpenBSD、NetBSD < 10 上自动启用 |
| `SUBPROCESS_ADDCHDIR_IS_POSIX` | 平台提供的是标准 `posix_spawn_file_actions_addchdir` 还是 `_np` 变体 |
| `SUBPROCESS_SPAWN_REPORTS_EXEC_ERRORS` | `exec` 失败能否回报给调用方 |

---

## 7. 常见问题

**Q：为什么子进程拿不到父进程的环境变量？**
A：默认就是空环境。加 `subprocess_option_inherit_environment`。

**Q：为什么提示找不到程序？**
A：程序名不在 PATH 中，且没加 `subprocess_option_search_user_path`。要么加该选项，要么写绝对路径。

**Q：为什么 `inherit_environment` 和自定义 `environment` 一起用会报错？**
A：官方明确禁止，会返回 `-3 invalid_environment`。需要"继承 + 追加"就自己读父进程环境变量拼进去。

**Q：程序卡住不动了？**
A：99% 是 [5.3](#53-死锁先-join-后读大输出) 的死锁。改用 `enable_async` 边读边等。

**Q：能拿到子进程的 PID 吗？**
A：POSIX 下可读 `process.child`（`pid_t`），Windows 下可读 `process.hProcess`。但这是内部字段，不保证未来兼容。

**Q：多个源文件都 include 会不会重复定义？**
A：不会。GCC/Clang 用 weak 符号，MSVC 用 inline。已实测双 TU 链接通过。

---

## 8. 速查表

### 典型用法对应的选项组合

| 需求 | 选项 |
|---|---|
| 用程序名（非绝对路径）启动 | `search_user_path` |
| 子进程要继承环境变量 | `inherit_environment` |
| 输出可能很大 / 需要实时读取 | `enable_async` |
| 想要非阻塞读取 | `enable_async` \| `enable_async_no_wait` |
| 不分流，stdout+stderr 一起看 | `combined_stdout_stderr` |
| Windows 上不要弹窗口 | `no_window` |

### 生命周期调用顺序

```
subprocess_create[_ex]
        │
        ├─（可选）subprocess_stdin  → fputs → fflush
        │
        ├─ 输出小：subprocess_join → fgets(stdout/stderr)
        │
        ├─ 输出大：subprocess_read_stdout 循环 → subprocess_join
        │
        └─（可选）subprocess_alive 轮询 → subprocess_terminate
        │
subprocess_join（若尚未调用）
        │
subprocess_destroy
```

### 返回值检查清单

- [ ] `subprocess_create` / `create_ex` 返回 0？
- [ ] 使用 `subprocess_stdin/stdout/stderr` 前确认进程创建成功？
- [ ] `combined_stdout_stderr` 下不再使用 `subprocess_stderr`（它为 NULL）？
- [ ] `join` / `destroy` 之后没有再读写任何流？
- [ ] 每个成功创建的进程最终都走到了 `join`（或明确接受孤儿进程）与 `destroy`？

---

## 参考

- 仓库：<https://github.com/sheredom/subprocess.h>
- 许可：Unlicense（公共领域）
