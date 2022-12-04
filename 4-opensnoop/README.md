# eBPF 入门开发实践指南四：捕获进程打开文件的系统调用集合，使用全局变量在 eBPF 中过滤进程 pid

eBPF (Extended Berkeley Packet Filter) 是 Linux 内核上的一个强大的网络和性能分析工具，它允许开发者在内核运行时动态加载、更新和运行用户定义的代码。

本文是 eBPF 入门开发实践指南的第四篇，主要介绍如何捕获进程打开文件的系统调用集合，并使用全局变量在 eBPF 中过滤进程 pid。

## 在 eBPF 中捕获进程打开文件的系统调用集合

首先，我们需要编写一段 eBPF 程序来捕获进程打开文件的系统调用，具体实现如下：

```c
// SPDX-License-Identifier: GPL-2.0
// Copyright (c) 2019 Facebook
// Copyright (c) 2020 Netflix
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>
#include "opensnoop.h"


/// Process ID to trace
const volatile int pid_target = 0;

SEC("tracepoint/syscalls/sys_enter_open")
int tracepoint__syscalls__sys_enter_open(struct trace_event_raw_sys_enter* ctx)
{
	u64 id = bpf_get_current_pid_tgid();
	u32 pid = id;

	if (pid_target && pid_target != pid)
		return false;
	// Use bpf_printk to print the process information
	bpf_printk("Process ID: %d enter sys open\n", pid);
	return 0;
}

SEC("tracepoint/syscalls/sys_enter_openat")
int tracepoint__syscalls__sys_enter_openat(struct trace_event_raw_sys_enter* ctx)
{
	u64 id = bpf_get_current_pid_tgid();
	u32 pid = id;

	if (pid_target && pid_target != pid)
		return false;
	// Use bpf_printk to print the process information
	bpf_printk("Process ID: %d enter sys openat\n", pid);
	return 0;
}

/// Trace open family syscalls.
char LICENSE[] SEC("license") = "GPL";
```

上面的 eBPF 程序通过定义两个函数 tracepoint__syscalls__sys_enter_open 和 tracepoint__syscalls__sys_enter_openat 并使用 SEC 宏把它们附加到 sys_enter_open 和 sys_enter_openat 两个 tracepoint（即在进入 open 和 openat 系统调用时执行）。这两个函数通过使用 bpf_get_current_pid_tgid 函数获取调用 open 或 openat 系统调用的进程 ID，并使用 bpf_printk 函数在内核日志中打印出来。


编译运行上述代码：

```console
$ ecc fentry-link.bpf.c
Compiling bpf object...
Packing ebpf object and config into package.json...
$ sudo ecli package.json
Runing eBPF program...
```

运行这段程序后，可以通过查看 /sys/kernel/debug/tracing/trace_pipe 文件来查看 eBPF 程序的输出：

```console
$ sudo cat /sys/kernel/debug/tracing/trace_pipe
           <...>-3840345 [010] d... 3220701.101143: bpf_trace_printk: Process ID: 3840345 enter sys open
           <...>-3840345 [010] d... 3220701.101179: bpf_trace_printk: Process ID: 3840345 enter sys openat
           <...>-3840345 [010] d... 3220702.157967: bpf_trace_printk: Process ID: 3840345 enter sys open
           <...>-3840345 [010] d... 3220702.158000: bpf_trace_printk: Process ID: 3840345 enter sys openat
```

此时，我们已经能够捕获进程打开文件的系统调用了。

## 使用全局变量在 eBPF 中过滤进程 pid

在上面的程序中，我们定义了一个全局变量 pid_target 来指定要捕获的进程的 pid。在 tracepoint__syscalls__sys_enter_open 和 tracepoint__syscalls__sys_enter_openat 函数中，我们可以使用这个全局变量来过滤输出，只输出指定的进程的信息。

可以通过执行 ecli -h 命令来查看 opensnoop 的帮助信息：

```c

```