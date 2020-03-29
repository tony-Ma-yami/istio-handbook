---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# Gateway

Gateway 是 Istio 中对流量控制的第一层服务资源，它定义了所有的流量入站或者出站的HTTP/TCP连接负载均衡器操作。它描述了一组对外公开的端口、协议、负载均衡、以及 SNI 配置。Istio 默认的 Gateway 使用 IngressController 负载均衡器来代理流量。

在下面的例子中，可以看到该 Gateway 被引用在`some-config-namespace`这个Namespace下，并使用label`my-gateway-controller`来关联负载均衡器代理的Pod。它对外公开了`80`、`443`、`9443`、`9080`、`2379`端口。

`80`端口附属配置的host为`uk.bookinfo.com`，`eu.bookinfo.com`，同时在`tls`中配置如果使用HTTP1.1协议访问将会被返回301，要求使用 HTTPS 访问，通过这种配置变相的禁止了对`uk.bookinfo.com`，`eu.bookinfo.com`域名的HTTP1.1协议的访问入口。

`443`端口为 TLS/HTTPS 访问的端口，表示接受`uk.bookinfo.com`，`eu.bookinfo.com`域名的HTTPS协议的访问，`protocol`属性指定了他的协议类型。在`tls`的配置中指定了他的会话模式为单向TLS，也指定了服务端证书和私钥的存放地址。

`9443`端口也是提供 TLS/HTTPS 访问，与 `443`不同的是他的认证不是指定存放证书的地址，而是通过credentialName名称从Kubernetes的证书管理中心拉取。

`9080`端口为一个提供简单 HTTP1.1 协议请求的端口。这里我们注意到它的hosts中配置了 `ns1/*` 与 `ns2/foo.bar.com` 这个配置项表示只允许`ns1`这个 Namespace 下的 VirtualService 绑定它以及 `ns2` 这个命名空间下配置了 `host` 为`foo.bar.com`的 VirtualService 绑定它。

`2379`端口提供了一个`MONGO`协议的请求端口。允许所有 `host` 绑定它。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
  namespace: some-config-namespace
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      httpsRedirect: true # sends 301 redirect for http requests
  - port:
      number: 443
      name: https-443
      protocol: HTTPS
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      mode: SIMPLE # enables HTTPS on this port
      serverCertificate: /etc/certs/servercert.pem
      privateKey: /etc/certs/privatekey.pem
  - port:
      number: 9443
      name: https-9443
      protocol: HTTPS
    hosts:
    - "bookinfo-namespace/*.bookinfo.com"
    tls:
      mode: SIMPLE # enables HTTPS on this port
      credentialName: bookinfo-secret # fetches certs from Kubernetes secret
  - port:
      number: 9080
      name: http-wildcard
      protocol: HTTP
    hosts:
    - "ns1/*"
    - "ns2/foo.bar.com"
  - port:
      number: 2379 # to expose internal service via external port 2379
      name: mongo
      protocol: MONGO
    hosts:
    - "*"

```



通常，我们会在 VirtualService 中配置 Gateway 的名称从而将流量的转发路由关联起来，下面的配置中，我们对 `bookinfo-rule` 这个 VirtualService 关联了一个在`some-config-namespace`命名空间的叫做 `my-gateway` 的 Gateway。我们配置 Gateway 时，可以使用`some-config-namespace/my-gateway`这样的格式配置，也可以使用`my-gateway.some-config-namespace.svc.cluster.local`这样的格式配置。这样，从这个 Gateway 转发过来的流量并且访问的域名是`reviews.prod.svc.cluster.local`、`uk.bookinfo.com`、`eu.bookinfo.com`其中的一个时，就会被下面的路由规则进行匹配。这里大家可以看到同时配置了一个叫做`mesh`名称的 Gateway，配置了它，实际上是指网格内的服务访问上面三个域名时也使用该 VirtualService 中配置的规则。同时如果配置的 VirtualService 与 Gateway 是在一个Namespace下，那么配置 `gateways`的时候可以省略命名空间。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-rule
  namespace: bookinfo-namespace
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  - uk.bookinfo.com
  - eu.bookinfo.com
  gateways:
  - some-config-namespace/my-gateway
  - mesh # applies to all the sidecars in the mesh
  http:
  - match:
    - headers:
        cookie:
          exact: "user=dev-123"
    route:
    - destination:
        port:
          number: 7777
        host: reviews.qa.svc.cluster.local
  - match:
    - uri:
        prefix: /reviews/
    route:
    - destination:
        port:
          number: 9080 # can be omitted if it's the only port for reviews
        host: reviews.prod.svc.cluster.local
      weight: 80
    - destination:
        host: reviews.qa.svc.cluster.local
      weight: 20

```

Gateway的一些配置项介绍：

### Gateway：

- **servers**： 必配项，配置一个 Server[] 类型的值。用来配置一个服务端列表。
- **selector**：必配项，配置一个 map<string, string> 类型的值。用来配置一组`labels`，从而关联使用哪个负载均衡代理实现。

### Port：

配置服务的端口信息。

- **number**：必配项，配置一个 uint32 类型的值。表示端口值。
- **protocol**：必配项，配置一个 string 类型的值。表示该端口支持的协议。可以配置有`HTTP|HTTPS|GRPC|HTTP2|MONGO|TCP|TLS`
- **name**：非必配项，配置一个string类型的值。表示该端口的一个名称。

### Server：

配置服务的一些属性。

**port**：必配项，配置一个Port类型的值。用来监听入站请求的端口。

**hosts**：必配项，配置一个string[]类型的值。表示一个或多个 gateway 公开的主机域名。如果应用于 HTTPS/TLS 的服务，这里还可以`SNI`名称。如果不是在当前的 Namespace下， 使用 `namespace/domain` 这样的格式匹配。也可以使用一个通配符来进行匹配，比如`prod/*.example.com`表示prod这个Namespace下以`.example.com`结尾的域名。同理， 这里也可是用`.`表示当前的命名空间下任意的域名、`*`表示任意的命名空间下，任意的域名。

> **提示**：hosts在以上的配置下，VirtualService是否可以顺利关联到gateway还与VirtualService的exportTo属性有关。必须保证exportTo属性公开的Namespace范围包含了需要关联的Gateway所在的Namespace。

**tls**：非必配项，配置一个TLSOptions类型的值。配置该服务的一些安全策略。

**defaultEndpoint**：非必配项，配置一个string类型的值。用来定义默认端点地址。

### Server.TLSOptions：

配置当前 Server 的一些握手协议，连接策略等。

**httpsRedirect**：非必配项，配置一个bool类型的值。如果设置为`true`,表示返回`301`，要求客户端使用 HTTPS 协议访问。

**mode**：非必配项，配置一个 TLSmode 类型的值。表示连接使用的TLS模式。

**serverCertificate**：非必配项，配置一个 string 类型的值，当`mode`是`SIMPLE`（单向TLS）或者`MUTUAL`（双向TLS）时，这项是比配置的，表示服务的证书地址。

**privateKey**：非必配项，配置一个 string 类型的值，当`mode`是`SIMPLE`（单向TLS）或者`MUTUAL`（双向TLS）时，这项是比配置的，表示服务的私钥地址。

**caCertificates**：非必配项，配置一个 string 类型的值，当`mode`是`MUTUAL`（双向TLS）时，这项是比配置的，表示为客户端签发证书的签发机构证书。

**credentialName**：非必配项，配置一个 string 类型的值，表示一个唯一指定的在 Kubernetes 中维护的`secret`资源的名称，用来从 Kubernetes 中拉取 `serverCertificate` 与 `privateKey` 配置。同时使用带有后缀`-cacert`的 credentialName 来表示 caCertificates 书。

**subjectAltNames**：非必配项，配置一个 string[] 类型的值。用来验证客户端提供的证书中主机身份的别名。

**verifyCertificateSpki**：非必配项，配置一个 string[] 类型的值。签发客户端证书的`SKPIs`的 Base64 码，与 verifyCertificateHash 一起配置时，一个 Hash 值与这两个值都匹配的证书才被接受。

**verifyCertificateHash**：非必配项，配置一个 string[] 类型的值， 验证证书的 Base64 码，与 erifyCertificateSpki 一起配置时，一个Hash值与这两个值都匹配的证书才被接受。

**minProtocolVersion**：非必配项，配置一个 TLSProtocol 类型的值，表示支持 TLS 的最小 TLS 协议版本，可选项有`TLS_AUTO|TLSV1_0|TLSV1_1|TLSV1_2|TLSV1_3`。

**maxProtocolVersion**：非必配项，配置一个 TLSProtocol 类型的值，表示支持 TLS 的最大 TLS 协议版本，可选项有`TLS_AUTO|TLSV1_0|TLSV1_1|TLSV1_2|TLSV1_3`。

**cipherSuites**：非必配项， 配置一个 string[] 类型的值，表示密码生成的组件，默认使用 Envoy。

### Server.TLSOptions.TLSmode：

TLS 协议模式分类

- PASSTHROUGH：使用客户端提供的SNI名称确定路由信息。

- SIMPLE：单向TLS，即只有服务端认证。

- MUTUAL：双向TLS， 服务端与客户端同时需要对对方进行认证。

- ISTIO_MUTUAL：自动管理服务器端证书，使用 mTLS 模式认证，相比较于`MUTUAL`模式，这个模式中的证书信息统一由 Istio 托管，不再需要配置证书与秘钥地址。

  