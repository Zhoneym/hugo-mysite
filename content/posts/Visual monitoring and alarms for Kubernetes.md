---
title: "Kubernetes 的可视化监控和告警" 
date: 2024-03-20T04:17:25+08:00
tags: ["Kubernetes"]
categories: ["Kubernetes"]
toc:
  depth_from: 1
  depth_to: 4
  ordered: false
draft: false
---

本文修订日期 ：2024 年 3 月 20 日  当前版本 ：1.29.3

# 安装 Helm

Helm 是 Kubernetes 的开源 Charts 管理器，它提供了提供、共享和使用为 Kubernetes 构建的软件的能力。Charts 是预先配置的 Kubernetes 资源文件。使用 Helm 可以：

 - 查找并使用打包为 Helm Charts 的流行软件在 Kubernetes 中运行
 - 以 Helm Charts 的形式共享自定义的 Kubernetes 服务应用程序
 - 创建 Kubernetes 应用程序的可复制构建
 - 智能管理 Kubernetes 清单文件
 - 管理 Helm Charts 包的发布

```bash
wget https://get.helm.sh/helm-v3.14.3-linux-amd64.tar.gz
tar -zxvf helm-v3.14.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

## 部署 Kubernetes Dashboard

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

IPv4 单栈 Kubernetes 集群 Dashboard 部署报错 kubernetes-dashboard-kong Pod CrashLoopBackOff，这个报错看 Pod 日志发现是 IPv6 协议栈没有开启引起的 Nginx 报错

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/charts/kubernetes-dashboard/values.yaml -O dashboard-value.yaml
```

编辑 dashboard-value.yaml 手工配置 KONG_ADMIN_LISTEN, KONG_PROXY_LISTEN 和 KONG_STATUS_LISTEN

```yaml
# Required Kong sub-chart with DBless configuration to act as a gateway
# for our all containers.
kong:
  enabled: true
  env:
    dns_order: LAST,A,SRV,CNAME
    ADMIN_LISTEN: 0.0.0.0:8444 http2 ssl
    PROXY_LISTEN: 0.0.0.0:8443 http2 ssl
    STATUS_LISTEN: 0.0.0.0:8100
  ingressController:
    enabled: false
  dblessConfig:
    configMap: kong-dbless-config
  proxy:
    type: ClusterIP
    http:
      enabled: false
```

部署 Chart

```bash
helm upgrade --install -f dashboard-value.yaml kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

编写清单文件创建一个管理员用户 kubernetes-admin.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
metadata:
  name: admin
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin"   
type: kubernetes.io/service-account-token
```

应用更改

```bash
kubectl apply -f kubernetes-admin.yaml
```

获取 Dashboard Token

```bash
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
```

## 部署 Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install grafana grafana/grafana --create-namespace --namespace monitoring
```

1. Get your 'admin' user password by running:

```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.monitoring.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:

```bash
kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}"
kubectl --namespace monitoring port-forward grafana-5c46f8dcb5-tq28r 3000
```

## 部署 Prometheus Community

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
mkdir -p /prometheus/prometheus-alertmanager-pv/
mkdir -p /prometheus/prometheus-server-pv/
chmod 777 /prometheus/
chmod 777 /prometheus/prometheus-alertmanager-pv/
chmod 777 /prometheus/prometheus-server-pv/
helm upgrade --install prometheus prometheus-community/prometheus --namespace monitoring
```

创建 Prometheus 所需的 Persistent Volumes 编写清单文件 `prometheus-alertmanager-pv.yaml`

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  namespace: monitoring
  name: prometheus-alertmanager-pv
  labels:
    type: local
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/prometheus/prometheus-alertmanager-pv"
```

编写清单文件 `prometheus-server-pv.yaml`

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  namespace: monitoring
  name: prometheus-server-pv
  labels:
    type: local
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/prometheus/prometheus-server-pv"
```

应用更改

```bash
kubectl apply -f prometheus-alertmanager-pv.yaml
kubectl apply -f prometheus-server-pv.yaml
```

Get the Prometheus server URL by running these commands in the same shell:
```bash
kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}"
kubectl --namespace monitoring port-forward prometheus-server-54886dcc45-pkm5p 9090
```

The Prometheus alertmanager can be accessed via port 9093 on the following DNS name from within your cluster:
prometheus-alertmanager.monitoring.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:

```bash
kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}"
kubectl --namespace monitoring port-forward prometheus-alertmanager-0 9093
```

登录 Grafana 仪表板 http://localhost:3000/ 登录名 admin

http://localhost:3000/profile 设置语言为中文（简体）

http://localhost:3000/connections/datasources/new 添加 Prometheus 数据源 http://prometheus-server.monitoring.svc

http://localhost:3000/dashboard/import 导入仪表板 https://grafana.com/grafana/dashboards/
