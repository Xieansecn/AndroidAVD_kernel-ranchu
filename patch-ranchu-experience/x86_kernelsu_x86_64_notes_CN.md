# KernelSU x86_64 移植笔记

> [!CAUTION]
> AI生成

## 适用范围

本文档记录了在启用 KernelSU 的情况下，使 Android 15 6.6 x86_64 虚拟设备内核在 AVD 中成功启动所做的修改与经验总结。

构建已通过以下命令验证：

```shell
tools/bazel run --config=fast --config=stamp --lto=none \
  --action_env=KSU_VERSION=10201 \
  //common-modules/virtual-device:virtual_device_x86_64_dist \
  -- --dist_dir=virt
```

最终构建成功完成，并将产物复制到 `virt/` 目录中。

## 主要经验

### 1. x86_64 上的 KernelSU 需要间接系统调用支持标记

KernelSU 在 x86_64 上会检查 `X86_FEATURE_INDIRECT_SAFE`，若该功能不存在则会提前中止。此处使用的内核源码树未包含 KernelSU 所期望的确切向后移植补丁，因此必须添加以下内容：

- 在 `arch/x86/include/asm/cpufeatures.h` 中添加 `X86_FEATURE_INDIRECT_SAFE`
- 在 `arch/x86/entry/common.c` 中添加间接系统调用表分发机制
- 导出 `ia32_sys_call_table[]` 和 `x32_sys_call_table[]`
- 在 `arch/x86/kernel/cpu/common.c` 中添加系统调用加固的启动时开关

### 2. x32 和 ia32 表必须显式导出

仅从 `arch/x86/entry/common.c` 使用间接分发是不够的。兼容模式和 x32 的系统调用表也必须作为符号存在：

- 来自 `arch/x86/entry/syscall_32.c` 的 `ia32_sys_call_table[]`
- 来自 `arch/x86/entry/syscall_x32.c` 的 `x32_sys_call_table[]`

缺少这些符号会导致 `arch/x86/entry/common.c` 构建失败。

### 3. `syscall_hardening` 需要同时具备启动逻辑和 sysfs 暴露接口

为使新模式可见且可控，实现中添加了：

- `x86_syscall_hardening_enabled`
- `early_param("syscall_hardening", ...)`
- `/sys/devices/system/cpu/syscall_hardening`

sysfs 属性通过以下文件连接：

- `arch/x86/kernel/cpu/common.c`
- `include/linux/cpu.h`
- `drivers/base/cpu.c`

### 4. `signal.h` 需要重写为 clang 无警告版本

原始的信号集辅助宏在本构建中触发了 clang 数组越界警告。稳定的修复方法是用简单的循环实现替换宏繁重的辅助逻辑，涉及函数包括：

- `sigisemptyset`
- `sigequalsets`
- `sigorsets`
- `sigandsets`
- `sigandnsets`
- `signotset`

此举在不改变行为的前提下避免了警告。

### 5. KernelSU 自身需要两个小的构建修复

在 `common/KernelSU` 子仓库中，需要两处修改：

- 在 `kernel/hook/patch_memory.h` 中添加 `#include <linux/bug.h>`
- 在 `kernel/Kbuild` 中添加 `-Wno-array-bounds`

这些更改位于 KernelSU 子仓库补丁中，而非主内核补丁。

### 6. Bazel 传入的 `KSU_VERSION` 被 KernelSU 自身的 Kbuild 逻辑覆盖

传递 `--action_env=KSU_VERSION=10201` 并未控制最终打印的 KernelSU 版本。原因是 `common/KernelSU/kernel/Kbuild` 会检测其自身的 Git 仓库，并通过以下方式内部重新计算 `KSU_VERSION`：

```make
KSU_GIT_VERSION := $(shell cd $(GIT_ROOT) && $(GIT) rev-list --count HEAD)
$(eval KSU_VERSION=$(shell expr 30000 + $(KSU_GIT_VERSION)))
```

因此最终构建使用了源自 `common/KernelSU` 的版本，这就是日志中打印 `32490` 的原因。

如果希望未来外部 `KSU_VERSION` 生效，需修补 `common/KernelSU/kernel/Kbuild`，使其在 `KSU_VERSION` 尚未定义时才计算版本。

### 7. 部分 `diff: Unknown option 'I'` 警告虽嘈杂但不阻塞构建

构建过程中出现了若干如下警告：

```text
diff: Unknown option 'I'
```

这些警告并未阻塞内核构建或最终的 dist 步骤。

## 补丁文件

在 `x86_patch/` 目录下生成了两个补丁文件：

- `x86_patch/01-common-x86-kernelsu.patch`
- `x86_patch/02-kernelsu-subrepo.patch`

按此顺序应用：

```sh
# 从 common/ 目录执行
git apply x86_patch/01-common-x86-kernelsu.patch

# 然后从 common/KernelSU/ 目录执行
git apply ../x86_patch/02-kernelsu-subrepo.patch
```

## 输出产物

成功构建后在 `virt/` 目录下生成了以下产物：

- `virt/bzImage`
- `virt/boot.img`
- `virt/vmlinux`
- `virt/initramfs.img`