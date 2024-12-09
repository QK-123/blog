---
layout: doc
title: docker 网络
---
## Docker 网络<Badge type="tip" text="docker" />

网络架构越来越复杂，但只要抓住关键点，就可以在混乱中保持清晰的思考。

1. 端点之间的连通性是关键
2. 其余的复杂度都是被附加性功能所引入，例如安全，管理的便捷性等。

四种网络模式对比：

| 网络模式  | 特点                   | 应用场景                     | 优劣               |
| --------- | ---------------------- | ---------------------------- | ------------------ |
| bridge    | 默认模式，隔离网络     | 多个容器通信，与外部网络隔离 | 简单易用，隔离性好 |
| host      | 共享宿主机的网络栈     | 访问宿主机的硬件或服务       | 性能高，配置简单   |
| container | 共享另一个容器的网络栈 | 代理、网关                   | 灵活配置           |
| none      | 不配置网络接口         | 自定义网络                   | 完全自定义         |

### 外部访问容器

```bash
-p hostPort:containerPort
# 例子 将通过主机的80端口
ocker run -d -p 80:80 nginx:alpine
```

### 容器之间互联

通过创建bridge模式的网络来实现

```bash
docker network create -d bridge my-net
```

运行两个容器来测试互联性

```bash
docker run -it --rm --name busybox1 --network my-net busybox sh

# 打开一个新终端
docker run -it --rm --name busybox2 --network my-net busybox sh
```

![image-20240729115639045](/k8s/image-20240729115639045.png)

### 配置容器DNS

容器默认继承主机的`/etc/resolv.conf` 文件中的配置，主机中的配置变动会立即刷新到容器中。

### Docker 底层网络原理

docker 启动后会创建一个名为docker0的虚拟网桥，通过veth pair将容器网络连接到docker0

![image-20240729122135689](/k8s/image-20240729122135689.png)

### 端口映射的实现

端口映射是通过主机的iptables实现的

#### 容器访问外部

容器访问外部需要进行SNAT转换  执行 `iptables -t nat -nL`

![image-20240729130540713](/k8s/image-20240729130540713.png)

#### 外部访问容器

外部访问容器需要进行DNAT转换  执行 `iptables -t nat -nL`

![image-20240729130620239](/k8s/image-20240729130620239.png)
