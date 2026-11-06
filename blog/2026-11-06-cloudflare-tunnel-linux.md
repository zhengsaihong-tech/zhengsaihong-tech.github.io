---
title: Cloudflare Tunnel 教程：用域名访问 Linux 内网服务
description: 不需要公网 IP，也不开放路由器端口，通过 Cloudflare Tunnel 把 Linux 上的网站、管理面板或 SSH 安全地映射到自己的域名。
tags: [Cloudflare, Linux, 内网穿透]
---

这篇教程介绍如何使用 **Cloudflare Tunnel**，把真实 Linux 电脑上的内网服务绑定到自己的域名。

整个过程不需要公网 IP，不需要 DDNS，也不需要在路由器上开放入站端口。Linux 电脑只需主动连接 Cloudflare。

本文先完成最常见的 Web 服务映射：

```text
https://app.example.com
          ↓
Cloudflare 网络
          ↓ Cloudflare Tunnel
Linux 电脑上的 http://127.0.0.1:8080
```

完成后，再根据需要配置 Cloudflare Access 或 SSH。

{/* truncate */}

## 一、开始前需要准备什么

你需要：

- 一台可以正常联网并长期运行的 Linux 电脑。
- 一个 Cloudflare 账号。
- 一个自己的域名，并且可以修改这个域名的 DNS 服务器。
- Linux 上已经运行的服务，例如网站、管理面板或 API。
- Linux 的 `sudo` 或 `root` 权限。

本文使用以下示例：

| 项目 | 示例值 |
|---|---|
| 域名 | `example.com` |
| 对外访问地址 | `app.example.com` |
| Linux 本地服务 | `http://127.0.0.1:8080` |
| Tunnel 名称 | `linux-home` |

请将示例域名和端口替换成自己的实际值。

:::info
域名不能直接“绑定整台电脑”。Cloudflare Tunnel 映射的是电脑上的具体服务，例如 `8080` 端口的网站或 `22` 端口的 SSH。
:::

## 二、检查 Linux 上的本地服务

先确认要发布的服务在 Linux 本机可以正常访问。本文以 `8080` 端口为例：

```bash
curl -I http://127.0.0.1:8080
```

查看端口监听情况：

```bash
sudo ss -lntp
```

正常情况下，可以看到类似内容：

```text
LISTEN 0 4096 127.0.0.1:8080 0.0.0.0:*
```

绑定到 `127.0.0.1` 没有问题，因为 `cloudflared` 和目标服务运行在同一台 Linux 电脑上。

如果 `curl` 访问失败，先修复本地服务。Cloudflare Tunnel 只能转发流量，不能让一个没有启动的服务恢复工作。

## 三、把域名接入 Cloudflare

如果域名已经在 Cloudflare 控制台中显示为 `Active`，可以直接进入下一节。

1. 登录 [Cloudflare 控制台](https://dash.cloudflare.com/)。
2. 添加自己的根域名，例如 `example.com`。
3. 选择套餐，个人使用通常选择 Free 即可。
4. 检查 Cloudflare 扫描出的 DNS 记录。
5. 前往域名注册商，将域名的 Nameserver 修改为 Cloudflare 提供的两个地址。
6. 返回 Cloudflare，等待域名状态变为 `Active`。

修改 Nameserver 前，重点确认原来的 MX、TXT、A、AAAA 和 CNAME 记录已经完整导入。如果域名正在提供网站或邮箱服务，遗漏记录可能导致现有服务中断。

建议先使用 `app.example.com` 这样的子域名，不要一开始就使用根域名 `example.com`。

## 四、创建 Cloudflare Tunnel

在 Cloudflare 控制台进入：

```text
Zero Trust → Networking → Tunnels
```

也可以直接打开 [Cloudflare Tunnels](https://dash.cloudflare.com/?to=/:account/tunnels)。

然后按下面的顺序操作：

1. 点击 **Create a tunnel**。
2. 选择 **Cloudflared**。
3. 输入 Tunnel 名称，例如 `linux-home`。
4. 点击 **Create Tunnel**。
5. 选择 Linux 和当前电脑对应的 CPU 架构。
6. 复制页面生成的安装命令。
7. 在需要穿透的 Linux 电脑上执行这条命令。

控制台生成的命令通常类似：

```bash
sudo cloudflared service install <TUNNEL_TOKEN>
```

实际使用时不要照抄 `<TUNNEL_TOKEN>`，必须执行控制台为当前 Tunnel 生成的完整命令。

:::warning
Tunnel Token 是连接凭证。不要把完整命令、Token、终端截图或日志公开到博客、群聊和代码仓库。
:::

如果系统尚未安装 `cloudflared`，先按照页面给出的命令安装。也可以参考 Cloudflare 的 [cloudflared 下载文档](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/)。

安装后检查版本：

```bash
cloudflared --version
```

检查服务状态：

```bash
sudo systemctl status cloudflared --no-pager
```

设置开机启动并立即运行：

```bash
sudo systemctl enable --now cloudflared
```

回到 Cloudflare 控制台。等待 Connector 显示已连接，然后继续下一步。Tunnel 最终应显示为 `Healthy`。

## 五、把域名映射到本地服务

打开刚才创建的 Tunnel：

```text
Networking → Tunnels → linux-home → Routes
```

然后：

1. 点击 **Add route**。
2. 选择 **Published application**。
3. 在 Subdomain 中填写 `app`。
4. 在 Domain 中选择 `example.com`。
5. Path 保持为空。
6. 在 Service URL 中填写 `http://localhost:8080`。
7. 点击 **Add route**。

最终配置关系是：

```text
app.example.com → http://localhost:8080
```

Cloudflare 会自动为这个主机名创建指向 Tunnel 的代理 DNS 记录。通常不需要自己创建 A 记录，也不需要填写家里的公网 IP。

如果 DNS 中已经存在同名的 `app` A、AAAA 或 CNAME 记录，需要先处理记录冲突。

## 六、验证是否成功

先在 Linux 本机测试：

```bash
curl -I http://127.0.0.1:8080
```

再测试外部域名：

```bash
curl -I https://app.example.com
```

也可以直接用浏览器打开：

```text
https://app.example.com
```

外部用户到 Cloudflare 之间的 HTTPS 证书由 Cloudflare 自动处理。本地服务可以继续使用 HTTP，因为示例中的这一段连接只经过 Linux 本机的回环地址。

到这里，基础的 Web 内网穿透已经完成。整个过程不需要：

- 公网 IP。
- DDNS。
- 路由器端口转发。
- 对公网开放 Linux 的 `80` 或 `443` 端口。

## 七、给管理后台增加登录保护

如果映射的是博客、公开网站或公开 API，可以直接开放。如果映射的是 NAS、监控系统、内部工具或管理面板，不建议直接暴露给所有互联网用户。

可以使用 Cloudflare Access 增加身份验证：

```text
Zero Trust
→ Access controls
→ Applications
→ Create new application
→ Self-hosted and private
→ Add public hostname
```

填写前面使用的主机名：

```text
app.example.com
```

然后创建 `Allow` 策略，只允许自己的邮箱、指定邮箱域名或指定身份提供商账号。

Cloudflare Access 默认拒绝没有匹配 `Allow` 策略的用户。配置完成后，访问者必须先通过 Cloudflare 登录验证，才能进入 Linux 上的服务。

建议先创建 Access 应用，再创建 Tunnel 的公开路由，避免管理后台在配置期间短暂暴露给所有人。

## 八、通过域名连接 SSH

如果需要远程管理这台 Linux，可以在同一个 Tunnel 中再添加一条路由。

先确认 SSH 服务正常运行。

Ubuntu 或 Debian：

```bash
sudo systemctl status ssh
```

RHEL、CentOS 或 Rocky Linux：

```bash
sudo systemctl status sshd
```

在 Tunnel 的 Routes 页面增加一个 **Published application**：

```text
Hostname: ssh.example.com
Service:  SSH
Address:  localhost:22
```

同时为 `ssh.example.com` 创建 Cloudflare Access 应用，只允许自己的账号访问。

用于连接服务器的客户端电脑也需要安装 `cloudflared`。先确认其安装路径：

```bash
command -v cloudflared
```

编辑客户端的 `~/.ssh/config`：

```text
Host ssh.example.com
    ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
```

如果 `command -v cloudflared` 返回的是 `/usr/bin/cloudflared`，就把上面的路径改为 `/usr/bin/cloudflared`。

随后连接：

```bash
ssh <Linux用户名>@ssh.example.com
```

首次连接时，`cloudflared` 会打开浏览器进行 Cloudflare Access 身份验证。通过验证后，SSH 客户端才会建立连接。

不需要在路由器或 Linux 防火墙中向公网开放 `22` 端口。

## 九、映射多个服务

一台 Linux 上的多个服务不需要分别创建 Tunnel。直接在同一个 Tunnel 中添加多条路由即可：

```text
app.example.com    → http://localhost:8080
nas.example.com    → http://localhost:5000
ssh.example.com    → ssh://localhost:22
```

如果 `cloudflared` 和目标服务不在同一台设备上，Service URL 中不能再写 `localhost`，应填写目标设备在局域网中的地址，例如：

```text
http://192.168.1.20:8080
```

## 十、Docker 服务怎么填写地址

如果 `cloudflared` 作为 systemd 服务运行在 Linux 宿主机上，而应用运行在 Docker 中，需要先把应用端口发布到宿主机。推荐只监听本机：

```yaml
ports:
  - "127.0.0.1:8080:8080"
```

Tunnel 中继续填写：

```text
http://localhost:8080
```

如果 `cloudflared` 也运行在 Docker 中，`localhost` 指的是 `cloudflared` 容器自己。把两个容器放在同一个 Docker 网络后，使用目标容器名：

```text
http://my-app:8080
```

## 十一、常见故障排查

### Tunnel 显示 Inactive、Down 或 Degraded

查看服务状态和日志：

```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared --no-pager
sudo journalctl -u cloudflared -n 200 --no-pager
```

Cloudflare Tunnel 是由 Linux 主动建立的出站连接。在受限网络中，需要确认服务器可以连接 Cloudflare，尤其是 TCP 或 UDP `7844` 端口，同时保留正常的 DNS 和 HTTPS 出站访问。

### 访问域名返回 502 Bad Gateway

这通常说明 Tunnel 已经连接，但 `cloudflared` 无法访问本地服务。

执行：

```bash
curl -v http://127.0.0.1:8080
sudo ss -lntp | grep 8080
```

检查以下内容：

- 应用是否已经启动。
- 本地端口是否正确。
- Tunnel 中填写的是 `http` 还是 `https`。
- `cloudflared` 和目标服务是否真的在同一台机器或同一个 Docker 网络中。

### 域名打不开或 DNS 冲突

检查 Cloudflare DNS 中是否已经存在同名记录：

```bash
dig app.example.com
```

如果同一个主机名已有 A、AAAA 或其他 CNAME 记录，需要删除或更换子域名后重新添加 Tunnel 路由。

### 本地 HTTPS 证书报错

如果应用同时支持 HTTP，优先在 Service URL 中使用：

```text
http://localhost:8080
```

外部访问仍然是 HTTPS。不要为了绕过错误而长期关闭源站证书验证。

## 十二、什么时候不应该使用公开主机名

如果目标是访问整个家庭局域网，或者需要直接连接多台设备的私有 IP，不要为每个端口都创建一个公网域名。

这种场景应该配置 Cloudflare Tunnel 的私有网络路由，并在客户端使用 Cloudflare One Client。公开主机名更适合网站、面板、API 和 SSH 这类明确的单项服务。

## 官方参考资料

- [创建 Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)
- [安装 cloudflared](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/)
- [保护自建应用](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)
- [通过 cloudflared 使用 SSH](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/use-cases/ssh/ssh-cloudflared-authentication/)
