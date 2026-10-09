# 通过 kmesh 连接 jump01（JumpServer）最佳实践

## 工作原理

```mermaid
flowchart LR
    Local["本地电脑"] -. "协助连接" .-> Server["kmesh 服务"]
    Server -. "协助连接" .-> Jump["jump01<br/>JumpServer"]
    Local ==>|"P2P 直连"| Jump
    Jump --> VNC["目标 VNC 实例"]
```

kmesh 协助本地电脑与 jump01 优先建立 P2P 直连，再通过 jump01 访问已授权的目标 VNC 实例。直连受限时，可通过kmesh 服务中继转发。

## 下载并安装 v0.4.2

[v0.4.2 GitHub Release](https://github.com/fnolk/kmesh/releases/tag/v0.4.2) 提供以下安装包。以下命令适用于 macOS Apple Silicon 和 Linux；Windows 用户使用 Windows ZIP 安装包并按本机环境安装。

| 平台 | 安装包 |
| --- | --- |
| macOS Apple Silicon | `kmesh-aarch64-apple-darwin.tar.gz` |
| Linux ARM64 | `kmesh-aarch64-unknown-linux-musl.tar.gz` |
| Linux x86_64 | `kmesh-x86_64-unknown-linux-musl.tar.gz` |
| Windows x86_64 | `kmesh-x86_64-pc-windows-msvc.zip` |

以下命令按本机系统和架构选择安装包，再下载、解压并安装：

```sh
(
set -eu
case "$(uname -s):$(uname -m)" in
  Darwin:arm64) asset="kmesh-aarch64-apple-darwin.tar.gz" ;;
  Linux:aarch64) asset="kmesh-aarch64-unknown-linux-musl.tar.gz" ;;
  Linux:x86_64) asset="kmesh-x86_64-unknown-linux-musl.tar.gz" ;;
  *) printf 'No v0.4.2 asset for this platform.\n' >&2; exit 1 ;;
esac
download_dir="$(mktemp -d)"
curl -fL "https://github.com/fnolk/kmesh/releases/download/v0.4.2/${asset}" \
  -o "${download_dir}/${asset}"
tar -xzf "${download_dir}/${asset}" -C "$download_dir"
kmesh_binary="$(find "$download_dir" -type f -name kmesh -print -quit)"
printf 'Installing %s\n' "$kmesh_binary"
mkdir -p "$HOME/.local/bin"
install -m 0755 "$kmesh_binary" "$HOME/.local/bin/kmesh"
"$HOME/.local/bin/kmesh" --version
)
```

确认版本输出为 `0.4.2`。后续命令可调用 `$HOME/.local/bin/kmesh`；如需直接运行 `kmesh`，将 `$HOME/.local/bin` 加入交互式 shell 的 `PATH`。SSH 配置中显式指定 `~/.local/bin/kmesh`。

## 配置 kmesh 客户端

将以下内容写入 `~/.kmesh/config.toml`，并将 token 占位符替换为通过组织安全渠道取得的个人 token：

```toml
server_addr = "portal.fics.top"
server_port = 9443

[auth]
method = "token"
token = "REPLACE_WITH_KMESH_TOKEN"
```

token 用于 kmesh 用户身份认证和 target 授权。只需将 token 占位符替换为分配给本人的值。

kmesh 默认读取 `~/.kmesh/config.toml`；更多字段以 [v0.4.2 配置说明](https://github.com/fnolk/kmesh/blob/v0.4.2/config.example.toml) 为准。

## 在 `jump01` 中配置 ProxyCommand

编辑现有 SSH 配置并合并 `Host jump01` 区块，保留其他主机配置：

```sh
"$HOME/.ssh/config"
```

本机 `Host jump01` 区块使用以下连接参数。kmesh target ID 固定为 `jump01`，SSH 登录使用区块中的 `HostName` 和 `User`。

```sshconfig
Host jump01
    HostName jump01
    User REPLACE_WITH_USER_NAME
    BatchMode yes
    ProxyCommand ~/.local/bin/kmesh proxy jump01
    ConnectTimeout 65
    ServerAliveInterval 20
    ServerAliveCountMax 2
    ControlMaster auto
    ControlPath ~/.ssh/kmesh-%C
    ControlPersist 5m
```

`Host jump01` 是本机 SSH 配置别名，`HostName jump01` 是 SSH 层连接使用的主机名；

`ProxyCommand` 中的 kmesh target ID 固定为 `jump01`， kmesh 认证会默认读取 `~/.kmesh/config.toml`；

`BatchMode yes` 适用于 OpenSSH 可直接使用的 SSH 私钥，例如已装入 `ssh-agent` 并解锁的密钥。若经 `jump01` 登录需要密码或 MFA 交互，可将其设为 `no`；

`ControlMaster auto` 与 `ControlPersist 5m` 会让 ControlMaster 在最后一个客户端退出后继续保留五分钟，供这段时间内的后续 SSH 命令复用。需要立即结束复用会话时运行 `ssh -O exit jump01`。新建连接时 kmesh 会重新检查会话授权。

## 检查并连接

先检查 kmesh 服务与身份，再确认账户可见的 target：

```sh
"$HOME/.local/bin/kmesh" status
"$HOME/.local/bin/kmesh" targets list
```

使用 target ID `jump01` 检查网络路径和目标 SSH 服务：

```sh
"$HOME/.local/bin/kmesh" doctor jump01
```

`doctor` 建立一次临时授权连接并读取 SSH identification，结果可能记录在服务端或 SSH 日志中。成功结果表示 SSH 服务有响应；OpenSSH 首次连接仍需核对主机密钥，并按 `jump01` 要求完成 SSH 账户认证。

从管理员或 `jump01` 管理界面通过独立可信渠道取得 SSH 主机密钥指纹。确认指纹后执行：

```sh
ssh -o BatchMode=no jump01
```

当 OpenSSH 显示首次连接的指纹时，将其与独立渠道提供的指纹逐字核对，再接受并按组织策略完成 SSH 登录。之后日常连接使用：

```sh
ssh jump01
```

保持 OpenSSH 默认的主机密钥验证。以 `ssh -vv jump01` 查看本地连接诊断；分享日志前先移除 token 值等凭据内容。

## 双层认证与维护

连接包含两层身份与授权：kmesh 凭据和 `ssh_connect` grant 决定账户能否通过 kmesh 打开指定 target；SSH 主机密钥、SSH 用户凭据及 `jump01` 策略决定登录的服务端身份、账户和 VNC 实例访问。平台管理员角色本身不授予 target 的 SSH 权限。

target ID、`jump01` 的 SSH 地址或端口、服务地址、凭据发生变化时，向管理员确认映射后再更新对应配置。修改 SSH 配置后可运行 `ssh -G jump01` 查看 OpenSSH 解析结果；修改后首次连接可先运行 `ssh -O exit jump01` 关闭旧的 ControlMaster，再执行日常连接。

## 参考

- [kmesh v0.4.2 Release 与安装包](https://github.com/fnolk/kmesh/releases/tag/v0.4.2)
- [v0.4.2 README：agent 与 OpenSSH 连接](https://github.com/fnolk/kmesh/blob/v0.4.2/README.md#connect-with-openssh)
- [v0.4.2 配置示例](https://github.com/fnolk/kmesh/blob/v0.4.2/config.example.toml)
- [v0.4.2 命令定义](https://github.com/fnolk/kmesh/blob/v0.4.2/src/client/cli.rs)
