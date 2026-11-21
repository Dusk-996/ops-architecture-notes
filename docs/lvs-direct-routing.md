# 为什么 LVS 收到请求，客户端却直接收到后端的响应？

切到 LVS 调度器抓包，可以看见客户端发起连接，却很少看见后端的响应。怎样区分正常的直接路由和真正的回程丢包？

![LVS-DR 时序图：入口网关把到 VIP 的请求转给活动调度器，调度器按后端 MAC 转发，后端用 VIP 经网关直接回给客户端](../diagrams/lvs-direct-routing.svg)

## 按图走一遍

本图采用经典 IPv4 以太网部署。客户端和服务端跨网段，调度器与真实服务器（RS）位于同一二层网络，后端本地绑定虚拟 IP（VIP）。图中的网关兼做后端的回程网关。现场也可能使用不同的出、入口网关。

| 观察位置 | 以太网帧的目的 MAC | IP 源 → 目的 | 应看到什么 |
|---|---|---|---|
| 入口网关向调度器发包 | 活动调度器的 MAC | 客户端 CIP → VIP | 网关已有 VIP 邻居项时直接使用；没有时才发 ARP 查询。 |
| 调度器向选中的 RS 发包 | RS 服务网卡的 MAC | CIP → VIP | 新连接由 IPVS 选后端；在这两段链路上，TCP 端口和目标 IP 保持一致。 |
| RS 从服务网段发出响应 | 回程网关的 MAC | VIP → CIP | RS 能本地接收 VIP，由本机套接字响应，按到 CIP 的路由发回。 |

若客户端与 VIP 同网段，客户端自己解析 VIP 的 MAC，RS 也可能直接把响应帧交给客户端。跨网段时，应在连着 VIP 网段的最后一跳观察 ARP。[LVS 项目对直接路由的说明](https://www.linuxvirtualserver.org/VS-DRouting.html)

## 原图修订

| 类型 | 原图线索 | 现场判断 |
|---|---|---|
| 原理保留 | RS 绑定 VIP，处理后直接响应 | 在图示方案中，RS 必须把 VIP 视为本地地址，服务监听 VIP 或合适的通配地址。响应使用 VIP 作源 IP，出接口取决于路由。[Red Hat 直接路由说明](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/load_balancer_administration/s1-lvs-direct-vsa) |
| 补充适用条件 | “后端抑制 ARP” | 目的是让 VIP 在该广播域内只由活动调度器对外应答。RS 对自身业务 IP 仍可正常参与 ARP。可用内核参数或其他机制实现，验收要观察实际应答者。[Linux 内核 ARP 参数](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html) |
| 明确报文字段 | “修改 VIP 对应的 MAC” | 向 RS 转发时改的是这一跳以太网帧的目的 MAC。网关的 VIP 邻居项仍应指向活动调度器。若这项变成某台 RS 的 MAC，新连接可能绕过调度器。[LVS 转发原理](https://www.linuxvirtualserver.org/VS-DRouting.html) |

## 动手验证

需要一套已配置的 DR 测试环境，或已获准观测的服务。下面的地址来自文档示例网段。**在每个终端先按实际情况替换 VIP、客户端地址、网卡和业务端口**。示例用 HTTP 健康路径；若实际服务使用 HTTPS 或依赖 Host，请使用该服务已有的探测方式。查看 IPVS 和抓包通常需要相应权限。

### 1. 确认虚拟服务、网卡和回程

```bash
# 在活动调度器执行，查 VIP:端口、后端和转发方式。
ip -br -4 addr
sudo ipvsadm -Ln
sudo ipvsadm -Ln --stats

# 在被选中的 RS 执行，替换为真实地址。
VIP=192.0.2.100
CIP=198.51.100.20
ip -br -4 addr
ip route get "$CIP" from "$VIP"
ss -lnt
```

预期在 IPVS 的对应服务下看到 RS 的转发方式为 `Route`。RS 的接口列表中应出现 VIP；查看回程的下一跳和接口是否符合部署。若服务能接受请求但客户端超时，优先检查 RS 回程路由及沿途是否允许以 VIP 为源的响应。

### 2. 在一笔连接中对齐三个观察点

调度器和 RS 各开一个终端，先启动以下观察，再从客户端发起一次健康请求。若服务有多台 RS，应观察被选中的那台，或同时在测试后端采集。使用具体的以太网接口以保留 MAC，`-p` 避免启用混杂模式，`-c` 和 `timeout` 限制采集范围。

```bash
# 在调度器服务网卡及 RS 服务网卡分别执行，按各机实际情况替换变量。
VIP=192.0.2.100
CIP=198.51.100.20
PORT=80
IFACE=eth0
ip -br link show dev "$IFACE"
sudo timeout 20 tcpdump -p -nne -i "$IFACE" -c 20 \
  "tcp and host $VIP and host $CIP and port $PORT"
```

在另一台测试客户端发起请求，使用与抓包过滤器一致的 VIP 和业务端口。`/health` 请替换为已知安全的健康检查路径。

```bash
curl --noproxy '*' --connect-timeout 3 --max-time 8 -sS -o /dev/null \
  -w 'HTTP %{http_code}\n' http://192.0.2.100/health
```

对照客户端源端口、业务端口及握手标志。在调度器入口和 RS 入口都应看到发往 VIP 的同一个 SYN；以太网目的 MAC 不同。RS 抓到源 IP 为 VIP 的 SYN-ACK，客户端也收到响应，才有证据支持“后端已直接回包”。仅凭调度器上看不到响应，还无法判断客户端是否真的收到。

### 3. 异常时再查 VIP 的 ARP

```bash
# 在 VIP 所在二层的 Linux 网关执行；同网段客户端也可在自身执行。
VIP=192.0.2.100
IFACE=eth0
ip -4 neigh show "$VIP" dev "$IFACE"
sudo timeout 15 tcpdump -p -nne -i "$IFACE" -c 8 "arp and host $VIP"
```

将邻居条目的 MAC 与活动调度器对照，同时留意高可用部署可能采用的虚拟 MAC。缓存尚有效时，15 秒内没有 ARP 报文很正常。若观察到不同设备为同一 VIP 应答，逐台核对 RS 的 VIP 地址、ARP 配置以及是否发生主备双活。先保存现有配置和观测结果，再单独安排变更。上述查询和定时抓包结束后无需配置回滚。

## 来源

核验日期为 2026-11-21。文中抓包现象为指定拓扑的预期结果，需在具备调度器、后端和客户端的环境中对照验证。[ProcessOn 原图 · LVS-DR](https://www.processon.com/view/link/5d43dfd1e4b0e42f2a2bd150) · [Linux Virtual Server 项目](https://www.linuxvirtualserver.org/VS-DRouting.html) · [Red Hat LVS 直接路由与 ARP](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/load_balancer_administration/s1-lvs-direct-vsa)
