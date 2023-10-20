+++
title = "为 bitnami/kafka 添加内网 SLB 的一次经历"
date = "2023-10-19"
author = "Jeff"
cover = "https://image.0xbb.dev/2023/10/202310192313330.jpeg"
description = "bitnami/kafka 外部访问从 NodePort 到 LoadBalancer + IP 再到 LoadBalancer + Domain"
+++

首先说一下这次记录是基于 bitnami/kafka 20.0.4 版本下操作的，不同版本请检查差异。

## Kafka 的外部访问方式

https://artifacthub.io/packages/helm/bitnami/kafka/20.0.4#accessing-kafka-brokers-from-outside-the-cluster

从上述的文档中可以得知，支持三种方式可以配置

- LoadBalancer
- NodePort
- ClusterIP

我们第一次其实还没有选择使用 LoadBalancer，因为 NodePort 更简单。

## 尝试 NodePort

Edit values.yaml
```yaml
externalAccess:
  enabled: false
  service:
    type: NodePort
    ports:
      external: 9094
    nodePorts:
      - 19094
      - 19095
      - 19096
    useHostIPs: true
```

helm install 之后，k8s service 会产生 3 个 external service 负责提供 node port 访问，然后 kafka client 使用 node_ip:node_port 的方式连接即可

但，这种方式配置在客户端是固定地址的，比如你会随便选择 3 个 node ip，万一某个 node 不可用了，你需要找到所有的客户端一个个修改新的 node ip，这种故障恢复起来对于客户端越多成本会越高，如果客户端少的话，node port 的方式问题不大

为了做到快速恢复，我们需要再尝试另外一种方案：**LoadBalancer**

## 尝试 LoadBalancer

配置 `LoadBalancer` 之前你需要先手动创建 3 个 k8s lb service，然后拿到 3 个 IP 提供给接下来的配置使用。

Edit values.yaml
```yaml
externalAccess:
  enabled: false
  service:
    type: LoadBalancer
    ports:
      external: 9094
    loadBalancerIPs:
      - ip1
      - ip2
      - ip3
```

helm install 之后，k8s service **可能**会创建 3 个 LoadBalancer 的 external service，而且 external service 的 IP 会跟你配置的 `loadBalancerIPs` 不同，所以你应该删除这 3 个 LoadBalancer 的 external service。

如果你不想手动创建 k8s lb service，你可以先用 `loadBalancerNames` 配置好域名，不解析 IP，helm install 自动创建的 k8s lb service 得到 IP 之后再做域名解析即可。

为什么用域名？

- 防止 LB 故障，你还是得改客户端配置，有了域名你就可以解析到新的 LB IP 解决问题
- 可以做到 3 个域名绑定到一个 LB 上适当降低成本

## 尝试用一个 LB

首先手动创建 1 个 k8s lb，用于接下来的域名解析和 `loadBalancerNames` 配置使用

Edit values.yaml
```yaml
externalAccess:
  enabled: false
  service:
    type: LoadBalancer
    ports:
      external: 9094
    loadBalancerNames:
      - domain1
      - domain2
      - domain3
```

因为 kafka 启动时会在 zookeeper 上注册 broker address，所以使用一个 LB 你填一个 IP 的话会启动失败，因为 broker address 不能重复。

那么我们可以通过申请 3 个域名来解决这个问题，实际上这 3 个域名都解析到一个 LB 上。

## 总结

bitnami/kafka 提供了 3 中外部访问的配置方式，我们可以根据不同的需求选择不同的配置方式

- 测试环境或者客户端少的情况下我们可以选择使用 NodePort
- 使用 LoadBalancer 可以让我们在客户端多的情况下做到故障快速恢复
- 利用域名可以降低 LB 数量、与 IP 解藕
