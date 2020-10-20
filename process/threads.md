Linux 中进程和线程使用相同的 task_struct 结构。区别仅在于线程创建时标志位不同，可以和其他线程共享资源。


Linux 创建线程时调用 clone(CLONE_VM|CLONE_FS|CLONE_FILES|SIGHAND, 0)

创建线程时调用 clone(SIGHAND, 0)

CLONE_VM 代表父子进程共享进程空间
CLONE_FS 代表父子进程共享文件系统信息
CLONE_FILES 共享打开的文件


thread 默认在遍历 proc 时不可见。


    /proc/[tid] subdirectories
            Each one of these subdirectories contains files and subdirec‐
            tories exposing information about the thread with the corre‐
            sponding thread ID.  The contents of these directories are the
            same as the corresponding /proc/[pid]/task/[tid] directories.

            The /proc/[tid] subdirectories are not visible when iterating
            through /proc with getdents(2) (and thus are not visible when
            one uses ls(1) to view the contents of /proc).