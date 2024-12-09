### IPTABLES 介绍 <Badge type="tip" text="linux" />

`iptables` 是一个用于配置 Linux 内核防火墙的工具。它通过操作 Linux 内核中的 **netfilter** 模块来过滤网络数据包，并为系统管理员提供了管理和控制网络流量的能力。

#### iptables 表的类型有

具有相同功能的规则的集合称为表。我们所创建的规则都被以下四种分类所包含。

- **filter:** 过滤数据包，决定是否允许数据包通过。
- **nat:** 网络地址转换，用于将内部 IP 地址转换为外部 IP 地址。
- **mangle:** 修改数据包，例如修改数据包的头部信息。
- **raw:** 决定数据包是否被状态跟踪机制处理。

<img src="https://phoenixnap.com/kb/wp-content/uploads/2021/04/iptables-diagram.png" alt="Iptables Tutorial: Ultimate Guide to Linux Firewall"  />

![image-20241016093816533](/image-20241016093816533.png)

![](/1_Vs4XnYTCI4fXYuGl2V3xfw.png)

#### 表和链之间的规则如下：

1. **表：filter**（默认表，用于数据包过滤）
   - 链：`INPUT`    （处理进入本机的数据包）
   - 链：`FORWARD`  （处理转发通过本机的数据包）
   - 链：`OUTPUT`   （处理本机发送的数据包）

2. **表：nat**（用于网络地址转换）
   - 链：`PREROUTING`  （在数据包路由之前处理，常用于改变目标地址）
   - 链：`INPUT`       （处理进入本机的 NAT 数据包）
   - 链：`OUTPUT`      （处理本机产生的 NAT 数据包）
   - 链：`POSTROUTING` （在数据包路由之后处理，常用于改变源地址）

3. **表：mangle**（用于修改数据包）
   - 链：`PREROUTING`  （在数据包路由之前进行处理）
   - 链：`INPUT`       （处理进入本机的数据包）
   - 链：`FORWARD`     （处理转发的数据包）
   - 链：`OUTPUT`      （处理本机发出的数据包）
   - 链：`POSTROUTING` （在数据包路由之后处理）

4. **表：raw**（用于设置连接跟踪标志）
   - 链：`PREROUTING`  （在数据包路由之前处理，不使用连接跟踪）
   - 链：`OUTPUT`      （在本机产生数据包时处理，不使用连接跟踪）

#### 表的优先级顺序

1. **raw** （最先处理）
2. **mangle**
3. **nat**
4. **filter** （最后处理）

#### 表链结合处理流程

![image-20241016100537502](/image-20241016100537502.png)

#### 规则介绍

规则是根据指定的匹配条件来尝试匹配每个流经此处的报文，一旦匹配成功，则由规则后面指定的处理动作进行处理。

##### 匹配条件

| 匹配条件            | 选项及描述                                           | 示例                                       |
|---------------------|----------------------------------------------------|--------------------------------------------|
| **协议匹配**         | `-p` 或 `--protocol`：匹配特定协议（如 `tcp`、`udp`、`icmp`） | `-p tcp`                                   |
| **源地址匹配**       | `-s` 或 `--source`：匹配数据包的源 IP 地址            | `-s 192.168.1.1` <br> `-s 192.168.1.0/24`  |
| **目的地址匹配**     | `-d` 或 `--destination`：匹配数据包的目的 IP 地址      | `-d 10.0.0.1`                              |
| **输入网络接口**     | `-i` 或 `--in-interface`：匹配进入的网络接口          | `-i eth0`                                  |
| **输出网络接口**     | `-o` 或 `--out-interface`：匹配发出的网络接口          | `-o eth1`                                  |
| **源端口匹配**       | `--sport` 或 `--source-port`：匹配 TCP/UDP 源端口      | `--sport 22` <br> `--sport 1024:65535`     |
| **目的端口匹配**     | `--dport` 或 `--destination-port`：匹配 TCP/UDP 目标端口 | `--dport 80`                               |
| **连接状态匹配**     | `-m state --state`：匹配数据包的连接状态              | `-m state --state NEW`                     |
| **ICMP 类型匹配**    | `--icmp-type`：匹配 ICMP 包类型                       | `--icmp-type echo-request`                 |
| **反向匹配**         | `!`：进行反向匹配                                    | `! -s 192.168.1.0/24`                      |

##### 匹配动作

| 处理动作            | 描述                                                                 | 示例                                         |
|---------------------|----------------------------------------------------------------------|----------------------------------------------|
| **ACCEPT**          | 允许数据包通过防火墙。                                               | `-j ACCEPT`                                  |
| **DROP**            | 丢弃数据包，不通知发送方。                                           | `-j DROP`                                    |
| **REJECT**          | 拒绝数据包并向发送方返回错误信息。                                   | `-j REJECT --reject-with icmp-port-unreachable` |
| **LOG**             | 记录数据包信息到系统日志中（通常是 `/var/log/syslog` 或 `/var/log/messages`）。 | `-j LOG --log-prefix "iptables log: "`       |
| **SNAT**            | 修改数据包的源地址（用于源网络地址转换，通常在 `POSTROUTING` 链中）。   | `-j SNAT --to-source 192.168.1.100`          |
| **DNAT**            | 修改数据包的目标地址（用于目的网络地址转换，通常在 `PREROUTING` 链中）。 | `-j DNAT --to-destination 192.168.1.200`     |
| **MASQUERADE**      | 类似于 SNAT，但动态修改源地址（适用于动态 IP 地址，如 PPP 连接）。     | `-j MASQUERADE`                              |
| **REDIRECT**        | 将数据包重定向到本地主机的特定端口（通常用于透明代理）。               | `-j REDIRECT --to-port 8080`                 |
| **MARK**            | 给数据包打上特定标记（通常用于流量控制）。                           | `-j MARK --set-mark 1`                       |
| **RETURN**          | 停止当前链中的规则匹配，并返回到调用链继续处理。                       | `-j RETURN`                                  |
| **QUEUE**           | 将数据包传递给用户空间程序进行处理。                                  | `-j QUEUE`                                   |
| **SNAP**            | 捕获数据包（与特定内核扩展一起使用）。                               | `-j SNAP`                                    |

#### filter 表规则介绍

filter负责过滤功能，比如允许哪些IP地址访问，拒绝哪些IP地址访问，允许访问哪些端口，禁止访问哪些端口，filter表会根据我们定义的规则进行过滤，filter表应该是我们最常用到的表。

#### CentOS 7 中操作 `iptables filter` 表的常用命令

##### 1. 查看规则
查看当前 `filter` 表中所有链的规则。使用 `-L` 列出规则，`-v` 显示详细信息，`-n` 禁用 DNS 解析。

```bash
sudo iptables -L -v -n
sudo iptables -t filter -L  # 查看指定表
```
##### 2.增加规则
向 `filter` 表的链中添加新规则，允许 TCP 80 端口的 HTTP 访问。

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```
##### 3.删除规则
删除 `filter` 表中指定的规则，可以通过规则编号删除。
查看链中的规则编号：
```bash
sudo iptables -L INPUT --line-numbers
```
删除特定规则（假设删除规则编号为 3）：
```bash
sudo iptables -D INPUT 3
```
##### 4.修改规则
修改特定规则需要先删除旧规则，然后添加新的规则。例如，修改 HTTP 端口为 8080：

```bash
# 删除旧的允许 80 端口的规则
sudo iptables -D INPUT -p tcp --dport 80 -j ACCEPT

# 添加新的允许 8080 端口的规则
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```
##### 5.保存规则
保存当前规则，以便系统重启后规则仍然生效。CentOS 7 默认使用 iptables-services 进行规则的保存。

```bash
sudo service iptables save
```
##### 6.查看详细规则信息
显示更加详细的 `iptables` 规则信息，包括字节、数据包计数等统计信息。

```bash
sudo iptables -L -v --line-numbers
```
##### 7.恢复规则
恢复之前保存的规则。
```bash
sudo service iptables restart
```

#### 端口转发

```bash
# 将80流量转发到8080
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
```

#### SNAT DNAT 结构解析

```bash
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 10.0.0.1
```

![](/snat.png)

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80
```

![](/dnat.png)

#### iptables docker使用案例

| 操作类型             | 说明                                                                 | `iptables` 规则示例                                      | Docker 操作示例                                     |
|----------------------|----------------------------------------------------------------------|----------------------------------------------------------|-----------------------------------------------------|
| **创建自定义链**     | Docker 启动时创建自定义链处理容器网络流量。                           | `-N DOCKER` <br> `-N DOCKER-USER`                        | Docker 自动创建，用户无需手动操作                   |
| **端口映射 (DNAT)**  | 将主机端口映射到容器端口，通过 DNAT 实现。                           | `-A PREROUTING -p tcp --dport 8080 -j DNAT --to 172.17.0.2:80` | `docker run -d -p 8080:80 nginx`                   |
| **源地址转换 (SNAT)**| 将容器的源地址转换为主机的地址，便于容器访问外部网络。               | `-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE` | Docker 自动处理出站流量的地址转换                  |
| **FORWARD 链规则**   | 控制容器之间和容器与外部网络之间的流量转发。                         | `-A FORWARD -i docker0 -o docker0 -j ACCEPT`             | Docker 自动允许容器之间和容器与外部的通信           |
| **阻止流量**         | 在 `DOCKER-USER` 链中自定义规则，阻止某些 IP 或网络访问容器。         | `-I DOCKER-USER -s 192.168.1.100 -j DROP`                | `sudo iptables -I DOCKER-USER -s 192.168.1.100 -j DROP` |
| **禁用自动规则**     | 禁用 Docker 自动修改 `iptables` 规则。                              | 无规则                                                     | `dockerd --iptables=false`                          |
| **删除规则**         | 手动删除 Docker 添加的 `iptables` 规则。                            | `-D PREROUTING -p tcp --dport 8080 -j DNAT --to 172.17.0.2:80` | Docker 停止或删除容器时自动删除相关 `iptables` 规则 |

---

##### 示例说明：

1. **端口映射**：
   - Docker 通过 `-p` 参数映射主机端口到容器内部端口，`iptables` 使用 `DNAT` 规则来实现。
   - 例子：`docker run -d -p 8080:80 nginx` 将主机的 8080 端口流量转发到容器的 80 端口。

2. **源地址转换 (SNAT)**：
   - Docker 通过 `iptables` 使用 `MASQUERADE` 规则，将容器内部的私有 IP 地址转换为主机 IP，以便容器访问外部网络。
   - 这通常是 Docker 自动处理的，用户无需手动干预。

3. **阻止特定 IP 的流量**：
   - 用户可以在 `DOCKER-USER` 链中添加自定义规则，例如阻止来自特定 IP 的流量。
   - 例子：`sudo iptables -I DOCKER-USER -s 192.168.1.100 -j DROP`，该规则阻止来自 IP 地址 `192.168.1.100` 的所有流量访问 Docker 容器。

4. **禁用自动 `iptables` 规则**：
   - 可以通过启动 Docker 守护进程时禁用 `iptables` 自动规则管理，使用 `dockerd --iptables=false`，这样用户可以完全手动管理 `iptables`。

