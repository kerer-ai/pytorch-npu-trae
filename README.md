# pytorch-npu CI

每日自动验证 [Ascend/pytorch](https://github.com/Ascend/pytorch) 在 PyTorch 主干最新 nightly 版本下的编译兼容性。

## 工作原理

- **触发方式**：每天 UTC 02:00（北京时间 10:00）自动触发，或手动触发
- **运行环境**：GitHub Actions 免费 x86 runner（`ubuntu-22.04`）
- **PyTorch 版本**：PyTorch 主干每日 nightly 构建（CPU 版）
- **CANN 依赖**：无需安装 CANN，使用仓库内置的桩库（`third_party/acl/libs/build_stub.sh`）

---

## 构建环境与参数详解

本节详细说明 workflow 的构建环境、依赖和编译参数，便于在其他环境复现构建。

### 1. 运行环境

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **操作系统** | `ubuntu-22.04` | GitHub Actions 托管 runner |
| **CPU 架构** | `x86_64` | Intel/AMD 64 位处理器 |
| **Python 版本** | `3.11` | 通过 `actions/setup-python@v5` 安装 |

### 2. 系统依赖

```bash
# 更新包列表
sudo apt-get update -qq

# 安装编译所需系统工具
sudo apt-get install -y --no-install-recommends \
  cmake          # 构建系统生成器
  ninja-build    # 高效构建工具
  gcc            # C 编译器
  g++            # C++ 编译器
  git            # 版本控制
  patchelf       # 修改 ELF 二进制文件（wheel 打包需要）
```

| 包名 | 用途 |
|------|------|
| `cmake` | CMake 构建系统，用于配置 C++ 项目 |
| `ninja-build` | Ninja 构建工具，比 Make 更快 |
| `gcc` / `g++` | GNU 编译器套件，编译 C/C++ 代码 |
| `git` | 克隆源码仓库和应用补丁 |
| `patchelf` | 修改 wheel 包中的 RPATH |

### 3. Python 依赖

```bash
# 升级 pip
pip install --upgrade pip

# 安装 PyTorch nightly (CPU 版本)
pip install --pre torch \
  --index-url https://download.pytorch.org/whl/nightly/cpu

# 安装构建工具
pip install pyyaml setuptools auditwheel
```

| 包名 | 版本 | 用途 |
|------|------|------|
| `torch` | nightly (CPU) | PyTorch 框架，提供 C++ 头文件 |
| `pyyaml` | latest | 解析 YAML 配置文件 |
| `setuptools` | latest | Python 包构建工具 |
| `auditwheel` | latest | 检查和修复 wheel 包兼容性 |

### 4. PyTorch Nightly 版本

| 项目 | 值 |
|------|-----|
| **下载源** | `https://download.pytorch.org/whl/nightly/cpu` |
| **版本格式** | `torch==2.11.0.devYYYYMMDD` |
| **变体** | CPU only（无 CUDA 支持） |

**指定日期安装：**
```bash
# 安装特定日期的 nightly 版本
DATE="20260308"
pip install --pre \
  "torch==2.11.0.dev${DATE}" \
  --index-url https://download.pytorch.org/whl/nightly/cpu
```

**安装最新 nightly：**
```bash
pip install --pre torch \
  --index-url https://download.pytorch.org/whl/nightly/cpu
```

### 5. 源码仓库

| 项目 | 值 |
|------|-----|
| **仓库地址** | `https://github.com/Ascend/pytorch.git` |
| **克隆深度** | `--depth=1`（浅克隆） |
| **子模块** | `--recurse-submodules`（包含 op-plugin） |

```bash
git clone --depth=1 --recurse-submodules \
  https://github.com/Ascend/pytorch.git ascend_pytorch
```

### 6. 编译环境变量

| 变量名 | 值 | 说明 |
|--------|-----|------|
| `DISABLE_INSTALL_TORCHAIR` | `TRUE` | 禁用 torchair（需要额外子模块） |
| `DISABLE_RPC_FRAMEWORK` | `TRUE` | 禁用 RPC 框架（简化构建） |
| `BUILD_WITHOUT_SHA` | `1` | 不在版本号中嵌入 git sha |

```bash
export DISABLE_INSTALL_TORCHAIR=TRUE
export DISABLE_RPC_FRAMEWORK=TRUE
export BUILD_WITHOUT_SHA=1
```

### 7. 补丁应用

补丁文件位于 `patches/` 目录，按文件名顺序应用：

```bash
PATCH_DIR="./patches"
for p in "${PATCH_DIR}"/*.patch; do
  [ -f "$p" ] || continue
  echo "Applying patch: $(basename $p)"
  git apply --directory=ascend_pytorch "$p"
done
```

### 8. 构建命令

```bash
cd ascend_pytorch

# 设置环境变量
export DISABLE_INSTALL_TORCHAIR=TRUE
export DISABLE_RPC_FRAMEWORK=TRUE
export BUILD_WITHOUT_SHA=1

# 构建 wheel 包
python setup.py build bdist_wheel

# 生成的 wheel 位于
ls dist/*.whl
```

---

## 完整复现步骤

以下是在本地 Ubuntu 22.04 x86_64 环境完整复现构建的步骤：

```bash
# 1. 安装系统依赖
sudo apt-get update -qq
sudo apt-get install -y --no-install-recommends \
  cmake ninja-build gcc g++ git patchelf

# 2. 安装 Python 3.11（如未安装）
# Ubuntu 22.04 默认 Python 3.10，需添加 PPA
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt-get install python3.11 python3.11-venv python3.11-dev

# 3. 创建虚拟环境
python3.11 -m venv venv
source venv/bin/activate

# 4. 安装 Python 依赖
pip install --upgrade pip
pip install --pre torch --index-url https://download.pytorch.org/whl/nightly/cpu
pip install pyyaml setuptools auditwheel

# 5. 克隆本仓库（包含 patches）
git clone https://github.com/kerer-ai/pytorch-npu-trae.git
cd pytorch-npu-trae

# 6. 克隆 Ascend/pytorch
git clone --depth=1 --recurse-submodules \
  https://github.com/Ascend/pytorch.git ascend_pytorch

# 7. 应用补丁
for p in patches/*.patch; do
  [ -f "$p" ] || continue
  git apply --directory=ascend_pytorch "$p"
done

# 8. 构建
cd ascend_pytorch
export DISABLE_INSTALL_TORCHAIR=TRUE
export DISABLE_RPC_FRAMEWORK=TRUE
export BUILD_WITHOUT_SHA=1
python setup.py build bdist_wheel

# 9. 查看产物
ls -la dist/*.whl
```

---

## 构建流程

1. 安装 PyTorch nightly（从 `download.pytorch.org/whl/nightly/cpu`）
2. 克隆 `Ascend/pytorch`（含 `op-plugin` 子模块）
3. 按序应用 `patches/` 目录下所有兼容性 patch
4. 编译 CANN 桩库（`build_stub.sh`，仅需 GCC）
5. 执行 `python setup.py build bdist_wheel`
6. 上传构建日志和生成的 wheel 包

## 查看结果

- [Actions 页面](../../actions/workflows/nightly-build.yml) 查看每次构建状态
- 每次运行的 Step Summary 包含 PyTorch 版本、Ascend/pytorch commit、已应用 patch 列表和构建结果
- 构建失败时可下载 `build-log` artifact 查看详细编译错误

## 手动触发

在 Actions 页面点击 **Run workflow**，可选填 PyTorch nightly 日期（格式 `YYYYMMDD`），留空使用最新版。

---

## 问题处理（Claude Code Skills）

本仓库内置三个 Claude Code slash commands，覆盖从检测到修复的完整闭环。在项目目录下启动 Claude Code 后即可使用。

### 典型工作流

```
CI 构建失败
  │
  ├─ /analyze-failure   ← 拉取日志、定位根因、对比 PyTorch 新旧 API
  │
  ├─ /report-issue      ← 按日期编号创建 issues/ 文档
  │
  └─ /gen-patch         ← 修改源码、生成 patch、提交、触发验证
                               │
                        下次构建推进更远？
                          ├─ 是 → 继续 /analyze-failure 处理下一个问题
                          └─ 否 → 检查 patch 是否正确应用
```

---

### `/analyze-failure` — 分析 CI 失败原因

CI 变红后的第一步。执行内容：

1. `gh run list` 找到最新失败 Run
2. `gh run view --log-failed` 拉取失败日志
3. 过滤 `error:`、`make[2]`、`Traceback` 等关键行
4. 读取受影响的 Ascend/pytorch 源文件
5. 在 PyTorch nightly 头文件（`/usr/local/lib/python3.12/dist-packages/torch/include/`）中查找变更后的 API 定义
6. 输出结构化报告：已生效 patch、错误摘要、根本原因、受影响范围、建议修复方向

```
/analyze-failure
```

**关键观察**：构建运行时长比上次更长 → 前序 patch 已生效，本次暴露新问题。

---

### `/report-issue` — 创建 issue 记录

在 `/analyze-failure` 完成后使用。按 `YYYY-MM-DD-NNN` 规则编号，在 `issues/` 下创建 Markdown 文档，记录：问题描述、根本原因分析（含错误日志）、修复方案、对应 patch 文件名、前置 patch（如有）。

```
/report-issue
```

示例：`issues/2026-03-07-001-CachingHostAllocator-HostBlockPool-api-break.md`

---

### `/gen-patch` — 生成 patch 并触发验证

执行内容：

1. 确认本地克隆（`/root/ascend_pytorch_tmp`）可用
2. 对比新旧 API，按最小改动原则修改受影响源文件
3. `git diff <文件> > patches/NNNN-fix-<模块>-<描述>.patch` 生成 patch
4. 提交 patch + issue 文档并 push
5. `gh workflow run` 触发新一轮构建
6. 观察构建进度判断 patch 是否生效

```
/gen-patch
```

示例：`patches/0002-fix-NPUSHMEMSymmetricMemory-GroupInfo-api-compat.patch`

**常见修复模式：**

| 错误类型 | 修复方式 |
|----------|----------|
| 结构体成员被删除（原用于缓存） | `.cpp` 模块级引入 `static map + mutex` 替代 |
| 成员重命名（如 `blocks` → `blocks_`） | 重命名 Ascend 侧冲突的同名类型，消除 typedef 遮蔽 |
| 基类新增纯虚函数 | 子类 `.hpp` 中补充 override（无对应语义则返回 0） |
| 虚函数签名新增参数 | 更新 override 签名，原逻辑提取为私有方法 |

---

### Patch 生命周期

```
新增 patch → CI 自动应用（git apply --directory=ascend_pytorch）
                │
     apply 成功 ✅ → 构建推进，问题修复
     apply 失败 ❌ → 上游已合入该修复 → 删除 patch 文件并提交
```

> Patch 打在 CI 临时克隆的副本上，**不修改 Ascend/pytorch 官方仓库**。
