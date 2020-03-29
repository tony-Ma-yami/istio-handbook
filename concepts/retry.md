---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# 重试

HTTP请求重试不是什么新鲜技术，在 Istio 架构中，HTTP 请求是通过配置在 VirtualService 中的 HTTPRetry 配置来实现的。

**HTTPRetry**：

- attempts：必配项，代表重试次数，表示当请求失败后经过多少次重试之后放弃。
- perTryTimeout：非必配项，表示重试请求的超时时间，格式为：1h/1m/1s/1ms， 最小不能小于1ms。
- retryOn：非必配项，表示在什么情况下进行重试，如果有多个重试条件，使用 “，” 隔开。更多详细配置参考：https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-on 、https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-grpc-on。

以下配置表示：对 `ratings.prod.svc.cluster.local` 服务发送的请求，当失败时，最多进行`3`次重试，重试间隔时间为 `2s`，并且只有当出现 `gateway-error`，`connect-failure`，`refused-stream` 这三种异常时才进行重试。

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
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error，connect-failure，refused-stream
```

