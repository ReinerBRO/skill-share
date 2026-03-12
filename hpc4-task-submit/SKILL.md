---
name: hpc4-task-submit
description: Submit and manage CPU/NPU tasks on HPC4 compute nodes with proper ownership and permissions.
---

# HPC4 Task Submit

## Purpose

Submit and manage CPU/NPU tasks on HPC4 compute nodes with proper ownership and permissions.

## Critical Rules

### 1. Ownership Safety (MANDATORY)

**All outputs in shared directories must be deletable by local user:**

```bash
# Dynamic owner detection
OWNER_REF_DIR="/data/user/user06"
OWNER_UID="$(stat -c '%u' "$OWNER_REF_DIR")"
OWNER_GID="$(stat -c '%g' "$OWNER_REF_DIR")"

# Helper functions
ensure_owner_dir() {
  install -d -m 755 -o "$OWNER_UID" -g "$OWNER_GID" "$1"
}

ensure_owner_path() {
  local target="$1"
  local rel="${target#"$PROJECT_ROOT"/}"
  local path="$PROJECT_ROOT"
  IFS='/' read -r -a parts <<< "$rel"
  for part in "${parts[@]}"; do
    [ -n "$part" ] || continue
    path="$path/$part"
    ensure_owner_dir "$path"
  done
}

# Privilege drop wrapper
if [ "$(id -u)" = "$OWNER_UID" ]; then
  run_as_owner() { "$@"; }
else
  command -v setpriv >/dev/null 2>&1 || {
    echo "[ERROR] current uid=$(id -u), owner uid=$OWNER_UID, but setpriv is unavailable"
    exit 1
  }
  run_as_owner() {
    setpriv --reuid="$OWNER_UID" --regid="$OWNER_GID" --clear-groups \
      env HOME="$OWNER_REF_DIR" USER="user06" LOGNAME="user06" \
      LD_LIBRARY_PATH="${LD_LIBRARY_PATH:-}" \
      PYTHONPATH="${PYTHONPATH:-}" \
      "$@"
  }
fi
```

### 2. No nohup in SSH heredoc (CRITICAL)

❌ **NEVER use `nohup ... &` inside SSH heredoc**
- Causes SSH connection hang
- Creates zombie processes
- Can freeze compute nodes

✅ **Correct approach:**
- Run commands directly in heredoc
- SSH auto-disconnects after completion
- Processes continue running

## Dev Node Resources

- **Available nodes**: miku, yui, mutsumi, eren
- **NPU per node**: 4 × Ascend 910 (2 chips each = 8 chips total)
- **Scheduling**: One task per node (exclusive)

### Check Node Status

```bash
DEV_NODE="miku"  # or yui, mutsumi, eren
ssh "$DEV_NODE" 'bash -s' <<'EOS'
export LD_LIBRARY_PATH="/usr/local/Ascend/driver/lib64/driver:/usr/local/Ascend/driver/lib64/common:${LD_LIBRARY_PATH}"
npu-smi info -l | awk -F: '
  /NPU ID/ {gsub(/ /,"",$2); n=$2}
  /Chip Count/ {gsub(/ /,"",$2); c=$2+0; for(i=0;i<c;i++) print n, i}
' | while read -r npu chip; do
  out="$(npu-smi info -t usages -i "$npu" -c "$chip" 2>/dev/null || true)"
  hbm="$(awk -F: "/HBM Usage Rate\\(%\\)/{gsub(/ /,\"\",$2);print $2;exit}" <<< "$out")"
  util="$(awk -F: "/NPU Utilization\\(%\\)/{gsub(/ /,\"\",$2);print $2;exit}" <<< "$out")"
  echo "npu${npu}_chip${chip} hbm=${hbm:-NA}% util=${util:-NA}%"
done
EOS
```

## Task Submission Templates

### CPU Task (Slurm)

```bash
#!/usr/bin/env bash
#SBATCH -J cpu_job
#SBATCH -p hpc
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 16
#SBATCH --mem=64G
#SBATCH -t 12:00:00
#SBATCH -o slurm/logs/%x_%j.out
#SBATCH -e slurm/logs/%x_%j.err

set -euo pipefail
cd <PROJECT_ROOT>
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source <MINICONDA_HOME>/etc/profile.d/conda.sh
conda activate <ENV_NAME>

python <CPU_ENTRY>.py
```

Submit: `sbatch slurm/job_cpu.sbatch`

### NPU Task (Dev Node Direct)

```bash
DEV_NODE="<miku|yui|mutsumi|eren>"
PROJECT_ROOT="<PROJECT_ROOT>"
ENV_NAME="<ENV_NAME>"
RUN_SCRIPT="<RUN_SCRIPT>"
RUN_ROOT="<RUN_ROOT>"
OWNER_REF_DIR="/data/user/user06"

ssh "$DEV_NODE" 'bash -s' <<EOS
set -euo pipefail
PROJECT_ROOT="$PROJECT_ROOT"
ENV_NAME="$ENV_NAME"
RUN_SCRIPT="$RUN_SCRIPT"
RUN_ROOT="$RUN_ROOT"
OWNER_REF_DIR="$OWNER_REF_DIR"

OWNER_UID="\$(stat -c '%u' "\$OWNER_REF_DIR")"
OWNER_GID="\$(stat -c '%g' "\$OWNER_REF_DIR")"

ensure_owner_dir() {
  install -d -m 755 -o "\$OWNER_UID" -g "\$OWNER_GID" "\$1"
}

ensure_owner_path() {
  local target="\$1"
  local rel="\${target#"\$PROJECT_ROOT"/}"
  local path="\$PROJECT_ROOT"
  IFS='/' read -r -a parts <<< "\$rel"
  for part in "\${parts[@]}"; do
    [ -n "\$part" ] || continue
    path="\$path/\$part"
    ensure_owner_dir "\$path"
  done
}

if [ "\$(id -u)" = "\$OWNER_UID" ]; then
  run_as_owner() { "\$@"; }
else
  command -v setpriv >/dev/null 2>&1 || {
    echo "[ERROR] current uid=\$(id -u), owner uid=\$OWNER_UID, but setpriv is unavailable"
    exit 1
  }
  run_as_owner() {
    setpriv --reuid="\$OWNER_UID" --regid="\$OWNER_GID" --clear-groups \
      env HOME="\$OWNER_REF_DIR" USER="user06" LOGNAME="user06" \
      LD_LIBRARY_PATH="\${LD_LIBRARY_PATH:-}" \
      PYTHONPATH="\${PYTHONPATH:-}" \
      "\$@"
  }
fi

ensure_owner_path "\$PROJECT_ROOT/results"
ensure_owner_path "\$RUN_ROOT"

cd "\$PROJECT_ROOT"
export LD_LIBRARY_PATH="\${LD_LIBRARY_PATH:-}"
export PYTHONPATH="\${PYTHONPATH:-}"
source /usr/local/Ascend/ascend-toolkit/set_env.sh || true
source <MINICONDA_HOME>/etc/profile.d/conda.sh

run_as_owner bash -lc '
  set -euo pipefail
  source <MINICONDA_HOME>/etc/profile.d/conda.sh
  conda activate '"\$ENV_NAME"'
  unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY all_proxy ALL_PROXY no_proxy NO_PROXY
  export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3
  bash '"\$RUN_SCRIPT"' > '"\$RUN_ROOT"'/launcher_\$(date +%F_%H%M%S).log 2>&1
'

stat -c "%u:%g %n" "\$RUN_ROOT"
find "\$RUN_ROOT" -maxdepth 2 -printf "%u:%g %p\n" | sed -n "1,20p"
EOS
```

## NPU Environment Variables

```bash
# Visible devices (chip IDs)
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3

# Single device mode
export ASCEND_DEVICE_ID=0

# Multi-card communication timeout
export HCCL_CONNECT_TIMEOUT=1800

# Memory fragmentation control
export PYTORCH_NPU_ALLOC_CONF=max_split_size_mb:256
```

## Monitoring

### Check NPU Usage
```bash
npu-smi info -l
npu-smi info -t usages -i <npu_id> -c <chip_id>
```

### Continuous Monitoring
```bash
while true; do
  date '+%F %T'
  npu-smi info -t usages -i <npu_id> -c <chip_id>
  sleep 10
done | tee logs/npu_probe_$(date +%F_%H%M%S).log
```

### Slurm Jobs
```bash
squeue -u $USER
scontrol show job <jobid>
scancel <jobid>
```

## Common Issues

### Issue: Permission denied on output directory
**Solution**: Ensure `ensure_owner_path` is called before task starts

### Issue: SSH hangs after task submission
**Solution**: Remove `nohup` and `&` from heredoc

### Issue: libdrvdsmi_host.so not found
**Solution**: Set `LD_LIBRARY_PATH` before npu-smi:
```bash
export LD_LIBRARY_PATH=/usr/local/Ascend/driver/lib64/driver:/usr/local/Ascend/driver/lib64/common:${LD_LIBRARY_PATH}
```

### Issue: torch_npu auto-load fails on eren
**Solution**: Add `TORCH_DEVICE_BACKEND_AUTOLOAD=0` to environment

## Best Practices

1. **Always check node status** before submitting
2. **Verify ownership** after task starts
3. **Use smoke tests** (1-5 min) before full runs
4. **Monitor HBM usage** during execution
5. **Clean up** failed runs to free resources

## Reference

Full documentation: `docs/HPC4_TEST_环境与任务提交通用指引.md`
