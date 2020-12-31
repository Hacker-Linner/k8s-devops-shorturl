# ShortUrl

Go-Zero 官方短链项目教程：[快速构建高并发微服务](https://github.com/tal-tech/zero-doc/blob/main/doc/shorturl.md)

关于 [go-zero](https://github.com/tal-tech/go-zero)，大家可以看[文档](https://www.yuque.com/tal-tech/go-zero)。为少认为它是中国目前最好用的 golang 微服务框架。

## 准备工作

我这里直接在 K8S 开发集群中部署相关实例。

生产求稳，建议大家还是买云数据库服务。

### 部署 Mysql、Redis、Etcd。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b7e1b0c86d24c8a99cb79e6cfb8c8af~tplv-k3u1fbpfcp-watermark.image)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/017f64738ed14216a8be50616de75ea3~tplv-k3u1fbpfcp-watermark.image)

### 部署 Drone、Drone-Runner-Kube

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/971674fb36f941c785866c74d01475d2~tplv-k3u1fbpfcp-watermark.image)

## 开始探索

### 准备 DevOps 部署相关配置

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e32bcc3d0954c1fa68f2c21ad57a06e~tplv-k3u1fbpfcp-watermark.image)

### Dockerfile.alpine.base

```Dockerfile
FROM alpine:3.12

RUN addgroup -S app \
    && adduser -S -g app app \
    && apk --no-cache add \
    ca-certificates curl netcat-openbsd
```

### Dockerfile.base

```Dockerfile
FROM golang:1.15-alpine

RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct
RUN mkdir -p /shorturl/

WORKDIR /shorturl
COPY go.mod go.mod
RUN go mod download
```

### Dockerfile.prod.rpc

```Dockerfile
### shorturl:base
FROM hub.your-domain.com/library/shorturl:base as builder

WORKDIR /shorturl-rpc
COPY . .
RUN CGO_ENABLED=0 go build -a -o bin/shorturl-rpc rpc/transform/*.go

### shorturl-alpine:base
FROM hub.your-domain.com/library/shorturl-alpine:base

LABEL maintainer="为少"
WORKDIR /home/app
COPY --from=builder /shorturl-rpc/bin/shorturl-rpc .
COPY ./rpc/transform/etc ./rpc/transform/etc
RUN chown -R app:app ./
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
USER app
CMD ["./shorturl-rpc", "-f", "rpc/transform/etc/transform.yaml"]
```

### Dockerfile.prod.api

```Dockerfile
### shorturl:base
FROM hub.your-domain.com/library/shorturl:base as builder

WORKDIR /shorturl-api
COPY . .
RUN CGO_ENABLED=0 go build -a -o bin/shorturl-api api/*.go

### shorturl-alpine:base
FROM hub.your-domain.com/library/shorturl-alpine:base

LABEL maintainer="为少"
WORKDIR /home/app
COPY --from=builder /shorturl-api/bin/shorturl-api .
COPY ./api/etc ./api/etc
RUN chown -R app:app ./
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
USER app
CMD ["./shorturl-api", "-f", "api/etc/shorturl-api.yaml"]
```

### shorturl-api-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shorturl-api
data:
  shorturl-api.yaml: |-
    Name: shorturl-api
    Host: 0.0.0.0
    Port: 8888
    Transform:
      Etcd:
        Hosts:
          - your-ip:2379
        Key: transform.rpc
```

### shorturl-rpc-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shorturl-transform-rpc
data:
  transform.yaml: |-
    Name: transform.rpc
    Log:
      Mode: console
    ListenOn: 0.0.0.0:8081
    Etcd:
      Hosts:
      - your-ip:2379
      Key: transform.rpc
    DataSource: root:123456@tcp(your-ip:3306)/shorturl?charset=utf8mb4&parseTime=true&loc=Asia%2FShanghai
    Table: shorturl
    Cache:
      - Host: your-ip:6379
```

### .drone.yml

```yaml
kind: pipeline
type: kubernetes
name: ShortUrl(transform.rpc)

steps:

  - name: 更新 Charts(transform.rpc)
    image: busybox
    commands:
      - echo $DRONE_COMMIT
      - '[ -n "$DRONE_COMMIT" ] && (
          sed -i "s/APP_VERSION/${DRONE_COMMIT}/g" k8s-devops/helm-shorturl/shorturl/Chart.yaml;
          sed -i "s/SHORTURL/shorturl-transform-rpc/g" k8s-devops/helm-shorturl/shorturl/Chart.yaml;
          sed -i "s/IMAGE/shorturl-transform-rpc/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/APICONFIGMAP//g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/RPCCONFIGMAP/shorturl-transform-rpc/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/CONTAINERPORT/8081/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/INGRESS/false/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/8080/8081/g" k8s-devops/helm-shorturl/shorturl/templates/NOTES.txt;
        )'
      - cat k8s-devops/helm-shorturl/shorturl/Chart.yaml
      - cat k8s-devops/helm-shorturl/shorturl/values.yaml

  - name: 构建 transform.rpc Image
    image: plugins/docker
    settings:
      debug: true
      dockerfile: Dockerfile.prod.rpc
      repo: hub.your-domain.com/library/shorturl-transform-rpc
      tags: ${DRONE_COMMIT}
      registry: hub.your-domain.com
      username:
        from_secret: docker_user
      password:
        from_secret: docker_pass

  - name: 上云 HelmV3(transform.rpc) -> K8S
    image: pelotech/drone-helm3
    settings:
      helm_command: upgrade
      chart: ./k8s-devops/helm-shorturl/shorturl
      release: shorturl-transform-rpc
      namespace: shorturl
      api_server:
        from_secret: api_server
      kubernetes_token:
        from_secret: k8s_token
      skip_tls_verify: true

trigger:
  branch:
    - main
---
kind: pipeline
type: kubernetes
name: ShortUrl(shorturl-api)

steps:

  - name: 更新 Charts(shorturl-api)
    image: busybox
    commands:
      - echo $DRONE_COMMIT
      - '[ -n "$DRONE_COMMIT" ] && (
          sed -i "s/APP_VERSION/${DRONE_COMMIT}/g" k8s-devops/helm-shorturl/shorturl/Chart.yaml;
          sed -i "s/SHORTURL/shorturl-api/g" k8s-devops/helm-shorturl/shorturl/Chart.yaml;
          sed -i "s/IMAGE/shorturl-api/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/APICONFIGMAP/shorturl-api/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/RPCCONFIGMAP//g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/CONTAINERPORT/8888/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/INGRESS/true/g" k8s-devops/helm-shorturl/shorturl/values.yaml;
          sed -i "s/8080/8888/g" k8s-devops/helm-shorturl/shorturl/templates/NOTES.txt;
        )'
      - cat k8s-devops/helm-shorturl/shorturl/Chart.yaml
      - cat k8s-devops/helm-shorturl/shorturl/values.yaml

  - name: 构建 shorturl-api Image
    image: plugins/docker
    settings:
      debug: true
      dockerfile: Dockerfile.prod.api
      repo: hub.your-domain.com/library/shorturl-api
      tags: ${DRONE_COMMIT}
      registry: hub.your-domain.com
      username:
        from_secret: docker_user
      password:
        from_secret: docker_pass

  - name: 上云 HelmV3(shorturl-api) -> K8S
    image: pelotech/drone-helm3
    settings:
      helm_command: upgrade
      chart: ./k8s-devops/helm-shorturl/shorturl
      release: shorturl-api
      namespace: shorturl
      api_server:
        from_secret: api_server
      kubernetes_token:
        from_secret: k8s_token
      skip_tls_verify: true

trigger:
  branch:
    - main
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/189847076563479fb7f449fa9f218c06~tplv-k3u1fbpfcp-watermark.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cabd13ef91545d6a1d6300fadee5ebc~tplv-k3u1fbpfcp-watermark.image)

### 验证

我这里已经部署好了一个开发测试的 Pod 实例，大家可以试用。

shorten api 调用

```sh
# curl -i "https://shorturl.your-domain.com/shorten?url=https://www.your-domain.com"
curl -i "https://shorturl.hacker-linner.com/shorten?url=https://www.hacker-linner.com"
```

expand api 调用

```sh
# curl -i "https://shorturl.your-domain.com/expand?shorten=6d11a1"
curl -i "https://shorturl.hacker-linner.com/expand?shorten=6d11a1"
```

### 利用工具查看 Redis 与 Etcd 中的键值对

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d39c0593bc304be5ba31abf22db3fe2c~tplv-k3u1fbpfcp-watermark.image)

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3b6614610f6478fad4e4c3860a36fcf~tplv-k3u1fbpfcp-watermark.image)

我是为少。

微信：uuhells123。

公众号：黑客下午茶。

谢谢点赞支持👍👍👍！


