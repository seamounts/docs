## 背景与挑战
在 Kubernetes 集群中，Kubelet 负责管理 Pod 所需的 ConfigMap。默认情况下，Kubelet 使用 Watch 机制监听 ConfigMap 的变化，并将更新同步到本地缓存。然而，在大规模集群中，频繁的 Watch 操作可能导致 API Server 的负载增加，影响集群性能。

## 缓存更新策略详解

Kubelet 提供了三种 ConfigMap 缓存更新策略，可通过配置 ConfigMapAndSecretChangeDetectionStrategy 参数进行设置：
1. Watch（默认策略）
机制：Kubelet 通过 Watch 机制实时监听 ConfigMap 的变化。
优点：更新实时性高。
缺点：在大规模集群中，Watch 数量众多，可能导致 API Server 负载过高。
2. TTLCache
机制：Kubelet 定期（根据 TTL 设置）从 API Server 获取 ConfigMap 的最新值。
优点：减少了 Watch 的数量，降低了 API Server 的负载。
缺点：更新存在延迟，具体取决于 TTL 的设置。
3. Get
机制：Kubelet 每次直接从 API Server 获取 ConfigMap 的值，不使用缓存。
优点：始终获取最新值。
缺点：频繁访问 API Server，可能导致其负载增加。
在大规模集群中，推荐使用 TTLCache 策略，以在更新延迟和 API Server 负载之间取得平衡。

