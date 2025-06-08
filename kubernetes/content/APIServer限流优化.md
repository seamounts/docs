# APIServer 限流优化

## 背景与挑战
在 Kubernetes 集群中，API Server 是控制平面的核心组件，负责处理所有对集群的请求。在高并发或恶意请求的情况下，API Server 可能成为性能瓶颈，影响整个集群的稳定性。因此，实施有效的限流策略对于保障集群的高可用性至关重要。

## 限流机制详解
### 基础限流参数
Kubernetes 提供了以下两个参数用于基础限流：
- --max-requests-inflight：限制同时处理的非变更类请求（如 GET、LIST、WATCH）的最大数量。
- --max-mutating-requests-inflight：限制同时处理的变更类请求（如 POST、PUT、PATCH、DELETE）的最大数量。
这些参数通过限制并发请求数，防止 API Server 被过载。

### API 优先级与公平性（APF）
1. API Server 接收到请求后，根据 FlowSchema 的规则对请求进行分类。
2. 每个 FlowSchema 关联一个 PriorityLevelConfiguration，定义该类请求的处理策略。
3. 请求被分配到对应的队列中，按照优先级和公平性进行处理。

通过 APF，可以实现基于用户、命名空间、资源类型等多维度的限流控制，提高系统的稳定性和公平性。

## 配置示例
以下是一个配置示例，展示如何使用 APF 实现限流控制：
FlowSchema 示例
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1beta1
kind: FlowSchema
metadata:
  name: high-priority
spec:
  matchingPrecedence: 100
  priorityLevelConfiguration:
    name: high-priority
  rules:
    - subjects:
        - kind: Group
          group:
            name: system:masters
      verbs: ["*"]
      resources:
        - group: "*"
          resources: ["*"]
      clusterScope: true
  distinguisherMethod:
    type: ByUser
```

PriorityLevelConfiguration 示例
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1beta1
kind: PriorityLevelConfiguration
metadata:
  name: high-priority
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 100
    limitResponse:
      type: Queue
      queuing:
        queues: 10
        handSize: 5
        queueLengthLimit: 50
``` 

## 优化效果
通过实施上述限流策略，可以实现以下优化效果：
- 提升稳定性：防止单一用户或服务过度占用资源，保障系统稳定运行。
- 提高公平性：确保不同用户或服务的请求得到公平处理，避免资源争用。
- 增强可控性：管理员可以根据实际需求，灵活配置限流策略，满足多样化的业务场景。

通过合理配置 API Server 的限流机制，特别是利用 APF 提供的细粒度控制能力，可以有效提升 Kubernetes 集群的性能和稳定性。建议在实际部署中，根据具体业务需求和集群规模，逐步实施并持续优化限流策略。