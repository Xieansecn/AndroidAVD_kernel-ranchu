# KernelSU x86_64 Bring-up Notes


> [!CAUTION]
> AI生成

## Scope

This document records the changes and lessons learned while making the
Android 15 6.6 x86_64 virtual-device kernel boot successfully in AVD with
KernelSU enabled.

The build was validated with:

```sh
tools/bazel run --config=fast --config=stamp --lto=none \
  --action_env=KSU_VERSION=10201 \
  //common-modules/virtual-device:virtual_device_x86_64_dist \
  -- --dist_dir=virt
```

The final build completed successfully and copied artifacts into `virt/`.

## Main lessons

### 1. KernelSU on x86_64 expects indirect-syscall support markers

KernelSU checks for `X86_FEATURE_INDIRECT_SAFE` on x86_64 and aborts early if
the feature is not present. The kernel tree used here did not have the exact
backport KernelSU expects, so the following had to be added:

- `X86_FEATURE_INDIRECT_SAFE` in `arch/x86/include/asm/cpufeatures.h`
- indirect syscall-table dispatch in `arch/x86/entry/common.c`
- exported `ia32_sys_call_table[]` and `x32_sys_call_table[]`
- a boot-time toggle for syscall hardening in
  `arch/x86/kernel/cpu/common.c`

### 2. The x32 and ia32 tables must be exported explicitly

Using indirect dispatch from `arch/x86/entry/common.c` is not enough. The
compat and x32 syscall tables must also exist as symbols:

- `ia32_sys_call_table[]` from `arch/x86/entry/syscall_32.c`
- `x32_sys_call_table[]` from `arch/x86/entry/syscall_x32.c`

Without those symbols, the build fails in `arch/x86/entry/common.c`.

### 3. `syscall_hardening` needs both boot logic and sysfs exposure

To make the new mode visible and controllable, the implementation added:

- `x86_syscall_hardening_enabled`
- `early_param("syscall_hardening", ...)`
- `/sys/devices/system/cpu/syscall_hardening`

The sysfs attribute is wired through:

- `arch/x86/kernel/cpu/common.c`
- `include/linux/cpu.h`
- `drivers/base/cpu.c`

### 4. `signal.h` needed a warning-safe rewrite for clang

The original signal-set helper macros triggered clang array-bounds warnings in
this build. The stable fix here was to replace macro-heavy helper logic with
simple loop-based implementations for:

- `sigisemptyset`
- `sigequalsets`
- `sigorsets`
- `sigandsets`
- `sigandnsets`
- `signotset`

This avoided the warning without changing behavior.

### 5. KernelSU itself needed two small build fixes

Inside the `common/KernelSU` subrepository, two changes were needed:

- add `#include <linux/bug.h>` in `kernel/hook/patch_memory.h`
- add `-Wno-array-bounds` in `kernel/Kbuild`

These changes live in the KernelSU subrepo patch, not in the main kernel patch.

### 6. `KSU_VERSION` from Bazel was overridden by KernelSU's own Kbuild logic

Passing `--action_env=KSU_VERSION=10201` did not control the final printed
KernelSU version. The reason is that `common/KernelSU/kernel/Kbuild` detects
its own git repository and recalculates `KSU_VERSION` internally with:

```make
KSU_GIT_VERSION := $(shell cd $(GIT_ROOT) && $(GIT) rev-list --count HEAD)
$(eval KSU_VERSION=$(shell expr 30000 + $(KSU_GIT_VERSION)))
```

So the final build used the version derived from `common/KernelSU`, which is
why the log printed `32490`.

If you want the external `KSU_VERSION` to win in the future, patch
`common/KernelSU/kernel/Kbuild` so it only computes a version when
`KSU_VERSION` is not already defined.

### 7. Some `diff: Unknown option 'I'` warnings are noisy but non-blocking

The build emitted several warnings like:

```text
diff: Unknown option 'I'
```

Those warnings did not block the kernel build or the final dist step.

## Patch files

Two patch files were generated in `x86_patch/`:

- `x86_patch/01-common-x86-kernelsu.patch`
- `x86_patch/02-kernelsu-subrepo.patch`

Apply them in this order:

```sh
# from common/
git apply x86_patch/01-common-x86-kernelsu.patch

# then from common/KernelSU/
git apply ../x86_patch/02-kernelsu-subrepo.patch
```

## Output artifacts

The successful build produced artifacts under `virt/`, including:

- `virt/bzImage`
- `virt/boot.img`
- `virt/vmlinux`
- `virt/initramfs.img`
