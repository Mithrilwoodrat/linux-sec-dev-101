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

## session group

session 是一个或多个进程组的集合。 shell 管道会把多个进程编成一组，如 `proc1 | proc2&`


## 监控方法


* /proc
* cn_proc
* kprobe
* ebpf


## cn_proc

initially added by IBM kernel 2.6.14 

https://github.com/ggrandes-clones/pmon/blob/master/README.md

https://www.slideshare.net/kerneltlv/kernel-proc-connector-and-containers

https://lwn.net/Articles/157150/

https://www.kernel.org/doc/Documentation/connector/connector.txt


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

