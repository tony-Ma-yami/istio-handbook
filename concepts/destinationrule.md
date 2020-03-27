---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# DestinationRule

DestinationRule 是 Istio 中定义的另外一个比较重要的资源， 它通常和 VirtualService 一起使用，来确定当请求路由来临的时候应该访问哪一个 Service 服务，同时也来控制负载均衡的调配，负责对具体的 Endpoint 中 Envoy 代理的 Sidecar 进行连接，并且提供异常点检测，健康检测，熔断控制等。先通过几个例子来认识一下它吧。

下面便是一个简单的配置例子：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN

```

从上面的配置中可以看出，在它的 metadata 中定义了的 name 叫做 “bookinfo-ratings”，这个 name 通常被使用在 VirtualService 的 destination 中。它定义的 host 为 “ratings.prod.svc.cluster.local”，表示流量将被转发到 ratings.prod 这个服务中去。它的路由的全局负载均衡策略是 LEAST_CONN（最少连接）策略。

下面列举出第二个例子：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

这个例子也是一个 DestinationRule 的配置，不同于上一个例子的是，在这个例子中，我们定义了 ratings.prod 服务的全部路负载均衡策略是 LEAST_CONN（最少连接）策略，但是在具体的 subsets 又配置了如果匹配到 v3 版本的 ratings.prod 服务时，它使用 ROUND_ROBIN（轮询）的负载均衡策略。

接下来，我们介绍下 DestinationRule 中的配置项：

### ConnectionPoolSettings：

ConnectionPoolSettings 用来配置与服务的 Sidecar 代理 Envoy 之间的连接属性。包括最大连接数，连接超时时间，连接保持时间等一系列的配置，这些配置实际上 Envoy 的配置。更详细的配置说明可参考 https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking。

- **tcp**：非必配项，配置一个 TCPSettings 类型的值，它包括HTTP与TCP协议的公共连接配置。
- **http**：非必配项，配置一个 HTTPSettings 类型值，它用来专门对HTTP协议的连接进行配置。

这里需要注意的， tcp 中的部分配置，也会作用在HTTP协议的请求，比如 maxConnections 配置等。

下面这个配置表示对 myredissrv.prod.svc.cluster.local 服务进行连接池的一些配置，表示最大连接数为100，连接超时时间为 30ms。连接保持时间为 7200s。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-redis
spec:
  host: myredissrv.prod.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
```

### ConnectionPoolSettings.HTTPSettings：

针对HTTP1.1 / HTTP2 / GRPC的连接配置属性。

- **http1MaxPendingRequests**： 非必配项，配置一个int类型的值。表示 HTTP请求的最大请求队列数。默认值为 2^32 - 1 。
- **http2MaxRequests**： 非必配项，配置一个int类型的值。表示HTTP2协议的最大请求数。默认值为 2^32 - 1 。
- **maxRequestsPerConnection**： 非必配项，配置一个int类型的值。每个连接处理请求的最大个数， 如果将该值设置为1，则表示禁用request请求连接的 keepalive属性。如果设置为 0 ， 表示不限制。
- **axRetries**： 非必配项，配置一个int类型的值。表示请求最大重试次数。
- **idleTimeout**： 非必配项， 配置一个Duration类型的值，表示连接的空闲回收时间。默认时间为1个小时，即如果该连接在1个小时之内没有处理请求，则关闭该连接。
- **h2UpgradePolicy**：指定如果HTTP1.1 协议的连接升级到HTTP2协议时的策略。

### ConnectionPoolSettings.HTTPSettings.H2UpgradePolicy：

HTTP1.1 协议的连接升级到HTTP2协议时的策略选择。

- **DEFAULT** ： 表示使用全局默认策略。
- **DO_NOT_UPGRADE**： 不允许升级到HTTP2协议。
- **UPGRADE**： 升级到HTTP2协议。

### ConnectionPoolSettings.TCPSettings：

连接池TCP协议相关的一些配置。

- **maxConnections**： 非必配项，配置一个int类型的值，表示最大连接数。默认是2^32-1。 注意这个配置既作用于TCP协议的连接上，也作用在HTTP1协议的连接上。
- **connectTimeout**： 非必配项，配置一个 Duration 类型的值。表示TCP连接的超时时间。
- **tcpKeepalive**： 非必配项，配置一个 TcpKeepalive 类型的值，用来配置keep-alive属性。

### ConnectionPoolSettings.TCPSettings.TcpKeepalive：

用于配置TCP keep-alive属性。

- **probes**：非必配项，配置一个int类型的值，表示连接存活探针最大探测次数，默认使用操作系统自己的配置。比如 Linux 的配置次数为 9 。
- **time**：非必配项，配置一个 Duration 类型的值。表示连接在空闲多长时间之后开始发起存活探针检测。默认使用操作系统自己的配置。比如 Linux 的配置次数为 7200s 。
- **interval**：非必配项，配置一个Duration类型的值。表示探针检测间隔时间。默认使用操作系统自己的配置。比如 Linux 的配置次数为 75s。

### DestinationRule：

DestinationRule 是接收到请求之后，指定请求转发路由的最高层配置。它具体有以下这些属性：

- **host**：必配项，配置一个string类型的值。是一个在服务注册中心注册过的 Service 名称。这个服务可以是 Kubernetes 的 services, 或者 Consul 中的 services 等。同样也可以是一个 ServiceEntry 类型的名称。这里需要注意，当我们配置短域名时， Istio 在补全域名的时候将与 DestinationRule 所应用的 Namespace 有关。 比如，我们配置  hosts: reviews， 当它在 default 命名空间下应用时，hosts 将会被补全为 reviews.default.svc.cluster.local， 当它在 test 命名空间下应用时， hosts 将会被补全为 reviews.test.svc.cluster.local 。
- **trafficPolicy**：非必配项，配置一个 TrafficPolicy 类型的值。表示处理请求连接的一些配置。包括负载均衡，连接池配置，以及异常点检测等。
- **subsets**： 非必配项，配置一个 Subset[] 类型的值，表示服务子集。在里面可以指定不同版本的服务以及提供覆盖公共的请求策略的功能。
- **exportTo**： 非必配项，配置一个string[]类型的值，表示该规则应用的 Namespace 。如果不设置，则表示当前 DestinationRule应用到所有的 Namespace下。
  - “.” 表示该 DestinationRule只应用到当前的 Namespace 下。
  - “*” 表示应用到所有的 Namespace下。

### LoadBalancerSettings：

连接请求时的负载均衡配置，它最后将被翻译成 Envoy 的负载均衡策略并应用到每一个 服务的 Endpoint上去。想要了解更多的配置规则，请参考 Envoy 的负载均衡配置文档： https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing。

正如下面这个简单的例子，它使用ROUND_ROBIN（轮询）的负载均衡策略对 ratings.prod.svc.cluster.local 服务进行转发。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN

```

下面这个例子，则表示负载策略使用session粘性的方式去做，而session粘性的Key的hash值使用用户的 cookie 来计算。

```yaml
 apiVersion: networking.istio.io/v1alpha3
 kind: DestinationRule
 metadata:
   name: bookinfo-ratings
 spec:
   host: ratings.prod.svc.cluster.local
   trafficPolicy:
     loadBalancer:
       consistentHash:
         httpCookie:
           name: user
           ttl: 0s
```

LoadBalancerSettings具体配置有：

- **simple**： 必配项，配置一个 SimpleLB 类型的值，表示使用简单负载均衡策略。
- **consistentHash**： 必配项，配置一个 ConsistentHashLB 类型的值，表示配置一个具有session粘性的负载均衡策略，它使用一致性哈希算法进行计算。
- **localityLbSetting**：非必配项，配置一个 LocalityLoadBalancerSetting 类型的值，表示地域性负载均衡策略。

### LoadBalancerSettings.ConsistentHashLB：

一致性哈希算法负载均衡策略配置，即我们通常说的session粘性，它的原理是根据请求的请求头或者请求参数等值进行一定的Hash运算，使用运算得到的Hash值来对服务进行绑定。假如，我们使用请求头中`user_id`来作为一致性Hash算法的key, 由于同一个用户的所有请求头中`user_id`的值都一样，所以根据它算出来的Hash值也是一样的，那么这个用户将会被绑定到只访问某一个服务。

- **httpHeaderName**：必配项，配置一个 string 类型的值，表示用来计算Hash值的 Header 名称。
- **httpCookie**： 必配项，配置一个 HTTPCookie 类型的值，表示用来计算Hash值的 Cookie 名称。
- **useSourceIp**： 必配项，配置一个bool类型的值，表示用来计算Hash值的ip地址。
- **minimumRingSize**： 非必配项，配置一个int类型的值。用来控制Hash算法的精确粒度。默认值1024，值越大，则表示它的哈希算法粒度越精确。

### LoadBalancerSettings.ConsistentHashLB.HTTPCookie：

用来定义Http cookie的一些属性，这些属性将直接作用在Hash值的计算当中。

- **name**：配置 cookie 的名称。
- **path**：配置 cookie 的路径
- **ttl**：设置 cookie 的存活时间。

### LoadBalancerSettings.SimpleLB：

标准的资源负载均衡算法选项。

- **ROUND_ROBIN**： 轮询算法。
- **LEAST_CONN**： 最少连接算法。
- **RANDOM**： 随机访问算法。
- **PASSTHROUGH**：这个选项表示不做任何的负载均衡，将请求发送直接发送到对应的服务中去。

### LocalityLoadBalancerSetting：

LocalityLoadBalancerSetting允许管理员根据不同区域做一些负载均衡的设置。使用这个配置一般可以处理一些请求的故障转移，从而提高系统的高可用性。

比如下面的配置表示从`us-west/zone1`区域发出的请求，将有20%的流量被转发到`us-west/zone2`区域下处理，有80%的流量被转发到`us-west/zone1`区域处理。从`us-west/zone2`区域发出的请求，将有20%的流量被转发到`us-west/zone1`区域下处理，有80%的流量被转发到`us-west/zone2`区域处理。

```yaml
  distribute:
    - from: us-west/zone1/*
      to:
        "us-west/zone1/*": 80
        "us-west/zone2/*": 20
    - from: us-west/zone2/*
      to:
        "us-west/zone1/*": 20
        "us-west/zone2/*": 80
```

下面这个配置表示故障转移，下面配置有效的前提是在`us-east`于`us-west`两个区域都有相同的服务，当us-east区域的请求发送故障或异常时将会被转移到us-east区域的服务上。同理，在`us-west`上配置的区域，如果发生异常将会被转移到`us-east`区域上的服务中去。

```yaml
 failover:
   - from: us-east
     to: eu-west
   - from: us-west
     to: us-east

```

LocalityLoadBalancerSetting有以下几个属性值

- **distribute**：非必配项，与 failover 之间只能配置一个，配置一个Distribute[] 类型的值。在这里描述路由转移的起始地址与接收地址。
- **failover**：非必配项，与 distribute 之间只能配置一个，配置当发生故障是，请求的转移属性。
- **enabled**：非必配项，配置一个 BoolValue 类型的值，表示是否开启。也可以在Istio安装时可以使用`--set global.localityLbSetting.enabled=false` 命令禁止。

### LocalityLoadBalancerSetting.Distribute：

使用 LocalityLoadBalancerSetting.Distribute 来指定原始区域与目标区域以及请求负载的权重。

- **from**： 非必配项，配置一个string类型的值，表示源地址。
- **to**： 非必配项，配置一个map<string, uint32>类型的值，表示目标地址以及每个目标地址的权重。

### LocalityLoadBalancerSetting.Failover：

使用 LocalityLoadBalancerSetting.Failover 来指定流量的故障转移策略。可以指定区域间的故障转移。

- **from**：故障开始区域名
- **to**：要将请求转移到的区域名

### OutlierDetection：

OutlierDetection 是 Istio 框架对`Circuit breaker`的一种实现。它的功能类似于 Spring Cloud Hystrix 。 他们的基本原理都是当请求压力超过服务能够承受的最大压力时，需要及时的熔断服务以保证服务端不会奔溃。在微服务的架构里，熔断机制是非常重要的一环，它保证了当流量过大时可以保证在服务支撑能力之内的一部分请求是可以正常运转的，而不至于由于某一个服务的奔溃而导致微服务的雪崩效应。Istio构架设计中，当服务端按照一定规则返回5xx的请求码时将会触发熔断机制。比如`502`,`503`,`504`等。这里需要注意的是`500`状态码不会触发熔断机制。Istio团队认为`500`属于服务内部错误，不在熔断检测的范畴之内。

- **consecutiveGatewayErrors**： 非必配项，配置一个UInt32Value类型的值。表示从连接池连续返回的网关类型错误的个数，这些错误包括`502`,`503`,`504`等状态，注意 consecutiveGatewayErrors 与 continuous5xxerrors 是可以单独使用或者一起使用的，因为 consecutiveGatewayErrors 的错误也属于 continuous5xxerrors， 所以当 consecutiveGatewayErrors 发生次数加1的时候，continuous5xxerrors 的值也会加1.同时当 consecutiveGatewayErrors 的值大于等于 continuous5xxerrors 的值时，continuous5xxerrors 配置的值将优先起作用。当它的值被设置为0时，表示关闭此项配置。
- **consecutive5xxErrors**：非必配项，配置一个UInt32Value类型的值。表示连续从连接池返回的5xx请求码的个数。这些错误包括`502`,`503`,`504`等状态，同理，它可以与 consecutiveGatewayErrors 一起使用或单独使用。该配置的默认值是5，它不能被设置成0。
- **interval**：非必配项，配置一个 Duration 类型的值。表示驱逐服务的时间间隔。配置格式为：1h/1m/1s/1ms， 最小不能小于1ms。通俗讲就是讲就是计算各个异常点检测占比数时候取的时间单位。
- **baseEjectionTime**：非必配项，配置一个 Duration 类型的值。表示服务被驱逐的驱逐时间基量，一个服务被驱逐的时间 = 最小驱逐时间间隔 * 驱逐次数。这就是为什么一个服务被驱逐的次数越多，则驱逐的时间越长的原因。所以这里 baseEjectionTime 也可以表示 服务第一次被驱逐时驱逐的时长。
- **maxEjectionPercent**：非必填项，配置一个int32类型的值。表示最大驱逐服务的在连接池中的占比。它的默认值为10%， 表示不管流量压力多大，其中最多只有10%的服务被驱逐。
- **minHealthPercent**：非必填项，配置一个int32类型的值。表示服务连接池中最小健康的服务占比，当服务连接池中健康的服务占比大于该值的配置时，才会开启熔断检测功能。当该值设置成0%时则表示该项配置无效，即直接开启异常点检测功能。

**Tips**: consecutiveGatewayErrors 配置与 consecutive5xxErrors 配置是 Istio 1.5 新增的配置，而在1.5版本之前，只有一个参数叫做 consecutiveErrors。 也就是说，1.5版本对返回5xx的错误设置了一个子集。这样在特定的坏境中，对于熔断检测的控制粒度将更精细。

在下面的例子中，定义了一个异常点检测的规则，它表示计算 reviews.prod.svc.cluster.local 服务在5分钟之内它返回5xx异常的错误数，当连续7次返回时驱逐该服务，第一次驱逐的时间为15分钟。当第一次驱逐时间结束后，该服务将继续转发流量，当第二次检测又发现有连续7次返回5xx错误时，再次驱逐它，这次它的驱逐时间将变成 15 * 2 = 30分钟。

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

### Subset：

Subset用来配置服务子集，即配置同一个服务不同版本的实例。它是 DestinationRule 配置中比较出彩的地方，通过Subset配置，以及配置 VirtualService，我们可以很轻松的做到对不同版本服务的访问。它通过一个或者多个labels属性来确定跟那个实例的 Envoy 建立连接。

- **name**： 必配项，配置一个 string 类型的值，表示某个 subset 的名称，这个名称非常重要，在 VirtualService 中对 subset 的指定就是通过该配置进行关联的。从而架起来具体服务实例与外界请求访问的通道。
- **labels**：非必配项，配置一个 map<string,string> 类型的值，这也是非常重要的一个配置，它建立了该服务子集由哪些服务实例组成，而 labels 就是 pod 中定义的 labels 。
- **trafficPolicy**：非必配项，配置一个 TrafficPolicy 类型的值，这里的配置规则等同于 DestinationRule 规则下的全局配置规则。如果同时配置了全局的规则与它自己的规则，那么它自己的规则将覆盖全部规则。

在下面的例子中，定义了`ratings.prod.svc.cluster.local`服务全局的负载策略为最小连接策略。同时定义了一个名称叫做 testversion的服务子集，该服务子集去关联在所有的`ratings.prod.svc.cluster.local`服务实例中有`version:v3`这个标签的Pod，当流量命中该服务子集时，将不再使用最小连接的负载均衡策略，而是使用它自己定义的ROUND_ROBIN（轮询）负载均衡策略。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

### TLSSettings：

用于SSL/TLS相关的上游连接配置，它最终也是被翻译为 Envoy 的 TLS 路由规矩而应用。这些配置是 TCP 与 HTTP上游通用的配置。

下面的例子配置了一个客户端使用单向 TLS 连接与 "*.foo.com" 匹配的主机进行通讯，SIMPLE 表示单项认证模式。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: tls-foo
spec:
  host: "*.foo.com"
  trafficPolicy:
    tls:
      mode: SIMPLE
```



下面的例子表示使用 Istio 双向TLS进行会话，如果将mode配置成 ISTIO_MUTUAL 的模式进行会话，Istio 会自动帮助服务管理证书，是非常方便的，ISTIO_MUTUAL 表示双向TLS认证模式。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings-istio-mtls
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```



下面的例子配置了一个客户端配置双向TLS 连接到上游的`mydbserver.prod.svc.cluster.local`服务的规则。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: db-mtls
spec:
  host: mydbserver.prod.svc.cluster.local
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

TLSSettings有以下这些属性配置：

- **mode**：必配项，配置一个 TLSmode 类型的值，TLS会话模式，它确定了会话的执行方式。
- **clientCertificate**：非必配项，配置一个string类型的值，当 mode 为 MUTUAL 时，此项配置是必须的。表示存放客户端认证证书的路径，当 mode 配置为 ISTIO_MUTUAL时，此项配置必须为空。
- **privateKey**：非必配项，配置一个 stirng 类型的值，当 mode 为 MUTUAL 时，此项配置是必须的。表示客户端存放私有秘钥的路径，当mode配置为 ISTIO_MUTUAL时，此项配置必须为空。
- **caCertificates**： 可选项，配置一个string类型的值，表示提供证书的颁发机构认证证书的路径，如果为空，则代理不会验证认证服务器证书。
- **subjectAltNames**： 非必配项，配置一个string[]类型的值，表示验证证书中 subject 身份的备用名称列表，如果该配置项被指定，则代理将会使用这个列表中的一个验证服务器证书 subject 的别名，同时如果这个参数指定了则 ServiceEntry 中指定的 subjectAltnames 将会被覆盖。
- **sni**：非必配项，配置一个string类型的值，表示在TLS握手时显示给服务器的 SNI 值。

### TLSSettings.TLSmode：

TLSmode表示TLS在连接会话是遵循的模式。

- **DISABLE**： 表示不使用一个TLS连接和上游通讯。
- **SIMPLE**：表示使用单向TLS进行会话。
- **MUTUAL**：表示使用双向TLS的模式进行会话。
- **ISTIO_MUTUAL**： 使用ISTIO自己设计的双向TSL模式会话，这种模式下，Istio会自动管理发放各端的证书，同时当证书过期后自动更新证书。这种模式下，不需要再对 TLSSettings 的其他配置项做配置。

### TrafficPolicy：

TrafficPolicy，顾名思义，就是配置流量的策略。它有以下属性：

- **loadBalancer**：非必配项，配置一个 LoadBalancerSettings 类型的值，此配置用来配置负载均衡策略。
- **connectionPool**：非必配项，配置一个 ConnectionPoolSettings 类型的值，用来配置和上游的服务的 Envoy 的连接池。
- **outlierDetection**：非必配项，配置一个OutlierDetection类型的值，此配置是用来配置服务熔断相关的配置。
- **tls**：非必配项，配置一个 TLSSettings 类型的值，此配置主要用于TLS相关。
- **portLevelSettings**： 非必配项，配置一个 PortTrafficPolicy[] 类型的值，用来配置上游服务端口级别的流量策略，这里需要注意，端口即流量策略将会覆盖 destination-level  的配置，即同一请求在 destination-level  与 port-level 都配置时，destination-level 的策略将不会起作用。port-level 配置默认是被忽略的。

### TrafficPolicy.PortTrafficPolicy：

PortTrafficPolicy的配置策略有以下属性：

- **port**：指定上游服务哪个端口需要被应用为 PortTrafficPolicy 类型的策略。
- **loadBalancer**：非必配项，配置一个 LoadBalancerSettings 类型的值，此配置用来配置负载均衡策略。
- **connectionPool**：非必配项，配置一个 ConnectionPoolSettings 类型的值，用来配置和上游的服务的 Envoy 的连接池。
- **outlierDetection**：非必配项，配置一个 OutlierDetection 类型的值，此配置是用来配置服务熔断相关的配置。
- tls：非必配项，配置一个 TLSSettings 类型的值，此配置主要用于TLS相关。

### google.protobuf.UInt32Value：

UInt32Value是对uint32类型的封装。

- **value**：非必配项，配置一个uint32类型的值。