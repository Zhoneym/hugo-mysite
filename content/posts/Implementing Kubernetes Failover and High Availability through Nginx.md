---
title: "通过 Nginx 实现 Kubernetes 故障转移和高可用性" 
date: 2024-02-28T04:18:25+08:00
tags: ["Kubernetes"]
categories: ["Kubernetes"]
draft: false
---

本文修订日期 ：2024 年 3 月 20 日  当前版本 ：1.29.3

## 通过 Nginx 实现 Kubernetes 故障转移和高可用性

### 网络接口规划

| 主机名        | 网卡                | IPv4地址        | 网关        |
| ------------- | ------------------- | --------------- | ----------- |
| master-1      | ens160              | 192.168.1.25/24 | 192.168.1.1 |
| master-2      | ens160              | 192.168.1.26/24 | 192.168.1.1 |
| master-3      | ens160              | 192.168.1.27/24 | 192.168.1.1 |
| node-1        | ens160              | 192.168.1.28/24 | 192.168.1.1 |
| node-2        | ens160              | 192.168.1.29/24 | 192.168.1.1 |
| node-3        | ens160              | 192.168.1.30/24 | 192.168.1.1 |
| backend (VIP) | ens160 (KeepaLived) | 192.168.1.31/24 | 192.168.1.1 |

### 在预先规划的管理节点上安装 Nginx 和 Keeplived

```bash
dnf install -y nginx keepalived nginx-all-modules
```

### 配置 Nginx stream 用来实现四层协议的转发、代理和负载均衡

```bash
vim /etc/nginx/nginx.conf
stream {
    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log  /var/log/nginx/control-plane.log  main;
    upstream backend {
        server 192.168.1.25:6443 weight=5 max_fails=4 fail_timeout=20s;  # master-1
        server 192.168.1.26:6443 weight=5 max_fails=4 fail_timeout=20s;  # master-2
        server 192.168.1.27:6443 weight=5 max_fails=4 fail_timeout=20s;  # master-3
    }
    server {
        listen 16443;  # control-plane
        proxy_pass backend;
    }
}
```

`log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';`: 定义了一个名为 `main` 的日志格式。其中 `$remote_addr` 是客户端的地址， `$upstream_addr` 是上游服务器的地址， `$time_local` 是本地时间， `$status` 是请求状态，`$upstream_bytes_sent`是发送给上游服务器的字节数

`access_log  /var/log/nginx/control-plane.log  main;` : 指定了访问日志的路径为 `/var/log/nginx/control-plane.log` ，并使用了前面定义的 `main` 日志格式

`upstream backend {...}`: 定义了一个名为 `backend` 的上游服务器组，包含两个服务器，每个服务器权重是5，最大失败次数为4，失败超时时间20秒

`server {...}`: 定义了一个服务器，监听端口`16443`，并将流量代理到 `backend` 这个上游服务器组

### Keeplived 带 Nginx 活性监视的故障转移设置

#### 编写一个 Nginx 服务器活性检测脚本

```bash
vim /etc/nginxkeepalived.sh
#!/bin/bash
# 1. Check if Nginx is alive
counter=$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|$$" )
if [ $counter -eq 0 ]; then
    # 2. If not alive, try to start Nginx
    service nginx start
    # 3. Wait for 2 seconds and then check the Nginx status again
    sleep 2
    counter=$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|$$" )
    # 4. Check again, if Nginx is still not alive, stop Keepalived to let the address drift
    if [ $counter -eq 0 ]; then
        service keepalived stop
    fi
fi
chmod a+x /etc/nginxkeepalived.sh
```

`service keepalived stop` 和 `systemctl stop keepalived` 都是用来停止 keepalived 服务的命令，但它们在处理方式上有所不同

`service` 命令是传统的 `sysvinit` 系统和服务管理器命令，它通过运行 `/etc/init.d/` 目录下的脚本来启动、停止或重启服务

`systemctl` 命令是 `systemd` 系统和服务管理器命令，它通过复制 `/usr/lib/systemd/system/` 目录下的服务单元文件到 `/etc/systemd/system/`来管理服务

`systemctl stop keepalived.service` 可能无法完全停止在某些情况下使用 `keepalived` 的所有进程。

这是因为 `systemd` 服务单元文件中设置默认 `KillMode=process` 这意味着当停止上 `keepalived` 的时候只停掉主进程，而主进程产生的子进程是不会被停掉。可以尝试修改 `keepalived.service` 文件，将 KillMode 设置 control-group 然后使用 `systemctl daemon-reload` 命令来重读配置。

#### Keeplived 带 Nginx 活性监视的故障转移设置（主节点）

```bash
vim /etc/keepalived/keepalived.conf
global_defs {
   router_id master-1
}
vrrp_script nginx {
    script "/etc/nginxkeepalived.sh"
}
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.31/24
    }
    track_script {
        nginx
    }
}
```

#### 从节点 Keeplived 配置

`state`: Keeplived 角色 从节点可设置为 BACKUP

`interface`: 修改为实际网卡名

`virtual_router_id`: VRRP 路由 ID实例，每个实例是唯一的 

`priority`: 优先级，从节点实例的数值应低于主节点 

`advert_int`: 指定VRRP 心跳包通告间隔时间，默认1秒

#### Keepalived 负载均衡转移设置（可选追加配置）

```bash
vim /etc/keepalived/keepalived.conf
virtual_server 192.168.1.31 16443 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    protocol TCP

    real_server 192.168.1.25 6443 {
        TCP_CHECK {
                connect_timeout 10
        }
    }
    real_server 192.168.1.26 6443 {
        TCP_CHECK {
                connect_timeout 10
        }
    }
    real_server 192.168.1.27 6443 {
        TCP_CHECK {
                connect_timeout 10
        }
    }
}
```

#### 启用高可用性

```bash
systemctl daemon-reload
systemctl enable keepalived.service
systemctl enable --now nginx.service
```

## Kubeadm 初始化集群设置

### 检查域名解析

```bash
vim /etc/hosts
192.168.1.25    master-1
192.168.1.26    master-2
192.168.1.27    master-3
192.168.1.28    node-1
192.168.1.29    node-2
192.168.1.30    node-3
192.168.1.31    backend
```

### 初始化 master-1 控制平面

```bash
unset http_proxy
unset https_proxy

kubeadm init --image-repository=registry.aliyuncs.com/google_containers \
--control-plane-endpoint=192.168.1.31 \
--service-cidr=10.96.0.0/16 --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```

`--cri-socket` 参数值对应容器运行时类型 Docker 用户应使用 `unix:///var/run/cri-dockerd.sock` Containerd 用户应使用 `unix:///var/run/containerd/containerd.sock`

`--control-plane-endpoint` 应配置为 keepaLived 生成的 Backend VIP

### 扩容其他节点 master-2 和 master-3 到控制平面

同步控制平面证书到 master-2 和 master-3

创建证书存放目录：

```bash
mkdir -p /etc/kubernetes/pki/etcd

scp /etc/kubernetes/pki/ca.crt master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/ca.key master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.key master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.pub master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.crt master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.key master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.crt master-2:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/pki/etcd/ca.key master-2:/etc/kubernetes/pki/etcd/

scp /etc/kubernetes/pki/ca.crt master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/ca.key master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.key master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.pub master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.crt master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.key master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.crt master-3:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/pki/etcd/ca.key master-3:/etc/kubernetes/pki/etcd/
```

在 master-1 查看节点加入命令加入其他节点，对于 master-2 和 master-3 应追加 `--control-plane` 参数

```bash
kubeadm token create --print-join-command
```

### ETCD 高可用

所有控制平面节点配置 etcd 参数文件

```bash
vim /etc/kubernetes/manifests/etcd.yaml
```

`--initial-cluster` 参数配置加入所有控制平面 

```bash
- --initial-cluster=master-1=https://192.168.1.25:2380,master-2=https://192.168.1.26:2380,master-3=https://192.168.1.27:2380
```

```bash
cat /etc/kubernetes/manifests/etcd.yaml | grep initial-cluster
```

所有节点重启 kubelet 服务应用 ETCD 高可用配置

```bash
systemctl restart kubelet.service
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

## 对于 Kubekey 工具搭建 Kubernetes 集群 ETCD 报错问题的解决方案

KubeSphere 的文档并未提及的一处过时配置，修改部署配置文件里的 etcd type 字段为 kubeadm 可以解决这个问题（值为 kubekey 的设置已经失效）
