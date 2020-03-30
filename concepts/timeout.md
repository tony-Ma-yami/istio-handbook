---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# 超时

请求超时配置是用来规定等待上游服务响应时间最大时间。如果在配置的时间范围内没有收到响应体，那么则抛出请求超时的错误。

请求超时可以使用两种方式来实现：

第一种：使用在 VirtualService 中`timeout`关键字来进行配置的。

第二种：在 Envoy 代理架构下，在请求头中加入 `x-envoy-upstream-rq-timeout-ms`标头即可。这里配置的时间单位为毫秒。

比如在下面的配置中，配置了`v2`版本的`reviews`服务请求超时时间为`5s`，表示请求访问`v2`版本`reviews`服务的最大等待时间为`5s`。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
    timeout: 5s

```

**提示**：在使用请求超时配置的时候，可以配合 VirtualService 的 HTTPFaultInjection 配置进行测试，调试起来会更香。

