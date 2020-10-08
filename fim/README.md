## File Integrity Monitoring 文件完整性监控

监控文件是否被篡改，以及被异常访问。包括文件监控和目录监控两种模式

## 实现方式

| 实现方式 	| 优点| 缺点 |
|----------------	|---------------------------------------	|----------------------------------------	|
| file hash | 实现简单 | 非实时 |
| fanotify  | 功能比inotify 更多，可以获取 process info | 在低版本内核(如3.10)功能缺失，会消耗内核内存，需要小心设置 |
| inotify   | 可以实时获文件读写信息 | 回调种没有进程信息(遍历 proc 有几率可以得到)，不支持递归监控目录 |
| syscall hook | 定制性高 | 实现复杂，风险高 |
| auditd| 监控全面 | 性能损耗比较严重，部署和使用成本高 |
| kprobe ebpf | 定制性高，风险可控 | 实现复杂，需要高版本内核支持 |

设置 inotify、fanotify 内存限制参考 osquery 文档 [file-integrity-monitoring.md](https://github.com/osquery/osquery/blob/b68d44546427e020f708935652cf5837823204fd/docs/wiki/deployment/file-integrity-monitoring.md)

## 开源实现

### osquery

osquery 支持目录和文件的监控，并且支持使用 wilchard 通配符设置路径。参考 [配置文档](https://osquery.readthedocs.io/en/stable/deployment/file-integrity-monitoring/),以及 [源码中的文档](https://github.com/osquery/osquery/blob/b68d44546427e020f708935652cf5837823204fd/docs/wiki/deployment/file-integrity-monitoring.md)

对应实现:

 * inotify  [/osquery/events/linux/inotify.h](https://github.com/osquery/osquery/blob/29f4694df2/osquery/events/linux/inotify.h) osquery 文档中说 fim 是由 inotify 实现，属于 file_events。

 * auditd [/osquery/tables/events/linux/process_file_events.h](https://github.com/osquery/osquery/blob/29f4694df214bc3bd4e7210873e05bb19374888b/osquery/tables/events/linux/process_file_events.h#L181) process_events 同样也会有文件读写记录，从安全监控角度看这块数据更全更有价值。


## inotify




## fanotify

The fanotify API was introduced in version 2.6.36 of the Linux kernel and enabled in version 2.6.37.  Fdinfo support was added in version 3.8.

fanotify 支持的事件类型参考 [fanotify_mark](https://man7.org/linux/man-pages/man2/fanotify_mark.2.html)

| Mask Name | SysCall | Kernel Version |
|----------------	|---------------------------------------	|----------------------------------------	|
| FAN_ACCESS | stat | > 3.10 |
| FAN_MODIFY  | mmap or open and write | > 3.10 |
| FAN_CLOSE_WRITE | open with O_WRONLY flag and close | > 3.10 |
| FAN_CLOSE_NOWRITE | open & close | > 3.10 |
| FAN_OPEN | open | > 3.10 |
| FAN_OPEN_EXEC  | execve? haven't tested yet | > 5.0 |
| FAN_ATTRIB | haven't tested yet | > 5.1 |
| FAN_CREATE | open with O_CREAT | > 5.1 |
| FAN_DELETE | unlinkat unlink rmdir | > 5.1 |
| FAN_DELETE_SELF | unlinkat unlink rmdir | > 5.1 |
| FAN_MOVED_FROM | rename | > 5.1 |
| FAN_MOVED_TO  | rename | > 5.1 |
| FAN_MOVE_SELF | rename | > 5.1 |
| FAN_OPEN_PERM | haven't tested yet | > 5.1 |
| FAN_OPEN_EXEC_PERM  | haven't tested yet | > 5.1 |
| FAN_ACCESS_PERM | haven't tested yet | > 5.1 |
| FAN_ONDIR | Create events for directorie |  > 3.10 |
| FAN_EVENT_ON_CHILD |  Events for the immediate children of marked directories shall be created. | > 3.10 |

特殊标记 FAN_EVENT_ON_CHILD

~~~
Events for the immediate children of marked directories shall
be created.  The flag has no effect when marking mounts and
filesystems.  Note that events are not generated for children
of the subdirectories of marked directories.  More
specifically, the directory entry modification events
FAN_CREATE, FAN_DELETE, FAN_MOVED_FROM, and FAN_MOVED_TO are
not generated for any entry modifications performed inside
subdirectories of marked directories.  Note that the events
FAN_DELETE_SELF and FAN_MOVE_SELF are not generated for
children of marked directories.  To monitor complete directory
trees it is necessary to mark the relevant mount or
filesystem.
~~~

### trace vim

~~~
stat("/home/ansible/test~", 0x7ffc68fe26f0) = -1 ENOENT (No such file or directory)
unlink("/home/ansible/test~")           = -1 ENOENT (No such file or directory) // delete back up file
rename("/home/ansible/test", "/home/ansible/test~") = 0 // move file to back up file
open("/home/ansible/test", O_WRONLY|O_CREAT|O_TRUNC, 0664) = 3 // recreate file
write(3, "1234\n123\n345\n\n\n", 15)    = 15
fsync(3)                                = 0
stat("/home/ansible/test", {st_mode=S_IFREG|0664, st_size=15, ...}) = 0
stat("/home/ansible/test", {st_mode=S_IFREG|0664, st_size=15, ...}) = 0
close(3)                                = 0
chmod("/home/ansible/test", 0100664)    = 0
~~~

unlink() deletes a name from the file system. If that name was the last link to a file and no processes have the file open the file is deleted and the space it was using is made available for reuse.

rename()  renames  a  file,  moving it between directories if required.  Any other hard links to the file (as created using link(2)) are unaffected.  Open file descriptors for oldpath are also unaffected.


### trace echo

<https://man7.org/linux/man-pages/man2/mmap.2.html>
https://man7.org/linux/man-pages/man2/dup.2.html



strace bash -c 'echo 123 > test'

int dup3(int oldfd, int newfd, int flags);


~~~
open("test", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 4
fcntl(1, F_GETFD)                       = 0
fcntl(1, F_DUPFD, 10)                   = 10
fcntl(1, F_GETFD)                       = 0
fcntl(10, F_SETFD, FD_CLOEXEC)          = 0
dup2(4, 1)                              = 1 //get fd copy
close(4)                                = 0
fstat(1, {st_mode=S_IFREG|0664, st_size=0, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f10b90b1000 // fd=-1 bacuse of flag MAP_ANONYMOUS
write(1, "123\n", 4)                    = 4 //direct write into file
dup2(10, 1)                             = 1
fcntl(10, F_GETFD)                      = 0x1 (flags FD_CLOEXEC)
close(10)                               = 0
~~~

### test mmap

~~~
import mmap

def mmap_io_write(filename, text):
    with open(filename, mode="w") as file_obj:
        with mmap.mmap(file_obj.fileno(), length=0, access=mmap.ACCESS_WRITE) as mmap_obj:
            mmap_obj.write(text)
mmap_io_write('known_hosts', '123')
~~~