## `Centos7`从零搭建`K8S`集群

### 主机配置：

```
k8s-master-1:192.168.20.148
k8s-node-1:192.168.20.149
k8s-node-2:192.168.20.150
k8s-master-2:192.168.20.151

硬件：2Core,2G,20G
系统：centos7 x86_64
```

##### 配置`hosts`

```
cat >> /etc/hosts << EOF
192.168.20.148 k8s-master-1
192.168.20.151 k8s-master-2
192.168.20.149 k8s-node-1
192.168.20.150 k8s-node-2
EOF
```

##### 配置Dockers镜像加速

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxxxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

##### 禁用`swap`

```
swapoff -a
vi /etc/fstab #注释swap所在行
```

##### 关闭防火墙

```
systemctl stop firewalld
systemctl disable firewalld
```

##### 关闭`selinux`

```
setenforce 0
sed -i 's/enforcing/disabled/' /etc/selinux/config
```

### 签名证书

##### 准备`cfssl`工具

```
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo
chmod +x /usr/local/bin/cfssl*
```

##### 自签`Etcd SSL`证书

创建目录：`mkdir -p /k8s/etcd/{ssl,cfg,bin}`

创建CA配置文件：`ca-config.json`

```
cd /k8s/etcd/ssl
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "etcd": {
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

创建CA证书签名请求文件：`ca-csr.json`

```
cat > ca-csr.json <<EOF
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "etcd",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
EOF
```

生成CA证书和私钥：`cfssl gencert -initca ca-csr.json | cfssljson -bare ca`

创建证书签名请求文件：`etcd-csr.json`

```
cat > etcd-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
      "192.168.20.148",
      "192.168.20.149",
      "192.168.20.150"
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
            "O": "etcd",
            "OU": "System"
        }
    ]
}
EOF
```

为`etcd`生成证书和私钥：`cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd etcd-csr.json | cfssljson -bare etcd`

### `Etcd`数据库集群部署

##### 下载`etcd`：

```
wget https://github.com/etcd-io/etcd/releases/download/v3.2.28/etcd-v3.2.28-linux-amd64.tar.gz
tar -xzvf etcd-v3.2.28-linux-amd64.tar.gz
cp etcd-v3.2.28-linux-amd64/{etcd,etcdctl} /k8s/etcd/bin
rm -rf etcd-v3.2.28-linux-amd64*
```

##### 配置etcd配置文件：etcd.conf

```
cat > /k8s/etcd/cfg/etcd.conf <<EOF 
# [member]
ETCD_NAME=etcd-1
ETCD_DATA_DIR=/k8s/data/default.etcd
ETCD_LISTEN_PEER_URLS=https://192.168.20.148:2380
ETCD_LISTEN_CLIENT_URLS=https://192.168.20.148:2379

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://192.168.20.148:2380
ETCD_ADVERTISE_CLIENT_URLS=https://192.168.20.148:2379
ETCD_INITIAL_CLUSTER=etcd-1=https://192.168.20.148:2380,etcd-2=https://192.168.20.149:2380,etcd-3=https://192.168.20.150:2380
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
ETCD_INITIAL_CLUSTER_STATE=new

# [security]
ETCD_CERT_FILE=/k8s/etcd/ssl/etcd.pem
ETCD_KEY_FILE=/k8s/etcd/ssl/etcd-key.pem
ETCD_TRUSTED_CA_FILE=/k8s/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/k8s/etcd/ssl/etcd.pem
ETCD_PEER_KEY_FILE=/k8s/etcd/ssl/etcd-key.pem
ETCD_PEER_TRUSTED_CA_FILE=/k8s/etcd/ssl/ca.pem
EOF
```

##### 创建`Etcd`服务：`etcd.service`

```
cat > /k8s/etcd/etcd.service <<'EOF'
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/k8s/etcd/cfg/etcd.conf
WorkingDirectory=${ETCD_DATA_DIR}

ExecStart=/k8s/etcd/bin/etcd \
  --name=${ETCD_NAME} \
  --data-dir=${ETCD_DATA_DIR} \
  --listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
  --listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
  --initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
  --advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
  --initial-cluster=${ETCD_INITIAL_CLUSTER} \
  --initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
  --initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \
  --cert-file=${ETCD_CERT_FILE} \
  --key-file=${ETCD_KEY_FILE} \
  --trusted-ca-file=${ETCD_TRUSTED_CA_FILE} \
  --peer-cert-file=${ETCD_PEER_CERT_FILE} \
  --peer-key-file=${ETCD_PEER_KEY_FILE} \
  --peer-trusted-ca-file=${ETCD_PEER_TRUSTED_CA_FILE}

Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### 查看集群状态

```
/k8s/etcd/bin/etcdctl \
    --ca-file=/k8s/etcd/ssl/ca.pem \
    --cert-file=/k8s/etcd/ssl/etcd.pem \
    --key-file=/k8s/etcd/ssl/etcd-key.pem \
    --endpoints=https://192.168.20.148:2379,https://192.168.20.149:2379,https://192.168.20.150:2379 \
cluster-health
```

![image-20220922102238477](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220922102238477.png)

### 部署`Master`组件

##### 自签`ApiServer SSL`证书

```
mkdir -p /k8s/kubernetes/{ssl,cfg,bin,logs}
```

创建CA配置文件：`ca-config.json`

```
cat > /k8s/kubernetes/ssl/ca-config.json <<EOF
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

创建CA证书签名请求文件：`ca-csr.json`

```
cat > /k8s/kubernetes/ssl/ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "kubernetes",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
EOF
```

生成CA证书和私钥：`cfssl gencert -initca ca-csr.json | cfssljson -bare ca`

创建证书签名请求文件：`kubernetes-csr.json`

```
cat > /k8s/kubernetes/ssl/kubernetes-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.0.0.1",
      "192.168.20.148",
      "192.168.20.149",
      "192.168.20.150",
      "192.168.20.151",
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
            "O": "kubernetes",
            "OU": "System"
        }
    ]
}
EOF
```

为`kubernetes`生成证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```

##### 部署`kube-apiserver`组件

下载`kubernetes`二进制包：[下载链接](https://storage.googleapis.com/kubernetes-release/release/v1.19.0/kubernetes-server-linux-amd64.tar.gz)，将下载好的文件`kubernetes-server-linux-amd64.tar.gz`上传到`/usr/local/src`文件夹下，并解压

```
tar -xzvf kubernetes-server-linux-amd64.tar.gz
```

将Master节点上部署的组件拷贝到`/k8s/kubernetes/bin`目录下：

```
cp -p /usr/local/src/kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler} /k8s/kubernetes/bin/
cp -p /usr/local/src/kubernetes/server/bin/kubectl /usr/local/bin/
```

创建Node令牌文件：`token.csv`

```
# 生成token字符串
head -c 16 /dev/urandom | od -An -t x |tr -d ' '
# 使用生成的字符串替换0c2be6b1473e60b5f723c404b34e64bf
cat > /k8s/kubernetes/cfg/token.csv <<'EOF'
0c2be6b1473e60b5f723c404b34e64bf,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

创建`kube-apiserver`配置文件：`kube-apiserver.conf`

```sh
cat > /k8s/kubernetes/cfg/kube-apiserver.conf <<'EOF'
KUBE_APISERVER_OPTS="--etcd-servers=https://192.168.20.148:2379,https://192.168.20.149:2379,https://192.168.20.150:2379 \
  --bind-address=192.168.20.148 \
  --secure-port=6443 \
  --advertise-address=192.168.20.148 \
  --allow-privileged=true \
  --service-cluster-ip-range=10.0.0.0/24 \
  --service-node-port-range=30000-32767 \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
  --authorization-mode=RBAC,Node \
  --enable-bootstrap-token-auth=true \
  --token-auth-file=/k8s/kubernetes/cfg/token.csv \
  --kubelet-client-certificate=/k8s/kubernetes/ssl/kubernetes.pem \
  --kubelet-client-key=/k8s/kubernetes/ssl/kubernetes-key.pem \
  --tls-cert-file=/k8s/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/k8s/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/k8s/kubernetes/ssl/ca.pem \
  --service-account-key-file=/k8s/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/k8s/etcd/ssl/ca.pem \
  --etcd-certfile=/k8s/etcd/ssl/etcd.pem \
  --etcd-keyfile=/k8s/etcd/ssl/etcd-key.pem \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/k8s/kubernetes/logs/k8s-audit.log"
EOF
```

创建 `apiserver` 服务：`kube-apiserver.service`

```sh
cat > /usr/lib/systemd/system/kube-apiserver.service <<'EOF'
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-apiserver.conf
ExecStart=/k8s/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS

Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

启动`kube-apiserver`服务：

```sh
systemctl daemon-reload
systemctl start kube-apiserver
systemctl enable kube-apiserver
```

将 `kubelet-bootstrap` 用户绑定到系统集群角色：

```sh
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

##### 部署 `kube-controller-manager` 组件

创建 `kube-controller-manager` 配置文件：`kube-controller-manager.conf`

```sh
cat > /k8s/kubernetes/cfg/kube-controller-manager.conf <<'EOF'
KUBE_CONTROLLER_MANAGER_OPTS="--leader-elect=true \
  --master=127.0.0.1:8080 \
  --address=127.0.0.1 \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.244.0.0/16 \
  --service-cluster-ip-range=10.0.0.0/24 \
  --cluster-signing-cert-file=/k8s/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/k8s/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/k8s/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/k8s/kubernetes/ssl/ca-key.pem \
  --experimental-cluster-signing-duration=87600h0m0s \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

创建 `kube-controller-manager` 服务：`kube-controller-manager.service`

```sh
cat > /usr/lib/systemd/system/kube-controller-manager.service <<'EOF'
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/k8s/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS

Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

 启动 `kube-controller-manager` 组件

```sh
# systemctl daemon-reload
# systemctl start kube-controller-manager
# systemctl enable kube-controller-manager
```

##### 部署 `kube-scheduler` 组件

创建 `kube-scheduler` 配置文件：`kube-scheduler.conf`

```sh
cat > /k8s/kubernetes/cfg/kube-scheduler.conf <<'EOF'
KUBE_SCHEDULER_OPTS="--leader-elect=true \
  --master=127.0.0.1:8080 \
  --address=127.0.0.1 \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

创建 `kube-scheduler` 服务：`kube-scheduler.service`

```sh
cat > /usr/lib/systemd/system/kube-scheduler.service <<'EOF'
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kube-scheduler.conf
ExecStart=/k8s/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS

Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

启动 `kube-scheduler` 组件

```sh
# systemctl daemon-reload
# systemctl start kube-scheduler
# systemctl enable kube-scheduler
```

查看集群状态：

```sh
# kubectl get cs
```

###  部署Node组件

##### 安装`docker`

参考文档：https://docs.docker.com/engine/install/centos/

##### `Node`节点证书

创建 Node 节点的证书签名请求文件：`kube-proxy-csr.json`

```
cat > /k8s/kubernetes/ssl/kube-proxy-csr.json <<EOF
{
    "CN": "system:kube-proxy",
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
            "O": "kubernetes",
            "OU": "System"
        }
    ]
}
EOF
```

 为 `kube-proxy` 生成证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

在 `k8s-node-1`节点上创建`k8s`目录

```
 -p /k8s/kubernetes/{bin,cfg,logs,ssl}
```

将 `k8s-master-1` 节点的`kubelet、kube-proxy `文件和证书拷贝到 `node` 节点

```sh
scp -r /usr/local/src/kubernetes/server/bin/{kubelet,kube-proxy} root@k8s-node-1:/k8s/kubernetes/bin/
scp -r /k8s/kubernetes/ssl/{ca.pem,kube-proxy.pem,kube-proxy-key.pem} root@k8s-node-1:/k8s/kubernetes/ssl/
```

##### 安装`kubelet`

创建请求证书的配置文件：`bootstrap.kubeconfig`

`bootstrap.kubeconfig` 将用于向 `apiserver` 请求证书，`apiserver` 会验证 `token`、证书 是否有效，验证通过则自动颁发证书。

```sh
cat > /k8s/kubernetes/cfg/bootstrap.kubeconfig <<'EOF'
apiVersion: v1
clusters:
- cluster: 
    certificate-authority: /k8s/kubernetes/ssl/ca.pem
    server: https://192.168.20.148:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet-bootstrap
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 0c2be6b1473e60b5f723c404b34e64bf
EOF
```

创建`kubelet`配置文件：`kubelet-config.yml`

```sh
cat > /k8s/kubernetes/cfg/kubelet-config.yml <<'EOF'
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2 
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509: 
    clientCAFile: /k8s/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthroizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 100000
maxPods: 110
EOF
```

创建 `kubelet` 服务配置文件：`kubelet.conf`

```
cat > /k8s/kubernetes/cfg/kubelet.conf <<'EOF'
KUBELET_OPTS="--hostname-override=k8s-node-1 \
  --network-plugin=cni \
  --cni-bin-dir=/opt/cni/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --cgroups-per-qos=false \
  --enforce-node-allocatable="" \
  --kubeconfig=/k8s/kubernetes/cfg/kubelet.kubeconfig \
  --bootstrap-kubeconfig=/k8s/kubernetes/cfg/bootstrap.kubeconfig \
  --config=/k8s/kubernetes/cfg/kubelet-config.yml \
  --cert-dir=/k8s/kubernetes/ssl \
  --pod-infra-container-image=kubernetes/pause:latest \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

创建`kubelet`服务：`kubelet.service`

```
cat > /usr/lib/systemd/system/kubelet.service <<'EOF'
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Before=docker.service

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kubelet.conf
ExecStart=/k8s/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

启动`kubelet`

```
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
```

##### 安装`kube-proxy`

创建 `kube-proxy` 连接 `apiserver` 的配置文件：`kube-proxy.kubeconfig`

```sh
cat > /k8s/kubernetes/cfg/kube-proxy.kubeconfig <<'EOF'
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /k8s/kubernetes/ssl/ca.pem
    server: https://192.168.20.148:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-proxy
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-proxy
  user:
    client-certificate: /k8s/kubernetes/ssl/kube-proxy.pem
    client-key: /k8s/kubernetes/ssl/kube-proxy-key.pem
EOF
```

创建 `kube-proxy` 配置文件：`kube-proxy-config.yml`

```
cat > /k8s/kubernetes/cfg/kube-proxy-config.yml <<'EOF'
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
address: 0.0.0.0
metrisBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /k8s/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-node-1
clusterCIDR: 10.0.0.0/24
mode: ipvs
ipvs:
  scheduler: "rr"
iptables:
  masqueradeAll: true
EOF
```

创建 `kube-proxy` 配置文件：`kube-proxy.conf`

```sh
cat > /k8s/kubernetes/cfg/kube-proxy.conf <<'EOF'
KUBE_PROXY_OPTS="--config=/k8s/kubernetes/cfg/kube-proxy-config.yml \
  --v=2 \
  --logtostderr=false \
  --log-dir=/k8s/kubernetes/logs"
EOF
```

创建 `kube-proxy` 服务：`kube-proxy.service`

```sh
cat > /usr/lib/systemd/system/kube-proxy.service <<'EOF'
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kube-proxy.conf
ExecStart=/k8s/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

启动 `kube-proxy`

```
systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy
```

部署`k8s-node-2`，流程与上述一致，只需将配置文件中的`k8s-node-1`改为`k8s-node-2`即可。

![image-20220922170044480](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220922170044480.png)

### 部署K8S容器集群网络

##### 创建CNI工作目录

```
mkdir -p /opt/cni/bin /etc/cni/net.d
```



### 部署`Dashboard`

##### 部署`dashboard`

下载`dashboard.yaml`文件，[下载地址](http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml)，按下图修改：![img](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/856154-20191206010304862-1366726165.png)

部署`dashboard`：`kubectl apply -f kubernetes-dashboard.yaml`

通过`https`访问`dashboard`：`https://192.168.20.149:30005`（建议使用Firefox浏览器）

##### 登录授权

```sh
cat > kubernetes-adminuser.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
```

启动：`kubectl apply -f kubernetes-adminuser.yaml`

获取登录的`token`：

```sh
# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk ' {print $1}')
Name:         admin-user-token-spctr
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 633dc31e-ee8c-4fc0-909f-c406f7106513

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImZ6c3hBbVNQRnV4YUFOdDVUV2I4cmx0RkNSamZfVGpPdy1kcjQ2N3hwRncifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXNwY3RyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2MzNkYzMxZS1lZThjLTRmYzAtOTA5Zi1jNDA2ZjcxMDY1MTMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.HLs531qRO7m3VJygOplL5G_ngGJ8lFkvpQjaJphLN14oCCeHiMehukUBKOOpp1-DIITghFylsfsJ_eFPqYFwyC4VYaMHkdEJipTLhX23smgUked8l9-a462tuso_h8b-caKZW-7FIH8gUBJlUriQq7Kj56gtfuf5mXXt2-SAMnyjE1g9V1-4waeTXnMkuyQhFTege2ig4gvDHNfrtTQgr3VP8rfpJDo-ZftKlS5q87S1-K0r1FSMgfFgh23rXZMMdjxbkQpWgBlYq7t0wwTjGJXFJp0FudZX0TzvINdOLIWu_Z8W3ihSv3lHr5ubL2lOG6OLNHDtFDyEPUMI76OZnw
ca.crt:     1383 bytes
```

使用`token`就可以登录`dashboard`页面了

### 部署Prometheus

##### 下载`kube-prometheus`

下载地址：https://github.com/prometheus-operator/kube-prometheus/releases

![image-20220924015007703](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220924015007703.png)

根据`Kubernetes`版本下载对应的`kube-prometheus`版本，将下载的压缩包上传到`k8s-master-1`主机，解压到`/k8s`文件夹：

````sh
tar xzf kube-prometheus-0.7.0.tar.gz -C /k8s
````

要想访问`grafana`，需要修改对应的配置文件，并按下图修改：

````sh
vi /k8s/kube-prometheus/manifests/grafana-service.yaml
````

![image-20220924015936913](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220924015936913.png)

##### 创建并启动监控（`monitoring`）

```sh
cd /k8s/kube-prometheus/
kubectl apply -f manifests/setup
kubectl apply -f manifests/
```

执行这两条命令将创建一系列监控相关的`kubernetes`对象，待命令执行完毕，使用命令

```sh
kubectl get pod -A -o wide|grep grafana
```

查看`grafana`被分派的节点，然后使用该节点的`IP`+端口号30030，即可访问`grafana`页面

##### 导入`prometheus`

在`grafana`官网下载`kube-prometheus`模板，[下载链接](https://grafana.com/api/dashboards/14491/revisions/1/download)，将`kube-prometheus`的`json`文件导入`grafana`：

![image-20220924021144615](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220924021144615.png)

导入之后页面如下：

![image-20220924021238795](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220924021238795.png)

### 问题排查流程

##### 通用排查

检查`swap`分区是否关闭：

```sh
free -m       # swap行后需全为0
swapoff -a    # 关闭交换分区
vi /etc/fstab # 注释swap所在行
```

检查防火墙是否禁用：

```sh
systemctl status firewalld       #查看防火墙状态
systemctl disable firewalld      #禁止防火墙开机启动
systemctl stop firewalld         #停止防火墙服务
```

检查是否禁用`selinux`

```sh
getenforce           #获取selinux状态
### 修改/etc/selinux/config 文件
### 将SELINUX=enforcing改为SELINUX=disabled，(需要重启)
```

##### 查看ETCD服务

```sh
systemctl status etcd         #查看etcd服务状态
vi /k8s/etcd/cfg/etcd.conf    #修改etcd配置文件
systemctl daemon-reload       #重新加载配置文件
systemctl start/restart etcd  #启动/重启etcd服务
systemctl enable etcd         #设置etcd服务开机启动
```

注意：首次部署和启动`etcd`服务时，配置文件`etcd.conf`中配置项`--initial-cluster-state=new`，如果二次或之后启动，需要修改为` --initial-cluster-state=existing`；应该先启动各`Node`节点的`etcd`服务，最后启动`Master`节点的`etcd`服务

##### 常用命令：

检查集群健康状态

```sh
/k8s/etcd/bin/etcdctl \
    --ca-file=/k8s/etcd/ssl/ca.pem \
    --cert-file=/k8s/etcd/ssl/etcd.pem \
    --key-file=/k8s/etcd/ssl/etcd-key.pem \
    --endpoints=https://192.168.20.148:2379,https://192.168.20.149:2379,https://192.168.20.150:2379 \
cluster-health
```

各服务启动状态命令：

```sh
## Master节点
systemctl status etcd
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
## Node节点
systemctl status docker
systemctl status kubelet
systemctl status kube-proxy
```

查看组件状态：`kubectl get cs`

查看是否有新的客户端请求颁发证书：`kubectl get csr`

查看节点信息：

```sh
kubectl get node
kubectl get node -o wide
```

部署`Flannel`：`kubectl apply -f kube-flannel.yml`

查看Pod信息：

```sh
kubectl get pods -n kube-system -o wide # -n:指定命名空间
kubectl get pods -o wide
```

创建`Pod`（`Nginx`服务）：`kubectl create deployment web --image=nginx`

暴露端口：`kubectl expose deployment web --port=80 --type=NodePort`

查看服务：`kubectl get svc`

查看所有Pod：`kubectl get pod -A -o wide`

![image-20220923144752397](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220923144752397.png)

获取`deployment`：`kubectl get deployment -A -o wide`

删除Pod（直接删除Pod不能删除，需要通过删除Deployment来删除），其中`DEPLOYMENTNAME`可以通过前面命令获取

```sh
kubectl delete pod PODNAME -n NAMESPACE --force --grace-period=0 #删不掉
kubectl delete deployment DEPLOYMENTNAME -n NAMESPACE #可以删除Pod
### 通过配置文件创建的Pod，需通过配置文件删除
kubectl apply -f CONFIGFILE
kubectl delete -f CONFIGFILE
```

`flannel`启动和删除

```sh
kubectl apply -f kube-flannel.yml
kubectl delete -f kube-flannel.yml
```

查看`Pod`日志：

```sh
kubectl describe pod PODNAME
```

