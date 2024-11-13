---
title: CentOS证书登录
date: 2024-11-13 15:30:00
categories:
  - Linux
tags:
  - Linux
---

在 CentOS 上配置 SSH 免密证书登录是一种常见且安全的做法。它允许用户通过公钥认证而非密码来访问服务器，这不仅提高了安全性，还简化了远程登录过程。以下是配置步骤：

### 准备工作

确保你的系统已经安装了 `openssh` 和 `openssh-server`，并且 SSH 服务正在运行。

### 1. 生成密钥对

首先，在客户端机器上生成 SSH 密钥对（如果你还没有的话）。这可以通过执行以下命令完成：

```bash
ssh-keygen -t rsa -b 4096
```

- `-t rsa` 指定了密钥类型为 RSA。
- `-b 4096` 表示密钥长度为 4096 位，更大的密钥提供更好的安全性。

按提示操作，通常可以直接按回车使用默认设置。这个过程会生成两个文件：
- `~/.ssh/id_rsa` 是私钥文件，应妥善保管。
- `~/.ssh/id_rsa.pub` 是公钥文件，将用于服务器端的验证。

### 2. 将公钥复制到服务器

有几种方法可以将公钥复制到服务器，最简单的是使用 `ssh-copy-id` 命令：

```bash
ssh-copy-id user@server_ip
```

其中 `user` 是你在服务器上的用户名，`server_ip` 是服务器的 IP 地址。

如果 `ssh-copy-id` 不可用或你希望手动完成此步骤，你可以直接编辑服务器上的 `~/.ssh/authorized_keys` 文件，并将公钥内容粘贴进去。例如：

```bash
cat ~/.ssh/id_rsa.pub | ssh user@server_ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 3. 配置服务器的 SSH 设置

为了确保 SSH 服务支持公钥认证，你需要检查并可能修改服务器上的 `/etc/ssh/sshd_config` 文件。确保以下几行未被注释且设置正确：

```bash
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no  # 可选，禁止密码登录以提高安全性
ChallengeResponseAuthentication no  # 可选
UsePAM no  # 可选
```

更改配置后，重启 SSH 服务使改动生效：

```bash
sudo systemctl restart sshd
```

### 4. 测试连接

现在，尝试从客户端无密码登录到服务器：

```bash
ssh user@server_ip
```

如果一切配置正确，你应该能够直接登录而无需输入密码。

### 注意事项

- 确保 `.ssh` 目录和 `authorized_keys` 文件的权限是正确的。`.ssh` 目录应该是 700 (`drwx------`)，而 `authorized_keys` 文件应该是 600 (`-rw-------`)。
- 如果遇到问题，可以通过查看服务器上的 `/var/log/secure` 日志文件来获取更多信息。

以上步骤完成后，你就成功地在 CentOS 上实现了 SSH 的免密证书登录。
