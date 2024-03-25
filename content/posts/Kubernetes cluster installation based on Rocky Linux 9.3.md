---
title: "基于 Rocky Linux 9.3 / openEuler 22.03 LTS SP3 的 Kubernetes 集群搭建" 
date: 2024-02-10T04:18:25+08:00
tags: ["Kubernetes"]
categories: ["Kubernetes"]
draft: false
---

本文修订日期 ：2024 年 3 月 20 日  当前版本 ：1.29.3

## Linux 操作系统环境的安装与配置

目标是做两台 Rocky Linux 9.3 / openEuler 22.03 LTS SP3 虚拟机，一台做 Kubernetes 的控制平面，另一台做计算节点，基本环境最小化安装

### 创建虚拟机

| 硬件配置                                      | 规格                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| Secure boot                                   | Enabled (openEuler 不支持 Secure boot)                                                      |
| RAM（单位： MB）                              | 3072                                                         |
| Disk（单位： GB，固态硬盘）                   | 34 SSD                                                       |
| CPU 核心数量，包括逻辑处理器个数（单位： 个） | 2                                                            |
| VMware 虚拟交换机类型（如果是 VMware 用户）   | 桥接网络，绑定实体机无线网卡 Intel(R) Dual Band Wireless-AC 3165 独占。实体机通过无线网卡上网。 |
| 外部网关                                      | 192.168.1.1 （中国电信宽带 Mesh 组网），同时接入 IPv6        |
| 声卡                                          | VMware 虚拟声卡                                              |
| 显卡                                          | VMware 虚拟显示适配器                                        |
| 芯片组                                        | 默认                                                         |
| 启动类型                                      | UEFI                                                         |

### DVD 启动光盘镜像文件

https://rockylinux.org/zh_CN/download

https://mirrors.tuna.tsinghua.edu.cn/openeuler/openEuler-22.03-LTS-SP3/ISO/x86_64/

###  分区规划

|    磁盘分区    |  挂载点   |                   用途及大小                   | 大小 | 格式  |
| :------------: | :-------: | :--------------------------------------------: | :--: | :---: |
| /dev/nvme0n1p1 | /boot/EFI | ESP启动分区，bootloader 固件，内核镜像所在目录 /boot |  1G  | FAT32 |
| /dev/nvme0n1p3 |     /     |                       根                       | 33G  | EXT4  |

### 网络接口规划

|  主机名  |  网卡  |    IPv4地址     |    网关     |
| :------: | :----: | :-------------: | :---------: |
| master-1 | ens160 | 192.168.1.16/24 | 192.168.1.1 |
|  node-1  | ens160 | 192.168.1.22/24 | 192.168.1.1 |

### 安装一些基本的软件包

```bash
dnf install vim git zsh sqlite wget lsof nano util-linux-user -y
```
### 写给 openEuler 用户的提示

openEuler 通过网络在线安装选项安装源读不出来，看来是openEuler 开发小组没处理好在线安装的逻辑，如果确实需要在线安装选项你应该手动输入在线镜像源比如 

```bash
https://mirrors.tuna.tsinghua.edu.cn/openeuler/openEuler-22.03-LTS-SP3/OS/x86_64/
```

和

```bash
https://mirrors.tuna.tsinghua.edu.cn/openeuler/openEuler-22.03-LTS-SP3/update/x86_64/
```

确定 openeuler 操作系统 `/etc/profile.d/system-info.sh` 脚本存在一处兼容性 bug

```bash
sed -i 's/if \[ "$whoiam" == "root" \]/if \[ "$(whoami)" = "root" \]/g' /etc/profile.d/system-info.sh
```

### 部署 Docker CE

```bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker.service
systemctl enable --now docker.socket
```

### 部署 CRI-Dockerd

```bash
export http_proxy=http://192.168.1.11:7890
export https_proxy=http://192.168.1.11:7890
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.11/cri-dockerd-0.3.11.amd64.tgz
tar xzvf cri-dockerd-0.3.11.amd64.tgz
wget -O /etc/systemd/system/cri-docker.service https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget -O /etc/systemd/system/cri-docker.socket https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
install -o root -g root -m 0755 cri-dockerd/cri-dockerd /usr/local/bin/cri-dockerd
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl enable --now cri-docker.service
systemctl enable --now cri-docker.socket
```
### 安装操作系统管理工具 Cockpit 软件包组

```bash
dnf install cockpit cockpit-ws cockpit-storaged cockpit-system
systemctl enable --now cockpit.socket

vim /etc/cockpit/disallowed-users
# root

dnf install gettext nodejs make
git clone https://github.com/chabad360/cockpit-docker.git
cd cockpit-docker
make && make install
systemctl restart cockpit.socket
```

Cockpit 管理工具疑难解问题

1. auditd 需要重启 `chmod 755 /var/log/audit/` 后 `service auditd restart`

### 设置 SELinux 状态为 Permissive

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 清空 IPTables 并禁用防火墙

```bash
iptables -F && iptables-save
systemctl stop firewalld.service && systemctl disable firewalld.service
```

### 启用 Kubernetes 软件仓库 并安装 Kubelet、Kubeadm 和 Kubectl

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/Kubernetes.repo
[kubernetes]
name=Kubernetes
# baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.29/rpm/
enabled=1
gpgcheck=1
# gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet.service
```

### 设置 Docker Cgroup Driver 及镜像加速器（推荐）

```bash
vim /etc/docker/daemon.json
{
   "exec-opts": [
        "native.cgroupdriver=systemd"
  ],
   "registry-mirrors": [
        "https://docker.mirrors.sjtug.sjtu.edu.cn/"
  ]
}    

vim /etc/systemd/system/multi-user.target.wants/cri-docker.service
ExecStart=/usr/local/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9 --container-runtime-endpoint fd://

systemctl daemon-reload
systemctl restart docker.service
systemctl restart docker.socket
systemctl restart cri-docker.service
systemctl restart cri-docker.socket
```

这里的镜像加速器配置可选用阿里云私有 Docker 镜像加速器，但阿里云私有 Docker 镜像加速器需要登录阿里云账号

```bash
docker login
```

### 设置 Docker 镜像服务器访问代理（可选但不推荐，不利于交付）

```bash
vim /etc/docker/daemon.json
{
   "exec-opts": [
        "native.cgroupdriver=systemd"
  ]
}

vim /etc/systemd/system/multi-user.target.wants/docker.service
[Service]
Environment="HTTP_PROXY=http://192.168.1.11:7890"
Environment="HTTPS_PROXY=http://192.168.1.11:7890"
Environment="NO_PROXY=10.96.0.0/16,10.244.0.0/16,127.0.0.1,192.168.1.0/24,localhost,master-1,node-1"


vim /etc/systemd/system/multi-user.target.wants/cri-docker.service
ExecStart=/usr/local/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9 --container-runtime-endpoint fd://
[Service]
Environment="HTTP_PROXY=http://192.168.1.11:7890"
Environment="HTTPS_PROXY=http://192.168.1.11:7890"
Environment="NO_PROXY=10.96.0.0/16,10.244.0.0/16,127.0.0.1,192.168.1.0/24,localhost,master-1,node-1"


systemctl daemon-reload
systemctl restart docker.service
systemctl restart docker.socket
systemctl restart cri-docker.service
systemctl restart cri-docker.socket
```
### 对于 Contarinerd 用户（可选）

Step 1: Installing containerd

```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
wget -O /etc/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

Step 2: Installing runc

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```

Step 3: Installing CNI plugins

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.1.tgz
```

Step 4: Generate containerd configuration file

```bash
mkdir -p /etc/containerd/
containerd config default > /etc/containerd/config.toml
vim /etc/containerd/config.toml
   [plugins."io.containerd.grpc.v1.cri"]
      sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
               [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                  SystemdCgroup = true
      [plugins."io.containerd.grpc.v1.cri".registry]
         config_path = ""
         [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
            [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
                endpoint = ["https://docker.mirrors.sjtug.sjtu.edu.cn"]
```

Step 5：Enable the Systemd daemon for containerd

```bash
systemctl daemon-reload
systemctl enable --now containerd
```

### 调整内核参数

```bash
cat>> /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
modprobe br_netfilter
sysctl -p /etc/sysctl.d/99-kubernetes.conf
```

### 开启内核 IPVS 以彻底弃用 IPTables

```bash
dnf install -y ipvsadm
mkdir -p /etc/sysconfig/modules/
cat>> /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in \${ipvs_modules}; do
/sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
if [ $? -eq 0 ]; then
/sbin/modprobe \${kernel_module}
fi
done
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules
lsmod | grep ip_vs
```
### 内核配置持久化

```bash
vim /etc/environment
modprobe br_netfilter
sysctl -p /etc/sysctl.d/99-kubernetes.conf
bash /etc/sysconfig/modules/ipvs.modules
```

### 日志持久化

```bash
mkdir /var/log/journal
mkdir /etc/systemd/journald.conf.d
cat >> /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
Storage=persistent
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
SystemMaxUse=10G
SystemMaxFileSize=200M
MaxRetentionSec=6week
ForwardToSyslog=no
EOF
systemctl restart systemd-journald
```

### 设置 Kubelet Cgroup Driver

```bash
vim /etc/systemd/system/multi-user.target.wants/kubelet.service
[Service]
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"

systemctl daemon-reload
systemctl restart kubelet.service
```

### 配置域名解析

```bash
vim /etc/hosts
192.168.1.16	master-1
192.168.1.22	node-1
```

### 配置时间同步

```bash
chronyc -a makestep && date
```

## 创建 Kubernetes 集群

### 初始化集群控制节点

```bash
unset http_proxy
unset https_proxy

kubeadm init --image-repository=registry.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.1.16 --control-plane-endpoint=192.168.1.16 --service-cidr=10.96.0.0/16 --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```

Your Kubernetes control-plane has initialized successfully!

注意:  使用单一大二层网络的云计算用户应特别注意 `--apiserver-advertise-address` 和 `--control-plane-endpoint` 应填写虚拟机的实际 IP 不应填写浮动路由提供的浮动 IP，具体原因见 [OpenStack Networking Option 1: Provider networks](https://docs.openstack.org/neutron/2023.2/admin/config-routed-networks.html#config-routed-provider-networks)，`--control-plane-endpoint` 可填写 `/etc/hosts` 绑定的控制节点或高可用状态的静态域名

使用多个小型二层网络云计算用户，因为虚拟机之间内网存在隔离，`--apiserver-advertise-address` 填写虚拟机的实际 IP，`--control-plane-endpoint` 填写浮动路由提供的浮动 IP 会出现集群伪 Ready 状态 （计算节点无法配置CNI网桥），所以如果非要在内网隔离环境下设置 Kubernetes 集群可使用 OpenVPN 隧道实现节点间的通信，相对应的集群间各个节点的接口应设置为 OpenVPN 隧道通信接口，这种架构同样适用于跨云计算服务提供商用户，使得 Kubernetes 集群的部署更加灵活但也使得网络模型更加复杂

`--cri-socket` 参数值对应容器运行时类型 Docker 用户应使用 `unix:///var/run/cri-dockerd.sock` containerd 用户应使用 `unix:///var/run/containerd/containerd.sock`

To start using your cluster, you need to run the following as a regular user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

关于在 Windows 上控制虚拟机或远程云服务器主机上的集群最方便的步骤

1. 下载 Windows 版 kubectl 可执行程序丢进 C:\Windows
2. 将复制出的 /etc/kubernetes/admin.conf 复制到用户目录 .kube 文件夹内改名为 config

其他节点的加入按照初始化集群控制节点后打印出的信息执行 kubeadm join，注意追加 --cri-socket 参数

### 控制平面节点隔离（Kubernetes AllinOne 污染节点/容忍污染）

默认情况下，出于安全原因，Kubernetes 集群不会在控制平面节点上调度 Pod。 如果你希望能够在单机 Kubernetes 集群等控制平面节点上调度 Pod，应该运行：

```bash
kubectl taint nodes master-1 node-role.kubernetes.io/control-plane-
```

相对应的污染节点应该运行：

```bash
kubectl taint nodes master node-role.kubernetes.io/control-plane=:NoSchedule
```

查看节点污染状态

```bash
kubectl describe nodes master |grep Taints
```

### 安装网络插件 Flannel（可选）

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

注意在创建资源前应保证 kube-flannel.yml 内 net-conf.json 字段的 IPv4 地址配置应和 --pod-network-cidr 所指定的地址一致

### 安装网络插件 Project Calico（可选）

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

vim custom-resources.yaml
cidr: 10.244.0.0/16

kubectl apply -f tigera-operator.yaml
kubectl apply -f custom-resources.yaml
```

注意在创建资源前应保证 custom-resources.yaml 内 IPv4 地址配置应和 --pod-network-cidr 所指定的地址一致

### 验证集群部署状态

```bash
kubectl get pods -A && kubectl get nodes
```

### 配置 crictl

```bash
crictl config runtime-endpoint /run/containerd/containerd.sock
```

### IPv4/IPv6 双协议栈接入的 Kubernetes 集群

特性状态： Kubernetes v1.23 [stable]
IPv4/IPv6 双协议栈网络能够将 IPv4 和 IPv6 地址分配给 Pod 和 Service。

从 1.21 版本开始，Kubernetes 集群默认启用 IPv4/IPv6 双协议栈网络， 以支持同时分配 IPv4 和 IPv6 地址。

支持的功能
Kubernetes 集群的 IPv4/IPv6 双协议栈可提供下面的功能：

 - 双协议栈 Pod 网络（每个 Pod 分配一个 IPv4 和 IPv6 地址）
 - IPv4 和 IPv6 启用的 Service
 - Pod 的集群外出口通过 IPv4 和 IPv6 路由

```bash
kubeadm init --image-repository=registry.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.1.16 --control-plane-endpoint=master-1 --service-cidr=10.96.0.0/16,2001:db8:42:1::/112 --pod-network-cidr=10.244.0.0/16,2001:db8:42:0::/56 --cri-socket=unix:///var/run/containerd/containerd.sock
```

标志 `--apiserver-advertise-address` 不支持双协议栈

#### Project Calico 的 IPv4/IPv6 双协议栈接入支持

编辑 custom-resources.yaml

```yaml
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
    - blockSize: 122
      cidr: 2001:db8:42:0::/56
      encapsulation: None
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

启用配置

```bash
kubectl apply -f tigera-operator.yaml
kubectl apply -f custom-resources.yaml
```