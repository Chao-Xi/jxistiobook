# **第五节 Istio面试**

### **传统框架组件存在的问题**

* 各种基础组件维护成本比较高，单点风险存在。
* 各语言客户端特性难以统一，熔断、重试等特性实现颇有不同，且不能做到动态线上进行调整。
* 客户端版本迭代难以推动。
* 微服务某些能力上跟其他大厂还有差距。
* 与社区、云原生距离较远，配合常见的开源项目紧密度较差。

### **Service Mesh 可以很好的解决上面的这些问题：**

* 提升微服务治理能力，**规范的引入精准的熔断、限流、流量管理等手段。**
* **减少客户端维护、迭代的投入**。
* **提升服务间通信速度**。
* **具备故障注入能力**。
* **具备动态服务路由调整的能力**。

### **迁移到Istio的基本要求**

* **保证业务无感知迁移，不变更代码**
* **保证可回滚**
* **保证高可用**
* **保证性能无明显下滑**
* **保证 Mesh 内的服务和原服务可以互相访问**

## 服务网格和 Istio 概览

* 服务网格如何帮助服务之间的通信逻辑: **将逻辑分离到基础实施层**
* 从宏观上看，Istio服务网格由哪两部分组成: **数据平面和控制平面**
* 服务网格的代理也叫做什么: **Sidecar**
* Istio中的哪个组件可以将YAML规则转换为Envoy配置: **Istiod**
* 从下面的列表中选出微服务架构的两个好处？: **可伸缩性 & 更小的团队**


### **为什么需要服务网格**

* 使用指标识别，性能和可靠性问题
* 在Grafana等工具中实现指标的可视化；这进一步允许改变并与PagerDuty整合
* 使用Jaeger或Zipkin对服务进行调试和追踪
* 基于权重和请求的流量路由，金丝雀部署，A/B测试
* 流量镜像
* 通过超时和重试提高服务的弹性
* 通过在服务之间注入故障和延迟来进行混沌测试
* 检测和弹出不健康的服务实例的断路器。


### **Istlo简介**

Istio是一个服务网格的开源实现。从宏观上来看Istio支持以下功能。

* 流量管理
* 可观察性
* 安全性

### **Istio组件**

**数据平面：**

服务网格中的数据平面由负责路由、负载均衡、服务发现、健康检查和授权/认证的Envoy代理。这些代理每个实例服务实例要边运行，拦截所有传入和传出的用户流量

**Envoy (data plane)**

* High-performance proxy (C++)
* Injected next to application containers
* Intercepts all traffic for the service
* Pluggable extension model based on WebAssembly


**控制平面：**

**服务网格中的控制平面提供配置并控制数据平面。**

**它由API和工具组成用于管理数据平面行为。**

服务网格运维可以访问控制平面以配置服务网格中的数据平面行为。例如，将流量配置应用于控制平面翻译配置并将其推送到数据平面

**Istiod (control plane)**

* Service discovery
* Configuration
* **Certificate management**
* Converts YAML to Envoy-readable configuration
	* propagates it to all sidecars in the mesh

**Envoy：**

Envoy代理是个现代的、高性能边缘的、小型的L7代理。Envoy是为大型现代面向服务架构设计的。它可以与Nginx和HAproxy等软件负截均街器相媲美。


## **可观察性：遥测和日志**

Istio生成三种类型的遥测数据，为网格中的服务提供可观察性：

* 指标度量（Metric)
* 分布式追踪
* 访问日志

**指标**

Istlo基于四个黄金信号生成指标：延迟、流量、错误和饱和度。

* **Latency**: the time it takes to service a request
* **Traffic**: how much demand is placed on the system
* **Errors**: rate of failed requests
* **Saturation**: how full the most constrained resources of service are

**Zipkin： Trace and spans**

* **Trace**: collection of spans
* **Span**: name, start/finish timestamp, tags, context
* **Tags**: applied to all spans, used for querying/filtering
* Spans are sent to the collector to validate, index & store the data

**Distributed Tracing with Zipkin**

* Zipkin = distributed tracing system
* Discover latency and performance issues
* Requirements: propagate HTTP headers (B3 and Envoy request ID)
	* Traces captured at service boundaries
* Instrument your applications



* Prometheus是用来做什么的：**记录指标，跟踪Istio和网格中的应用程序的健康状况**
* 什么是Zipkin： **分布式追踪系统**
* 什么是Kiali： **Istio可观测性面板**
* Istio依靠哪些头信息来进行分布式追踪： **B3 header和Envoy requestID**
* 什么是Grafana： **Prometheus和其他来源的数据的可视化平台**

## **流量管理**

### **使用Gateway**

* 使用入口网关，我们可以对进入集群的流量应用路由规则。我们可以有一个指向入口网关的单一外部IP地址，并根据主机头将流量路由到集群内的不同服务。
* 我们可以使用Gateway资源来配置网关。网关资源描述了负载均衡器的暴露端口、协议、SNI （服务器名称指示）配置等。

`kind: Gateway`

### **简单路由**

* 使用VirtualService资源在Istio服务网格中进行流量路由。
* 目的地指的是不同的子集（subset）或服务版本。通过子集，我们可以识别应用程序的不同变体。
* **我们可以在一个名为DestinationRule的资源类型中声明子集,  Destination Rule中的流量策略**。
	* 我们可以在`trafficPolicy`字段下设置流量策略设置: `kind: DestinationRule`
		*  Load balancer settings
		* Connection pool settings
		* Outlier detection
		* Client TLS settings
		* Port traffic policy

* **Load balancer settings**

```
trafficPolicy:
   loadBalancer:
    simple: ROUND_ROBIN
```

* Connection pool settings

```
trafficPolicy:
   connectionPool:
    http:
     http2MaxRequests: 50
```
  
* **异常点检测**

如果一个主机开始返回5xx HTTP错误，它就会在预定的时间内被从负载均衡池中弹出。对于TCP服务，Envoy将连接超时或失败计算为错误。

* **客户端TLS设置**
	* 其他支持的TLS模式有DISABLE（没有TLS连接）,
	* SIMPLE（在上游端点发起TLS连接,）
	* `ISTIO_MUTUAL`（与MUTUAL类似，使用Istio的mTLS证书）

```
trafficPolicy:
  tls:
   mode: MUTUAL
   clientCertificate: /etc/certs/cert.pem
   privateKey: /etc/certs/key.pem
   caCertificates: /etc/certs/ca.pem
```
 
* **端口流量策略**

```
trafficPolicy:
  portLevelSettings:
  - port:
    number: 80
   loadBalancer:
    simple: LEAST_CONN
```

* **弹性**

**这不是为了避免故障，而是以一种没有停机或数据丢失的方式来应对故障。弹性的目标 是在故障发生后将服务恢复到一个完全正常的状态**

* Timeouts
* Retry policies
* Configurable on VirtualService
* Retries
	* Number of retries
	* Timeout per try
	* Conditions to retry o

使服务可用的一个关键因素是在提出服务请求时使用超时（timeout）和重试（retry）策略。 我们可以在Istio的VirtualService上配置这两者。

```
...
- route:
  - destination:
    host: customers.default.svc.cluster.local
    subset: v1
  timeout: 10s
```

**重试策略**

```
...
- route:
  - destination:
    host: customers.default.svc.cluster.local
    subset: v1
  retries:
   attempts: 10
   perTryTimeout: 2s
   retryOn: connect-failure,reset
...
```
上述重试策略将尝试重试任何连接超时（connect-failure)或服务器完全不响应 (reset)的失败请求。我们将每次尝试的超时时间设置为2秒，尝试的次数设置为10次。


* **故障注入**

	* **有两种类型的故障注入**。
		* 我们可以在转发前延迟（delay）请求，模拟缓慢的网络或过载的服务，我们可以中止（abort) HTTP请求，并返回一个特定的HTTP错误代码给调用者。
		* 可选的延迟

```
- route:
  - destination:
    host: customers.default.svc.cluster.local
    subset: v1
  fault:
   abort:
    percentage:
     value: 30
    httpStatus: 404
```

```
 fault:
   delay:
    percentage:
     value: 5
    fixedDelay: 3s
```

### **高级路由**

**Istio允许我们使用传入请求的一部分，并将其与定义的值相匹配。例如，我们可以匹配传入请求的URI前缀，并基于此路由流量。**

* **重定向和重写请求**

```
...
http:
  - match:
   - headers:
     my-header:
      exact: hello
   redirect:
    uri: /hello
    authority: my-service.default.svc.cluster.local:8000
...
```

### **ServiceEntry**

通过ServiceEntry资源，我们可以向Istio的内部服务注册表添加额外的条目，使不属于我们网格的外部服务或内部服务看起来像是我们服务网格的一部分。

* Add entries to Istio's service registry
* Use traffic routing, failure injection, and other features against external services

`kind: ServiceEntry`

```
hosts:
   - api.external-svc.com
  ports:
   - number: 443
    name: https
    protocol: TLS
  resolution: DNS
  location: MESH_EXTERNAL
```

### **WorkloadEntry**

* Handle migration of VM workloads to Kubernetes
* Specify VM workloads and make them part of Istio's service registry

在`Workload Entry`中，我们可以指定在虚拟机上运行的工作负载的细节（名称、地址、标签），然后使用`ServiceEntry`中的`workloadSelector`字段，使虚拟机成为Istio内部服务注册表的一部分。

### **Sidecar**

Sidecar资源由三部分组成，一个工作负载选择器、一个入口（ingress)监听器和一个出口 (egress)监听器。

**Sidecar Resource**

* Proxies accept traffic on all ports
* Configure Sidecar to specific ports/services
* Three parts:
	* Workload selector
	* Ingress listener
	* Egress listener

工作负载选择器决定了哪些工作负载会受到sidecar配置的影响。

### **Envoy Filter**

EnvoyFilter资源允许你定制由Istio Pilot生成的Envoy配置。使用该资源，你可以更新数值，添加特定的过滤器，甚至添加新的监听器、集群等等。小心使用这个功能，因为不正确的 定制可能会破坏整个网格的稳定性。

**`kind: EnvoyFilter`**

### 测试

* 我们可以用什么资源配置Istlo ingress gateway?  **Gateway : istio: ingressgateway**
* 使用**VirtualService**资源，你可以将流量路由到Kubernetes内部的一个特定服务。
* Istio ingress 网关是作为哪种服务类型部署的？: **Loadbalancer**
* 你可以在哪个资源中为HTTP流量指定路由规则？: **VirtuaLService**
* 如何定义多个服务版本，以便能够在它们之间进行流量路由？**使用Destination Rule和subset**
* 当使用离群检测时，主机何时会被从负载均衡池中弹出？ **当返回HTTP 5xx错误时**
* 在离群检测配置中，间隔设置指的是什么  **扫描上游主机时间的间隔**
* 能将流量策略应用于特定的端口吗？: **能** `trafficPolicy: portLevelSettings: port`
* `ISTIO_MUTUAL`和`MUTUAL TLS`模式之间有什么区别？ **`ISTIO_MUTUAL`在mTLS中使用Istio的证书，而在MUTUAL TLS中你可以用你自己的证书**
	*  `ISTIO_MUTUAL`（与MUTUAL类似，使用Istio的mTLS证书）
* 我们可以只在上游服务器返回5xx响应代码时重试请求，或者只在网关错误（HTTP 5O2、 503或504）时重试，或者甚至在请求头中指定可重试的状态代码。
* 如果一个端点失败并触发重试，会发生什么: **当Envoy重试请求时，端点被从负载均衡池中排除** 重试和超时都发生在客户端。当Envoy重试一个失败的请求时，最初失败并导致重试的端点就不再包含在负载均衡池中了。假设Kubernetes服务有3个端点（Pod)，其中一个失败了， 并出现了可重试的错误代码
* 你能为同一个请求同时注入延迟和中止吗. **是**
* 考虑到下面的片段，有多少百分比的请求会被中止？ **如果我们不指定百分比，所有的请求将被中止**
* 故障注入是否触发重试策略？**注意，故障注入将不会触发我们在路由上设置的任何重试策略。例如，如果我们注入了一个HTTP 500的错误，配置为HTTP 500上的重试策略将不会被触发**。
* 你可以使用VirtualService中的哪个字段将请求重定向到Kubernetes中一个完全不同的服务？ **redirect**
* 你可以使用哪种资源来使外部服务成为你的服务网格的一部分 **ServiceEntry** 
	* 通过ServiceEntry资源，我们可以向Istio的内部服务注册表添加额外的条目，使不属于我们网格的外部服务或内部服务看起来像是我们服务网格的一部分
*  Envoy sidecar检查ServiceEntry资源中的hosts字段的顺序是什么.  **HTTP、 SNI、 IP地址、端口**
*  ServiceEntry资源中的workloadSelector字段的目的是什么？**要选择WorkloadEntry资源，使虚拟机成为Istio内部服务注册表的一部分**
*  如何配置sidecar代理，使其只对某些服务使用特定的端口  **使用Sidecar资源**
*  Sidecar资源中的Ingress和egress监听器字段的目的是什么   **定义哪些入站／出站流量被sidecar接受**
	*  Workload selector 
	* Ingress listener 
	* Egress listener 
* EnvoyFilter资源允许你定制由Istio Pilot生成的Envoy配置
* 当有多个EnvoyFilter部署时，哪些过滤器会先被应用  **根（如istio-system)命名空间的过滤器** 根命名空间（例如istio-system)中的过滤器首先被应用，然后是工作负载命名空间中的所有匹配过滤器。
* 如何将VirtualService与Gateway绑定  **使用gateways字段**

```
...
kind: VirtualService
...
spec:
  hosts:
    - "*"
  gateways:
    - gateway
```

* Destination Rule配置流量策略
* 如果一个端点失败并触发重试，会发生什么  **当Envoy重试请求时，端点被从负载均衡池中排除**
* 你可以用Workload Entry资源做什么？**处理虚拟机工作负载向Kubernetes的迁移**

## **Istio Secuirty**

### **认证(Authentication)**

**Authentication**

* Principal = Kubernetes service
* Action = **HTTP request methods** (e.g. GET, POST, DELETE, ...)
* Object = Kubernetes service
* All about the principal (service identity)
* "Validating a credential and ensuring it's both valid and trusted"

**Identity**

* Unique identity in Kubernetes = Kubernetes service account
* X.509 certificate (from SA) + SPIFFE spec = Identity
	* Naming scheme (spiffe://cluster.localinsidefault/sa/my-sa)
	* **Encoding names into X.509**
	* Validating the X.509 to authenticate the SPIFFE identity

### **证书创建和轮换**

对于网格中的每个工作负载，Istio提供一个`X.509`证书。

**一个名为`pilot-agent`的代理在每个Envoy代理旁边运行**，并与控制平面（istiod）一起工作，自动进行密钥和证书的轮转。

**Certificate creation**

* Istio Agent
* Envoy's Secret Discovery Service(SDS)
* Citadel(part of the control plane)

**Istio Agent与Envoy sidecar一起工作通过安全地传递配置和秘密帮助它们连接到服务网格**。即便Istio代理在每个pod中运行我们也认为它是控制平面的一部分。

秘钥发现服务(SDS)简化了证书管理。如果没有SDS证书必须作为秘密(Secret) 创建，然后装入代理容器的文件系统中。

**每当证书过期时SDS会推送更新的证书Envoy可以立即便用它们。不需要重新部署代理服务器也不需要中断流量。**

### **Peer and Request Authentication(对等认证和请求认证)**

* **Peer authentication**
	* Peer authentication
	* service-to-service
* All mTLS about the communication between services
* Request authentication
	* end user authentication
	* uses JWTs

### **双向TLS**

**Mutual TLS(mTLS)**


* Traffic is re-routed through proxies
* Envoy starts an mTLS handshake
	* Secure naming check is done
* Once mTLS connection is established:
	* **Request is forwarded to server-side proxy**
	* **Server-side forwards traffic to the workload**

**mTLS Behavior**

* **DISABLE**: no TLS connection
* **SIMPLE**: originate to the upstream
* **MUTUAL**: uses mTLS and certs for auth
* **`ISTIO_MUTUAL`**: like MUTUAL; but uses Istlo's certs for mTLS

**Permissive mode**

* **Allows services to accpet both plaintext and mTLS**
* Improves mTLS onboarding experience
* Gradually Install/configure sidecars for mTLS

### **授权**

**Authorization**

* Can user A send a GET request to /hello on service B?
* Authn without authz (and vice-versa) is useless
* Control authenticated principals with AuthorizationPolicy

**Authorization policy**

* Make use of identities extracted from **PeerAuthentication** and **RequestAuthentication**:
	* **requestPrincipals (users)**
	* **principals (service/peer)**
* Defined at mesh, namespace, and workload level

**Authorization policy**

* Workloads policy applies to
* Action to take (deny, allow, or audit)
* Rules when to take action

**Istio允许我们使用`AuthorizationPolicy`资源在网格、命名空间和工作负载层面定义访问控制。`AuthorizationPolicy `支持DENY、ALLOW、 AUDIT和CUSTOM操作**。

`kind: AuthorizationPolicy`

### **安全测试**

*  在通信时，Envoy代理使用什么作为凭证:  **X509证书**
	* **Validating the X.509 to authenticate the SPIFFE identity**
* 哪个组件为一个服务账户创建SPIFFE身份？ **Citadel**
	* **每次我们创建一个新的服务账户时, Citadel都会为它创建一个SPIFFE身份**。
* 你会使用哪种资源来启用命名空间中的严格mTLS模式？ **PeerAuthentication**
* AuthorizationPolicy资源中使用哪两个动作？ **DENY和ALLOW**
	* 每个Envoy代理实例都运行一个授权引擎，在运行时对请求进行授权。当请求到达代理时，引擎会根据授权策略评估请求的上下文，并返回ALLOW或DENY
* 当秘密发现服务（SDS)更新证书时，是否需要重新部署代理以使用新证书。 **否**
* AUDIT动作的目的是什么？ **记录请求**
	* AUDIT动作决定是否记录符合规则的请求。注意AUDIT策略并不影响请求被允许或拒绝。
* 什么是允许模式（permissive mode)?  **一个允许服务同时接受纯文本流最和mTLS流量的选项**
* 是否可以在Istio中使用JSON Web Tokens (JWTs)? **是**
* 选择三个负贵在运行时创建身份的组件  **Citedel 、 lstio Agent 、Envoys Secret Discovery Service**
* Istlo Agent的工作是什么？  **与Envoy sidecar合作，通过专递配置和秘密，帮助他们链接到Service mesh**  **Istio Agent与Envoy sidecar一起工作通过安全地传递配置和秘密帮助它们连接到服务网格**
*  Istlo提供哪两种类型的认证？  **对等和请求认证**
*   Istlo代理将密钥和证书存储在哪里？**内存**  将密钥存储在内存中比存储在磁盘上更安全；
	*  在使用SDS时，绝不会将任何密钥写入磁盘。Istio代理还定期刷新凭证， 在当前凭证过朋前从Citadel检索任何新的SVID(SPIFFE可验证身份文件）
* 哪种代理与istiod一起工作，以实现钥匙和证书轮换的自动化  **pilot-agent**
	* 一个名为`pilot-agent`的代理在每个Envoy代理旁边运行，并与控制平面（istiod）一起工作，自动进行密钥和证书的轮转。
* 安全命名信息包含哪些内容？ **从服务身份到服务名称的映射**
* 如果一个工作负载有多个策略，哪些策略会先被评估？ **Deny 策略**
* 你如何在Istio中启用授权功能？  **不需要明确地启用该功能**
	* 没有必要明确地启用授权功能。**为了执行访问控制，我们可以创建一个授权策略来应用于我们的工作负载 `AuthorizationPolicy`资源是我们可以利用`PeerAuthentication`策略和`RequestAuthentication`策略中的主体的地方**。
* Istio如何验证用户／进程的身份  **通过从请求中提取凭证**
* 你如何在整个网格中强制执行严格的mTLS模式 **通过将PeerAuthentication资源部署到根命名空间(Istlo system）中**
	* 我们可以创建PeerAuthentication资源，首先在每个命名空间中分别执行严格模式。然后，我们可以在根命名空间（在我们的例子中是istio-system)创建一个策略，在整个服务网格中执行该策略：
*  对等认证是用来做什么的:  **服务间认证**
*  果Kubernetes工作负载没有Envoy代理，Istio是否会发送mTLS流量 **否**
*   Authorization Policy资源中的selector和matchLabel字 段有什么用途  **要选择策略适用的工作负载**

### **高级功能测试**

* Istio在网格中使用什么作为租户单位: **Namespace**
* 你使用哪些资源将虚拟机工作负载连接到服务网格？**WorkloadGroup 和 WorkloadEntry**
	* 工作负载组(WorkloadGroup资源）类似于Kubernetes 中的部署(Deployments)它代表了共享共同属性的虚拟机工作负载的逻辑组
	* 描述虚拟机工作负载的第二种方法是使用工作负载条目(WorkloadEntry资源）。工作负载条目类似于Pod它代表了一个虚拟机工作负载的单一实例
* 你是否可以通过将所有集群作为远程集群来实现控制平面和数据平面的完全分离？**是**
	* 另一种部署模式是，我们把所有的集群都视为由外部控制平面控制的远程集群。这使我们在控制平面和数据平面之间有了完全的分离。
* **Istio是如何解决不同租户之间的配置隔离的: 不可行**
	* **租户的另一个重要功能是隔离不同租户的配置。目前，Istio并没有解决这个问题，不过，它通过在命名空间级别上的范围配置来尝试解决这个问题**
* 多集群部署的目的是什么？  **为了获得更大程度的隔离和可用性**． 最佳的多集群部署拓扑结构是每个集群都有自己的控制平面。对于正常的服务网格部署规模建议你使用多网格部署，并有一个单独的系统在外部协调网格。
* 什么是租户？ **租户是一组共享工作负载的共同访问权和权限的用户。租户之间的隔离是通过网络配置和策略完成的**。
* What does the shared control plane model involve?  **多个集群控制平面只在一个集群中运行（主集群）**
* 什么被认为是最佳的多集群部署拓扑结构？ **每个集群都有自己的控制平面**
* 不同租户之间的隔离是如何实现的？ **使用网络配置和策略**
* 工作负载应该如何在集群之间进行通信？ **通过ingress gateway**

## **Istio 问题排查**

* `istioctl proxy-config`列出监听器
* 配置 `istioctl validate -f myresource.yaml`
* 我们可以使用另一个命令`istioctl analyze`
* `istioctl proxy-status`
* `istioctl dashboard controlz`
* 如果虚拟监听器找不到请求的目的地会怎样  **根据OutboundTrafficPolicy发送流量**
* 你可以使用哪个Istio CLI命令来列出所有监听器  **`istiocti proxy-config listeners`**
* 监听器使用哪个服务来寻找路由配置  **RDS(route discovery service)**
* Envoy的基本概念是什么？  **listener、route、cluster和endpoint**
* **每个sidecar都有多个监听器生成。每个sidecar都有一个监听器，它被绑定到 0.0.0.0:15006**
* Where are the circuit breakers defined **In clusters configuration**
* 什么类型的流量会被路由到15006端口 **入站Pod流量**




























































 
  










