---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# 故障注入

故障注入配置是用来模拟上游服务对请求响应行为的一种手段。

在一个微服务架构的服务群中，一定存在部分服务有着较高的健壮性要求，比如电商中的订单系统，当订单相关的服务出现异常时，一般需要第一时间通知维护人员，这就要求对于服务的异常处理以及监听方面要求很高。从运维方面，我们可以对服务订单服务的相关主机或者云虚拟主机的 CPU，内存等重要指标进行监听，若发现异常，以短信、消息或者邮件的方式告知。另一方面，若程序内部访问异常，那么服务也要第一时间通知到维护人员。而这些异常在开发测试阶段人为干涉或者模拟是一个非常繁杂的工作，而 Istio 提供的 HTTPFaultInjection 就是为了解决在开发测试节点调试难问题而设计的。它可以用来对服务进行故障注入，大家可以通过配置不同的 HTTPFaultInjection 策略来检测自己服务的健壮性以及对于异常处理的能力而不需要花费过多的精力。

**HTTPFaultInjection** 是 VirtualService 中的一种规则，它包括了两种类型的故障注入：

- **abort**：非必配项，配置一个 Abort 类型的对象。用来注入请求异常类故障。简单的说，就是认为模拟上游服务返回异常时，我们自己的服务是否具备处理能力。
- **delay**：非必配项，配置一个 Delay 类型的对象。用来注入延时类故障。通俗一点讲，就是人为模拟上游服务的响应时间，看在这个时间内无响应的情况下，我们自己的服务是否具备容错容灾的能力。

### HTTPFaultInjection.Abort：

- **httpStatus**：必配项，是一个整型的值。表示注入 HTTP 请求的故障状态码。
- **percentage**：非必配项，是一个 Percent 类型的值。表示对多少请求进行故障注入。如果不指定改配置，那么所有请求都将会被注入故障。
- **percent**：已经过期的一个配置，与 percentage 配置功能一样，已经被 percentage 代替。

如下的配置表示对 `v1` 版本的 `ratings.prod.svc.cluster.local` 服务访问的时候进行故障注入，`0.1`表示有千分之一的请求被注入故障， `400` 表示故障为该请求的 HTTP 响应码为 `400` 。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
    fault:
      abort:
        percentage:
          value: 0.1
        httpStatus: 400
```

### HTTPFaultInjection.Delay：

- **fixedDelay**：比配置项，表示请求响应的模拟处理时间。格式为：`1h/1m/1s/1ms`， 不能小于 `1ms`。
- **percentage**：非必配项，是一个 Percent 类型的值。表示对多少请求进行故障注入。如果不指定改配置，那么所有请求都将会被注入故障。
- **percent**：已经过期的一个配置，与 `percentage` 配置功能一样，已经被 `percentage` 代替。

如下的配置表示对 `v1` 版本的 `reviews.prod.svc.cluster.local` 服务访问的时候进行故障注入，`0.1` 表示有千分之一的请求被注入故障，`5s` 表示故障为该请求的HTTP配置 `5s` 的延迟。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - match:
    - sourceLabels:
        env: prod
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
```

> 故障注入一般适用在开发测试阶段，在上线前一定要对配置文件做检查校正，否则，你可能要被老板请去喝茶。

