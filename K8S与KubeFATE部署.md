# 以下将详细展示如何构建双k8s集群并搭建双方KubeFATE然后通过toy_example测试是否搭建成功

# k8s

## 先决条件
七台具有公网Ip的CentOS7.8机器（或者在同一网段下），每台机器配置为8G8C，具有公网访问能力, 以下所有命令均使用root用户执行.
## 预配置
以下选用7台机器作为范例构建k8s, 机器信息如下。
|ip|角色|
|---|---|
|192.168.92.138| k8s-master1|
|192.168.92.146| k8s-master2|
|192.168.92.141| k8s-master3|
|192.168.92.142| k8s-node1|
|192.168.92.143| k8s-node2|
|192.168.92.144| k8s-lb1|
|192.168.92.145| k8s-lb2|

另准备一个Virtual IP(VIP) 192.168.92.170

以下使用到各主机ip时会使用$k8s-master1_ip, $k8s-master2_ip, ... $vip替代。
### 设置每台主机的主机名，加入k8s时默认以主机名作为k8s集群中的节点名
k8s-master1主机

``` 
hostnamectl set-hostname k8s-master1
```

k8s-master2主机

```
hostnamectl set-hostname k8s-master2
```

k8s-master3主机

```
hostnamectl set-hostname k8s-master3
```

k8s-node1主机

```
hostnamectl set-hostname k8s-node1
```

k8s-node2主机

```
hostnamectl set-hostname k8s-node2
```

k8s-lb1主机

```
hostnamectl set-hostname k8s-lb1
```

k8s-lb2主机

```
hostnamectl set-hostname k8s-lb2
```


可以通过`hostname` 查看主机名

### 将主机的域名解析存入本地/etc/hosts中(每台主机都需要执行)

```
cat >>/etc/hosts<<EOF
$k8s-master1_ip k8s-master1
$k8s-master2_ip k8s-master2
$k8s-master3_ip k8s-master3
$k8s-node1_ip k8s-node1
$k8s-node2_ip k8s-node2
$k8s-lb1_ip k8s-lb1
$k8s-lb2_ip k8s-lb2
EOF
```

### 添加源 (每台主机都需要执行)
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
### 关闭防火墙(每台主机都需要执行)
```
systemctl stop firewalld
systemctl disable firewalld
```
如果需要开启防火墙则需要开放以下端口

master
```
systemctl restart firewalld
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=8472/udp
#firewall-cmd --permanent --add-port=443/udp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=9153/tcp
firewall-cmd --add-masquerade --permanent
# only if you want NodePorts exposed on control plane IP as well
firewall-cmd --permanent --add-port=30000-32767/tcp
systemctl restart firewalld
```
worker
```
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=8472/udp
#firewall-cmd --permanent --add-port=443/udp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=9153/tcp
firewall-cmd --add-masquerade --permanent 
systemctl restart firewalld
```
备注：8472/udp为flannel的通信端口，443/tcp 为Kubernetes server端口
如果需要使用istio则应将istio-pilot的端口加到防火墙里：

```
firewall-cmd --permanent --add-port=15010-15014/tcp
```

### 关闭SWAP(每台主机都需要执行)
```
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab 
echo vm.swappiness=0 >> /etc/sysctl.conf 
sysctl -p # 使配置生效
```

### 关闭SELINUX(每台主机都需要执行)
```
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```
### 安装docker-ce并配置开机启动(每台主机都需要执行)
```
yum -y install docker-ce
systemctl enable docker
```
版本应为19.x
### 修改docker cgroup 和添加registry-mirrors(每台主机都需要执行)
```
cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=cgroupfs"],
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "1000m"
    },
    "storage-driver": "overlay2",
    "registry-mirrors":[
        "https://kfwkfulq.mirror.aliyuncs.com",
        "https://2lqq34jg.mirror.aliyuncs.com",
        "https://pee6w651.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
    ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload
systemctl restart docker
```
### 其他配置(两台主机都需要执行)
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

### 下载镜像(每台主机都需要执行)
```
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.19.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.19.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.19.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.19.2 k8s.gcr.io/kube-apiserver:v1.19.2

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.19.2 k8s.gcr.io/kube-controller-manager:v1.19.2

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.19.2 k8s.gcr.io/kube-scheduler:v1.19.2

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.2 k8s.gcr.io/kube-proxy:v1.19.2

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0

```

## etcd集群部署
为节约设备etcd集群部署在k8s-master1, k8s-master2, k8s-master3上
|主机名|ip|角色|
|---|---|---|
|k8s-master1|$k8s-master1_ip|etcd-1|
|k8s-master2|$k8s-master2_ip|etcd-2|
|k8s-master3|$k8s-master3_ip|etcd-3|
### 准备cfssl证书生成工具(k8s-master1)
```
yum install -y wget
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```
### 自答证书颁发机构（k8s-master1）
```
mkdir -p ~/TLS/{etcd,k8s}
cd ~/TLS/etcd
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "www": {
         "expiry": "876000h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```
### 生成证书(k8s-master1)
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
ls *pem
```
### 创建证书申请文件(k8s-master1)
```
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "$k8s-master1_ip",
    "$k8s-master2_ip",
    "$k8s-master3_ip"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
```
### 生成证书(k8s-master1)
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

ls server*pem
```
### 下载二进制文件(k8s-master1)
```
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz 
```

### 创建工作目录（k8s-master1）
```
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.4.13-linux-amd64.tar.gz
mv etcd-v3.4.13-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```
### 创建etcd配置文件（k8s-master1）
```
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://$k8s-master1_ip:2380"
ETCD_LISTEN_CLIENT_URLS="https://$k8s-master1_ip:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://$k8s-master1_ip:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://$k8s-master1_ip:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://$k8s-master1_ip:2380,etcd-2=https://$k8s-master2_ip:2380,etcd-3=https://$k8s-master3_ip:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

### systemd管理etcd(k8s-master1)
```
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

```
### 拷贝证书(k8s-master1)
```
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/
```

### 文件拷贝到k8s-master2. k8s-master3(k8s-master1)
```
scp -r /opt/etcd/ root@$k8s-master2_ip:/opt/

scp /usr/lib/systemd/system/etcd.service root@$k8s-master2_ip:/usr/lib/systemd/system/


scp -r /opt/etcd/ root@$k8s-master3_ip:/opt/

scp /usr/lib/systemd/system/etcd.service root@$k8s-master3_ip:/usr/lib/systemd/system/
```

### 修正etcd配置(k8s-master2, k8s-master3)
```
vi /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"   # 修改此处，节点2改为etcd-2，节点3改为etcd-3
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://$k8s-master1_ip:2380"   # 修改此处为当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://$k8s-master1_ip:2379" # 修改此处为当前服务器IP

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://$k8s-master1_ip:2380" # 修改此处为当前服务器IP
ETCD_ADVERTISE_CLIENT_URLS="https://$k8s-master1_ip:2379" # 修改此处为当前服务器IP
ETCD_INITIAL_CLUSTER="etcd-1=https://$k8s-master1_ip:2380,etcd-2=https://$k8s-master2_ip:2380,etcd-3=https://$k8s-master3_ip:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
### 启动并设置开机启动（k8s-master{1,2,3}）
```
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```
### 检查etcd集群状态(k8s-master1)
```
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://$k8s-master1_ip:2379,https://$k8s-master2_ip:2379,https://$k8s-master3_ip:2379" endpoint health
```



## Master 部署（k8s-master1）
### 自签证书颁发机构
```
cd ~/TLS/k8s/

cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "876000h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

### 生成CA证书(k8s-master1)
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

ls *pem
```
### 签发kube-apiserver https证书（k8s-master1）
创建证书申请文件
```
cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "$k8s-master1_ip",
      "$k8s-master2_ip",
      "$k8s-master3_ip",
      "$k8s-lb1_ip",
      "$k8s-lb2_ip",
      "$vip",
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
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```
生成证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

ls server*pem
```
### 下载二进制文件(k8s-master1)
```
wget https://dl.k8s.io/v1.18.10/kubernetes-server-linux-amd64.tar.gz
```

### 解压(k8s-master1)
```
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
tar zxf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
cp kubectl /usr/bin/
```
### 部署kube-apiserver(k8s-master1)
创建配置文件
```
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://$k8s-master1_ip:2379,https://$k8s-master2_ip:2379,https://$k8s-master3_ip:2379 \\
--bind-address=$k8s-master1_ip \\
--secure-port=6443 \\
--advertise-address=$k8s-master1_ip \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF
```
拷贝证书
```
cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
```

创建TLS Bootstrapping 机制token
```
cat > /opt/kubernetes/cfg/token.csv << EOF
8054b7219e601b121e8d2b4f73d255ad,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```
生成token命令
```
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

systemd管理kube-apiserver
```
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```
启动设置开机启动
```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl enable kube-apiserver
```
授权kubelet-bootstrap用户允许请求证书
```
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```

### 部署kube-controller-manager(k8s-master1)
创建配置文件
```
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--master=127.0.0.1:8080 \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=876000h0m0s"
EOF
```
systemd管理controller-manager
```
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```
启动并设置开机启动
```
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
```
### 部署kube-scheduler(k8s-master1)
创建配置文件
```
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
EOF
```
systemd管理scheduler
```
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```
启动并设置开机启动
```
systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler
```

### 查看集群状态(k8s-master1)
```
kubectl get cs
```
## Worker部署(k8s-master1)
### 创建工作目录
```
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
```
### 拷贝二进制文件
```
cp kubelet kube-proxy /opt/kubernetes/bin   # 本地拷贝
```
### 部署kubelet
创建配置文件
```
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s-master1 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"
EOF
```
配置参数文件
```
cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
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
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF
```
生成bootstrap.kubeconfig文件
```
KUBE_APISERVER="https://$k8s-master1_ip:6443" # apiserver IP:PORT
TOKEN="8054b7219e601b121e8d2b4f73d255ad" # 与token.csv里保持一致

# 生成 kubelet bootstrap kubeconfig 配置文件
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=bootstrap.kubeconfig

kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
拷贝到配置文件路径：
```
cp bootstrap.kubeconfig /opt/kubernetes/cfg
```
systemd管理kubelet
```
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```
启动并设置开机启动
```
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
```
### 批准kubelet证书申请并加入集群
查看kubelet证书请求

```
kubectl get csr
```

批准

```
kubectl certificate approve $name
```



### 部署kube-proxy(k8s-master1)
创建配置文件
```bash
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF
```
配置参数文件
```yaml
cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-master1
clusterCIDR: 10.0.0.0/24
EOF
```
生成kube-proxy证书
```json
# 切换工作目录
cd ~/TLS/k8s

# 创建证书请求文件
cat > kube-proxy-csr.json << EOF
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

ls kube-proxy*pem
```
生成kubeconfig文件：
```bash
KUBE_APISERVER="https://$k8s-master1_ip:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
拷贝到配置文件指定路径
```
cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
```
systemd管理kube-proxy
```
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```
启动并设置开机启动
```
systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy
```
### 部署CNI网络(k8s-master1)
下载文件
```
wget https://github.com/containernetworking/plugins/releases/download/v0.8.7/cni-plugins-linux-amd64-v0.8.7.tgz
```
解压
```
mkdir -p /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v0.8.7.tgz -C /opt/cni/bin
```
部署
```
wget https://github.com/coreos/flannel/releases/download/v0.13.0/flanneld-v0.13.0-amd64.docker

docker load < flanneld-v0.13.0-amd64.docker

docker tag quay.io/coreos/flannel:v0.13.0-amd64 quay.io/
coreos/flannel:v0.13.0

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
### 授权apiserver访问kubelet
```yaml
cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl apply -f apiserver-to-kubelet-rbac.yaml
```
## 新加worker节点(k8s-node{1,2}以worker node加入k8s集群)
### 拷贝文件
```
scp -r /opt/kubernetes root@$k8s-node1_ip:/opt/
scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@$k8s-node1_ip:/usr/lib/systemd/system
scp -r /opt/cni/ root@$k8s-node1_ip:/opt/
scp -r /opt/kubernetes/ssl/ca.pem root@$k8s-node1_ip:/opt/kubernetes/ssl

scp -r /opt/kubernetes root@$k8s-node2_ip:/opt/
scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@$k8s-node2_ip:/usr/lib/systemd/system
scp -r /opt/cni/ root@$k8s-node2_ip:/opt/
scp -r /opt/kubernetes/ssl/ca.pem root@$k8s-node2_ip:/opt/kubernetes/ssl
```
### 删除kubelet证书和kubeconfig文件(k8s-node{1,2}上执行)
```
rm -f /opt/kubernetes/cfg/kubelet.kubeconfig
rm -f /opt/kubernetes/ssl/kubelet*
```
### 修改主机名
k8s-node1执行
```
sed -i "s/k8s-master1/k8s-node1/g" /opt/kubernetes/cfg/kubelet.conf
sed -i "s/k8s-master1/k8s-node1/g" /opt/kubernetes/cfg/kube-proxy-config.yml
```
k8s-node2执行
```
sed -i "s/k8s-master1/k8s-node2/g" /opt/kubernetes/cfg/kubelet.conf
sed -i "s/k8s-master1/k8s-node2/g" /opt/kubernetes/cfg/kube-proxy-config.yml
```
### 启动并设置开机启动(k8s-worker1/2上执行)
```bash
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
systemctl start kube-proxy
systemctl enable kube-proxy
```
### 在Master上批准新Node kubelet证书申请(k8s-master1)
```
kubectl get csr
kubectl certificate approve $name
```
### 查看node状态(k8s-master1)
```
kubectl get node
```

### 调理master最大pod数量

```
sed -i "s/110/1/g" /opt/kubernetes/cfg/kubelet-config.yml
systemctl restart kubelet
```

限制master节点pod为1即仅有一个flannel的pod，使master节点更专注于其控制平面的功能。



## 部署CoreDNS(k8s-master1)
```yaml
cat >coredns.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. In order to make Addon Manager do not reconcile this replicas parameter.
  # 2. Default is 1.
  # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        beta.kubernetes.io/os: linux
      containers:
      - name: coredns
        image: k8s.gcr.io/coredns:1.7.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 4G
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.0.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP

EOF

kubectl apply -f coredns.yaml 

```
查看pod状态
```bash
kubectl get pods -n kube-system
```
DNS解析测试

```
kubectl run -it --rm dns-test --image=busybox:1.28.4 sh

nslookup kubernetes
```

正确解析示例
```
Server:    10.0.0.2
Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local

```

## Master扩容（k8s-master2/3将以master角色加入k8s集群）
### 拷贝文件（k8s-master1）
```
scp -r /opt/kubernetes root@$k8s-master2_ip:/opt
scp -r /opt/cni/ root@$k8s-master2_ip:/opt
scp -r /opt/etcd/ssl root@$k8s-master2_ip:/opt/etcd
scp /usr/lib/systemd/system/kube* root@$k8s-master2_ip:/usr/lib/systemd/system
scp /usr/bin/kubectl  root@$k8s-master2_ip:/usr/bin


scp -r /opt/kubernetes root@$k8s-master3_ip:/opt
scp -r /opt/cni/ root@$k8s-master3_ip:/opt
scp -r /opt/etcd/ssl root@$k8s-master3_ip:/opt/etcd
scp /usr/lib/systemd/system/kube* root@$k8s-master3_ip:/usr/lib/systemd/system
scp /usr/bin/kubectl  root@$k8s-master3_ip:/usr/bin

```
### 删除证书文件(k8s-master2/3)
```
rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
rm -f /opt/kubernetes/ssl/kubelet*

```
### 修改配置文件IP和主机名（k8s-master2/3）
```
vi /opt/kubernetes/cfg/kube-apiserver.conf 
    ...
    --bind-address=10.1.1.230 \ #修改为本地ip
    --advertise-address=10.1.1.230 \  #修改为本地ip
    ...

vi /opt/kubernetes/cfg/kubelet.conf
    --hostname-override=k8s-master2 #修改为本地主机名

vi /opt/kubernetes/cfg/kube-proxy-config.yml
    hostnameOverride: k8s-master2 #修改为本地主机名

```
### 启动并设置开机启动(k8s-master2/3)
```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kube-apiserver
systemctl enable kube-controller-manager
systemctl enable kube-scheduler
systemctl enable kubelet
systemctl enable kube-proxy

```
### 批准kubelet证书申请(k8s-master1)
```
kubectl get csr

kubectl certificate approve $name

```
## kube-apiserver高可用
### 安装必要软件(k8s-lb1/2)
```
yum install epel-release -y
yum install nginx keepalived -y

```
### Nginx配置文件（k8s-lb1/2）
```
cat > /etc/nginx/nginx.conf << "EOF"
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server $k8s-master1_ip:6443;   # Master1 APISERVER IP:PORT
       server $k8s-master2_ip:6443;   # Master2 APISERVER IP:PORT
       server $k8s-master3_ip:6443;   # Master3 APISERVER IP:PORT
    }
    
    server {
       listen 6443;
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
        }
    }
}
EOF

```
### keepalived配置文件（k8s-lb1）
```
cat > /etc/keepalived/keepalived.conf << EOF
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_MASTER
} 
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}
vrrp_instance VI_1 { 
    state MASTER 
    interface ens33
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 100    # 优先级，备服务器设置 90 
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    # 虚拟IP
    virtual_ipaddress { 
        $vip/24
    } 
    track_script {
        check_nginx
    } 
}
EOF

```
### keepalived配置文件（k8s-lb2）
```
cat > /etc/keepalived/keepalived.conf << EOF
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_BACKUP
} 
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}
vrrp_instance VI_1 { 
    state BACKUP 
    interface ens33
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 90
    advert_int 1
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    virtual_ipaddress { 
        $vip/24
    } 
    track_script {
        check_nginx
    } 
}
EOF

```
### 运行脚本(k8s-lb1/2)
```
cat > /etc/keepalived/check_nginx.sh  << "EOF"
#!/bin/bash
count=$(ps -ef |grep nginx |egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi
EOF

chmod +x /etc/keepalived/check_nginx.sh

```

### 启动并设置开机启动(k8s-lb1/2)
```
systemctl daemon-reload
systemctl start nginx
systemctl start keepalived
systemctl enable nginx
systemctl enable keepalived

```
### 查看keepalived工作状态(k8s-lb1)
```
ip a
```
### kube-apiserver高可用测试
k8s-lb1执行
```
pkill nginx
```
k8s-lb1执行
```
ip a
```
查看是否绑定vip

### 集群内任一节点测试
```
curl -k https://$vip:6443/version
```

## 修改worker 连接LB VIP(k8s-master1/2/3,k8s-node1/2)
```
sed -i 's#$k8s-master1_ip:6443#$vip:6443#' /opt/kubernetes/cfg/*
systemctl restart kubelet
systemctl restart kube-proxy
```


## 安装ingress
```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.35.0/deploy/static/provider/cloud/deploy.yaml

sed -i "s|k8s.gcr.io/ingress-nginx/controller:v0.35.0@sha256:fc4979d8b8443a831c9789b5155cded454cb7de737a8b727bc2ba0106d2eae8b|scofield/ingress-nginx-controller:v0.35.0|" deploy.yaml

kubectl apply -f deploy.yaml
```


# Kubefate(v1.4.4)
假设有两个k8s集群，分别为A方， B方。

A方信息如下

## KubeFATE部署(AB双方master主机执行)
### 新建kubefate文件夹（以后执行kubefate命令需要在此文件夹下）下载kubefate并解压
```
cd ~
mkdir kubefate

cd kubefate && wget https://github.com/FederatedAI/KubeFATE/releases/download/v1.4.4/kubefate-k8s-1.4.4.tar.gz

tar -zxvf kubefate-k8s-1.4.4.tar.gz
```
### 安装RBAC(AB双方master主机执行)
```
kubectl apply -f ./rbac-config.yaml
```

### 安装kubefate(AB双方master主机执行)
```
sed -i "147s|8080|80|" kubefate.yaml
kubectl apply -f ./kubefate.yaml
```
### 创建命名空间(AB双方master主机执行)
A方 master

```
kubectl create namespace fate-10000
````

B方 master

````
kubectl create namespace fate-9999
````

### 检查pod状态(AB双方master主机执行)
```
kubectl get pod -n kube-fate
````

当所有pod状态均为runnging时执行下一步

### 配置cluster信息
A方 master

```
sed -i "s|registry: \"\"|registry: \"hub.c.163.com/federatedai\"|" cluster.yaml

sed -i "1,5s|9999|10000|" cluster.yaml
sed -i "21s|30009|30010|" cluster.yaml
sed -i "26s|10000|9999|" cluster.yaml
sed -i "27s|192.168.10.1|192.168.92.130|" cluster.yaml
sed -i "28s|30010|30009|" cluster.yaml
sed -i '22,24d' cluster.yaml
```

B方 master
```
sed -i "s|registry: \"\"|registry: \"hub.c.163.com/federatedai\"|" cluster.yaml

sed -i "27s|192.168.10.1|192.168.92.128|" cluster.yaml
sed -i '22,24d' cluster.yaml
```
### kubefate移入到服务(AB双方master主机执行)
```
chmod +x ./kubefate && sudo mv ./kubefate /usr/local/bin/kubefate
```
### 获取kubefate的cluster ip(双方分别执行)
```
kubectl get svc -n kube-fate
```

假设得到ip 为$ip, 执行

```
echo $ip kubefate.net >> /etc/hosts
```

### 开始部署(AB双方master主机执行)
```
kubefate cluster install -f ./cluster.yaml
```

### 查看部署进度(AB双方master主机执行)
```
kubefate job ls
```

当状态为sucess时部署完成


### bug修复

通过修改deployment/python修正1.5.0版本的bug

```
# 进入deployment/python开始修正
kubectl edit deployment python -n fate-10000
```

针对以下部分进行修正

```
        image: hub.c.163.com/federatedai/client:1.5.0-release
        imagePullPolicy: IfNotPresent
        name: client
        ports:
        - containerPort: 20000
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
```

修正后

```
        image: hub.c.163.com/federatedai/client:1.5.0-release
        imagePullPolicy: IfNotPresent
        name: client
        command: ["/bin/bash"]
        args: ["-c", "flow init --ip=${POD_IP} --port=9380 && pipeline init --ip=${POD_IP} --port=9380 && jupyter notebook --ip=0.0.0.0 --port=20000 --allow-root --debug --NotebookApp.notebook_dir='/fml_manager/Examples' --no-browser --NotebookApp.token='' --NotebookApp.password=''"]
        ports:
        - containerPort: 20000
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
```

## 



## 测试toy_example
### 双方分别进入容器
A方 master

```
kubectl exec -it svc/fateflow -c python -n fate-10000 -- bash
```

B方 master

```
kubectl exec -it svc/fateflow -c python -n fate-9999 -- bash
```

### 开始测试
A方master执行
```
cd examples/toy_example/
python run_toy_example.py 10000 9999 1
```
# Kubefate(v1.5.0)
假设有两个k8s集群，分别为A方， B方。

A方信息如下

## KubeFATE部署(AB双方master主机执行)
### 新建kubefate文件夹（以后执行kubefate命令需要在此文件夹下）下载kubefate并解压
```
cd ~
mkdir kubefate

cd kubefate && wget https://github.com/FederatedAI/KubeFATE/releases/download/v1.5.0/kubefate-k8s-v1.5.0.tar.gz

tar -zxvf kubefate-k8s-v1.5.0.tar.gz
```
### 安装RBAC(AB双方master主机执行)
```
kubectl apply -f ./rbac-config.yaml
```

### 安装kubefate(AB双方master主机执行)
```
sed -i "147s|8080|80|" kubefate.yaml
kubectl apply -f ./kubefate.yaml
```
### 创建命名空间(AB双方master主机执行)
A方 master

```
kubectl create namespace fate-10000
```

B方 master

````
kubectl create namespace fate-9999
````

### 检查pod状态(AB双方master主机执行)
```
kubectl get pod -n kube-fate
```

当所有pod状态均为runnging时执行下一步

### 配置cluster信息
A方 master

```
sed -i "s|registry: \"\"|registry: \"hub.c.163.com/federatedai\"|" cluster.yaml

sed -i "1,5s|9999|10000|" cluster.yaml
sed -i "21s|30009|30010|" cluster.yaml
sed -i "26s|10000|9999|" cluster.yaml
sed -i "27s|192.168.10.1|192.168.92.130|" cluster.yaml
sed -i "28s|30010|30009|" cluster.yaml
sed -i '22,24d' cluster.yaml
```

B方 master
```
sed -i "s|registry: \"\"|registry: \"hub.c.163.com/federatedai\"|" cluster.yaml

sed -i "27s|192.168.10.1|192.168.92.128|" cluster.yaml
sed -i '22,24d' cluster.yaml
```
### kubefate移入到服务(AB双方master主机执行)
````
chmod +x ./kubefate && sudo mv ./kubefate /usr/local/bin/kubefate
````
### 获取kubefate的cluster ip(双方分别执行)
```
kubectl get svc -n kube-fate
```

假设得到ip 为$ip, 执行

```
echo $ip kubefate.net >> /etc/hosts
```
### 修正mariadb
进入mariadb容器
```
kubectl exec -it svc/mariadb -n kube-fate -- bash
```
登录mysql
```
mysql -u kubefate -p
```
密码:kubefate


修正
```
use kube_fate;
ALTER TABLE helm_charts MODIFY templates mediumblob;
```
退出 mysql
```
exit
```
退出容器
```
ctrl + A +D
```
### 开始部署(AB双方master主机执行)
```
kubefate cluster install -f ./cluster.yaml
```

### 查看部署进度(AB双方master主机执行)
```
kubefate job ls
```

当状态为sucess时部署完成

## 测试toy_example
### 双方分别进入容器
A方 master

```bash
kubectl exec -it svc/fateflow -c python -n fate-10000 -- bash
```

B方 master

```bash
kubectl exec -it svc/fateflow -c python -n fate-9999 -- bash
```

### 开始测试
A方master执行
```
cd examples/toy_example/
python run_toy_example.py 10000 9999 1
```

## flow不能用的问题，解决办法

```
localedef -v -c -i en_US -f UTF-8 en_US.UTF-8
echo "export LC_ALL=en_US.utf-8" >> /etc/profile
source /etc/profile
```



# 参考

https://blog.csdn.net/weixin_30254435/article/details/102045117

# 附录
## 安装Helm
### 安装epel
`yum install -y epel-release`
### 安装snapd
`yum install -y snapd`
### 添加snap启动通信socket
`systemctl enable --now snapd.socket`
### 创建软链接
`ln -s /var/lib/snapd/snap /snap`
### 安装helm
`snap install helm --classic`

## 重启后kube-apiserver, etcd未启动
```
systemctl daemon-reload
systemctl restart docker
```
## 更改服务配置
```
kubectl edit service_name -n namespace_name
```
## 后期加入联邦学习的其他方
```
kubectl edit cm rollsite-config -n fate-10000
```

## k8s复制文件

```
kubectl cp $pods_name:$dir -c $container $destinction_dir -n $namespace
# example
kubectl cp python-5594d88c57-5lnxn:/data/projects/fate/logs/202011200231350021282 -c python ./log -n fate-10000
```

## stop_job

```
python fate_flow_client.py -f stop_job -j $job_id
```

## python pod故障重建容器
python pod出现故障后，一般会自动新建pod, 但原先的pod 为terminating状态，并导致pipeline不能正常使用使用以下命令立即删除故障pod
```
kubectl delete  --force pod $pod_name -n $fate_namespace
```
以后在notebook里重新启动kernel
## 安装NFS

在k8s-node2上安装nfs server

```bash
yum install -y nfs-utils rpcbind
mkdir /var/nfs -p
echo "/var/nfs 192.168.92.0/24(rw,sync,no_subtree_check,no_root_squash)" > /etc/exports
chown nfsnobody //var/nfs
systemctl start nfs
systemctl enable nfs
```

## 创建StorageClass

rbac

```bash
cd ~
mkdir nfs
cd nfs
```



```
cat > rbac.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default        #根据实际环境设定namespace,下面类同
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
    # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
EOF
```

创建NFS资源的StorageClass

```
cat > nfs-StorageClass.yaml << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: qgg-nfs-storage #这里的名称要和provisioner配置文件中的环境变量PROVISIONER_NAME保持一致
parameters:
  archiveOnDelete: "false"
reclaimPolicy: Retain
EOF
```

创建NFS provisioner

```
cat > nfs-provisioner.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default  #与RBAC文件中的namespace保持一致
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: qgg-nfs-storage  #provisioner名称,请确保该名称与 nfs-StorageClass.yaml文件中的provisioner名称保持一致
            - name: NFS_SERVER
              value: $k8s-node2_ip  #NFS Server IP地址
            - name: NFS_PATH  
              value: /var/nfs    #NFS挂载卷
      volumes:
        - name: nfs-client-root
          nfs:
            server: $k8s-node2_ip  #NFS Server IP地址
            path: /var/nfs     #NFS 挂载卷
EOF            
```

为mysql创建pvc

```
cat > mysql-pvc.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-data
  namespace: fate-8888
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"   #与nfs-StorageClass.yaml metadata.name保持一致
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
EOF

kubectl replace --force -f mysql-pve.yaml
```

mysql与nfs有冲突，需要设置mysql不使用pvc

```
kubectl edit deployment mysql -n fate-8888
# 找到pvc对应的volume 替换为
- emptyDir: {}
name: data
```

为nodemanager0开启pvc

```
cat > nodemanager0-pvc.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nodemanager-0-data
  namespace: fate-8888
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"   #与nfs-StorageClass.yaml metadata.name保持一致
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
EOF

kubectl replace --force -f nodemanager0-pvc.yaml
```

为nodemanager1开启pvc

```
cat > nodemanager1-pvc.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nodemanager-1-data
  namespace: fate-8888
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"   #与nfs-StorageClass.yaml metadata.name保持一致
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
EOF

kubectl replace --force -f nodemanager1-pvc.yaml
```

为notebook开启持久化存储

```
kubectl edit deployment python -n fate-10000
# 在Volume里增加
      - name: notebook-dir
        persistentVolumeClaim:
          claimName: notebook-data
# 在client镜像volumeMount中添加
        - mountPath: /fml_manager/Examples
          name: notebook-dir
```

构建python-pvc

```
cat > python-pvc.yaml << EOF
apiVersion: v1
metadata:
  name: notebook-data
  namespace: fate-10000
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"   #与nfs-StorageClass.yaml metadata.name保持一致
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
EOF

kubectl replace --force -f python-pvc.yaml
```

## model-local-cache保存
```
kubectl edit deployment python -n fate-10000
```
在volumes加入
```
      - name: model-local-cache
        persistentVolumeClaim:
          claimName: model-data

```
python容器中挂载
```
        - mountPath: /data/projects/fate/model_local_cache/
          name: model-local-cache
```

## etcd备份
k8s-master1
```
cat > /opt/etcd/bak.sh << EOF
#!/bin/bash

if [ -d "/data/etcd_bak" ]; then
    echo "the folder is exists"
else
    mkdir -p "/data/etcd_bak"
    echo "the folder is created"
fi

if [ ! -f "/data/etcd_bak/etcd1.db" ]; then 
    echo "Creating the first db"
    ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master1_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save /data/etcd_bak/etcd1.db
else
    if [ ! -f "/data/etcd_bak/etcd2.db" ]; then
        echo "Creating the second db"
        ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master1_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save  /data/etcd_bak/etcd2.db
    else
        if [ ! -f "/data/etcd_bak/etcd3.db" ]; then
            echo "Creating the third db"
            ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master1_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save /data/etcd_bak/etcd3.db
        else
            mv /data/etcd_bak/etcd2.db /data/etcd_bak/etcd1.db
            mv /data/etcd_bak/etcd3.db /data/etcd_bak/etcd2.db
            ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master1_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save /data/etcd_bak/etcd3.db
        fi
    fi
fi
EOF

chmod +x /opt/etcd/bak.sh
```
k8s-master2
```
cat > /opt/etcd/bak.sh << EOF
#!/bin/bash

if [ -d "/data/etcd_bak" ]; then
    echo "the folder is exists"
else
    mkdir -p "/data/etcd_bak"
    echo "the folder is created"
fi

if [ ! -f "/data/etcd_bak/etcd1.db" ]; then 
    echo "Creating the first db"
    ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master2_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save /data/etcd_bak/etcd1.db
else
    if [ ! -f "/data/etcd_bak/etcd2.db" ]; then
        echo "Creating the second db"
        ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master2_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save  /data/etcd_bak/etcd2.db
    else
        if [ ! -f "/data/etcd_bak/etcd3.db" ]; then
            echo "Creating the third db"
            ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master2_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save /data/etcd_bak/etcd3.db
        else
            mv /data/etcd_bak/etcd2.db /data/etcd_bak/etcd1.db
            mv /data/etcd_bak/etcd3.db /data/etcd_bak/etcd2.db
            ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="https://$k8s-master2_ip:2379" --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --cacert=/opt/etcd/ssl/ca.pem snapshot save /data/etcd_bak/etcd3.db
        fi
    fi
fi
EOF

chmod +x /opt/etcd/bak.sh
```
k8s-master1/2
```
crontab -e

*/15 * * * * /opt/etcd/bak.sh
```
