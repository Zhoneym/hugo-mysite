---
title: "OpenStack 2024.2 安装指南（基于 Debian Bookworm + Kolla Ansible）" 
date: 2025-04-24T04:18:25+08:00
tags: ["OpenStack"]
categories: ["OpenStack"]
toc:
  depth_from: 1
  depth_to: 4
  ordered: false
draft: false
---

本文修订日期 ：2025 年 4 月 24 日  当前版本 ：2024.2


# OpenStack 2024.2 安装指南（基于 Debian Bookworm + Kolla Ansible）

本指南详细解释了部署 OpenStack 2024.2 所涉及的每一步命令，帮助您理解其作用及背后的逻辑。适用于希望使用 [Kolla Ansible](https://docs.openstack.org/kolla-ansible/latest/) 自动部署的用户。

---

## 环境准备

### 替换默认 APT 源

```bash
rm -rf /etc/apt/sources.list
```
删除原有 APT 源配置。

```bash
cat << EOF | tee /etc/apt/sources.list.d/bookworm.sources
Types: deb
URIs: http://mirrors.ustc.edu.cn/debian
Suites: bookworm bookworm-updates bookworm-backports
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: http://mirrors.ustc.edu.cn/debian-security
Suites: bookworm-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
EOF
```
写入 USTC 镜像源，提高更新速度。

```bash
apt-get update -y && apt-get upgrade -y
```
更新系统软件包。

### 安装基础依赖

```bash
apt-get install vim nano network-manager gnupg apt-transport-https wget -y
```
安装编辑器、网络管理、GPG、HTTPS 支持及下载工具。

### 添加 Docker 源并安装 Docker

```bash
wget -qO- http://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-ce.gpg
```
导入 Docker 的 GPG 密钥。

```bash
cat << EOF | tee /etc/apt/sources.list.d/docker-ce.sources
Types: deb
URIs: http://mirrors.aliyun.com/docker-ce/linux/debian
Suites: bookworm
Components: stable
Signed-By: /usr/share/keyrings/docker-ce.gpg
Architectures: amd64
EOF
```
配置 Docker 镜像源。

```bash
apt-get update -y
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```
安装 Docker 相关组件。

### 安装 Miniconda 并配置环境

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod a+x /root/Miniconda3-latest-Linux-x86_64.sh
bash /root/Miniconda3-latest-Linux-x86_64.sh
source /root/.bashrc
```
安装 Miniconda 并加载环境变量。

### 安装构建依赖

```bash
apt-get install git python3-dev libffi-dev gcc libssl-dev build-essential -y
```
安装构建 Kolla 所需的依赖。

### 创建 Kolla 虚拟环境

```bash
conda create -n kolla python=3.12
conda activate kolla
```
创建 Python 3.12 的独立环境用于 Kolla。

## 配置存储卷

```bash
lsblk
apt-get install -y lvm2
pvcreate /dev/sda3
vgcreate cinder-volumes /dev/sda3
lsblk
blkid
vgdisplay
```
使用 `lvm2` 创建 Cinder 使用的卷组。

## 安装 Cockpit

```bash
apt-get install cockpit cockpit-ws cockpit-storaged cockpit-system -y
```
Cockpit 是用于 Web 管理服务器的工具。

## 安装 Python 包和 Kolla Ansible

```bash
python -m pip install -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple --upgrade pip
pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
```
配置使用清华镜像源加速 pip。

```bash
apt install -y pkg-config libdbus-1-dev libglib2.0-dev
pip install selinux python-docker ansible==10.7 python-openstackclient dbus-python
```
安装 Kolla 依赖包。

```bash
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.2
pip install bcrypt==4.0.1
```
安装 Kolla Ansible 和加密库。

## 配置 Kolla

```bash
mkdir -p /etc/kolla
cp miniconda3/envs/kolla/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
```
创建配置目录并复制示例配置。

### 编辑 globals.yml

```bash
cat <<EOF | tee /etc/kolla/globals.yml
---
workaround_ansible_issue_8743: "yes"
ansible_python_interpreter: "/root/miniconda3/envs/kolla/bin/python"
enable_docker_repo: "false"
kolla_base_distro: "debian"
openstack_release: "2024.2"
kolla_container_engine: "docker"
docker_registry: "quay.io"
docker_namespace: "openstack.kolla"
network_interface: "ens32"
kolla_internal_vip_address: "192.168.1.32"
neutron_external_interface: "ens33"
neutron_plugin_agent: "ovn"
enable_openstack_core: "yes"
enable_haproxy: "no"
enable_ceilometer: "yes"
enable_ceilometer_ipmi: "yes"
enable_gnocchi: "yes"
enable_gnocchi_statsd: "yes"
enable_cloudkitty: "yes"
enable_mariadb: "yes"
enable_influxdb: "yes"
enable_redis: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_heat: "yes"
enable_horizon: "yes"
enable_horizon_ironic: "no"
enable_grafana: "yes"
enable_grafana_external: "yes"
enable_skyline: "yes"
enable_proxysql: "no"
enable_ironic: "yes"
enable_ironic_neutron_agent: "yes"
enable_ironic_prometheus_exporter: "yes"
enable_prometheus: "yes"
enable_prometheus_server: "yes"
enable_prometheus_mysqld_exporter: "yes"
enable_prometheus_node_exporter: "yes"
enable_kuryr: "yes"
enable_zun: "yes"
enable_magnum: "yes"
enable_placement: "yes"
enable_trove: "yes"
enable_trove_singletenant: "yes"
cinder_volume_group: "cinder-volumes"
nova_compute_virt_type: "qemu"
ironic_dnsmasq_interface: "ens32"
ironic_cleaning_network: "public"
ironic_dnsmasq_boot_file: "undionly.kpxe"
ironic_dnsmasq_uefi_ipxe_boot_file: "snponly.efi"
docker_configure_for_zun: "yes"
containerd_configure_for_zun: "yes"
gnocchi_incoming_storage: "redis"
cloudkitty_collector_backend: "gnocchi"
cloudkitty_storage_backend: "influxdb"
ironic_dnsmasq_dhcp_ranges:
  - range: "192.168.1.2,192.168.1.254,255.255.255.0"
EOF
```

注意: ens33 无 IP

## 初始化 Kolla 密码与依赖

```bash
kolla-genpwd
kolla-ansible install-deps
cp miniconda3/envs/kolla/share/kolla-ansible/ansible/inventory/all-in-one ~
```

## 配置 Ironic 镜像

```bash
wget https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/ipa-centos9-stable-2024.2.initramfs
wget https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/ipa-centos9-stable-2024.2.kernel
mkdir -p /etc/kolla/config/ironic
mv ipa-centos9-stable-2024.2.initramfs /etc/kolla/config/ironic/ironic-agent.initramfs
mv ipa-centos9-stable-2024.2.kernel /etc/kolla/config/ironic/ironic-agent.kernel
```

## 开始部署

```bash
kolla-ansible bootstrap-servers -i all-in-one
kolla-ansible prechecks -i all-in-one
kolla-ansible pull -i all-in-one
kolla-ansible bootstrap-servers -i all-in-one
kolla-ansible prechecks -i all-in-one
kolla-ansible deploy -i all-in-one
```

```bash
mkdir -p /etc/kolla/ansible/inventory/
cp miniconda3/envs/kolla/share/kolla-ansible/ansible/inventory/all-in-one /etc/kolla/ansible/inventory/
kolla-ansible post-deploy
```

## 初始化 OpenStack 网络和资源

```bash
apt-get install curl -y
vim miniconda3/envs/kolla/share/kolla-ansible/init-runonce
/root/miniconda3/envs/kolla/share/kolla-ansible/init-runonce
```

## 可选命令

```bash
# kolla-ansible deploy -i all-in-one -t neutron 
# kolla-ansible destroy --yes-i-really-really-mean-it -i all-in-one
```

### 安装 Magnum/Ironic 客户端

```bash
pip install python-magnumclient python-ironicclient
```

---

如需进一步配置 Dashboard、API、镜像、外部网络等，请参考 [官方文档](https://docs.openstack.org/)。

