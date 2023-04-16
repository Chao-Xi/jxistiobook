# 8 APISIX认证与自定义插件

身份认证在日常生活当中是非常常见的一项功能，大家平时基本都会接触到。比如用支付宝消费时的人脸识别确认、公司上班下班时的指纹/面部打卡以及网站上进行账号密码登录操作等，其实都是身份认证的场景体现。

![Alt Image Text](../images/api1_8_1.png "body image")

如上图，Jack 通过账号密码请求服务端应用，服务端应用中需要有一个专门用做身份认证的模块来处理这部分的逻辑。

请求处理完毕子后，如果使用 JWT Token 认证方式，服务器会反馈一个 Token 去标识这个用户为 Jack。如果登录过程中账号密码输入错误，就会导致身份认证失败。

但是每个应用服务模块去开发一个单独的身份认证模块，用来支持身份认证的一套流程处理，当服务量多了之后，就会发现这些模块的开发工作量都是非常巨大且重复的。这个时候，我们可以通过把这部分的开发逻辑放置到 Apache APISIX 的网关层来实现统一，减少开发量。

![Alt Image Text](../images/api1_8_2.png "body image")

如上图所示，用户或应用方直接去请求 Apache APISIX，然后 Apache APISIX 通过识别并认证通过后，会将鉴别的身份信息传递到上游应用服务，之后上游应用服务就可以从请求头中读到这部分信息，然后进行后续的逻辑处理。

Apache APISIX 作为一个 API 网关，目前已开启与各种插件功能的适配合作，插件库也比较丰富。目前已经可与大量身份认证相关的插件进行搭配处理，如下图所示。

![Alt Image Text](../images/api1_8_3.png "body image")

基础认证插件比如 Key-Auth、Basic-Auth，他们是通过账号密码的方式进行认证。

复杂一些的认证插件如 Hmac-Auth、JWT-Auth，如 Hmac-Auth 通过对请求信息做一些加密，**生成一个签名，当 API 调用方将这个签名携带到 Apache APISIX，Apache APISIX 会以相同的算法计算签名，只有当签名方和应用调用方认证相同时才予以通过**。

其他则是一些通用认证协议和联合第三方组件进行合作的认证协议，例如 OpenID-Connect 身份认证机制，以及 LDAP 认证等。

Apache APISIX 还可以针对每一个 Consumer （即调用方应用）去做不同级别的插件配置。

如下图所示，我们创建了两个消费者 Consumer A、Consumer B，我们将 Consumer A 应用到应用 1，则后续应用 1 的访问将会开启 Consumer A 的这部分插件，例如 IP 黑白名单，限制并发数量等。

将 Consumer B 应用到应用 2 ，由于开启了 http-log 插件，则应用 2 的访问日志将会通过 HTTP 的方式发送到日志系统进行收集。

![Alt Image Text](../images/api1_8_4.png "body image")

> 配置灵活

总体说来 APISIX 的认证系统功能非常强大，我们非常有必要掌握。

### basic-auth

首先我们来了解下最简单的基本认证在 APISIX 中是如何使用的。

basic-auth 是一个认证插件，它需要与 Consumer 一起配合才能工作。添加 Basic Auth 到一个 Service 或 Route，然后 Consumer 将其用户名和密码添加到请求头中以验证其请求。

首先我们需要在 APISIX Consumer 消费者中增加 basic auth 认证配置，为其指定用户名和密码，我们这里在 APISIX Ingress 中，可以通过 ApisixConsumer 资源对象进行配置，比如这里我们为前面的 nexus 实例应用添加一个基本认证，如下所示：

```
# apisix-basic-auth.yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  name: nexus-bauth
spec:
  authParameter:
    basicAuth:
      value:
        username: admin
        password: admin321
```

ApisixConsumer 资源对象中只需要配置 authParameter 认证参数即可，目前支持 basicAuth、hmacAuth、jwtAuth、 keyAuth、wolfRBAC 等多种认证类型，在 basicAuth 下面可以通过 value 可直接去配置相关的 username 和 password，也可以直接使用 Secret 资源对象进行配置，比起明文配置会更安全一些。

然后在 `ApisixRoute` 中添加 `authentication`，将其开启并指定认证类型即可，就可以实现使用 Consumer 去完成相关配置认证，如下所示：

```
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: nexus
  namespace: default
spec:
  http:
    - name: root
      match:
        hosts:
          - ops.foobar.com
        paths:
          - "/nexus*"
          - "/static/*"
          - "/service/*"
      plugins:
        - name: proxy-rewrite
          enable: true
          config:
            regex_uri: ["^/nexus(/|$)(.*)", "/$2"]
        - name: redirect
          enable: true
          config:
            regex_uri: ["^(/nexus)$", "$1/"]
        - name: redirect
          enable: true
          config:
            http_to_https: true
      backends:
        - serviceName: nexus
          servicePort: 8081
      authentication: # 开启 basic auth 认证
        enable: true
        type: basicAuth
```

直接更新上面的资源即可开启 basic auth 认证了，在 Dashboard 上也可以看到创建了一个 Consumer：

![Alt Image Text](../images/api1_8_5.png "body image")

然后我们可以进行如下的测试来进行验证：

```
# 缺少 Authorization header
➜ curl -i http://test.foobar.com/nexus/
HTTP/1.1 401 Unauthorized
Date: Tue, 28 Mar 2023 08:12:01 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
WWW-Authenticate: Basic realm='.'
Server: APISIX/3.2.0

{"message":"Missing authorization in request"}
# 用户名不存在
➜ curl -i -ubar:bar http://test.foobar.com/nexus/
HTTP/1.1 401 Unauthorized
Date: Tue, 28 Mar 2023 08:12:19 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.2.0

{"message":"Invalid user authorization"}
# 成功请求
➜ curl -i -uadmin:admin321 http://test.foobar.com/nexus/
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 9024
Connection: keep-alive
Date: Tue, 28 Mar 2023 08:12:28 GMT
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Last-Modified: Tue, 28 Mar 2023 08:12:28 GMT
Pragma: no-cache
Cache-Control: no-cache, no-store, max-age=0, must-revalidate, post-check=0, pre-check=0
Expires: 0
Server: APISIX/3.2.0

<html>
<head><title>301 Moved Permanently</title></head>
# ......
</html>
```

### consumer-restriction

不过这里大家可能会有一个疑问，在 Route 上面我们并没有去指定具体的一个 Consumer，然后就可以进行 Basic Auth 认证了，那如果我们有多个 Consumer 都定义了 Basic Auth 岂不是都会生效的？

确实是这样的，这就是 APISIX 的实现方式，所有的 Consumer 对启用对应插件的 Route 都会生效的，如果我们只想 Consumer A 应用在 Route A、Consumer B 应用在 Route B 上面的话呢？

**<mark>要实现这个功能就需要用到另外一个插件：consumer-restriction</mark>**。

**`consumer-restriction` 插件可以根据选择的不同对象做相应的访问限制**，该插件可配置的属性如下表所示：

![Alt Image Text](../images/api1_8_6.png "body image")

> consumer restriction

其中的 type 字段是个枚举类型，可以设置以下值：

* `consumer_name`：把 Consumer 的 username 列入白名单或黑名单来限制 Consumer 对 Route 或 Service 的访问。
* `consumer_group_id`: 把 Consumer Group 的 id 列入白名单或黑名单来限制 Consumer 对 Route 或 Service 的访问。
* **`service_id`：把 Service 的 id 列入白名单或黑名单来限制 Consumer 对 Service 的访问，需要结合授权插件一起使用**。
* `route_id`：把 Route 的 id 列入白名单或黑名单来限制 Consumer 对 Route 的访问。

比如现在我们有两个 Consumer：jack1 和 jack2，这两个 Consumer 都配置了 Basic Auth 认证，配置如下所示：

> 执行命令 `kubectl port-forward --address 0.0.0.0 svc/apisix-admin 9180:9180 -n apisix` 暴露 admin 端点。

Conumer jack1 的认证配置：

```
➜ curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "username": "jack1",
    "plugins": {
        "basic-auth": {
            "username":"jack2019",
            "password": "123456"
        }
    }
}'
```

Conumer jack2 的认证配置：

```
➜ curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "username": "jack2",
    "plugins": {
        "basic-auth": {
            "username":"jack2020",
            "password": "123456"
        }
    }
}'
```


现在我们只想给一个 Route 路由对象启用 jack1 这个 Consumer 的认证配置，则除了启用 basic-auth 插件之外，还需要在 `consumer-restriction` 插件中配置一个 whitelist 白名单（当然配置黑名单也是可以的），如下所示：

```
➜ curl http://127.0.0.1:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/index.html",
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "10.244.1.125:8081": 1
        }
    },
    "plugins": {
        "basic-auth": {},
        "consumer-restriction": {
            "whitelist": [
                "jack1"
            ]
        }
    }
}'
```

然后我们使用 jack1 去访问我们的路由进行验证：

```
➜ curl -u jack2019:123456 http://127.0.0.1/index.html -i
HTTP/1.1 302 Found
...
```

正常使用 jack2 访问就会认证失败了：

```
➜ curl -u jack2020:123456 http://127.0.0.1/index.html -i
HTTP/1.1 403 Forbidden
Date: Tue, 28 Mar 2023 08:22:38 GMT
Content-Type: text/html; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.2.0

{"message":"The consumer_name is forbidden."}
```

所以当你只想让一个 Route 对象关联指定的 Consumer 的时候，记得使用 `consumer-restriction` 插件。

### jwt-auth

在平时的应用中可能使用 jwt 认证的场景是最多的，同样在 APISIX 中也有提供 jwt-auth 的插件，它同样需要与 Consumer 一起配合才能工作，我们只需要添加 JWT Auth 到一个 Service 或 Route，然后 Consumer 将其密钥添加到查询字符串参数、请求头或 cookie 中以验证其请求即可。

当然除了通过 ApisixConsumer 这个 CRD 去配置之外，我们也可以直接通过 Dashboard 页面操作。在 Dashboard 消费者页面点击创建消费者：

![Alt Image Text](../images/api1_8_7.png "body image")

点击下一步进入插件配置页面，这里我们需要启用 jwt-auth 这个插件：

![Alt Image Text](../images/api1_8_8.png "body image")

在插件配置页面配置 jwt-auth 相关属性，可参考插件文档 [https://apisix.apache.org/zh/docs/apisix/plugins/jwt-auth/](https://apisix.apache.org/zh/docs/apisix/plugins/jwt-auth/):

![Alt Image Text](../images/api1_8_9.png "body image")

可配置的属性如下表所示：

![Alt Image Text](../images/api1_8_10.png "body image")

然后提交即可创建完成 Consumer，然后我们只需要在需要的 Service 或者 Route 上开启 `jwt-auth` 即可，比如同样还是针对上面的 `nexus` 应用，我们只需要在 `ApisixRoute` 对象中启用一个 `jwt-auth` 插件即可：

```
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: nexus
  namespace: default
spec:
  http:
    - name: root
      match:
        hosts:
          - ops.foobar.com
        paths:
          - "/nexus*"
          - "/static/*"
          - "/service/*"
      plugins:
        - name: redirect
          enable: true
          config:
            http_to_https: true
        - name: redirect
          enable: true
          config:
            regex_uri: ["^(/nexus)$", "$1/"]
        - name: proxy-rewrite
          enable: true
          config:
            regex_uri: ["^/nexus(/|$)(.*)", "/$2"]
      backends:
        - serviceName: nexus
          servicePort: 8081
      authentication: # 开启 jwt auth 认证
        enable: true
        type: jwtAuth
```

重新更新上面的对象后我们同样来测试验证下：

```
➜ curl -i http://test.foobar.com/nexus/
HTTP/1.1 401 Unauthorized
Date: Tue, 28 Mar 2023 08:30:02 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.2.0

{"message":"Missing JWT token in request"}
```

要正常访问我们的服务就需要先进行登录获取 jwt-auth 的 token，**首先，你需要为签发 token 的 API 配置一个 Route，该路由将使用 public-api 插件**。

```
➜ curl http://127.0.0.1:9180/apisix/admin/routes/jas \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/apisix/plugin/jwt/sign",
    "plugins": {
        "public-api": {}
    }
}'
{"error_msg":"unknown plugin [public-api]"}
```

执行上面命令后会出现不识别 `public-api` 插件，这是因为我们这里使用 Helm Chart 方式安装的 APISIX 默认没有安装该插件，所以我们需要到 Helm Chart 的 values.yaml 文件中在 plugins 属性下面添加上该插件，然后重新更新 APISIX 即可：

```
➜ helm upgrade --install apisix ./apisix -f ./apisix-values.yaml -n apisix --create-namespace
```

更新后重新执行命令：

```

➜ curl http://127.0.0.1:9180/apisix/admin/routes/jas -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/apisix/plugin/jwt/sign",
    "plugins": {
        "public-api": {}
    }
}'
```

之后就可以通过调用它来获取 token 了。

```
➜ curl http://127.0.0.1/apisix/plugin/jwt/sign?key=user-key -i
HTTP/1.1 200 OK
Date: Tue, 28 Mar 2023 08:44:45 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.2.0

eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2ODAwNzk0ODUsImtleSI6InVzZXIta2V5In0.n4o_w3AgNC6C1pujEUScSBe0Mzw5vbjIzKpQpbrBhO8
```

要注意上面我们在获取 token 的时候需要传递创建消费者的标识 key，因为可能有多个不同的 Consumer 消费者，然后我们将上面获得的 token 放入到 Header 头中进行访问：

```
➜ curl -i http://test.foobar.com/nexus/ -H 'Authorization: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2ODAwNzk0ODUsImtleSI6InVzZXIta2V5In0.n4o_w3AgNC6C1pujEUScSBe0Mzw5vbjIzKpQpbrBhO8'
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 9024
Connection: keep-alive
Date: Tue, 28 Mar 2023 08:45:24 GMT
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Last-Modified: Tue, 28 Mar 2023 08:45:24 GMT
Pragma: no-cache
Cache-Control: no-cache, no-store, max-age=0, must-revalidate, post-check=0, pre-check=0
Expires: 0
Server: APISIX/3.2.0


<!DOCTYPE html>
<html lang="en">
......
```

可以看到可以正常访问。同样也可以放到请求参数中验证：

```
➜ curl -i http://test.foobar.com/nexus/?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2ODAwNzk0ODUsImtleSI6InVzZXIta2V5In0.n4o_w3AgNC6C1pujEUScSBe0Mzw5vbjIzKpQpbrBhO8
HTTP/1.1 200 OK
......
```

此外还可以放到 cookie 中进行验证：

```
➜ curl -i http://test.foobar.com/nexus/ --cookie jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2ODAwNzk0ODUsImtleSI6InVzZXIta2V5In0.n4o_w3AgNC6C1pujEUScSBe0Mzw5vbjIzKpQpbrBhO8
HTTP/1.1 200 OK
......
```

### 自定义插件

除了 APISIX 官方内置的插件之外，我们也可以根据自己的需求去自定义插件，要自定义插件需要使用到 APISIX 提供的 Runner，目前已经支持 Java、Go、Node 和 Python 语言的 Runner，**这个 Runner 相当于是 APISIX 和自定义插件之间的桥梁，比如 `apache-apisix-python-runner` 这个项目通过 Python Runner 可以把 Python 直接应用到 APISIX 的插件开发中，整体架构如下所示**：

![Alt Image Text](../images/api1_8_11.jpeg "body image")


左边是 APISIX 的工作流程，右边的 Plugin Runner 是各语言的插件运行器，当在 APISIX 中配置一个 Plugin Runner 时，APISIX 会启动一个子进程运行 Plugin Runner，该子进程与 APISIX 进程属于同一个用户，当我们重启或重新加载 APISIX 时，Plugin Runner 也将被重启。

如果你为一个给定的路由配置了 `ext-plugin-*` 插件，请求命中该路由时将触发 APISIX 通过 `Unix Socket` 向 `Plugin Runner` 发起 `RPC` 调用。调用分为两个阶段：

* ext-plugin-pre-req：在执行 APISIX 内置插件之前
* ext-plugin-post-req：在执行 APISIX 内置插件之后
* ext-plugin-post-resp：将在请求获取到上游的响应之后执行。

接下来我们就以 Python 为例来说明如何自定义插件，首先获取 apache-apisix-python-runner 项目

```
➜ git clone https://github.com/apache/apisix-python-plugin-runner.git
➜ cd apisix-python-plugin-runner
➜ git checkout 0.2.0  # 切换到0.2.0版本
```

如果是开发模式，则我们可以直接使用下面的命令启动 Python Runner：

```
➜ APISIX_LISTEN_ADDRESS=unix:/tmp/runner.sock python3 bin/py-runner start
```

启动后需要在 APISIX 配置文件中新增外部插件配置，如下所示：

```
➜ vim /path/to/apisix/conf/config.yaml
apisix:
  admin_key:
    - name: "admin"
      key: edd1c9f034335f136f87ad84b625c8f1
      role: admin

ext-plugin:
  path_for_test: /tmp/runner.sock
```

通过 `ext-plugin.path_for_test` 指定 Python Runner 的 unix socket 文件路径即可，如果是生产环境则可以通过 `ext-plugin.cmd` 来指定 Runner 的启动命令即可：

```
ext-plugin:
  cmd: [ "python3", "/path/to/apisix-python-plugin-runner/apisix/bin/py-runner", "start" ]
```

我们这里的 APISIX 是运行 Kubernetes 集群中的，所以要在 APISIX 的 Pod 中去执行 Python Runner 的代码，我们自然需要将我们的 Python 代码放到 APISIX 的容器中去，然后安装自定义插件的相关依赖，直接在 APISIX 配置文件中添加上面的配置即可，所以我们这里基于 APISIX 的镜像来重新定制包含插件的镜像，在 apisix-python-plugin-runner 项目根目录下新增如下所示的 Dockerfile 文件：

```
ROM apache/apisix:3.2.0-debian

WORKDIR /apisix-python
ADD . /apisix-python
USER root

RUN apt-get update && \
    apt-get install -y python3 python3-pip make && \
    rm -rf /var/lib/apt/lists/* && apt-get clean && \
    make setup && make install
```

基于上面 Dockerfile 构建一个新的镜像，推送到 Docker Hub：

```
➜ docker build -t cnych/apisix:py3-plugin-3.2.0-debian .
# 推送到DockerHub
➜ docker push cnych/apisix:py3-plugin-3.2.0-debian
```

接下来我们需要使用上面构建的镜像来安装 APISIX，我们这里使用的是 Helm Chart 进行安装的，所以需要通过 Values 文件进行覆盖，如下所示：

```
# ci/prod.yaml
apisix:
  enabled: true

  image:
    repository: cnych/apisix
    tag: py3-plugin-3.2.0-debian
......

extPlugin:
  # -- Enable External Plugins. See [external plugin](https://apisix.apache.org/docs/apisix/next/external-plugin/)
  enabled: true
  # -- the command and its arguements to run as a subprocess
  cmd: ["python3", "/apisix-python/bin/py-runner", "start"]
```

注意这里需要将自定义插件开启，并且将 `extPlugin.cmd` 配置为 `["/apisix-python/bin/py-runner", "start"]`，因为我们是在 APISIX 的镜像中安装了 Python Runner，所以需要指定 Python Runner 的启动命令。

接着就可以重新部署 APISIX 了：

```
➜ helm upgrade --install apisix ./apisix -f ./apisix-values.yaml -n apisix --create-namespace
```

![Alt Image Text](../images/api1_8_12.png "body image")

在插件目录 `/apisix-python/apisix/plugins` 中的 `.py` 文件都会被自动加载，上面示例中有两个插件 `stop.py` 和 `rewrite.py`，我们以 `stop.py` 为例进行说明，该插件代码如下所示：

```
from typing import Any
from apisix.runner.http.request import Request
from apisix.runner.http.response import Response
from apisix.runner.plugin.core import PluginBase


class Stop(PluginBase):

    def name(self) -> str:
        """在runner中注册的插件的名称"""
        return "stop"

    def config(self, conf: Any) -> Any:
        """解析插件配置"""
        return conf

    def filter(self, conf: Any, request: Request, response: Response):
        """插件执行的主函数
        :param conf:
            解析后的插件配置
        :param request:
            请求参数和信息
        :param response:
            响应参数和信息
        :return:
        """

        # 打印插件配置
        print(conf)

        # 获取请求 nginx 变量 `host`
        host = request.get_var("host")
        print(host)

        # 获取请求体
        body = request.get_body()
        print(body)

        # 设置响应头
        response.set_header("X-Resp-A6-Runner", "Python")

        # 设置响应体
        response.set_body("Hello, Python Runner of APISIX")

        # 设置响应状态码
        response.set_status_code(201)
```

实现插件首先必须要继承 PluginBase 类，必须实现 filter 函数，插件执行核心业务逻辑就是在 filter 函数中，该函数只包含 Request 和 Response 类对象作为参数，Request 对象参数可以获取请求信息，Response 对象参数可以设置响应信息，conf 可以获取插件配置信息。

然后我们在前面的 Nexus 应用中新增一个路由来测试我们上面的 stop 插件，在 ApisixRoute 对象中新增一个路由规则，如下所示：

```
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: nexus
  namespace: default
spec:
  http:
    - name: ext
      match:
        hosts:
          - test.foobar.com
        paths:
          - "/extPlugin"
      plugins:
        - name: ext-plugin-pre-req # 启用ext-plugin-pre-req插件
          enable: true
          config:
            conf:
              - name: "stop" # 使用 stop 这个自定义插件
                value: '{"body":"hello"}'
      backends:
        - serviceName: nexus
          servicePort: 8081
```

直接创建上面的路由即可，核心配置是启用 `ext-plugin-pre-req` 插件（前提是在配置文件中已经启用该插件，在 Helm Chart 的 Values 中添加上），然后在 config 下面使用 conf 属性进行配置，conf 为数组格式可以同时设置多个插件，插件配置对象中 name 为插件名称，该名称需要与插件代码文件和对象名称一致，value 为插件配置，可以为 JSON 字符串。

创建后同样在 Dashboard 中也可以看到 APISIX 中的路由配置格式：

![Alt Image Text](../images/api1_8_13.png "body image")

接着我们可以来访问 http://ops.youdianzhishi.com/extPlugin 这个路径来验证我们的自定义插件：

```
➜ curl -i http://test.foobar.com/extPlugin
HTTP/1.1 201 Created
Date: Tue, 28 Mar 2023 13:10:34 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
x-resp-a6-runner: Python
Server: APISIX/3.2.0

Hello, Python Runner of APISIX
```

可以看到得到的结果和响应码和我们在插件中的定义是符合的。到这里就完成了使用 Python 进行 APISIX 自定义插件，我们有任何的业务逻辑需要处理直接去定义一个对应的插件即可。

