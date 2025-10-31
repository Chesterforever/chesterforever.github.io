---
title: 从零学习写makefile(一)background
date: 2025-10-01 00:42:24
tags: [学习记录]
index_img: /img/avatar.png

---

# 从零学习写makefile(一)background

## 一.为什么要写makefile？

这个问题的答案可以先去看我的另外一篇博客（[一篇文章讲明白代码编译，链接，烧录全过程（尽可能涵盖不同芯片和不同编译器） - 技术挖掘师](https://chesterforever.github.io/2025/08/03/代码编译，链接，烧录全过程/)），于是我们知道makefile其实是为代码如何编译和链接制订规则的。

正常来讲，比如用keil IDE，你点击build执行代码编译时，keil实际上就会动态生成一个makefile在临时目录中，然后调用工具链来执行这个makefile,然后得到.o文件交给链接器（.sct）处理。keil提供了一体化的开发环境，让初学者能很快地上手嵌入式开发，但也为进阶学习造成了障碍。

- **编译器**：ARMCC/ARMCLANG（专为 ARM 优化）
- **链接器**：自动处理分散加载（`.sct`文件）
- **调试器**：无缝支持硬件调试（无需额外配置 GDB）
- **芯片支持包**：内置寄存器定义、启动文件、驱动库

而vscode，它是一种代码编辑器+插件平台，无内置编译器和调试器，所以它适用于通用开发，所有工具链，构建系统，调试器都需自己配置

下面是我让ai生成的keil的makefile结构，仅作参考

```c
# 示例：Keil 自动生成的 Makefile 结构
PROJECT_NAME = MyProject
TARGET = Project.axf
DEVICE = STM32F103ZE
TOOLCHAIN = ARMCC

# 工具定义
CC = armcc
AS = armasm
LD = armlink
OBJCOPY = fromelf

# 包含路径
INCLUDES = -I./Inc -I./Drivers/STM32F1xx/Include -I./Drivers/CMSIS/Include

# 编译选项
CFLAGS = --c99 --cpu=Cortex-M3 -g -O1 -DUSE_STDPERIPH_DRIVER -DSTM32F10X_HD
ASFLAGS = --cpu=Cortex-M3 --pd "__MICROLIB SETA 1"
LDFLAGS = --cpu=Cortex-M3 --library_type=microlib --scatter ".\Objects\Project.sct"

# 源文件列表
C_SOURCES = \
    Src/main.c \
    Src/stm32f10x_it.c \
    Drivers/STM32F1xx/Src/stm32f10x_gpio.c \
    Drivers/STM32F1xx/Src/stm32f10x_rcc.c

ASM_SOURCES = \
    Startup/startup_stm32f10x_hd.s

# 目标文件列表
OBJECTS = $(C_SOURCES:.c=.o) $(ASM_SOURCES:.s=.o)

# 默认目标
all: $(TARGET)

# 链接目标
$(TARGET): $(OBJECTS)
    $(LD) $(LDFLAGS) --map --list ./Listings/Project.map -o $@ $^

# 编译 C 文件
%.o: %.c
    $(CC) $(CFLAGS) $(INCLUDES) -o $@ $<

# 编译汇编文件
%.o: %.s
    $(AS) $(ASFLAGS) -o $@ $<

clean:
    del $(OBJECTS) $(TARGET)
```

在了解上述这些知识背景后，通过比对就知道了为什么要自己写makefile，原因无非是以下几点：

1. 灵活性与控制力

   Keil 提供了一个图形化界面（GUI），你通过点击复选框和填写输入框来配置项目。这很方便，但同时也是一种“黑箱”操作。

   Makefile 是“白箱“：你可以精确地控制编译过程的每一个步骤：用了哪个编译器（`arm-none-eabi-gcc`）、哪些编译标志（`-O2`, `-mcpu=cortex-m4`）、链接顺序、链接脚本的位置、预处理器的定义等。当遇到极其棘手的链接错误或优化问题时，这种细粒度的控制是无可替代的。

   超越 IDE 的限制：Keil 的配置选项是有限的。如果你需要使用一个非常冷门的编译器特性，或者想实现一些复杂的构建前/后步骤（比如自动生成版本号、加密固件、调用自定义的代码生成工具），在 Makefile 中可以轻松实现，而在 Keil 中可能非常困难甚至不可能。

2. 自动化与持续集成/持续部署

   命令行驱动：Makefile 可以通过简单的 `make`命令在终端中运行。这意味着它可以被轻松地集成到各种自动化流程中。

   CI/CD 流水线：在 Jenkins, GitLab CI, GitHub Actions 等自动化平台上，你可以直接调用 `make`命令来编译整个项目。服务器上不需要安装和配置庞大的 Keil IDE，只需要安装轻量级的编译器工具链（如 GNU Arm Embedded Toolchain）即可。这实现了快速、可重复的自动化构建、测试和发布。

   夜间构建：可以设置服务器在每晚自动拉取最新代码，用 Makefile 编译，并报告成功与否。

3. 跨平台与团队协作

   操作系统无关性：Keil 主要面向 Windows。而 GNU Make 和 GCC 工具链是跨平台的，可以在 Windows（通过 MSYS2、WSL）、Linux 和 macOS 上完美运行。这对于使用不同操作系统的开发团队（尤其是在Linux上做开发的团队）至关重要。

   统一的构建方式：团队中每个成员，无论使用什么电脑，都可以用同一套 Makefile 获得完全一致的构建结果，避免了因 IDE 配置差异导致的问题。

   与版本控制系统完美契合：Makefile 是纯文本文件，可以很好地被 Git 等版本控制系统管理。你可以清晰地看到构建配置的每一次变更。而 Keil 的 `.uvprojx`项目文件是 XML格式，虽然也能版本控制，但可读性和可维护性远不如 Makefile。

4. 可维护性与可复用性

   模块化：一个编写良好的 Makefile 可以非常模块化，通过变量和包含其他 Makefile 片段来管理大型项目，使得代码结构清晰。

   知识复用：Makefile 的知识是通用的。你为 ARM 处理器写的 Makefile，其核心思想（目标、依赖、规则）可以应用到编译 Linux C++ 程序、Go 语言项目甚至文档生成上。而熟练使用 Keil 的技能，其通用性相对较低。

   依赖管理：对于复杂的项目，尤其是需要集成多个第三方库（如 FreeRTOS, LWIP, FatFs）时，用 Makefile 来管理这些依赖关系可能比在 Keil 中手动添加文件更加清晰和自动化。

5. 成本与开源文化

   Keil 是商业软件：虽然有针对芯片厂商的免费版本（有代码大小限制），但完整版价格昂贵。

   GNU 工具链是开源免费的：GCC + Make 的组合是完全免费且开源的，这符合许多项目和公司的预算及技术选型策略。整个开源的嵌入式生态（如 Zephyr RTOS, ESP-IDF 等）都基于 Makefile/CMake 这类构建系统。



## 二.Makefile，Make，CMake的关系

CMake (CMakeLists.txt) → 生成 Makefile → 由 Make 执行编译

**CMake** 是一个 跨平台的高级构建系统生成器，它不直接编译代码，而是：

1. 读取 `CMakeLists.txt`（项目配置文件）
2. 生成 平台相关的构建文件**（如 `Makefile`、`Visual Studio.sln`、`Ninja`）
3. 调用底层工具（`make`、`msbuild`、`ninja`）执行编译

**Make** 是一个命令行工具，用于解析Makefile并执行构建任务。它根据文件修改时间（`timestamp`）决定哪些文件需要重新编译。

**Makefile** 是一个纯文本文件，定义了如何编译、链接和构建项目的规则。它描述了源文件（`.c`、`.cpp`）如何编译成目标文件（`.o`），目标文件如何链接成可执行文件（`.elf`、`.exe`），依赖关系（哪些文件需要重新编译）清理规则（`make clean`删除临时文件）

由于这篇博客主要以学习记录为主，所以暂时不会接触CMake，等我学完makefile后如果有需要再来看看cmake相关的知识点。



