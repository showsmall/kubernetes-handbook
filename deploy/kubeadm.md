# kubeadm

> Kubernetes一键部署脚本（使用docker运行时）

```sh
# on master
git clone https://github.com/feiskyer/ops
cd ops
kubernetes/install-kubernetes.sh
# 记住控制台输出的TOEKN和MASTER地址，在其他Node安装时会用到

# on node
git clone https://github.com/feiskyer/ops
cd ops
export TOKEN=xxxxx
export MASTER_IP=xx.xx.xx.xx
kubernetes/add-docker-node.sh
```

以下是详细的安装步骤。

## 初始化系统

所有机器都需要初始化docker和kubelet。

### ubuntu

```sh
# for ubuntu 16.04+
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update

# Install docker if you don't have it already.
apt-get install -y docker.io
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet
```

### centos

```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

setenforce 0
yum install -y docker kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet
```

国内用户也可以使用阿里云的镜像来安装

```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF
```

## 安装master

```sh
# --api-advertise-addresses <ip-address>
# for flannel, setup --pod-network-cidr 10.244.0.0/16
kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version latest

# enable schedule pods on the master
export KUBECONFIG=/etc/kubernetes/admin.conf
# for v1.5-, use kubectl taint nodes --all dedicated-
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```

如果需要修改kubernetes服务的配置选项，则需要创建一个MasterConfiguration配置文件，其格式为

```yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: <address|string>
  bindPort: <int>
etcd:
  endpoints:
  - <endpoint1|string>
  - <endpoint2|string>
  caFile: <path|string>
  certFile: <path|string>
  keyFile: <path|string>
networking:
  dnsDomain: <string>
  serviceSubnet: <cidr>
  podSubnet: <cidr>
kubernetesVersion: <string>
cloudProvider: <string>
authorizationModes:
- <authorizationMode1|string>
- <authorizationMode2|string>
token: <string>
tokenTTL: <time duration>
selfHosted: <bool>
apiServerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
controllerManagerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
schedulerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
apiServerCertSANs:
- <name1|string>
- <name2|string>
certificatesDir: <string>
```

比如

```yaml
# cat kubeadm.yml
kind: MasterConfiguration
apiVersion: kubeadm.k8s.io/v1alpha1
kubernetesVersion: "stable"
apiServerCertSANs: []
controllerManagerExtraArgs:
  horizontal-pod-autoscaler-use-rest-clients: "true"
  horizontal-pod-autoscaler-sync-period: "10s"
  node-monitor-grace-period: "10s"
  feature-gates: "AllAlpha=true"
  enable-dynamic-provisioning: "true"
apiServerExtraArgs:
  runtime-config: "api/all=true"
  feature-gates: "AllAlpha=true"
networking:
  podSubnet: "10.244.0.0/16"
```

然后，在初始化master的时候指定kubeadm.yml的路径：

```sh
kubeadm init --config ./kubeadm.yaml
```

## 配置Network plugin

### CNI bridge

```sh
mkdir -p /etc/cni/net.d
cat >/etc/cni/net.d/10-mynet.conf <<-EOF
{
    "cniVersion": "0.3.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.244.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0"  }
        ]
    }
}
EOF
cat >/etc/cni/net.d/99-loopback.conf <<-EOF
{
    "cniVersion": "0.3.0",
    "type": "loopback"
}
EOF
```

### flannel

注意：需要 `kubeadm init` 时设置 `--pod-network-cidr=10.244.0.0/16`

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```

### weave

```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

### calico

注意：需要 `kubeadm init` 时设置 `--pod-network-cidr=192.168.0.0/16`

```sh
kubectl apply -f https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
```

## 添加Node

```sh
token=$(kubeadm token list | grep authentication,signing | awk '{print $1}')
kubeadm join --token $token ${master_ip}
```

跟Master一样，添加Node的时候也可以自定义Kubernetes服务的选项，格式为

```yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: NodeConfiguration
caCertPath: <path|string>
discoveryFile: <path|string>
discoveryToken: <string>
discoveryTokenAPIServers:
- <address|string>
- <address|string>
tlsBootstrapToken: <string>
```

在把Node加入集群的时候，指定NodeConfiguration配置文件的路径

```sh
kubeadm join --config ./nodeconfig.yml --token $token ${master_ip}
```

## 删除安装

```
kubeadm reset
```

## 动态升级

kubeadm v1.8开始支持动态升级，升级步骤为

* 首先上传kubeadm配置，如`kubeadm config upload from-flags [flags]`（使用命令行参数）或 `kubeadm config upload from-file --config [config]`（使用配置文件）
* 在master上检查新版本 `kubeadm upgrade plan`， 当有新版本（如v1.8.0）时，执行 `kubeadm upgrade apply v1.8.0` 升级控制平面
* **手动**升级CNI插件（如果有新版本的话）
* 添加自动证书回滚的RBAC策略 `kubectl create clusterrolebinding kubeadm:node-autoapprove-certificate-rotation --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient --group=system:nodes`
* 最后升级kubelet

```sh
$ kubectl drain $HOST --ignore-daemonsets

# 升级软件包
$ apt-get update
$ apt-get upgrade
# CentOS上面执行yum升级
# $ yum update

$ kubectl uncordon $HOST
```

### 手动升级

kubeadm v1.7以及以前的版本不支持动态升级，但可以手动升级。

#### 升级Master

假设你已经有一个使用kubeadm部署的Kubernetes v1.6集群，那么升级到v1.7的方法为：

1. 升级安装包 `apt-get upgrade && apt-get update`
2. 重启kubelet `systemctl restart kubelet`
3. 删除kube-proxy DaemonSet `KUBECONFIG=/etc/kubernetes/admin.conf kubectl delete daemonset kube-proxy -n kube-system`
4. kubeadm init --skip-preflight-checks --kubernetes-version v1.7.1
5. 更新CNI插件

#### 升级Node

1. 升级安装包 `apt-get upgrade && apt-get update`
2. 重启kubelet `systemctl restart kubelet`

## 安全选项

默认情况下，kubeadm会开启Node客户端证书的自动批准，如果不需要的话可以选择关闭，关闭方法为

```sh
$ kubectl delete clusterrole kubeadm:node-autoapprove-bootstrap
```

关闭后，增加新的Node时，`kubeadm join`会阻塞等待管理员手动批准，匹配方法为

```sh
$ kubectl get csr
NAME                                                   AGE       REQUESTOR                 CONDITION
node-csr-c69HXe7aYcqkS1bKmH4faEnHAWxn6i2bHZ2mD04jZyQ   18s       system:bootstrap:878f07   Pending

$ kubectl certificate approve node-csr-c69HXe7aYcqkS1bKmH4faEnHAWxn6i2bHZ2mD04jZyQ
certificatesigningrequest "node-csr-c69HXe7aYcqkS1bKmH4faEnHAWxn6i2bHZ2mD04jZyQ" approved

$ kubectl get csr
NAME                                                   AGE       REQUESTOR                 CONDITION
node-csr-c69HXe7aYcqkS1bKmH4faEnHAWxn6i2bHZ2mD04jZyQ   1m        system:bootstrap:878f07   Approved,Issued
```

## 参考文档

- [kubeadm参考指南](https://kubernetes.io/docs/admin/kubeadm/)
- [Upgrading kubeadm clusters from 1.7 to 1.8](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm-upgrade-1-8/)
- [Upgrading kubeadm clusters from 1.6 to 1.7](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm-upgrade-1-7/)
