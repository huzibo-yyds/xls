# XLS 入门与 Ubuntu 无 sudo 构建指南

本文面向在 Linux Ubuntu 服务器上工作的普通用户（无 sudo 权限），目标是：理解 XLS 是什么、看懂仓库结构、并尽可能在当前权限下把 XLS 跑起来。

## 1. XLS 工具介绍

XLS（Accelerated HW Synthesis）是一个开源的高层次综合（HLS）工具链，许可证为 Apache 2.0。它的核心能力是把高层描述（主要是 DSLX，也支持部分 C++ 前端）转换为可综合的 Verilog/SystemVerilog。

从工程实践看，XLS 的典型价值有三点：

1. 用更接近软件开发的方式描述硬件行为（DSLX/IR）。
2. 在同一语义基础上完成解释执行、优化、调度、代码生成，减少“软件模型”和“硬件实现”不一致的问题。
3. 支持形式化验证、仿真与可视化，便于定位优化和正确性问题。

一个典型流水线是：

1. DSLX 源码 -> IR（中间表示）
2. IR 优化（pass pipeline）
3. 调度（决定时序/流水级）
4. 代码生成（Verilog/SystemVerilog）
5. 可选：仿真、形式化等价检查、可视化

常见命令行工具包括：

- `interpreter_main`：执行 DSLX 测试/解释运行。
- `ir_converter_main`：DSLX 到 IR。
- `opt_main`：IR 优化。
- `codegen_main`：IR 到 RTL。
- `ir_viz`：IR 可视化服务。

## 2. 仓库代码组成解析

你这个仓库是标准 Bazel 大仓风格，主干在 `xls/`，周边是依赖与文档。可以按“职责层”理解：

### 2.1 顶层目录角色

- `dependency_support/`：第三方依赖加载与补丁（Bazel 外部仓配置、Python requirements 锁定等）。
- `docs_src/`：官方文档源文件（构建文档、IR/DSLX 说明、教程）。
- `third_party/`：内置第三方代码或镜像。
- `xls/`：核心实现，包含前端、IR、优化、调度、后端生成、仿真、测试等。
- `MODULE.bazel` 与 `WORKSPACE`：Bazel 模块化依赖与兼容配置。

### 2.2 `xls/` 目录中的关键子系统

- `xls/dslx/`：DSLX 语言前端（解析、类型系统、解释器、IR 转换入口）。
- `xls/ir/`：XLS IR 定义与处理基础设施。
- `xls/passes/`：IR 优化 pass。
- `xls/scheduling/`：时序/流水调度。
- `xls/codegen/`：RTL 生成（含 VAST 等抽象）。
- `xls/interpreter/` 与 `xls/jit/`：IR 执行引擎（解释/JIT）。
- `xls/simulation/`：仿真相关封装与测试基建。
- `xls/solvers/`：形式化/SMT 相关能力（例如与 Z3 的接口）。
- `xls/tools/`：面向用户的 CLI 工具集合。
- `xls/visualization/`：IR 可视化等工具。
- `xls/tests/`：跨模块集成测试。
- `xls/contrib/xlscc/`：可选 C++ 前端相关内容（构建更重）。

### 2.3 构建系统关键信息

- Bazel 版本文件是 `.bazelversion`，当前固定为 `7.4.1`。
- `.bazelrc` 中启用了 bzlmod（`--enable_bzlmod`），并包含较多编译选项。
- `MODULE.bazel` 指定了 Python 工具链版本（3.12）和大量外部依赖。

因此，建议你优先按仓库固定版本和默认配置来构建，不要自己随意换 Bazel 主版本。

## 3. Ubuntu 无 sudo 如何构建并跑起来

先说结论：无 sudo 的关键是“把工具链装到用户目录”，并尽量复用服务器已有编译环境。

### 3.1 先判断你是否具备基础编译条件

执行下面检查：

```bash
command -v gcc && gcc --version
command -v g++ && g++ --version
command -v zip && zip -v | head -n 1
command -v unzip && unzip -v | head -n 1
command -v python3 && python3 --version
command -v java && java -version
```

如果其中 `gcc/g++/zip/unzip/java` 缺失，纯用户权限通常无法补齐系统依赖；你需要：

1. 让管理员安装系统包，或
2. 使用你可控的容器/环境（如 rootless 容器、可用的集群模块环境）。

### 3.2 无 sudo 安装 Bazel（用户目录）

仓库要求 Bazel 7.4.1。最简单是安装 bazelisk 到 `~/bin`：

```bash
mkdir -p "$HOME/bin"
curl -L https://github.com/bazelbuild/bazelisk/releases/download/v1.27.0/bazelisk-linux-amd64 -o "$HOME/bin/bazel"
chmod +x "$HOME/bin/bazel"
export PATH="$HOME/bin:$PATH"

bazel version
```

说明：bazelisk 会自动读取仓库中的 `.bazelversion` 并下载匹配 Bazel。

### 3.3 处理“无 sudo 但需要 python 命令”问题

官方文档里用 `python-is-python3`（需 sudo）来提供 `python` 命令。无 sudo 可用用户级替代：

```bash
mkdir -p "$HOME/bin"
ln -sf "$(command -v python3)" "$HOME/bin/python"
export PATH="$HOME/bin:$PATH"

python --version
```

这样 `/usr/bin/env python` 能在你的 PATH 里找到 Python 3。

### 3.4 构建命令（推荐从轻到重）

在仓库根目录执行：

```bash
# 先做轻量验证：仅构建常用工具
bazel build -c opt //xls/dslx:interpreter_main //xls/dslx/ir_convert:ir_converter_main //xls/tools:opt_main //xls/tools:codegen_main

# 再做 DSLX 主线测试（不含 xlscc，构建更快）
bazel test -c opt -- //xls/... -//xls/contrib/xlscc/...
```

如果你要全量（更慢）：

```bash
bazel test -c opt -- //xls/...
```

### 3.5 最小可运行验证（确认“跑起来”）

```bash
cat > /tmp/simple_add.x <<'EOF'
fn add(x: u32, y: u32) -> u32 {
  x + y + u32:0
}

#[test]
fn test_add() {
  assert_eq(add(u32:2, u32:3), u32:5)
}
EOF

# 1) 解释执行 DSLX
./bazel-bin/xls/dslx/interpreter_main /tmp/simple_add.x

# 2) 转 IR
./bazel-bin/xls/dslx/ir_convert/ir_converter_main --top=add /tmp/simple_add.x > /tmp/simple_add.ir

# 3) IR 优化
./bazel-bin/xls/tools/opt_main /tmp/simple_add.ir > /tmp/simple_add.opt.ir

# 4) 生成 Verilog
./bazel-bin/xls/tools/codegen_main --pipeline_stages=1 --delay_model=unit /tmp/simple_add.opt.ir > /tmp/simple_add.v

head -n 40 /tmp/simple_add.v
```

只要上述链路成功，你的 XLS 在当前账号下就已经可用。

### 3.6 常见问题与无 sudo 场景建议

- 问题：下载依赖慢或失败
  - 建议：配置代理/镜像；为 Bazel 开启磁盘缓存（用户目录）。

```bash
mkdir -p "$HOME/.bazel_disk_cache"
echo "build --disk_cache=$HOME/.bazel_disk_cache" >> "$HOME/.bazelrc"
echo "test --disk_cache=$HOME/.bazel_disk_cache" >> "$HOME/.bazelrc"
```

- 问题：权限受限、系统依赖不齐导致无法源码构建

  - 备选 A：使用官方发布的预编译工具包（不编译源码，直接运行工具）。
  - 备选 B：使用 Conda 包（若服务器已有 conda/mamba）。
- 问题：只想先体验 DSLX 到 RTL

  - 建议：先只构建 4 个核心工具（上文 3.4 第一条），避免全仓测试耗时。

## 4. 给你的实操建议（按成功率排序）

1. 先跑 3.1 的依赖检查。
2. 依赖齐全就走 3.2 -> 3.3 -> 3.4（轻量构建）-> 3.5（最小验证）。
3. 若依赖不齐且无 sudo，优先申请管理员补齐 `build-essential/python3-dev/libtinfo6/zip/default-jdk` 这类系统包，或转用预编译发行包先开展工作。

---

## 5. 代理和环境问题诊断与修复（2026-03-17）

本章记录了在 Ubuntu 服务器上，利用 Clash 代理后进行源码构建的完整诊断和修复过程。

### 5.1 初期问题现象

用户反馈：
- VNC 页面浏览器能正常访问 Google 等网站
- 终端 `ping github.com` 时超时
- `bazel build` 在拉取外部依赖时失败，报告 GitHub 访问超时

这看似是终端网络问题与浏览器不一致的情况。

### 5.2 网络诊断步骤与结果

#### 步骤 1：检查代理环境变量

```bash
env | grep -iE '^(http|https|all|no)_proxy='
```

**结果**：终端未设置任何代理环境变量（为空）。

#### 步骤 2：检查本地 Clash 监听端口

```bash
ss -lntp | grep -E ':(7890|7891|1080|9090|9091)\b'
```

**结果**：Clash 进程在监听四个端口：
- 127.0.0.1:7890（HTTP 代理）
- 127.0.0.1:7891（SOCKS5 代理）
- 127.0.0.1:9090（UI 管理端口）
- 127.0.0.1:9091（UI 管理端口）

#### 步骤 3：测试终端直连 GitHub HTTPS

```bash
curl -I --connect-timeout 8 --max-time 15 https://github.com
```

**结果**：HTTP/2 200 OK，访问成功。

#### 步骤 4：测试 git 访问 GitHub 仓库

```bash
git ls-remote https://github.com/hedronvision/bazel-compile-commands-extractor.git HEAD
```

**结果**：成功返回 commit abb61a688167623088f8768cc9264798df6a9d10。

#### 步骤 5：测试 GitHub Release 下载

```bash
curl -I -L --connect-timeout 8 --max-time 30 \
  https://github.com/bazel-contrib/rules_python/releases/download/1.5.4/rules_python-1.5.4.tar.gz
```

**结果**：HTTP/2 302 重定向至 release-assets.githubusercontent.com，连接正常。

### 5.3 根因分析

#### 问题 1：ping 失败不代表网络不通

- `ping` 使用 ICMP 协议，不走 HTTP/SOCKS 代理
- 很多网络环境禁用 ICMP 出站
- curl 和 git 都是 TCP 连接，走不同的路由，因此能访问

**结论**：ping 失败是假象，真实网络通路正常。

#### 问题 2：真实的编译环境污染

首轮 Bazel 构建在清掉 `--ignore_dev_dependency` 后进入大规模编译，但在链接 clang 工具链时失败：

```
external/toolchains_llvm~~llvm~llvm_toolchain_llvm/bin/clang: 
  /mnt/home/tools/xilinx/Vitis_HLS/2022.1/lib/lnx64.o/Default/libstdc++.so.6: 
  version `GLIBCXX_3.4.26' not found
```

**根因**：你的 `LD_LIBRARY_PATH` 包含：
```
:/usr/local/cuda/lib64
:/mnt/home/tools/xilinx/Vitis_HLS/2022.1/lib/lnx64.o
:/mnt/home/tools/xilinx/Vitis_HLS/2022.1/lib/lnx64.o/Default
:/mnt/home/tools/xilinx/Vivado/2022.1/lib/lnx64.o
:/mnt/home/tools/xilinx/Vivado/2022.1/lib/lnx64.o/Default
```

这导致了 Xilinx 提供的旧版 `libstdc++.so.6` 被 Bazel 的 clang 链接器使用，而后者需要更新版本的 C++ 标准库符号。

### 5.4 修复方案

#### 方案 A：隔离 LD_LIBRARY_PATH（推荐用于单次构建）

在运行 Bazel 时，清除 `LD_LIBRARY_PATH` 环境变量：

```bash
cd /home/huzibo/large2/xls

# 清除 LD_LIBRARY_PATH，保持其他环境不变
env -u LD_LIBRARY_PATH "$HOME/bin/bazel" build -c opt \
  //xls/dslx:interpreter_main \
  //xls/dslx/ir_convert:ir_converter_main \
  //xls/tools:opt_main \
  //xls/tools:codegen_main
```

#### 方案 B：同时隔离 PATH 中的 Xilinx 路径（更稳定）

如果仍有问题，进一步清理 PATH：

```bash
cd /home/huzibo/large2/xls

env -u LD_LIBRARY_PATH \
  PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$HOME/bin:/opt/anaconda3/condabin:/opt/anaconda3/bin" \
  "$HOME/bin/bazel" build -c opt \
  //xls/dslx:interpreter_main \
  //xls/dslx/ir_convert:ir_converter_main \
  //xls/tools:opt_main \
  //xls/tools:codegen_main
```

#### 方案 C：仓库补丁

顶层 `BUILD` 文件对开发依赖 `hedron_compile_commands` 有强制引用，导致 `--ignore_dev_dependency` 失效。已应用最小补丁：

- 移除 `load("@hedron_compile_commands//:refresh_compile_commands.bzl", "refresh_compile_commands")`
- 移除 `refresh_compile_commands` target 定义

这不影响 XLS 编译器功能，仅移除了开发工具链生成功能。

### 5.5 代理环境配置（可选但推荐）

如果你想让终端中的 curl、git、pip、bazel 显式走 Clash 代理，在当前 shell 中设置：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=socks5://127.0.0.1:7891
export no_proxy=127.0.0.1,localhost,.local
```

或添加到 `~/.bashrc` / `~/.zshrc` 使其持久化。

**验证代理生效**：

```bash
curl -I https://github.com
git ls-remote https://github.com/google/xls.git HEAD
```

### 5.6 可直接运行的修复脚本

创建 `xls-build-clean.sh`，自动隔离环境污染后构建：

```bash
#!/bin/bash
set -e
cd /home/huzibo/large2/xls

echo "=== XLS Build with Clean Environment ==="
echo "Clearing Xilinx/CUDA library path pollution..."

env -u LD_LIBRARY_PATH \
  PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$HOME/bin:/opt/anaconda3/condabin:/opt/anaconda3/bin" \
  "$HOME/bin/bazel" build --ignore_dev_dependency -c opt \
  //xls/dslx:interpreter_main \
  //xls/dslx/ir_convert:ir_converter_main \
  //xls/tools:opt_main \
  //xls/tools:codegen_main

echo "✓ Build completed successfully!"
echo "Tools located at:"
ls -1 ./bazel-bin/xls/dslx/interpreter_main \
      ./bazel-bin/xls/dslx/ir_convert/ir_converter_main \
      ./bazel-bin/xls/tools/opt_main \
      ./bazel-bin/xls/tools/codegen_main
```

使用方式：

```bash
chmod +x xls-build-clean.sh
./xls-build-clean.sh
```

### 5.7 诊断经验总结

1. **ping 不是网络排查的可靠指标**：ICMP 走不同路由，可能被独立屏蔽。
2. **优先检查 HTTP/HTTPS 和 git 访问**：这是实际工具依赖的协议。
3. **环境变量污染隐蔽性强**：LD_LIBRARY_PATH 中旧库版本会导致链接器崩溃，需要显式诊断。
4. **代理配置不是必需的**：如果系统已有 TUN 或路由策略，curl/git 可能已经走代理，但仍需显式设置来确保一致性。

### 5.8 两条並行方案建议

你当前有两条可用路径：

- **conda 预编译包**：已验证可用，快速上手，但无法调整编译器源码。
- **源码 Bazel 构建**：需要隔离环境污染（上述步骤），可完全定制，适合开发编译器本体。

建议：
- 先用 conda 方案快速验证 DSLX 设计和 RTL 生成。
- 如需修改编译器源码，使用上述"方案 B"环境隔离后的源码构建。


### 2026-03-17尝试


已开始并完成一轮真实构建尝试，结果如下：

1. 源码构建尝试结果

* 已确认本机基础工具齐全：gcc/g++/zip/unzip/python/java 都有。
* Bazel 7.4.1 可用（用户目录）。
* 但源码构建失败在外部依赖下载阶段，根因是到 github.com:443 超时。
* 典型失败点：
  * hedron\_compile\_commands 拉取失败
  * rules\_python 压缩包下载超时

2. 已打通“可运行”方案（无 sudo）
   我改走 conda 预编译包路径，已成功安装并验证 XLS 能跑：

* 已创建环境：xls-run
* 已安装包：litex-hub 渠道的 xls
* 已成功跑通最小链路：
  * interpreter\_main 运行 DSLX 单测通过
  * ir\_converter\_main 生成 IR
  * opt\_main 生成优化后 IR
  * codegen\_main 生成 Verilog
* 已看到 Verilog 输出文件并预览内容，说明工具链可用。

3. 你现在可直接复用的命令 🚀
   conda activate xls-run
   interpreter\_main [simple\_add.x](vscode-file://vscode-app/h:/ProgramFiles/VSCode/ce099c1ed2/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
   ir\_converter\_main --top=add [simple\_add.x](vscode-file://vscode-app/h:/ProgramFiles/VSCode/ce099c1ed2/resources/app/out/vs/code/electron-browser/workbench/workbench.html) > [simple\_add.ir](vscode-file://vscode-app/h:/ProgramFiles/VSCode/ce099c1ed2/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
   opt\_main [simple\_add.ir](vscode-file://vscode-app/h:/ProgramFiles/VSCode/ce099c1ed2/resources/app/out/vs/code/electron-browser/workbench/workbench.html) > [simple\_add.opt.ir](vscode-file://vscode-app/h:/ProgramFiles/VSCode/ce099c1ed2/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
   codegen\_main --pipeline\_stages=1 --delay\_model=unit [simple\_add.opt.ir](vscode-file://vscode-app/h:/ProgramFiles/VSCode/ce099c1ed2/resources/app/out/vs/code/electron-browser/workbench/workbench.html) > [simple\_add.v](vscode-file://vscode-app/h:/ProgramFiles/VSCode/ce099c1ed2/resources/app/out/vs/code/electron-browser/workbench/workbench.html)
4. 下一步建议
5. 先用当前 conda 方案开展开发和验证（已经可用）。
6. 如果你必须源码构建，我可以继续帮你做“代理/镜像版 Bazel 构建配置”（例如配置 HTTPS 代理或内网镜像），把 GitHub 依赖下载问题绕过去。
