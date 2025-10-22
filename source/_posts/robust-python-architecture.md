---
title: 健壮的现代 Python 项目架构
date: 2025-10-22 11:38:00
excerpt: 告别祖传代码，搭建一个能打的 Python 项目架构
categories:
  - 工程实践
tags: 
  - Python
---

我们将分为两个部分：首先是适用于大多数中小型应用的**标准项目结构**，然后是针对大型复杂应用的 **Monorepo（多包仓库）架构**。

## **一、基础理念：`uv` 与 `src-layout`**

在开始构建项目之前，我们先统一我们的基础工具和核心原则。

1.  **项目与环境管理：`uv`**

    - `uv` 是一个用 Rust 编写的极速 Python 包安装器和解析器，旨在替代 `pip` 和 `pip-tools`。它提供了闪电般的依赖解析和安装速度，并能与虚拟环境无缝集成。我们将使用 `uv` 来管理项目的虚拟环境和所有依赖。

3.  **源代码布局：`src-layout`**

    - 我们将所有核心应用代码放置在 `src` 目录中。这种布局能明确区分项目的**可安装源码**与**开发辅助文件**（如测试、脚本、配置等），从根本上解决了许多 Python 导入路径可能引发的模糊性和潜在冲突，确保了项目在不同环境中的一致性。

## **二、标准项目结构：简单而强大**

这个结构适用于绝大多数独立应用、库或工具，它在简洁性和可扩展性之间取得了完美的平衡。

### **1. 项目目录结构**

我们首先创建一个清晰的目录树，用于存放不同类型的文件。

```plaintext
simple_project/
├── .gitignore          # Git 忽略规则，用于排除虚拟环境、缓存等
├── .venv/              # 由 uv 创建和管理的虚拟环境目录
├── src/                # 存放所有可安装的 Python 源代码
│   └── simple_project/
│       ├── __init__.py
│       └── utils.py    # 示例模块，包含核心功能
├── tests/              # 存放所有测试用例
│   └── test_utils.py
├── main.py             # 项目的主入口或启动脚本
├── pyproject.toml      # 项目配置文件 (PEP 621)，定义元数据和依赖
└── README.md           # 项目说明文档
```

### **2. 配置 `pyproject.toml`**

`pyproject.toml` 是现代 Python 项目的中心配置文件。我们在这里定义项目的基本信息和依赖。

```toml
# pyproject.toml

[project]
name = "simple-project"
version = "0.1.0"
description = "一个标准 Python 项目的示例"
requires-python = ">=3.10"

# 生产环境依赖
dependencies = [
    "requests>=2.30.0",
]

[project.optional-dependencies]
# 开发环境依赖（测试、代码检查等）
dev = [
    "pytest>=8.0.0",
    "ruff>=0.4.0",
]
```

### **3. 初始化环境与安装依赖**

现在，我们使用 `uv` 来准备开发环境。

```bash
# 1. 进入项目根目录
cd simple_project

# 2. 使用 uv 创建一个名为 .venv 的虚拟环境
uv venv

# 3. 激活虚拟环境 (Windows/Linux/macOS)
# Windows (PowerShell):
# .\.venv\Scripts\Activate.ps1
# Linux/macOS:
# source .venv/bin/activate

# 4. 安装所有依赖（包括 dev 依赖）
uv pip install -e ".[dev]"
```
**关键点解释**：
*   `uv venv`: 在当前目录下创建虚拟环境。
*   `uv pip install -e ".[dev]"`:
    *   `-e` 代表 **"editable"（可编辑模式）**。它将 `src` 目录下的代码链接到虚拟环境中，而不是复制。这意味着你对源代码的任何修改都会立刻生效，无需重新安装。
    *   `.` 代表当前项目。
    *   `[dev]` 指明同时安装 `project.optional-dependencies` 中定义的 `dev` 依赖组。

### **4. 编写与导入代码**

得益于可编辑模式安装，我们可以在项目的任何地方使用简洁、一致的导入路径。

**`src/simple_project/utils.py`**
```python
import requests

def fetch_data(url: str) -> str:
    """从给定的 URL 获取数据并返回文本内容。"""
    try:
        response = requests.get(url)
        response.raise_for_status()  # 如果请求失败则抛出异常
        return response.text
    except requests.RequestException as e:
        return f"请求失败: {e}"
```

**`main.py`** (项目根目录的启动脚本)
```python
# 导入路径从包名开始，src 目录是透明的
from simple_project.utils import fetch_data

def main():
    print("正在执行主程序...")
    data = fetch_data("https://httpbin.org/get")
    print("获取到的数据（部分）:", data[:100])

if __name__ == "__main__":
    main()
```
运行主程序：
```bash
# 确保虚拟环境已激活
python main.py
```

## **三、Monorepo 架构：应对复杂性**

当项目演变为多个相互关联、但又希望保持独立的子包时，Monorepo 架构是理想的选择。例如，一个项目可能包含一个核心库、一个数据库访问层和一个对外提供服务的 API 应用。

### **1. Monorepo 目录结构**

我们将不同的子包分别存放在 `packages/` 和 `apps/` 目录中，以明确它们的角色。

```
complex_project/
├── .venv/
├── apps/                 # 存放最终的可执行应用
│   └── api_server/
│       ├── pyproject.toml
│       └── src/
│           └── api_server/
│               └── main.py
├── packages/             # 存放可被复用的基础库
│   └── core_utils/
│       ├── pyproject.toml
│       └── src/
│           └── core_utils/
│               └── string_helpers.py
├── pyproject.toml        # **工作区根配置文件**
└── README.md
```

### **2. 配置工作区 (Workspace)**

`uv` 支持工作区（Workspace）的概念，允许我们在根 `pyproject.toml` 中统一管理多个子包。

**根 `pyproject.toml`**:
```toml
[tool.uv.workspace]
# 声明工作区包含的成员
members = [
    "packages/*",
    "apps/*",
]

# 这里可以定义一些在所有子包中共享的配置
# 例如，开发依赖可以放在这里，但更推荐放在各自的包中以保持独立性
```

**`packages/core_utils/pyproject.toml`**:
```toml
[project]
name = "core-utils"
version = "0.1.0"
dependencies = [] # 这个包是纯粹的基础库，没有外部依赖
```

**`apps/api_server/pyproject.toml`**:
```toml
[project]
name = "api-server"
version = "0.1.0"
dependencies = [
    # 关键：使用相对路径声明对工作区内另一个包的依赖
    "core-utils @ {root:uri}/packages/core_utils",
    "fastapi>=0.110.0",
    "uvicorn>=0.29.0",
]
```
`{root:uri}` 是一个占位符，`uv` 会在解析时将其替换为工作区的根路径。

### **3. 初始化与安装**

在 Monorepo 项目中，安装过程与标准项目类似，但 `uv` 会智能地识别并处理工作区内的所有包及其相互依赖。

```bash
# 1. 进入 Monorepo 项目根目录
cd complex_project

# 2. 创建并激活虚拟环境
uv venv
source .venv/bin/activate

# 3. 安装工作区内的所有包及其依赖
uv pip install -e "."
```
`uv` 会自动发现 `workspace.members` 中定义的所有包，并将它们以可编辑模式安装到同一个虚拟环境中。

### **4. 跨包代码调用**

现在，`api_server` 应用可以无缝地调用 `core_utils` 库中的代码。

**`packages/core_utils/src/core_utils/string_helpers.py`**:
```python
def capitalize_text(text: str) -> str:
    """将文本的首字母大写。"""
    if not text:
        return ""
    return text.capitalize()
```

**`apps/api_server/src/api_server/main.py`**:
```python
from fastapi import FastAPI
# 直接导入本地包，就像导入普通的第三方库一样
from core_utils.string_helpers import capitalize_text

app = FastAPI()

@app.get("/")
def read_root():
    message = "hello world"
    capitalized_message = capitalize_text(message)
    return {"message": message, "capitalized": capitalized_message}
```
运行 FastAPI 应用：
```bash
# 在 complex_project 根目录执行
uvicorn api_server.main:app --reload --app-dir src
```
`--app-dir src` 参数告诉 `uvicorn` 在 `src` 目录中查找应用模块，这与我们的 `src-layout` 结构完美契合。

## **结论**

一个经过深思熟虑的项目架构是高质量软件的基石。通过采用 `uv` 这样的高效工具和 `src-layout` 这样的最佳实践，你可以为你的 Python 项目打下坚实的基础。无论是构建一个简单的脚本工具还是一个复杂的多包系统，遵循这些原则都将使你的开发过程更加顺畅，项目也更易于维护和扩展。