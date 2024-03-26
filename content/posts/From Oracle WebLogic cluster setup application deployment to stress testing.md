---
title: "WebLogic 集群搭建 应用部署再到压力测试" 
date: 2023-12-31T4:18:25+08:00
draft: false
---


# 一. 搭建开发环境

VMWare Workstations 17 有针对  HypervisorType-1 侧通道缓解措施但效率低下。

如果使用 VMware 请启用 HypervisorType-2

```powershell
bcdedit /set hypervisorlaunchtype off
```

# 安装 Hyper-V 虚拟机环境（可选，对于 WSL 2 和 Android 子系统 用户）

## 通过“设置”启用 Hyper-V 角色

右键单击 Windows 按钮 设置 —— 应用 —— 可选功能 ——更多 Windows 功能。

选择相关设置下右侧的“程序和功能”。

选择“打开或关闭 Windows 功能”。

选择“Hyper-V”，然后单击“确定”。

选择“Windows 虚拟化平台”，然后单击“确定”。（针对 WSL 2 和 Android 子系统 用户）

## 或使用 DISM 启用 Hyper-V

```powershell
Dism /Online /Enable-Feature /All /FeatureName:Microsoft-Hyper-V
```

# Linux 虚拟机环境的安装

因为要以文档的形式呈现本操作题目，Ubuntu 或者 Deban 图形化操作界面没什么可说的，所以本次采用 Arch Linux 以控制台输入 bash/zsh 命令的方式去安装一个虚拟机。

## 创建虚拟机

| 硬件配置                                      | 规格                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| Secure boot                                   | 对于实体机如果配置为 true 并在 DVD 安装时应配置为开放模式（custom），对于虚拟机推荐配置为 disabled |
| RAM（单位： MB）                              | 6144                                                         |
| Disk（单位： GB，固态硬盘）                   | 60 SSD                                                       |
| CPU 核心数量，包括逻辑处理器个数（单位： 个） | 4                                                            |
| Hyper-V 虚拟交换机类型（如果是 Hyper-V 用户） | 外部网络，绑定实体机有线网卡 Realtek PCIe GbE Family Controller 独占。实体机通过无线网卡上网。 |
| 外部网关                                      | 192.168.1.1 （中国电信宽带 Mesh 组网），同时接入 IPv6        |
| 声卡                                          | Hyper-V 虚拟声卡                                             |
| 显卡                                          | Hyper-V 虚拟显示适配器                                       |
| 芯片组                                        | 默认                                                         |
| 启动类型                                      | UEFI                                                         |

## DVD 启动光盘镜像文件

Arch Linux 2023.12

https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/latest/archlinux-2023.12.01-x86_64.iso  (清华大学开源软件镜像站)

## 配置 SSH 连接（可选）

开机进入 CLI 安装模式会自动以 root 模式登录，Arch Linux LiveDVD 的 root 密码为随机字符串，配置 SSH 时应配置为：

免密码认证运行root 用户登录

```bash
vim /etc/sshd.config
PermitRootLogin yes
PermitEmptyPasswords yes
:wq
systemctl restart sshd.service
```

Windows Terminal Powershell 通过 SSH 连接虚拟机

右键单击 Windows 按钮 设置 —— 应用 —— 可选功能 ——添加可选功能

OpenSSH 客户端

```powershell
PS C:\Users\Administrator>ssh root@192.168.1.5
yes
```

## 网络连通性测试

```bash
ipaddr
ping -c4 www.163.com
```

## 确定安装源刷新软件仓库密钥

```bash
echo 'Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch' > /etc/pacman.d/mirrorlist
# 清华大学开源软件镜像站
pacman -Sy
pacman -S --noconfirm wget -y
pacman -S --noconfirm archlinux-keyring -y
```

## 增强软件包管理器下载软件包的稳定性

```bash
vim /etc/pacman.conf
XferCommand = /usr/bin/wget --passive-ftp -c -O %o %u
Color
ILoveCandy
CheckSpace
ParallelDownloads = 8
:wq!
```

## 分区规划

创建分区并保持退出

```bash
cfdisk /dev/nvme0n1
```

|    磁盘分区    |  挂载点   |                   用途及大小                   | 大小 |  格式  |
| :------------: | :-------: | :--------------------------------------------: | :--: | :----: |
| /dev/nvme0n1p1 | /mnt/boot | ESP启动分区，bootloader 固件，内核镜像所在目录 |  1G  | FAT32  |
| /dev/nvme0n1p2 |  [SWAP]   |                    交换空间                    | 12G  | [SWAP] |
| /dev/nvme0n1p3 |   /mnt/   |                       根                       | 47G  |  EXT4  |

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p3
mkswap -f /dev/nvme0n1p2
swapon /dev/nvme0n1p2
mkdir -p /mnt/boot/
mount /dev/nvme0n1p1 /mnt/boot/
mount /dev/nvme0n1p3 /mnt/
```

## Pacstrap 阶段，安装基础软件包组，内核（linux-zen）和驱动程序包（linux-firmware）

```bash
pacstrap -i /mnt base base-devel net-tools linux-firmware linux-firmware-qcom linux-firmware-qlogic linux-firmware-whence linux-firmware-nfp linux-firmware-bnx2x linux-firmware-liquidio linux-firmware-marvell linux-firmware-mellanox linux-zen linux-zen-docs linux-zen-headers
genfstab -U -p /mnt > /mnt/etc/fstab
```

## Chroot 阶段，进一步配置

```bash
arch-chroot /mnt/root/ /bin/bash
```

## 配置计算机名称

```bash
echo 'Arch-Linux' > /etc/hostname
```

## 配置 Locales

```bash
config_locale(){
    echo "Please choose your locale time"
    select TIME in `ls /usr/share/zoneinfo`;do
        if [ -d "/usr/share/zoneinfo/$TIME" ];then
            select time in `ls /usr/share/zoneinfo/$TIME`;do
                ln -sf /usr/share/zoneinfo/$TIME/$time /etc/localtime
                break
            done
        else
            ln -sf /usr/share/zoneinfo/$TIME /etc/localtime
            break
        fi
        break
    done
    hwclock --systohc --utc
    echo "Choose your language"
    select LANG in "en_US.UTF-8" "zh_CN.UTF-8";do
        echo "$LANG UTF-8" > /etc/locale.gen
        locale-gen
        echo LANG=$LANG > /etc/locale.conf
        break
    done
}
config_locale
```

Locales 玄学：如果首次启动操作系统 Locales 设置为英语 以后总有相当多的字符无法改成中文，QT 窗口和 Java 窗口上此问题尤为严重

## 配置 bootloader （grub 2，Secure boot = disabled）

```bash
install_grub(){
        pacman -S --noconfirm grub efibootmgr dosfstools -y
        rm -f /sys/firmware/efi/efivars/dump-*
        grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Linux
        grub-mkconfig -o /boot/grub/grub.cfg
}
install_grub
```

## 新建用户

```bash
add_user(){
    echo "The username will be set to arch"
    useradd -m -g wheel arch
    echo "Set the password"
    passwd arch
    echo "Set the root password"
    passwd
    pacman -S --noconfirm sudo
    echo 'root ALL=(ALL:ALL) ALL' > /etc/sudoers
    echo '%sudo ALL=(ALL:ALL) ALL' >> /etc/sudoers
    echo '%wheel ALL=(ALL:ALL) ALL' >> /etc/sudoers
    echo '%wheel ALL=(ALL:ALL) NOPASSWD: ALL' >> /etc/sudoers
    echo '@includedir /etc/sudoers.d' >> /etc/sudoers
}
add_user
```

## 安装 Hyper-V 增强功能和键盘，鼠标，视频驱动

```bash
pacman -S --noconfirm hyperv xf86-video-fbdev xf86-input-vmmouse -y
systemctl enable hv_fcopy_daemon.service
systemctl enable hv_kvp_daemon.service
systemctl enable hv_vss_daemon.service
```

## 安装 VMware 客户机工具和键盘，鼠标，视频驱动（可选，对于 VMware 用户）

```bash
pacman -S --noconfirm open-vm-tools gtkmm3 -y
pacman -S --noconfirm xf86-video-vmware -y
pacman -S --noconfirm xf86-input-vmmouse -y
systemctl enable vmtoolsd.service
systemctl enable vmware-vmblock-fuse.service
```



## 安装网络管理器，输入法，图标包，中文字体，防火墙和反病毒软件和其它系统所必须的软件包

```bash
install_app(){
    pacman -S --noconfirm networkmanager xorg-server xorg-xauth wqy-zenhei wqy-microhei wqy-microhei-lite wqy-bitmapfont ruby lolcat zsh vim nano git openssh noto-fonts-emoji p7zip papirus-icon-theme
    pacman -S --noconfirm ufw unrar cmake go llvm lldb podman cni-plugins cockpit cockpit-podman podman-docker
    systemctl enable ufw.service
    systemctl enable NetworkManager
    systemctl enable sshd
    pacman -S --noconfirm clamav clamtk
    systemctl enable clamav-daemon.service
    systemctl enable clamav-daemon.socket
    systemctl enable clamav-clamonacc.service
    systemctl enable clamav-freshclam.service
    pacman -S --noconfirm ibus ibus-rime rime-luna-pinyin
    pacman -S --noconfirm libreoffice-fresh libreoffice-fresh-zh-cn
}
install_app
```

## 安装 Plasma 桌面（可选，stable branch）

```bash
pacman -S plasma-meta kde-accessibility-meta kde-graphics-meta kde-multimedia-meta kde-network-meta kde-pim-meta kde-sdk-meta kde-system-meta kde-utilities-meta sddm
systemctl enable sddm
```

## 重启虚拟机，清理实体机所保存的 SSH 密钥，操作系统安装结束

```bash
reboot
```

为防止后续操作出现问题，可将虚拟机导出为模板

# 开发环境安装阶段

## 配置 SSH 连接

开机进入虚拟机桌面，但我们通常可以通过SSH完成绝大部分工作（依赖软件包 xorg-server）

```bash
vim /etc/sshd.config
PermitRootLogin yes
PermitEmptyPasswords yes
X11Forwarding yes
:wq
systemctl restart sshd.service
```

Windows Terminal Powershell 通过 SSH 连接虚拟机

```
PS C:\Users\Administrator>ssh arch@192.168.1.5
yes
```

## 安装 git 并通过 Arch Linux 用户社区仓库打包安装 Oracle JDK-11

## 下载 Oracle JDK-11

https://www.oracle.com/java/technologies/downloads/#java11

Linux x64 Compressed Archive

## 打包安装

```bash
[arch@Arch-Linux ~]$ sudo pacman -Syu --noconfirm git
[arch@Arch-Linux ~]$ git clone https://aur.archlinux.org/jre11.git && cd jre11
[arch@Arch-Linux jre11]$ mv ../Downloads/jdk-11.0.21_linux-x64_bin.tar.gz ./jdk-11.0.21_linux-x64_bin.tar.gz
[arch@Arch-Linux jre11]$ makepkg -si
[arch@Arch-Linux jre11]$ cd ..
[arch@Arch-Linux ~]$ git clone https://aur.archlinux.org/jdk11.git && cd jdk11
[arch@Arch-Linux jdk11]$ mv ../jre11/jdk-11.0.21_linux-x64_bin.tar.gz ./jdk-11.0.21_linux-x64_bin.tar.gz
[arch@Arch-Linux jdk11]$ makepkg -si
[arch@Arch-Linux jdk11]$ cd ..
[arch@Arch-Linux ~]$ java --version
java 11.0.21 2023-10-17 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.21+9-LTS-193)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.21+9-LTS-193, mixed mode)
```

## 打包安装 Eclipse-jee

```bash
[arch@Arch-Linux ~]$ git clone https://aur.archlinux.org/eclipse-jee.git
[arch@Arch-Linux ~]$ cd eclipse-jee/
[arch@Arch-Linux eclipse-jee]$ makepkg -si
# 可以注释掉 eclipse-jee 打包时对 openjdk 的强制依赖
```

## 安装 Apache Benchmark 和 Apache Jmeter

```bash
[arch@Arch-Linux ~]$ sudo pacman -Syu --noconfirm apache
[arch@Arch-Linux ~]$ git clone https://aur.archlinux.org/jmeter.git
[arch@Arch-Linux ~]$ cd jmeter
[arch@Arch-Linux jmeter]$ makepkg -si
```

## 打包安装 Microsoft Edge

```bash
[arch@Arch-Linux ~]$ git clone https://aur.archlinux.org/microsoft-edge-stable-bin
[arch@Arch-Linux ~]$ cd microsoft-edge-stable-bin/
[arch@Arch-Linux microsoft-edge-stable-bin]$ makepkg -si
```

## 安装 Mariadb 数据库

```bash
[arch@Arch-Linux ~]$ sudo pacman -Syu --noconfirm mariadb mariadb-libs
# KDE Plasma 桌面强制依赖 Mariadb 只需安装 mariadb-libs，可选 Dbeaver 社区版作为数据库管理工具
[arch@Arch-Linux ~]$ sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
[arch@Arch-Linux ~]$ sudo systemctl enable --now mariadb.service
[arch@Arch-Linux ~]$ sudo mariadb-admin -u root password 'root'
```

## 安装 Weblogic

到社群里问了一下，Arch Linux 用户社区仓库里没有 Weblogic 的 PKGBUILD，需要手动下载

下载 Generic Installer for Oracle WebLogic Server and Oracle Coherence : https://www.oracle.com/middleware/technologies/weblogic-server-downloads.html

```bash
[arch@Arch-Linux ~]$ mv ../Downloads/fmw_14.1.1.0.0_wls_lite_Disk1_1of1.zip ~
[arch@Arch-Linux ~]$ unzip fmw_14.1.1.0.0_wls_lite_Disk1_1of1.zip
Archive:  fmw_14.1.1.0.0_wls_lite_Disk1_1of1.zip
  inflating: fmw_14.1.1.0.0_wls_lite_generic.jar
[arch@Arch-Linux ~]$ java -jar fmw_14.1.1.0.0_wls_lite_generic.jar
# 此处存在多个值得注意的问题：
# 1.Weblogic 并不支持 OpenJDK 发行版 请切换至 Oracle JDK
# 2.Weblogic 安装程序强制依赖 xorg-server，对显示器及显示服务器有严格要求，暂不确定 Wayland 上是否也可以（WSL2 和 Wayland 用户需特别注意）
# 3.当操作系统 Locales 设置为 en_US UTF-8 时操作界面为英语（WSL2 和 Docker/Podman 用户需特别注意 glibc 软件包是否安装）
# 4.Oracle JDK 版本默认推荐 8u191 高版本也可以
```

产品清单目录

/home/arch/oraInventory

Oracle 主目录

/home/arch/Oracle

创建域

/home/arch/Oracle/projects/domains/weblogic

部署完成后通过 Konsole 执行 startWebLogic.sh 脚本，防火墙放行对应端口，通过 Edge 浏览器 访问 http://localhost:7001/console 或 https://localhost:7002/console 登录信息见下表

```bash
[arch@Arch-Linux ~]$ cd Oracle/projects/domains/weblogic
[arch@Arch-Linux weblogic]$ ./startWebLogic.sh
```

启用 cockpit 服务，放行 9090 端口，开启Arch Linux web console 通过 Edge 浏览器 访问 http://localhost:9090

```bash
[arch@Arch-Linux ~]$ sudo systemctl enable --now cockpit.socket
```

## WebLogic 集群的搭建

误区：Server 的概念不同于计算机网络对物理层设备的定义，如果我们假设一个端口只提供一个应用程序服务，那么一个套接字（Socket）就可以被视为一个单独的 服务器，套接字是网络通信的基础，它是在特定的 IP 地址和端口上监听传入请求的一种方式。每个套接字都可以看作是一个独立的服务器，它在特定的端口上提供特定的服务。这种理解方式强调了服务的分布式和并发性。

在启动服务器之前应该在 web 控制台设置对应服务器的登录信息

| 文件夹路径                            | 逻辑名称    | 所属集群 | 监听端口  | 登录信息(账号/密码) |
| :-----------------------------------: | :---------: | :------------: | :---------------: | :---------------: |
| weblogic/servers/admin | admin | 独立 | 127.0.0.1:7001/7002 | admin/Aa426855 |
| weblogic/servers/server-1 | server-1   | cluster-1 | 127.0.0.1:7003/7004 | admin/Aa426855 |
| weblogic/servers/server-2 | server-2   | cluster-1 | 127.0.0.1:7005/7006 | admin/Aa426855 |
| weblogic/servers/server-3 | server-3   | cluster-1 | 127.0.0.1:7007/7008 | admin/Aa426855 |
| weblogic/servers/server-4 | server-4   | cluster-1 | 127.0.0.1:7009/7010 | admin/Aa426855 |
| weblogic/servers/server-5 | server-5   | cluster-1 | 127.0.0.1:7011/7012 | admin/Aa426855 |

通过 Konsole 执行 startManagedWebLogic.sh 脚本启动集群内其他服务器

```bash
[arch@Arch-Linux ~]$ cd Oracle/projects/domains/weblogic
[arch@Arch-Linux weblogic]$ bin/startManagedWebLogic.sh server-1 http://127.0.0.1:7001
[arch@Arch-Linux weblogic]$ bin/startManagedWebLogic.sh server-2 http://127.0.0.1:7001
[arch@Arch-Linux weblogic]$ bin/startManagedWebLogic.sh server-3 http://127.0.0.1:7001
[arch@Arch-Linux weblogic]$ bin/startManagedWebLogic.sh server-4 http://127.0.0.1:7001
[arch@Arch-Linux weblogic]$ bin/startManagedWebLogic.sh server-5 http://127.0.0.1:7001
```



# 二.  开发一个 Java Web 应用并部署到 WebLogic

# Eclipse-jee 镜像源配置

Eclipse-jee 插件商店可配置为清华大学开源软件镜像

https://download.eclipse.org 全部替换为 https://mirrors.tuna.tsinghua.edu.cn/eclipse

# 新建 Dynamic Web Project

Files > New > Project > Dynamic Web Project

下载服务器软件（Targrt Runtime）为 Apache Tomcat v9

Dynamic Web module version 调整为 4.0

设置项目名称 RegistrationSystem

项目描述：写一个简单的 Javaweb 实现手机号和密码注册登录，mariadb创建一个数据库有一张表 用来存手机号和密码

使用MariaDB数据库进行用户注册和登录。以下是源码：

## 数据库表创建 (在MariaDB中执行)

```sql
CREATE DATABASE UserDB;
USE UserDB;
CREATE TABLE Users (
    phone VARCHAR(20) PRIMARY KEY,
    password VARCHAR(100)
);
```

## Java 数据库连接类 (DBConnection.java)

```java
import java.sql.*;
public class DBConnection {
    private static final String DB_URL = "jdbc:mariadb://localhost:3306/UserDB";
    private static final String USER = "root";
    private static final String PASS = "root";

    public static Connection getConnection() {
        Connection conn = null;
        try {
            Class.forName("org.mariadb.jdbc.Driver");
            conn = DriverManager.getConnection(DB_URL, USER, PASS);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return conn;
    }
}
```

## 用户模型 (User.java)

```java
public class User {
    private String phone;
    private String password;
    public String getPhone() {
        return phone;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }
    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

## 用户DAO (UserDAO.java)

```java
import java.sql.*;
public class UserDAO {
    public static int register(User u) {
        int status = 0;
        try {
            Connection conn = DBConnection.getConnection();
            PreparedStatement ps = conn.prepareStatement("INSERT INTO Users VALUES (?,?)");
            ps.setString(1, u.getPhone());
            ps.setString(2, u.getPassword());
            status = ps.executeUpdate();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return status;
    }
    public static boolean login(User u) {
        boolean status = false;
        try {
            Connection conn = DBConnection.getConnection();
            PreparedStatement ps = conn.prepareStatement("SELECT * FROM Users WHERE phone=? AND password=?");
            ps.setString(1, u.getPhone());
            ps.setString(2, u.getPassword());
            ResultSet rs = ps.executeQuery();
            status = rs.next();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return status;
    }
}
```

## Registration Page (register.jsp)

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <title>Register</title>
</head>
<body>
    <form action="RegisterServlet" method="post">
        Phone: <input type="text" name="phone"><br>
        Password: <input type="password" name="password"><br>
        <input type="submit" value="Register">
    </form>
</body>
</html>
```

## Login Page (login.jsp)

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <title>Login</title>
</head>
<body>
    <form action="LoginServlet" method="post">
        Phone: <input type="text" name="phone"><br>
        Password: <input type="password" name="password"><br>
        <input type="submit" value="Login">
    </form>
</body>
</html>
```

## Login Success Page (login_success.jsp)

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <title>Login Success</title>
</head>
<body>
    <h1>Login successful! Welcome back!</h1>
</body>
</html>
```

## Login Failure Page (login_fail.jsp)

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <title>Login Failure</title>
</head>
<body>
    <h1>Login failed! Please check your phone number and password.</h1>
</body>
</html>
```

## Registration Success Page (register_success.jsp)

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <title>Registration Success</title>
</head>
<body>
    <h1>Registration successful! Welcome to join us!</h1>
</body>
</html>
```

## Registration Failure Page (register_fail.jsp)

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <title>Registration Failure</title>
</head>
<body>
    <h1>Registration failed! Please try again later.</h1>
</body>
</html>
```

## 注册 Servlet (RegisterServlet.java)

```java
import java.io.*;
import jakarta.servlet.*;
import jakarta.servlet.http.*;

public class RegisterServlet extends HttpServlet {
    /**
	 * 
	 */
	private static final long serialVersionUID = 1L;

	protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html");
        PrintWriter out = response.getWriter();

        String phone = request.getParameter("phone");
        String password = request.getParameter("password");

        User user = new User();
        user.setPhone(phone);
        user.setPassword(password);

        int status = UserDAO.register(user);
        if (status > 0) {
            RequestDispatcher rd = request.getRequestDispatcher("register_success.jsp");
            rd.forward(request, response);
        } else {
            RequestDispatcher rd = request.getRequestDispatcher("register_fail.jsp");
            rd.forward(request, response);
        }
        out.close();
    }
}
```

## 登录 Servlet (LoginServlet.java)

```java
import java.io.*;
import jakarta.servlet.*;
import jakarta.servlet.http.*;

public class LoginServlet extends HttpServlet {
    /**
	 * 
	 */
	private static final long serialVersionUID = 1L;

	protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html");
        PrintWriter out = response.getWriter();

        String phone = request.getParameter("phone");
        String password = request.getParameter("password");

        User user = new User();
        user.setPhone(phone);
        user.setPassword(password);

        boolean status = UserDAO.login(user);
        if (status) {
            RequestDispatcher rd = request.getRequestDispatcher("login_success.jsp");
            rd.forward(request, response);
        } else {
            RequestDispatcher rd = request.getRequestDispatcher("login_fail.jsp");
            rd.forward(request, response);
        }
        out.close();
    }
}
```

## 项目的配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
         http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <display-name>Registration System</display-name>

    <servlet>
        <servlet-name>LoginServlet</servlet-name>
        <servlet-class>LoginServlet</servlet-class>
    </servlet>

    <servlet-mapping>
        <servlet-name>LoginServlet</servlet-name>
        <url-pattern>/LoginServlet</url-pattern>
    </servlet-mapping>

    <servlet>
        <servlet-name>RegisterServlet</servlet-name>
        <servlet-class>RegisterServlet</servlet-class>
    </servlet>

    <servlet-mapping>
        <servlet-name>RegisterServlet</servlet-name>
        <url-pattern>/RegisterServlet</url-pattern>
    </servlet-mapping>

    <welcome-file-list>
        <welcome-file>login.jsp</welcome-file>
    </welcome-file-list>

</web-app>

```

MariaDB JDBC 驱动：https://mariadb.com/downloads/connectors/

Servlet API：https://maven.java.net/content/repositories/releases/javax/servlet/javax.servlet-api/4.0.0/

# 打包上传至 Weblogic

Files > Export > WAR Files

Weblogic 部署 > 上传

## 兼容性问题

-  JavaWeb 项目应和中间件使用的 jdk 版本一致，否则部署必然失败
- Apache Tomcat 10 中，所有的servlet类都应该使用 jakarta.servlet，而不是 javax.servlet 而 Weblogic 最新版本仍不识别 jakarta.servlet

# 三. 压力测试

# Apache Benchmark

找这个软件包找了半天，查阅Arch Linux 软件包文档才发现是 Apache Httpd 网页服务器内置的压力测试工具

注意：Apache 软件包在不同的操作系统上的软件包名称不同，Rocky Linux 上是 httpd

```bash
[arch@Arch-Linux ~]$ ab -v && ab -h
# 查阅版本及命令文档
[arch@Arch-Linux ~]$ ab -n 1000 -c 900 http://127.0.0.1:7001/RegistrationSystem/login.jsp
[arch@Arch-Linux ~]$ ab -n 1000 -c 900 http://127.0.0.1:7001/RegistrationSystem/login_success.jsp
[arch@Arch-Linux ~]$ ab -n 1000 -c 900 http://127.0.0.1:7001/RegistrationSystem/login_fail.jsp
[arch@Arch-Linux ~]$ ab -n 1000 -c 900 http://127.0.0.1:7001/RegistrationSystem/register.jsp
[arch@Arch-Linux ~]$ ab -n 1000 -c 900 http://127.0.0.1:7001/RegistrationSystem/register_success.jsp
[arch@Arch-Linux ~]$ ab -n 1000 -c 900 http://127.0.0.1:7001/RegistrationSystem/register_fail.jsp
# -n 访问的总次数，-c 访问的并发量
```

# Apache Jmeter

没有什么技术含量，默认英语界面可切换为简体中文

Options > Choose Languages > 简体中文

## 主要元件

- 测试计划：是使用 JMeter 进行测试的起点，它是其它 JMeter测试元件的容器

- 线程组：代表一定数量的用户，它可以用来模拟用户并发发送请求。实际的请求内容在 Sampler 中定义，它被线程组包含。

- 配置元件：维护 Sampler 需要的配置信息，并根据实际的需要修改请求的内容。

- 前置处理器：负责在请求之前工作，常用来修改请求的设置

- 定时器：负责定义请求之间的延迟间隔。

- 取样器：是性能测试中向服务器发送请求，记录响应信息、响应时间的最小单元，可以根据设置的参数向服务器发出不同类型的请求。

- 后置处理器：负责在请求之后工作，常用获取返回的值。

- 断言：用来判断请求响应的结果是否如用户所期望的。

- 监听器：负责收集测试结果，同时确定结果显示的方式。

- 逻辑控制器：可以自定义 JMeter 发送请求的行为逻辑，它与 Sampler 结合使用可以模拟复杂的请求序列。


## 作用域和执行顺序

元件作用域

- 配置元件：影响其作用范围内的所有元件。

- 前置处理器：在其作用范围内的每一个sampler元件之前执行。

- 定时器：在其作用范围内的每一个sampler有效

- 后置处理器：在其作用范围内的每一个sampler元件之后执行。

- 断言：在其作用范围内的对每一个sampler元件执行后的结果进行校验。

- 监听器：在其作用范围内对每一个sampler元件的信息收集并呈现。


元件执行顺序

配置元件 > 前置处理器 > 定时器  >取样器 > 后置处理程序 > 断言 > 监听器

## Jmeter 接口测试流程

测试计划 > 线程组 > HTTP Cookie管理器 > 请求默认值 > Sampler > 断言 > 监听器
