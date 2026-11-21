# 同一 IP 上，NGINX 如何选中站点和路径？

两次请求都发往 `127.0.0.1:18080/healthz`，一次返回商店的健康状态，另一次却返回 404。只改了 HTTP 的 `Host` 头，为什么连健康检查结果都变了？

![NGINX 在同一监听地址上按 Host 选站点，商店站点继续匹配路径，陌生 Host 落到默认站点](../diagrams/nginx-request-routing.svg)

## 按图走一遍

1. **先看连接落到哪个监听地址。** 实验中的连接都落到 `127.0.0.1:18080`。同一个地址和端口上配置了商店站点和默认站点，因此知道目的 IP 还不足以判断响应内容。
2. **再用 Host 选站点。** 请求头是 `Host: shop.example.test` 时，命中商店的 `server_name`。若是 `unknown.example.test`，没有名称匹配，请求交给该监听地址和端口的 `default_server`。
3. **在选中的站点内匹配 URI。** 商店的 `location = /healthz` 精确匹配健康检查，返回 `shop-health`。访问 `/` 等其他路径时，由该站点的 `location /` 返回 `shop-home`。默认站点的 `location /` 对所有路径返回 404 和正文 `default`。即使请求的路径叫 `/healthz`，也只会在已选站点内查找。

这里使用明文 HTTP、精确域名和本地响应。线上 HTTPS 连接在 HTTP 请求之前还有 TLS 握手；若出现证书或 SNI 问题，应先核对握手阶段选用的证书，再对照 HTTP 的站点选择。[NGINX 虚拟主机选择阶段](https://nginx.org/en/docs/http/server_names.html#virtual_server_selection)

## 原图修订

| 类型 | 原图中容易引起误判的说法 | 运维时如何判断 |
|---|---|---|
| 事实纠正 | 直接访问 IP 会“随机”匹配站点。 | 访问 IP 时客户端通常把 IP 写入 `Host`。若没有对应的 `server_name`，请求进入该地址和端口的默认站点。显式配置 `listen … default_server` 可以指定它；省略时使用该监听组的第一个 `server`。[请求处理规则](https://nginx.org/en/docs/http/request_processing.html) |
| 适用条件 | 可以凭 URL 一路推导出站点目录下的文件。 | 静态文件由相应的 `root` 或 `alias` 映射。命中 `return` 时直接生成响应，命中 `fastcgi_pass` 时交给 FastCGI 上游。应先读命中的 `location`，再决定检查文件、返回规则还是上游。[location 规则](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) · [FastCGI 转发](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass) |

## 动手验证

**执行位置与影响范围。** 在已有发行版标准 NGINX 和 curl 的 Linux 测试机上，用同一普通用户、同一 Shell 运行下列代码。实验只监听本机回环地址，另起一个 NGINX 实例，PID、日志和临时文件全部放在本次实验目录。若启动时报 `address already in use`，换到端口空闲的测试机再做。开始前可用 `command -v nginx curl` 确认可执行文件在 PATH 中。

### 1. 写入配置，启动实验实例

```bash
lab=$(mktemp -d) || exit 1
mkdir -p "$lab/logs"

cat > "$lab/nginx.conf" <<'NGINX'
worker_processes 1;
pid logs/nginx.pid;
error_log logs/error.log notice;

events { worker_connections 64; }

http {
    # 使用独立目录，普通用户也能初始化临时文件。
    client_body_temp_path client_temp;
    proxy_temp_path       proxy_temp;
    fastcgi_temp_path     fastcgi_temp;
    uwsgi_temp_path       uwsgi_temp;
    scgi_temp_path        scgi_temp;

    default_type text/plain;
    log_format route 'host=$http_host server=$server_name uri=$uri status=$status';
    access_log logs/access.log route;

    server {
        listen 127.0.0.1:18080;
        server_name shop.example.test;
        location = /healthz { return 200 "shop-health\n"; }
        location /          { return 200 "shop-home\n"; }
    }

    server {
        # 即使排在第二位，这个 server 仍由 default_server 显式指定为默认站点。
        listen 127.0.0.1:18080 default_server;
        server_name fallback.example.test;
        location / { return 404 "default\n"; }
    }
}
NGINX

nginx -p "$lab/" -c "$lab/nginx.conf" -t &&
    nginx -p "$lab/" -c "$lab/nginx.conf"
```

配置检查通过且启动成功后再发请求。部分发行版会在读入实验配置前提示无法打开系统默认错误日志；应同时核对启动命令的退出状态、`$lab/logs/error.log` 和 `$lab/logs/nginx.pid`，确认实验实例已运行。

### 2. 只改一个变量，观察三次响应

```bash
# 第一次和第二次固定 URI，只换 Host；第三次保留商店 Host，只换 URI。
curl --noproxy '*' -sS -w 'HTTP %{http_code}\n' -H 'Host: shop.example.test'    http://127.0.0.1:18080/healthz
curl --noproxy '*' -sS -w 'HTTP %{http_code}\n' -H 'Host: unknown.example.test' http://127.0.0.1:18080/healthz
curl --noproxy '*' -sS -w 'HTTP %{http_code}\n' -H 'Host: shop.example.test'    http://127.0.0.1:18080/
tail -n 3 "$lab/logs/access.log"
```

| 请求 | 预期正文与状态码 | 日志中应命中的站点 |
|---|---|---|
| 商店域名，`/healthz` | `shop-health`，200 | `server=shop.example.test` |
| 陌生域名，`/healthz` | `default`，404 | `server=fallback.example.test` |
| 商店域名，`/` | `shop-home`，200 | `server=shop.example.test` |

**实测记录（2026-11-21）**。在 WSL 2 的 Ubuntu 26.04 LTS 下，以普通用户运行上面的配置和三条请求。NGINX 版本为 1.28.3，启动前端口空闲，配置检查通过。三次响应的正文、状态码和命中的站点均与上表一致。访问日志如下：

```text
host=shop.example.test server=shop.example.test uri=/healthz status=200
host=unknown.example.test server=fallback.example.test uri=/healthz status=404
host=shop.example.test server=shop.example.test uri=/ status=200
```

第二次的 404 是预期结果。若第一次也得到 `default`，先比对日志中的 `host=` 与 `server=`；若站点正确但返回 `shop-home`，再查 `uri=` 是否确实为 `/healthz`（末尾多一个斜杠也会走另一条路径）。若收到了响应而实验日志没有新增记录，应核对请求目标及占用 18080 的监听进程。

### 3. 停止本次实验

**影响范围：以下命令会停止本次启动的实验实例。** 在启动它的 Shell 中确认 `$lab` 仍指向刚创建的临时目录，再使用同一配置中的 PID 文件发送优雅退出信号。

```bash
printf '实验目录：%s\n' "$lab"
nginx -p "$lab/" -c "$lab/nginx.conf" -s quit
```

待已有连接结束后，`$lab/logs/nginx.pid` 应消失，再请求该端口应收到连接失败。临时目录中的配置和日志可留作对照；如需删除，先确认目录路径和内容均属于本次实验。[NGINX 命令行参数](https://nginx.org/en/docs/switches.html) · [进程控制与优雅退出](https://nginx.org/en/docs/control.html)

本次实测确认实验 PID 文件消失、18080 端口释放，再次运行 curl 的退出码为 7（无法连接）。

## 来源

原图与规则核验日期为 2026-11-21。[ProcessOn 原图 · NGINX 处理请求流程](https://www.processon.com/view/link/60aefce8f346fb715d52b6a9) · [NGINX 官方请求处理说明](https://nginx.org/en/docs/http/request_processing.html) · [NGINX 监听地址与默认站点](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)
