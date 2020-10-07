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


## fanotify

fanotify 支持的事件类型参考 [fanotify_mark](https://man7.org/linux/man-pages/man2/fanotify_mark.2.html)