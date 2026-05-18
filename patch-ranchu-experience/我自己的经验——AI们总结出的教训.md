# 初

> *从头开始，窥探**初**明*

从获取 内核版本 开始

```shell
adb shell cat /proc/version
Linux version 6.6.66-android15-8-gb66429556fb8-ab13070261 (kleaf@build-host) (Android (11368308, +pgo, +bolt, +lto, +mlgo, based on r510928) clang version 18.0.0 (https://android.googlesource.com/toolchain/llvm-project 477610d4d0d988e69dbc3fae4fe86bff3f07f2b5), LLD 18.0.0) #1 SMP PREEMPT Fri Feb 14 22:29:59 UTC 2025
```

**明确版本: `6.6.66-android15-8-gb66429556fb8-ab13070261`**  
> 按[经验](https://5ec1cff.github.io/my-blog/2024/01/31/avd-ksu2/)讲，我要从Android CI里找[13070261](https://ci.android.com/builds/submitted/13070261/kernel_virt_x86_64/latest)这一个构建号，然后拿到manifest进行拉取。

我们当然要用[repo](https://help.mirrorz.org/git-repo/)工具:

```bash
curl -L https://mirrors.bfsu.edu.cn/git/git-repo -o repo
chmod +x repo
export REPO_URL='https://mirrors.bfsu.edu.cn/git/git-repo'
```

至于我为啥不用清华源了，是因为最近清华源有排队问题，总之我从上个月就没连上过了

---

## 拉取仓库

```bash
mkdir android15-6.6 && cd android15-6.6
repo init -u https://android.googlesource.com/kernel/manifest -b common-android15-6.6 --depth=1
```

> 使用国内镜像源替换`https://android.googlesource.com`，比如用北外源:`https://mirrors.bfsu.edu.cn/git/AOSP/`

这一段是最基础最简单的国内镜像AOSP源拉取方式，它被repo工具使用，以便拉取内容。  

---

现在有几种方法获取manifest:

|方法|链接|
|:---:|:---:|
|复现CI build|[CI manifest android15-6.6-2025-02](https://ci.android.com/builds/submitted/13070261/kernel_virt_x86_64/latest/manifest_13070261.xml)|
|同步6.6内核最新分支|[googlesource manifestandroid15-6.6](https://android.googlesource.com/kernel/manifest/+/refs/heads/common-android15-6.6/default.xml)|
|跟随2025-02月份|[googlesource manifest android15-6.6-2025-02](https://android.googlesource.com/kernel/manifest/+/refs/heads/common-android15-6.6-2025-02/default.xml)|

其中从CI获取的方式无法使用curl,wget等工具拉取，需要从网页下载。  

---

从源码下载:

```bash
wget -O default.xml 'https://android.googlesource.com/kernel/manifest/+/refs/heads/common-android15-6.6-2025-02/default.xml?format=TEXT'
```

但wget下载下来是base64内容，所以我们还需要进行base64解码，使用base64 -d default.xml解码它。  
如果你不使用CI方法下载，需要把它重命名成与[CI 13070261](https://ci.android.com/builds/submitted/13070261/kernel_virt_x86_64/latest)一致的名字，但可选，后续也要使用这个文件名。  
得到CI的清单内容:  
[manifest](manifest_13070261.xml)文件也应该被换源: `https://mirrors.ustc.edu.cn/aosp/`  

> [!TIP]
> 我推荐使用CI快照的清单文件，其他那两个方法下载的都是变动的，不绝对与原内核相同。但从CI获取的清单文件也应该做些更改。参考[修改好的清单文件](manifest_13070261(1).xml)与[清单文件差异](#清单文件差异)

---

### 清单文件差异

差异 1：`<remote>` 源地址（手动替换镜像）

```diff
- <remote name="aosp" fetch="https://android.googlesource.com/"
-         review="https://android.googlesource.com/" />

+ <remote name="aosp" fetch="https://mirrors.ustc.edu.cn/aosp/"
+         review="https://mirrors.ustc.edu.cn/aosp/" />
```

差异 2：`<superproject>` revision（去除了日期后缀）

```diff
- <superproject ... revision="common-android15-6.6-2025-02" />

+ <superproject ... revision="common-android15-6.6" />
```

差异 3：`kernel/common` 去除 `upstream` / `dest-branch`

```diff
- <project path="common" name="kernel/common"
-     revision="b66429556fb8cd9718a39198ef7aec8a1fa9ec32"
-     upstream="android15-6.6-2025-02"
-     dest-branch="android15-6.6-2025-02" />

+ <project path="common" name="kernel/common"
+     revision="b66429556fb8cd9718a39198ef7aec8a1fa9ec32" />
```

差异 4：`kernel/tests` 整段删除

```diff
- <project path="kernel/tests" name="kernel/tests"
-     revision="b118a72b0e0e898661aec81dd75d65da8d548857"
-     upstream="main" dest-branch="main">
-     <linkfile dest="run_test_only.sh" src="tools/run_test_only.sh" />
-     <linkfile dest="launch_cvd.sh" src="tools/launch_cvd.sh" />
-     <linkfile dest="flash_device.sh" src="tools/flash_device.sh" />
- </project>
```

---

最后，我们把[修改好的清单文件](manifest_13070261(1).xml)放入到`.repo/manifests`文件夹内  
终于到拉取同步的环节了

```bash
cp manifest_11987101.xml .repo/manifests/
repo init -m manifest_11987101.xml
repo sync -c --no-tags --no-clone-bundle -j$(nproc)
```

---

# 曙

> *不断摸索，前方**曙**光*

## 开始编译

先尝试编译原内核，看它能不能用

```shell
tools/bazel run --config=fast --config=stamp --lto=none //common-modules/virtual-device:virtual_device_x86_64_dist -- --dist_dir=virt
```

做好你的github账户配置

```shell
git config --global user.name "Xieansecn"
git config --global user.email "xieansecn@163.com"
```

打上KernelSU，先交一版，防止和后面的补丁冲突

```bash
cd common
git clone https://github.com/tiann/KernelSU.git
./KernelSU/kernel/setup.sh main
git add -A
git commit -m "Add KernelSU"
```

按道理讲Kbuild会帮助设置好版本，不需要下面这一步

```bash
cd KernelSU
export KSU_VER=$(($(git rev-list --count HEAD) + 10200))
echo $KSU_VER
```

### 这次和上次不同

是的，由于Kernel Commit: [x86/syscall: Don't force use of indirect calls for system calls](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=1e3ad78334a69b36e107232e337f9d693dcc9df2)，这个提交内容本质上是移除了对x86架构上的sys_call_table的强制使用，而KernelSU正是使用Hook sys_call_table达到直接修改内存中的sys_call_table数组来拦截系统调用。  
这就导致了新版KernelSU也不再主动支持x86,x86_64架构，内核部分也需要打上[KernelSU Support x86_64](https://kernelsu.org/guide/x86_64-support.html)提到的补丁：

```
For kernel 6.6:
https://github.com/android-generic/kernel_common/commit/fe9a9b4c320577c30e1f22d04039e414c6a3cdec
https://github.com/android-generic/kernel_common/commit/df772e99e392f24b395ceaf7b26974e3e4828ee9

For kernel 6.12:
https://github.com/android-generic/kernel-zenith/commit/dd2c602268fdc81f4d3b662f6a15142ac0ec7bcd
https://github.com/android-generic/kernel-zenith/commit/7d99237ae5da61c19447138da3282ae37d43857b

For kernel 6.18:
https://github.com/android-generic/kernel-zenith/commit/40b1c323d1ad29c86e041d665c7f089b9a3ccfb5
https://github.com/android-generic/kernel-zenith/commit/f5813e10b7630e1ccd86fc2c4cf30eef60b64a82
```

> [!WARNING]
> 通过应用这些补丁并禁用系统调用加固，实际上是在有意绕过一个旨在防止投机性执行漏洞的缓解措施。
> 这会重新打开系统调用的间接分支攻击面。 如果你运行的是生产服务器或严格侧信道安全至关重要的系统，请不要应用这些补丁。这适用于那些通过 KernelSU 优先考虑 root 访问而非特定硬件漏洞缓解的测试环境。

这些补丁会创建`X86_FEATURE_INDIRECT_SAFE`,可以通过`cmdline syscall_hardening=off`激活  
我们使用kernel 6.6内核的补丁来实现在x86_64架构上使用KernelSU，参考：  
[android15-6.6_x86_64.patch](patchs/android15-6.6_x86_64.patch)  
[kernelSU](/patchs/kernelSU.patch)

使用:
`git apply android15-6.6_x86_64.patch` 和 `git apply kernelSU.patch` 来进行使用

如果你不愿意`-dirty`出现在你的内核名称上，请在打入补丁后分别为KernelSU和内核写一个提交:

```shell
cd common
git commit -am "Patch for KernelSU"
cd KernelSU
git commit -am "Patch for KernelSU"
```

~~提交消息写你认为你该写的~~

然后跑:

```shell
tools/bazel run --config=fast --config=stamp --lto=none //common-modules/virtual-device:virtual_device_x86_64_dist -- --dist_dir=virt
```

如果成功说明没问题，如果没有成功要么是版本不匹配，要么是patch位置不对，这时候~~就该借助AI来写补丁了~~
然后回退`git reset HEAD~1`，尝试看patch的详细内容，手动patch下。或者跟我上面说的那样，**使用AI**

# 终

> *蝉的一生，也要**终**归于地下*

`virt`文件夹产出一堆.ko文件和`initramfs.img` `boot.img` `vmlinux` `bzImage`  
我们最终需要的是`bzImage`这个LZ4压缩内核镜像
要么在你的sdk目录下，找到你的虚拟机镜像目录使用这个文件替换掉你的`kernel-ranchu`内核文件  
要么使用`-kernel`参数，在你启动emulator时加入

```shell
# 启动虚拟机
./emulator -avd Pixel_9 -kernel bzImage -show-kernel -no-snapshot-load
```

这样，总之如果没爆找不到设备或者开不了机，就是一切都大功告成了！
