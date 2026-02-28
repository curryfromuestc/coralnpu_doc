# 第十七章 开发指南

本章为 CoralNPU 项目的开发者提供详细的开发指南，包括贡献流程、编码规范、Git 工作流、代码审查和测试要求。

## 17.1 贡献指南

### 17.1.1 如何贡献代码

我们欢迎并感谢您对 CoralNPU 项目的贡献。在开始贡献之前，请遵循以下步骤：

#### 贡献者许可协议（CLA）

对本项目的贡献必须附带贡献者许可协议（Contributor License Agreement）。您（或您的雇主）保留对您贡献内容的版权；CLA 只是授予我们使用和重新分发您的贡献作为项目一部分的权限。

- 访问 <https://cla.developers.google.com/> 查看您当前的协议或签署新协议
- 通常您只需要提交一次 CLA，即使是为不同的项目

#### 社区准则

本项目遵循 [Google 开源社区准则](https://opensource.google/conduct/)。

### 17.1.2 贡献流程

1. **Fork 和 Clone 仓库**
   ```bash
   git clone https://github.com/your-username/coralnpu.git
   cd coralnpu
   ```

2. **创建功能分支**
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **进行开发**
   - 编写代码
   - 添加测试
   - 确保代码符合编码规范
   - 运行测试确保通过

4. **提交更改**
   ```bash
   git add <files>
   git commit -m "feat: add new feature description"
   ```

5. **推送到远程仓库**
   ```bash
   git push origin feature/your-feature-name
   ```

6. **创建 Pull Request**
   - 在 GitHub 上创建 Pull Request
   - 填写详细的 PR 描述
   - 等待代码审查

### 17.1.3 代码审查

所有提交（包括项目成员的提交）都需要经过审查。我们使用 Gerrit 代码审查系统进行此过程。

- 参考 [Gerrit 用户指南](https://gerrit-documentation.storage.googleapis.com/Documentation/3.8.2/intro-user.html) 了解更多信息
- 代码审查者会检查代码质量、设计、测试覆盖率等方面
- 根据审查意见进行修改，直到获得批准

## 17.2 编码风格

### 17.2.1 Scala/Chisel 编码风格

CoralNPU 使用 Scala 3 和 Chisel 进行硬件描述。编码风格配置在 `.scalafmt.conf` 文件中。

#### 基本规范

- **最大列宽**: 80 字符
- **Scala 版本**: Scala 3
- **格式化工具**: scalafmt 3.6.1

#### 代码示例

```scala
package coralnpu

import chisel3._
import chisel3.util._

class ExampleModule extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(32.W))
    val out = Output(UInt(32.W))
  })

  // 使用有意义的变量名
  val processedData = Wire(UInt(32.W))
  processedData := io.in + 1.U

  io.out := processedData
}
```

#### 命名规范

- **模块名**: 使用 PascalCase（如 `ExampleModule`）
- **信号名**: 使用 camelCase（如 `processedData`）
- **常量**: 使用 camelCase 或 UPPER_CASE
- **包名**: 使用小写（如 `coralnpu`）

#### 注释规范

```scala
// Copyright 2024 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
```

每个文件都必须包含 Apache 2.0 许可证头。

### 17.2.2 C/C++ 编码风格

C/C++ 代码使用 clang-format 进行格式化。

#### 基本规范

- 使用 clang-format 自动格式化
- 支持的文件扩展名: `.c`, `.h`, `.cc`, `.cpp`
- 格式化命令:
  ```bash
  clang-format -i --style=file your_file.cc
  ```

#### 代码示例

```cpp
// Copyright 2025 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

#include <string.h>

// 使用有意义的变量名
float input1[8] __attribute__((section(".data")));
float input2[8] __attribute__((section(".data")));
float output[8] __attribute__((section(".data")));

int main() {
  // 循环处理数据
  for (int i = 0; i < 8; i++) {
    output[i] = input1[i] + input2[i];
  }
  return 0;
}
```

#### 命名规范

- **函数名**: 使用 snake_case（如 `process_data`）
- **变量名**: 使用 snake_case（如 `input_data`）
- **常量**: 使用 UPPER_CASE（如 `MAX_SIZE`）
- **类型名**: 使用 PascalCase（如 `DataProcessor`）

### 17.2.3 Python 编码风格

Python 代码使用 yapf 和 pylint3 进行格式化和检查。

#### 基本规范

- 遵循 PEP 8 风格指南
- 使用 yapf 进行自动格式化
- 使用 pylint3 进行代码质量检查

#### 格式化命令

```bash
# 使用 yapf 格式化
yapf -i your_file.py

# 使用 pylint 检查
pylint3 your_file.py
```

#### 代码示例

```python
"""模块文档字符串。

这个模块提供了测试工具函数。
"""

import os
import sys


class TestFixture:
    """测试夹具类。
    
    Attributes:
        name: 测试名称
        config: 配置字典
    """

    def __init__(self, name, config=None):
        """初始化测试夹具。
        
        Args:
            name: 测试名称
            config: 可选的配置字典
        """
        self.name = name
        self.config = config or {}

    def run_test(self):
        """运行测试。
        
        Returns:
            测试结果布尔值
        """
        # 实现测试逻辑
        return True
```

#### 命名规范

- **模块名**: 使用 snake_case（如 `test_utils`）
- **类名**: 使用 PascalCase（如 `TestFixture`）
- **函数名**: 使用 snake_case（如 `run_test`）
- **变量名**: 使用 snake_case（如 `test_name`）
- **常量**: 使用 UPPER_CASE（如 `MAX_RETRIES`）

### 17.2.4 BUILD.bazel 文件风格

BUILD.bazel 文件使用 buildifier 进行格式化。

#### 格式化命令

```bash
buildifier -mode=fix BUILD.bazel
```

## 17.3 Git 工作流

### 17.3.1 分支策略

CoralNPU 项目使用以下分支策略：

#### 主分支

- **main**: 主开发分支，包含最新的稳定代码
- 所有功能开发都从 main 分支创建

#### 功能分支

- 命名格式: `feature/<feature-name>`
- 示例: `feature/add-i2c-master`
- 用于开发新功能

#### 修复分支

- 命名格式: `fix/<bug-description>`
- 示例: `fix/uart-timeout-issue`
- 用于修复 bug

#### 测试分支

- 命名格式: `test/<test-description>`
- 示例: `test/add-spi-tests`
- 用于添加或改进测试

### 17.3.2 提交信息规范

提交信息应该清晰、简洁，遵循以下格式：

#### 提交信息格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

#### 类型（type）

- **feat**: 新功能
- **fix**: 修复 bug
- **docs**: 文档更新
- **style**: 代码格式化（不影响代码功能）
- **refactor**: 代码重构
- **test**: 添加或修改测试
- **chore**: 构建过程或辅助工具的变动

#### 范围（scope）

指明提交影响的范围，例如：
- `ip`: IP 核心
- `bus`: 总线相关
- `sim`: 仿真相关
- `doc`: 文档
- `build`: 构建系统

#### 示例

```
feat(ip): Integrate I2C Master peripheral into the SoC

Add I2C Master controller with TileLink-UL interface.
Includes register map, interrupt handling, and clock stretching support.

Fixes #123
```

```
fix(bus): Fix UART timeout handling

The UART controller was not properly handling timeout conditions,
causing data loss in high-load scenarios.
```

```
test(sim): Update test utilities and simulation infrastructure

Improve test fixture setup and add helper functions for
common test scenarios.
```

### 17.3.3 Pull Request 流程

#### 创建 Pull Request

1. **确保代码质量**
   - 运行所有测试
   - 确保代码格式化正确
   - 检查没有 lint 错误

2. **编写 PR 描述**
   - 清晰描述更改的内容和原因
   - 列出相关的 issue 编号
   - 说明测试方法

3. **PR 标题格式**
   ```
   <type>(<scope>): <brief description>
   ```

#### PR 描述模板

```markdown
## 描述
简要描述此 PR 的目的和内容。

## 更改类型
- [ ] 新功能
- [ ] Bug 修复
- [ ] 文档更新
- [ ] 代码重构
- [ ] 测试改进

## 测试
描述如何测试这些更改：
- 运行的测试命令
- 测试结果

## 相关 Issue
Fixes #<issue-number>

## 检查清单
- [ ] 代码已格式化
- [ ] 添加了测试
- [ ] 测试通过
- [ ] 文档已更新
```

## 17.4 审查流程

### 17.4.1 代码审查要点

代码审查者应该关注以下方面：

#### 功能正确性

- 代码是否实现了预期的功能
- 是否有逻辑错误
- 边界条件是否处理正确

#### 代码质量

- 代码是否清晰易读
- 变量和函数命名是否合理
- 是否有重复代码
- 是否遵循项目的编码规范

#### 设计

- 设计是否合理
- 是否有更好的实现方式
- 是否考虑了可扩展性

#### 测试

- 是否有足够的测试覆盖
- 测试是否有意义
- 是否测试了边界条件

#### 文档

- 是否有必要的注释
- 公共 API 是否有文档
- 复杂逻辑是否有说明

### 17.4.2 如何进行代码审查

#### 审查步骤

1. **理解上下文**
   - 阅读 PR 描述
   - 了解相关的 issue
   - 理解更改的目的

2. **检查代码**
   - 逐行阅读代码
   - 关注上述审查要点
   - 运行代码和测试

3. **提供反馈**
   - 使用建设性的语言
   - 具体指出问题所在
   - 提供改进建议

4. **批准或请求更改**
   - 如果代码质量良好，批准 PR
   - 如果需要修改，清楚说明原因

#### 审查评论示例

**好的评论**:
```
建议将这个函数拆分成更小的函数，以提高可读性。
例如，可以将数据验证逻辑提取到单独的函数中。
```

**不好的评论**:
```
这段代码写得不好。
```

## 17.5 测试指南

### 17.5.1 如何编写测试

CoralNPU 项目使用多种测试框架：

#### Chisel 测试

使用 ChiselTest 框架编写硬件模块测试。

**测试示例**:

```scala
package common

import chisel3._
import chisel3.simulator.scalatest.ChiselSim
import chisel3.util._
import org.scalatest.freespec.AnyFreeSpec
import scala.util.Random

class ExampleModuleTest extends AnyFreeSpec with ChiselSim {
  "ExampleModule should process data correctly" in {
    simulate(new ExampleModule) { dut =>
      // 设置输入
      dut.io.in.poke(42.U)
      dut.clock.step()
      
      // 检查输出
      dut.io.out.expect(43.U)
    }
  }
  
  "ExampleModule should handle edge cases" in {
    simulate(new ExampleModule) { dut =>
      // 测试边界条件
      dut.io.in.poke(0.U)
      dut.clock.step()
      dut.io.out.expect(1.U)
      
      dut.io.in.poke(0xFFFFFFFF.U)
      dut.clock.step()
      dut.io.out.expect(0.U)  // 溢出
    }
  }
}
```

#### Cocotb 测试

使用 Cocotb 进行 Verilog/SystemVerilog 级别的测试。

**测试示例**:

```python
import cocotb
from cocotb.triggers import RisingEdge, Timer
from cocotb.clock import Clock

@cocotb.test()
async def test_basic_operation(dut):
    """测试基本操作。"""
    # 启动时钟
    clock = Clock(dut.clk, 10, units="ns")
    cocotb.start_soon(clock.start())
    
    # 复位
    dut.rst.value = 1
    await RisingEdge(dut.clk)
    dut.rst.value = 0
    await RisingEdge(dut.clk)
    
    # 测试逻辑
    dut.in_data.value = 42
    await RisingEdge(dut.clk)
    
    assert dut.out_data.value == 43, \
        f"Expected 43, got {dut.out_data.value}"
```

#### C/C++ 测试

为软件库编写单元测试。

```cpp
#include <gtest/gtest.h>
#include "your_module.h"

TEST(YourModuleTest, BasicOperation) {
  YourModule module;
  EXPECT_EQ(module.process(42), 43);
}

TEST(YourModuleTest, EdgeCases) {
  YourModule module;
  EXPECT_EQ(module.process(0), 1);
  EXPECT_EQ(module.process(-1), 0);
}
```

### 17.5.2 测试覆盖率要求

#### 覆盖率目标

- **代码覆盖率**: 目标 > 80%
- **分支覆盖率**: 目标 > 70%
- **关键路径**: 必须 100% 覆盖

#### 检查覆盖率

```bash
# 运行测试并生成覆盖率报告
bazel coverage //tests/...

# 查看覆盖率报告
genhtml bazel-out/_coverage/_coverage_report.dat -o coverage_html
```

#### 测试要求

1. **单元测试**
   - 每个模块都应该有单元测试
   - 测试所有公共接口
   - 测试边界条件和错误情况

2. **集成测试**
   - 测试模块之间的交互
   - 测试完整的数据流

3. **回归测试**
   - 修复 bug 后添加回归测试
   - 确保 bug 不会再次出现

### 17.5.3 CI/CD 流程

CoralNPU 使用 GitHub Actions 进行持续集成。

#### CI 工作流

CI 工作流定义在 `.github/workflows/main.yml` 文件中。

**触发条件**:
- Push 到 main 分支
- 创建 Pull Request
- 创建 Tag
- 手动触发

**CI 步骤**:

1. **检出代码**
   ```yaml
   - uses: actions/checkout@v5
     with:
       fetch-depth: 1
   ```

2. **设置 Bazel**
   ```yaml
   - uses: bazel-contrib/setup-bazel@0.15.0
     with:
       bazelisk-cache: true
       disk-cache: ${{ github.workflow }}
       repository-cache: true
   ```

3. **构建 RTL**
   ```yaml
   - name: build
     run: bazel build //hdl/chisel/src/coralnpu:rvv_core_mini_axi_cc_library_emit_verilog
   ```

4. **上传构建产物**
   ```yaml
   - uses: actions/upload-artifact@v4
     with:
       name: RvvCoreMiniAxi
       path: |
         bazel-bin/hdl/chisel/src/coralnpu/RvvCoreMiniAxi.sv
         bazel-bin/hdl/chisel/src/coralnpu/RvvCoreMiniAxi.zip
   ```

5. **发布 Release**（仅在创建 Tag 时）
   ```yaml
   - uses: softprops/action-gh-release@v2
     if: github.ref_type == 'tag'
     with:
       files: |
         bazel-bin/hdl/chisel/src/coralnpu/RvvCoreMiniAxi.sv
         bazel-bin/hdl/chisel/src/coralnpu/RvvCoreMiniAxi.zip
   ```

#### 本地运行 CI 检查

在提交代码之前，建议在本地运行以下检查：

```bash
# 运行格式化检查
bazel run //:scalafmt_check
buildifier -mode=check $(find . -name BUILD.bazel)

# 运行 lint 检查
pylint3 $(find . -name "*.py")
clang-format --dry-run --Werror $(find . -name "*.cc" -o -name "*.h")

# 运行测试
bazel test //tests/...

# 构建 RTL
bazel build //hdl/chisel/src/coralnpu:rvv_core_mini_axi_cc_library_emit_verilog
```

#### Pre-upload Hooks

项目配置了 pre-upload hooks（在 `PREUPLOAD.cfg` 中定义），在上传代码前自动运行检查。

**配置的 Hooks**:

1. **pylint3**: Python 代码质量检查
2. **cpplint**: C++ 代码风格检查
3. **clang_format**: C/C++ 代码格式化检查
4. **buildifier**: BUILD.bazel 文件格式化检查
5. **yapf-diff**: Python 代码格式化检查

**运行 Hooks**:

```bash
# 手动运行 pre-upload hooks
repo upload --no-verify  # 跳过 hooks（不推荐）
repo upload              # 运行所有 hooks
```

## 17.6 开发环境设置

### 17.6.1 系统要求

- **操作系统**: Linux (推荐 Ubuntu 22.04)
- **Bazel**: 7.4.1
- **Python**: 3.9-3.12
- **SRecord**: 用于二进制文件转换

### 17.6.2 安装依赖

```bash
# 安装 Bazel
sudo apt install apt-transport-https curl gnupg
curl -fsSL https://bazel.build/bazel-release.pub.gpg | gpg --dearmor > bazel.gpg
sudo mv bazel.gpg /etc/apt/trusted.gpg.d/
echo "deb [arch=amd64] https://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
sudo apt update && sudo apt install bazel-7.4.1

# 安装 Python 依赖
pip install -r requirements.txt

# 安装 SRecord
sudo apt install srecord

# 安装代码格式化工具
sudo apt install clang-format
pip install yapf pylint
```

### 17.6.3 快速开始

```bash
# 克隆仓库
git clone https://github.com/google/coralnpu.git
cd coralnpu

# 运行测试套件
bazel run //tests/cocotb:core_mini_axi_sim_cocotb

# 构建示例程序
bazel build //examples:coralnpu_v2_hello_world_add_floats

# 构建仿真器
bazel build //tests/verilator_sim:core_mini_axi_sim

# 运行仿真
bazel-bin/tests/verilator_sim/core_mini_axi_sim \
  --binary bazel-out/k8-fastbuild-ST-dd8dc713f32d/bin/examples/coralnpu_v2_hello_world_add_floats.elf
```

## 17.7 常见问题

### 17.7.1 构建问题

**问题**: Bazel 构建失败

**解决方案**:
```bash
# 清理构建缓存
bazel clean --expunge

# 重新构建
bazel build //...
```

**问题**: Python 版本不兼容

**解决方案**:
```bash
# 使用 pyenv 管理 Python 版本
pyenv install 3.11
pyenv local 3.11
```

### 17.7.2 测试问题

**问题**: 测试超时

**解决方案**:
```bash
# 增加测试超时时间
bazel test --test_timeout=600 //tests/...
```

**问题**: 仿真器运行失败

**解决方案**:
```bash
# 检查依赖是否安装
which verilator
which srecord

# 重新构建仿真器
bazel clean
bazel build //tests/verilator_sim:core_mini_axi_sim
```

### 17.7.3 代码格式化问题

**问题**: clang-format 版本不匹配

**解决方案**:
```bash
# 安装指定版本的 clang-format
sudo apt install clang-format-14
sudo update-alternatives --install /usr/bin/clang-format clang-format /usr/bin/clang-format-14 100
```

## 17.8 资源链接

- **项目主页**: https://github.com/google/coralnpu
- **文档**: https://developers.google.com/coral
- **问题跟踪**: https://github.com/google/coralnpu/issues
- **Gerrit 代码审查**: https://gerrit-documentation.storage.googleapis.com/
- **Chisel 文档**: https://www.chisel-lang.org/
- **Bazel 文档**: https://bazel.build/

## 17.9 总结

本章介绍了 CoralNPU 项目的开发指南，包括：

1. **贡献流程**: 从 Fork 仓库到创建 Pull Request 的完整流程
2. **编码规范**: Scala/Chisel、C/C++、Python 的编码风格和命名规范
3. **Git 工作流**: 分支策略、提交信息规范和 PR 流程
4. **代码审查**: 审查要点和如何进行有效的代码审查
5. **测试指南**: 如何编写测试、测试覆盖率要求和 CI/CD 流程

遵循这些指南将帮助您更好地为 CoralNPU 项目做出贡献，并确保代码质量和项目的可维护性。
