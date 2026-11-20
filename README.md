# TrainingPod 用户指南

## 第一步：联系管理员创建个人 Pod

把以下资料发给管理员。

- 你的姓名或账号，用于创建个人持久目录 `/fsx/<姓名>`。
- 你的 OpenSSH 公钥（通常是本机的 `~/.ssh/id_ed25519.pub`）。私钥留在自己电脑上。
- 你的 SSH 登录来源 IP 或网段，用于开通入口权限。
- 出网方式。选择**直连**时，Pod 经宿主机直接出网。选择**代理**时，请通过私密渠道向管理员提供你自己的 VPN/代理节点连接 URL 和端口。当前配置支持 SOCKS5 和 AnyTLS。代理凭据只交给管理员。

管理员会交付登录命令、SSH 配置和用于核对服务器身份的主机公钥。宿主机地址和登录用户是固定的，SSH 别名与入口端口以管理员给你的配置为准。下面的端口仅作示例。

```sshconfig
Host tp-alice
    HostName 192.168.10.2
    User root
    Port 22222
```

将配置添加到你本机的 `~/.ssh/config`，按管理员交付的主机公钥核验服务器身份，然后运行 `ssh tp-alice`。这里的 `root` 是 Pod 内的账号。SSH 登录使用提交的公钥，管理员不会发放 SSH 登录密码。如果本机 SSH 私钥设置过口令，客户端提示时输入你自己设置的口令。

## 第二步：迁移数据

**先让你的 AI Agent 读取并遵循 [TrainingPod 数据迁移 Skill](skills/trainingpod-data-migration/SKILL.md)，再告诉它源目录、你的个人目录名，以及哪些内容可以共享。**

| 路径 | 放什么 |
| --- | --- |
| `/fsx/<姓名>` | 个人代码、项目虚拟环境、私有数据、实验日志和 checkpoint |
| `/fsx/shared/data/raw/<数据集>` | 已获准共享的原始下载数据，保留原始内容并记录来源、版本和许可 |
| `/fsx/shared/data/<项目名>/<数据集>/` | 已获准共享的项目处理产物，记录来源数据版本和处理方式 |
| `/fsx/shared/hub` | Hugging Face Hub 模型缓存，目录结构由 Hugging Face 工具管理 |

请遵守以下存储规则。

- 需要跨 Pod 重建保留的文件放在 `/fsx`。登录后的 `/root` 属于当前 Pod 的系统盘。宿主机的 `/home/<账号>` 需要在宿主机侧访问。
- 容器内可访问自己的 `/fsx/<姓名>` 和公共的 `/fsx/shared`。宿主机和容器访问的是同一份数据，实际访问权限由宿主机的 ACL 和组权限决定。请让管理员为发布目录配置预期使用者的权限。
- SSH 私钥、令牌、`.env` 和未获准共享的数据留在个人目录。更新别人正在使用的共享数据集前，先协调版本和切换时间。
- 项目依赖用项目自己的 `.venv` 和锁文件管理。Pod 已预设共享下载缓存。

| 环境变量 | 缓存路径 |
| --- | --- |
| `HF_HUB_CACHE` | `/fsx/shared/hub` |
| `HF_DATASETS_CACHE` | `/fsx/shared/huggingface/datasets` |
| `PIP_CACHE_DIR` | `/fsx/shared/pip` |
| `UV_CACHE_DIR` | `/fsx/shared/uv` |
| `TORCH_HOME` | `/fsx/shared/torch` |
