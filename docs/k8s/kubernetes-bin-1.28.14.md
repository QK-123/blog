## Kubernetes 1.28高可用集群二进制部署（基于containerd）<Badge type="tip" text="k8s" />

### 一，集群环境准备

#### 1.1 主机规划

| IP            | 主机名   | 主机配置 | 主机角色 | 软件列表                                                     |
| ------------- | -------- | -------- | -------- | ------------------------------------------------------------ |
| 172.19.219.49 | master01 | 2C8G     | master   | kube-apiserver，etcd，kube-scheduler，kube-controller-manager，kubelet，kube-proxy |
| 172.19.219.50 | master02 | 2C8G     | master   | kube-apiserver，etcd，kube-scheduler，kube-controller-manager，kubelet，kube-proxy |
| 172.19.219.51 | master03 | 2C8G     | master   | kube-apiserver，etcd，kube-scheduler，kube-controller-manager，kubelet，kube-proxy |
| 172.19.219.51 | worker01 | 2C8G     | worker   | kubelet，kube-proxy                                          |

#### 1.2 软件版本

| 软件       | 版本            | 备注 |
| ---------- | --------------- | ---- |
| Centos7u9  | CentOS 7.9 64位 |      |
| kubernetes | v1.28.14        |      |
| etcd       | 3.5.6+          |      |
| containerd | 1.7.0+, 1.6.15+ |      |
| calico     | v3.28           |      |
| coredns    | v1.10.1         |      |

#### 1.3 网络分配

| 类型         | 网段            | 备注 |
| ------------ | --------------- | ---- |
| Node 网络    | 172.19.208.0/20 |      |
| Pod 网络     | 10.244.0.0/16   |      |
| Service 网络 | 10.96.0.0/16    |      |

### 二，集群部署

#### 2.1 主机准备

##### 2.1.1 主机名设置

```bash
hostnamectl set-hostname master01
hostnamectl set-hostname master02
hostnamectl set-hostname master03
hostnamectl set-hostname worker01
```

##### 2.1.2 主机名与IP地址解析

```bash
cat <<EOF >> /etc/hosts
172.19.219.52 worker01
172.19.219.50 master02
172.19.219.49 master01
172.19.219.51 master03
EOF
```

##### 2.1.3 主机安全设置

###### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
firewall-cmd --state
```

###### 关闭selinux

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sestatus
```

##### 2.1.4 交换分区设置

```bash
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
echo "vm.swappiness = 0" >> /etc/sysctl.conf
sysctl -p
```

##### 2.1.5 主机系统时间同步

```bash
yum install -y ntpdate
crontab -l
# 配置时间同步 */5 * * * * /usr/sbin/ntpdate time.aliyun.com 使用命令实现
echo "*/5 * * * * /usr/sbin/ntpdate time.aliyun.com" >> /var/spool/cron/root
crontab -l
```

##### 2.1.6 主机系统优化

```bash
ulimit -SHn 65535

cat <<EOF >> /etc/security/limits.conf
* soft nofile 655350
* hard nofile 655350
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

##### 2.1.7 ipvs管理工具和模块加载

###### 安装相关工具

```bash
yum install -y ipset ipvsadm sysstat conntrack libseccomp
```

###### 配置模块开启自加载

```bash
cat > /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

systemctl enable --now systemd-modules-load.service
```

##### 2.1.8 Linux 内核优化

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
#开启 IP 转发，允许服务器作为路由器转发流量，这是 Kubernetes 需要的功能，以便不同节点间进行流量转发。
net.ipv4.ip_forward = 1
# 允许桥接流量经过 iptables 规则检查，通常在容器和虚拟机的网络设置中使用，以确保网络包经过防火墙规则的处理。
net.bridge.bridge-nf-call-iptables = 1
# 与 bridge-nf-call-iptables 类似，针对 IPv6 流量。
net.bridge.bridge-nf-call-ip6tables = 1
# 允许卸载被占用的挂载点，优化文件系统的操作。
fs.may_detach_mounts = 1
# 允许内存过度分配，有利于性能优化，但需要注意内存不足的情况。
vm.overcommit_memory = 1
# 当系统内存不足时不触发内核 panic，避免因内存溢出（OOM）导致系统崩溃。
vm.panic_on_oom = 0
# 设置 inotify 实时文件监控的最大监听文件数，这在处理大量文件操作的场景中很重要。
fs.ionotify.max_user_watches = 89100
# 系统可同时打开的最大文件数，适用于高并发的文件操作系统。
fs.file-max = 52706963
# 系统最大打开文件描述符数，配合 fs.file-max 提高系统处理能力。
fs.nr_open = 52706963
# 调整 Netfilter 连接跟踪表的最大条目数，适合处理大量并发连接的网络环境。
net.netfilter.nf_conntrack_max = 2310720

# 优化网络参数
# 设置 TCP 连接的保活时间，表示在 600 秒（10 分钟）内保持空闲状态的 TCP 连接将发送 keepalive 包检测对方是否在线。
net.ipv4.tcp_keepalive_time = 600
# 每隔 15 秒发送一次 keepalive 包，用于检测 TCP 连接的状态。
net.ipv4.tcp_keepalive_intvl = 15
# 保活探测的次数，超过 3 次未响应则认为连接已断开。
net.ipv4.tcp_keepalive_probes = 3
# 设置 TIME-WAIT 状态的最大数量，避免过多的 TIME-WAIT 连接占用系统资源。
net.ipv4.tcp_max_tw_buckets = 36000
# 开启 TIME-WAIT 状态的快速重用，避免过多的 TIME-WAIT 连接占用系统资源。
net.ipv4.tcp_tw_reuse = 1
# 系统中无主（orphan）TCP 连接的最大数量，如果超过这个数，系统将清除部分连接。
net.ipv4.tcp_max_orphans = 327680
# 在放弃 TCP 连接前最多进行 3 次重试，以防止网络抖动引起的短暂断线。
net.ipv4.tcp_orphan_retries = 3
# 开启 SYN Cookie，防止 SYN Flood 攻击。
net.ipv4.tcp_syncookies = 1
# 指定 连接跟踪表（conntrack table） 的最大条目数
net.ipv4.ip_conntrack_max = 655350
# 设置系统的 SYN 队列长度，表示在处理完三次握手之前能容纳的最大半连接请求数。
net.ipv4.tcp_max_syn_backlog = 262144
# 关闭 TCP 时间戳，减少 TCP 包的开销，适合高性能网络环境。
net.ipv4.tcp_timestamps = 0
# 调整监听队列的最大长度，允许更多的并发连接请求。
net.core.somaxconn = 32768
EOF
```

###### 配置生效

```bash
sysctl --system
```

###### 重启系统

```bash
reboot
```

###### 查看模块加载情况

```bash
lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

2.1.9 其他工具安装（选装）

```bash
yum install -y wget jq vim net-tools telnet tcpdump nmap
```

#### 2.2 配置免密登录

> 在master01上操作

```bash
ssh-keygen
```

###### 将公钥复制到各个主机

```bash
for i in master01 master02 master03 worker01; do
  ssh-copy-id $i
done
```

###### 测试

```bash
ssh root@master02
```

#### 2.4 部署ETCD 集群

> 在master01上操作

##### 2.4.1 创建工作目录

```bash
mkdir -p /data/k8s-worker
```

##### 2.4.2 获取cfssl工具

```bash
cd /data/k8s-worker
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64
```

###### 配置工具

```bash
chmod +x cfssljson_1.6.5_linux_amd64 cfssl_1.6.5_linux_amd64
mv cfssljson_1.6.5_linux_amd64 /usr/local/bin/cfssljson
mv cfssl_1.6.5_linux_amd64 /usr/local/bin/cfssl
```

###### 测试

```bash
cfssl version
cfssljson --version
```

##### 2.4.3 创建CA证书

###### 配置CA证书请求文件

```bash
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "87600h"
  }
}
EOF
```

###### 生成证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

###### 配置CA证书策略

```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

##### 2.4.4 创建etcd证书

###### 生成etcd证书请求文件

```bash
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.19.219.49",
    "172.19.219.50",
    "172.19.219.51"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

###### 生成证书

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

##### 2.4.5 部署etcd集群

###### 获取软件并配置

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.5.13/etcd-v3.5.13-linux-amd64.tar.gz

tar -xvf etcd-v3.5.13-linux-amd64.tar.gz
cp -p etcd-v3.5.13-linux-amd64/etcd* /usr/local/bin/
```

###### 分发到其他master节点

```bash
scp etcd-v3.5.13-linux-amd64/etcd* root@master02:/usr/local/bin/
scp etcd-v3.5.13-linux-amd64/etcd* root@master03:/usr/local/bin/
```

###### 在各个master节点创建配置文件 IP 需要修改

```bash
mkdir -p /etc/etcd

cat > /etc/etcd/etcd.conf <<EOF
#[Member]
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://172.19.219.49:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.19.219.49:2379,https://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.19.219.49:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://172.19.219.49:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://172.19.219.49:2380,etcd2=https://172.19.219.50:2380,etcd3=https://172.19.219.51:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

###### 创建systemd服务

```bash
mkdir -p /var/lib/etcd/default.etcd
mkdir -p /etc/etcd/ssl
```

```bash
cd /data/k8s-worker
cp ca*.pem etcd*.pem /etc/etcd/ssl/
```

```bash
cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

###### 同步配置到其他master节点

```bash
mkdir -p /var/lib/etcd/default.etcd
chmod 700 /var/lib/etcd/default.etcd
mkdir -p /etc/etcd/ssl
scp ca*.pem etcd*.pem root@master02:/etc/etcd/ssl/
scp ca*.pem etcd*.pem root@master03:/etc/etcd/ssl/

scp /etc/systemd/system/etcd.service root@master02:/etc/systemd/system/
scp /etc/systemd/system/etcd.service root@master03:/etc/systemd/system/
```

###### 启动集群

```bash
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

###### 验证集群状态

```bash
etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem \
  --endpoints="https://172.19.219.49:2379,https://172.19.219.50:2379,https://172.19.219.51:2379" endpoint health
```

#### 2.5 Kubernetes 集群部署

##### 2.5.1 软件包准备

下载地址 https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.28.md#v12814

##### 2.5.2 软件安装

```bash
tar -xf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin/
cp kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

##### 2.5.3 软件分发

```bash
scp kube-apiserver kube-controller-manager kube-scheduler kubectl root@master02:/usr/local/bin/
scp kube-apiserver kube-controller-manager kube-scheduler kubectl root@master03:/usr/local/bin/
```

###### 将master节点作为工作节点使用

```bash
scp kubelet kube-proxy root@worker01:/usr/local/bin/
scp kubelet kube-proxy root@master01:/usr/local/bin/
scp kubelet kube-proxy root@master02:/usr/local/bin/
scp kubelet kube-proxy root@master03:/usr/local/bin/
```

##### 2.5.4 集群节点创建工作目录

> 所有master

```
mkdir -p /etc/kubernetes/ssl
mkdir -p /var/log/kubernetes
```

##### 2.5.5 部署api-server

###### 创建api-server证书请求文件

```bash
cat > kube-apiserver-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.19.219.49",
    "172.19.219.50",
    "172.19.219.51",
    "10.96.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],

  "key": {
    "algo": "rsa",
    "size": 2048
  },

  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

###### 生成api-server证书和私钥

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
```

###### 创建token文件

```bash
cat > token.csv <<EOF
$RANDOM,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

###### 创建spiserver systemd 服务文件

```bash
cat > /etc/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
After=etcd.service
Wants=etcd.service

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --anonymous-auth=false \
  --bind-address=172.19.219.49 \
  --secure-port=6443 \
  --advertise-address=172.19.219.49 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/ssl/token.csv \
  --service-cluster-ip-range=10.96.0.0/16 \
  --service-node-port-range=30000-50000 \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-issuer=api \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://172.19.219.49:2379,https://172.19.219.50:2379,https://172.19.219.51:2379 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --event-ttl=1h \
  --v=4
Restart=on-failure
LimitNOFILE=65536
RestartSec=5
Type=notify

[Install]
WantedBy=multi-user.target
EOF
```

###### 同步文件到集群master节点

```bash
cp ca*.pem kube-apiserver*.pem token.csv /etc/kubernetes/ssl/
scp ca*.pem kube-apiserver*.pem token.csv root@master02:/etc/kubernetes/ssl/
scp ca*.pem kube-apiserver*.pem token.csv root@master03:/etc/kubernetes/ssl/


scp /etc/systemd/system/kube-apiserver.service root@master02:/etc/systemd/system/
scp /etc/systemd/system/kube-apiserver.service root@master03:/etc/systemd/system/
```

###### 启动apiserver服务

```bash
systemctl daemon-reload
systemctl enable --now kube-apiserver
systemctl status kube-apiserver
```

###### 测试

```bash
curl --insecure https://172.19.219.49:6443/version
curl --insecure https://172.19.219.50:6443/version
curl --insecure https://172.19.219.51:6443/version
```

##### 2.5.6 部署kubectl

###### 创建kubectl 证书请求文件

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```

###### 生成kubectl 证书和私钥

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

###### 复制证书到指定目录

```bash
cp admin*.pem /etc/kubernetes/ssl/
```

###### 创建kubectl 配置文件

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://172.19.219.49:6443 --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --client-key=/etc/kubernetes/ssl/admin-key.pem \
  --embed-certs=true --kubeconfig=admin.kubeconfig

kubectl config set-context kubernetes \
  --cluster=kubernetes --user=admin --kubeconfig=admin.kubeconfig

kubectl config use-context kubernetes --kubeconfig=admin.kubeconfig
```

###### 配置kubectl 默认路径并绑定角色

```bash
mkdir -p $HOME/.kube
cp admin.kubeconfig $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl create clusterrolebinding kubelet-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user=kubernetes \
  --kubeconfig=$HOME/.kube/config
```

###### 测试

```bash
kubectl cluster-info

```

##### 2.5.7 部署kube-controller-manager

###### 创建kube-controller-manager 证书请求文件

```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [
  "127.0.0.1",
  "172.19.219.49",
  "172.19.219.50",
  "172.19.219.51"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
EOF
```

###### 生成kube-controller-manager 证书和私钥

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

cp kube-controller-manager*.pem /etc/kubernetes/ssl/

```

###### 创建kube-controller-manager kubeconfig 文件

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://172.19.219.49:6443 --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=/etc/kubernetes/ssl/kube-controller-manager.pem \
  --client-key=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
  --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

cp kube-controller-manager.kubeconfig /etc/kubernetes/
```

###### 创建kube-controller-manager systemd 服务文件

```bash
cat > /etc/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.244.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.96.0.0/16 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

###### 启动测试

```bash
systemctl daemon-reload
systemctl enable --now kube-controller-manager
systemctl status kube-controller-manager
```

###### 分发文件到其他节点

```bash
scp kube-controller-manager*.pem root@master02:/etc/kubernetes/ssl/
scp kube-controller-manager*.pem root@master03:/etc/kubernetes/ssl/

scp kube-controller-manager.kubeconfig root@master02:/etc/kubernetes/
scp kube-controller-manager.kubeconfig root@master03:/etc/kubernetes/

scp /etc/systemd/system/kube-controller-manager.service root@master02:/etc/systemd/system/
scp /etc/systemd/system/kube-controller-manager.service root@master03:/etc/systemd/system/
```

##### 2.5.8 部署kube-scheduler

###### 创建kube-scheduler 证书请求文件

```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [
  "127.0.0.1",
  "172.19.219.49",
  "172.19.219.50",
  "172.19.219.51"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
EOF
```

###### 生成kube-scheduler 证书和私钥

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler


cp kube-scheduler*.pem /etc/kubernetes/ssl/
```

###### 创建kube-scheduler kubeconfig 文件

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://172.19.219.49:6443 --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=/etc/kubernetes/ssl/kube-scheduler.pem \
  --client-key=/etc/kubernetes/ssl/kube-scheduler-key.pem \
  --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

cp kube-scheduler.kubeconfig /etc/kubernetes/
```

###### 创建kube-scheduler systemd 服务文件

```bash
cat > /etc/kubernetes/kube-scheduler.yaml <<EOF
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/etc/kubernetes//kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

cat > /etc/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/kube-scheduler.yaml \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

###### 启动测试

```bash
systemctl daemon-reload
systemctl enable --now kube-scheduler
systemctl status kube-scheduler
```

###### 分发文件到其他节点

```bash
scp kube-scheduler*.pem root@master02:/etc/kubernetes/ssl/
scp kube-scheduler*.pem root@master03:/etc/kubernetes/ssl/

scp kube-scheduler.kubeconfig root@master02:/etc/kubernetes/
scp kube-scheduler.kubeconfig root@master03:/etc/kubernetes/

scp /etc/kubernetes/kube-scheduler.yaml root@master02:/etc/kubernetes/
scp /etc/kubernetes/kube-scheduler.yaml root@master03:/etc/kubernetes/

scp /etc/systemd/system/kube-scheduler.service root@master02:/etc/systemd/system/
scp /etc/systemd/system/kube-scheduler.service root@master03:/etc/systemd/system/
```

#### 2.6 工作节点配置

###### 生成工作节点kubeconfig和kube-proxy kubeconfig

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://172.19.219.49:6443 \
  --kubeconfig=worker01.kubeconfig

kubectl config set-credentials system:node:worker01 \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --client-key=/etc/kubernetes/ssl/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=worker01.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=system:node:worker01 \
  --kubeconfig=worker01.kubeconfig

kubectl config use-context default \
  --kubeconfig=worker01.kubeconfig


kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/ssl/admin.pem \
    --embed-certs=true \
    --server=https://172.19.219.49:6443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --client-key=/etc/kubernetes/ssl/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-proxy.kubeconfig
```

###### 分发到工作节点

```bash
scp worker01.kubeconfig root@worker01:/etc/kubernetes/
scp kube-proxy.kubeconfig root@worker01:/etc/kubernetes/

scp /etc/kubernetes/ssl/ca*.pem root@worker01:/etc/kubernetes/ssl/
```

###### 下载工作节点软件

```bash
cni-plugins-linux-amd64-v1.5.1.tgz
containerd-1.7.22-linux-amd64.tar.gz
crictl-v1.31.1-linux-amd64.tar.gz
runc.amd64
kubernetes-node-linux-amd64.tar.gz
```

```bash
tar -xvf kubernetes-node-linux-amd64.tar.gz
cd kubernetes/node/bin/
cp kubelet kube-proxy /usr/local/bin/
```

###### 创建配置目录

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

###### 解压安装软件

```bash
mkdir -p containerd
tar -xvf crictl-v1.31.1-linux-amd64.tar.gz
tar -xvf containerd-1.7.22-linux-amd64.tar.gz -C containerd
tar -xvf cni-plugins-linux-amd64-v1.5.1.tgz -C /opt/cni/bin/
mv runc.amd64 runc
chmod +x crictl runc 
mv crictl runc /usr/local/bin/
mv containerd/bin/* /bin/
```

###### 创建CNI配置文件

```bash
cat > /etc/cni/net.d/10-bridge.conf <<EOF
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "10.244.0.0/16"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
EOF
```

```bash
cat > /etc/cni/net.d/99-loopback.conf <<EOF
{
  "cniVersion": "0.3.1",
  "name": "lo",
  "type": "loopback"
}
EOF
```

###### 配置containerd

```bash
mkdir -p /etc/containerd/

cat > /etc/containerd/config.toml << EOF
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"
    default_runtime_name = "runc"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
[plugins."io.containerd.grpc.v1.cri".cni]
  bin_dir = "/opt/cni/bin"
  conf_dir = "/etc/cni/net.d"
EOF
```

```bash
cat > /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

###### 配置kubelet

```bash
cat > /var/lib/kubelet/kubelet-config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/ssl/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.2"
cgroupDriver: systemd
containerRuntimeEndpoint: "unix:///var/run/containerd/containerd.sock"
podCIDR: "10.244.0.0/16"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/etc/kubernetes/ssl/ca.pem"
tlsPrivateKeyFile: "/etc/kubernetes/ssl/ca-key.pem"
EOF

```

```bash
cat > /etc/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/etc/kubernetes/worker01.kubeconfig \
  --register-node=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

###### 配置kube-proxy

```bash
cat > /var/lib/kube-proxy/kube-proxy-config.yaml <<EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/etc/kubernetes/kube-proxy.kubeconfig"
mode: "iptables"
clusterCIDR: "10.244.0.0/16"
EOF
```

```bash
cat > /etc/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

###### 启动测试

```bash
systemctl daemon-reload
systemctl enable containerd kubelet kube-proxy
systemctl restart containerd kubelet kube-proxy

systemctl status containerd 
systemctl status kubelet
systemctl status kube-proxy
```

###### 拉取 registry.k8s.io/pause:3.8

```bash
crictl -n k8s.io pull k8s.m.daocloud.io/pause:3.8
ctr --namespace k8s.io image tag k8s.m.daocloud.io/pause:3.8 registry.k8s.io/pause:3.8
```

