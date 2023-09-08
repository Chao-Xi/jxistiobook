# **3 大规模场景下对Istio的性能优化**

## 简介

**当前istio下发xDS使用的是全量下发策略，也就是网格里的所有sidecar(envoy)，内存里都会有整个网格内所有的服务发现数据。**

这样的结果是，每个sidecar内存都会随着网格规模增长而增长。

## **Aeraki-mesh**

* egress：充当类似网格模型中默认网关角色
* controller：用来分析并补全服务间的依赖关系

### Egress

对应的配置文件为：lazyxds-egress.yaml

下面来一一查看该组件的组成部分

组件配置

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-egressgateway-lazyxds
  namespace: istio-system
  labels:
    app: istio-egressgateway-lazyxds
    istio: egressgateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: istio-egressgateway-lazyxds
      istio: egressgateway
  template:
    metadata:
      annotations:
        sidecar.istio.io/discoveryAddress: istiod.istio-system.svc:15012
        sidecar.istio.io/inject: "false"
      labels:
        app: istio-egressgateway-lazyxds
        istio: egressgateway
    spec:
      containers:
        - args:
            ......
          image: docker.io/istio/proxyv2:1.10.0
          imagePullPolicy: IfNotPresent
          name: istio-proxy
          ports:
            - containerPort: 8080
              protocol: TCP
            - containerPort: 15090
              name: http-envoy-prom
              protocol: TCP
          ......
          volumeMounts:
            - mountPath: /etc/istio/custom-bootstrap
              name: custom-bootstrap-volume
            ......
      volumes:
        - configMap:
            defaultMode: 420
            name: lazyxds-als-bootstrap
          name: custom-bootstrap-volume
```

**由于配置太多，这里只挑选主要的部分，从上面可以看出，其实是启动一个`istio proxy`，该`proxy`的启动配置文件是使用的`configmap`挂载出来的**。

### 启动配置

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: lazyxds-als-bootstrap
  namespace: istio-system
data:
  custom_bootstrap.json: |
    {
      "static_resources": {
        "clusters": [{
          "name": "lazyxds-accesslog-service",
          "type": "STRICT_DNS",
          "connect_timeout": "1s",
          "http2_protocol_options": {},
          "dns_lookup_family": "V4_ONLY",
          "load_assignment": {
            "cluster_name": "lazyxds-accesslog-service",
            "endpoints": [{
              "lb_endpoints": [{
                "endpoint": {
                  "address": {
                    "socket_address": {
                      "address": "lazyxds.istio-system",
                      "port_value": 8080
                    }
                  }
                }
              }]
            }]
          },
          "respect_dns_ttl": true
        }]
      }
    }
```
 
 从上面配置可以知道：

* 定义了proxy组件代理的集群，该集群为"`lazyxds-accesslog-service`"
 
* 该集群对应的后端服务地址为"lazyxds.istio-system"，端口为8080 这个后端就是lazyxds controller，后面细说

### EnvoyFilter

从yaml文件我们看到，还定义了一个`envoyfilter`来修改`proxy`代理的流量配置

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: lazyxds-egress-als
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      app: istio-egressgateway-lazyxds
  configPatches:
    - applyTo: NETWORK_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: MERGE
        value:
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
            access_log:
              ......
              - name: envoy.access_loggers.http_grpc
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.access_loggers.grpc.v3.HttpGrpcAccessLogConfig
                  common_config:
                    log_name: http_envoy_accesslog
                    transport_api_version: "V3"
                    grpc_service:
                      envoy_grpc:
                        cluster_name: lazyxds-accesslog-service
```

从这个配置文件，可以看出在启动envoy时，会向其注入一个accesslog service，也就是envoy的日志收集器，而这个service就是`lazyxds-accesslog-service`

### Controller

具体的lazy xds实现就是通过这个controller实现的

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: lazyxds
  name: lazyxds
  namespace: istio-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lazyxds
  template:
    metadata:
      labels:
        app: lazyxds
    spec:
      serviceAccountName: lazyxds
      containers:
        - image: aeraki/lazyxds:latest
          imagePullPolicy: Always
          name: app
          ports:
            - containerPort: 8080
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: lazyxds
  name: lazyxds
  namespace: istio-system
spec:
  ports:
    - name: grpc-als
      port: 8080
      protocol: TCP
  selector:
    app: lazyxds
  type: ClusterIP
```

从配置可以看到，在egress环节我们知道了proxy的代理的后端地址为`lazyxds.istio-system`，刚好对应这里的`controller`。

并且我们还知道，envoy的访问日志最终会发送给这个controller来处理，而这就是实现增量下发envoy配置的关键之处，也就是解决istio性能的解决之法。

### 增量下发

**Accesslog接口**

要接受envoy的访问日志，必须实现envoy定义的接口：

```
type AccessLogServiceServer interface {
   // Envoy will connect and send StreamAccessLogsMessage messages forever. It does not expect any
   // response to be sent as nothing would be done in the case of failure. The server should
   // disconnect if it expects Envoy to reconnect. In the future we may decide to add a different
   // API for "critical" access logs in which Envoy will buffer access logs for some period of time
   // until it gets an ACK so it could then retry. This API is designed for high throughput with the
   // expectation that it might be lossy.
   StreamAccessLogs(AccessLogService_StreamAccessLogsServer) error
}
```

### 日志解析

lazyxds实现如下：

```
func (server *Server) StreamAccessLogs(logStream als.AccessLogService_StreamAccessLogsServer) error {
   for {
      data, err := logStream.Recv()
      if err != nil {
         return err
      }

      httpLog := data.GetHttpLogs()
      if httpLog != nil {
         for _, entry := range httpLog.LogEntry {
            server.log.V(4).Info("http log entry", "entry", entry)
            fromIP := getDownstreamIP(entry)
            if fromIP == "" {
               continue
            }

            upstreamCluster := entry.CommonProperties.UpstreamCluster
            svcID := utils.UpstreamCluster2ServiceID(upstreamCluster)

            toIP := getUpstreamIP(entry)

            if err := server.handler.HandleAccess(fromIP, svcID, toIP); err != nil {
               server.log.Error(err, "handle access error")
            }
         }
      }
   }
}
```

上面主要的逻辑就是解析envoy的访问日志，然后进行处理：

* lazy xds Controller 会对接收到的日志进行访问关系分析，然后把新的依赖关系表达到 sidecar CRD 中。
* 同时 Controller 还会更新 Egress 的规则：删除、更新或创建。