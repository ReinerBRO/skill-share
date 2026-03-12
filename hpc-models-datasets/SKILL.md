---
name: hpc-models-datasets
description: HPC 通用模型与数据集下载/分发指引，包括 ModelScope/HF Mirror 下载策略、跨服务器分发、完整性校验和离线运行配置。
---

# HPC 通用模型与数据集下载/分发指引

更新时间：2026-03-02

## 1. 适用范围

本指引用于 HPC 环境中的通用模型与数据集管理，覆盖：

- 下载（有外网权限的节点）
- 分发（HPC2 / HPC3 / HPC4 / HPC4-test）
- 完整性校验
- 离线运行配置

> 本文不绑定任何单一项目（如 ParamAgent）。

## 2. 目录约定（推荐）

建议统一使用以下缓存根目录：

- HPC2：`/hpc2hdd/home/jzhu997/cache`
- HPC3：`/data/user/jzhu997/cache`
- HPC4：`/data/user/jzhu997/cache`
- HPC4-test：`/data/user/user06/cache`

建议子目录结构：

- 模型：`<cache_root>/Models/<model_name>/`
- 数据集：`<cache_root>/data/<dataset_name>/`

在 HPC2 上，下载落地路径固定为：

- 数据集：`/hpc2hdd/home/jzhu997/cache/data/`
- 模型：`/hpc2hdd/home/jzhu997/cache/Models/`

## 3. 下载原则

### 3.1 下载节点选择

- 优先在能联网且磁盘空间充足的一台机器先下载（通常 HPC2）。
- 其他机器通过 `rsync` / `scp` 分发，避免重复外网下载。
- 若在 HPC2 下载，统一写入 `jzhu997/cache/data` 或 `jzhu997/cache/Models`（即 `/hpc2hdd/home/jzhu997/cache/...`）。

### 3.2 网络与镜像

下载前先取消代理（必须）：

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
```

下载优先级（必须遵循）：

1. 优先使用 ModelScope 下载（无论模型或数据集）。
2. 若 ModelScope 无该资源，再使用 Hugging Face 镜像（`hf-mirror`）。

当需要走 Hugging Face 镜像时，再设置：

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

### 3.3 通用下载模板

优先：ModelScope（模型/数据集）

```bash
# 模型（ModelScope）
modelscope download --model <org_or_user>/<model_repo> \
  --local_dir /hpc2hdd/home/jzhu997/cache/Models/<model_name>

# 数据集（ModelScope）
modelscope download --dataset <org_or_user>/<dataset_repo> \
  --local_dir /hpc2hdd/home/jzhu997/cache/data/<dataset_name>
```

兜底：HF Mirror（仅当 ModelScope 不提供时）

```bash
# 数据集（HF Mirror）
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
export HF_ENDPOINT=https://hf-mirror.com
huggingface-cli download --repo-type dataset <org_or_user>/<dataset_repo> \
  --local-dir /hpc2hdd/home/jzhu997/cache/data/<dataset_name>

# 模型（HF Mirror）
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
export HF_ENDPOINT=https://hf-mirror.com
huggingface-cli download <org_or_user>/<model_repo> \
  --local-dir /hpc2hdd/home/jzhu997/cache/Models/<model_name>
```

## 4. 分发到其他服务器

以下示例默认在"已下载源机器"执行。

### 4.1 分发数据集

```bash
# -> HPC3
rsync -avh --delete ~/cache/data/<dataset_name>/ \
  jzhu997@hpc3login.hpc.hkust-gz.edu.cn:/data/user/jzhu997/cache/data/<dataset_name>/

# -> HPC4
rsync -avh --delete ~/cache/data/<dataset_name>/ \
  jzhu997@hpc4login.hpc.hkust-gz.edu.cn:/data/user/jzhu997/cache/data/<dataset_name>/

# -> HPC4-test
rsync -avh --delete ~/cache/data/<dataset_name>/ \
  user06@hpc4-test.hpc.hkust-gz.edu.cn:/data/user/user06/cache/data/<dataset_name>/
```

### 4.2 分发模型

```bash
# -> HPC3
rsync -avh --delete ~/cache/Models/<model_name>/ \
  jzhu997@hpc3login.hpc.hkust-gz.edu.cn:/data/user/jzhu997/cache/Models/<model_name>/

# -> HPC4
rsync -avh --delete ~/cache/Models/<model_name>/ \
  jzhu997@hpc4login.hpc.hkust-gz.edu.cn:/data/user/jzhu997/cache/Models/<model_name>/

# -> HPC4-test
rsync -avh --delete ~/cache/Models/<model_name>/ \
  user06@hpc4-test.hpc.hkust-gz.edu.cn:/data/user/user06/cache/Models/<model_name>/
```

## 5. 完整性校验（通用）

### 5.1 数据集文件校验

```bash
# 按实际文件名检查（示例）
test -f ~/cache/data/<dataset_name>/test.jsonl && echo "test split ok"
test -f ~/cache/data/<dataset_name>/train.jsonl && echo "train split ok"
```

### 5.2 目录摘要校验

```bash
python3 - <<'PY'
from pathlib import Path
p = Path('~/cache/data/<dataset_name>').expanduser()
print('exists:', p.exists())
if p.exists():
    files = sorted([x.name for x in p.iterdir()])
    print('count:', len(files))
    print('sample:', files[:20])
PY
```

## 6. 离线运行建议

在离线机器的运行脚本（如 `sbatch`）中建议固定：

```bash
export HF_HOME=~/.cache/huggingface
export HF_HUB_OFFLINE=1
export HF_DATASETS_OFFLINE=1
export TRANSFORMERS_OFFLINE=1
```

必要时显式指定本地路径，避免代码触发在线拉取。

## 7. 常见问题

- 下载不完整：先检查磁盘空间、网络稳定性，再重试下载。
- 目标机器缺目录：先 `mkdir -p` 再 `rsync`。
- 数据缺 split（如只有 `test` 无 `train`）：先确认该数据集仓库是否本就不提供该 split；若需要训练集，需额外下载对应来源。
- 远端权限问题：确认目标路径所属用户与配额。

## 8. 推荐操作顺序

1. 在单台可联网机器下载到 `~/cache`。
2. 本地完成完整性校验。
3. 使用 `rsync --delete` 分发到其他机器。
4. 在每台目标机器做文件存在性校验。
5. 训练/评测脚本启用离线环境变量。
