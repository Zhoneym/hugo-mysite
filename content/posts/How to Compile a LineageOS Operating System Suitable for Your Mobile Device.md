---
title: "如何编译适合自己的移动设备的 LineageOS 操作系统" 
date: 2023-08-13T14:12:15+08:00
tags: ["Android"]
draft: false
---

# 如果当前环境是 Microsoft WSL2

```bash
sudo su root
[sudo] password for ubuntu:
rm -rf /etc/wsl.conf /etc/resolv.conf
cat>> /etc/resolv.conf <<EOF
nameserver 192.168.1.1 
EOF
cat>> /etc/wsl.conf <<EOF
[user]
default = ubuntu
[boot]  
systemd = true
[network]
generateResolvConf = false
EOF
```

# 安装必要工具(Ubuntu 20.04)

```bash
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get install -y bc bison build-essential ccache curl flex \
g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick lib32ncurses5-dev \  
lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5 libncurses5-dev \
libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool \
squashfs-tools xsltproc zip zlib1g-dev cmake clang lldb neofetch python-is-python3 \
libwxgtk3.0-gtk3-dev python3-protobuf brotli
```

# 安装 OpenJDK

```bash
wget https://download.bell-sw.com/java/11.0.20+8/bellsoft-jdk11.0.20+8-linux-amd64-full.deb
sudo dpkg -i bellsoft-jdk11.0.20+8-linux-amd64-full.deb
```

# 安装 repo

```bash 
mkdir -p ~/bin
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > ~/bin/repo
chmod a+x ~/bin/repo
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
```

在 `~/.profile` 文件末尾加入:

```bash
# set PATH so it includes user's private bin if it exists  
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH" 
fi
```

```bash
source ~/.profile
```


# 安装 platform-tools

```bash
curl https://dl.google.com/android/repository/platform-tools-latest-linux.zip
unzip platform-tools-latest-linux.zip -d ~
```

在 `~/.profile` 文件末尾加入:

```bash  
# add Android SDK platform tools to path
if [ -d "$HOME/platform-tools" ] ; then
    PATH="$HOME/platform-tools:$PATH"
fi
```

```bash
source ~/.profile
```
# 启用 ccache

在 `~/.bashrc` 文件末尾加入:

```bash
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
export USE_CCACHE=1  
export CCACHE_EXEC=/usr/bin/ccache
```

```bash
source ~/.bashrc 
```

```bash
ccache -M 80G
```

# 配置 git

```bash
git config --global user.email "Zhoneym@Outlook.com"
git config --global user.name "Zhoneym"
```

# 同步源码镜像

```bash
mkdir -p ~/android/lineage  
cd ~/android/lineage
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/lineageOS/LineageOS/android.git -b lineage-20.0 --git-lfs
```

修改 `.repo/manifests/default.xml` 前24行替换为:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

  <remote  name="lineage"
           fetch="https://mirrors.tuna.tsinghua.edu.cn/git/lineageOS/"
           review="review.lineageos.org" />
           
  <remote  name="private"
           fetch="ssh://git@github.com" />
           
  <remote  name="aosp"
           fetch="https://mirrors.tuna.tsinghua.edu.cn/git/AOSP"
           review="android-review.googlesource.com"
           revision="refs/tags/android-13.0.0_r61" />
           
  <default revision="refs/heads/lineage-20.0"
           remote="lineage"
           sync-c="true"
           sync-j="4" />
           
  <superproject name="platform/superproject" remote="aosp" revision="android-13.0.0_r61" />
  <contactinfo bugurl="go/repo-bug" />

  <!-- AOSP Projects -->
```

```bash
repo sync
```

# 添加设备树和设备内核代码

修改 `.repo/manifest.xml` 在`<include name="default.xml" />` 下一行添加:

```xml
<remote name="github"
           fetch="https://github.com/" />
           
<project path="device/essential/mata"
         name="LineageOS/android_device_essential_mata"
         remote="github"
         revision="lineage-20" />
         
<project path="kernel/essential/msm8998"
         name="LineageOS/android_kernel_essential_msm8998"
         remote="github"
         revision="lineage-20" />
```

```bash
repo sync
```

# 提取设备 blob(以 Essential PH-1 为例)

```bash
mkdir ~/android/system_dump/
cd ~/android/system_dump/
wget https://laotzu.ftp.acc.umu.se/mirror/lineageos/full/mata/20230807/lineage-20.0-20230807-nightly-mata-signed.zip
unzip lineage-*.zip payload.bin

python ~/android/lineage/lineage/scripts/update-payload-extractor/extract.py payload.bin --output_dir ./

mkdir system/
sudo mount -o ro system.img system/
sudo mount -o ro vendor.img system/vendor/

cd ~/android/lineage/device/essential/mata/
./extract-files.sh ~/android/system_dump/
cd ~/android/lineage/  
sudo umount -R ~/android/system_dump/system/
rm -rf ~/android/system_dump/
```

# 编译(不带签名,以 Essential PH-1 为例)

```bash
source build/envsetup.sh
breakfast mata
croot 
brunch mata
cd $OUT && ls -l
```

