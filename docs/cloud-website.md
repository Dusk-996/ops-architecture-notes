# 阿里云网站的一次请求，什么时候才会到达数据库？

用户反馈商品图很慢，同一秒应用服务器的 CPU 却很低。这张图以一个把静态文件放在 OSS、动态业务放在 ECS 的网站为例，沿途找出哪些层真正处理了这次请求。

![浏览器先解析域名，再访问 CDN；静态未命中可回源 OSS，动态请求经 ALB 到达 ECS，应用按需读取 Tair 和 RDS](../diagrams/cloud-website.svg)

## 按图走一遍

1. **DNS 找入口。** 示例站点的加速域名以 CNAME 接入 CDN。浏览器得到边缘地址后才建立 HTTPS 连接。排查解析慢以及排查 HTTP 响应慢，需要记录不同阶段的耗时。[CDN 接入流程](https://help.aliyun.com/zh/cdn/getting-started/getting-started)
2. **先看资源属于哪条路径。** 对允许缓存的商品图，边缘有新鲜副本就能返回。需要回源时，本例静态资源走 OSS；动态接口通过配置好的动态回源路径交给应用负载均衡 ALB，再转发至 ECS。生产中按域名、路径分别配置源站或动静分离域名，应以现场规则为准。[CDN 回源术语](https://help.aliyun.com/zh/cdn/product-overview/terms) · [ALB 监听与转发](https://help.aliyun.com/zh/slb/application-load-balancer/what-is-alb)
3. **应用决定读哪些数据。** 以查商品详情为例，应用可以先查 Tair 的商品键，未命中再查 RDS，按业务策略将结果写回缓存。命中 CDN 的商品图可以绕过整条应用链路。图中的两处缓存分别使用 HTTP 资源和业务键，排查时应分别取证。

## 原图修订

| 类型 | 原图线索 | 使用时如何判断 |
|---|---|---|
| 已经过时 | 防火墙示例按 CentOS 6、7 分组 | CentOS Linux 6、7 分别于 2020-11-30、2024-06-30 结束上游更新。这组命令适合识别存量主机；给新系统写规则前先核对发行版与实际管理工具。[CentOS 生命周期](https://wiki.centos.org/FAQ%282f%29General.html) |
| 已经过时 | 运维接入同时列了 PPTP 和 OpenVPN | 将 PPTP 留在历史环境说明中。它的安全特性已有明确局限，微软的 Windows Server 2025 新建 RRAS 配置默认不接受 PPTP。[微软 VPN 协议说明](https://learn.microsoft.com/zh-cn/windows-server/remote/remote-access/configure-vpn-protocols) |
| 适用条件 | CDN 节点旁标有「网站加速 90%」 | 先定义是请求命中率、字节命中率、源站带宽下降比例，还是同一批请求的延迟变化，分开记录分子、分母及观察窗口。[阿里云 CDN 监控指标](https://help.aliyun.com/zh/cdn/getting-started/getting-started) |

## 动手验证

**执行条件。** 在实际用户所在网络的 Linux 测试机执行，准备 `dig` 和 `curl`。以下以 CNAME 接入的阿里云 CDN 为例，请替换为自己站点中一个稳定、可公开读取的小静态资源。若静态资源和 API 分属不同域名，分别记录每个域名的结果。

```bash
# 示例域名和资源路径需替换；两次请求使用完全相同的 URL。
site=static.example.test
asset=/assets/logo.svg
date -u '+%FT%TZ'
dig +noall +answer "$site" CNAME
dig +noall +answer "$site" A

for attempt in 1 2; do
    printf '\nGET %s\n' "$attempt"
    curl --noproxy '*' -sS --max-time 10 -D - -o /dev/null \
        -w 'peer=%{remote_ip} code=%{http_code} dns=%{time_namelookup}s ttfb=%{time_starttransfer}s total=%{time_total}s\n' \
        "https://$site$asset"
done
```

| 观测点 | 预期及下一步 |
|---|---|
| DNS 应答与 `peer` | CNAME 应与控制台分配的名称相符，A 记录可随调度变化。`peer` 是本次 TCP 连接的对端；若需要代理才能出网，去掉直连选项并记录代理位置。 |
| GET 响应与 CDN 日志 | 记录 `X-Cache`、`Age`、内容状态码和时间。已预热的资源第一次就可能 HIT；连续 MISS 时检查缓存键、路径规则和响应头。边缘日志中的 MISS 也可能由上级缓存满足，须结合回源日志核实是否真正触及源站。[阿里云 CDN 缓存排查](https://help.aliyun.com/zh/cdn/user-guide/cache-troubleshooting) · [边缘日志字段](https://help.aliyun.com/zh/cdn/user-guide/offline-logs-quick-start) |
| 源站与应用 | 在同一时段，按主机名、路径和请求 ID（若链路传递）查 OSS 或 ALB、应用访问日志。确有应用请求，再按该接口的 trace 或慢查询记录检查 Tair/RDS。若源站日志采样或有投递延迟，先确认日志覆盖范围。 |
| 耗时 | curl 的 `time_starttransfer` 是从操作开始到首字节到达的累计时间，包含前序解析与连接。用同一地区、同一资源的多次样本比较，单次差值无法直接归因于数据库。[curl 计时字段](https://curl.se/docs/manpage.html#-w) |

上述为现场核对方法和预期现象。本次未接入业务 CDN、OSS 或 ECS，未测得命中率和加速比例。

## 来源

核验日期为 2026-11-21。原图提供 CDN、负载均衡、缓存、数据层等教学素材，图中的 ALB、OSS、Tair、RDS 是用于验证请求分支的一组云上实现。[ProcessOn 原图 · 阿里云架构](https://www.processon.com/view/link/5d1f04c6e4b0aad4c926ba41)
