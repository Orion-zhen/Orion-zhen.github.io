---
date: "2025-04-22T16:19:27+08:00"
draft: false
title: "现代的 Python 项目管理"
slug: "modern-python-pacman"
description: "使用 pyproject.toml 和 uv"
image: /post/tutorial/modern-python-pacman/cover.jpg
categories:
    - 教程
tags:
    - Python
    - 编程
license: GNU General Public License v3.0
---

> [封面出处](https://www.pixiv.net/artworks/99448917)

相较于传统的 `setup.py` + `requirements.txt` 的方式, 现代 Python 常使用 `pyproject.toml` 来管理项目依赖. 而 `uv` 是一个用 Rust 编写的, 速度远超 `pip` 的 Python 项目管理工具.

## 现代 Python 项目管理

### 现代的 Python 项目文件结构

假设项目的名称为 `myproject`, 那么它的文件结构如下:

```text
myproject/  <-- 项目根目录
├── src/        <-- 源代码目录
│   └── myproject/  <-- 包代码目录
│       ├── __init__.py  <-- 包初始化文件
│       └── cli.py       <-- 源代码文件
├── tests/      <-- 测试目录
├── pyproject.toml
├── README.md
└── ...
```

### `pyproject.toml` 概览

一个标准的项目配置文件中可以包含以下几个部分:

- `[build-system]`: 定义了如何构建整个项目, 例如构建时需要的依赖, 构建工具等
- `[project]`: 定义了项目的各种元数据, 包括名称, 版本, 项目网址, 项目依赖等
- `[tool]`: 可扩展性非常强的配置, 定义了各种工具的配置

下面我们逐个检查这些字段.

### `[build-system]`

这里主要声明构建后端和构建依赖, 构建后端常使用 `setuptools`:

```toml
[build-system]
requires = ["setuptools >= 77.0.3"]
build-backend = "setuptools.build_meta"
```

如果你就是不想遵循默认的项目文件结构, 那么可以在 `[tool.setuptools.package.find]` 中声明:

```toml
[tool.setuptools.package.find]
where = "srb"
```

这样, `setuptools` 将在 `srb/` 目录下寻找名为 `myproject` 的包入口. 默认情况况下, `setuptools` 将在 `.` 或者 `src` 目录下寻找包入口.

### `[project]`

一个标准的 `[project]` 配置项如下:

```toml
[project]
name = "myproject"
version = "114.514"

# 可选动态确定的元数据
dynamic = ["version"]

# 对 Python 版本的要求
requires-python = ">= 3.8"

# 依赖项, 类似于以前的 requirements.txt
dependencies = [
    "numpy",
    "torch",
    "transformers",
    # 可以按照不同的操作系统来配置
    "django>2.1; os_name != 'nt'",
    "django>2.0; os_name == 'nt'",
]

# 可选依赖, 可以通过类似于 `pip install ".[cli]" 的形式安装
[project.optional-dependencies]
gui = ["PyQt5"]
cli = [
    "rich",
    "click",
]
```

#### 可执行脚本入口

很多时候, 在你安装了一个软件包后, 会发现它还提供了新的命令, 例如 `vllm serve` 之类的. 这是通过配置 `[project.scripts]` 来实现的. 其格式如下:

```toml
[project.scripts]
command_name = "module.path:function_name"
```

- `command_name`: 脚本命令名, 例如 `myproject` 或者是 `vllm`
- `module.path`: 需要调用的函数所在的模块名, 例如 `myproject.cli`, 对应的文件目录是 `src/myproject/cli.py`.
- `function_name`: 调用的函数名, 例如 `main` 或者是 `serve`.

你可以注册多个不同的命令.

#### 项目元数据

在 `[project]` 中, 可以配置一些额外的信息来说明这个项目. 其格式如下:

```toml
[project]
description = "My project"
authors = [
    {name = "Pradyun Gedam", email = "pradyun@example.com"},
    {name = "Tzu-Ping Chung", email = "tzu-ping@example.com"},
    {name = "Another person"},
    {email = "different.person@example.com"},
]
maintainers = [
    {name = "Brett Cannon", email = "brett@exampple.com"}
]
readme = "README.md"
license = "GPL-3.0-or-later"
license-files = ["LICEN[CS]E*", "vendored/licenses/*.txt", "AUTHORS.md"]
keywords = ["egg", "bacon", "sausage", "tomatoes", "Lobster Thermidor"]

# 内容参考 https://packaging.python.org/en/latest/specifications/well-known-project-urls/#well-known-labels
[project.urls]
Homepage = "https://example.com"
Documentation = "https://readthedocs.org"
Repository = "https://github.com/me/spam.git"
Issues = "https://github.com/me/spam/issues"
Changelog = "https://github.com/me/spam/blob/master/CHANGELOG.md"

# 用于 PyPI 的分类器
classifiers = [
    # How mature is this project? Common values are
    #   3 - Alpha
    #   4 - Beta
    #   5 - Production/Stable
    "Development Status :: 4 - Beta",

    # Indicate who your project is intended for
    "Intended Audience :: Developers",
    "Topic :: Software Development :: Build Tools",

    # Specify the Python versions you support here.
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.6",
    "Programming Language :: Python :: 3.7",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
]
```

## `uv`

`uv` 是一个用 Rust 编写的高性能的 Python 管理工具, 包含了管理 Python 版本, 管理虚拟环境, 管理项目依赖等功能. 为什么要引入 `uv` 而不是使用 Python 原生的工具? 因为:

1. `uv` 能方便地管理不同的 Python 版本, 而不需要自己需下载 system wide 的 Python
2. `uv` 求解和下载依赖项的速度远超 `pip`, 大约是 10 倍. 而且求解依赖项的效率远高于 `pip`, 不会存在下载多个版本一个一个试的情况
3. `uv` 提供了方便的项目管理入口

命令格式:

```shell
uv <command> [<args>]
```

### Python 版本管理

使用 `uv python` 来对 Python 进行操作, 例如:

- `uv python list`: 列出所有安装的 Python 版本
- `uv python install 3.7`: 安装指定版本的 Python

### 虚拟环境

使用 `uv venv` 来对虚拟环境进行操作, 命令格式为: `uv venv -p <version>`, 其中 `-p` 表示安装指定版本的 Python, 如果不传递这个参数, 则默认选取 `PATH` 中的第一个 Python.

### 项目管理

使用 `uv init <project-name>` 来初始化一个新项目. 初始项目提供了最简单的配置文件, 脚本文件和 Git.

### 依赖管理

- `uv add <package>`: 添加依赖, 将会修改 `pyproject.toml` 文件, 并且会智能求解依赖项
- `uv remove <package>`: 移除依赖, 将会修改 `pyproject.toml` 文件, 并且会智能移除多余的依赖
- `uv sync`: 安装依赖项, 对于可选依赖项:
  - `uv sync --with [optional]`: 安装某个可选项
  - `uv sync --with op1,op2`: 安装多个可选项
  - `uv sync --with all`: 安装所有可选项
- `uv tree`: 查看依赖关系树
- `uv export -o <requirements.txt>`: 导出依赖项到 `requirements.txt`
- `uv pip`: 等效于 `pip`, 提供了和 `pip` 一样的接口, 用于老式项目管理
- `uv run <script.py>`: 运行脚本, 等价于 `python script.py`, 但是 `uv` 会自动识别并激活虚拟环境

## 实战示例

### 目标

实现一个 `myproject` 命令, 满足:

- `myproject say <words> -n <n>`, 会把传入的 `<words>` 打印出来, 重复 `<n>` 次
- `myproject test`, 会输出 `OK`

### 1. 初始化项目

找一个风水宝地, 运行命令 `uv init myproject`. 此时生成了一个 `myproject` 文件夹. 但是这个文件夹里的内容并不是严格遵循前文提到的文件目录. 我们不去管它, 还是严格按照前文的文件结构来.

### 2. 编写项目配置文件

按照前文给出的字段去填就是了, 如果有依赖项, 可以直接自己手写, 或者用 `uv add` 来添加. 这个项目没啥依赖项, 所以就不用管它了.

在配置文件中添加构建字段:

```toml
[build-system]
requires = ["setuptools"]
build-backend = "setuptools.build_meta"
```

### 3. 编写代码

创建 `src/myproject/__init__.py`, 是个空文件都没关系, 主要是为了让 Python 把这个目录当成一个包来处理.

然后创建 `src/myproject/cli.py`:

```python
# src/myproject/cli.py
import argparse
import sys # 通常需要导入 sys 来访问命令行参数，尽管 argparse 会帮你处理大部分

def say_command(args):
    """处理 'say' 子命令的逻辑"""
    for _ in range(args.n):
        print(args.words)

def test_command(args):
    """处理 'test' 子命令的逻辑"""
    print("OK")

def main():
    """
    这个函数是 'myproject' 命令的入口点。
    它使用 argparse 来解析命令行参数和子命令。
    """
    parser = argparse.ArgumentParser(
        description="My awesome project command line tool" # 命令行工具的描述
    )

    # 创建子解析器，用于定义子命令 (say, test)
    subparsers = parser.add_subparsers(
        dest="command", # 这个参数的名字将存储用户输入的子命令名称
        help="Available commands"
    )

    # --- 定义 'say' 子命令 ---
    say_parser = subparsers.add_parser("say", help="Say something multiple times")
    # 添加 'say' 命令所需的参数
    say_parser.add_argument(
        "words", # 位置参数，用户必须提供
        type=str,
        help="The words to say"
    )
    say_parser.add_argument(
        "-n", # 可选参数
        type=int,
        default=1, # 如果用户不提供 -n，默认值为 1
        help="Number of times to say the words (default: 1)"
    )
    # 将处理 'say' 子命令的函数绑定到这个解析器
    # 当用户输入 'myproject say ...' 时，argparse 会调用 say_command 函数
    say_parser.set_defaults(func=say_command)

    # --- 定义 'test' 子命令 ---
    test_parser = subparsers.add_parser("test", help="Run a simple test")
    # 将处理 'test' 子命令的函数绑定到这个解析器
    # 当用户输入 'myproject test' 时，argparse 会调用 test_command 函数
    test_parser.set_defaults(func=test_command)

    # 解析命令行参数
    args = parser.parse_args()

    # 根据解析结果执行相应的函数
    # args.func 是我们在 set_defaults 中设置的函数 (say_command 或 test_command)
    # args 中包含了所有的参数值
    if hasattr(args, "func"): # 检查是否输入了子命令
        args.func(args)
    else:
        # 如果用户只输入了 'myproject' 没有子命令，则显示帮助信息
        parser.print_help()

# 这个 if __name__ == "__main__": 块使得你可以直接运行 cli.py 文件进行测试
# 但是当作为入口点被调用时，是直接执行上面的 main() 函数
if __name__ == "__main__":
    main()
```

### 4. 添加入口

在配置文件中添加如下字段:

```toml
[project.scripts]
myproject = "myproject.cli:main"
```

### 5. 构建并运行

使用 `uv sync` 构建项目, 然后激活虚拟环境, 你将会看到 `myproject` 命令在终端中已经可用了.
