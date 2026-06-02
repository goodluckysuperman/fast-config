# fast-config

把本机 WSL Ubuntu 里的终端工作环境快速迁移到新租的 Ubuntu 服务器。

默认目标是 root 服务器环境：远端 home 为 `/root`，程序入口写到 `/usr/local/bin`，程序目录写到 `/opt/fast-config` 或 `/root/.local`。

## 快速使用

最常用的一条命令：

```bash
./fast-sync 3090-1
```

`3090-1` 是你的 SSH 别名。主机名必须写，避免误同步到旧服务器。

这条命令默认等价于：

```bash
./fast-sync --offline --ai 3090-1
```

也就是：

```text
1. 同步 nvim/yazi/zellij 配置、程序本体、插件缓存
2. 同步 Codex/Claude 安装和配置
3. 自动创建远端命令入口
4. 自动选择 rsync 快路径；远端没有 rsync 时回退 tar 管道
```

同步完成后登录服务器验证：

```bash
ssh 3090-1
nvim --version
yazi --version
zellij --version
codex --version
claude --version
source ~/.bashrc
yz
```

以后本地配置更新了，继续跑同一条：

```bash
./fast-sync 3090-1
```

## 常用模式

完整离线迁移，包含终端环境和 AI 工具：

```bash
./fast-sync 3090-1
```

只同步终端环境，不同步 Codex/Claude：

```bash
./fast-sync --offline 3090-1
```

只补同步 Codex/Claude：

```bash
./fast-sync --ai 3090-1
```

如果前面终端环境已经成功，只是 AI 部分失败，直接补跑：

```bash
./fast-sync --ai 3090-1
```

预览动作：

```bash
./fast-sync --dry-run 3090-1
```

监听本地配置变化并自动同步：

```bash
./fast-sync --watch 3090-1
```

`--watch` 需要本机安装：

```bash
sudo apt install inotify-tools
```

## 同步内容

终端环境会同步：

```text
~/.config/nvim    -> /root/.config/nvim
~/.config/yazi    -> /root/.config/yazi
~/.config/zellij  -> /root/.config/zellij

本机 nvim 程序目录  -> /opt/fast-config/nvim
本机 yazi 程序目录  -> /opt/fast-config/yazi
本机 zellij 程序    -> /usr/local/bin/zellij

~/.local/share/nvim
~/.local/state/nvim
~/.cache/nvim
~/.local/share/yazi
~/.local/state/yazi
~/.cache/yazi
~/.local/share/zellij
~/.cache/zellij
```

远端会创建：

```text
/usr/local/bin/nvim
/usr/local/bin/yazi
/usr/local/bin/ya
/usr/local/bin/zellij
```

缺少的本地目录会跳过并提示 `skip missing ...`，不会中断。

## Codex 和 Claude

`--ai` 会同步：

```text
~/.codex
~/.claude
~/.claude.json
~/.local/share/claude
当前 codex 使用的 Node 运行时和 npm 全局包
```

远端会创建：

```text
/usr/local/bin/codex
/usr/local/bin/claude
```

Codex 如果是 nvm/npm 全局安装，脚本会从 `codex` 入口一路向上查找最近的 `bin/node`，然后把这个 Node 环境同步到远端。这样远端不需要自己安装 Node 或 npm 包。

这部分当前本机估算约 `1.6G`，而且包含登录状态、token、历史记录等敏感信息。只建议同步到你完全信任的自用 root 服务器。

退租服务器前建议删除：

```bash
rm -rf /root/.codex /root/.claude /root/.claude.json /root/.local/share/claude
```

## yz 命令

脚本会在远端 `/root/.bashrc` 写入一个 `yz()` shell 函数。

这个 `yz` 用来实现：退出 yazi 后，当前 shell 自动进入 yazi 最后停留的目录。

使用：

```bash
source ~/.bashrc
yz
```

`ya` 是 yazi 自带 helper 程序，不能替代这个目录跳转功能。

## SSH 别名

脚本不解析服务器名字。你传什么 host，它就原样交给 `ssh`、`rsync` 或 `tar` 管道，所以完全遵循 `~/.ssh/config`。

例如：

```sshconfig
Host 5090-1
  HostName 1.2.3.4
  User root
  IdentityFile ~/.ssh/id_ed25519
```

直接运行：

```bash
./fast-sync 5090-1
```

## 传输方式

脚本会自动选择传输方式：

```text
本地和远端都有 rsync -> rsync 快速增量同步
远端没有 rsync        -> tar | ssh | tar
```

执行输出会显示当前方式：

```text
==> [rsync] ...
==> [tar] ...
```

### rsync 快路径

`rsync` 参数是：

```bash
rsync -avzP --delete --no-owner --no-group
```

含义：

```text
-a                    归档模式，保留目录结构、权限、软链接等
-v                    显示传输过程
-z                    通过 SSH 传输时压缩数据
-P                    显示进度，并保留未完成的部分传输
--delete              远端目标目录严格跟随本地，本地删了远端也删
--no-owner --no-group 不把本地用户/用户组强行带到远端 root 环境
```

脚本没有使用 `--mkpath`、ACL、xattr 等更激进参数，避免因为 rsync 版本或文件系统差异引入问题。

### tar 回退路径

远端没有 `rsync` 时，不会失败，而是用：

```bash
tar -C 本地父目录 -cpf - 目录名 \
  | ssh 3090-1 'tar -C 远端目录 -xpf -'
```

本地 `tar` 把目录写到标准输出，管道把数据交给 `ssh`，远端 `tar` 从标准输入解包。

脚本实际流程：

```text
1. 本地 tar 打包目标目录
2. 通过 ssh 把 tar 数据流送到服务器
3. 远端在目标目录旁边创建临时目录
4. 远端 tar 先解包到临时目录
5. 解包成功后删除旧目标，再移动到目标路径
```

所以即使服务器没有 `rsync`，只要有 Ubuntu 常见的 `sh`、`tar`、`mktemp`、`uname`、`awk`，也可以完成同步。

## 远端命令执行原理

`ssh HOST 'command'` 的语义就是在远端执行命令。例如：

```bash
ssh 3090-1 'uname -m'
ssh 3090-1 'mkdir -p /root/.config'
ssh 3090-1 'ln -sf /opt/fast-config/nvim/bin/nvim /usr/local/bin/nvim'
```

脚本只是把这些命令组织起来，并通过 SSH 把本地文件流送到远端。

## 其它选项

远端 home 默认是 `/root`。可以显式指定：

```bash
./fast-sync 3090-1:/root
./fast-sync --remote-home /root 3090-1
```

只同步程序本体，不同步配置：

```bash
./fast-sync --bins --no-configs 3090-1
```

允许远端用 apt 安装基础依赖：

```bash
./fast-sync --all 3090-1
```

`--all` 会安装：

```text
rsync git curl ca-certificates python3
ripgrep fd-find file unzip xz-utils tar gzip
build-essential gcc g++ make
```

不想删除远端已有文件：

```bash
./fast-sync --no-delete 3090-1
```

## 边界和风险

不要直接复制 apt/dpkg 系统包数据库，例如：

```text
/usr/bin/git
/usr/bin/curl
/lib
/var/lib/dpkg
```

这些内容直接 rsync 可能破坏远端包管理状态。

当前本机的 `yazi.toml` 有 WSL 专用的 `wslview` 和 `explorer.exe` 打开方式。脚本同步后会在远端把它们改成服务器安全命令，本地配置不会被改。

这个工具更像是“你的服务器环境一键迁移器”。别人也能用，但最好先跑：

```bash
./fast-sync --dry-run HOST
```

确认会同步哪些路径。
