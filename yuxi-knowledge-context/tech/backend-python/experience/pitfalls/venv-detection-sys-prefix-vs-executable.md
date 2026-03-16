---
type: pitfall
keywords: [venv, sys.executable, sys.prefix, os.execv, 无限循环, macOS, Python 3.13, 虚拟环境检测]
related_files:
  - .claude/skills/vector-db-syncer/scripts/sync_vector_db.py
severity: high
date: 2026-03-16
---

# Python venv 检测：用 sys.executable 比较路径导致无限循环

## 问题描述

在使用 `os.execv` 切换到 venv Python 后，脚本无法正确检测自己已运行在 venv 内，导致反复调用 `os.execv` 或子进程形成**无限循环**。

### 触发场景

脚本需要"先切换到 venv 再继续执行"时，通常会写：

```python
def _is_running_in_venv() -> bool:
    venv_py = project_root / '.venv-agentkit' / 'bin' / 'python'
    # ❌ 错误：在 macOS + Python 3.13 上不可靠
    return Path(sys.executable).resolve() == venv_py.resolve()
```

### 根本原因

在 macOS + Python 3.13 venv 中，`sys.executable` 被 OS 设定为**带版本号的规范路径**，而检测用的 `venv_py` 指向**无版本号的 symlink**：

```
sys.executable  →  .venv-agentkit/bin/python3.13   # 带版本后缀
venv_py         →  .venv-agentkit/bin/python        # 无版本后缀（symlink）
```

两者调用 `.resolve()` 经过不同的 symlink 链，在 Python 3.13 venv 将解释器**复制**而非 symlink 的情况下，路径不相等，`_is_running_in_venv()` 始终返回 `False`。

### 循环链路

```
① 系统 Python 运行脚本
   └─ _is_running_in_venv() = False
   └─ venv 存在且依赖完整 → os.execv(venv_python, ...)
                                    ↓
② venv Python 运行脚本（os.execv 替换进程）
   └─ _is_running_in_venv() = False  ← BUG！应为 True
   └─ 继续触发 execv 或子进程 → 无限循环
```

用户现象：终端疯狂输出"向量数据库依赖安装向导"，进程 CPU 拉满，只能强制终止。

## 影响范围

- 所有需要"自举切换到 venv"的 Python 脚本
- 在 macOS + Python 3.13（及更高版本）上必现
- Linux 上视 venv 构建方式而定（symlink 构建时不触发）

## 解决方案

### ✅ 正确做法：使用 `sys.prefix`

`sys.prefix` 在 venv Python 中**始终等于 venv 目录本身**，是 Python 官方推荐的 venv 检测方式，不受解释器路径格式影响：

```python
def _is_running_in_venv() -> bool:
    """通过 sys.prefix 判断是否在项目 venv 中运行（跨平台可靠）"""
    venv_dir = project_root / '.venv-agentkit'
    try:
        return Path(sys.prefix).resolve() == venv_dir.resolve()
    except Exception:
        return False
```

### ❌ 错误做法对比

```python
# ❌ 不可靠：sys.executable 路径格式因平台/Python版本而异
return Path(sys.executable).resolve() == venv_py.resolve()

# ❌ 也不可靠：直接比较字符串，忽略路径格式差异
return sys.executable.startswith(str(venv_dir))
```

### 备用方案：`VIRTUAL_ENV` 环境变量

```python
# ✅ 备用方案（os.execv 不会传递环境变量时可能失效）
virtual_env = os.environ.get('VIRTUAL_ENV', '')
if virtual_env:
    return Path(virtual_env).resolve() == venv_dir.resolve()
```

> **注意**：`os.execv` 会继承父进程环境变量，因此 `VIRTUAL_ENV` 通常可用，但 `sys.prefix` 更可靠。

## 验证方法

```python
# 在脚本内打印调试信息验证
import sys
from pathlib import Path

print(f"sys.executable = {sys.executable}")
print(f"sys.prefix     = {sys.prefix}")
print(f"resolved exe   = {Path(sys.executable).resolve()}")
print(f"resolved prefix= {Path(sys.prefix).resolve()}")
```

## 相关代码位置

- 修复位置：`.claude/skills/vector-db-syncer/scripts/sync_vector_db.py:39-45`
- 函数：`_is_running_in_venv()`
- 修复 commit：直接修改，将 `sys.executable` 比较替换为 `sys.prefix` 比较
