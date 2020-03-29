---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# 路由

当一个请求到来时， 我们需要根据请求的URL， 请求方式等来将该请求转发、重写、重定向到不同的业务服务当中，这个过程就叫做路由转发， 在 Spring Cloud 时代，我们使用 `Zuul` 来充当网关来做路由转发，或者后来 Spring Cloud 自己出了 `Spring Cloud Gateway` 组件来实现路由转发， 同样在 Istio 中，也有组件来做路由转发，而不同的是，Istio 不仅仅只在路由转发上发力，与此同时，它在此基础之上， 而实现了多版本的控制， 故障注入， 熔断检测等一系列功能。它的核心资源包括 `VirtualService`， `DestinationRule`， `ServiceEntry`， `Gateway` 等。

**VirtualService**：主要作用为Istio的路由规则配置集合以及请求协议的选择。多与DestinationRule一起使用。

**DestinationRule**：主要作为目标主机地址，多版本控制，熔断即负载均衡选择等。

**ServiceEntry**：主要作用为将网格外部服务注册到Istio的服务注册表中去，使用Istio统一管理。

**Gateway**：主要作用为用户微服务集的网关控制。 

