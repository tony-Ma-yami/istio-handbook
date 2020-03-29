---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# 熔断

随着微服务架构模式的兴起，熔断技术已经是为保证服务 SLA 的一种关键手段，它的弹性熔断机制显的越来越重要。相信熟悉 Spring Cloud 的架构模式的同学对`Spring Cloud Hystrix`也不陌生，它是第一个将熔断理念落地并形成产品的组件，它里面的线程隔离，信号量隔离机制以及对于请求连接池，最大请求拒接队列的设计策略无疑是完善的，可惜的是，它的配置往往需要同`Spring Cloud Ribbon`的配置参数以及`HTTP connection Pool`参数一起调试，这就导致了参数调优的过程是相当艰难的，而且笔者在压力测试的过程中发现`Spring Cloud Hystrix`对请求处理的稳定性很差。

Istio 中使用的熔断机制实际上是 Envoy 熔断机制。它是通过 DestinationRule 中的`ConnectionPoolSettings`与`OutlierDetection`配置来实现。

在下面的配置中，配置了下游服务连接`httpbin`服务的一些配置。

配置连接池中最大连接数为`1`，最大请求等待数为`1`，每个连接处理请求的最大值为`1`，即不启用`Keep-alive`属性。

异常点检测配置在`1s`的时间内若连续收到`1`个请求返回`5xx`则触发熔断，第一次熔断的时间`3`分钟，对`httpbin`服务池最大的熔断占比配置为`100%`，也就是说会将所有的`httpbin`服务熔断。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
```

> **提示**： 这里需要注意的是，Envoy 对应熔断的处理时存在一些小误差的。但整体上的表现还是符合配置的参数的。

### OutlierDetection：

OutlierDetection 是 Istio 框架对`Circuit breaker`的一种实现。它的功能类似于 `Spring Cloud Hystrix` 。 他们的基本原理都是当请求压力超过服务能够承受的最大压力时，需要及时的熔断服务以保证服务端不会奔溃。在微服务的架构里，熔断机制是非常重要的一环，它保证了当流量过大时可以保证在服务支撑能力之内的一部分请求是可以正常运转的，而不至于由于某一个服务的奔溃而导致微服务的雪崩效应。Istio 构架设计中，当服务端按照一定规则返回`5xx`的请求码时将会触发熔断机制。比如`502`,`503`,`504`等。这里需要注意的是`500`状态码不会触发熔断机制。Istio团队认为`500`属于服务内部错误，不在熔断检测的范畴之内。

- **consecutiveGatewayErrors**： 非必配项，配置一个UInt32Value类型的值。表示从连接池连续返回的网关类型错误的个数，这些错误包括`502`,`503`,`504`等状态，注意 consecutiveGatewayErrors 与 continuous5xxerrors 是可以单独使用或者一起使用的，因为 consecutiveGatewayErrors 的错误也属于 continuous5xxerrors， 所以当 consecutiveGatewayErrors 发生次数加1的时候，continuous5xxerrors 的值也会加 1.同时当 consecutiveGatewayErrors 的值大于等于 continuous5xxerrors 的值时，continuous5xxerrors 配置的值将优先起作用。当它的值被设置为 `0` 时，表示关闭此项配置。
- **consecutive5xxErrors**：非必配项，配置一个 UInt32Value 类型的值。表示连续从连接池返回的`5xx`请求码的个数。这些错误包括`502`,`503`,`504`等状态，同理，它可以与 consecutiveGatewayErrors 一起使用或单独使用。该配置的默认值是`5`，它不能被设置成`0`。
- **interval**：非必配项，配置一个 Duration 类型的值。表示驱逐服务的时间间隔。配置格式为：1h/1m/1s/1ms， 最小不能小于1ms。通俗讲就是讲就是计算各个异常点检测占比数时候取的时间单位。
- **baseEjectionTime**：非必配项，配置一个 Duration 类型的值。表示服务被驱逐的驱逐时间基量，`服务被驱逐的时间 = 最小驱逐时间间隔 * 驱逐次数`。这就是为什么一个服务被驱逐的次数越多，则驱逐的时间越长的原因。所以这里 baseEjectionTime 也可以表示 服务第一次被驱逐时驱逐的时长。
- **maxEjectionPercent**：非必填项，配置一个 int32 类型的值。表示最大驱逐服务的在连接池中的占比。它的默认值为`10%`， 表示不管流量压力多大，其中最多只有`10%`的服务被驱逐。
- **minHealthPercent**：非必填项，配置一个 int32 类型的值。表示服务连接池中最小健康的服务占比，当服务连接池中健康的服务占比大于该值的配置时，才会开启熔断检测功能。当该值设置成`0%`时则表示该项配置无效，即直接开启异常点检测功能。

在下面的例子中，定义了一个异常点检测的规则，它表示计算 `reviews.prod.svc.cluster.local` 服务在`5`分钟之内它返回`5xx`异常的错误数，当连续`7`次返回时驱逐该服务，第一次驱逐的时间为`15`分钟。当第一次驱逐时间结束后，该服务将继续转发流量，当第二次检测又发现有连续`7`次返回`5xx`错误时，再次驱逐它，这次它的驱逐时间将变成 `15 * 2 = 30分钟`。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-cb-policy
spec:
  host: reviews.prod.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 7
      interval: 5m
      baseEjectionTime: 15m
```

> **提示**： consecutiveGatewayErrors 配置与 consecutive5xxErrors 配置是 Istio 1.5 新增的配置，而在1.5版本之前，只有一个参数叫做 consecutiveErrors。 也就是说，1.5版本对返回5xx的错误设置了一个子集。这样在特定的坏境中，对于熔断检测的控制粒度将更精细。

