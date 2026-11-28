# TrainingPod 用户指南

也可以把本指南克隆到本机的 `~/Project`，供你和 AI Agent 查阅。

```sh
mkdir -p ~/Project
git clone https://github.com/Urban-Visual-Intelligence-PKU/TrainingPodOnboarding.git ~/Project/TrainingPodOnboarding
```

## 申请 Pod 并登录

让你的 AI Agent 先读取 [TrainingPod 申请、登录和数据迁移 Skill](skills/trainingpod-data-migration/SKILL.md)。它会收集个人目录名、GitHub 用户名和出网方式，生成可直接复制给管理员的三行申请。请在 GitHub 添加本机 SSH 公钥。选择 AnyTLS 代理时，按 Skill 的说明通过私密渠道交付 profile。

同一用户可申请多个 Pod。Pod ID 为 `<个人目录名>-<index>`，例如 `huyuan-1`、`huyuan-2`。旧式实例计作索引 1。追加时沿用个人目录名，另发一句「请为这个已有个人目录追加一个 Pod」。

管理员回复一行登录命令，例如 `ssh -p 22226 root@192.168.10.2`。把该行交给 AI Agent，按 Skill 在本机 `~/.ssh/config` 添加自定义别名，在首次登录前从可信渠道单独核验主机密钥。优先用 Pod ID 做别名，只知道端口时可用 `huyuan-22226`。每个 Pod 的 SSH 端口和系统盘独立。

## 迁移数据

登录后，告诉 AI Agent 源目录和哪些内容可以共享，按同一个 Skill 迁移。追加 Pod 后，已存入个人持久目录的数据可直接使用。

| 路径 | 放什么 | Pod 中是否能查看并读写 |
| --- | --- | --- |
| `/fsx/<个人目录名>` | 个人代码、项目虚拟环境、私有数据、实验日志和 checkpoint | 同一用户的所有 Pod |
| `/fsx/shared/<姓名>` | 个人共享目录，用于发布希望在 Pod 之间共享的个人文件或产物，例如 `/fsx/shared/ruizhe` | 全部 Pod |
| `/fsx/shared/data/raw/<数据集>` | 已获准共享的原始下载数据，保留原始内容并记录来源、版本和许可 | 全部 Pod |
| `/fsx/shared/data/<项目名>/<数据集>/` | 已获准共享的项目处理产物，记录来源数据版本和处理方式 | 全部 Pod |
| `/fsx/shared/models` | Hugging Face 和 ModelScope 的 Hub 缓存，以及从 Hugging Face 以外的来源下载的公开 checkpoint | 全部 Pod |
| `/fsx/shared/data` | Hugging Face 和 ModelScope 的 Datasets 缓存 | 全部 Pod |

请遵守以下存储规则。

- 大型数据、checkpoint、训练输出和需要跨 Pod 重建保留的文件直接写到 `/fsx`。`/root`、`/tmp` 属于当前 Pod 的系统盘，与 `/fsx` 之间的 `mv` 仍会跨文件系统复制后删除源。宿主机的 `/home/<账号>` 需要在宿主机侧访问。
- 新 Pod 在个人目录和 `/fsx/shared` 之间 `mv` 时直接改名，几乎瞬间完成，保留原属主、权限和 ACL。发布后请从获授权的 Pod 核对接收方可读。其他私有目录名可见，内容由 ACL 隔离。旧 Pod 在这两个目录之间 `mv` 仍会复制后删除源。
- 跨挂载 `mv` 如果只出现 `preserving permissions ... Invalid argument`，文件已移动，提示的是 ACL 保留失败。核对目标存在、源已移走，并检查目标权限。也可用 `rsync -rltH` 复制，核对内容后再删除源。`cp -a` 遇到未映射的 ACL 而失败时，可用 `cp -a --no-preserve=mode,ownership` 跳过原权限、ACL 和属主的保留。
- SSH 私钥、令牌、`.env` 和未获准共享的数据留在个人目录。更新别人正在使用的共享数据集前，先协调版本和切换时间。
- 手动下载的公开 checkpoint 按模型归档到 `/fsx/shared/models/<模型名>/`。通过 Hugging Face 或 ModelScope SDK 下载时，由 SDK 管理缓存目录结构。
- 项目依赖用项目自己的 `.venv` 和锁文件管理。Pod 已预设共享下载缓存。

| 环境变量 | 缓存路径 |
| --- | --- |
| `HF_HUB_CACHE` | `/fsx/shared/models` |
| `HF_DATASETS_CACHE` | `/fsx/shared/data` |
| `PIP_CACHE_DIR` | `/fsx/shared/pip` |
| `UV_CACHE_DIR` | `/fsx/shared/uv` |
| `TORCH_HOME` | `/fsx/shared/torch` |
| `MODELSCOPE_CACHE` | `/fsx/shared/models` |
| `MS_DATASETS_CACHE` | `/fsx/shared/data` |
| `CONDA_PKGS_DIRS` | `/fsx/shared/conda/pkgs,/fsx/<姓名>/.cache/conda/pkgs` |

Conda 按需安装。已缓存的包可从管理员维护的共享目录读取，新下载的包写到个人缓存。共享下载缓存只放允许所有获授权使用者访问的内容。Triton、CUDA、TorchInductor、FlashInfer 和 vLLM 的编译及自动调优缓存保留在各自的 Pod 或个人目录。
