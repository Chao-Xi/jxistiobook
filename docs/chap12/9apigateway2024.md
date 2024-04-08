# API Gateway 技术漫谈 2024

## 什么是 API Gateway?

所谓的 API Gateway 就是在数据面开发一些鉴权限流功能。


针对 api gateway 来说，**限流插件更关心 consumer 这样的概念**，针对不同的 consumer 进行不同的限流配置。

除此之外，很多业务服务都会有通用的逻辑，比如对业务请求进行 metric 统计，对异常请求进行重试，对外提供统一的 API 协议等等，这些通用逻辑都可以放到 api gateway 中来实现。更近一步，api gateway 应该是一个对 api 进行托管的服务，提供对 API 的完整生命周期管理，包括创建、维护、发布、运行以及下线。

## APISIX

笔者所在公司内部使用的 api gateway 产品就是基于 apisix为底座进行二次开发，主要考虑到 apisix 的以下优点：

1. 丰富的开箱即用的插件，基本上常见的需求都能找到对应的插件，并且个性化的需求自己写扩展的插件也方便。
2. **apisix 所抽象的 route/service/upstream/consumer/plugin 这些概念都很优雅**，作为网关来说，几乎能够满足所有的场景，比 k8s 里面的 ingress/service 这两个概念好很多，比如你想表达按百分比转发流量，这个用 ingress/service 就无法表达出来。（目前 k8s 也认识到 ingress/service 对流量管控表达能力弱的问题，推出了新的 gatewayapi，但是实际体验下来，对比 apisixroute 仍有一定的差距）
3. 支持多集群管理：**一套 apisix 可以代理多套集群，某些场景下确实存在这种需求。（开源的 nginx-ingress-controller 就不能一套网关代理多套 k8s 集群）**
4. 支持对接多种服务发现组件：ZooKeeper、Consul、Nacos、kubernetes等
5. **高性能的路由匹配算法（用 radixtree 实现了一个路由匹配算法）**


apisix 官方宣称是一个动态、实时、高性能 api gateway，前三个词 动态、实时、高性能其实不是 apisix 自身的功能，而是 openresty(nginx) 的功能。


* 高性能: apisix 底层依托于高性能的 nginx+luajit(openresty)，当然这个高性能是跟 java 之类的网关相比，如果跟 envoy 或者一些使用 rust 写的网关比的话，性能平分秋色，没有绝对的优劣。
* 高稳定性：稳定是作为网关最核心的要素，而底层的 nginx 作为一个入口网关，经过了几十年的发展，使用基数也很庞大，很多特殊需求场景都考虑到，这个从  nginx 大量可配置的参数就能够感受到。而如果使用一个全新的网关底座，说不定哪天就会遇到一个坑。
* 动态：不仅修改配置无需重启服务，甚至修改 luajit 的代码也无需重启服务。这对于网关这种零停服要求的基础设施来说，是非常重要的功能。比如对于长连接的请求，这个连接可能存在几个小时甚至几天，如果直接重启网关服务，就会造成影响。笔者认为这是 openresty 时至今日依然无可替代的优点，目前几乎没有其他语言或者框架实现这一特性。因为对于静态编译语言来说，符号之间的引用关联非常复杂，很难实现解耦。笔者自己在做一些软件的时候，曾经也实现过一些 reload 的功能，发现这并非是一件易事。而 nginx 的内部组件依赖关系要更加复杂的多，可以想象这是一件难度很大的事情。


### 当然 apisix 也存在一些缺点：

* **官方提供的一些插件质量不高，没有经过较大规模的生产环境的验证**。笔者曾在生产环境中使用过几款 apisix 官方的插件，均出现过一些小问题。
* **apisix 使用 etcd 作为它的数据存储，etcd v3 使用的是 grpc 协议(底层 http2 协议)，但是由于 openresty 生态一直都缺乏一个 http 2.0 客户端（一是 http2.0 本身较复杂，二是 cosocket 也很难实现 http2.0），apisix 官方只能在 http1.1 基础上，通过 long poll 的方式与 etcd 进行通讯，这就导致每个 nginx worker 要与 etcd 建立多个连接，而如果使用 http2.0 ，那么每个 worker 只要建立一个连接即可**。除此之外 apisix 官方实现的 lua-resty-etcd，存在某些问题，比如当你 watch 的版本号被 etcd 压缩后，etcd 就会取消这个 watcher，你需要重建 watcher，而之前 apisix 却没有处理这个错误，导致问题（目前该问题已经修复）。这里对比其他基于 openresty 开发的网关，kong 就是使用 pg 作为它的数据库，而 PG 有成熟稳定的开源库，它就不会遇到 apisix 的这些问题。
* **apisix 支持 webassemby，但是它实现的 runtime 还是过于简陋，暴露出来的接口太少，达不到生产可用的状态**。另外虽然也支持 java/go/php 等 其他语言写插件，但是一旦用了这些语言写的插件，性能就会指数级下降。目前只有用 luajit 写的插件才会有高性能。

## OpenResty

开源社区中基于 openresty 推出的 apigateway 产品有 apisix 和 kong，笔者之前一直有个疑问，为什么 openresty 不推出自己的 ingress 网关和 apigateway 产品，后来了解到，openresty 官方其实有对应的商业产品 openresty edge，只是没有开源。

毫无疑问，过去 10 年由 Kubernetes 代表的云原生方向是 IT 基础设施领域最火的一个技术方向。其中 kubernetes 的一个核心理念就是 pod 的动态变化，**而这个理念与 openresty 的动态不谋而合（先有 openresty，后有 kubernetes），所以 kubernetes 官方所推出的 nginx-ingress-controller 的底层就是使用 nginx-lua-module （openresty 的核心模块）所实现**

**另外依托于 kubernetes 所诞生的 service mesh 技术，它的数据面目前使用的是 envoy，其实在早期的时候，谷歌最开始的计划是使用 nginx，但是由于一些问题最终放弃了 nginx。**

**luajit：**


* 生态较差，比如你要把数据发送到 kafka，或者从 zookeeper 拿点数据，就会缺少稳定的开源库（有些场景虽然存在开源库，但是稳定性又不够）。唯一的解决办法就是代理到别的服务去做，如果这样为何要用？
* luajit 的高性能是建立在能够 jit 的情况下，但是实际很难 100% 保证 jit，经常遇到不能 jit 的情况。

### nginx 多进程架构：

* 访问 nginx 的请求被调度到哪个进程其实是随机，如果一个进程所处理的连接里面的http请求很多很繁忙，那么它也无法委托给其他进程来代劳，即便其他进程是空闲的。这点对于最新的 http2 而言非常明显，因为 http2.0 一旦建立了一个连接，它就只能固定在一个进程里面，这带来的问题就是前端应用在迁移到 http2.0 之后，如果遇到 nginx worker 繁忙，它的 TTFB（首字节时间）性能下降。
* 多进程之间无法安全地共享资源。nginx的方案是放数据在共享内存里面，并且只能将字符串和数字放入共享内存，每次访问共享内存都必须使用互斥锁。openresty 的 share_dict 就是放在共享内存中的红黑树，每次对它进行修改，就要涉及多个地方的改动（红黑树的操作比较复杂，涉及较多的代码），而如果在改动期间，恰好 worker 进程崩溃了，虽然 nginx 会捕获 SIGCHILD 信号，进行释放锁，但是父进程并不会对这些操作到一半的过程进行撤销恢复，导致数据处于一个诡异的状态。

## ingress-controller

由于 kubernetes 越来越流行，新时代的 api gateway 必须要支持 kuberentes/ingress。目前谷歌官方为 kubernetes  ingress 推出的产品是 nginx-ingress-controller ，但是它的问题在于，数据面与控制面绑定在一起（一个 pod 中含有两个容器），这就导致它的横向扩展能力受限。对于一家大型公司来说，它的流量可能会非常的庞大，那么它的数据面需要的节点数量可能会有几十上百个，这就导致控制面也相应的增加，从而对 k8s 产生一定的压力。

另外针对大规模 k8s 集群，控制面的内存消耗也会比较多（需要在本地内存中缓存所有的 pod/service/endpoint/ingress 的资源）。

apisix 官方推出的 apisix-ingress-controller 让 apisix 支持 kubernetes 作为服务发现，并且是控制面和数据面分离的架构。**其原理是 apisix-ingress-controller watch kubernetes resource 的变化，将数据同步写入到 apisix 的 etcd 中，但目前官方 apisix-ingress-controller 存在一些问题**：

1. 目前官方 apisix-ingress-controller  对 ingress/service 的支持不够完善，部分特性依然不支持（部分 ingress 的 annotation 不支持）。其实 apisix-ingress-controller 推荐使用 apisix 自己的 apisixroute/apisixupstream 等 CRD
2. apisix-ingress-controller 的代码质量不高的，很多模块耦合严重，并且有一些 bug

apisix 需要一个etcd 来存储数据，而 apisix-ingress-controller 的本质却是使用 k8s 的 etcd ，你会发现这里有两个 etcd。如果 apisix 需要同时代理 kuberentes 的流量和非 kubernetes 的流量，两套 etcd 没有问题，但如果 apisix 只代理 kubernetes 的流量的话，严格来说只需要一套存储，并且始终应该以 kubernetes 的数据为准。两套存储不仅增加维护的成本，还会带来数据不一致的问题。

所以为了解决这个弊端，在新版本的 apisix-ingress-controller 中实现了 etcd server 的功能，让 apisix 直接从 apisix-ingress-controller 中读取数据，大幅优化了系统架构。
