# ZJC Shared Servers Via Dev

## Hard Remote Boundary

On `dev` and every compute node, all remote filesystem operations are restricted to:

```text
/gemini/space/private/zjc
```

Do not create, read project files from, edit, delete, move, copy, sync, extract archives into, install project files into, or use scratch files outside this directory. This prohibition includes `/root`, `/home`, `/data`, `/tmp`, system directories, and any other user's tree.

Before any remote command that touches files, first `cd /gemini/space/private/zjc` or use absolute paths under `/gemini/space/private/zjc`. If a tool needs cache, temp, output, checkpoint, log, or environment paths, point them to a subdirectory under `/gemini/space/private/zjc`. If that cannot be done safely, stop instead of running the command.

Non-file node checks such as `nvidia-smi`, `tmux ls`, and `ps` are allowed when needed to inspect node-local state, but any path touched by those commands or follow-up commands must still remain under `/gemini/space/private/zjc`.

## Core Model

The local Mac can SSH to `dev`. For compute work, connect in two hops:

```bash
ssh dev
ssh -o StrictHostKeyChecking=no -p <port> root@<ip>
```

Always use `-o StrictHostKeyChecking=no` when connecting from `dev` to a compute node. These are shared container environments and their host keys are not pre-registered on `dev`.

Do not assume node aliases such as `mutsumi`, `mahiro`, `madoka`, `eren`, `yui`, or `azusa` exist on `dev`. Those aliases are local-machine conveniences only. Once on `dev`, use the real endpoint table below.

## Public Internet Access

`dev` can reach the public internet through an HTTP proxy. Set these environment variables when tools like `pip`, `curl`, `wget`, or `huggingface_hub` need external access:

```bash
export https_proxy="http://172.16.113.28:3128"
export http_proxy="http://172.16.113.28:3128"
```

`dev` already runs **mihomo** as a proxy client, so the above environment variables are sufficient — no additional daemon setup is needed.

## Endpoint Map

| local alias | real host | port | role |
|---|---:|---:|---|
| `dev` | `10.127.16.73` | `40478` | sync gateway and management server |
| `mutsumi` | `10.127.16.20` | `40938` | shared-storage compute node, 8 GPUs |
| `mahiro` | `10.127.16.62` | `43936` | shared-storage compute node, 8 H100 80GB GPUs; external TCP port `43936` maps to internal port `5600` |
| `madoka` | `10.127.16.63` | `43939` | shared-storage compute node, 8 GPUs |
| `eren` | `10.127.16.3` | `43937` | shared-storage compute node, 8 H100 80GB GPUs |
| `yui` | `10.127.16.4` | `43938` | shared-storage compute node, 8 H100 80GB GPUs; external TCP port `43938` maps to internal port `5600` |
| `azusa` | `10.127.16.104` | `43933` | shared-storage compute node, 8 H100 80GB GPUs; external TCP port `43933` maps to internal port `5600` |
| `miku` | `10.127.16.4` | `43939` | Tianyi nested Linux server |

If the Tianyi UI shows both an internal container port and an external TCP port, use the external mapped port when connecting from `dev`. For `azusa`, that means:

```bash
ssh -o StrictHostKeyChecking=no -p 43933 root@10.127.16.104
```

Examples from `dev` (always skip host key check):

```bash
ssh -o StrictHostKeyChecking=no -p 40938 root@10.127.16.20   # mutsumi
ssh -o StrictHostKeyChecking=no -p 43936 root@10.127.16.62   # mahiro
ssh -o StrictHostKeyChecking=no -p 43939 root@10.127.16.63   # madoka
ssh -o StrictHostKeyChecking=no -p 43937 root@10.127.16.3    # eren
ssh -o StrictHostKeyChecking=no -p 43938 root@10.127.16.4    # yui
ssh -o StrictHostKeyChecking=no -p 43933 root@10.127.16.104  # azusa
```

## Storage Boundary

The shared servers use the same storage. Files under `/gemini/space/private/zjc` on `dev` should be visible from `mutsumi`, `mahiro`, `madoka`, `eren`, `yui`, and `azusa`.

Remote filesystem operations are only allowed under:

```text
/gemini/space/private/zjc
```

Do not create, read project files from, edit, delete, move, copy, or sync files elsewhere on the servers. If a command would touch `/root`, `/home`, `/data`, `/tmp`, system directories, or another user's tree, stop and adjust it to use `/gemini/space/private/zjc`.

Use `/gemini/space/private/zjc` as the default remote project root for this repository unless the user gives another path under `/gemini/space/private/zjc`.

## Sync Workflow

Code sync must go through `dev`.

Default direction:

```text
Mac -> dev:/gemini/space/private/zjc
```

Because storage is shared, do not separately copy the same code to `mutsumi`, `mahiro`, `madoka`, `eren`, `yui`, or `azusa` unless there is clear evidence the shared mount is unavailable.

Use Mac-compatible tools such as `rsync` or `scp`. Respect repository ignore files such as `.tianyi-syncignore`; avoid syncing large or generated directories like `runs/`, `data/`, caches, logs, and checkpoints unless the user explicitly asks.

## Reading Results

Read shared results directly from `dev`. For files under `/gemini/space/private/zjc`, do not first SSH to `dev` and then SSH again to a compute node just to read metrics, summaries, logs, configs, or result JSON/CSV files.

Use direct `dev` reads for paths such as:

```bash
ssh dev "cd /gemini/space/private/zjc && cat results/summary/example.md"
ssh dev "cd /gemini/space/private/zjc && tail -n 20 logs/train/example/metrics.jsonl"
ssh dev "cd /gemini/space/private/zjc && python3 - <<'PY'
from pathlib import Path
print(Path('results/train/example/summary.json').read_text())
PY"
```

Only hop from `dev` to a compute node when checking node-local state such as `tmux` sessions, GPU utilization, running processes, node-specific ports, or when launching/stopping work on that node.

## Remote Execution

Run project commands from the shared project root:

```bash
cd /gemini/space/private/zjc
```

From the Mac:

```bash
ssh dev
ssh -o StrictHostKeyChecking=no -p 40938 root@10.127.16.20
cd /gemini/space/private/zjc
```

Launch compute work on the requested target node. Use all eight cards only when the user asks for a full-machine run or the selected script/config expects it. Otherwise, preserve the script's existing device settings.

Prefer `tmux` for long-running jobs. Name sessions clearly with the experiment or config stem and target server. Do not use fragile backgrounding patterns when a project launcher already manages tmux or logging.

Before destructive cleanup under `/gemini/space/private/zjc`, inspect the exact resolved path and confirm it belongs to the current project or the user explicitly requested that target.

## Practical Checks

Connectivity from the Mac:

```bash
ssh dev "pwd; ls -ld /gemini/space/private/zjc"
```

Compute-node connectivity from `dev`:

```bash
ssh dev "ssh -o StrictHostKeyChecking=no -p 40938 root@10.127.16.20 'pwd; ls -ld /gemini/space/private/zjc'"
ssh dev "ssh -o StrictHostKeyChecking=no -p 43936 root@10.127.16.62 'pwd; ls -ld /gemini/space/private/zjc'"
ssh dev "ssh -o StrictHostKeyChecking=no -p 43939 root@10.127.16.63 'pwd; ls -ld /gemini/space/private/zjc'"
ssh dev "ssh -o StrictHostKeyChecking=no -p 43933 root@10.127.16.104 'pwd; ls -ld /gemini/space/private/zjc'"
ssh dev "ssh -o StrictHostKeyChecking=no -p 43938 root@10.127.16.4 'pwd; ls -ld /gemini/space/private/zjc'"
```

Shared-storage sanity:

```bash
ssh dev "mkdir -p /gemini/space/private/zjc/.codex_probe && date > /gemini/space/private/zjc/.codex_probe/dev_seen.txt"
ssh dev "ssh -o StrictHostKeyChecking=no -p 40938 root@10.127.16.20 'cat /gemini/space/private/zjc/.codex_probe/dev_seen.txt'"
```

Remove probes only under `/gemini/space/private/zjc`.
