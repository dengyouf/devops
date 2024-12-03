# Kubeadm 安装 Kubernetes

基于 Kubeadm 部署Kubernetes集群。操作系统为 Ubuntu 20.04 LTS，用到的各相关程序版本如下：

- kubernetes: v1.28.6
- docker: 20.10.22
- cri-dockerd: v0.3.8
- cni: flannel

**环境说明**

| 主机地址 | 节点名称 | 角色 |
| --- | --- | --- |
| 192.168.122.11 | k8s-master01 | master |
| 192.168.122.21 | k8s-worker01 | worker |
| 192.168.122.22 | k8s-worker02 | worker |
| 192.168.122.23 | k8s-worker03 | worker |

## 一、准备虚拟机

### 1.1 初始化虚拟机

- 允许root用户远程登陆
```
~# echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
~# systemctl  restart sshd
~# passwd root
```

- 更新apt源为阿里源
```
cat > /etc/apt/sources.list <<"EOF"
deb https://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

# deb https://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
# deb-src https://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
EOF
```

- 关闭防火墙
```
~# ufw  disable
```

- 关闭swap分区
```
~# sed  -ri  's@/.*swap.*@# &@' /etc/fstab && swapoff -a
```

- 调整时区和时间同步
```
~# sudo cp /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
~# date -R

~# sudo apt update
~# sudo apt install -y ntpdate
~# /usr/sbin/ntpdate ntp.aliyun.com 2&>1 /dev/null

~# crontab -l
*/3 * * * * /usr/sbin/ntpdate ntp.aliyun.com 2&>1 /dev/null
```

- 打开内核转发
```
~# cat > /etc/sysctl.d/k8s.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
net.ipv4.tcp_tw_reuse = 0
net.core.somaxconn = 32768
net.netfilter.nf_conntrack_max=1000000
vm.swappiness = 0
vm.max_map_count=655360
fs.file-max=6553600
EOF

~# sysctl -p

~# cat >> /etc/modules-load.d/k8s.conf << "EOF"
overlay
br_netfilter
EOF
~# sudo modprobe overlay
~# sudo modprobe br_netfilter
```
- 加载 IPVS 模块
- 
```shell
apt install ipvsadm ipset -y

cat > /etc/modules-load.d/ipvs.conf << "EOF"
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir | grep -o "^[^.]*"); do
    /sbin/modinfo -F filename $i  &> /dev/null
    if [ $? -eq 0 ]; then
        /sbin/modprobe $i
    fi
done
EOF

bash /etc/modules-load.d/ipvs.conf

~#  lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

- 关机
```
~# init 0
```

### 1.2. 克隆虚拟机

- 查看模板虚拟机状态
```
~]# virsh  list --all
 Id    Name                           State
----------------------------------------------------
 -     ubuntu20.04                    shut off
```
- 克隆虚拟机
```
~]# virt-clone --auto-clone -o ubuntu20.04 -n k8s-master01
~]# virt-clone --auto-clone -o ubuntu20.04 -n k8s-worker01
~]# virt-clone --auto-clone -o ubuntu20.04 -n k8s-worker02
~]# virt-clone --auto-clone -o ubuntu20.04 -n k8s-worker03
```
- 修改主机IP地址和主机名
```
~]# virt-sysprep  \
--operations defaults,machine-id,-ssh-userdir,-lvm-uuids \
--hostname k8s-master01 \
--run-command "sed -i 's@192.168.122.7@192.168.122.11@g' /etc/netplan/00-installer-config.yaml && dpkg-reconfigure openssh-server" \
-d k8s-master01
~]# virt-sysprep  --operations defaults,machine-id,-ssh-userdir,-lvm-uuids \
--hostname k8s-worker01 \
--run-command "sed -i 's@192.168.122.7@192.168.122.21@g' /etc/netplan/00-installer-config.yaml && dpkg-reconfigure openssh-server" \
-d k8s-worker01
~]# virt-sysprep  --operations defaults,machine-id,-ssh-userdir,-lvm-uuids \
--hostname k8s-worker02 \
--run-command "sed -i 's@192.168.122.7@192.168.122.22@g' /etc/netplan/00-installer-config.yaml && dpkg-reconfigure openssh-server" \
-d k8s-worker02
~]# virt-sysprep  --operations defaults,machine-id,-ssh-userdir,-lvm-uuids \
--hostname k8s-worker03 \
--run-command "sed -i 's@192.168.122.7@192.168.122.23@g' /etc/netplan/00-installer-config.yaml && dpkg-reconfigure openssh-server" \
-d k8s-worker03
```

### 1.3. 启动虚拟机

```
~]# virsh start k8s-master01
~]# virsh start k8s-worker01
~]# virsh start k8s-worker02
~]# virsh start k8s-worker03
```

## 二、安装集群

### 2.1 主机名解析

```
~# cat  >> /etc/hosts <<EOF
192.168.122.11 k8s-master01 k8s-master01.linux.io
192.168.122.21 k8s-worker01 k8s-worker01.linux.io
192.168.122.22 k8s-worker02 k8s-worker02.linux.io
192.168.122.23 k8s-worker03 k8s-worker03.linux.io

192.168.122.11 k8s.linux.io
EOF
```

### 2.2 安装DOCKER

- 配置 apt 源
```
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# step 2: 信任 Docker 的 GPG 公钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Step 3: 写入软件源信息
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

- 安装docker
```
sudo apt-get update
sudo apt-cache madison docker-ce
sudo apt install -y docker-ce=5:20.10.22~3-0~ubuntu-focal
```
- 配置docker加速器
```
mkdir -pv /etc/docker
sudo cat > /etc/docker/daemon.json <<-'EOF'
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://docker.rainbond.cc"
    ]

}
EOF
systemctl restart docker
```

### 2.3 安装 cri-dockerd

```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.8/cri-dockerd-0.3.8.amd64.tgz
tar -xzvf cri-dockerd-0.3.8.amd64.tgz
sudo install -m 0755 -o root -g root -t /usr/local/bin cri-dockerd/cri-dockerd

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket

sudo install cri-docker.service /etc/systemd/system
sudo install cri-docker.socket /etc/systemd/system
sudo sed -i -e 's@/usr/bin/cri-dockerd@/usr/local/bin/cri-dockerd@' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.socket
sudo systemctl start cri-docker.service && systemctl status cri-docker.service
```

### 2.4 安装 kubeadm、kubelet 和 kubectl

- 配置kubernetes源
```
apt-get update && apt-get install -y apt-transport-https
sudo mkdir -m 755 /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-cache madison kubeadm

apt install -y kubeadm=1.28.6-1.1 kubelet=1.28.6-1.1 kubectl=1.28.6-1.1
```

## 三、初始化集群

### 3.1 拉取镜像

```
~]# kubeadm config images pull --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
--kubernetes-version=1.28.6 \
--cri-socket=unix:///var/run/cri-dockerd.sock

[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.28.6
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.28.6
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.28.6
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.28.6
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.10-0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.10.1
```

### 3.2 初始化 master

- 配置cri-docker中初始化容器


```
~# cp /etc/systemd/system/cri-docker.service{,.bak}
~# vim  /etc/systemd/system/cri-docker.service
[Unit]
...
[Service]
Type=notify
#ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd://
ExecStart=/usr/local/bin/cri-dockerd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9 --container-runtime-endpoint fd://

~# systemctl  daemon-reload && systemctl  restart  cri-docker.service && systemctl  status cri-docker.service

for i in k8s-worker01 k8s-worker02 k8s-worker03;do
    scp  /etc/systemd/system/cri-docker.service root@$i:/etc/systemd/system/cri-docker.service
    ssh $i "systemctl  daemon-reload && systemctl  restart  cri-docker.service && systemctl  status cri-docker.service"
done
```

- 初始化master

```
# 所有节点需要解析 k8s.linux.io, 此域名用于后续扩展集群为高可用集群
~]# kubeadm init --kubernetes-version=v1.28.6 \
    --control-plane-endpoint=k8s.linux.io \
    --apiserver-advertise-address=0.0.0.0 \
    --pod-network-cidr=10.244.0.0/16   \
    --service-cidr=10.96.0.0/12 \
    --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
    --ignore-preflight-errors=Swap \
    --cri-socket=unix:///var/run/cri-dockerd.sock | tee kubeadm-init.log
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join k8s.linux.io:6443 --token 40zawh.qty5miole7ny5hf8 \
        --discovery-token-ca-cert-hash sha256:832dd459e1bf101d54f6ff80bec406f468aa9cd4b359c9536c845d696fdc8f21 \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s.linux.io:6443 --token 40zawh.qty5miole7ny5hf8 \
        --discovery-token-ca-cert-hash sha256:832dd459e1bf101d54f6ff80bec406f468aa9cd4b359c9536c845d696fdc8f21
```

- 配置kubectl
```

~# mkdir -p $HOME/.kube
~#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
~#   sudo chown $(id -u):$(id -g) $HOME/.kube/config
~# kubectl  get nodes
NAME           STATUS     ROLES           AGE    VERSION
k8s-master01   NotReady   control-plane   113s   v1.28.6
```

### 3.2 加入 Worker
```
kubeadm join k8s.linux.io:6443 --token 40zawh.qty5miole7ny5hf8 \
        --discovery-token-ca-cert-hash sha256:832dd459e1bf101d54f6ff80bec406f468aa9cd4b359c9536c845d696fdc8f21 \
        --cri-socket=unix:///var/run/cri-dockerd.sock

~# kubectl  get nodes
NAME           STATUS     ROLES           AGE     VERSION
k8s-master01   NotReady   control-plane   9m28s   v1.28.6
k8s-worker01   NotReady   <none>          2m33s   v1.28.6
k8s-worker02   NotReady   <none>          2m30s   v1.28.6
k8s-worker03   NotReady   <none>          2m27s   v1.28.6

~# kubectl  get pod  -A
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
kube-system   coredns-6554b8b87f-k9jxl               0/1     Pending   0          9m17s
kube-system   coredns-6554b8b87f-tfkv8               0/1     Pending   0          9m17s
kube-system   etcd-k8s-master01                      1/1     Running   0          9m29s
kube-system   kube-apiserver-k8s-master01            1/1     Running   0          9m29s
kube-system   kube-controller-manager-k8s-master01   1/1     Running   0          9m29s
kube-system   kube-proxy-fk5jk                       1/1     Running   0          2m40s
kube-system   kube-proxy-hzqdp                       1/1     Running   0          9m17s
kube-system   kube-proxy-l8grw                       1/1     Running   0          2m43s
kube-system   kube-proxy-vgn8b                       1/1     Running   0          2m37s
kube-system   kube-scheduler-k8s-master01            1/1     Running   0          9m29s
```

### 3.3 安装网络插件

- 安装

```
~# wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
~# grep -C 5 244 kube-flannel.yml
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
~# grep image kube-flannel.yml
        image: docker.io/flannel/flannel:v0.26.1
        image: docker.io/flannel/flannel-cni-plugin:v1.5.1-flannel2
        image: docker.io/flannel/flannel:v0.26.1

~# kubectl  apply -f kube-flannel.yml
~# kubectl  get nodes
NAME           STATUS   ROLES           AGE     VERSION
k8s-master01   Ready    control-plane   15m     v1.28.6
k8s-worker01   Ready    <none>          8m40s   v1.28.6
k8s-worker02   Ready    <none>          8m37s   v1.28.6
k8s-worker03   Ready    <none>          8m34s   v1.28.6
~# kubectl  get pod  -A -o wide
NAMESPACE      NAME                                   READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
kube-flannel   kube-flannel-ds-h8zrm                  1/1     Running   0          102s    192.168.122.21   k8s-worker01   <none>           <none>
kube-flannel   kube-flannel-ds-l8cgv                  1/1     Running   0          102s    192.168.122.23   k8s-worker03   <none>           <none>
kube-flannel   kube-flannel-ds-q2nk2                  1/1     Running   0          102s    192.168.122.11   k8s-master01   <none>           <none>
kube-flannel   kube-flannel-ds-tg7r9                  1/1     Running   0          102s    192.168.122.22   k8s-worker02   <none>           <none>
kube-system    coredns-6554b8b87f-k9jxl               1/1     Running   0          15m     10.244.0.3       k8s-master01   <none>           <none>
kube-system    coredns-6554b8b87f-tfkv8               1/1     Running   0          15m     10.244.0.2       k8s-master01   <none>           <none>
kube-system    etcd-k8s-master01                      1/1     Running   0          15m     192.168.122.11   k8s-master01   <none>           <none>
kube-system    kube-apiserver-k8s-master01            1/1     Running   0          15m     192.168.122.11   k8s-master01   <none>           <none>
kube-system    kube-controller-manager-k8s-master01   1/1     Running   0          15m     192.168.122.11   k8s-master01   <none>           <none>
kube-system    kube-proxy-fk5jk                       1/1     Running   0          8m45s   192.168.122.22   k8s-worker02   <none>           <none>
kube-system    kube-proxy-hzqdp                       1/1     Running   0          15m     192.168.122.11   k8s-master01   <none>           <none>
kube-system    kube-proxy-l8grw                       1/1     Running   0          8m48s   192.168.122.21   k8s-worker01   <none>           <none>
kube-system    kube-proxy-vgn8b                       1/1     Running   0          8m42s   192.168.122.23   k8s-worker03   <none>           <none>
kube-system    kube-scheduler-k8s-master01            1/1     Running   0          15m     192.168.122.11   k8s-master01   <none>           <none>
```

- 调整为ipvs模式

```shell
kubectl  edit cm kube-proxy -n kube-system
    mode: "ipvs" 

kubectl  delete pod -n kube-system -l k8s-app=kube-proxy
```

## 四、验证集群

### 4.1 验证网络

- 创建 Deployment 和 Service 资源
```
~# kubectl  create deployment myapp --image=ikubernetes/myapp:v1 --replicas=3
~# kubectl  get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
myapp-5d9c4b4647-7xmkx   1/1     Running   0          31s   10.244.2.2   k8s-worker02   <none>           <none>
myapp-5d9c4b4647-qrkdl   1/1     Running   0          31s   10.244.1.2   k8s-worker01   <none>           <none>
myapp-5d9c4b4647-w9lq7   1/1     Running   0          31s   10.244.3.2   k8s-worker03   <none>           <none>

~# kubectl  expose deployment/myapp --type=NodePort --port=80 --target-port=80
~# kubectl  get svc myapp -o wide
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
myapp   NodePort   10.106.198.16   <none>        80:30940/TCP   25s   app=myapp
```

- 通过 Service 或者 NodePort 资源轮询 Pod
```
~# for i in `seq 5`;do curl 10.106.198.16/hostname.html;done
myapp-5d9c4b4647-w9lq7
myapp-5d9c4b4647-qrkdl
myapp-5d9c4b4647-qrkdl
myapp-5d9c4b4647-qrkdl
myapp-5d9c4b4647-7xmkx

~]# for i in `seq 5`; do curl 192.168.122.11:30940/hostname.html;done
myapp-5d9c4b4647-w9lq7
myapp-5d9c4b4647-w9lq7
myapp-5d9c4b4647-w9lq7
myapp-5d9c4b4647-qrkdl
myapp-5d9c4b4647-7xmkx

```

> 注意：Service IP 和 NodePort 仅是网络规则，所以不支持ping， 但是支持telnet

### 4.1 验证DNS

```
~# kubectl  get svc -n kube-system -o wide
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE   SELECTOR
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   29m   k8s-app=kube-dns


~# kubectl  run busybox --image=busybox:1.28 -- sleep 3600
~# kubectl  get pod/busybox
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          16s

~# kubectl  exec -it busybox -- nslookup myapp.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      myapp.default.svc.cluster.local
Address 1: 10.106.198.16 myapp.default.svc.cluster.local
```

## 五、插件安装

### 5.1 Metrics-Server

> 官网： `https://github.com/kubernetes-sigs/metrics-server`

```shell
~# wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
~# vim components.yaml
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
   
~# kubectl  apply -f components.yaml 
```
```shell
~# kubectl  top nodes 
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01   189m         4%     1766Mi          22%       
k8s-worker01   61m          1%     964Mi           12%       
k8s-worker02   59m          1%     913Mi           11%       
k8s-worker03   61m          1%     937Mi           12%       
~# kubectl  top pods
NAME                     CPU(cores)   MEMORY(bytes)   
busybox                  0m           1Mi             
myapp-5d9c4b4647-4t4sp   0m           2Mi             
myapp-5d9c4b4647-fb4kh   0m           2Mi             
myapp-5d9c4b4647-zgqtw   0m           2Mi 
```

### 5.2 Dashboard

> 官网：`https://github.com/kubernetes/dashboard`

从 7.0.0 版开始，我们已不再支持基于 Manifest 的安装。目前仅支持基于 Helm 的安装。

```shell

```