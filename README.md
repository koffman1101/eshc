# eshc — Enhanced Shell Script Compiler (增强型 Shell 脚本编译器)

**eshc** is a security-hardened fork of the classic `shc` (Shell Script Compiler) for modern Linux production environments. It eliminates plaintext source exposure in process lists (`ps aux`, `systemd`) and breaks through the 128KB script size limit while preserving interactive `stdin`, `$0` process arguments, and cross-architecture compatibility.

**eshc** 是经典工具 `shc` 针对现代 Linux 生产环境深度安全加固的增强分支。它彻底解决了原版 `shc` 在进程列表（`ps aux`、`systemd`）中暴露源码明文以及 128KB 脚本体积上限的两大痛点，同时完整保留了交互式标准输入（`stdin`）、`$0` 进程名以及跨架构 Linux 环境的兼容性。

---

## 📖 Background & Motivation | 分支起因

This fork was created to resolve two critical issues encountered when using vanilla `shc` in real-world production deployments:

创建本增强分支源于在实际生产环境使用原版 `shc` 时发现的两个致命问题：

### 1. Plaintext Exposure in `systemd` / Process Lists | 源码明文在 `systemd` 列表中完全暴露
Vanilla `shc` invokes the shell via `bash -c "<script_content>"`, causing the entire decrypted script source to be publicly visible in `systemctl status`, `ps aux`, and `/proc/<PID>/cmdline`.

原版 `shc` 通过 `bash -c "<脚本内容>"` 方式拉起解释器，导致解密后的完整脚本源码直接明文显示在 `systemctl status`、`ps aux` 以及 `/proc/<PID>/cmdline` 中，加密形同虚设。

### 2. Inability to Execute Scripts Larger than 128KB | 无法运行大于 `128KB` 的脚本
Because Linux restricts the maximum length of a single command-line argument (`MAX_ARG_STRLEN`, typically `128KB` / `32` memory pages), any encrypted script exceeding `128KB` immediately fails at runtime with `Argument list too long` (`E2BIG`).

受限于 Linux 内核对单个命令行参数最大长度的硬性限制（`MAX_ARG_STRLEN`，通常为 `128KB`），一旦原始脚本体积超过 `128KB`，运行加密程序时就会直接触发 `Argument list too long`（`E2BIG`）错误而崩溃退出。

---

## 🔒 Core Security Enhancements | 核心增强特性

### 1. Anonymous Pipe Execution (`pipe` + `bash -s`) | 匿名管道流式加载
Replaces `execvp(shll, "-c", text)` with an anonymous kernel pipe stream, eliminating both the `128KB` `MAX_ARG_STRLEN` size limit and plaintext exposure in `systemd`, `ps aux`, and `/proc/<PID>/cmdline`.

使用内核匿名管道替代危险的 `-c` 命令行参数传输脚本，既彻底突破了 `128KB` 命令行参数长度上限，又实现了进程列表与 `systemd` 状态零暴露。

### 2. Anti-Dump & Memory Erasure (`prctl` + `memset`) | 防 Core Dump 与内存即时擦除
Enforces `prctl(PR_SET_DUMPABLE, 0)` (in traceable mode) to block core dumps and non-root `ptrace` attachment, and immediately zeroes out decrypted plaintext buffers (`memset`) after piping.

在非 `-U` 追踪模式下默认启用内核级 `PR_SET_DUMPABLE` 禁止核心转储及非 root 进程调试附加，并在管道传输完成后立即擦除内存中的明文缓冲区。

### 3. Zero-Conflict Stdin & `$0` Preservation (`fd 199` + `BASH_ARGV0`) | 高位描述符还原输入与 `$0`
Backs up original `stdin` to high file descriptor `199` to avoid colliding with common business script descriptors (such as `flock` FDs `3–100`), wraps the payload in a `{ ... }` compound block so the interpreter reads the complete stream before executing, and restores `$0` via `BASH_ARGV0`.

通过高位描述符 `fd 199` 无损备份还原交互式标准输入（避免与业务脚本的 `flock` 句柄冲突），通过 `{ ... }` 代码块确保解释器完整读取管道后再执行，并通过 `BASH_ARGV0` 精准还原 `$0`。

### 4. Cross-Architecture & Process Tree Alignment | 跨架构兼容与进程树对齐
Uses pure POSIX syscalls (`pipe`, `fork`, `dup2`, `execvp`), working seamlessly across **x86-64** and **ARM / Raspberry Pi (32/64-bit)** while keeping the parent PID and process tree depth consistent with vanilla `shc`.

采用纯标准 POSIX 系统调用，无缝兼容 **x86-64** 与 **ARM / 树莓派（32/64位）**，同时保持父进程 PID 与进程树层级与原版完全一致。

---

## 🛠️ Build & Installation | 编译与安装

### Static Build (Recommended / 推荐静态编译，零目标机动态库依赖)

```bash
git clone https://github.com
cd eshc
./autogen.sh
./configure
make CFLAGS="-static -O2"
sudo make install
```

---

## 🚀 Usage & Deployment | 使用与部署示例

### 1. Encrypting a Script | 加密脚本

Compile a shell script into a hardened standalone binary (`-r` allows redistribution across identical architectures; `PR_SET_DUMPABLE` protection is enabled by default):

将脚本编译为加固后的独立二进制文件（`-r` 支持跨同架构机器分发，默认已内置 `PR_SET_DUMPABLE` 内核级防转储保护）：

```bash
shc -r -f your_script.sh -o your_app
```

> **Note on `-U` (Untraceable) Flag | 关于 `-U` 参数的说明**：  
> On modern Linux distributions with Yama `kernel.yama.ptrace_scope = 1` (e.g., Ubuntu), vanilla `-U` child-to-parent `ptrace` attachment requires `root` privileges or it will terminate with `Operation not permitted`. In normal user environments, omit `-U` and rely on `eshc`'s built-in `PR_SET_DUMPABLE` protection.

> 在开启了 Yama `ptrace_scope = 1` 的现代 Linux 发行版（如 Ubuntu）中，原版 `-U` 参数的子进程反向 `ptrace` 父进程机制需要 `root` 权限才能运行，否则会触发 `Operation not permitted` 退出。普通用户环境下无需加 `-U`，直接依赖 `eshc` 默认内置的 `PR_SET_DUMPABLE` 防护即可。

### 2. Recommended `systemd` Service Hardening | 推荐服务加固配置

For production daemons, pair `eshc` with the following `systemd` directives to further block external `ptrace` attachment and core dumps:

生产环境下部署守护进程时，建议在 `.service` 单元中加入以下指令以配合封堵 `ptrace` 调试与核心转储：

```ini
[Service]
ExecStart=/usr/local/bin/your_app
LimitCORE=0
CapabilityBoundingSet=~CAP_SYS_PTRACE
SystemCallFilter=~@debug
```

---

## ⚠️ Limitations & Requirements | 缺陷与环境要求

### 1. Interpreter Version Requirement (`Bash 5.0+` Recommended) | 解释器版本要求
While the anonymous pipe mechanism works across standard `sh` and `bash`, full compatibility with `\$0` process/script name restoration (`BASH_ARGV0`) and high file descriptor redirection (`fd 199`) requires **Bash 5.0 or above**. On minimal POSIX shells (e.g., `dash`, BusyBox `ash`, or Bash < 5.0), scripts still execute normally, but `\$0` will display the interpreter path rather than the binary name, and minimal shells may not support FDs > 9.

虽然匿名管道机制本身兼容常规 `sh` 与 `bash`，但要完整发挥所有加固特性（尤其是通过 `BASH_ARGV0` 精准还原 `\$0` 进程/脚本名，以及高位文件描述符 `fd 199` 重定向），**目标运行环境建议使用 Bash 5.0 及以上版本**。在精简版 POSIX Shell（如 `dash`、BusyBox `ash` 或 Bash 4.x 及更早版本）下，脚本仍可正常执行，但 `\$0` 将显示为解释器路径而非原始二进制名，且极简 Shell 可能不支持超过 `9` 的文件描述符。

### 2. Caller Identity Verification | 父进程身份校验方式
Scripts verifying whether they were launched by `systemd` or a specific parent service should traverse ancestor PIDs up to `PID 1` or inspect `/proc/\[/cgroup` rather than hardcoding fixed grandparent PID checks.  

若业务脚本需要校验是否由 `systemd` 或特定守护进程拉起，建议向上追溯祖先进程链直至 `PID 1` 并结合 `/proc/\]/cgroup` 校验，而非写死固定层级的父/祖父进程判断。

### 3. OS Kernel Dependency | 操作系统限制
`prctl(PR_SET_DUMPABLE, 0)` is Linux-specific; non-Linux Unix systems automatically skip this syscall while retaining anonymous pipe execution protection.

`prctl(PR_SET_DUMPABLE, 0)` 仅在 Linux 内核下生效，其他类 Unix 系统会自动跳过该系统调用，但仍保留匿名管道防 `ps` 泄露保护。

---

## 🙏 Acknowledgments | 致谢

Special thanks to **Francisco Javier Rosales García** (the original creator of `shc`) and **neurobin** (`neurobin/shc`) along with all open-source community contributors for their foundational work, maintenance, and inspiration that made this security enhancement possible.

特别感谢原版 `shc` 的创始人 **Francisco Javier Rosales García**、长期维护者 **neurobin**（`neurobin/shc`）以及所有开源社区贡献者的前期工作与长期维护，为本安全增强分支奠定了坚实的基础。

---

## 📜 License | 开源协议

Based on original `shc`, distributed under the **GNU General Public License v3.0 (GPLv3)**.

