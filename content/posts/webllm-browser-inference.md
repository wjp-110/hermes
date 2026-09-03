---
title: "在浏览器里跑大模型:WebLLM 端侧推理深度解析"
date: 2026-09-03
tags: [WebLLM, WebGPU, WASM, LLM, 端侧推理, MLC-LLM]
author: Smilex
---

> 写作日期:2026-09-03。文中所有时效性信息(版本号、模型清单、性能数字)均标注了官方核验入口,引用前请以官方仓库最新状态为准。

# 在浏览器里跑大模型:WebLLM 端侧推理深度解析

打开一个网页,等上十几秒,就能和一个完全运行在你电脑里的 80 亿参数模型对话——提示词不出设备、没有服务器账单、断网也能用。这不是演示动画,而是 WebLLM 这类项目已经做到的日常。

这篇文章从编译器一路拆到浏览器 API,讲清楚三件事:**模型是怎么被"编译"进浏览器的、推理时 GPU 上到底发生了什么、以及它离"替代云 API"还有多远**。

## 1. 为什么要把大模型塞进浏览器

大模型推理的主流形态是服务器端:你的输入上传到数据中心的 GPU,生成结果再传回来。它成熟、强大,但有四个结构性代价:

- **隐私**:提示词与输出都经过第三方服务器,敏感数据(医疗、法律、代码)天然不适合;
- **成本**:按 token 计费,长会话、批量任务会持续产生费用;
- **延迟**:每一轮都要走网络往返,且受服务器负载影响;
- **可用性**:离线场景(飞机、偏远地区、内网)完全不可用。

端侧推理(On-Device Inference)是这些问题的答案——把模型放在用户自己的设备上跑。而**浏览器是覆盖面最大的"端"**:无需安装、天然跨平台、自动获得沙箱安全边界,用户只要打开一个 URL。过去浏览器推理是痴人说梦,因为缺一块拼图:**GPU 通用计算能力**。这块拼图在 2023 年补齐了,名字叫 WebGPU。

## 2. 为什么是现在:WebGPU 补齐的拼图

浏览器跑神经网络的尝试由来已久,技术路线经历了三个阶段:

1. **纯 WASM 时代**:把推理代码编译成 WebAssembly 在 CPU 上跑。可行,但只能吃 CPU,跑大模型慢得没有实用价值。
2. **WebGL2 时代**:WebGL 本质是图形 API,要拿它做通用计算,得把数据编码成纹理、把计算伪装成渲染,别扭且低效。
3. **WebGPU 时代(2023 起)**:WebGPU 是现代图形 API(如 Vulkan/Metal/DX12)在浏览器里的投影,提供了真正的 **compute shader(计算着色器)** 和通用 buffer,让浏览器能直接、高效地驱动 GPU 做大规模并行数值计算——这正是 LLM 推理的核心形态。

对 LLM 推理而言,WebGPU 的三个特性缺一不可:

- **Compute shader**:矩阵乘法、注意力计算等核心算子可以直接写成 WGSL 计算内核,在 GPU 上大规模并行执行;
- **通用 buffer + 显式内存管理**:权重可以驻留在 GPU 显存里反复读取,而不是每次从 CPU 拷贝;
- **低开销、贴近现代 GPU 抽象**:相比 WebGL 的纹理解码绕路,性能损失小得多。

浏览器支持方面,Chrome/Edge 自 2023 年 5 月的 Chrome 113 起在桌面默认启用 WebGPU,是最稳妥的目标平台;Safari 自 18 起随系统提供 WebGPU;Firefox 也在 2025 年年中起逐步默认启用(先 Windows,后扩展到其他平台,具体版本以 Firefox 发布说明为准)。**移动端与各浏览器实现细节差异较大,落地前务必用 webgpureport.org 或 caniuse 复核目标用户群的浏览器版本。**

## 3. 模型是怎么"走进"浏览器的:编译管线

很多人以为"浏览器跑模型"就是把 Hugging Face 上的 safetensors 权重下载下来,用某个 JS 库加载——**这是最大的误解**。现代推理引擎走的是另一条路:先在编译期把模型变成针对目标硬件优化的可执行产物,浏览器只负责运行它。

### 3.1 从 PyTorch 权重到 WGSL 内核

WebLLM 属于 MLC-LLM 项目家族。MLC-LLM 构建在 Apache TVM(具体是 TVM Unity 运行时)之上,是一条**模型编译管线**:

1. 读入模型结构与权重(PyTorch/Hugging Face 格式);
2. 经过 TVM 的图优化与算子编译,把模型翻译成**针对 WebGPU 的计算内核**(WGSL compute shader)与运行调度代码;
3. 内核与 TVM runtime 一起打包成浏览器可加载的 wasm 产物(WebLLM 术语里叫 `model_lib`);
4. 权重本身则被转换(含量化)成 MLC 的专属格式。

**关键含义:模型与编译产物是一一对应的。** 每个"模型 + 量化方式"组合,都对应一份专属的预编译产物。这就是为什么 WebLLM 不能像 transformers.js 那样"随便拿个 ONNX 模型就跑"——它的每个模型都要先经过 MLC 编译管线,这是它性能潜力的来源,也是它生态受限的原因(下文第 10 节展开)。

### 3.2 量化:浏览器推理的生命线

权重多大,决定了模型能不能塞进浏览器。一个直观公式:**权重体积 ≈ 参数量 × 每参数位宽 ÷ 8**。80 亿参数 fp16 权重约 16GB,任何浏览器都扛不住;但压缩到 4-bit 后约 4~5GB,高端一点的设备就能接受。

MLC 有一套自己的量化命名,形如 `q4f16_1`,拆开看:

- `q4`:权重量化到 4 bit;
- `f16`:scale(缩放因子)用 fp16 存储;
- `_1`:分组方式/变体编号(通常是按组量化,组内共享一个 scale)。

常见的还有 `q3f16_1`、`q4f16_ft`(调优变体)以及 `q0f32`(32 位未量化基线,用于对照)。**注意:MLC 用的是自己的量化格式,不是 GGUF**——GGUF 是 llama.cpp 生态的格式,MLC 侧与 GGUF 的关系只是"转换工具链",WebLLM 运行时并不直接消费 GGUF。

量化的意义在浏览器场景被放大,因为推理速度直接受**内存带宽**约束(详见第 7 节):权重瘦身不仅省显存,还直接提速。

### 3.3 分发:模型库与按需下载

编译好的模型(权重 + 配套 wasm)发布在 Hugging Face 的 `mlc-ai` 组织下,仓库命名形如 `Llama-...-q4f16_1-MLC`。WebLLM 包内自带一张默认模型清单(config),应用可以按模型 id 直接引用,引擎负责下载权重、加载并运行。

这种分发模式带来一个工程现实:**首次冷启动要下载数百 MB 到数 GB 的权重**。下载体验(进度、断点、缓存)是 WebLLM 应用的第一道用户体验关,官方 SDK 提供初始化进度回调,工程上通常还要配合缓存策略(第 5 节再谈)。

## 4. 运行时解剖:推理时 GPU 上发生了什么

### 4.1 加载阶段

`CreateMLCEngine(modelId)` 这行代码背后是一连串动作:

1. 解析模型 id,定位权重与 wasm 产物的 URL;
2. 下载权重 + 加载 wasm runtime(TVM 运行时);
3. 请求 WebGPU 设备,把编译好的 compute shader 加载到 GPU;
4. 把权重写入 GPU buffer,按配置分配 KV cache;
5. 引擎就绪,开始响应请求。

初始化进度回调(`initProgressCallback`)会把每一步的进度推给 UI——在权重较大的场景,这一步可能持续数十秒,必须有良好的进度与失败(如不支持 WebGPU)呈现。

### 4.2 Prefill 与 Decode:同一模型,两种截然不同的计算

大模型生成一次回复,内部其实是两个计算阶段:

- **Prefill(预填充)**:一次性并行处理用户输入的整段 token,计算每个位置对后续输出的贡献。它是**算力密集型**的——矩阵乘法可以在 GPU 上千路并行,吃得越饱跑得越快。
- **Decode(解码)**:逐 token 自回归生成。每个新 token 都要读取全部权重,是**带宽密集型**的,几乎无法并行(串行依赖)。Decode 速度直接决定你看到的"打字机效果"有多流畅。

WebLLM 继承了这一套语义,并且会对超长输入的 prefill 做分块处理,避免一次提交把 UI 和 GPU 卡死。

### 4.3 KV Cache:上下文从哪里来

模型能"记住"对话历史,靠的是 KV cache——把历史 token 的注意力中间结果缓存下来,避免每生成一个 token 就重算一遍。KV cache 的大小直接决定最大上下文长度,WebLLM 中由应用配置里的 `kv_cache` 参数(页/块数量)控制:页越多,能容纳的上下文越长,占用显存也越大。

**这就是为什么上下文长度不是免费的**:想在浏览器里跑长文档分析,先回答显存够不够。

### 4.4 流式输出

真实对话产品必须逐字吐出结果,而不是等全部生成完。WebLLM 的接口支持流式(stream),引擎在 worker 内逐增量生成并通过异步迭代器推回主线程,UI 边收边渲染,配合 WebGPU 的分块 prefill,观感上与云 API 的流式输出几乎一致。

## 5. 为什么推理不能占着主线程:Worker 架构

浏览器的主线程负责渲染与交互。LLM 推理是重计算 + 可能持续数十秒的长任务,如果直接在页面主线程跑,结果就是:**页面冻结、按钮点不动、滚动像幻灯片**。

WebLLM 的成熟形态是把引擎放进 **Web Worker**——一个与主线程并行、不阻塞 UI 的后台线程:

- Worker 内部创建真正的推理引擎,处理下载、GPU 初始化、prefill/decode;
- 主线程与 worker 之间通过消息通信,只交换"请求"与"增量结果",页面全程流畅;
- 官方同时提供 service worker 方向的变体,用于更好地管理模型缓存与生命周期。

工程上还有一个容易被忽略的点:**模型下载与 GPU 就绪的进度、失败重试、多会话并发**都应该封装在 worker 层,让 UI 层只面对一个干净的异步接口。

## 6. 上手:最小可用示例

> 以下代码基于 2024–2025 年间长期稳定的 v0.2.x API 形态。API 名称与模型 id 随版本演进,**写代码前请对照官方 README 的示例**。

```js
// 主线程极简用法(实际项目请用 worker,见下文)
import * as webllm from "@mlc-ai/web-llm";

const engine = await webllm.CreateMLCEngine(
  "<模型 id,以官方 config.ts 清单为准>",
  {
    initProgressCallback: (report) => {
      console.log("初始化:", report.text, report.progress);
    },
  }
);

// OpenAI 风格对话接口
const reply = await engine.chat.completions.create({
  messages: [{ role: "user", content: "用一句话解释什么是 KV cache" }],
  stream: true,  // 流式:返回异步迭代器,逐增量输出
});

for await (const chunk of reply) {
  process.stdout.write(chunk.choices[0]?.delta.content ?? "");
}

// 切换模型
await engine.reload("<另一个模型 id>");
```

生产级形态是把引擎放进 worker:

```js
// worker.js —— 引擎真正运行的地方
import * as webllm from "@mlc-ai/web-llm";
new webllm.WebWorkerMLCEngineHandler();
```

```js
// 主线程 —— 与 worker 对话
const engine = await webllm.CreateWebWorkerMLCEngine(
  new Worker(new URL("./worker.js", import.meta.url), { type: "module" }),
  "<模型 id>",
  { initProgressCallback }
);
```

模型 id 不是随便填的——必须是官方预编译清单里的条目(权重和 wasm 都为你准备好了)。去官方 README 或 npm 包内的模型清单里挑一个与你的目标设备匹配的(显存小选小模型,追求质量选大模型)。

## 7. 性能:天花板与瓶颈在哪里

不搬未经核验的数字,讲清楚决定性能的物理规律,你就知道该期待什么、该优化什么。

### 7.1 Decode 是带宽游戏

自回归解码时,每生成一个 token 都要把**全部权重**从显存读一遍。于是:decode 速度 ≈ 显存带宽 ÷ 权重字节数。

- 权重越小(q4 < fp16),每 token 读取的字节越少,速度越快——**量化直接换速度**;
- 显存带宽是硬指标,取决于用户的 GPU/集显,浏览器无法改变它;
- 这就是为什么 1B 模型能跑出比 8B 模型快数倍的体验:它每步只需读 1/8 的字节。

### 7.2 Prefill 是算力游戏

输入处理阶段是并行矩阵乘,受 GPU 算力(FLOPS)约束。分块 prefill 让长输入也能平滑推进,但总耗时与输入长度线性相关。

### 7.3 浏览器层的额外开销

同硬件下,浏览器 WebGPU 推理通常慢于原生(CUDA/Metal)推理:驱动层抽象、提交与同步开销、无法像原生那样吃满全部硬件特性,都是损耗来源。**差距有多大,请以官方 README 中带测量条件的 benchmark 表为准**(注意口径:模型 + 量化 + 设备 + 浏览器 + 是否含下载时间,缺一不可)。

### 7.4 冷启动:被低估的"性能"

用户感知的性能不只是 token/s,还有**从打开页面到第一句话的时间**:下载权重(数百 MB 到数 GB)+ 初始化 WebGPU + 加载内核 + prefill。对 8B 级模型,这可能是几十秒。工程上要用缓存、预加载、清晰的进度设计来消化这段等待。

## 8. 隐私与安全:本地推理的承诺与边界

WebLLM 的安全模型很简单也很强:**全程客户端推理,没有服务器转发**。提示词与输出在设备上产生、在设备上消费,除模型下载本身外没有任何网络传输。对隐私敏感场景(本地文档分析、私有代码助手),这是云 API 给不了的承诺。

几点诚实的边界:

- **模型下载本身是网络行为**:权重来自 Hugging Face 或你自建的镜像,供应链上仍可被观察(用哪个模型是可见的,内容不可见);
- **浏览器沙箱是双刃剑**:页面被沙箱隔离,恶意网页无法越界读文件,但这也意味着推理无法直接访问本地任意资源(需要用户授权机制配合);
- **开源可审计**:WebLLM(Apache-2.0)与 MLC-LLM 代码公开,安全团队可以审计推理链路本身。

## 9. 现实的局限:诚实清单

在决定"用 WebLLM 做生产"之前,把这些局限摆在桌面上:

1. **内存墙**:整个权重必须驻留 GPU/浏览器内存,上下文越长 KV cache 越大。8B 级模型只适合内存充裕的设备,低端笔记本与手机直接出局或只能跑小模型;
2. **下载墙**:首次加载数百 MB 到数 GB,弱网环境劝退;
3. **速度墙**:同硬件不如原生推理,复杂任务(长文档、长生成)等待时间长;
4. **单浏览器约束**:依赖 WebGPU 支持与实现质量,worker 生命周期随页面刷新重置,移动端能力更弱;
5. **生态墙**:模型必须经过 MLC 编译管线,官方清单之外的新模型需要自己编译(见下节对比);
6. **无服务端管理**:没有集中式的用量统计、模型热更新、A/B 与审计,运营能力要自己补。

## 10. 生态对照:WebLLM 不是唯一的浏览器推理路线

| 路线 | 引擎基础 | 强项 | 代价 |
|---|---|---|---|
| **WebLLM** | MLC/TVM 深度编译 → WGSL 内核 | 为 WebGPU 逐模型定制,吞吐上限高,prefill/KV cache 控制精细 | 模型需专属编译,清单受官方支持面约束 |
| **transformers.js** | ONNX Runtime(WebGPU/WASM/WebNN) | 生态广、模型多(含多模态),上手快,HF 全家桶无缝 | 通用 ONNX 内核非逐模型定制,极致性能通常不及深度编译路线 |
| **llama.cpp Web/GGUF 移植** | llama.cpp 编译到 WASM/WebGPU | GGUF 权重即下即用,社区活跃 | 浏览器端工程成熟度与调优参差 |
| **ONNX Runtime Web** | 微软 ORT 的 Web 后端 | 底层引擎级能力,可直接调用 | 偏底层,需要自己搭上层 |

一句话选型:**要生态广度与上手速度选 transformers.js,要 WebGPU 上的极致性能与精细控制选 WebLLM;GGUF 系适合"权重现成优先"的玩家,ONNX Runtime Web 则是自建引擎的地基。** 这不是谁取代谁的关系——同一产品完全可以按模型类型混用。

## 11. 值得盯的方向

以下方向代表了浏览器端推理的演进趋势,是否/何时进入 WebLLM 官方能力,以仓库的 News、Releases 与 CHANGELOG 为准:

- **多模态**:把视觉等非文本输入纳入浏览器推理(历史上 WebLLM 以文本为主);
- **结构化输出 / JSON mode**:让模型输出可被程序直接消费的格式,是 Agent 类应用的前提;
- **推测解码等加速技术**:用一个小草稿模型猜多个 token 再让大模型一次验证,绕开 decode 的串行瓶颈;
- **模型本地缓存**:利用 OPFS(源私有文件系统)等能力把权重持久化在本地,让"二次打开秒加载";
- **WebGPU 跨浏览器收敛**:随着 Safari/Firefox 的实现成熟,碎片化问题逐步缓解。

## 12. 结语

WebLLM 把"模型编译"这门通常属于数据中心的技术,搬到了每一个用户的浏览器里。它用 TVM/MLC 的深度编译换来了浏览器内推理的上限,也用这种方式提醒我们:端侧 AI 的竞争本质上是**生态广度与硬件效率之间的权衡**。

对你来说,值得记住的判断框架是:隐私敏感、设备可控、模型不必最大的场景,浏览器推理已经可以上岗;而它每一年都在变得更小、更快、更省——下次打开那个网页时,等待的时间大概又短了一点。

## 附录 A:时效性核验清单

本文写作于 2026-09-03,以下信息变化快,引用前请现场核验:

1. **npm 最新版本与发布日期**:https://www.npmjs.com/package/@mlc-ai/web-llm
2. **模型清单与官方示例代码**:https://github.com/mlc-ai/web-llm (README + `src/config.ts`)
3. **Release / 近期功能(多模态、结构化输出、推测解码等)**:https://github.com/mlc-ai/web-llm/releases 与 CHANGELOG
4. **性能 benchmark(注意设备/口径)**:WebLLM README benchmark 章节 + https://mlc.ai/blog/
5. **WebGPU 浏览器支持**:https://webgpureport.org 、https://caniuse.com/webgpu
6. **编译工具链(自定义模型时用)**:https://llm.mlc.ai/docs/ 、https://github.com/mlc-ai/mlc-llm

## 附录 B:参考链接

- WebLLM 仓库:https://github.com/mlc-ai/web-llm
- MLC-LLM 仓库:https://github.com/mlc-ai/mlc-llm
- 预编译模型库:https://huggingface.co/mlc-ai
- 官方文档:https://llm.mlc.ai/docs/
- MLC 博客(WebGPU-LLM 系列):https://mlc.ai/blog/
- transformers.js:https://github.com/huggingface/transformers.js
- ONNX Runtime Web:https://onnxruntime.ai/
- llama.cpp:https://github.com/ggml-org/llama.cpp
