# SOCI Snapshotter 并行模式（Parallel Mode）解析与实践

## 背景

传统的镜像拉取流程是 顺序执行 的：
1. 拉取 layer 压缩文件
2. 解压缩
3. 写入 snapshotter

对于大镜像（如 AI 训练环境，动辄几十 GB），这种顺序模式会导致下载和解压操作串行化，启动容器非常慢。

**SOCI Snapshotter Parallel Mode** 的目标就是 并行化下载与解压，充分利用网络带宽和 CPU I/O，缩短大镜像的准备时间。

## 核心思路
Parallel Mode 主要做了两件事：
1. Seekable 镜像层（可随机访问）
    - 基于 SOCI Index，把镜像层（tar.gz）切分成多个可以随机访问的“块”。
    - 这样 snapshotter 就能并行地请求镜像 registry 上的不同字节范围（Range Request）。

2. 并行下载 + 并行解压
    - 多个 goroutine 同时下载不同的数据块；
    - 下载完成的数据块立即进入解压队列，不再等待整个层下载完成；

这种模式避免了“先等下载完 → 再解压”的长尾瓶颈。

## 优势
- 显著降低大镜像的冷启动时间：官方测试中，几十 GB 的镜像拉取可从 分钟级降低到几十秒。
- 最大化资源利用：带宽和 CPU 不再相互等待。
- 兼容懒加载（lazy pull）：如果镜像不需要完整下载，按需加载的场景仍然能工作。

## 配置方法
要在 Kubernetes 集群中启用 Parallel Mode，需要做以下步骤：
1. 安装 SOCI Snapshotter
    - 作为 containerd 的一个插件运行（CRI 插件 snapshotter）。
    - 一般通过 DaemonSet 部署到所有节点。

2. 在 containerd 配置中指定 SOCI 作为 snapshotter

    ```toml
    [proxy_plugins]
    [proxy_plugins.soci]
        type = "snapshot"
        address = "/run/containerd-soci/soci.sock"
    ```

3. 启用 Parallel Mode

    ```toml
    [plugins."io.containerd.snapshotter.v1.soci"]
    parallel_mode = true
    max-concurrency = 8     # 最大并行下载线程数
    max-decompression = 4   # 最大并行解压线程数
    buffer-size = "64MB"    # 下载缓冲区大小
    ```

4. 构建带索引的 SOCI 镜像

    基于已有镜像构建 SOCI 镜像索引
    假设已有镜像：
        ```
        myregistry.example.com/ml/train:latest
        ```
    运行命令生成 SOCI 索引：
        ```
        soci create myregistry.example.com/ml/train:latest
        ```
    该命令会：拉取镜像的 layer，为每个 layer 生成 TOC（table of contents）索引文件，最后将索引文件 push 到镜像 registry。
    原有镜像保持不变，只是额外生成索引。


5. 在 Pod 配置中指定使用 SOCI Snapshotter

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: test-soci
    spec:
    containers:
    - name: app
        image: <your-image>:<tag>
    runtimeClassName: soci-runc
    ```

    当节点上的 containerd 配置了 SOCI snapshotter 并开启 parallel mode 时：

    - 如果镜像仓库中存在 SOCI Index，SOCI snapshotter 就会 并行拉取和解压
    - 如果没有 Index，则退回传统拉取模式


## SOCI Snapshotter 在 containerd 架构中的位置

- 在 containerd 里，snapshotter 是一个可插拔组件（比如 overlayfs、btrfs、soci 等）。
- 可以通过 containerd config.toml 指定默认使用哪个 snapshotter。

SOCI Snapshotter 的模式是：
- 代理型 snapshotter：它自己调用 containerd 的下层 snapshotter（通常是 overlayfs），并在其中插入“按需拉取 / 并行拉取”逻辑。
- 如果启用了 SOCI 作为默认 snapshotter，所有镜像拉取和解压都会先走 SOCI。

### 如果 SOCI Snapshotter 插件挂了, 会发生什么?

情况 A：containerd 配置了 SOCI 作为默认 snapshotter
- containerd 会调用失败（因为默认的 snapshotter 不可用），此时所有镜像拉取都会受影响，包括没有 SOCI Index 的镜像。
- 等同于 snapshotter 整个链路断掉，Pod 无法启动。

情况 B：overlayfs 是默认 snapshotter，SOCI 仅作为附加 snapshotter
- 这种情况下，如果没有显式在 Pod RuntimeClass 里指定用 SOCI（例如 runtimeClassName: soci-runc），镜像拉取就直接走 overlayfs，不会受 SOCI 故障影响。
- 只有那些显式选择了 SOCI 的 workload 会受到影响。

### 推荐做法
1. overlayfs 作为默认 snapshotter（兜底稳定性）。
2. 在 workload 需要加速时（例如 AI 大镜像、Serverless 冷启动）使用 RuntimeClass 选择 SOCI snapshotter。
3. SOCI 插件挂掉时，不会影响大多数业务，只影响 opt-in 的那部分。

## 总结
- 传统模式：下载 → 解压 → 写 snapshot，完全串行，冷启动慢。
- SOCI Parallel Mode：把 layer 切成多个块，并行下载和解压，充分利用资源，缩短启动时间。
- 价值：显著降低大镜像冷启动延迟，提升 Kubernetes 集群的镜像分发效率。



SOCI Snapshotter 并行模式（Parallel Mode）解析与实践