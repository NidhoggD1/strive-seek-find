# Codex SSH 远程连接配置与问题总结

## 0. 前置条件

当前CodeX SSH功能处于beta阶段，需手动开启feature flag

编辑本地 Codex 配置文件(或直接从CodeX设置中打开)：

```sh
~/.codex/config.toml
```

加入如下内容：

```json
[features]
remote_control = true
remote_connections = true
```

## 1. 目标

在 Windows 本地使用 Codex 通过 SSH 连接远端 Linux 服务器 `<SSH_HOST_ALIAS>`，并让 Codex 能在远端环境中运行。

核心链路：

```text
Windows 本地 Codex / PowerShell
        ↓ SSH
远端服务器 <SSH_HOST_ALIAS>（SSH 配置中的 Host 别名）
        ↓ 远端需要能执行 codex
Codex CLI / app-server
```

## 2. Windows 本地 SSH 配置位置

Windows 用户级 SSH 配置文件通常是：

```powershell
C:\Users\<WindowsUser>\.ssh\config
```

也可以用 PowerShell 打开：

```powershell
notepad $env:USERPROFILE\.ssh\config
```

示例配置：

```sshconfig
Host <SSH_HOST_ALIAS>
    HostName <SERVER_HOSTNAME_OR_IP>
    User <REMOTE_USER>
    Port 22
    IdentityFile C:\Users\<WindowsUser>\.ssh\id_ed25519
```

占位符说明：

- `<SSH_HOST_ALIAS>`：Windows `~/.ssh/config` 中的 `Host` 别名，例如 `devbox`、`remote-server`。
- `<SERVER_HOSTNAME_OR_IP>`：服务器 IP 或域名。
- `<REMOTE_USER>`：远端登录用户名，例如 `root` 或普通用户。
- `<WindowsUser>`：Windows 用户名；实际命令中也可以优先使用 `$env:USERPROFILE`，避免写死路径。


测试：

```powershell
ssh <SSH_HOST_ALIAS>
```

如果能成功登录，说明 SSH 主机别名、用户名、私钥基本可用。

## 3. SSH 密钥配置流程

### 3.1 Windows 本地生成密钥

```powershell
ssh-keygen -t ed25519 -C "codex-remote"
```

生成时会提示：

```text
Enter file in which to save the key (C:\Users\<WindowsUser>/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
```

说明：

- `id_ed25519` 是私钥，不能泄露。
- `id_ed25519.pub` 是公钥，可以放到服务器。
- `passphrase` 是私钥口令，不是服务器密码。可以直接回车不设置，也可以设置。

### 3.2 查看 Windows 本地公钥

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

输出类似：

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... codex-remote
```

### 3.3 加到服务器 authorized_keys

登录服务器后：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
vim ~/.ssh/authorized_keys
```

把 `id_ed25519.pub` 的整行内容追加到文件末尾。

或者：

```bash
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... codex-remote' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

然后 Windows 本地测试：

```powershell
ssh <SSH_HOST_ALIAS>
```

## 4. Codex 远程连接的关键要求

Codex CLI 官方安装方式是：

```bash
npm i -g @openai/codex
```

安装后远端应该能执行：

```bash
codex --version
```

对于 Codex SSH 连接，不能只看手动登录是否成功，还要看 Windows 本地通过非交互 SSH 是否能找到远端 `codex`：

```powershell
ssh <SSH_HOST_ALIAS> "which codex && codex --version"
```

这条命令有输出，才说明 Codex App 通过 SSH 启动远端命令时大概率能找到 `codex`。

## 5. 本次遇到的问题与分析

### 问题 1：SSH 已经能登录，但 Codex 提示更新连接失败

现象：

```powershell
ssh <SSH_HOST_ALIAS>
```

可以成功登录，但 Codex 里更新连接失败。

分析：

SSH 登录成功只说明认证没问题，不代表 Codex 远程环境满足要求。Codex 还需要远端能够执行 `codex` 命令，并且远端 SSH 环境可能需要支持端口转发。

排查命令：

```powershell
ssh <SSH_HOST_ALIAS> "which codex && codex --version"
ssh <SSH_HOST_ALIAS> "sshd -T | grep -i allowtcpforwarding"
```

### 问题 2：npm 启动时报 `Cannot find module 'semver'`

现象：

```bash
npm install -g @openai/codex
```

报错：

```text
Error: Cannot find module 'semver'
Require stack:
- /usr/share/nodejs/npm/lib/utils/unsupported.js
- /usr/share/nodejs/npm/lib/cli.js
- /usr/share/nodejs/npm/bin/npm-cli.js
```

分析：

Node 能运行，但 npm 自身损坏或混用了系统 npm。典型表现是：

```bash
which node
which npm
node -v
npm -v
```

看到 `node` / `npm` 路径在 `/usr/local/bin`，但 npm 实际加载了 `/usr/share/nodejs/npm/...`。

本次进一步检查：

```bash
ls -l /usr/local/bin/node
ls -l /usr/local/bin/npm
ls -l /usr/local/bin/npx
readlink -f /usr/local/bin/node
readlink -f /usr/local/bin/npm
```

输出显示：

```text
/usr/local/bin/node -> /usr/local/nodejs/bin/node
/usr/local/bin/npm  -> /usr/local/nodejs/bin/npm
/usr/local/bin/npx  -> /usr/local/nodejs/bin/npx
/usr/local/nodejs/bin/node
/usr/local/nodejs/lib/node_modules/npm/bin/npm-cli.js
```

但 npm 仍然加载 `/usr/share/nodejs/npm`，说明 `/usr/local/nodejs` 里的 npm 内容可能损坏或混入了系统 npm 依赖。

解决思路：

重新安装官方 Node.js 二进制包，并确保 `node/npm/npx` 都来自同一个 `/usr/local/nodejs`。

示例：

```bash
cd /tmp
wget https://nodejs.org/dist/v18.20.0/node-v18.20.0-linux-arm64.tar.xz
tar -xf node-v18.20.0-linux-arm64.tar.xz
sudo mv node-v18.20.0-linux-arm64 /usr/local/nodejs

sudo ln -sf /usr/local/nodejs/bin/node /usr/local/bin/node
sudo ln -sf /usr/local/nodejs/bin/npm /usr/local/bin/npm
sudo ln -sf /usr/local/nodejs/bin/npx /usr/local/bin/npx
hash -r
```

检查：

```bash
which node
which npm
node -v
npm -v
readlink -f $(which node)
readlink -f $(which npm)
```

### 问题 3：`npm install -g @openai/codex` 成功，但 `codex` 找不到

现象：

```bash
npm install -g @openai/codex
```

输出：

```text
added 2 packages in 18s
```

但：

```bash
which codex
codex --version
```

输出：

```text
bash: codex: command not found
```

排查：

```bash
npm prefix -g
npm root -g
ls -l "$(npm prefix -g)/bin"
find "$(npm prefix -g)" -name codex -type f 2>/dev/null
```

本次结果：

```text
npm prefix -g -> /usr/local/nodejs
npm root -g   -> /usr/local/nodejs/lib/node_modules
/usr/local/nodejs/bin/codex -> ../lib/node_modules/@openai/codex/bin/codex.js
```

分析：

Codex 已经安装成功，但 `/usr/local/nodejs/bin` 不在当前 PATH 中。

修复：

```bash
/usr/local/nodejs/bin/codex --version
sudo ln -sf /usr/local/nodejs/bin/codex /usr/local/bin/codex
hash -r
which codex
codex --version
```

也可以加 PATH：

```bash
echo 'export PATH=/usr/local/nodejs/bin:$PATH' >> ~/.bashrc
echo 'export PATH=/usr/local/nodejs/bin:$PATH' >> ~/.profile
source ~/.bashrc
```

注意：Codex 远程连接可能走非交互 SSH，所以仅写 `.bashrc` 不一定够，需要用 Windows 本地命令验证：

```powershell
ssh <SSH_HOST_ALIAS> "which codex && codex --version"
```

### 问题 4：`hash -r` 是不是刷新软链？

不是。

```bash
hash -r
```

作用是清空当前 Bash 对命令路径的缓存，让 shell 下次重新根据 `$PATH` 查找命令。

创建或更新软链接的命令是：

```bash
ln -sf <源文件> <目标软链接>
```

例如：

```bash
sudo ln -sf /usr/local/nodejs/bin/codex /usr/local/bin/codex
hash -r
```

含义：

- `ln -sf ...`：创建/更新软链接。
- `hash -r`：让当前 shell 忘掉旧命令路径缓存。

### 问题 5：Windows 本地 SSH 找不到 `/usr/local/bin/codex`

现象：

Windows 本地执行：

```powershell
ssh <SSH_HOST_ALIAS> "/usr/local/bin/codex --version"
```

报错：

```text
bash: line 1: /usr/local/bin/codex: No such file or directory
```

但在“服务器”里执行：

```bash
ls -l /usr/local/bin/codex
/usr/local/bin/codex --version
```

可以看到 Codex。

最终确认：这些 Node/Codex 配置是在 Docker 容器里做的。

根因：

Windows 的：

```powershell
ssh <SSH_HOST_ALIAS>
```

登录的是服务器宿主机，而不是 Docker 容器。

所以：

```powershell
ssh <SSH_HOST_ALIAS> "/usr/local/bin/codex --version"
```

访问的是宿主机的 `/usr/local/bin/codex`，不是容器里的 `/usr/local/bin/codex`。

因此报 `No such file or directory` 是合理的。

## 6. Docker 对 Codex SSH 连接的影响

当前实际链路是：

```text
Windows 本地
  ↓ ssh <SSH_HOST_ALIAS>
远端宿主机

Docker 容器中的 codex 不在这个 SSH 会话里
```

而你实际安装 Codex 的位置是：

```text
Docker 容器内部 /usr/local/nodejs/bin/codex
```

因此 Codex App 连接 `<SSH_HOST_ALIAS>` 时不会自动进入 Docker，也不会自动看到容器内的 `codex`。

## 7. 后续推荐方案

### 方案一：在宿主机安装 Codex，推荐

如果 Codex SSH Host 配置的是 `<SSH_HOST_ALIAS>`，那么应在 `<SSH_HOST_ALIAS>` 宿主机安装 Codex。

宿主机执行：

```bash
which node
which npm
node -v
npm -v
npm install -g @openai/codex
which codex
codex --version
```

Windows 本地验证：

```powershell
ssh <SSH_HOST_ALIAS> "which codex && codex --version"
```

这条有输出后，再回 Codex 更新连接。

### 方案二：宿主机包装 codex，转发到 Docker 容器，不一定完全兼容

如果必须让 Codex 跑在容器里，可以在宿主机创建包装脚本：

```bash
sudo tee /usr/local/bin/codex >/dev/null <<'WRAPPER_EOF'
#!/bin/bash
docker exec -i <容器名或容器ID> /usr/local/bin/codex "$@"
WRAPPER_EOF

sudo chmod +x /usr/local/bin/codex
```

然后 Windows 本地测试：

```powershell
ssh <SSH_HOST_ALIAS> "which codex && codex --version"
```

但这个方案有风险：Codex 远程连接可能不仅执行 `codex --version`，还可能涉及端口转发、工作目录、文件系统访问、app-server 等能力。`docker exec` 包装不一定完全兼容。

### 方案三：宿主机装 Codex，代码目录挂载进容器

更稳的工程实践：

```text
Codex 连接宿主机
Codex 修改宿主机上的代码目录
Docker 容器挂载同一个代码目录进行编译/运行
```

这样 Codex 不需要直接进入容器，但仍可以配合容器环境完成构建和测试。

## 8. 最小检查清单

### Windows 本地检查

```powershell
ssh <SSH_HOST_ALIAS>
ssh <SSH_HOST_ALIAS> "hostname; whoami; echo \$PATH"
ssh <SSH_HOST_ALIAS> "which codex && codex --version"
ssh <SSH_HOST_ALIAS> "sshd -T | grep -i allowtcpforwarding"
```

### 远端宿主机检查

```bash
which node
which npm
node -v
npm -v
which codex
codex --version
ls -l /usr/local/bin/codex
```

### Docker 容器内检查

```bash
which node
which npm
node -v
npm -v
which codex
codex --version
```

### 判断当前是在宿主机还是容器

```bash
hostname
cat /proc/1/cgroup | head
ls /.dockerenv 2>/dev/null && echo "in docker"
```

## 9. 本次结论

本次 Codex SSH 连接失败的关键原因不是 SSH 密钥，而是：

1. Windows 本地 SSH 到 `<SSH_HOST_ALIAS>` 登录的是远端宿主机。
2. Node.js / npm / Codex CLI 的安装和修复是在 Docker 容器中完成的。
3. 宿主机上没有 `/usr/local/bin/codex`，所以 Windows 本地执行：

```powershell
ssh <SSH_HOST_ALIAS> "/usr/local/bin/codex --version"
```

会报：

```text
No such file or directory
```

最终建议：

- 如果 Codex 连接目标是 `<SSH_HOST_ALIAS>`，就在该 SSH Host 对应的宿主机上安装 Codex。
- 如果实际开发环境必须在 Docker 内，优先考虑“宿主机 Codex + 代码目录挂载进容器”的方式。
- 每次改完后，都用 Windows 本地验证：

```powershell
ssh <SSH_HOST_ALIAS> "which codex && codex --version"
```

只有这条命令成功，Codex SSH 更新连接才有较大概率成功。

## 10. 参考

- OpenAI Codex CLI 官方文档：Codex CLI 可通过 `npm i -g @openai/codex` 安装，安装后运行 `codex`，首次运行会要求登录。
- OpenAI Codex Windows 文档：Windows 可原生运行 Codex，也可使用 WSL2。
- OpenAI Codex changelog：2026 年 4 月的 Codex CLI 版本持续更新，远程连接仍在迭代中。
