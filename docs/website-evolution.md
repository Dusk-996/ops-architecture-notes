# 网站变慢后，下一步该拆数据库、加 Web 实例，还是加缓存？

同一个活动页在低峰正常，高峰开始排队。加一台 Web 服务器可能有效，也可能只给已经饱和的数据库增加连接。先找出等待发生的位置，再决定改哪一层。

![网站从单机到数据库拆分、Web 集群和缓存的演进条件；缓存命中、回源和响应缓冲各自的路径](../diagrams/website-evolution.svg)

## 按图走一遍

| 拓扑 | 进入下一阶段前应看到的证据 | 新引入的代价 |
|---|---|---|
| 单机，Web 和数据库同机 | 同一时段，应用和数据库争用内存、CPU 或磁盘；能从两个进程各自的指标解释延迟 | 把数据库独立部署后，应用到数据库增加了网络往返和连接故障点 |
| Web 与数据库分机 | 单个 Web 实例的工作线程或连接队列持续堆积，数据库还有余量；同负载下增加一个测试实例可以降低排队时间 | 要确定会话、上传文件和定时任务由谁持有，避免请求换实例就丢状态 |
| 负载均衡分发到多个 Web 实例 | 热门公开资源被重复读取。同一版本的内容可复用，并且回源开销在延迟或带宽中占了可观比例 | 缓存键、过期时间和更新策略需要与业务一致。登录态和个性化响应需另定规则 |
| 对可复用的响应启用缓存 | 命中时边缘节点或反向代理直接回答，未命中时请求源站，并按规则决定是否保存响应 | 热点同时过期、内容更新后仍可见旧版本，以及缓存盘占用，都需单独观察 |

图上“响应传输”讲的是另一个环节。代理从上游收到数据后，可先将本次响应写入内存缓冲区，必要时写入临时文件，从而让上游和客户端以不同速率收发。开启缓冲并不能证明下一次请求会命中缓存。[NGINX 响应缓冲](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering) · [NGINX 响应缓存](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache)

## 原图修订

| 类型 | 原图内容 | 重新整理后的含义 |
|---|---|---|
| 适用条件 | 单机、迁移或拆分、集群、集群缓存优化四个阶段 | 保留四种拓扑。每次变化由可观测的瓶颈触发；若问题来自一个慢查询，应先检查执行计划和查询负载。 |
| 名称与用途 | 原图并列写了 `proxy_buffer`、`proxy_cache` | NGINX 的响应缓冲开关为 `proxy_buffering`，缓冲区由 `proxy_buffers` 等指令控制。跨请求复用由 `proxy_cache` 控制，默认关闭。代理记录的 `$upstream_cache_status` 可区分命中、未命中、绕过等结果。[代理模块指令](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) · [上游变量](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_cache_status) |

## 动手验证

### 1. 固定请求口径，判断缓存有没有帮上忙

**执行位置与前提。** 在测试机或日志分析机使用 Bash 和 GNU awk。先选一个具有固定版本号、无登录态、无查询参数的静态资源。以下函数读取六列日志，依次为 NGINX 的 `$msec $host $uri $status $request_time $upstream_cache_status`，时间窗口左闭右开。可从已有日志系统导出这些字段；默认 combined 日志缺少缓存状态，不能直接套用。

```bash
report_window() {
    LC_ALL=C gawk -v begin="$1" -v end="$2" -v host="$3" -v uri="$4" '
    $1 >= begin && $1 < end && $2 == host && $3 == uri {
        requests++
        if ($4 >= 500 && $4 < 600) errors++
        if ($6 == "HIT") hits++
        if ($4 == 200) elapsed[++ok] = $5 + 0
    }
    END {
        if (!requests) { print "没有匹配记录，请核对时间、域名、路径和字段顺序"; exit 1 }
        printf "requests=%d 5xx=%d HIT=%d\n", requests, errors, hits
        if (ok) {
            asort(elapsed)
            rank = int((95 * ok + 99) / 100)
            printf "200_samples=%d request_time_p95=%.3fs\n", ok, elapsed[rank]
        }
    }'
}

# 自检数据。一条请求等到回源后返回，两条命中缓存，一条服务端错误。
report_window 1000 1300 assets.example.test /logo.v3.svg <<'LOG'
1001.000 assets.example.test /logo.v3.svg 200 0.180 MISS
1002.000 assets.example.test /logo.v3.svg 200 0.008 HIT
1003.000 assets.example.test /logo.v3.svg 200 0.010 HIT
1004.000 assets.example.test /logo.v3.svg 502 0.500 MISS
LOG
```

预期得到 `requests=4 5xx=1 HIT=2` 和 `200_samples=3 request_time_p95=0.180s`。小样本只用于检查计算和分母。P95 取成功状态为 200 的响应时长，5xx 单列，防止错误请求快速返回后使延迟曲线看起来变好。

**实测记录（2026-11-21）。** 将上面四行自检数据交给 GNU awk 5.3.2，得到 4 条记录、1 次 5xx、2 次 HIT，3 条成功记录的 P95 为 0.180 秒。该结果验证了统计逻辑，业务收益仍需使用真实访问日志衡量。

### 2. 比较真实流量与源站负荷

在已导出日志的分析机上执行，按现场修改时间和名称。使用变更前后请求速率、响应体大小、用户地区相近的窗口，另行标出缓存刚开始预热的时段。

```bash
log=./edge-sample.log
begin=$(date -d '2026-11-20 09:00:00 +0800' +%s)
report_window "$begin" "$((begin + 300))" assets.example.test /logo.v3.svg < "$log"
```

- 对照负载均衡或各个源站的访问日志，汇总同一路径、同一窗口实际到达源站的请求数。用“源站请求数 ÷ 边缘请求数”比较回源压力；重试、预取和后台更新都会影响这两个计数。
- 若成功响应 P95 与源站单位时间请求数均下降，并且 5xx 比例平稳，再结合相邻窗口复核收益。若 `HIT` 增加但用户仍慢，分别检查 TLS、首字节和响应体传输时间。[服务器监控体系](server-monitoring.md)
- 若多数记录为 `BYPASS`，检查当前请求是否触发绕过条件。若始终为 `MISS`，核对请求是否使用同一缓存键、响应是否可缓存、过期时间是否过短。命中状态与源站日志应能相互解释。[NGINX 缓存控制](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_bypass)

## 来源

原图与规则核验日期为 2026-11-21。图中的演进条件是对原图四个阶段的工程化补充，实网收益取决于现场测量。[ProcessOn 原图 · 网站架构演变](https://www.processon.com/view/link/6285fa13e0b34d35a1f37680) · [NGINX 代理模块](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) · [NGINX upstream 模块](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
