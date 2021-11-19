# eks-distro

此專案用來記錄eks-distro 包括安裝, 更新等等細節


# Installion

此紀錄根據[此文章](https://distro.eks.amazonaws.com/users/install/kubeadm-onsite/)安裝

## Prerequisites

### L4 Load balancer

此節點作為`kube-apiserver`及`ingress`的入口點

1. 關閉 selinux
```sh
setenforce 0; \
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

2. 關閉 firewalld
```sh
systemctl stop firewalld; \
systemctl disable firewalld
```

3. 安裝 HAproxy
```sh
sudo yum install haproxy -y
```

4. 編輯 HAproxy 設定
```sh
vim /etc/haproxy/haproxy.cfg
```

5. 啟動 HAproxy
```sh
systemctl start haproxy; \
systemctl enable haproxy
```



### Each Nodes

下列指令需在每一個節點執行
1. 更新系統
```sh
yum update -y
```

2. 關閉firewall
```sh
systemctl stop firewalld; \
systemctl disable firewalld
```

3. 關閉 selinux
```sh
setenforce 0; \
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

4. 關閉 swap
```sh
swapoff -a; \
echo “vm.swappiness = 0”>> /etc/sysctl.conf  

vim /etc/fstab (comment out swap)

```

5. 安裝 docker 及其他packages
```sh
yum install -y iproute socat conntrack-tools wget nfs-utils yum-utils; \
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo; \
yum install -y docker-ce docker-ce-cli containerd.io; \
systemctl start docker; \
systemctl enable docker
```

6. 新增kubernetes RPM repository  `/etc/yum.repos.d/kubernetes.repo`
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```

7. 安裝 kubeadm, kubectl, kubelet
```sh
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
cd /usr/bin; \
rm -f kubelet kubeadm kubectl

wget https://distro.eks.amazonaws.com/kubernetes-1-21/releases/2/artifacts/kubernetes/v1.21.2/bin/linux/amd64/kubelet; \
wget https://distro.eks.amazonaws.com/kubernetes-1-21/releases/2/artifacts/kubernetes/v1.21.2/bin/linux/amd64/kubeadm; \
wget https://distro.eks.amazonaws.com/kubernetes-1-21/releases/2/artifacts/kubernetes/v1.21.2/bin/linux/amd64/kubectl; \
chmod +x kubeadm kubectl kubelet; \
ls -al kubeadm kubectl kubelet
reboot
```

9. 開機啟動kubelet, 設定config
```sh
systemctl enable kubelet
KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd —network-plugin=cni —pod-infra-container-image=public.ecr.aws/eks-distro/kubernetes/pause:3.4.1"
```

#### Control Plane
1. 下載images
```sh
docker pull public.ecr.aws/eks-distro/kubernetes/pause:v1.21.2-eks-1-21-2; \
docker pull public.ecr.aws/eks-distro/coredns/coredns:v1.8.3-eks-1-21-2; \
docker pull public.ecr.aws/eks-distro/etcd-io/etcd:v3.4.16-eks-1-21-2; \
docker tag public.ecr.aws/eks-distro/kubernetes/pause:v1.21.2-eks-1-21-2 public.ecr.aws/eks-distro/kubernetes/pause:3.4.1; \
docker tag public.ecr.aws/eks-distro/coredns/coredns:v1.8.3-eks-1-21-2 public.ecr.aws/eks-distro/kubernetes/coredns:v1.8.0; \
docker tag public.ecr.aws/eks-distro/etcd-io/etcd:v3.4.16-eks-1-21-2 public.ecr.aws/eks-distro/kubernetes/etcd:3.4.13-0; \
docker images | wc -l
```

3. 新增'/etc/modules-load.d/k8s.conf'
```sh
echo 'br_netfilter' > /etc/modules-load.d/k8s.conf
```

4. 新增'/etc/sysctl.d/99-k8s.conf'
```sh
echo 'net.bridge.bridge-nf-call-iptables = 1' > /etc/sysctl.d/99-k8s.conf
systemctl -p /etc/sysctl.d/99-k8s.conf
```

**Master1**

1. 執行init腳本, 可以設定Serive IP range
```sh
sudo kubeadm init --image-repository public.ecr.aws/eks-distro/kubernetes --kubernetes-version v1.21.2-eks-1-21-2 --service-cidr 172.29.0.0/16 --control-plane-endpoint 'LB:6443'
```

2. 檢查元件
```
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get no
NAME     STATUS     ROLES    AGE   VERSION
k8s-m1   NotReady   master   31s   v1.15.4

kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```
如果你的scheduler, controller-manager不是健康的, 請根據[此連結](https://www.gushiciku.cn/pl/p70v/zh-tw)解決


3. 安裝 CNI

```sh
curl https://docs.projectcalico.org/manifests/calico.yaml -O
# 請根據需求設定pods ip pool, 避免與內網其他服務衝突
kubectl apply -f calico.yaml
```

3. 修復 coredns 權限不足問題
```sh
kubectl edit clusterrole system:coredns
```
新增下列權限
```
- apiGroups:
  - "discovery.k8s.io"
  - "v1beta1"
  resources:
  - endpointslices
  verbs:
  - watch
  - list
```


**Master2＆Master3**

1. 手動拉取憑證
```sh
mkdir -p /etc/kubernetes/pki/etcd
scp root@<MASTER1>:/etc/kubernetes/pki/ca.* /etc/kubernetes/pki/
scp root@<MASTER1>:/etc/kubernetes/pki/sa.* /etc/kubernetes/pki/
scp root@<MASTER1>:/etc/kubernetes/pki/front-proxy-ca.* /etc/kubernetes/pki/
scp root@<MASTER1>:/etc/kubernetes/pki/etcd/ca.* /etc/kubernetes/pki/etcd/
```

2. 加入cluster
```sh
kubeadm join <LB-IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <CERT-HASH> --control-plane
```

#### Worker Nodes

1. 請確定有執行Each Nodes所寫步驟

2. 下載image
```sh
docker pull public.ecr.aws/eks-distro/kubernetes/pause:v1.21.2-eks-1-21-2; \
docker tag public.ecr.aws/eks-distro/kubernetes/pause:v1.21.2-eks-1-21-2 public.ecr.aws/eks-distro/kubernetes/pause:3.4.1
```

3. 將worker node 加入 cluster
```sh
sudo kubeadm join  <LB-IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <CERT-HASH>
```

如果在加入worker node時遇到`kubeadm init /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1`

```sh
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables; \
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
```
