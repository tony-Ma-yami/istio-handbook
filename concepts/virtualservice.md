---
authors: ["tony-Ma-yami"]
reviewers: [""]

---

# VirtualService

顾名思义，VirtualService 是 Istio 中的一种虚拟 service，通过 VirtualService 来代理 Kubernetes 中的 service 资源，从而进行转发。VirualService 是 Istio 架构中最路由最复杂的配置，它定义了一组流量路由的规则，每个规则都定义了不同的匹配条件，当请求与规则匹配成功后，则将请求转发到指定上游去。它的一个简单的例子格式如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - name: "reviews-v2-routes"
    match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
  - name: "reviews-v1-route"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
```

## VirtualService中配置项介绍：

### VirtualService:

- **hosts**： 必配项，配置一个 String[] 类型的值，可以配置多个。指定了发送流量的目标主机， 可以使用FQDN（Fully Qualified Domain Name - 全限定域名）或者短域名， 也可以一个前缀匹配的域名格式或者一个具体的IP地址， 这里需要注意，当我们配置短域名时， Istio 在补全域名的时候将与 VirtualService 所应用的 Namespace 有关。 比如，我们配置  hosts: reviews， 当它在 default 命名空间下应用时，hosts 将会被补全为 reviews.default.svc.cluster.local， 当它在 test 命名空间下应用时， hosts 将会被补全为 reviews.test.svc.cluster.local 。
- **gateways**： 非必配项，配置一个 String[] 类型的值，用来指定该 VirtualService 的只接收从哪个 Gateway 资源过来的请求访问。这个配置包括两种访问类型的网关， 如果是外网访问， 那么配置的是自定义的外部域名， 如果是 Kubernetes 集群内部的访问， 使用关键字 mesh 即可。
- **http**： 官方解释为非必配项，但实际使用过程中一般都需要配置，配置一个  HTTPRoute[] 类型的值，其中配置的是一个有序的路由规则，HTTP路由将这里配置的路由规则中根据配置顺序挨个匹配，并当匹配到直接进行路由转发，不再继续进行后面的匹配。同时可以在这个配置项中可以配置对匹配到的路径redirect或者 rewrite 、设置请求的超时时间、设置请求的重试策略、镜像发布、故障注入、 cors 跨域策略设置等， 具体在HTTPRoute配置项中介绍。
- **tls**： 非必配项，配置一个 TLSRoute[] 类型的值，TLS 与HTTPS协议请求的路由规则配置项，应用于 “https-”、“TLS-” 的请求调用。
- **tcp**：非必配项，配置一个 TCPRoute[] 类型的值，专门用于 TCP 协议请求的路由规则匹配。这个配置项的优先级较低，当请求既不是一个 HTTP 类型请求，也不是一个 TLS 类型请求的时候，才会进入到该配置项中进行匹配。
- **exportTo**：非必配项，配置一个 String[] 类型的值， 表示该 VirtualService 应用在哪些 Namespace 下面。而应用到哪些 Namespace 下则表示别的 Namespace 下的请求将无法访问该 VirtualService，而该 VirtualService 也无法托管其他 Namespace下的 service 。如果不设置，则表示当前 VirtualService 应用到所有的 Namespace下。
  - “.”  表示该 VirtualService 只应用到当前的 Namespace 下。
  - “*”  表示应用到所有的namespace下。

### HTTPRoute：

- **name**： 非必配项，配置一个string类型的值，表示当前这条规则的一个名称。该名称会在请求的 access logs中展示。
- **match**：官方解释为非必配项，但实际使用过程中一般都需要配置，配置一个 HTTPMatchRequest[] 类型的值，其中包括了对于路由请求URI的一些匹配规则的设置。包括对请求的 uri、method、authority、headers、port、queryParams 以及是否对 uri 大小写敏感的配置。
- **route**：官方解释为非必配项，但实际使用过程中一般都需要配置，配置一个 HTTPRouteDestination[] 类型的值。表示当请求匹配成功之后在路由转发前可以对请求做的一些操作，包括请求权重的设置、对请求头、响应头中数据的增删改等。
- **redirect**：非必配项，配置一个 HTTPRedirect 类型的对象，当路由匹配成功后可以对url进行重定向。
- **rewrite**：非必配项，配置一个 HTTPRewrite 类型的对象，当路由匹配成功后可以对url进行重写。
- **timeout**：非必配项，HTTP请求的超时时间。
- **retries**：非必配项，配置一个 HTTPRetry 类型的对象，表示请求失败后的重试策略。
- **fault**：非必配项，配置一个 HTTPFaultInjection 类型的对象，表示对该请求进行配置故障注入配置。一般在开发阶段及性能测试阶段使用，通过该配置就可以模拟当你的服务所依赖的上游服务出现响应超时或者返回异常的时候你的异常处理机制是否完善。
- **mirror**：非必配项，配置一个 Destination 类型的对象，配合一个[0-100]的数值。表示将该请求的流量镜像到别的服务中去。通过此项配置通过镜像复制流量可以很好的调试开发测试坏境无法复现的产品环境问题。当然在实际开发过程中，这种选择仍然是备用选择，最好的方式还是通过收集日志去分析线程的异常问题。
- **mirrorPercent**：非必配项，配置一个 UInt32Value 类型的值。与 mirror 配置成对出现，表示需要镜像原始流量的占比。比如100表示将所有流量全部镜像到你在 mirror 中配置的服务中去。
- **corsPolicy**：非必配项，CORS跨域（Cross-Origin Resource Sharing）访问的策略配置，此配置用来解决在多个 Domain 访问该 VirtualService 的访问策略。
- **headers**：非必配项，配置一个 Headers 类型的对象，操作 Header 中信息的增删改。包括 Request Header 和 Response Header 。

### google.protobuf.UInt32Value：

对int类型数值的一个封装对象，只有一个属性 value

- **value**: 配置一个int类型的数值。

### HTTPMatchRequest：

首先，我们先来使用一个请求链接来介绍下它在 HTTPMatchRequest 中对应的的属性成分。

```
curl -X GET http://productpage.com:80/domain-a/findstr?wish=apple -H 'end-user: jason'
```

上面这个例子中：

- scheme 属性值为 “http://”
- authority 属性值为 “productpage.com:80”
- method 属性为 “GET”
- headers 属性为 “end-user: jason”
- port 属性为 “80”
- queryParams 属性为 “wish=apple”
- path 属性值为 “/domain-a/findstr”

接下来， 我们介绍下 HTTPMatchRequest 中各个属性配置项的意义：

- **name**：非必配项，配置一个string类型的值， 标记一个 match 的配置，会跟他父级的 route 的 name 一起记录在请求调用的 access logs当中。

- **uri**：非必配项，配置对请求的 uri 的匹配，支持exact，prefix，regex三种方式的匹配。

- **scheme**：非必配项，配置对 uri 中的 scheme 部分进行匹配，支持exact，prefix，regex三种方式的匹配。

- **method**：非必配项，配置对uri请求方式的匹配，支持exact，prefix，regex三种方式的匹配。

- **headers**：非必配项，配置对请求头的匹配， 也支持 exact、prefix、regex 三种类型的配置。比如以下配置表示匹配请求头中存在 end-user:jason 样的请求。

  ```yaml
  ...
    http:
    - match:
      - headers:
          end-user:
            exact: jason
  ...
  ```

- **port**：配置对 uri 请求中 port 的匹配。

- **sourceLabels**：配置请求源的工作负载（pod） 的标签属性。比如以下配置表示匹配请求源所在 pod 中同时拥有 app: reviews 与 version:v2 两个 label 的请求。

  ```yaml
  ...
    http:
    - match:
      - sourceLabels:
          app: reviews
          version: v2
  ...
  ```

- **queryParams**: 对请求参数进行匹配，也支持 exact、prefix、regex 三种类型的配置。比如以下配置表示匹配请求参数中有 wish=apple 的请求。

  ```yaml
  ...
    http:
    - match:
      - uri:
          prefix: /productpage
        queryParams:
          wish:
            exact: apple
  ...
  ```

- **ignoreUriCase**：配置是否对request请求大小写敏感。比如以下配置表示请求的 uri 忽略大小写。

  ```yaml
  ...
    http:
    - match:
      - uri:
          prefix: "/productpage"
        ignoreUriCase: true
  ...
  ```

### HTTPRedirect：

- **uri**：非必配项，需要重定向的 uri 地址。
- **authority**：非必配项，需要重定向的 uri 中 authority 属性值。
- **redirectCode**：非必配项，定义重定向请求返回的状态码，默认为 MOVED_PERMANENTLY (301)

比如以下这个配置表示：将请求 http://ratings.prod.svc.cluster.local/v1/getProductRatings 的请求重定向到 http://newratings.default.svc.cluster.local/v1/bookRatings，并返回状态码301.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - uri:
        exact: /v1/getProductRatings
    redirect:
      uri: /v1/bookRatings
      authority: newratings.default.svc.cluster.local
```

### HTTPRewrite：

- **uri**：非必配项，需要重写的 uri 地址。
- **authority**：非必配项，需要重写的 uri 中 authority 属性值。

比如以下这个配置表示：将 http://ratings.prod.svc.cluster.local/ratings 的请求重写到 http://ratings.prod.svc.cluster.local/v1/bookRatings 这个地址上。 HTTPRewrite 在我们服务间调用时对于各个 Domain 前缀的重写将会非常有用。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: /ratings
    rewrite:
      uri: /v1/bookRatings
    route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
```

### HTTPRetry：

- **attempts**：必配项，代表重试次数，表示当请求失败后经过多少次重试之后放弃。
- **perTryTimeout**：非必配项，表示重试请求的超时时间，格式为：1h/1m/1s/1ms， 最小不能小于1ms。
- **retryOn**：非必配项，表示在什么情况下进行重试，如果有多个重试条件，使用 “，” 隔开。更多详细配置参考：https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-on 以及https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-grpc-on。

以下配置表示：对 ratings.prod.svc.cluster.local 服务发送的请求，当失败时，最多进行3次重试，重试间隔时间为 2s，并且只有当出现 gateway-error，connect-failure，refused-stream 这三种异常时才进行重试。

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

### CorsPolicy：

**Cors**：（Cross-Origin Resource Sharing）跨域资源共享， 通俗来讲，就是不同的域名要访问同一个请求。这个时候 Server 端对于不同来源的请求，设置一定的访问策略进行控制。Cors 的策略配置并不是 Istio 独有的特性，在很多网络服务或者组件中都是支持的， 比如 Nginx、Springboot、Gloo 等。

- **allowOrigin**：非必配项，配置一个 string[] 类型的值，表示允许跨域的请求来源，使用通配符 “*” 表示允许所有的请求源。
- **allowMethods**： 非必配项，配置一个 string[] 类型的值，表示允许跨域请求的请求方式。若不配置表示允许所有类型的请求方式。
- **allowHeaders**：非必配项，配置一个 string[] 类型的值，表示允许跨域请求的请求头名称。
- **exposeHeaders**：非必配项，配置一个 string[] 类型的值，它是一个允许浏览器直接访问的请求头白名单。
- **maxAge**：非必配项，配置跨域请求的预检结果保存时间，格式为：1h/1m/1s/1ms。此配置在请求时将会被翻译为 Access-Control-Max-Age 参数。这里解释一下什么叫做预检请求：当进行跨域访问的时候，浏览器不会直接发送请求，而是先发送一个预检请求，预检请求的作用在于获取目标服务是否允许本站的跨域请求，如果允许才会发起真正的请求。所以 maxAge 属性就是配置预检请求结果的有效时间。
- **allowCredentials**： 非必配项，跨域请求中目标源提供的是否允许请求源携带验证信息，此配置在请求时将会被翻译为 Access-Control-Allow-Credentials 参数。 它的作用不同于 Access-Control-Max-Age 。

比如以下这个配置表示：接收来自 example.com 域名的请求，但是值允许 POST 与 GET 方式提交的方法，并且只公开 X-Foo-Bar 标头，跨域请求的预检时间最大为24小时，同时 Access-Control-Allow-Credentials 参数为false。

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
    corsPolicy:
      allowOrigin:
      - example.com
      allowMethods:
      - POST
      - GET
      allowCredentials: false
      allowHeaders:
      - X-Foo-Bar
      maxAge: "24h"
```

### Destination：

Destination 用来配置 VirtualService 的上游主机，同时可以配合 DestinationRule 指定访问上游某个版本的工作负载（pod）。

- **host**：必配项，是一个 string 类型的值。表示上游资源的地址，host 可配置的资源为所有在集群中注册的所有的 service 域名或者资源， 可以是一个 Kubernetes 的 service，也可以是一个 Consul 的 services， 或者是一个 ServiceEntry 类型的资源。同样，这里配置的域名可以是一个 FQDN，也可以是一个短域名。注意，如果您是 Kubernetes 用户，那么这里的资源地址补全同样依赖于该 VirtualService 所在的命名空间，比如在 prod 命名空间下配置了 destination.host=reviews，那么 host 最终将被解释并补全为 reviews.prod.svc.cluster.local。
- **subset**：非必配项，是一个 string 类型的值。表示访问上游工作负载组中指定版本的工作负载，如果需要使用 subset 配置，必须先在 DestinationRule 中定义一个 subset 组，而这里配置的即为 subset 组中某一个 subset 的 name 值。
- **port**：非必配项，是一个 PortSelector 类型的对象。指定上游工作负载的端口号，如果上游工作负载只公开了一个端口，那么这个配置将可以不用配置。

在 VirtualService中，Destination 的配置与使用也是非常重要的一环，Istio 的很多功能都是基于 Destination 的配置实现的，接下来介绍几种不同类型的配置方式以及他们的路由效果。

1. 最简单的一种配置，一个 VirtualService ，不使用 DestinationRule 。下面的配置中，表示访问 http://reviews/wpcatalog 或者 http://reviews/consumercatalog 时，都会被转发到review:9080的服务实例中去，同时重写了该请求的请求路径，统一被转发到 http://reviews:9080/newcatalog 地址中去。当该VirtualService被应用时， destination.host 的配置 reviews 会被解释并补全为 reviews.foo.svc.cluster.local， 这个地址实际上是一个 Envoy 的 Cluster 地址。用于 Envoy 做路由选择时使用。对于在 match 配置中匹配上的请求， 会被转发到 reviews.foo.svc.cluster.localCluster 对应的服务，并且端口为 9080 的服务上去。这里需要注意的是，该配置中并没有配置 subset，在这种配置下，如果上游的 reviews 工作负载组中有着多个不同版本的 reviews 服务实例，并全部也都将 9080 端口公开， 这个时候会 Istio 将会以轮询的方式按个访问每一个服务实例。当然，如果 review 的服务只对外公开了一个端口 9080，那么这里的 port 配置即可省略。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
  namespace: foo
spec:
  hosts:
  - reviews # interpreted as reviews.foo.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        port: 9080
```

2. 使用 DestinationRule 定义 reviews 多个不同的版本，在 VirtualService 中通过subset指定需要访问的版本。接下来我们配置 reviews 有两个不同版本的服务实例 v1 和 v2 ，下面的配置表示访问 http://reviews/wpcatalog 或者 http://reviews/consumercatalog 时，重写到 http://reviews/newcatalog 请求中去，并且调用的是 v2 版本服务实例。其他请求则被路由到 v1 版本的服务实例中去。同样，如果这里不配置 subset，那么将会轮询访问 v1 和 v2 版本的服务实例。

先定义 VirtualService，如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
  namespace: foo
spec:
  hosts:
  - reviews # interpreted as reviews.foo.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: v1
```

然后关联 DestinationRule，如下配置，DestinationRule 也是 Istio 规范中的另外一种资源， 用来区分不同版本的上游服务。通常使用 pod 中定义的 label 来区分，具体介绍请参考 DestinationRule 配置章节的资料。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: foo
spec:
  host: reviews # interpreted as reviews.foo.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

3. 在一个 Namesapce 下去访问另外一个 Namespace 下的资源。如下配置， VirtualService 是定义在 istio-system 这个 Namespace 下，如果我们要访问 prod 这个 Namespace 下的 productpage 服务，就必须将 host 配置成 FQDN 格式的域名，即 productpage.prod.svc.cluster.local，这里就不能配置成短域名，加入这里配置成 productpage 。那么它将会被解释补全为 productpage.isito-system.svc.cluster.local 。同时我们注意到，对所有访问 productpage 服务的请求设置了超时时间为 5s 。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-productpage-rule
  namespace: istio-system
spec:
  hosts:
  - productpage.prod.svc.cluster.local # ignores rule namespace
  http:
  - timeout: 5s
    route:
    - destination:
        host: productpage.prod.svc.cluster.local
```

4. Destination 的 host 配置还可以是一个 ServiceEntry 类型的资源。下面的配置表示将 wikipedia.org:80 服务托管到 Istio 坏境中，并作为名称为 my-wiki-rule 的 VirtualService 的上游服务存在。这样就可以在网格内部通过 http://wikipedia.org 的方式访问到 wikipedia.org：80 的服务上了。同时还设置了请求超时时间为 5s 。

ServiceEntry 资源是将一个网格外部的服务注册到 Istio 坏境中去，从而达到使用 Istio 统一托管控制的技术，请大家查看 ServiceEntry 相关章节的介绍获取更为详细的资料。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-wikipedia
spec:
  hosts:
  - wikipedia.org
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: example-http
    protocol: HTTP
  resolution: DNS


```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-wiki-rule
spec:
  hosts:
  - wikipedia.org
  http:
  - timeout: 5s
    route:
    - destination:
        host: wikipedia.org
```

### HTTPFaultInjection：

在一个微服务架构的服务群中，一定存在部分服务有着较高的健壮性要求，比如电商中的订单系统，当订单相关的服务出现异常时，一般需要第一时间通知维护人员，这就要求对于服务的异常处理以及监听方面要求很高。从运维方面，我们可以对服务订单服务的相关主机或者云虚拟主机的 CPU，内存等重要指标进行监听，若发现异常，以短信、消息或者邮件的方式告知。另一方面，若程序内部访问异常，那么服务也要第一时间通知到维护人员。而这些异常在开发测试阶段人为干涉或者模拟是一个非常繁杂的工作，而 Istio 提供的 HTTPFaultInjection 就是为了解决在开发测试节点调试难问题而设计的。它可以用来对服务进行故障注入，大家可以通过配置不同的 HTTPFaultInjection 策略来检测自己服务的健壮性以及对于异常处理的能力而不需要花费过多的精力。

HTTPFaultInjection 是 VirtualService 中的一种规则，它包括了两种类型的故障注入：

- **abort**：非必配项，配置一个 Abort 类型的对象。用来注入请求异常类故障。简单的说，就是认为模拟上游服务返回异常时，我们自己的服务是否具备处理能力。
- **delay**：非必配项，配置一个 Delay 类型的对象。用来注入延时类故障。通俗一点讲，就是人为模拟上游服务的响应时间，看在这个时间内无响应的情况下，我们自己的服务是否具备容错容灾的能力。

### HTTPFaultInjection.Abort：

- **httpStatus**：必配项，是一个整型的值。表示注入 HTTP 请求的故障状态码。
- **percentage**：非必配项，是一个 Percent 类型的值。表示对多少请求进行故障注入。如果不指定改配置，那么所有请求都将会被注入故障。
- **percent**：已经过期的一个配置，与 percentage 配置功能一样，已经被 percentage 代替。

如下的配置表示对 v1 版本的 ratings.prod.svc.cluster.local 服务访问的时候进行故障注入，0.1表示有千分之一的请求被注入故障， 400 表示故障为该请求的 HTTP 响应码为 400 。

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

- **fixedDelay**：比配置项，表示请求响应的模拟处理时间。格式为：1h/1m/1s/1ms， 不能小于 1ms。
- **percentage**：非必配项，是一个 Percent 类型的值。表示对多少请求进行故障注入。如果不指定改配置，那么所有请求都将会被注入故障。
- **percent**：已经过期的一个配置，与 percentage 配置功能一样，已经被 percentage 代替。

如下的配置表示对 v1 版本的 reviews.prod.svc.cluster.local 服务访问的时候进行故障注入，0.1 表示有千分之一的请求被注入故障，5s 表示故障为该请求的HTTP配置 5s 的延迟。

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

### HTTPRouteDestination：

VirtualService 的每一个 HTTPMatchRequest 配置项都需要定义一个加入匹配成功后应该路由到的目的地。HTTPRouteDestination 就是用来定义这里所说的目的地，它不仅可以指定请求应该被路由到哪里，还支持路由到某个服务的权重，以及在路由之前对请求做一些处理，包括对请求标头数据的增删改操作。

- **destination**： 必配项，是一个 Destination 类型的对象，它指定了请求应该被转发的地址。
- **weight**：非必配项，是一个整型的数值，取值范围为（0~100），它与 destination 成对出现，指定了被转发到 destination 中的权重。所有的 destination 配置的 weight 权重之和必须等于 100。
- **headers**：非必填项，是一个Headers类型对象，通过这个配置可以对转发的请求的请求标头做一些增删改。
- **removeResponseHeaders**：已过期，用来删除 response 的某些标头信息，现在已经被 headers 配置代替。
- **appendResponseHeaders**：已过期，用来补充增加 response 的某些标头细腻，现在已经被 headers 配置代替。
- **removeRequestHeaders**：已过期，用来删除 request 的某些标头信息，现在已经被 headers 配置代替。
- **appendRequestHeaders**：已过期，用来补充增加 request 的某些标头信息，现在已经被 headers 配置代替。

HTTPRouteDestination 是非常常用的一个配置，他也支持多种不同方式的配置，下面举几个l例子说明。

1. 一个 HTTPRouteDestination 的简单配置如下，它表示 reviews.prod.svc.cluster.local 的服务将全部转发到 reviews.prod.svc.cluster.local：9080 的 Kubernetes 的 service 资源上。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: reviews.prod.svc.cluster.local
        port: 9080
```

2. 以下的配置中配置了两个 HTTPRouteDestination，分别是 reviews.prod.svc.cluster.local 服务的 v1 和 v2 版本，其中 v2 版本的权重为 25，v1 版本的权重为 75，也就是说，当访问 reviews.prod.svc.cluster.local 服务时，有 25% 的请求将会命中 v2 版本的 reviews 服务，有 75% 的请求将会命中 v1 版本的 reviews 服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
      weight: 25
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
      weight: 75

```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews.prod.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

3. HTTPRouteDestination 不仅仅可以配置相同服务不同版本的访问权重， 还可以配置为不同服务不同权重的访问策略。比如以下配置表示访问 reviews.com 服务时，将会有 25% 的请求被转发到 dev.reviews.com 服务上去，有 75% 的服务转发到 reviews.com 服务中去。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route-two-domains
spec:
  hosts:
  - reviews.com
  http:
  - route:
    - destination:
        host: dev.reviews.com
      weight: 25
    - destination:
        host: reviews.com
      weight: 75
```

### Headers：

Headers 配置较简单，就是对于请求标头的一些操作，它包括对 request 标头的操作和对 response 标头的操作。

- **request**：非必填项，是一个 HeaderOperations 类型的对象，表示对 request 标头的操作，表示在请求转发前对请求标头的一些维护处理。
- **response**：非必填项，是一个 HeaderOperations 类型的对象，表示对 response 标头的操作，表示在请求转发前对请求标头的一些维护处理。

比如在以下的配置中，首先对请求访问 reviews.prod.svc.cluster.local 服务时，首先会将 request 标头的 test 属性值设置为 true。然后根据权重路由，当路由到 reviews.prod.svc.cluster.local 服务的 v2 版本时，直接访问，当路由到 reviews.prod.svc.cluster.local 的 v1 版本时，在转发前不对请求标头再做处理，而是当返回 response 时，把 response 的标头中的 foo 属性删除掉。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - headers:
      request:
        set:
          test: true
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
      weight: 25
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
      headers:
        response:
          remove:
          - foo
      weight: 75
```

### Headers.HeaderOperations：

HeaderOperations 主要用来定义对请求标头或者响应标头的操作类型。它包括以下三种操作类型

- **add**：非必配项，是一个 map<stirng，string> 类型的对象，表示在请求头或者响应头中加入某些标头。
- **remove**：非必填项，是一个 string[] 类型的对象，表示删除某些标头，这里没有设计成 map<string，string> 的格式是因为删除时，只需要知道标头的 key 即可。
- **set**：非必配项，是一个 map<string，string> 类型的对象，表示对某个标头属性进行重新赋值。这个区别于 add 配置，set 的配置当 ma p类型数据的 key 存在时重新赋值，当 key 不存在时则不进行 add 操作，而是忽略本次 set。

### L4MatchAttributes：

表示四层网络连接时的一些匹配规则配置。具体有：

- **destinationSubnets**：非必配项，配置一个 string[] 类型的值。配置一个 IPv4 或者 IPv6 的地址，格式为：a.b.c.d/xx 或者 a.b.c.d
- **port**： 非必配项，配置一个整型类型的值。指定该提供服务主机的服务端口，如果服务只公开了一个端口供外部访问，那么可以不用配置该项。
- **sourceLabels**：非必配项，配置一个 map<string，string> 类型的值。用来匹配只接收带有某一组 label 的请求源发来的请求。这里的 label 即为请求源所在 pod 中的 label 。
- **gateways**：非必填项，配置一个string[]类型的值。即配置 gateway 的名称，以用来匹配从某个 gateway 转发过来的请求。如果同时要支持集群内部服务访问该 VirtualService，那么需要配置一个关键字 mesh 。

### Percent：

Percent 是用来定义百分比的一个对象，它的属性只有一个：

- **value**：非必填项，配置一个 double 类型的值。它的取值范围为 [0.0~100.0]，用来标记某个路由或者负载转发的权重。

### PortSelector：

PortSelector 用来定义一个端口， 它的属性只有一个：

- **number**：非必填项，配置一个int类型的数值，用来标记某个路由地址的服务端口。

### RouteDestination：

RouteDestination 是L4层路由规则必配中的一个配置项，用来定义一组路由转发的目标服务。它有两个属性

- **destination**：必配项，是一个Destination类型的值。用来指定请求或者连接转发的地址。
- **weight**：非必配项，是一个int类型的值。用来指定目标服务的路由权重。

### StringMatch：

StringMatch 是用来描述 string 值的一个匹配方式，它包括以下三种类型：

- **exact**：必配项，配置一个 string 类型的值。表示精确匹配。
- **prefix**：必配项，配置一个 string 类型的值。表示前缀匹配。
- **regex**：必配项，配置一个 string 类型的值。表示使用正则来进行匹配。

下面举几个例子来说明下他们的使用规则：

- exact：表示对uri字符串的精确匹配。比如：

```yaml
...
  http:
  - match:
    - uri:
        exact: /productpage
...
```

- prefix：表示对uri字符串进行前缀匹配。比如：

```yaml
...
  http:
  - match:
    - uri:
        prefix: /static # 表示以/static开头的请求
...
```

- regex：表示使用正则表达式对uri进行匹配。比如：

```yaml
...
  http:
  - match:
    - uri:
        regex: "^(.*?;)?$"
...
```

### TCPRoute：

TCPRoute用来定义TCP的流量路由，它的子属性有两个。

- **match**：非必配项，配置一个 L4MatchAttributes[] 类型的值。TCP 路由请求的规则配置都是在这里配置的。如果你配合了多个 L4MatchAttributes 类型的匹配规则，那么它在进行匹配时会根据你的配置顺序，命中后直接转发路由而不再向下继续匹配。
- **route**：非必配项，配置一个 RouteDestination[] 类型的值。用来指定一组路由规则匹配成功需要路由转发的地址。**这里可以配置多个转发地址**，但是所有的转发地址权重加起来必须等于100。

### TLSMatchAttributes：

TLSMatchAttributes用来匹配 TLS 路由的属性匹配，它的作用类似于 HTTP 协议中的 HTTPMatchRequest ，也是用来对请求源的一些请求属性进行匹配，如果匹配成功后，进行下一步路由的转发。它有以下的子属性：

- **sniHosts**：必配项，配置一个string[]类型的值。SNI（Server Name Indication），指SNI主机的名称，可以配置一个通配符来进行匹配。比如使用 “*.com” 可以匹配到 “foo.example.com” ，也可以匹配到 “example.com”。若不使用通配符，那么一个SNI的配置值必须是一个虚拟服务主机的一个子集。
- **destinationSubnets**：非必配项，配置一个string[]类型的值。配置一个IPv4或者IPv6的地址，格式为： a.b.c.d/xx 或者a.b.c.d
- **port**：非必配项，配置一个int类型的值。指定host提供服务的服务端口。同样，如果只公开了一个端口提供服务，这里将不需要配置。
- **sourceLabels**：非必配项，配置一个 map<string，string> 类型的值。用来匹配只接收带有某一组 label 的请求源发来的请求。这里的 label 即为请求源所在 pod 中的 label 。
- **gateways**：非必配项，配置一个 string[] 类型的值.用来指定该 VirtualService 的只接收从哪个 gateway 资源过来的请求访问.这个配置包括两种访问类型的网关， 如果是外网访问， 那么配置的是自定义的外部域名， 如果是 Kubernetes 集群内部的访问， 使用关键字 mesh 即可.

下面通过一组表格来表示是否sniHosts的通配符的匹配规则：

| sniHosts          | VirtualService Host | 匹配结果 |
| ----------------- | ------------------- | -------- |
| *                 | productpage.com     | true     |
| productpage.com   | productpage.com     | true     |
| *.productpage.com | aws.productpage.com | true     |
| *.productpage.com | productpage.com     | false    |
| productpage/*     | productpage.com     | true     |

### TLSRoute：

TLSRoute用来定义 TLS/HTTPS 类型的流量路由规则，它与 HTTPRoute，TCPRoute 的功能类似，但是它的配置更简洁。它只包括 TLS/HTTPS 路由的匹配规则已经路由的地址。

- **match**：必配项， 配置一个 TLSMatchAttributes[] 类型的值。TLS/HTTPS 路由请求的规则配置都是在这里配置的。如果你配合了多个 TLSMatchAttributes 类型的匹配规则，那么它在进行匹配时会根据你的配置顺序，命中后直接转发路由而不再向下继续匹配。
- **route**：非必配项，配置一个 RouteDestination[] 类型的值。用来指定一组路由规则匹配成功需要路由转发的地址。这里可以配置多个转发地址，但是所有的转发地址权重加起来必须等于100。

VirtualService是Istio框架中最核心的配置，同时涉及到的专有名词以及配置项也非常的繁多，大家在阅读学习的过程中要善于总结，对类似的配置项进行归类，对容易出现混淆的配置仔细区分，了解了VirtualService的配置规则，对于整个Istio框架的使用将会有很大的帮助。

