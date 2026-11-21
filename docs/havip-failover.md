# ECS 主备切换后，HaVip 能覆盖多大的故障范围？

两台反向代理共用一个入口地址。主机宕机后备用主机还能接管；如果两台机器所在的可用区一起受影响，这个地址还有后端可以接吗？

![在同一可用区的同一交换机内，HaVip 稳态流量到 ECS A，主备依靠单播 VRRP 通告；ECS B 接管后，后续流量改到 B](../diagrams/havip-failover.svg)

## 按图走一遍

1. **先划故障边界。** 本例使用阿里云专有网络 VPC 的 HaVip。控制台中把同一交换机的两台 ECS 或其弹性网卡与该 HaVip 绑定，主备处在同一可用区。图中的实线表示正常时对服务地址的新请求落到 ECS A，虚线表示切换后的另一种状态。[阿里云 HaVip 使用限制与工作原理](https://help.aliyun.com/zh/vpc/highly-available-virtual-ip-address-havip)
2. **再看谁持有服务地址。** 两端配置同一虚拟路由实例和 HaVip，使用单播 VRRP。稳态下当前主机发送通告，备机监听。备机接管后成为新的通告发送方，对外服务地址保持一致。主机上的地址、Keepalived 状态日志和云平台的绑定信息要分别观察。[Keepalived 配置说明](https://www.keepalived.org/documentation/user-guide/configuration-synopsis/)
3. **最后验证业务响应。** Keepalived 正常运行时，应用仍可能返回 500。只有应用故障被健康检查捕获，并配置了能引起角色变化的跟踪策略时，该类故障才会触发切换。旧连接也可能中断，切换成功要用新的请求确认。若故障影响整个可用区，这一组主备无法提供跨可用区接管。[阿里云 HaVip](https://help.aliyun.com/zh/vpc/highly-available-virtual-ip-address-havip) · [Keepalived 的跟踪项](https://www.keepalived.org/documentation/user-guide/configuration-synopsis/)

## 原图修订

| 类型 | 原图线索 | 新图中的判断条件 |
|---|---|---|
| 保留原理，补充适用条件 | 主备 ECS 共用 HaVip 和 Keepalived | 核对两台实例所属交换机、VPC 和可用区。阿里云要求同一交换机，Keepalived 配为单播；只在两台操作系统上添加相同 IP 无法完成云侧绑定。[阿里云 HaVip](https://help.aliyun.com/zh/vpc/highly-available-virtual-ip-address-havip) |
| 故障范围 | 「高可用 VIP」容易被理解为任何故障都可切 | 分三类记录验收结果：单机失联、应用异常、整个可用区故障。只有第一类直接对应这套主备的设计目标；第二类取决于应用检查，第三类需要另一个可用区仍有独立入口和可工作的后端。 |
| 高可用指标 | 一眼只能看到切换的箭头 | 记录最后一次成功、第一次失败、第一次恢复以及采样间隔。按业务成功条件统计中断，而不是只记录备用节点成为 MASTER 的时间戳。 |

## 动手验证

以已有的隔离测试 VPC 为例，使用一个可从 VPC 内探测的私网 HaVip。下列检查只读，示例地址需替换为现场值。先在控制台核对 HaVip、关联实例或网卡、交换机 ID 与两台 ECS 的可用区。

### 1. 在主机和备机上分别取证

```bash
# 两台 ECS 分别运行；输入实际的 IPv4 HaVip。
vip=192.0.2.50
hostname
date -u '+%FT%TZ'
systemctl is-active keepalived
ip -o -4 addr show | awk -v vip="$vip" '{split($4,a,"/"); if (a[1]==vip) print $2,$4}'
journalctl -u keepalived --since '15 minutes ago' --no-pager | tail -n 30
```

稳态预期为两端 Keepalived 服务均运行，恰有一台网卡持有目标地址。`systemctl is-active` 只报告进程状态；判断角色要结合持有地址和日志中的状态转换。日志不可读时使用有日志读取权限的账号。若双机同时显示该地址，先核对单播对端、VRID、通告可达性及日志；此时应暂停主动切换演练。

### 2. 在第三台 VPC 内客户端观察连续业务请求

```bash
# 填写实际业务域名、HaVip 和可信证书覆盖的只读业务检查路径。
site=shop.example.test
vip=192.0.2.50
endpoint=/healthz

for sample in $(seq 1 10); do
    printf '%s ' "$(date -u '+%FT%TZ')"
    curl --noproxy '*' --resolve "$site:443:$vip" -sS --connect-timeout 2 \
        --max-time 3 -o /dev/null -w 'http=%{http_code} total=%{time_total}s\n' \
        "https://$site$endpoint"
    sleep 1
done
```

`--resolve` 保留域名校验和 TLS SNI，仅将连接目标固定为 HaVip。把客户端记录与两台代理的同期访问日志对照；业务路径应选用能够代表所需能力的检查。若接口对任何实例都返回相同的 200，通过主备日志中的请求时间、路径和探针地址确认实际接管者。采样循环之间有请求耗时和等待时间，只能给出中断窗口的估计。

### 3. 判读现象并准备恢复

| 故障或现象 | 该检查能回答什么 |
|---|---|
| 原主机失联 | 已有变更窗口中，观察 B 是否获得地址、云上流量是否到 B、客户端是否恢复。三者记录同一时钟基准。 |
| 原主机在线，业务持续失败 | 检查应用健康判据、连续失败次数、跟踪权重，以及是否足以改变主备状态。只有主备进程日志正常，仍无法证明应用成功。 |
| 两台主机同时不可达 | 对照同可用区、同交换机上的其他服务，再看云侧事件。此组 HaVip 仅覆盖同区主备，跨区能力需单独验证。 |

**演练影响和恢复。** 主动停机或停止 Keepalived 会改变流量归属并可能中断连接。以上命令不触发切换。若另外安排隔离环境演练，事先确认备用机能承载请求、记录当前主角色和抢占策略；恢复时先让原主机及应用恢复健康，再按策略决定是否回切。回切会再次改变归属，完成后重做地址与业务检查。

本次未在阿里云实际切换。以上均为预期和采样方法，真实业务恢复时间须由现场演练给出。

## 来源

核验日期为 2026-11-21。原图着重说明用户访问和主备关系，本图专门展开其网络边界与失效条件。[ProcessOn 原图 · 毕业高级架构 v2](https://www.processon.com/view/link/61b9b4a81efad406de4343cf)
