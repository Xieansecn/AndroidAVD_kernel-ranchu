### 第一步：准备 Arch Linux 宿主机环境

> [!CAUTION]
> AI生成

Kleaf 构建系统是“密封的（Hermetic）”，这意味着它会自己下载特定的编译器（Clang）和工具链。但宿主机仍需要安装一些基础的调用工具（特别是 `rsync`，Bazel 强依赖它同步文件）。

打开你的 Arch 终端，执行：

```bash
sudo pacman -Syu
sudo pacman -S base-devel git repo python curl rsync zip unzip cpio ncurses libelf bc

```

### 第二步：拉取完整的内核源码树

为了最大程度兼容你的 AVD 虚拟机（基于 Android 15，内核 `6.6.66`），我们需要拉取对应的 `android15-6.6` 分支代码。

```bash
# 创建工作目录
mkdir -p ~/android-kernel && cd ~/android-kernel

# 初始化 repo 仓库 (使用 android15-6.6 分支)
repo init -u https://android.googlesource.com/kernel/manifest -b common-android15-6.6

# 开始同步代码 (源码巨大，包含工具链约 30GB+，请耐心等待)
repo sync -c -j$(nproc) --no-clone-bundle

```

*注：如果你想绝对精确地匹配你之前的 `13070261` 批次，你可以去 CI 页面下载它的 `manifest_13070261.xml` 文件，并用 `repo init -m` 来替换上面的命令。*

### 第三步：注入 KernelSU 及处理 x86_64 安全补丁

由于你是为 x86_64 编译完整内核，我们可以直接把 KernelSU 集成进源码树，这样就能从根本上解决之前遇到的 `X86_FEATURE_INDIRECT_SAFE` 宏缺失问题。

```bash
# 进入核心源码目录
cd common

# 1. 运行 KernelSU 官方安装脚本
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -

# 2. (非常重要) 为 x86_64 架构打上间接调用绕过补丁 (Indirect Syscall Bypass)
# 这个补丁在 KernelSU 的仓库里，通常会在 setup.sh 时给出提示，我们需要手动 apply
# 确保在 common 目录下执行：
patch -p1 < KernelSU/kernel/patches/x86_64_bpf_indirect_call.patch 

# 退回到根目录准备编译
cd ..

```

*(说明：打上这个 patch 后，内核才会生成那个必须的宏，KernelSU 才能在 x86_64 的安全机制下安全运行，不会引起 Kernel Panic。)*

### 第四步：使用 Bazel/Kleaf 启动编译

由于我们要编译的是 AVD 虚拟机专用内核，Google 已经为它配置好了专属的构建目标：`virtual_device_x86_64_dist`。

```bash
# 回到 android-kernel 根目录
# 启动 Bazel 密封构建，并将最终的内核镜像、头文件和 .ko 模块全部导出到 out/dist 目录
tools/bazel run //common-modules/virtual-device:virtual_device_x86_64_dist -- --dist_dir=out/dist

```

*构建过程极度消耗 CPU 和内存，根据机器性能可能需要 30 分钟到数小时。*

### 第五步：提取产物与替换 AVD 内核

编译完成后，你会在 `out/dist/` 目录下找到海量文件。你需要提取以下几个关键文件：

1. **`bzImage`**：这就是编译好的 x86_64 内核本体（包含了 KernelSU 的钩子）。
2. **`initramfs.img`**：虚拟机的初始内存盘。
3. 如果你在 KernelSU 中开启了作为模块编译，里面还会有一个 `kernelsu.ko`。

**如何让 AVD 使用新内核启动？**
找到你本地 Android SDK 的 emulator 路径，使用命令行带参数启动 AVD，强行替换它的内核镜像：

```bash
# 假设你的 AVD 名字叫 Pixel_8_API_35
# 替换为新编译出的内核镜像启动
emulator -avd Pixel_8_API_35 -kernel /path/to/your/out/dist/bzImage -show-kernel

```

### 总结

1. **舍弃孤立编译**：在 GitHub Actions 里用残缺的 Headers 强行编 `.ko` 是黑客解法（容易死机）。
2. **拥抱 Kleaf**：用 Arch Linux 完整跑一遍 `repo sync` -> `patch` -> `bazel run`。你不仅能得到完美的 `bzImage`，还能顺带产出 100% 匹配的 `vmlinux` 和 `kernel-headers.tar.gz`，一劳永逸。