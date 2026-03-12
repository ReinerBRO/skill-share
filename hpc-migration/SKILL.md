---
name: hpc-migration
description: HPC2 → HPC3 迁移操作规范，包括数据迁移、环境迁移、容器镜像迁移的完整流程和最佳实践。
---

# HPC2 → HPC3 迁移操作规范

## 前提条件

本地 `~/.ssh/config` 已配置以下别名，可直接使用，无需密码：

| 别名 | 实际主机 | 用户 |
|------|----------|------|
| `hpc2` | hpc2login.hpc.hkust-gz.edu.cn | jzhu997 |
| `hpc3` | hpc3login.hpc.hkust-gz.edu.cn | jzhu997 |

HPC2 和 HPC3 之间也已配置 SSH 互信，可直接跨节点传输。

### HPC3 目录结构

| 路径 | 说明 |
|------|------|
| `/data/user/jzhu997/` | Home 目录（`$HOME`） |
| `/data/user/jzhu997/miniforge3/envs/` | miniforge3 默认 conda 环境路径，现有：`memgen`、`pm` |
| `/data/user/jzhu997/envs/` | 自定义 conda 环境路径，现有：`cluster`、`coconut`、`cyclecot`、`peft_pkg` |

---

## 一、数据迁移

> 所有命令在 **HPC3** 上执行，从 HPC3 拉取 HPC2 的数据。

### 1.1 确认网络连通性

```bash
# 登录 HPC3
ssh hpc3

# 在 HPC3 上确认能访问 HPC2
ping hpc2login.hpc.hkust-gz.edu.cn
```

### 1.2 方法一：rsync（推荐，适合大量文件）

```bash
# 在 HPC3 上执行
rsync -vrlptDSH jzhu997@hpc2login.hpc.hkust-gz.edu.cn:<source_path> <target_path>
```

参数说明：
- `-v` 显示详细进度
- `-r` 递归同步子目录
- `-l` 保持符号链接
- `-p` 保持文件权限
- `-t` 保持时间戳
- `-D` 保持设备文件和特殊文件
- `-S` 压缩传输（节省带宽，增加 CPU 负担）
- `-H` 保持硬链接

示例（迁移 home 目录下的项目）：
```bash
rsync -vrlptDSH jzhu997@hpc2login.hpc.hkust-gz.edu.cn:~/projects/ ~/projects/
```

### 1.3 方法二：scp（适合少量文件）

```bash
# 拷贝单个文件
scp -p jzhu997@hpc2login.hpc.hkust-gz.edu.cn:<source_file> <target_path>

# 拷贝整个目录
scp -rp jzhu997@hpc2login.hpc.hkust-gz.edu.cn:<source_path> <target_path>
```

---

## 二、环境迁移

### 2.1 Conda 环境迁移

#### 步骤一：在 HPC2 上打包环境

> HPC2 上 conda 安装包或下载环境时，使用清华源以加速：
> ```bash
> conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
> conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
> conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
> conda config --set show_channel_urls yes
> ```

```bash
# 登录 HPC2
ssh hpc2

# 查看所有 conda 环境
conda env list

# 激活要迁移的环境
conda activate <env_name>

# 安装打包工具（如未安装，使用清华源）
conda install -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/ conda-pack

# 打包环境
conda pack -n <env_name> -o <env_name>.tar.gz
```

#### 步骤二：在 HPC3 上拉取并解压

新环境统一放到 `/data/user/jzhu997/envs/` 目录下。

```bash
# 登录 HPC3
ssh hpc3

# 从 HPC2 拉取打包文件（在 HPC3 上执行）
scp jzhu997@hpc2login.hpc.hkust-gz.edu.cn:~/<env_name>.tar.gz /data/user/jzhu997/

# 创建目标目录并解压（统一放到 ~/envs/）
mkdir -p /data/user/jzhu997/envs/<env_name>
tar -xzf /data/user/jzhu997/<env_name>.tar.gz -C /data/user/jzhu997/envs/<env_name>
```

#### 步骤三：恢复并激活环境

```bash
# 修复环境路径（必须执行）
python3 /data/user/jzhu997/envs/<env_name>/bin/conda-unpack

# 激活环境
source /data/user/jzhu997/envs/<env_name>/bin/activate
```

### 2.2 环境迁移（通用流程）

#### 步骤一：在 HPC2 上创建并验证环境

1. 在 HPC2 上创建所需环境（conda/virtualenv 等）
2. 安装所有依赖包
3. 运行 smoke 测试验证环境可用性

#### 步骤二：打包环境

```bash
# 使用 conda-pack 打包（推荐）
conda pack -n <env_name> -o <env_name>.tar.gz

# 或使用 tar 直接打包虚拟环境目录
tar -czf <env_name>.tar.gz -C /path/to/env .
```

#### 步骤三：传输到 HPC3

```bash
# 在 HPC3 上执行，从 HPC2 拉取打包文件
scp jzhu997@hpc2login.hpc.hkust-gz.edu.cn:~/<env_name>.tar.gz /data/user/jzhu997/
```

#### 步骤四：在 HPC3 上解压并验证

```bash
# 创建目标目录并解压
mkdir -p /data/user/jzhu997/envs/<env_name>
tar -xzf /data/user/jzhu997/<env_name>.tar.gz -C /data/user/jzhu997/envs/<env_name>

# 修复路径（conda-pack 必须）
python3 /data/user/jzhu997/envs/<env_name>/bin/conda-unpack

# 激活并验证
source /data/user/jzhu997/envs/<env_name>/bin/activate
# 运行 smoke 测试确认环境正常
```

---

## 注意事项

- HPC3 有网络访问限制，建议在 HPC2 或本地配置好环境后再迁移
- rsync 的 `-S` 压缩选项会增加 CPU 负担，带宽充足时可去掉
- conda-unpack 步骤不可跳过，否则环境路径错误无法正常使用
- 迁移前务必在源机器上运行 smoke 测试，确保环境完整可用
