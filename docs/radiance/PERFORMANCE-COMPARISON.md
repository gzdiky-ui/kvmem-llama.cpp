# Radiance：与 KVMem README 完整指标对照

更新日期：2026-10-31。上游基准固定为 [`89c66980ed88`](https://github.com/kvmem/kvmem/blob/89c66980ed885866251414e1897518bdf008d211/docs/recommended-config-performance.md)，不是随时间变化的 master。本文保留其全部配置行、Task 1 / MTP长度对照 / Task 2 指标行；没有测的数据明确写“未测”，不拿其他任务补空。

**Windows 表是 2026-10-28 合格二进制的历史33请求数据。2026-10-30循环生成版、新master接入版分别列验证状态，不能继承这张表作为自己的GPU成绩。**

## 推荐配置逐行对照

| 配置 | IQ3 | IQ4 | Radiance Windows（历史测试/当前交互分列说明） |
|---|---|---| --- |
| 启动脚本 | [start-iq3.sh](https://github.com/kvmem/kvmem/blob/89c66980ed885866251414e1897518bdf008d211/scripts/start-iq3.sh) | [start-iq4.sh](https://github.com/kvmem/kvmem/blob/89c66980ed885866251414e1897518bdf008d211/scripts/start-iq4.sh) | 见配套部署文档（旧33轮不是DSH） |
| 主权重 | IQ3_S，GSQ-RCO | IQ4_XS，MTP 权重 Q4_0 | AMD MXFP4骨干；额外6个MTP投影和主输出头MXFP4 |
| 主模型设备 | RTX 5060 Ti | RTX 5060 Ti | RX 9060 XT 16 GiB（RDNA4/gfx1200） |
| 视觉头 / 设备 | Q8_0 / GPU | BF16 / CPU | 未加载；纯文本部署 |
| 主 KV / MTP KV | Q8_0 / F16 | Q5_0 / F16 | FP8 / FP8（非Q8_0、非F16） |
| MTP 草稿长度 / 状态保存 | 3 / ReplaySSM | 3 / ReplaySSM | 3 / Radiance原生recurrent checkpoint + replay |
| 检索预算 | 36864（36 × 1024） | 32768（32 × 1024） | 36864 |
| 生成预留 / 默认最大输出 | 16384（16 × 1024） | 12288（12 × 1024） | 16384 / 旧测试512；新交互65536（循环生成） |
| context / batch | 262144 / 512 | 262144 / 512 | 262144 / 512 |
| 脚本默认图片 token 上限 | 512 | 512 | 本次未启用图片 |
| query replay / policy | auto / user | auto / user | 冻结user query / mean-key评分；不冒称新版attention反馈 |
| 正式脚本默认思考预算 | 4096 | 4096 | 旧33轮128；新交互不设独立上限、xhigh |

## Task 1：图文测试，保留所有指标

| 指标 | IQ3 | IQ4 | Radiance Windows |
|---|---:|---:| --- |
| 全任务聚合 prefill，首遍 | **574.49 token/s** | **595.60 token/s** | 未测（当前部署未加载视觉模块） |
| 全任务聚合 prefill，有效 | **544.65 token/s** | **505.52 token/s** | 未测（当前部署未加载视觉模块） |
| 首次图片编码，预热轮 | **0.41 秒（GPU）** | **21.79 秒（CPU）** | 未测（当前部署未加载视觉模块） |
| 全任务聚合 decode，含思考 | **38.55 token/s** | **44.30 token/s** | 未测（当前部署未加载视觉模块） |
| 图片回答 decode，含思考 | 38.64 token/s | 45.83 token/s | 未测（当前部署未加载视觉模块） |
| 图片回答输出量，每轮含思考 | 94 token | 64 token | 未测（当前部署未加载视觉模块） |
| 写代码 decode，含思考 | **39.88 token/s** | **44.35 token/s** | 未测（当前部署未加载视觉模块） |
| MTP 草稿接受率 | 70.74% | 80.85% | 未测（当前部署未加载视觉模块） |
| 整卡采样峰值显存 | **15591.10 MiB** | **15445.10 MiB** | 未测（当前部署未加载视觉模块） |
| 峰值时可用显存 | **460.90 MiB** | **606.90 MiB** | 未测（当前部署未加载视觉模块） |
| 运行阶段进程 RAM 峰值（RSS，不含加载） | **4308.21 MiB** | **4846.82 MiB** | 未测（当前部署未加载视觉模块） |

原任务包含约12K文本、896×896图像、HTML/SVG生成；1次预热、2次正式运行。鹈鹕纯文字请求不是这个图文任务，不能代填。

## 原表 MTP3 / MTP4 / MTP5 对照

| 设置 | IQ3 聚合 decode | IQ3 峰值显存 | IQ4 聚合 decode | IQ4 峰值显存 | Radiance同任务对照 |
|---|---:|---:|---:|---:| --- |
| MTP3 | **38.55 token/s** | 15591.10 MiB | **44.30 token/s** | 15445.10 MiB | 未测 |
| MTP4 | 37.82 token/s | 15595.10 MiB | 43.04 token/s | 15449.10 MiB | 未测 |
| MTP5 | 34.09 token/s | 15597.10 MiB | 未测 | 未测 | 未测 |

## Task 2：256K / 33请求，全部原指标

| 指标 | IQ3 | IQ4 | 历史ROCm IQ3 / RX7900XTX | 历史ROCm IQ4 / RX7900XTX | Radiance WSL / RX9060XT | Radiance Windows / RX9060XT |
|---|---:|---:| --- | --- | --- | --- |
| 全任务聚合 prefill，首遍 | **437.13 token/s** | **463.18 token/s** | 384.47 token/s | 419.52 token/s | 未独立计时 | 未独立计时 |
| 全任务聚合 prefill，有效 | **242.06 token/s** | **253.41 token/s** | 216.81 token/s | 232.00 token/s | 940.83 token/s | **792.93 token/s** |
| 工具轮次聚合 decode，含思考 | **31.74 token/s** | **33.31 token/s** | 33.77 token/s | 39.31 token/s | 48.01 token/s | **45.24 token/s** |
| 最终代码 decode，512 token | 30.53 token/s | 38.15 token/s | 39.34 token/s（仅232 token） | 44.78 token/s | 44.44 token/s；代码不合格 | 41.78 token/s；512 token，但代码不合格 |
| 工具轮次 MTP 草稿接受率 | 64.70% | 67.08% | 59.66% | 60.79% | 68.27% | 68.27% |
| 整卡采样峰值显存 | **15617.10 MiB** | **15591.69 MiB** | 16011 MiB | 16164 MiB | 15736.91 MiB（Windows适配器专用计数） | 16082.15 MiB（Windows适配器专用计数，采样覆盖加载及运行） |
| 峰值时可用显存 | **434.90 MiB** | **460.31 MiB** | 8549 MiB（24GiB卡） | 8396 MiB（24GiB卡） | 未独立采集 | 未独立采集，不以总容量减占用冒充实测 |
| 运行阶段进程 RAM 峰值（RSS，不含加载） | **13483.52 MiB** | **11244.75 MiB** | 13067.95 MiB | 10978.13 MiB | 11055.93 MiB（WSL VmRSS） | 10925.13 MiB（Windows working set，排除加载） |

ROCm历史两列来自同一上游commit的 [rocm-recommended-config-performance.md](https://github.com/kvmem/kvmem/blob/89c66980ed885866251414e1897518bdf008d211/docs/rocm-recommended-config-performance.md)。这是PR33中的RX7900XTX24GiB记录，不是本机后来ROCm10/RX9060XT记录。原文标SDK“7.14”，本表保留历史标注，不自行更正为其他版本。

### 计算口径与一次必要更正

- Prefill：全部33请求的**新增逻辑输入位置** / 全部prefill秒数。Windows为261527 / 329.8222894 = **792.93 token/s**。首遍计算没有单独计时。
- Windows旧报告的817.12是269506个`prompt_n` / 同一耗时，包含额外处理位置，不是README定义的有效prefill。旧报告保留，本表用严格对齐口径；不是速度变慢，也没有重新跑模型。
- Decode：后32个工具请求返回的总输出（含thinking）6469 / 142.9989088 = **45.24 token/s**，排除初始基础请求。
- 接受率：sum(accepted_draft_tokens) / sum(draft_tokens) = 4332 / 6345 = **68.27%**，不是逐轮百分比平均。
- 末轮按API返回512token/原生计时得到41.7770；原生`predicted_per_second`按N−1计数为41.6954，两者不能混列。
- Windows RAM只采`elapsed >= startup_seconds`，共484条运行采样；working set是Windows对应的驻留工作集指标，并非Linux VmRSS字段。private commit不等于实际RAM。
- 显存为Windows适配器DedicatedUsage计数，含其他进程，不是假称NVML板卡采样。共享GPU内存不能直接解释成KV溢出。

生成代码和原始数字保存在 `prepare_release.py`、`evidence/windows-final33-readme-metrics.json`、`evidence/wsl-final33-readme-metrics.json`；摘要不包含个人进程列表、凭据或DSH聊天。

### 完成状态与不可混淆的差异

Windows33请求完成，最终261528输入+512输出=262040，31次可复用前缀精确恢复；固定54528物理KV tokens / 2104.125MiB。spill1829、restore0。这组输入没有覆盖Host旧块召回；后来独立短测真实restore191/197块，并正确返回两个目标标记，不能把短测并入33轮速度。

Windows与WSL历史33轮的回答和reasoning逐条一致；批次确认均7/31，末轮代码未通过语法检查。运行/容量通过不等于语义质量通过。WSL原始driver因为旧prefix检查器判定而exit1；离线逐request复核通过，原始失败证据保留，不改写为原driver成功。

CUDA采用RTX5060Ti、GGUF IQ3/IQ4、Q8/Q5主KV及F16草稿KV并保持视觉头加载；Radiance使用RX9060XT、MXFP4、FP8 KV且视觉不加载。平台、模型表示、GPU、算法版本、缓存行为不同，不能声称这是单变量引擎加速。尤其当前mean-key选择在此fixture未回载冷块，旧llama.cpp会回载；更高prefill不能全归功于算子。

## 新交互版和新master接入状态

| 版本 | 真实验证 | 不应宣称 |
|---|---|---|
| 2026-10-28 Windows合格版，exe `53a7d5fa…` | 上述33请求；独立Host召回 | 通用accuracy通过 |
| 循环候选、prefix修复前 | 输入64886+强制输出18432，真实spill98，物理池不变 | 重复输出58.57t/s代表日常能力 |
| 2026-10-30 DSH部署版，exe `52969463…` | prefix修复CPU回归；原生思考续接生成完整动画HTML；真实App短调用 | 本版已重新33轮，或一次请求直接自然输出50K |
| PR候选（KVMem 89c66980，2026-10-31） | 新Windows全量构建；5请求短测：2次Host召回+3次prefix；详见QUICK-VALIDATION-20261031.md | 本版重新通过33轮、通用质量通过或Linux/WSL已重新验证 |

新交互版配置仍B36864/R16384/batch512/context262144、FP8 K/V、MTP3。xhigh、不独立限制thinking、max_tokens65536来自新客户端/循环适配；旧33测试仍是thinking128/max512。两类任务不能混算平均速度。
