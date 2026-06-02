# fast-config

把当前 WSL Ubuntu 里的 `nvim`、`yazi`、`zellij` 配置快速同步到新租的 Ubuntu 服务器。

## 常用命令

你的场景优先使用这一条：

```bash
./fast-sync 3090-1
```

这条默认会做完整离线迁移：终端环境加 Codex/Claude。主机名不能省，避免误同步到旧服务器。

跑完后登录服务器验证：

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

以后只是改了配置，继续跑这一条就可以；它会重新覆盖远端的配置、程序和插件缓存。

```bash
./fast-sync 3090-1
```

如果你临时想同步到另一台服务器，不改默认值也可以直接指定：

```bash
./fast-sync 5090-1
```

这会临时等价于 `./fast-sync --offline --ai 5090-1`。

如果只想补同步 Codex/Claude，不同步终端环境：

```bash
./fast-sync --ai 3090-1
```

如果只想同步终端环境，不同步 Codex/Claude：

```bash
./fast-sync --offline 3090-1
```

`--ai` 会同步 Codex/Claude 的本地安装和配置，包括：

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

Codex 如果是 nvm/npm 全局安装，脚本会从 `codex` 入口一路向上查找最近的 `bin/node`，然后把这个 Node 环境一起同步到远端。这样远端不需要自己安装 Node 或 npm 包。

如果前面终端环境已经同步成功，只是 AI 部分失败或后来才想补上，直接跑：

```bash
./fast-sync --ai 3090-1
```

这部分体积会明显更大。按当前本机内容估算约 `1.6G`，而且包含登录状态、token、历史记录等敏感信息，只建议同步到你完全信任的自用 root 服务器。

普通配置同步会镜像这些目录到远端 `/root`：

```text
~/.config/nvim    -> 3090-1:/root/.config/nvim
~/.config/yazi    -> 3090-1:/root/.config/yazi
~/.config/zellij  -> 3090-1:/root/.config/zellij
```

如果远端是新机器，而且你希望服务器不要自己下载 `nvim/yazi/zellij` 工具本体和已有插件，推荐直接跑离线迁移：

```bash
./fast-sync --offline 3090-1
```

`--offline` 不会在远端执行 `apt-get`，也不要求远端提前安装 `rsync`。它通过 `ssh + tar` 把本地内容同步过去，远端只需要有 Ubuntu 默认常见的 `sh`、`tar`、`uname`、`mktemp`。

它会同步：

```text
~/.config/nvim
~/.config/yazi
~/.config/zellij

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

这些目录里包含 nvim 的 lazy 插件、treesitter parser、yazi 插件缓存、zellij 插件缓存等。也就是说，你本地已经下载好的插件会被一起搬过去。同步完成后，脚本会在远端检查：

```text
nvim --version
yazi --version
zellij --version
```

如果你愿意让远端用 apt 安装基础依赖，也可以跑完整初始化：

```bash
./fast-sync --all 3090-1
```

`--all` 会做三件事：

```text
1. 用 apt 安装基础依赖：rsync、git、ripgrep、fd、编译工具等
2. 同步 nvim/yazi/zellij 配置
3. 同步本机的 nvim/yazi/zellij 程序本体到远端
```

之后本地配置更新了，再跑：

```bash
./fast-sync 3090-1
```

远端配置会跟着本地更新。

也可以监听本地配置变化，自动同步：

```bash
./fast-sync --watch 3090-1
```

`--watch` 需要本地 WSL 装有 `inotifywait`：

```bash
sudo apt install inotify-tools
```

如果使用 `./fast-sync --watch --all 3090-1`，脚本只会在启动时初始化和同步程序本体，后续文件变化只会重新同步配置。

## 目标路径

默认远端 home 是 `/root`。也可以显式指定：

```bash
./fast-sync 3090-1:/root
./fast-sync --remote-home /root 3090-1
```

## 预览变更

```bash
./fast-sync --dry-run 3090-1
```

## 删除策略

脚本默认使用 `rsync --delete`，也就是远端目标目录会严格跟随本地。比如本地删掉了 `~/.config/nvim/a.lua`，下次同步时远端 `/root/.config/nvim/a.lua` 也会删除。

如果只想覆盖和新增，不想删除远端已有文件：

```bash
./fast-sync --no-delete 3090-1
```

## 程序同步

只同步程序本体，不同步配置：

```bash
./fast-sync --bins --no-configs 3090-1
```

程序会放到远端：

```text
/opt/fast-config/nvim
/opt/fast-config/yazi
/usr/local/bin/zellij
```

并创建这些入口：

```text
/usr/local/bin/nvim
/usr/local/bin/yazi
/usr/local/bin/ya
/usr/local/bin/zellij
```

同时会在远端 `/root/.bashrc` 写入一个 `yz()` shell 函数。这个 `yz` 才是“退出 yazi 后让当前 shell 跳到 yazi 最后目录”的入口：

```bash
source ~/.bashrc
yz
```

`ya` 是 yazi 自带 helper 程序，不能替代这个目录跳转功能。

## SSH 别名

脚本不会自己解析服务器名字，也不会要求名字必须是 `3090-1`。你传什么 host，它就原样交给 `ssh`、`rsync` 或 `scp/tar` 流程，所以完全遵循 `~/.ssh/config`。

比如你有：

```sshconfig
Host 5090-1
  HostName 1.2.3.4
  User root
  IdentityFile ~/.ssh/id_ed25519
```

就直接跑：

```bash
./fast-sync 5090-1
```

## 注意

当前本机的 `yazi.toml` 里有 WSL 专用的 `wslview` 和 `explorer.exe` 打开方式。脚本同步后会在远端把它们改成服务器安全命令，本地配置不会被改。

`--offline` 可以把你的终端工作环境几乎完整搬过去，但不建议也不会直接复制 apt/dpkg 的系统包数据库。像 `/usr/bin/git`、`/usr/bin/curl`、`/lib`、`/var/lib/dpkg` 这类系统内容直接 rsync 可能破坏远端包管理状态。脚本会自动同步本地能安全携带的静态辅助工具，例如 `rg`、`fzf`。如果某个插件运行时硬性依赖远端系统包，例如 `git` 或编译工具，那么那部分仍然需要远端已有这些系统包，或者改用 `--all` 让 apt 安装。

`--ai` 会复制认证和会话相关文件。服务器退租前建议手动删除远端的 `/root/.codex`、`/root/.claude`、`/root/.claude.json`、`/root/.local/share/claude`。
