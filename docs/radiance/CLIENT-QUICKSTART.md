# 给接手者的启动和客户端接入说明

## 先分清三个东西

1. **模型文件**：ModelScope 私有仓库 `Zeshion/Qwen3.8-27B-Radiance-KVMem-MXFP4` 中的 `target-head-mxfp4-a.rad`。不是GGUF，不能交给llama.cpp启动。
2. **推理服务**：Radiance 的 C++ / AMD GPU 程序，加上 KVMem adapter。Python启动脚本只负责参数、环境和内存保护，不在Python里执行模型算子。
3. **Harness / DSH**：调用本机 HTTP API 的客户端。启动客户端本身不等于其他机器已安装模型/编译好服务；这台机器的DSH有专门的生命周期插件。

## 当前机器：现成可用版本

直接使用 Windows DeepSeek Harness。已部署的本地 provider 为 `radiance-local`，API为 `http://127.0.0.1:18380/v1`。新建会话默认xhigh；旧会话可能保留以前的Off设置。现用版详细启动和回退保留在原部署目录；个人profile备份不包含在本公开源码包中。

不要再额外启动第二个同模型GPU进程。手动部署测试时使用独立端口也不能解决显存被两个模型占满的问题。

## 新机器：统一入口及系统选择

解压源码包后进入其中的 `kvmem` 目录。旁边的 `../radiance-native` 已包含完整配套源码和校验清单，可离线用作 `--source`。不要混入最新ABI31的源码或插件。模型另外从ModelScope仓库获取；当前是私有仓库，需账号有权访问。

统一构建入口为 `backends/radiance/tools/build.py`，统一服务入口为 `backends/radiance/tools/serve.py`。
两者均支持 `--platform auto|windows|linux`；`auto` 检测当前系统。在Windows用 `windows`，在原生Linux或WSL终端内用 `linux`。这只是选择当前系统的实现，**不会把Windows进程变成Linux进程，也不会自动进入WSL**。各系统分别编译自己的程序和插件。

### Windows原生

准备Python3、CMake、Ninja、VS x64 C++ Build Tools/Windows SDK、AMD HIP SDK7.2及兼容驱动。以下在 **x64 Native Tools 开发者终端**运行，模型/SDK路径按本机替换：

```powershell
python backends/radiance/tools/build.py --platform windows --source ../radiance-native --build ../radiance-build-windows --rocm "C:/Program Files/AMD/ROCm/7.2" --jobs 2

python backends/radiance/tools/serve.py --platform windows --exe ../radiance-build-windows/bin/kvmem-radiance-observer-probe.exe --home ../radiance-build-windows/radiance_home --model D:/models/target-head-mxfp4-a.rad --rocm "C:/Program Files/AMD/ROCm/7.2"
```

### 原生Linux

准备Python3、CMake、Ninja、C/C++构建工具、ROCm开发SDK及正常GPU驱动。以下在Linux shell中运行；SDK版本和模型路径按真实安装调整：

```bash
python3 backends/radiance/tools/build.py --platform linux --source ../radiance-native --build ../radiance-build-linux --rocm /opt/rocm-7.2.3 --arch gfx1200 --jobs 2

python3 backends/radiance/tools/serve.py --platform linux --exe ../radiance-build-linux/bin/kvmem-radiance-observer-probe --home ../radiance-build-linux/radiance_home --model /data/models/target-head-mxfp4-a.rad --rocm /opt/rocm-7.2.3
```

### WSL

在WSL发行版终端内使用Linux构建命令。模型路径可为 `/mnt/d/models/target-head-mxfp4-a.rad`，不是Windows的 `D:/...`。
历史WSL完整配置还依赖独立、已验证的scratch运行库修复；**这个库不由源码包自动安装，也不等于Windows HIP SDK**。
必须先备好文档指定的同版本运行库，再传 `--wsl-runtime-dir`：

```bash
python3 backends/radiance/tools/serve.py --platform linux --exe ../radiance-build-linux/bin/kvmem-radiance-observer-probe --home ../radiance-build-linux/radiance_home --model /mnt/d/models/target-head-mxfp4-a.rad --rocm /opt/rocm-7.2.3 --wsl-runtime-dir /path/to/verified-rocdxg-reserve-v
```

`/path/to/...` 是需要替换的路径，并非包内已有目录。缺少已验证库时会明确报错，不会静默装系统库或降低预算。在另一台WSL机器上目前不能承诺仅解压后一个参数就复现历史成绩；具体库名、哈希与限制见 `docs/radiance-integration.md`。独立运行库的补丁、来源/许可和构建步骤在资料包 `wsl-runtime/README.md`，也收录于源码包 `backends/radiance/wsl-runtime/README.md`。重编产生不同SHA时先独立验收，不通过替换历史哈希绕过校验。

### 启动前检查与停止

在对应服务命令末尾加 `--print-command` 可核查路径/参数而不加载模型；去掉后才真正启动服务。`Ctrl+C` 停止这个启动器创建的模型子进程，不停止其他服务、WSL或应用。不要同时运行两套模型进程。

本轮实测设备为RX9060XT/gfx1200；其他卡不由入口参数自动获得支持。新master候选的CPU检查、Windows全量重建与旧GPU验收分开，首次部署先做短请求。新增跨平台入口检查不等于最新代码已经在Linux/WSL重跑整模型。

## 默认值：不要把不同预算混成一个数

| 名称 | 当前显式部署profile |
|---|---:|
| KVMem选择预算 | 36864 tokens |
| 生成/验证KV工作空间预留 | 16384 tokens |
| batch / context | 512 / 262144 |
| MTP | 3 |
| K和V类型 | FP8 / FP8 |
| Host archive上限 | 9216 MiB |
| GPU headroom | 256 MiB |
| 块大小 | 128 tokens |
| 日常单请求总输出上限 | 客户端传65536，含思考和答案 |
| 思考 | xhigh，不传独立thinking_budget |

max_tokens65536不等于实际每次生成65536，也不等于GPU预留65536；长生成使用循环回收。并发保持1。服务只监听loopback，不向公网暴露，无需把本机服务当云端API发布。

## 客户端最小调用

服务ready之后，任何支持OpenAI兼容chat/completions的客户端可用：

- base URL：`http://127.0.0.1:18380/v1`
- model：`radiance-local`
- API key：本机服务不验证云端密钥；客户端若要求非空可使用无敏感含义的本地占位值。不要填DeepSeek/OpenAI真实密钥。
- temperature=1、top_p=.95、top_k=20；xhigh；不另设thinking_token_budget。

Python标准库示例（无需安装OpenAI SDK）：

```python
import json
import urllib.request

payload = {
    "model": "radiance-local",
    "messages": [{"role": "user", "content": "写个鹈鹕骑自行车的html网页，要求使用html 和 svg 动画实现，有速度调节功能"}],
    "reasoning_effort": "xhigh",
    "max_tokens": 65536,
    "temperature": 1.0,
    "top_p": 0.95,
    "top_k": 20,
    "seed": 42,
    "stream": False,
}
request = urllib.request.Request(
    "http://127.0.0.1:18380/v1/chat/completions",
    data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
    headers={"Content-Type": "application/json"},
)
with urllib.request.urlopen(request, timeout=3600) as response:
    result = json.load(response)
print(result["choices"][0]["message"].get("content", ""))
print("finish_reason:", result["choices"][0].get("finish_reason"))
print("usage:", result.get("usage"))
```

这是连接方式示例，本轮打包过程中未重新运行鹈鹕。Harness可能附加system prompt、工具描述、历史和默认采样参数；想做模型效果对照时须保存实际请求，不能只看用户输入的一句话。

## 发给维护者的话（可直接改称呼使用）

> 这次整理了Radiance + KVMem的RDNA4适配，Windows原生和WSL的既有实测分别保留。已把adapter叠到最新KVMem固定commit `89c66980`，增加Radiance原生packed KV页的布局契约并复用现有pressure_selection；没有用旧core覆盖新版。模型单独在ModelScope私有仓库，代码包不含权重。默认BUDGET36864、生成预留16384、batch512、context262144、MTP3，K/V均FP8。另附完整README同指标表、量化七个张量的清单和构建/启动命令。新master候选仍以自身构建和后续GPU结果为准，未把旧33轮成绩自动贴给它；当前仍是ABI15、mean-key评分，不宣称ABI31或完整新版attention反馈/RDNA3验收。

PR标题/正文和精确测试状态见代码包的 `PR-DRAFT.md`。本轮未push或创建PR。
