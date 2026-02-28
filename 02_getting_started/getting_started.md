# 快速开始

本章将指导你完成 CoralNPU 开发环境的搭建，并运行第一个程序。

## 系统要求

在开始之前，请确保你的系统满足以下要求：

### 操作系统



    - Linux（推荐 Ubuntu 20.04 或更高版本）
    - macOS（部分功能可能受限）
    - Windows（需要 WSL2）



### 必需工具

\paragraph{Bazel}



    - **版本要求**：Bazel 7.4.1
    - **说明**：Bazel 是 Google 开发的构建工具，CoralNPU 项目使用它来管理编译过程



\paragraph{Python}



    - **版本要求**：Python 3.9 - 3.12
    - **说明**：用于运行构建脚本和测试工具（Python 3.13 支持正在开发中）



\paragraph{SRecord}



    - **说明**：用于处理二进制文件格式转换的工具
    - **官网**：\url{https://srecord.sourceforge.net/}



\paragraph{其他工具}



    - Git：用于克隆代码仓库
    - GCC/Clang：C/C++ 编译器
    - Make：构建工具



### 可选工具

\paragraph{VCS 仿真器}

如果你需要使用 VCS 进行仿真，需要：



    - Synopsys VCS 仿真器
    - 有效的 VCS 许可证



### 硬件要求



    - **内存**：建议至少 8GB RAM（16GB 或更多更佳）
    - **磁盘空间**：至少 10GB 可用空间
    - **CPU**：多核处理器（构建过程会使用多线程）



## 安装指南

### 安装 Bazel

\paragraph{Linux 系统}

使用 Bazelisk（推荐方式，可以自动管理 Bazel 版本）：

```
# 下载 Bazelisk
sudo wget -O /usr/local/bin/bazel \
  https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-amd64

# 添加执行权限
sudo chmod +x /usr/local/bin/bazel

# 验证安装
bazel --version
```

或者直接安装 Bazel 7.4.1：

```
# 下载 Bazel 安装脚本
wget https://github.com/bazelbuild/bazel/releases/download/7.4.1/\
bazel-7.4.1-installer-linux-x86_64.sh

# 运行安装脚本
chmod +x bazel-7.4.1-installer-linux-x86_64.sh
./bazel-7.4.1-installer-linux-x86_64.sh --user

# 将 Bazel 添加到 PATH（添加到 ~/.bashrc 或 ~/.zshrc）
export PATH="$PATH:$HOME/bin"

# 重新加载配置
source ~/.bashrc
```

\paragraph{macOS 系统}

使用 Homebrew：

```
# 安装 Bazelisk
brew install bazelisk

# 或者直接安装 Bazel
brew install bazel
```

### 安装 Python

\paragraph{Linux 系统}

```
# Ubuntu/Debian
sudo apt update
sudo apt install python3.11 python3.11-dev python3-pip

# 验证安装
python3.11 --version
```

\paragraph{macOS 系统}

```
# 使用 Homebrew
brew install python@3.11

# 验证安装
python3.11 --version
```

### 安装 SRecord

\paragraph{Linux 系统}

```
# Ubuntu/Debian
sudo apt update
sudo apt install srecord

# 验证安装
srec_cat --version
```

\paragraph{macOS 系统}

```
# 使用 Homebrew
brew install srecord

# 验证安装
srec_cat --version
```

### 安装其他依赖

```
# Ubuntu/Debian 系统
sudo apt update
sudo apt install -y \
    git \
    build-essential \
    gcc \
    g++ \
    make \
    wget \
    curl

# macOS 系统（需要先安装 Xcode Command Line Tools）
xcode-select --install
```

### 克隆 CoralNPU 代码仓库

```
# 克隆代码
git clone https://github.com/google/coralnpu.git
cd coralnpu

# 查看项目结构
ls -la
```

### 配置 VCS 仿真器（可选）

如果你有 VCS 许可证并想使用 VCS 进行仿真，需要设置以下环境变量：

```
# 设置 VCS_HOME（替换为你的 VCS 安装路径）
export VCS_HOME=/path/to/your/vcs/home

# 设置许可证文件
export LM_LICENSE_FILE=/path/to/your/license/file

# 更新库路径和 PATH
export LD_LIBRARY_PATH="${VCS_HOME}/linux64/lib:${LD_LIBRARY_PATH}"
export PATH="${VCS_HOME}/bin:${PATH}"
```

建议将这些环境变量添加到 `\textasciitilde/.bashrc` 或 `\textasciitilde/.zshrc` 文件中，这样每次打开终端时会自动加载。

## 快速开始

现在环境已经配置好了，让我们运行第一个测试来验证安装是否成功。

### 运行测试套件

```
# 进入 CoralNPU 项目目录
cd coralnpu

# 运行核心测试（这会验证基本功能是否正常）
bazel run //tests/cocotb:core_mini_axi_sim_cocotb
```

**说明**：



    - `bazel run` 是 Bazel 的命令，用于构建并运行目标
    - `//tests/cocotb:core\_mini\_axi\_sim\_cocotb` 是测试目标的完整路径
    

        - `//` 表示从项目根目录开始
        - `tests/cocotb` 是目录路径
        - `:core\_mini\_axi\_sim\_cocotb` 是具体的测试目标名称
    




第一次运行时，Bazel 会下载所有依赖项，这可能需要一些时间。请耐心等待。

### 构建示例程序

```
# 构建一个简单的浮点加法示例
bazel build //examples:coralnpu_v2_hello_world_add_floats
```

**说明**：



    - `bazel build` 命令用于编译程序但不运行
    - 编译成功后，生成的二进制文件会保存在 `bazel-bin/` 目录下



### 构建仿真器

```
# 构建 Verilator 仿真器（不包含 RVV 支持，编译更快）
bazel build //tests/verilator_sim:core_mini_axi_sim
```

**说明**：



    - Verilator 是一个开源的硬件仿真器
    - 这个命令会构建一个可以运行 CoralNPU 程序的仿真环境
    - 编译时间可能较长（5-15 分钟，取决于你的机器性能）



### 在仿真器上运行程序

```
# 运行之前编译的示例程序
bazel-bin/tests/verilator_sim/core_mini_axi_sim \
    --binary bazel-out/k8-fastbuild-ST-dd8dc713f32d/bin/examples/\
coralnpu_v2_hello_world_add_floats.elf
```

**说明**：



    - `bazel-bin/tests/verilator\_sim/core\_mini\_axi\_sim` 是仿真器可执行文件
    - `--binary` 参数指定要运行的程序
    - `.elf` 是可执行文件格式（Executable and Linkable Format）



**注意**：`bazel-out/` 路径中的哈希值（如 `dd8dc713f32d`）可能在你的系统上不同。你可以使用以下命令找到正确的路径：

```
# 查找生成的 .elf 文件
find bazel-out -name "coralnpu_v2_hello_world_add_floats.elf"
```

如果一切正常，你应该看到程序成功运行的输出。

## 第一个程序

现在让我们编写、编译并运行一个简单的 Hello World 程序。

### 理解示例程序

首先，让我们看看 CoralNPU 的一个简单示例程序。这是一个浮点数加法程序：

```
// hello_world_add_floats.cc
#include <string.h>

// 定义三个浮点数组，存储在数据段
float input1[8] __attribute__((section(".data")));
float input2[8] __attribute__((section(".data")));
float output[8] __attribute__((section(".data")));

int main() {
  // 将 input1 和 input2 对应元素相加，结果存入 output
  for (int i = 0; i < 8; i++) {
    output[i] = input1[i] + input2[i];
  }
  return 0;
}
```

**代码解释**：



    - **头文件**：`\#include <string.h>` 引入字符串处理函数（虽然这个例子没用到）
    
    - **数组定义**：
    ```
    float input1[8] __attribute__((section(".data")));
    ```
    

        - `float input1[8]` 定义了一个包含 8 个浮点数的数组
        - `\_\_attribute\_\_((section(".data")))` 是 GCC 的扩展语法，指定变量存储在特定的内存段
        - 这里的 `.data` 段是程序的数据段，用于存储已初始化的全局变量
    

    
    - **main 函数**：
    ```
    for (int i = 0; i < 8; i++) {
        output[i] = input1[i] + input2[i];
    }
    ```
    

        - 这是一个简单的循环，将两个数组对应位置的元素相加
        - `for` 循环是 C 语言的基本语法，重复执行 8 次
    




### 创建你自己的程序

让我们创建一个稍微复杂一点的程序，计算数组元素的总和：

```
# 在 examples 目录下创建新文件
cd coralnpu/examples
```

创建文件 `my\_first\_program.cc`：

```
// my_first_program.cc
#include <string.h>

// 定义输入数组和输出变量
int numbers[10] __attribute__((section(".data"))) = 
    {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
int sum __attribute__((section(".data"))) = 0;

int main() {
  // 计算数组所有元素的总和
  for (int i = 0; i < 10; i++) {
    sum += numbers[i];  // sum = sum + numbers[i]
  }
  return 0;
}
```

**新语法说明**：



    - `int numbers[10] = \{1, 2, 3, 4, 5, 6, 7, 8, 9, 10\`} 是数组初始化语法，在定义数组的同时给它赋初值
    - `sum += numbers[i]` 是 `sum = sum + numbers[i]` 的简写形式，这是 C 语言的复合赋值运算符



### 配置构建规则

编辑 `examples/BUILD.bazel` 文件，添加新程序的构建规则：

```
# 在文件末尾添加
coralnpu_v2_binary(
    name = "coralnpu_v2_my_first_program",
    srcs = ["my_first_program.cc"],
)
```

**说明**：



    - `coralnpu\_v2\_binary` 是一个 Bazel 规则（rule），专门用于构建 CoralNPU v2 的程序
    - `name` 是构建目标的名称，你可以用这个名称来引用它
    - `srcs` 指定源代码文件列表



### 编译程序

```
# 返回项目根目录
cd ..

# 编译新程序
bazel build //examples:coralnpu_v2_my_first_program
```

### 运行程序

```
# 找到生成的 .elf 文件
find bazel-out -name "coralnpu_v2_my_first_program.elf"

# 在仿真器上运行（替换路径中的哈希值）
bazel-bin/tests/verilator_sim/core_mini_axi_sim \
    --binary bazel-out/k8-fastbuild-ST-XXXXXXXX/bin/examples/\
coralnpu_v2_my_first_program.elf
```

### 使用 RISC-V 向量扩展

CoralNPU 支持 RISC-V 向量扩展（RVV），可以进行高效的 SIMD（单指令多数据）运算。下面是一个使用向量指令的示例：

```
// rvv_add_example.cc
#include <riscv_vector.h>
#include <string.h>

int8_t input_1[1024];
int8_t input_2[1024];
int16_t output[1024];

int main() {
  // 初始化输入数组
  memset(input_1, 1, 1024);  // 将 input_1 所有元素设为 1
  memset(input_2, 6, 1024);  // 将 input_2 所有元素设为 6
  
  const int8_t* input1_ptr = &input_1[0];
  const int8_t* input2_ptr = &input_2[0];
  int16_t* output_ptr = &output[0];

  // 使用向量指令处理数据，每次处理 32 个元素
  for (int idx = 0; (idx + 31) < 1024; idx += 32) {
    // 从内存加载 32 个 int8_t 元素到向量寄存器
    vint8m4_t input_v2 = __riscv_vle8_v_i8m4(input2_ptr + idx, 32);
    vint8m4_t input_v1 = __riscv_vle8_v_i8m4(input1_ptr + idx, 32);

    // 向量加法并扩展到 int16_t（防止溢出）
    vint16m8_t temp_sum = __riscv_vwadd_vv_i16m8(input_v1, input_v2, 32);
    
    // 将结果存回内存
    __riscv_vse16_v_i16m8(output_ptr + idx, temp_sum, 32);
  }

  return 0;
}
```

**新概念说明**：



    - **RISC-V 向量扩展（RVV）**：
    

        - RISC-V 是一种开源的指令集架构（ISA）
        - 向量扩展允许一条指令同时处理多个数据（SIMD）
        - 这比普通的循环快得多，因为可以并行处理
    

    
    - **向量类型**：
    

        - `vint8m4\_t`：向量类型，存储多个 int8\_t 值
        - `vint16m8\_t`：向量类型，存储多个 int16\_t 值
        - `m4`、`m8` 表示 LMUL（向量寄存器组倍数）
    

    
    - **向量内置函数**：
    

        - `\_\_riscv\_vle8\_v\_i8m4(ptr, len)`：从内存加载向量
        

            - `vle8` = vector load element 8-bit
            - `ptr` 是内存地址
            - `len` 是要加载的元素数量
        

        
        - `\_\_riscv\_vwadd\_vv\_i16m8(v1, v2, len)`：向量加法并扩展
        

            - `vwadd` = vector widening add（加法并扩展位宽）
            - `vv` = vector-vector（两个向量相加）
            - 将两个 int8\_t 向量相加，结果是 int16\_t 向量
        

        
        - `\_\_riscv\_vse16\_v\_i16m8(ptr, vec, len)`：将向量存回内存
        

            - `vse16` = vector store element 16-bit
        

    

    
    - **memset 函数**：
    ```
    memset(input_1, 1, 1024);
    ```
    

        - `memset` 是 C 标准库函数，用于设置内存块的值
        - 参数：`memset(指针, 值, 字节数)`
        - 这里将 input\_1 数组的所有 1024 个字节都设为 1
    




## 常见问题排查

### Bazel 相关问题

\paragraph{问题：Bazel 版本不匹配}

**错误信息**：

```
ERROR: This project requires Bazel 7.4.1, but you have Bazel X.X.X
```

**解决方法**：

```
# 使用 Bazelisk 自动管理版本
sudo wget -O /usr/local/bin/bazel \
  https://github.com/bazelbuild/bazelisk/releases/latest/download/\
bazelisk-linux-amd64
sudo chmod +x /usr/local/bin/bazel
```

\paragraph{问题：Bazel 缓存问题}

**症状**：构建失败，错误信息不明确

**解决方法**：

```
# 清理 Bazel 缓存
bazel clean --expunge

# 重新构建
bazel build //examples:coralnpu_v2_hello_world_add_floats
```

\paragraph{问题：磁盘空间不足}

**错误信息**：

```
ERROR: No space left on device
```

**解决方法**：

```
# 清理 Bazel 缓存
bazel clean

# 检查磁盘空间
df -h

# 如果需要，删除旧的构建输出
rm -rf ~/.cache/bazel
```

### Python 相关问题

\paragraph{问题：Python 版本不兼容}

**错误信息**：

```
ERROR: Python version 3.X is not supported
```

**解决方法**：

```
# 安装正确的 Python 版本
sudo apt install python3.11

# 设置默认 Python 版本
sudo update-alternatives --install /usr/bin/python3 python3 \
  /usr/bin/python3.11 1
```

\paragraph{问题：缺少 Python 依赖}

**错误信息**：

```
ModuleNotFoundError: No module named 'XXX'
```

**解决方法**：

```
# 安装 pip
sudo apt install python3-pip

# 安装缺失的模块
pip3 install XXX
```

### 编译相关问题

\paragraph{问题：找不到编译器}

**错误信息**：

```
ERROR: gcc: command not found
```

**解决方法**：

```
# Ubuntu/Debian
sudo apt update
sudo apt install build-essential

# 验证安装
gcc --version
g++ --version
```

\paragraph{问题：SRecord 未安装}

**错误信息**：

```
ERROR: srec_cat: command not found
```

**解决方法**：

```
# Ubuntu/Debian
sudo apt install srecord

# macOS
brew install srecord
```

### 仿真相关问题

\paragraph{问题：仿真器构建时间过长}

**症状**：`bazel build //tests/verilator\_sim:core\_mini\_axi\_sim` 运行很久

**说明**：这是正常现象。Verilator 需要将硬件描述语言（Verilog）转换为 C++ 代码并编译，这个过程比较耗时。

**优化方法**：

```
# 使用多线程编译（假设你有 8 个 CPU 核心）
bazel build --jobs=8 //tests/verilator_sim:core_mini_axi_sim

# 或者在 ~/.bazelrc 中添加
# build --jobs=8
```

\paragraph{问题：找不到 .elf 文件}

**症状**：运行仿真器时提示找不到二进制文件

**解决方法**：

```
# 使用 find 命令查找
find bazel-out -name "*.elf"

# 或者使用 Bazel 查询
bazel cquery //examples:coralnpu_v2_hello_world_add_floats --output=files
```

\paragraph{问题：VCS 仿真失败}

**错误信息**：

```
ERROR: VCS_HOME is not set
```

**解决方法**：

```
# 设置 VCS 环境变量
export VCS_HOME=/path/to/vcs
export LM_LICENSE_FILE=/path/to/license
export LD_LIBRARY_PATH="${VCS_HOME}/linux64/lib:${LD_LIBRARY_PATH}"
export PATH="${VCS_HOME}/bin:${PATH}"

# 使用 VCS 配置构建
bazel build --config=vcs //your/target
```

### 权限相关问题

\paragraph{问题：权限被拒绝}

**错误信息**：

```
Permission denied
```

**解决方法**：

```
# 检查文件权限
ls -la

# 添加执行权限
chmod +x filename

# 如果是目录权限问题
chmod -R u+rwX directory/
```

### 网络相关问题

\paragraph{问题：下载依赖失败}

**错误信息**：

```
ERROR: Failed to download https://...
```

**解决方法**：

```
# 检查网络连接
ping google.com

# 如果在中国大陆，可能需要配置代理
export http_proxy=http://your-proxy:port
export https_proxy=http://your-proxy:port

# 或者使用镜像源（如果项目支持）
```

\paragraph{问题：下载速度慢}

**解决方法**：

```
# 使用 Bazel 的下载缓存
bazel build --repository_cache=~/.bazel_cache //your/target

# 配置 Bazel 使用更多并发下载
bazel build --experimental_repository_downloader_retries=3 //your/target
```

### 获取帮助

如果遇到无法解决的问题，可以：



    - **查看项目文档**：
    ```
    # 查看 README
    cat README.md
    
    # 查看其他文档
    ls doc/
    ```
    
    - **查看 Bazel 构建日志**：
    ```
    # 使用详细输出模式
    bazel build --verbose_failures //your/target
    ```
    
    - **搜索已知问题**：
    

        - 访问项目的 GitHub Issues 页面
        - 搜索相关错误信息
    

    
    - **提交问题报告**：
    

        - 在 GitHub 上创建新的 Issue
        - 包含完整的错误信息和系统环境信息
    




## 下一步

恭喜你完成了 CoralNPU 的快速开始！现在你已经：



    - 搭建了完整的开发环境
    - 成功编译并运行了示例程序
    - 了解了如何编写简单的 CoralNPU 程序
    - 学会了基本的问题排查方法



接下来，你可以：



    - **深入学习 CoralNPU 架构**：阅读第三章了解 CoralNPU 的硬件架构和工作原理
    - **探索更多示例**：查看 `examples/` 目录下的其他示例程序
    - **学习 RISC-V 指令集**：了解 RV32IMF 和向量扩展
    - **尝试编写更复杂的程序**：实现自己的算法和应用



祝你在 CoralNPU 的学习之旅中取得成功！
