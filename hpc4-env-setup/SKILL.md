---
name: hpc4-env-setup
description: Configure and manage Conda environments on HPC4 login nodes for NPU/CPU tasks.
---

# HPC4 Environment Setup

## Purpose

Configure and manage Conda environments on HPC4 login nodes for NPU/CPU tasks. This skill handles environment creation, dependency installation, and validation.

## Critical Rules

1. **Environment location**: All environments are created on login nodes
2. **Compute nodes**: Use login node environments via shared paths
3. **Network**: Disable proxies before any downloads
4. **Sources**: Use Tsinghua mirrors for Conda packages
5. **NPU dependencies**: Install torch_npu matching CANN version

## Environment Variables

```bash
# Required paths (customize per project)
MINICONDA_HOME="/data/user/user06/miniconda3"
ENV_NAME="<your_env_name>"
PROJECT_ROOT="<your_project_root>"
```

## Standard Workflow

### 1. Initial Setup (One-time)

```bash
# Disable proxies
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
unset all_proxy ALL_PROXY no_proxy NO_PROXY

# Configure Conda mirrors
source ${MINICONDA_HOME}/etc/profile.d/conda.sh
conda config --remove-key channels || true
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge
conda config --set show_channel_urls yes
```

### 2. Create Environment

```bash
# Load Ascend toolkit
source /usr/local/Ascend/ascend-toolkit/set_env.sh

# Create environment
source ${MINICONDA_HOME}/etc/profile.d/conda.sh
conda create -n ${ENV_NAME} python=3.10 -y
conda activate ${ENV_NAME}

# Upgrade pip
pip install -U pip setuptools wheel
```

### 3. Install NPU Dependencies

```bash
conda activate ${ENV_NAME}

# Install PyTorch (CPU version first)
pip install torch==2.8.0 --index-url https://download.pytorch.org/whl/cpu

# Install torch_npu (must match CANN version)
pip install torch_npu==2.8.0

# Install project dependencies (filter CUDA-only packages)
grep -vE '^(nvidia-|triton|bitsandbytes)' requirements.txt > requirements_npu.txt
pip install -r requirements_npu.txt
```

### 4. Validation

**On login node** (NPU usually not available):
```bash
python - <<'PY'
import torch
print("torch:", torch.__version__)
print("npu available on login:", hasattr(torch, "npu") and torch.npu.is_available())
PY
```

**On compute node** (via SSH):
```bash
DEV_NODE="miku"  # or yui, mutsumi, eren
ssh "$DEV_NODE" 'bash -s' <<'EOS'
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /data/user/user06/miniconda3/etc/profile.d/conda.sh
conda activate <ENV_NAME>

python - <<'PY'
import torch, torch_npu
print("torch:", torch.__version__)
print("torch_npu:", torch_npu.__version__)
print("npu available:", torch.npu.is_available())
print("npu count:", torch.npu.device_count())
PY
EOS
```

## Common Issues

### Issue: SSL errors during download
**Solution**: Ensure proxies are disabled

### Issue: torch_npu version mismatch
**Solution**: Check CANN version with `cat /usr/local/Ascend/ascend-toolkit/latest/version.cfg`

### Issue: Import errors on compute node
**Solution**: Verify environment path is accessible from compute node

## Best Practices

1. **Prefer Conda over pip** for packages available in Conda repos
2. **Test on compute node** before running large jobs
3. **Document versions** in project README
4. **Use requirements_npu.txt** to track NPU-specific dependencies
5. **Add to ~/.bashrc** for persistent environment setup:
   ```bash
   [ -f /usr/local/Ascend/ascend-toolkit/set_env.sh ] && source /usr/local/Ascend/ascend-toolkit/set_env.sh
   [ -f ${MINICONDA_HOME}/etc/profile.d/conda.sh ] && source ${MINICONDA_HOME}/etc/profile.d/conda.sh
   ```

## Reference

Full documentation: `docs/HPC4_TEST_环境与任务提交通用指引.md`
