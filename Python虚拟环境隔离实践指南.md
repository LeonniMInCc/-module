# Python 虚拟环境隔离工程实践指南

> 面向工程实践的操作手册：涵盖 venv 及主流虚拟环境工具的创建、激活、依赖管理，以及与 Git 协作的安全规范。
> 适用对象：Python 开发者、安全工程师（依赖供应链安全视角）。

---

## 目录

1. [为什么需要虚拟环境](#1-为什么需要虚拟环境)
2. [虚拟环境的工作原理](#2-虚拟环境的工作原理)
3. [主流工具对比与选型](#3-主流工具对比与选型)
4. [venv 实操步骤](#4-venv-实操步骤)
5. [依赖管理与版本锁定](#5-依赖管理与版本锁定)
6. [与 Git 协作规范](#6-与-git-协作规范重点)
7. [安全视角](#7-安全视角供安全从业者参考)
8. [常见问题排查](#8-常见问题排查)
9. [完整操作速查表](#9-完整操作速查表)

---

## 1. 为什么需要虚拟环境

| 问题 | 说明 |
|------|------|
| **依赖冲突** | 项目 A 需要 `requests==2.28`，项目 B 需要 `requests==2.31`，全局环境无法共存 |
| **版本漂移** | 升级系统级包后，旧项目突然无法运行 |
| **可复现性** | 团队、CI、生产环境无法复现本地开发环境 |
| **环境污染** | 全局 `site-packages` 越装越乱，卸载困难、互相干扰 |
| **安全风险** | 全局环境权限过高，被污染的依赖影响面更大 |

**核心思想**：每个项目拥有独立的 Python 解释器环境 + 独立的 `site-packages` 目录，互不干扰、按需销毁。

---

## 2. 虚拟环境的工作原理

虚拟环境的本质是：

- 一个独立目录（惯例命名 `.venv` 或 `venv`）
- 内含 Python 解释器的符号链接/副本，以及独立的 `site-packages`、`Scripts`/`bin` 目录
- **激活 = 修改环境变量**：把 `.venv/Scripts`（Windows）或 `.venv/bin`（Linux/macOS）插入 `PATH` 最前，并设置 `VIRTUAL_ENV` 变量
- 激活后 `python`、`pip` 均指向虚拟环境内版本，安装的包只进入虚拟环境的 `site-packages`

> 验证技巧：`which python`（Linux/macOS）或 `where python`（Windows）查看当前解释器路径；命令行前缀出现 `(.venv)` 表示已在虚拟环境中。

---

## 3. 主流工具对比与选型

| 工具 | 特点 | 适用场景 |
|------|------|----------|
| **venv**（标准库） | Python ≥ 3.3 自带，零依赖、零安装 | 绝大多数项目的默认选择 |
| **virtualenv** | 更老牌，支持 Python 2、支持复制 | 兼容遗留项目 |
| **conda** | 跨语言环境管理、自带科学计算二进制包 | 数据科学、需要非 Python 原生依赖 |
| **pipenv** | `Pipfile` + 锁文件，面向应用 | 传统应用开发 |
| **poetry** | `pyproject.toml` + 严格依赖解析 | 库开发、现代项目 |
| **uv** | Rust 实现，速度极快，兼容 pip/venv 生态 | 追求开发体验的新项目 |

**工程实践建议**：默认使用 `venv`；需要复杂依赖解析或发布库时，选用 `poetry` 或 `uv`。

---

## 4. venv 实操步骤

### 4.1 创建虚拟环境

```bash
# 进入项目目录
cd myproject

# 创建虚拟环境（推荐目录名 .venv，隐藏目录避免误提交）
python -m venv .venv

# 指定其他 Python 版本（本机安装多个版本时）
py -3.12 -m venv .venv          # Windows（py 启动器）
python3.12 -m venv .venv        # Linux / macOS
```

### 4.2 激活虚拟环境

```bash
# Windows（cmd）
.venv\Scripts\activate.bat

# Windows（PowerShell）
.venv\Scripts\Activate.ps1

# Linux / macOS
source .venv/bin/activate
```

激活成功后命令行前缀出现 `(.venv)`。

> **PowerShell 报错**「无法加载文件 ... 因为在此系统上禁止运行脚本」时：
>
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

### 4.3 安装依赖

```bash
pip install -U pip              # 优先升级 pip 本身
pip install requests flask      # 安装包
pip list                        # 查看当前环境已安装的包
pip show requests                # 查看单个包详情
```

### 4.4 退出与删除

```bash
deactivate                       # 退出虚拟环境（回到全局环境）

# 删除虚拟环境 = 直接删除目录，不影响系统 Python
rm -rf .venv                     # Linux / macOS
Remove-Item -Recurse -Force .venv   # Windows PowerShell
```

---

## 5. 依赖管理与版本锁定

### 5.1 requirements.txt 基础用法

```bash
pip freeze > requirements.txt       # 导出当前环境所有包的精确版本
pip install -r requirements.txt     # 在另一台机器/CI 上复现环境
```

### 5.2 分离生产与开发依赖

```text
# requirements.txt（生产依赖）
flask==3.0.0
requests==2.31.0

# requirements-dev.txt（开发依赖，引用生产文件）
-r requirements.txt
pytest==8.0.0
ruff==0.3.0
```

安装：`pip install -r requirements-dev.txt`

### 5.3 现代方式：pyproject.toml（poetry / uv）

```toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "flask>=3.0",
    "requests>=2.31",
]

[project.optional-dependencies]
dev = ["pytest", "ruff"]
```

```bash
uv sync          # 或 poetry install
```

### 5.4 版本锁定要点（工程实践关键）

- `requirements.txt` 使用**精确版本**（`==`），避免 `>=` 带来的不确定性
- 使用 `pip-tools`（`pip-compile`）或 `uv lock` 生成**锁文件**，保证完全可复现
- 锁文件应随代码一起提交到仓库

---

## 6. 与 Git 协作规范（重点）

### 6.1 绝对不要提交虚拟环境目录

虚拟环境目录动辄数百 MB，且包含本机绝对路径，提交会带来：

- **仓库臃肿**：clone 极慢，浪费存储
- **平台不兼容**：Windows 下的 `.exe` 无法在 Linux 使用
- **安全风险**：可能包含带已知漏洞的包、泄露本机路径信息

**原则：提交"如何构建环境"的描述（requirements/pyproject），而不是环境本身。**

### 6.2 .gitignore 模板（Python 项目）

```gitignore
# 虚拟环境
.venv/
venv/
env/
ENV/

# Python 缓存与编译产物
__pycache__/
*.py[cod]
*.egg-info/
.pytest_cache/
.mypy_cache/
.ruff_cache/
build/
dist/

# 敏感文件（密钥、配置）
.env
.env.*
*.pem
*.key
*.p12
id_rsa
id_ed25519
config.ini
```

### 6.3 应提交的文件结构示例

```
myproject/
├── .venv/              ← 忽略，绝不提交
├── src/                ← 提交
├── requirements.txt    ← 提交（环境可复现的关键）
├── pyproject.toml      ← 提交
├── .gitignore          ← 提交
└── README.md           ← 提交
```

---

## 7. 安全视角（供安全从业者参考）

1. **venv 不是安全边界**：它只隔离依赖，不隔离进程与文件系统。恶意代码在 venv 中同样可以读写用户文件。
2. **供应链安全**：
   ```bash
   # 校验依赖哈希，防止包被篡改（配合 requirements.txt 中的 --hash 行）
   pip install --require-hashes -r requirements.txt

   # 扫描依赖已知漏洞（OSV 数据库）
   pip install pip-audit
   pip-audit
   ```
3. **不要用管理员/root 权限安装依赖**：降低恶意包提权后的影响范围。
4. **警惕 venv 中的二进制被提交**：`.exe`/`.so` 可能被 CI 或他人执行，成为供应链投毒入口。
5. **防范依赖仿冒（typosquatting）**：检查 requirements 中的包名拼写，如 `request`（仿冒）vs `requests`（正版）、`pygame` vs `pygame-py` 等。
6. **隔离开发环境中的密钥**：`.env`、`*.pem` 必须进 `.gitignore`，并确认从未被提交进 Git 历史（已泄露则需轮换密钥）。

---

## 8. 常见问题排查

| 现象 | 原因 | 解决方法 |
|------|------|----------|
| `pip` 装到了全局环境 | 未激活虚拟环境 | 激活后检查 `which pip` 路径 |
| PowerShell 激活失败 | 执行策略限制脚本 | `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` |
| 用了错误的 Python 版本 | 系统存在多个 Python | 用 `py -3.12` 显式指定版本创建 |
| venv 里的包突然消失 | `.venv` 目录被删除 | 依据 requirements.txt 重建：`python -m venv .venv && pip install -r requirements.txt` |
| 误提交了 `.venv` | 缺少 .gitignore | 补上规则后执行 `git rm -r --cached .venv` 并重新提交 |
| `deactivate` 命令找不到 | 不在虚拟环境中 | 检查提示符前缀 `(.venv)` |
| IDE 未识别虚拟环境 | 未选择解释器 | VSCode：`Ctrl+Shift+P` → Python: Select Interpreter → 选择 `.venv` 路径 |

---

## 9. 完整操作速查表

| 操作 | 命令 |
|------|------|
| 创建虚拟环境 | `python -m venv .venv` |
| 激活（Windows cmd） | `.venv\Scripts\activate.bat` |
| 激活（Windows PowerShell） | `.venv\Scripts\Activate.ps1` |
| 激活（Linux/macOS） | `source .venv/bin/activate` |
| 退出虚拟环境 | `deactivate` |
| 导出依赖 | `pip freeze > requirements.txt` |
| 安装依赖 | `pip install -r requirements.txt` |
| 依赖漏洞扫描 | `pip-audit` |
| 删除虚拟环境 | `rm -rf .venv` |

---

*本指南基于工程实践总结，适用于 Python 3.3+ 环境。安全相关建议结合 OWASP 供应链安全最佳实践。*
