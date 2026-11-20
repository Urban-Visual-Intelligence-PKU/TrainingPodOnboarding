# TrainingPod Onboarding

训练代码、数据和实验产物需要跨容器重建保留时，放在 `/fsx`。通过 `ssh TrainingPod` 进入容器后，默认工作目录是容器内的 `/root`。开始实验前，先切换到自己的持久化项目目录。

## /fsx 怎么用

| 路径 | 用途 | 谁可以访问 |
| --- | --- | --- |
| `/fsx/<姓名>` | 个人代码、虚拟环境、私有数据、实验日志和 checkpoint | 管理员创建目录并授权。容器挂载所属使用者的个人目录，宿主机按文件权限和 ACL 访问 |
| `/fsx/shared` | 所有获授权 TrainingPod 共用的存储 | 获授权的容器可读写公共目录，宿主机账号按 ACL 和组权限访问。子目录和文件可以有自己的权限 |
| `/fsx/shared/data/<姓名>/<数据集>` | 经确认可供其他使用者访问的数据集 | 由管理员为发布目录配置访问权限。发布前确认数据许可和共享范围 |
| `/fsx/shared/hub` | Hugging Face Hub 的模型缓存 | 供已授权的容器共享，由 Hugging Face 工具管理目录结构 |

TrainingPod 启动时预设以下缓存路径。

| 环境变量 | 路径 |
| --- | --- |
| `HF_HUB_CACHE` | `/fsx/shared/hub` |
| `HF_DATASETS_CACHE` | `/fsx/shared/huggingface/datasets` |
| `PIP_CACHE_DIR` | `/fsx/shared/pip` |
| `UV_CACHE_DIR` | `/fsx/shared/uv` |
| `TORCH_HOME` | `/fsx/shared/torch` |

### 存放规则

- 私有项目、SSH 私钥、令牌、`.env` 和未获准共享的数据留在个人目录，并按需要限制权限。一个目录里混有私有文件时，先整理出单独的数据发布目录。
- 将共享数据放在 `/fsx/shared/data/<姓名>/<数据集>`，并注明来源、版本和使用许可。更新已有数据集时，先与使用者协调。
- 共享目录的写权限通过宿主机 ACL 和组权限管理。管理员核对目标访问范围，避免用 `chmod -R 777` 授权。共同读写的目录可能被其他获授权的使用者修改，实验需要复现时应记录数据版本或校验和。
- 项目的 `.venv` 保存在个人项目中。uv 依赖缓存共用，项目仍通过 `pyproject.toml` 和 `uv.lock` 管理依赖。
- `/fsx` 的同一路径指向同一份持久数据。宿主机的 `/home/<账号>` 由宿主机侧访问，迁移该目录中的文件时，在宿主机上操作。

## 把个人数据目录迁到 shared

以下以把 `/fsx/alice/data/example` 发布到 `/fsx/shared/data/alice/example` 为例。若源仍在宿主机 home，将示例中的源改为实际路径，例如 `/home/alice/data/example`。只迁移已经确认可以共享的子目录。

1. 停止向源目录写入的实验或服务。检查隐藏文件、软链接和数据许可。软链接如果指向个人目录或宿主机路径，应在发布前处理，否则其他容器可能无法读取。
2. 请管理员在宿主机确认 `findmnt -M /fsx` 指向持久存储，创建目标父目录 `/fsx/shared/data/alice`，并为迁移操作者和预期使用者设置合适的访问权限及默认 ACL。
3. 在能访问源和目标的 Linux shell 中执行下面的整目录移动。把示例路径替换成实际路径，先确认目标名称可用。命令只在源和目标父目录位于同一文件系统时执行。

   ```bash
   src=/fsx/alice/data/example
   dst=/fsx/shared/data/alice/example
   parent=$(dirname -- "$dst")

   if [ -d "$src" ] && [ ! -L "$src" ] &&
      [ -d "$parent" ] &&
      [ ! -e "$dst" ] && [ ! -L "$dst" ] &&
      [ "$(stat -c %d -- "$src")" = "$(stat -c %d -- "$parent")" ]; then
     mv --no-clobber --no-target-directory -- "$src" "$dst" &&
       test ! -e "$src" && test -d "$dst" &&
       printf '已移动到 %s\n' "$dst"
   else
     printf '请检查源目录、目标父目录、目标是否存在及文件系统。\n' >&2
   fi
   ```

   如果设备编号不同，交由管理员分阶段迁移，在目标核对文件与权限后再清理源。整目录移动会保留原有文件权限和 ACL，因此移动完成后仍需按共享范围调整已有目录和文件的权限。默认 ACL 适用于之后新建的内容。

4. 从发布者的 TrainingPod 和一个预期有权限的其他 TrainingPod 分别读取代表性文件。若约定允许共同写入，也验证该身份能在约定目录创建并删除测试文件。权限通过后，把训练配置和脚本里的数据路径更新为新路径，再启动一轮小规模实验确认能读取数据。
