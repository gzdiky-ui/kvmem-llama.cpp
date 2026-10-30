# WSL 独立 ROCDXG scratch 修复材料

这是历史 WSL 验收所用的**独立运行库补丁**，不是 Radiance 数学算子、KVMem 选块算法或 Windows 原生 HIP 后端的改动。
Windows 原生与 Linux 原生不要加载此库。目标为 WSL Ubuntu 24.04、ROCm 7.2.3、RX9060XT/gfx1200。

## 来源与修改

- 上游：[ROCm/librocdxg](https://github.com/ROCm/librocdxg)，v1.2.0。
- 固定 commit：`fbd6f28765809e485d63a6a3edeb3fdbf76dc63c`。
- 补丁：`rocdxg-v1.2.0-scratch-reserve.patch`，仅修改 `src/wddm/queue.cpp`，10 行新增、1 行删除。
- 对非零 scratch 请求，初始每 wave 至少预留 8192 bytes；历史设备为 1024 waves，因而初始分配 8 MiB。这个数字并非所有 GPU 固定不变。
- `UpdateScratch` 分配失败后立即返回 `HSA_STATUS_ERROR_OUT_OF_RESOURCES`，避免原调用方忽略失败继续提交。保留 `SCRATCH_RESERVE_V` 日志。

原作者、版权与许可归属见 `UPSTREAM-LICENSE.md`。大部分源代码为 MIT；其中 `src/thunk_proxy/libthunk_proxy.a` 有单独的 AMD 二进制许可，不能将整个上游或最终链接产物笼统说成纯 MIT。本材料只分发补丁、许可与说明，不捆绑该二进制。

## 在 WSL 构建到独立目录

需要 Git、CMake >=3.15、Ninja、GCC >=11.4，以及已安装的 Windows SDK 头文件。历史 CMakeCache 记录 GCC13 的 `/usr/bin/cc`、`/usr/bin/c++`，Release/Ninja，Windows SDK 10.0.26100.0。完整原始 shell 命令未留存；下面根据该构建缓存重建等价配置，不宣称逐字复制历史命令。

在本目录执行，目标目录必须尚不存在，Windows SDK 版本按本机调整：

```bash
git clone https://github.com/ROCm/librocdxg.git ./librocdxg-source
git -C ./librocdxg-source checkout --detach fbd6f28765809e485d63a6a3edeb3fdbf76dc63c
git -C ./librocdxg-source apply --check ../rocdxg-v1.2.0-scratch-reserve.patch
git -C ./librocdxg-source apply ../rocdxg-v1.2.0-scratch-reserve.patch

cmake -S ./librocdxg-source -B ./librocdxg-build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=/usr/bin/cc \
  -DCMAKE_CXX_COMPILER=/usr/bin/c++ \
  -DWIN_SDK='/mnt/c/Program Files (x86)/Windows Kits/10/Include/10.0.26100.0/shared'
cmake --build ./librocdxg-build --parallel 2
sha256sum ./librocdxg-build/librocdxg.so.1.2.0
```

**不要执行 `sudo make install` / `cmake --install`，不要覆盖 `/opt/rocm` 中的库。** 编译目录内保留 `librocdxg.so`、`librocdxg.so.1`、`librocdxg.so.1.2.0` 的正常符号链接关系。

## 仅为模型子进程选择该库

历史验收采用以下环境逻辑（`runtime_dir` 和 `sdk_dir` 改为各自绝对路径）：

```bash
runtime_dir='/absolute/path/to/librocdxg-build'
sdk_dir='/opt/rocm-7.2.3'
env -u LD_PRELOAD \
  HSA_ENABLE_DXG_DETECTION=1 \
  LD_LIBRARY_PATH="$runtime_dir:$sdk_dir/lib:$sdk_dir/llvm/lib" \
  /absolute/path/to/kvmem-radiance-observer-probe [其余完整模型参数]
```

上面是加载机制示意，`[其余完整模型参数]` 不是可直接执行的占位命令；完整服务例程应使用源码包里的统一 launcher，并显式给 `--wsl-runtime-dir`。历史测试脚本使用 `KVMEM_DIAG_ROCDXG_DIR` 再前置到子进程 `LD_LIBRARY_PATH`；仅设置这个变量而不经对应脚本不会自动改变动态链接器行为。

不使用 `LD_PRELOAD`：历史准备阶段发现它导致初始化失败。此目录也不配置 shell 全局环境、系统服务或驱动。

## 历史库与新构建库不能混为一谈

历史已验收文件：

```text
librocdxg.so.1.2.0
SHA256 2b2a082ede46324a5e3771b4b14f2850fdb9bd38348abfbeedee83966bbd6088
```

`provenance.json` 记录来源、补丁与源文件校验和；`historical-build-evidence.json` 记录历史工具链及实际加载证据。最终 33 请求的 maps 确认加载该库，日志确认初始 `requested_bytes=8388608 result=0`。

**同一源码重新编译的 SHA256 可能不同**，因为编译器、路径、链接器或依赖不同。新产物只能称候选，不能修改历史哈希或把历史 33 轮成绩自动归给它。默认 launcher 对历史库的校验应保持严格。

新库的验收至少要独立记录：

1. 源 commit、补丁 SHA、源码差异、编译器、SDK与产物完整SHA。
2. 小型 GPU scratch/复制/回读一致性测试，然后少数固定模型请求；与现用服务串行，不启动两个大模型。
3. 运行时 `/proc/PID/maps` 实际加载路径、`SCRATCH_RESERVE_V`分配结果、MTP host/device一致性、RAM/swap与新增驱动错误。
4. 若要声称替代历史完整验收，再做同参数长上下文/33请求；退出期已知 `destroy_allocation -22` 历史记录仍保留，不宣称补丁解决所有DXG错误。

由维护者审核新候选后，才能提供显式、可追溯的新运行库验收记录供 launcher 使用；编译完成本身不是放行理由。本材料不运行GPU测试，也不安装运行库。
