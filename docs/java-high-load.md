# Java 服务响应变慢，load 很高，接下来查 CPU 还是查等待？

同一分钟里，接口 P95 延迟上升，Linux 的一分钟 load 也翻了几倍。值班时第一步要弄清：哪些任务在等 CPU，哪些任务卡在内核等待点，这些任务又是否属于出问题的 JVM？

![排查图：对齐延迟与系统负载时段，可运行队列积压时定位热点线程及其栈，不可中断等待增多时检查等待点和存储依赖](../diagrams/java-high-load.svg)

## 按图走一遍

1. **先对齐时间、实例和影响。** 记录服务实例、告警起止时间、同窗口的请求量、错误率与延迟。Linux 的 1、5、15 分钟 load 包含可运行任务以及不可中断等待的任务。主机上其他进程也会贡献 load。
2. **沿 CPU 分支定位到代码。** 连续采样时，若运行队列持续拥挤、CPU 空闲很少，同时 Java 进程内有线程持续消耗 CPU，记录线程 TID。对 HotSpot 平台线程，可将 Linux TID 转为十六进制，与线程栈中的 `nid` 对应。
3. **沿等待分支定位到依赖。** 如果 D 状态线程增多，按 PID/TID 和 `wchan` 找等待点，再看相关块设备、文件系统或远端存储的延迟。CPU 分支和等待分支可能同时出现；故障期间也可能仅个别核心打满，主机总 CPU 仍有空闲。

## 原图修订

| 类型 | 原图线索 | 验证依据 |
|---|---|---|
| 判断条件 | 用“CPU 核数的 60%～70%”判断高负载 | load 是任务数的时间平均值，阈值需结合业务基线、可用 CPU 和持续时间。容器还要确认有效 CPU 配额。对服务影响的判断以同实例延迟和错误为据。[Linux load 定义](https://man7.org/linux/man-pages/man5/proc_loadavg.5.html) · [cgroup v2 CPU 控制器](https://docs.kernel.org/admin-guide/cgroup-v2.html) |
| 证据范围 | 根据 `vmstat r/b` 或 `top` 的等待比例选方向 | `r` 包括正在运行及等待 CPU 的任务；`b` 反映等待 I/O 完成的阻塞进程。持续时间、线程归属和设备时延可帮助定位，单个采样值不能指出故障磁盘。[vmstat 字段说明](https://man7.org/linux/man-pages/man8/vmstat.8.html) |
| 取证成本 | 线程栈和堆转储同时列入排查工具 | 热线程先做线程级取样。若持续出现堆占用回收不下去、Full GC 频繁或 OOM 等内存线索，再评估堆转储；JDK 25 将 `GC.heap_dump` 列为高影响命令，默认请求 Full GC。[JDK 25 jcmd 命令参考](https://docs.oracle.com/en/java/javase/25/docs/specs/man/jcmd.html) |

## 动手验证

### 1. 在受影响的 Linux 节点采集同一时间窗口

```bash
date -Is
uptime
cat /proc/loadavg
vmstat 1 6
ps -eLo state,pid,lwp,comm,wchan:32 | awk 'NR == 1 || $1 == "D"'
```

使用 `vmstat` 首行之后的五组间隔数据判断趋势。若 `r` 持续较高、`us`/`sy` 提高，检查争用 CPU 的进程；若 `b` 上升且有 D 状态线程，记录其 PID、TID、等待点。单次 `ps` 容易错过短暂阻塞，可在告警窗口重复查询。若热点来自同机的其他服务，先记录其资源占用及影响范围。

容器现场应分别看主机和目标容器。在 cgroup v2 下，若容器内 `/sys/fs/cgroup` 的根就是目标 cgroup，可比较间隔采样的 `cpu.stat` 并读取 `cpu.max`；其他挂载方式由 `/proc/self/cgroup` 定位对应目录。`nr_throttled` 在两次采样间增长，说明发生过 CPU 限流，仍需与业务延迟对齐。主机核数不能直接作为容器的可用核数。[Linux 内核 cgroup v2 文档](https://docs.kernel.org/admin-guide/cgroup-v2.html)

### 2. 热线程分支：逐次对应 Linux TID 与 JVM 栈

在运行目标 HotSpot JVM 的主机或容器中执行。使用与目标 JVM 相同的有效用户和组、可见相同的 PID 命名空间及匹配版本的 JDK。主机如有多个 JVM，按服务归属选择 PID。

```bash
# 确认目标 JVM，再由操作者输入 PID；pidstat 属于 sysstat。
ps -C java -o pid,user,comm
read -r -p '目标 JVM PID: ' JVM_PID
pidstat -u -t -p "$JVM_PID" 1 5
```

在连续区间中找 TID 行里反复出现的高 `%CPU` 线程，区分进程汇总行与具体线程。没有 `pidstat` 时可用 `top -H -p "$JVM_PID"` 连看几次刷新。线程瞬时冲高后消失时，应在下一次异常重新采样。

```bash
# 延续同一 Shell。留存一份栈便于稍后比对，目录只允许本用户访问。
read -r -p '持续消耗 CPU 的十进制 TID: ' HOT_TID
nid_hex=$(printf '%x' "$HOT_TID")
stack_dir=$(mktemp -d) || exit 1
jcmd "$JVM_PID" Thread.print > "$stack_dir/threads.txt" &&
    grep -i -E -A 12 "nid=0x0*${nid_hex}([[:space:]]|$)" "$stack_dir/threads.txt"
```

预期对应的线程名和调用栈会出现在匹配行及其下方。线程栈是一个瞬间的状态，`RUNNABLE` 本身不能证明正在消耗 CPU。必要时按相同方法隔几秒取第二份栈，核对是不是同一 TID 反复停在同一调用路径。若 `nid` 无匹配，先查线程是否已退出、工具和 JVM 是否处在同一 PID 命名空间。虚拟线程可能在载体线程间切换，此方法只能把原生 TID 对应到载体或其他平台线程。Oracle 将 `Thread.print` 的影响列为中等，抓取时应考虑线程数和服务时延，避免高频循环采集。[Oracle jcmd 使用与影响说明](https://docs.oracle.com/en/java/javase/25/docs/specs/man/jcmd.html)

### 3. 不可中断等待分支：关联等待点和设备

```bash
# 在出现 D 状态的节点执行，使用第 1 步记录的进程号。
read -r -p '含 D 状态线程的 PID: ' WAIT_PID
ps -L -p "$WAIT_PID" -o pid,lwp,stat,wchan:32,comm
findmnt -t nfs,nfs4
iostat -xz -y 1 5
```

若多个线程持续卡在同一等待点，且服务所用本地设备在相同窗口的 `await`、队列长度一起升高，进一步查该设备上的其他读写和存储侧时延。若等待点指向 NFS，则核对挂载对应的服务端及网络；本地盘统计正常仍可能发生远端挂载等待。权限限制可能使 `wchan` 显示为 `0` 或占位符，此时结合存储侧指标判断。

### 4. 处理后验证与生产影响

修复取决于证据。例如热点栈对应最近上线的计算逻辑，可用已知正常版本做小流量对照；远端存储等待应先恢复相应依赖。处理后在相近请求量下比较 P95、错误率、热点线程 CPU 或 D 状态数量。load 自身带时间平滑，恢复时可能晚于请求延迟回落。

**堆转储可能明显暂停 JVM 并占用大量磁盘。** 只有内存证据指向堆分析时才安排采集，先核对当前 JDK 的 `jcmd "$JVM_PID" help GC.heap_dump`、目标磁盘空间、可用副本和数据访问权限。采集导致的停顿无法回滚；若业务已接近超时阈值，应在具备冗余和可恢复流量的实例上单独安排。以上常规检查均为临时采样；线程栈保存在本次临时目录，按团队诊断数据留存策略处理。

## 来源

核验日期为 2026-11-21。文中判读示例描述预期现象，具体热点、等待点和调用栈由现场采样确定。[ProcessOn 原图 · Java 系统负载高排查](https://www.processon.com/view/link/6281a2e76376890bfe677e1b) · [Linux load average](https://man7.org/linux/man-pages/man5/proc_loadavg.5.html) · [Oracle JDK 25 诊断命令](https://docs.oracle.com/en/java/javase/25/docs/specs/man/jcmd.html)
