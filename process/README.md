# linux process

https://unix.stackexchange.com/questions/18166/what-are-session-leaders-in-ps

https://www.win.tue.nl/~aeb/linux/lk/lk-10.html

90

In Linux, every process has several IDs associated with it, including:

Process ID (PID)

This is an arbitrary number identifying the process. Every process has a unique ID, but after the process exits and the parent process has retrieved the exit status, the process ID is freed to be reused by a new process.

Parent Process ID (PPID)

This is just the PID of the process that started the process in question.

Process Group ID (PGID)

This is just the PID of the process group leader. If PID == PGID, then this process is a process group leader.

Session ID (SID)

This is just the PID of the session leader. If PID == SID, then this process is a session leader.

Sessions and process groups are just ways to treat a number of related processes as a unit. All the members of a process group always belong to the same session, but a session may have multiple process groups.

tty 分为终端和伪终端（pseudoterminal），伪终端可以同时处理终端login以及网络 login

## session group

![apue-会话组](/imgs/apue-sessions-1.png)

参考 apue 第九章“会话"

session 是一个或多个进程组的集合。 shell 管道会把多个进程编成一组，如 `proc1 | proc2&`

进程调用 `pid_t setsid(void)` 函数可以建立一个新的会话。如果调用进程不是进程组的组长，则此函数会创建一个新的会话
1. 该函数会变成会话首进程
2. 该进程成功新进程组的组长
3. 该进程没有控制终端

如果调用 setsid 的进程已经是进程组组长则会返回错误。

## 控制终端

网络登录 (如 sshd ) 对应的为 pts


## Example
`ps  xfao pid,ppid,pgid,sid,tty,comm` 显示进程树的同事会显示 sid 和 控制终端 tty

```
PID    PPID PGID   SID   TTY      COMMAND
1134     1  1134  1134  ?         crond
21604  1134  1134  1134 ?         \_ crond
21606 21604 21606 21606 ?         |   \_ run-parts
21629 21606 21606 21606 ?         |       \_ check_ntp_statu
21632 21629 21606 21606 ?         |       |   \_ sleep
21630 21606 21606 21606 ?         |       \_ awk
23627  1134  1134  1134 ?         \_ crond
23631 23627 23631 23631 ?         |   \_ sh
23634 23631 23631 23631 ?         |       \_ puppetrun.sh
23637 23634 23631 23631 ?         |           \_ sleep
29767  1134  1134  1134 ?         \_ crond
29769 29767 29769 29769 ?             \_ sadc
 1144     1  1144  1144 ?        atd
 1149     1  1149  1149 tty1     agetty
 1150     1  1150  1150 ttyS0    agetty
 3841     1  3840  3840 ?        sgagent
12807     1 12807 12807 ?        sshd
13167 12807 13167 13167 ?         \_ sshd
13169 13167 13167 13167 ?             \_ sshd
13170 13169 13170 13170 pts/0             \_ bash
30064 13170 30064 13170 pts/0                 \_ ps
10270     1 10270 10270 ?        bash
11036 10270 10270 10270 ?         \_ bash
11039 11036 11039 10270 ?             \_ stat
```

## 监控方法


* /proc
* cn_proc
* kprobe
* ebpf


## /proc

https://man7.org/linux/man-pages/man5/proc.5.html

###  /proc/[pid]/stat

Status information about the process.  This is used by ps(1).
It is defined in the kernel source file fs/proc/array.c.

```
cat /proc/self/stat
768 (cat) R 377 768 376 1025 0 0 0 0 0 0 0 1 0 0 20 0 1 0 11758533 187742060544 224 18446744073709551615 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
```

前面主要的字段含义如下

* (1) pid  %d The process ID.

* (2) comm  %s

    The filename of the executable, in parentheses. 
    Strings longer than TASK_COMM_LEN (16) characters
    (including the terminating null byte) are silently
    truncated.  This is visible whether or not the exe‐
    cutable is swapped out.
    执行文件的文件名，最长16字节包括\0，文件是否被删除不会影响该字段

* (3) state  %c
        One of the following characters, indicating process
        state:

        R  Running

        S  Sleeping in an interruptible wait

        D  Waiting in uninterruptible disk sleep

        Z  Zombie

        T  Stopped (on a signal) or (before Linux 2.6.33)
            trace stopped

        t  Tracing stop (Linux 2.6.33 onward)

        W  Paging (only before Linux 2.6.0)

        X  Dead (from Linux 2.6.0 onward)

        x  Dead (Linux 2.6.33 to 3.13 only)

        K  Wakekill (Linux 2.6.33 to 3.13 only)

        W  Waking (Linux 2.6.33 to 3.13 only)

        P  Parked (Linux 3.9 to 3.13 only)

* (4) ppid  %d The PID of the parent of this process.

* (5) pgrp  %d The process group ID of the process.

* (6) session  %d The session ID of the process.

* (7) tty_nr  %d
        
        The controlling terminal of the process.  (The minor
        device number is contained in the combination of
        bits 31 to 20 and 7 to 0; the major device number is
        in bits 15 to 8.)
        sshd 及子进程tty值通常为 pts/0，参考 man pts

* (8) tpgid  %d
        
        The ID of the foreground process group of the con‐
        trolling terminal of the process.

## cn_proc

initially added by IBM kernel 2.6.14 

https://github.com/ggrandes-clones/pmon/blob/master/README.md

https://www.slideshare.net/kerneltlv/kernel-proc-connector-and-containers

https://lwn.net/Articles/157150/

https://www.kernel.org/doc/Documentation/connector/connector.txt


通过netlink从内核获取 fork exec exit 的进程 pid。在高负载频繁创建pid情况下（如k8s机器）拿到 pid 或无法及时从 /proc 获取进程信息

```
switch (nlcn_msg.proc_ev.what) {
    case PROC_EVENT_NONE: {
        printf("set mcast listen ok\n");
        break;
    }
    case PROC_EVENT_FORK: {
        printf("fork: parent tid=%d pid=%d -> child tid=%d pid=%d\n",
        nlcn_msg.proc_ev.event_data.fork.parent_pid,
        nlcn_msg.proc_ev.event_data.fork.parent_tgid,
        nlcn_msg.proc_ev.event_data.fork.child_pid,
        nlcn_msg.proc_ev.event_data.fork.child_tgid);
        break;
    }
    case PROC_EVENT_EXEC: {
        printf("exec: tid=%d pid=%d\n",
        nlcn_msg.proc_ev.event_data.exec.process_pid,
        nlcn_msg.proc_ev.event_data.exec.process_tgid);
        break;
    }
    case PROC_EVENT_UID: {
        printf("uid change: tid=%d pid=%d from %d to %d\n",
        nlcn_msg.proc_ev.event_data.id.process_pid,
        nlcn_msg.proc_ev.event_data.id.process_tgid,
        nlcn_msg.proc_ev.event_data.id.r.ruid,
        nlcn_msg.proc_ev.event_data.id.e.euid);
        break;
    }
    case PROC_EVENT_GID: {
        printf("gid change: tid=%d pid=%d from %d to %d\n",
        nlcn_msg.proc_ev.event_data.id.process_pid,
        nlcn_msg.proc_ev.event_data.id.process_tgid,
        nlcn_msg.proc_ev.event_data.id.r.rgid,
        nlcn_msg.proc_ev.event_data.id.e.egid);
        break;
    }
    case PROC_EVENT_EXIT: {
        printf("exit: tid=%d pid=%d exit_code=%d\n",
        nlcn_msg.proc_ev.event_data.exit.process_pid,
        nlcn_msg.proc_ev.event_data.exit.process_tgid,
        nlcn_msg.proc_ev.event_data.exit.exit_code);
        break;
    }
    default: {
        printf("unhandled proc event\n");
        break;
    }
}
```

可以监控 fork exec exit

## 开源实现

https://github.com/htop-dev/htop

### htop

https://github.com/htop-dev/htop/blob/master/linux/LinuxProcessList.c#L189

`ProcessList_new`

初始化 netlink 以及程序结构

`ProcessList_goThroughEntries` 开始获取进程信息，调用 `LinuxProcessList_recurseProcTree` 递归遍历 proc 并调用 `LinuxProcessList_readStatFile` 从 `/proc/<pid>/stat` 文件中解析出进程各种状态，解析后调用 `ProcessList_add` 将进程加入到 proc list 结构中。

