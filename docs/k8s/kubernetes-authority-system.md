#### K8S 权限体系<Badge type="tip" text="k8s" />

##### API 请求流程

![image-20241031150003433](/image-20241031150003433.png)

##### 身份验证流程

![image-20241031150440136](/image-20241031150440136.png)

##### 1. 权限列表

###### API 资源组织方式

```bash
resources:
  - /api/v1/namespaces/{namespace}/pods/{name}
  - /api/v1/namespaces/{namespace}/pods/{name}/log
  - /api/v1/namespaces/{namespace}/serviceaccounts/{name}
verbs:
  - get
  - list
  - watch
---
# 简化后
resources:
  - pods
  - pods/logs
  - serviceaccounts
verbs:
  - get
  - list
  - watch
```

###### 内置和自定义资源组织方式

![image-20241031152749376](/image-20241031152749376.png)

###### 权限组织方式 rule

![image-20241031152944712](/image-20241031152944712.png)

##### 2. 角色

| 特性           | Role                     | ClusterRole              |
|----------------|--------------------------|--------------------------|
| **作用范围**   | 特定命名空间             | 整个集群                 |
| **使用场景**   | 限制特定命名空间的权限   | 管理跨命名空间或集群级别资源 |
| **绑定方式**   | RoleBinding              | ClusterRoleBinding        |


##### 3. 授权方式

###### 创建用户并颁发证书

```bash
openssl genrsa -out myuser.key 2048
openssl req -new -key myuser.key -out myuser.csr -subj "/CN=myuser"

cat myuser.csr | base64 | tr -d "\n"

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
```

###### 批准证书

```bash
kubectl get csr

kubectl certificate approve myuser

kubectl get csr/myuser -o yaml

# 导出证书
kubectl get csr myuser -o jsonpath='{.status.certificate}'| base64 -d > myuser.crt
```

###### 创建SA并绑定应用

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa:app1             # arbitrary but unique string
  namespace: demo-namespace
---
# 在应用配置中指定sa name
```

###### 创建角色

```bash
kubectl create role developer --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods
```

###### 创建绑定

```bash
kubectl create rolebinding developer-binding-myuser --role=developer --user=myuser
```

###### 创建kubeconfig

```bash
kubectl config set-credentials myuser --client-key=myuser.key --client-certificate=myuser.crt --embed-certs=true

kubectl config set-context myuser --cluster=kubernetes --user=myuser

kubectl config use-context myuser
```

##### 4. PKI 体系

###### 组件间通信关系图

![image-20241031160912181](/image-20241031160912181.png)

###### 数字证书颁发

![image-20241031161030083](/image-20241031161030083.png)

###### 双向TLS认证流程

![image-20241031161151353](/image-20241031161151353.png)

###### K8S 证书调用关系

![image-20241031161354390](/image-20241031161354390.png)
1. **etcd 集群中各个节点之间相互通信使用的证书**
   - 该证书同时用作服务器证书和客户端证书，因为一个 etcd 节点既为其他节点提供服务，又需要作为客户端访问其他节点。
2. **etcd 集群向外提供服务使用的证书**
   - 该证书是服务器证书。
3. **kube-apiserver 作为客户端访问 etcd 使用的证书**
   - 该证书是客户端证书。
4. **kube-apiserver 对外提供服务使用的证书**
   - 该证书是服务器证书。
5. **kube-controller-manager 作为客户端访问 kube-apiserver 使用的证书**
   - 该证书是客户端证书。
6. **kube-scheduler 作为客户端访问 kube-apiserver 使用的证书**
   - 该证书是客户端证书。
7. **kube-proxy 作为客户端访问 kube-apiserver 使用的证书**
   - 该证书是客户端证书。
8. **kubelet 作为客户端访问 kube-apiserver 使用的证书**
   - 该证书是客户端证书。
9. **管理员用户通过 kubectl 访问 kube-apiserver 使用的证书**
   - 该证书是客户端证书。
10. **kubelet 对外提供服务使用的证书**
    - 该证书是服务器证书。
11. **kube-apiserver 作为客户端访问 kubelet 采用的证书**
    - 该证书是客户端证书。
12. **kube-controller-manager 用于生成和验证 service-account token 的证书**
    - 该证书并不会像其他证书一样用于身份认证，而是将证书中的公钥/私钥对用于 service account token 的生成和验证。
    - kube-controller-manager 会用该证书的私钥来生成 service account token，然后以 secret 的方式加载到 pod 中。
    - pod 中的应用可以使用该 token 来访问 kube-apiserver，kube-apiserver 会使用该证书中的公钥来验证请求中的 token。

###### 证书存放路径

```bash
/etc/kubernetes/pki/etcd/ca.key
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/apiserver-etcd-client.key
/etc/kubernetes/pki/apiserver-etcd-client.crt
/etc/kubernetes/pki/ca.key
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/apiserver.key
/etc/kubernetes/pki/apiserver.crt
/etc/kubernetes/pki/apiserver-kubelet-client.key
/etc/kubernetes/pki/apiserver-kubelet-client.crt
/etc/kubernetes/pki/front-proxy-ca.key
/etc/kubernetes/pki/front-proxy-ca.crt
/etc/kubernetes/pki/front-proxy-client.key
/etc/kubernetes/pki/front-proxy-client.crt
/etc/kubernetes/pki/etcd/server.key
/etc/kubernetes/pki/etcd/server.crt
/etc/kubernetes/pki/etcd/peer.key
/etc/kubernetes/pki/etcd/peer.crt
/etc/kubernetes/pki/etcd/healthcheck-client.key
/etc/kubernetes/pki/etcd/healthcheck-client.crt
/etc/kubernetes/pki/sa.key
/etc/kubernetes/pki/sa.pub
```

##### 引用


> -  https://www.zhaohuabing.com/post/2020-05-19-k8s-certificate/
> - https://kubernetes.io/docs/setup/best-practices/certificates/
> - https://learnk8s.io/rbac-kubernetes