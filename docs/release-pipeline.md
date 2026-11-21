# 流水线已经绿了，为什么用户仍然访问到旧版本？

预生产验证通过，生产 Deployment 也显示可用，但用户的版本接口偶尔返回旧提交号。沿图上方追一次制品，沿下方追一次请求，两条证据会在具体的生产实例汇合。

![上方从 Git 提交经 Jenkins、镜像仓库、预生产验证进入生产；下方用户请求经业务入口到达同一个生产应用](../diagrams/release-pipeline.svg)

## 按图走一遍

1. **从提交找到制品。** 让构建记录同时包含完整 Git 提交号和推送成功后的镜像摘要。即使再次使用相同标签构建，具体内容仍可依据摘要核对。将提交号编入应用的只读版本响应，便于从请求结果追回源码。[Kubernetes 镜像名称与摘要](https://kubernetes.io/docs/concepts/containers/images/#image-names) · [Jenkins 流水线阶段](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)
2. **用同一制品依次走过预生产和生产。** 预生产验证与生产部署都引用同一份镜像摘要，环境差异通过各环境配置注入。图中从预生产指向生产的箭头表示通过验证后放行该制品。应用实例启动时仍从仓库拉取镜像。
3. **从请求找到实例。** 入口经 Service 或负载均衡将请求交给可接流量的实例。滚动发布期间，新旧副本可能短暂并存。以返回的构建版本、请求 ID、就绪端点和日志核对用户实际命中了谁。[Deployment 滚动更新](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment) · [Service 端点](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)

## 原图修订

| 类型 | 原图线索 | 调整与取证 |
|---|---|---|
| 保留并细化 | Git、Jenkins、预生产、生产处于同一张大图 | 拆出发布线与用户流量线。分别保留构建成功的记录、生产就绪副本数、实际业务入口的响应，才能找到版本从哪一步开始不一致。[Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/) |
| 名称更新 | 缓存节点写作 `rds for redis` | 阿里云托管 Redis 的现行产品名为「云数据库 Tair（兼容 Redis）」，Redis 开源版继续提供服务。数据库和缓存属于业务依赖，名称变更无需改动这条发布路径。[阿里云品牌合并公告](https://help.aliyun.com/zh/redis/product-overview/notice-apsaradb-for-redis-and-tair-have-been-merged) |
| 适用条件 | 预生产注明“与生产一致” | 核对制品摘要、启动参数和关键功能开关；将流量规模、实例数、外部服务地址等差异写入发布记录。预生产通过后仍需验证生产入口。 |

## 动手验证

以下用 Kubernetes 承载生产应用。示例命令只读。示例集群、命名空间、标签和版本接口均需替换，先核对 `kubectl config current-context`。在基于 ECS 的部署中可按同样顺序对照制品校验和、实例上的版本清单和入口响应。

### 1. 在 Jenkins 构建记录或当次工作目录核对来源

```bash
# 在该次构建确实检出的 Git 工作目录内执行。
git rev-parse --verify HEAD
```

把输出与该次构建日志里的提交号核对，并从镜像推送阶段取出其 `sha256:…` 摘要。稍后从开发机查询同名分支的最新提交，可能已经指向另一次变更。

### 2. 在运维终端核对生产声明、运行实例和接流量端点

```bash
ns=shop-prod
deploy=shop-api
service=shop-api
selector='app=shop-api'

kubectl config current-context
kubectl -n "$ns" get deployment "$deploy" \
    -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
kubectl -n "$ns" rollout status "deployment/$deploy" --timeout=20s
kubectl -n "$ns" get pods -l "$selector" \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .status.containerStatuses[*]}{"  "}{.name}{"\t"}{.ready}{"\t"}{.image}{"\t"}{.imageID}{"\n"}{end}{end}'
kubectl -n "$ns" get endpointslices -l "kubernetes.io/service-name=$service" \
    -o jsonpath='{range .items[*].endpoints[*]}{.targetRef.name}{"\t"}{.addresses[0]}{"\tready="}{.conditions.ready}{"\n"}{end}'
```

若使用不可变镜像引用，Deployment 的业务容器镜像应与发布记录的 `仓库@sha256:…` 一致。多容器 Pod 要按容器名对齐字段。运行时的 `imageID` 格式随实现变化，多架构索引及其平台镜像也有不同摘要；先核对平台及镜像清单，再判定内容差异。通过 Service 选择器、EndpointSlice 的 Pod 名和就绪状态核对实际后端，排除误选了另一个 Deployment 的 Pod。[镜像摘要](https://kubernetes.io/docs/concepts/containers/images/#image-names) · [检查服务后端](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/#debugging-services)

### 3. 从真实入口核对版本

```bash
# 替换为该应用已提供、按策略绕过缓存的只读版本接口。
version_url=https://shop.example.test/version
for sample in 1 2 3; do
    date -u '+%FT%TZ'
    curl -sS --max-time 5 -D - "$version_url"
    printf '\n'
done
```

| 现象 | 下一项检查 |
|---|---|
| Deployment 声明仍是上一版 | 回查 Jenkins 中生产发布阶段是否执行，目标集群、命名空间和渲染后的镜像引用是否正确。 |
| 声明已更新，部分旧副本仍接流量 | 核对发布状态、就绪探针、EndpointSlice 及是否处于有意保留旧版本的灰度阶段。 |
| 所有目标后端已更新，入口响应仍旧 | 查入口的 Host/路由规则、CDN 或反向代理缓存、后端访问日志与请求 ID。连续三次相同响应只代表这些样本；会话粘滞可令多次请求命中同一实例。 |

本次未连接 Jenkins 或生产集群。表格列出预期的判断依据，现场发布结果需以实际摘要、端点和响应为准。

## 来源

核验日期为 2026-11-21。原图同时展示用户访问和开发上线，本例补入容器镜像摘要和 Kubernetes 检查点。[ProcessOn 原图 · 综合架构面试第 1 套](https://www.processon.com/view/link/60a764ade401fd3681f6f4fd)
