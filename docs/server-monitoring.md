# 网站响应慢时，先从用户侧还是服务器查起？

下午某地用户说“页面要等三秒”，值班同事看到 Web 主机 CPU 只有 20%。先记录用户访问的时间、地区、域名和路径，再把站外探测与实际承接该请求的网关、应用、主机放在同一时间窗口里。

![从 DNS、TCP、TLS 和 HTTP 观测用户体验，将网关与应用日志、主机和设备指标放入同一故障窗口](../diagrams/server-monitoring.svg)

## 按图走一遍

| 观察位置 | 先看什么 | 根据结果缩小到哪里 |
|---|---|---|
| 用户地区的探测点 | 实际解析出的 IP、DNS 耗时、建连耗时、TLS 握手耗时 | 仅一个地区慢时，找相同运营商或地区的第二个探测点，核对它们命中的地址及边缘节点 |
| HTTP 入口 | 状态码、首字节时间、下载完成时间、内容是否符合预期 | 连接快而首字节慢时，用请求时间或请求 ID 去找命中的网关。首字节快而总耗时长时，再查响应大小、网络吞吐及客户端接收速度 |
| 网关与应用 | 对应请求的总耗时、上游地址、上游耗时、重试和应用日志 | 若上游耗时同时升高，进入该应用实例排查。若只在入口看到延迟，核对代理排队、请求上传、向客户端发送响应等环节 |
| 主机与网络设备 | 同一实例的 CPU 运行队列、磁盘时延、接口错误或丢包、设备监控 | 核对异常是否先于或伴随用户侧变慢出现，恢复时是否同步。一次主机采样只能解释当前时刻 |

窗口示例取五分钟。先把告警、探测和日志时间统一到同一时区，再按域名、路由、实例及所在机房匹配。CDN 命中时请先看边缘日志，源站可能根本没有收到该请求。[NGINX 上游计时变量](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_response_time)

## 原图修订

| 类型 | 原图内容 | 使用方式 |
|---|---|---|
| 保留 | 从各地观察访问情况，同时观察硬件、系统、服务和业务 | Zabbix/SNMP 可继续用于设备，Node Exporter 可采集 Linux 主机指标；外部探测点单独记录访问目标的成功与耗时。[主机指标采集](https://prometheus.io/docs/guides/node-exporter/) · [Blackbox Exporter 探测](https://prometheus.io/docs/guides/multi-target-exporter/) |
| 已标注失效的示例 | 原图已将一个外部测速地址注明“无法使用” | 选能长期保存结果、能识别探测地区的自有探测点或监测服务。变更供应商后重新建立该地区的基线。 |
| 适用条件 | `ping`、端口探测、HTTP 探测并列 | ICMP 可用于观察网络；TCP 成功只说明端口可连接；HTTP 探测应按业务检查预期状态或内容。用对方真正访问的路径复核，防止首页正常而接口已经异常。[Prometheus 探测模块示例](https://prometheus.io/docs/guides/multi-target-exporter/) |

## 动手验证

### 1. 在故障地区测量一次 HTTPS 请求

在能代表用户网络的 Linux 探测机运行。将示例域名换成自己负责的、无副作用的 HTTPS 只读地址。这里检查直接连接、HTTP/1.1、单次 GET，不跟随重定向。若用户经企业代理上网，另在实际代理路径复测并记录差别。选业务路径时也要确认成功状态和内容应是什么。

```bash
url='https://shop.example.test/catalog'
date -u '+%Y-%m-%dT%H:%M:%SZ'
curl --http1.1 --noproxy '*' --connect-timeout 3 --max-time 10 \
    --silent --show-error --output /dev/null \
    --write-out 'ip=%{remote_ip} code=%{http_code} dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} first=%{time_starttransfer} total=%{time_total}\n' \
    "$url"
printf 'curl_exit=%s\n' "$?"
```

这些计时值均从请求开始累计，单位为秒。对一次成功的直接 HTTPS 请求，`tcp − dns` 近似建连耗时，`tls − tcp` 近似握手耗时，`first − tls` 包含发送请求、网络往返与服务端首字节等待。`total − first` 反映余下响应体的接收时长。若 curl 失败，先看退出码和错误阶段；尚未完成的计时值可能为 0。[curl 官方计时字段](https://curl.se/docs/manpage.html#-w)

例如某次输出 `dns=0.015 tcp=0.056 tls=0.090 first=0.440 total=0.445`。这是演算用数据：TCP 约 41 ms，TLS 约 34 ms，TLS 完成至首字节约 350 ms，后续接收约 5 ms。下一步优先查网关和上游计时。重复几次并与正常地区对比，记录两者的 `remote_ip`。

**实测记录（2026-11-21）。** 在 WSL Ubuntu 内运行仅监听回环地址的 HTTPS 服务，并用 `CURL_CA_BUNDLE` 信任临时证书。将示例 URL 换为该服务的 IP 和端口后，得到 `code=200 dns=0.000019 tcp=0.000122 tls=0.003455 first=0.003805 total=0.003823`，curl 退出码为 0。该次请求直接使用 IP，可验证 TCP/TLS/HTTP 计时输出，DNS 与跨地区网络延迟需要在真实域名上测量。

### 2. 在监控平台对齐窗口

若已部署 Prometheus 和 Blackbox Exporter，在查询界面把结束时间设为故障时间，选择相应任务与目标。下面沿用官方示例的标签约定，现场应按实际的 `job`、`instance` 和探测地区筛选。

```promql
avg_over_time(probe_success{job="blackbox-http",instance="https://shop.example.test/catalog"}[5m])
```

成功探测通常取值 1，失败取值 0。平均值 0.8 表示五分钟内已采到的探测样本有 80% 成功。若无结果，查标签、目标和同期 `up`。`up=1` 表示 Prometheus 抓取 exporter 成功，此时 `probe_success` 仍可能为 0。用下式查看实际样本数，估计一次失败占多大比重。[Blackbox Exporter 工作方式](https://prometheus.io/docs/guides/multi-target-exporter/)

```promql
count_over_time(probe_success{job="blackbox-http",instance="https://shop.example.test/catalog"}[5m])
```

在网关按同一 URL、时间范围及请求 ID 查已有访问日志。若日志已包含 `$request_time`、`$upstream_addr` 和 `$upstream_response_time`，对比慢请求与正常样本的上游地址、耗时和状态。出现多段上游值时，逐次核对重试。不要把两个计时直接相减当作网络延迟。后端为 Java 且同期开销上升时，接着读 [Java 服务高负载排查](java-high-load.md)。

### 3. 在命中的 Linux 主机检查当前资源

以下命令只读取计数器，适合故障仍在发生时执行。过去的故障须到监控平台或历史日志查当时的数据。

```bash
date -u '+%Y-%m-%dT%H:%M:%SZ'
vmstat 1 6            # 跳过第一行自开机以来的平均值，观察后续五个间隔
iostat -xz -y 1 5     # 需已安装 sysstat，-y 跳过开机以来的首份报告
ip -br link           # 找到实际承载流量的网卡名
```

若 `vmstat` 的运行队列 `r` 持续增长且 CPU 空闲时间下降，找占用 CPU 的进程；若磁盘 `await` 与业务延迟同步升高，再检查对应块设备及读写来源。若怀疑网络，用 `ip -s link show dev 接口名` 连续读取两次，对比期间的错误与丢包增量，并与交换机同一端口的设备指标核对。

本节提供现场核验方法，探测域名、故障时段及示例耗时均需以实际观测替换。只读取日志和指标即可完成这轮定位。

## 来源

原图与规则核验日期为 2026-11-21。[ProcessOn 原图 · 服务器监控体系](https://www.processon.com/view/link/5a7ab69ae4b064e9ddbc5c59) · [Prometheus 多目标探测](https://prometheus.io/docs/guides/multi-target-exporter/) · [Linux 主机监控](https://prometheus.io/docs/guides/node-exporter/) · [curl 手册](https://curl.se/docs/manpage.html)
