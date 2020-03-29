- # ServiceEntry

  ServiceEntry允许将Istio网格外的服务注册到网格内部Istio的注册表中去，这样在Istio内部就可以对外部服务进行管理，把它当做 Istio 内部的服务操作。包括服务发现，路由控制等，在ServiceEntry中可以配置 `hosts`，`vips`，`ports`，`protocols`，`endpoints`等。

  下面的例子定义来在网格内部使用HTTPS协议访问外部的几个服务的配置。通过以下配置，网格内部的服务就可以把`api.dropboxapi.com`，`www.googleapis.com`,`www.googleapis.com`这几个外部的服务当做网格内部服务去访问了。`MESH_EXTERNAL`表示是网格外服务。

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: ServiceEntry
  metadata:
    name: external-svc-https
  spec:
    hosts:
    - api.dropboxapi.com
    - www.googleapis.com
    - api.facebook.com
    location: MESH_EXTERNAL
    ports:
    - number: 443
      name: https
      protocol: TLS
    resolution: DNS
  
  ```

  

  下面这个例子中，将 mongodb 作为一个网格内部的服务写入Istio的注册表中，这样在网格内部，它就拥有网格内部其他服务拥有的所有特性。比如，在 DestinationRule 中配置它的上游为 `mymongodb.somedomain`，同时使用mTLS的方式对 mogodb 进行访问。这种方式托管的服务，适用于是那些手动在云主机或宿主机上安装的本地服务。

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: ServiceEntry
  metadata:
    name: external-svc-mongocluster
  spec:
    hosts:
    - mymongodb.somedomain # not used
    addresses:
    - 192.192.192.192/24 # VIPs
    ports:
    - number: 27018
      name: mongodb
      protocol: MONGO
    location: MESH_INTERNAL
    resolution: STATIC
    endpoints:
    - address: 2.2.2.2
    - address: 3.3.3.3
  
  ```

  与他关联的下游DestinationRule配置：

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: mtls-mongocluster
  spec:
    host: mymongodb.somedomain
    trafficPolicy:
      tls:
        mode: MUTUAL
        clientCertificate: /etc/certs/myclientcert.pem
        privateKey: /etc/certs/client_private_key.pem
        caCertificates: /etc/certs/rootcacerts.pem
  
  ```

  

  一下是ServiceEntry的配置属性：

  - **hosts**：必配项，配置一个 string[] 类型的值。与ServiceEntry关联的需要托管的服务主机名，可以是一个前缀匹配的DNS域名。
  - **addresses**：非必配项，配置一个string[]类型的值。与服务关联的`vip`, 表示目标主机的服务 ip 地址。这样的配置下，请求不仅会对`hosts`进行匹配，且也会匹配请求的 ip 地址，从而来判定是不是属于该服务。
  - **ports**：必配项，配置一个 Port[] 类型的值。这个配置用来关联外部服务。
  - **location**：非必配项，配置一个 Location 类型的值，表示指定将该服务指定为一个网格内部还是网格外部服务。
  - **resolution**：必配项，配置一个Resolution类型的值，表示将`hosts`中配置的地址做服务发现的模式。注意如果是对一个没有配置IP，只配置了端口的服务上设置`resolution`值为`NONE`，那么任意IP都将被路由转发，即相当于`0.0.0.0:<port>`。
  - **endpoints**：非必配项，配置一个 Endpoint[] 类型的值。表示关联服务的节点。
  - **exportTo**：非必配项，配置一个 string[] 类型的值，表示该 ServiceEntry 应用在哪些 Namespace 下面。而应用到哪些 Namespace 下则表示别的 Namespace 下的请求将无法访问该 ServiceEntry ，而该 ServiceEntry 也无法托管其他 Namespace下的 service 。如果不设置，则表示当前 ServiceEntry 应用到所有的 Namespace下。
    - “.”  表示该 ServiceEntry 只应用到当前的 Namespace 下。
    - “*”  表示应用到所有的Namespace下。

  ### ServiceEntry.Endpoint：

  用来配置外部服务的节点。

  - **address**：必配项，配置一个 string 类型的值。配置外部服务节点的地址，如果`resolution`的配置是`DNS`时，这里可以配置成一个域名，并且这里的域名不能使用通配符。
  - **ports**：非必配项，配置一个 map<string, uint32> 类型的值。表示该节点对外提供服务的一组端口。
  - **labels**：非必配项，配置一个 map<string, string> 类型的值。表示一个或多个与节点关联的`labels`。
  - **network**：非必配项，配置一个 string 类型的值。这是一个高级配置，被用在一个istio 网格覆盖多个集群的情况下对子网的指定。具体参考 Istio官方文档。
  - **locality**：非必配项，配置一个 string 类型的值。节点的所在区域配置，用来对 `endpoint` 的区域配置，在处理多实例区域性故障时可以考虑使用。
  - weight：非必配项，配置一个 uint32 类型的值。表示该节点的权重。

  ### ServiceEntry.Location：

  ServiceEntry.Location用来标记该服务的行为特性，比如服务间的 mTLS 会话认证，策略执行。

  - MESH_EXTERNAL：标记为网格外服务。用来标记对外部服务的API访问。使用这种方式，服务将不作为网格的一部分。
  - MESH_INTERNAL：标记服务为网格内服务的一部分。

  ### ServiceEntry.Resolution：

  - NONE：如果一个请求连接已经被解析到一个。
  - STATIC：表示使用 `endpoints` 中配置的 ip 地址作为服务后端。
  - DNS：表示请求通过配置的 DNS 查询 ip地址。

