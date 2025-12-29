
问题： windows11通过VirtualBox安装Linux

下面是一套 **Windows 11 + VirtualBox → 安装 Linux（以 Ubuntu Server 22.04 LTS 为例）→ 安装 Docker → 通过 MobaXterm（你提到的“xmobal”）SSH 登录并在 Docker 跑 Nginx，最终从 Windows 访问 Nginx 首页** 的完整、可执行步骤。每一步都给出关键命令与可验证的官方文档来源。

***

## 一、准备工作（Windows 11 主机）

1.  **启用硬件虚拟化**  
    进入 BIOS/UEFI 开启 Intel VT‑x 或 AMD‑V；保存并重启。

> VirtualBox 要求宿主机支持硬件虚拟化（官方《Installation Details》） [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/7.1/user/installation.html)

2.  **安装 VirtualBox（含 Extension Pack）**

*   到 Oracle 官方下载并安装 VirtualBox（管理员运行安装程序）。安装向导会同时安装网络组件（如 Bridged Networking）。 [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/7.1/user/installation.html)
*   如需 USB2/3、NVMe 等增强功能，安装 **Extension Pack**（VirtualBox 主界面 → *File → Tools → Extension Pack Manager*）。 [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/7.1/user/installation.html)

> 若你更习惯图形化“手把手”的安装指南，也可参考这类分步文章（以 Win11 为例）。 [\[msftwebcast.com\]](https://msftwebcast.com/2025/04/how-to-install-oracle-virtualbox-on-windows-11.html), [\[amkstation.com\]](https://amkstation.com/step-by-step-guide-to-installing-virtualbox-on-windows-11-and-10-2025/)

3.  **下载 Ubuntu Server 22.04 LTS ISO**

*   官方下载页（`ubuntu-22.04.5-live-server-amd64.iso`）或 Desktop 版都可，用 Server 版更轻量。 [\[ubuntu.com\]](https://ubuntu.com/download/server), [\[releases.ubuntu.com\]](https://releases.ubuntu.com/jammy/)

***

## 二、在 VirtualBox 创建并安装 Linux 虚拟机（Ubuntu 22.04 Server）

> 下述参数是“稳定起步”配置，你可按主机性能微调。

1.  **创建虚拟机（VM）**

*   VirtualBox → **New**：Name 例如 `Ubuntu-22.04-Server`，Type=Linux，Version=`Ubuntu (64-bit)`。
*   **Base Memory**：4096 MB（或更高）。**Processors**：2–4 vCPU。
*   **Hard Disk**：创建 `VDI`，动态扩展，**40–60 GB**。
*   **ISO 挂载**：在 *Storage* 把光驱指向你下载的 Ubuntu ISO。

2.  **网络模式选择（影响你在 Windows 上怎么连入 VM）**

*   方案 A（**NAT + 端口转发**，主机独立性强、默认安全）：
    *   *Settings → Network → Adapter 1 → Attached to: NAT → Advanced → Port Forwarding*。
    *   添加规则：**TCP**，Host Port=`2222` → Guest Port=`22`（SSH）；Host Port=`8080` → Guest Port=`80`（Nginx）。
    *   NAT 下，来宾不可被外网直接访问；通过端口映射即可从主机访问来宾服务（VirtualBox 官方“**Configuring Port Forwarding with NAT**”）。 [\[virtualbox.org\]](https://www.virtualbox.org/manual/ch06.html), [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/network_nat.html)
*   方案 B（**Bridged**，让 VM 获得局域网 IP）：
    *   *Settings → Network → Adapter 1 → Attached to: Bridged Adapter*，选择你的物理网卡。
    *   适合“同局域网其他设备也要访问 VM”的场景（VirtualBox 虚拟网络章节）。 [\[virtualbox.org\]](https://www.virtualbox.org/manual/ch06.html)

> 不确定选哪个？——**单机开发测试**首选 NAT+端口转发；**需要让同网段设备访问**选 Bridged。官方文档对两者的行为和限制有详细说明。 [\[virtualbox.org\]](https://www.virtualbox.org/manual/ch06.html), [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/network_nat.html)

3.  **安装 Ubuntu**

*   启动 VM，按向导（语言、键盘、网络、磁盘分区）完成安装。ISO 来自 Ubuntu 官方。 [\[ubuntu.com\]](https://ubuntu.com/download/server)

4.  **（可选）安装 Guest Additions（主要用于桌面版/需要共享剪贴板与文件夹等）**  
    VirtualBox 菜单：**Devices → Insert Guest Additions CD image**，在来宾系统里安装以获得更好的集成体验。 [\[virtualbox.org\]](https://www.virtualbox.org/manual/ch04.html)

***

## 三、在 Ubuntu 来宾系统内：启用 SSH、安装 Docker

> 以下命令均在 **Ubuntu 虚拟机的终端** 中执行。

### 1) 启用 OpenSSH Server（用于 MobaXterm 连接）

```bash
# 更新包索引
sudo apt update
# 安装并启动 SSH 服务
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh    # 确认 Active (running)
```

*   使用 **UFW**（如开启）开放 22 与 80 端口：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw reload
```

### 2) 安装 Docker（官方 APT 仓库方法，适用于 Ubuntu 22.04/24.04）

> **强烈建议**使用 Docker 官方仓库而非发行版自带 `docker.io` 包，以获取最新稳定版本（Docker 官方文档）。 [\[docs.docker.com\]](https://docs.docker.com/engine/install/ubuntu/)

```bash
# 卸载可能冲突的旧包（如存在）
sudo apt remove -y docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc

# 先装基础工具与证书
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 添加 Docker 官方 GPG key 与源列表
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 写入官方源（自动取发行版代号，如 jammy）
echo "deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine、CLI、containerd、Buildx、Compose v2
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# 允许当前用户免 sudo 使用 docker（重新登录后生效）
sudo usermod -aG docker $USER
newgrp docker

# 验证安装
docker run --rm hello-world
```

> 以上步骤和包名称来自 Docker 官方安装指南（Ubuntu）。 [\[docs.docker.com\]](https://docs.docker.com/engine/install/ubuntu/)

***

## 四、在 Docker 里安装并运行 Nginx

1.  **拉取并运行官方 Nginx 镜像**（映射到来宾 80 端口）：

```bash
# 拉取镜像
docker pull nginx
# 后台运行，并将容器端口 80 映射到来宾系统的 80
docker run -d --name my-nginx -p 80:80 nginx
# 查看容器状态
docker ps
```

> 使用 **Docker Hub 官方 Nginx 镜像**，标签与使用说明见镜像主页。 [\[hub.docker.com\]](https://hub.docker.com/_/nginx)

2.  **（可选）自定义首页**  
    默认镜像首页在容器 `/usr/share/nginx/html`。如需自定义，可映射卷：

```bash
# 在来宾系统准备一个目录并放置 index.html
mkdir -p ~/nginx-web && echo "Hello from Nginx in Docker!" > ~/nginx-web/index.html
# 重新运行容器，挂载静态目录
docker rm -f my-nginx
docker run -d --name my-nginx -p 80:80 \
  -v ~/nginx-web:/usr/share/nginx/html:ro nginx
```

> 运行与端口映射参数用法同镜像说明。 [\[hub.docker.com\]](https://hub.docker.com/_/nginx)

***

## 五、在 Windows 上通过 MobaXterm（“xmobal”）验证连接与访问

> **MobaXterm** 是 Windows 下常用的 SSH/SFTP 客户端；下载安装后即可创建 SSH 会话。官方文档详述了 “Session → SSH → Remote host/Port/Username” 的配置。 [\[mobaxterm....obatek.net\]](https://mobaxterm.mobatek.net/documentation.html)

### 1) SSH 连接（两种网络模式下的地址与端口）

*   **若 VM 采用 NAT + 端口转发（推荐单机）**：
    *   在 VirtualBox 的 NAT 端口规则中，我们已将 `Host:2222 → Guest:22`。 [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/network_nat.html)
    *   打开 **MobaXterm** → **Session** → **SSH**：
        *   **Remote host**：`127.0.0.1`
        *   **Port**：`2222`
        *   **Specify username**：填写 Ubuntu 的用户名（如 `ubuntu`），确认连接。
    *   首次连接会提示主机指纹，接受后输入密码即可登录。（MobaXterm SSH 会话配置） [\[mobaxterm....obatek.net\]](https://mobaxterm.mobatek.net/documentation.html)

*   **若 VM 采用 Bridged（同 LAN）**：
    *   在 Ubuntu 来宾执行 `ip a` 获取 VM 的局域网 IP（如 `192.168.1.x`）。
    *   MobaXterm → SSH：**Remote host** 填该 IP，Port=`22`，用户名照填。
    *   这时你也可以从其他同网段设备访问该 VM。（VirtualBox 网络模式说明） [\[virtualbox.org\]](https://www.virtualbox.org/manual/ch06.html)

### 2) 在 Windows 上访问 Docker 中的 Nginx 首页

*   **NAT + 端口转发**：
    *   我们已经把 `Host:8080 → Guest:80`。在 **Windows 浏览器**或 **MobaXterm 终端**访问：
        *   浏览器：`http://127.0.0.1:8080`
        *   终端：`curl http://127.0.0.1:8080`
    *   若页面显示默认 Nginx 欢迎页或你自定义的 `index.html`，即验证成功。（NAT 端口转发原理） [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/network_nat.html)

*   **Bridged**：
    *   直接在浏览器访问 `http://<VM的LAN IP>:80` 或 `curl http://<VM的LAN IP>`。

***

## 六、常见问题与排查

1.  **Windows 上 VirtualBox 与 Hyper‑V 并存**

*   若发现第三方虚拟机性能下降或无法启动，可考虑临时关闭 Hyper‑V（命令提示符管理员）：
    ```cmd
    bcdedit /set hypervisorlaunchtype off
    ```
    重启；需要恢复再设为 `auto`。

2.  **NAT 下无法从 Windows 访问 VM 服务**

*   检查 **Port Forwarding** 规则是否正确（特别是协议 TCP/端口号），并确保 VM 内服务监听在 `0.0.0.0`（Nginx 默认如此）。官方文档强调 NAT 下必须做端口转发才能从主机访问来宾服务。 [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/network_nat.html)

3.  **SSH 连接失败**

*   Ubuntu 内确认 `sshd` 运行：`systemctl status ssh`；
*   防火墙（UFW）是否开放 22/80；
*   NAT 模式下确认使用 `127.0.0.1:2222`；Bridged 模式下确认 VM 所获 IP 与网段策略允许访问。
*   MobaXterm 会话参数（Remote host/Port/Username/私钥）填写正确（MobaXterm 文档）。 [\[mobaxterm....obatek.net\]](https://mobaxterm.mobatek.net/documentation.html)

4.  **Docker 安装问题**

*   按 Docker 官方指南确认已移除冲突包、GPG key 与源配置正确、安装的包名称无误；运行 `docker run hello-world` 验证。 [\[docs.docker.com\]](https://docs.docker.com/engine/install/ubuntu/)

***

## 七、你可以直接跟着做的“速查清单”

**A. VirtualBox 新建 VM（NAT 模式）**

*   Port Forwarding：`Host 2222 → Guest 22`，`Host 8080 → Guest 80`。 [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/network_nat.html)
*   安装 Ubuntu 22.04 Server（官方 ISO）。 [\[ubuntu.com\]](https://ubuntu.com/download/server)

**B. Ubuntu 来宾系统命令：**

```bash
# SSH
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo ufw allow 22/tcp && sudo ufw allow 80/tcp

# Docker（官方仓库）
sudo apt remove -y docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc
sudo apt update && sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
docker run --rm hello-world

# Nginx 容器
docker run -d --name my-nginx -p 80:80 nginx
```

> Docker（Ubuntu）与 Nginx（Docker Hub 官方镜像）参考文档。 [\[docs.docker.com\]](https://docs.docker.com/engine/install/ubuntu/), [\[hub.docker.com\]](https://hub.docker.com/_/nginx)

**C. Windows（MobaXterm）**

*   SSH：`127.0.0.1:2222`（NAT）或 `<VM的LAN IP>:22`（Bridged），用户名为 Ubuntu 的用户。 [\[mobaxterm....obatek.net\]](https://mobaxterm.mobatek.net/documentation.html)
*   访问 Nginx：浏览器打开 `http://127.0.0.1:8080`（NAT）或 `http://<VM的LAN IP>:80`（Bridged）。 [\[docs.oracle.com\]](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/network_nat.html)

***

如果你更偏向 **Bridged**（让手机或同事电脑也能访问你的 Nginx），我可以帮你核对本机网卡与 DHCP 情况，给出针对你局域网的网段与防火墙配置建议；或者你希望 **NAT 模式**更安全，我也能直接帮你生成 VirtualBox 的 **端口转发规则截图式引导**。你目前打算用 **NAT** 还是 **Bridged** 呢？
