### Kubernetes v1.30 配置安装 基于docker ubuntu22.04<Badge type="tip" text="k8s" />

#### 主机规划

| 主机名   | IP              | 角色   |
| -------- | --------------- | ------ |
| master   | 192.168.116.146 | master |
| worker01 | 192.168.116.149 | worker |
| worker02 | 192.168.116.148 | worker |

#### 主机配置

##### 1. 设置主机名和域名解析

```bash
hostnamectl set-hostname master
hostnamectl set-hostname worker01
hostnamectl set-hostname worker02
```

```bash
cat <<EOF >> /etc/hosts
192.168.116.146 master
192.168.116.149 worker01
192.168.116.148 worker02
EOF
```

##### 2. 关闭swap分区

```bash
free -h
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
```

##### 3. 加载所需模块和配置IP转发

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
```

##### 4. 安装通用软件

```bash
apt-get update && sudo apt-get upgrade -y

apt-get update && sudo apt-get install -y apt-transport-https curl
```

##### 5. 安装并配置docker

```bash
apt-get install docker.io -y

# 配置docker cgroup
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

# 安装cri-docker
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.15/cri-dockerd_0.3.15.3-0.ubuntu-jammy_amd64.deb

dpkg -i cri-dockerd_0.3.15.3-0.ubuntu-jammy_amd64.deb


# 修改cri-docker 服务启动参数
vim /lib/systemd/system/cri-docker.service

# 添加启动参数
--pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9

ExecStart=/usr/bin/cri-dockerd --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9
```

重启cri-docker docker

```bash
systemctl daemon-reload
systemctl enable cri-docker
systemctl enable docker
systemctl restart cri-docker
systemctl restart docker
systemctl status docker
systemctl status cri-docker
```

#### 集群配置

##### 1. 安装 kubeadm kubectl kubelet

配置软件源 k8s 版本 v1.30

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

apt-get install -y apt-transport-https ca-certificates curl gpg

mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

安装

```bash
apt-get update
apt-get install -y kubelet kubeadm kubectl

apt-mark hold kubelet kubeadm kubectl

systemctl enable --now kubelet
```

##### 2. 拉取所需镜像  根据节点角色按需拉取

```bash
# 设置版本号，例如 v1.27.1
KUBE_VERSION=v1.30.5
ETCD_VERSION=3.5.15-0
COREDNS_VERSION=v1.11.3
PAUSE_VERSION=3.9

# 使用阿里云镜像仓库
ALIYUN_REGISTRY=registry.aliyuncs.com/google_containers

# 拉取并重新标记镜像
sudo docker pull $ALIYUN_REGISTRY/kube-apiserver:$KUBE_VERSION
sudo docker tag $ALIYUN_REGISTRY/kube-apiserver:$KUBE_VERSION k8s.gcr.io/kube-apiserver:$KUBE_VERSION

sudo docker pull $ALIYUN_REGISTRY/kube-controller-manager:$KUBE_VERSION
sudo docker tag $ALIYUN_REGISTRY/kube-controller-manager:$KUBE_VERSION k8s.gcr.io/kube-controller-manager:$KUBE_VERSION

sudo docker pull $ALIYUN_REGISTRY/kube-scheduler:$KUBE_VERSION
sudo docker tag $ALIYUN_REGISTRY/kube-scheduler:$KUBE_VERSION k8s.gcr.io/kube-scheduler:$KUBE_VERSION

sudo docker pull $ALIYUN_REGISTRY/kube-proxy:$KUBE_VERSION
sudo docker tag $ALIYUN_REGISTRY/kube-proxy:$KUBE_VERSION k8s.gcr.io/kube-proxy:$KUBE_VERSION

sudo docker pull $ALIYUN_REGISTRY/pause:$PAUSE_VERSION
sudo docker tag $ALIYUN_REGISTRY/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION

sudo docker pull $ALIYUN_REGISTRY/etcd:$ETCD_VERSION
sudo docker tag $ALIYUN_REGISTRY/etcd:$ETCD_VERSION k8s.gcr.io/etcd:$ETCD_VERSION

sudo docker pull $ALIYUN_REGISTRY/coredns:$COREDNS_VERSION
sudo docker tag $ALIYUN_REGISTRY/coredns:$COREDNS_VERSION k8s.gcr.io/coredns/coredns:$COREDNS_VERSION
```

##### 3. 初始化集群

```bash
sudo kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

##### 4. 安装flannel插件

```bash
# 下载配置文件 将镜像源改为国内

kubectl apply -f kube-flannel.yml
```

##### 5. 添加工作节点到集群

```bash
kubeadm join 192.168.116.146:6443 --token 6dvpxl.54oerc8gmb9gmx03 \
        --cri-socket unix:///var/run/cri-dockerd.sock \
        --discovery-token-ca-cert-hash sha256:80afda89e6eeb5438c36c6940195372f6381eff7acabf95d314c52ae59b5472e
```

##### 6. 验证集群

```bash
kubectl run nginx --image=dockerpull.com/library/nginx:1.19
```

