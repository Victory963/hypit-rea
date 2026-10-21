# Hypit 代码架构说明

> 本文档基于仓库 `hypit-ai/hypit`（版本 0.3.1，`main` 分支）源码整理，用于归档备份。

## 1. 项目概述

Hypit 是一个面向 AI Agent 的「视频创作发行版（Video authoring Distribution）」。作者用声明式源文件描述视频（`.svml` 结构/组件、`.svrun` 运行选择、`.svs` 参数配方），Hypit 将其编译为作者图（Author Graph）与运行图（Run Graph），再由运行时（Runtime）调度确定性 Producer 与外部 Provider（图像/视频/语音生成、ffmpeg、浏览器截帧、WhisperX 对齐等）执行，最终产出可复用的构建结果（Build Result）与成片。

- 包名：`@hypit/hypit`，可执行文件：`hypit`（`bin/hypit.mjs`）
- 语言：TypeScript 5.9（运行时通过 `tsx` 直接加载源码），ESM
- 运行环境：Node.js >= 22.15.0，包管理器 pnpm 10.33.0（workspace monorepo）
- 许可证：见 `LICENSE`（Apache-2.0 附加条件）

## 2. 顶层目录结构

| 路径 | 说明 |
| --- | --- |
| `bin/hypit.mjs` | CLI 启动器：注册 tsx、安装发行版包解析、组装 Video 发行版并分发命令 |
| `packages/` | 约 160 个 workspace 包（核心、编译、运行时、作者组件、Provider、Studio 等） |
| `examples/` | 示例工程（排名、播客、街头采访、复杂讲解、语义组合、最小作者包、Provider 包） |
| `docs/` | VitePress 文档站（`docs/zh` 为中文：快速开始、开发指南、运行时、Provider、测试等） |
| `skills/hypit` | 供 AI Agent 使用的 Hypit 技能说明 |
| `migrations/` | 版本迁移说明（0.3、0.3.1，中英文） |
| `scripts/` | 打包、类型构建、独立包检查、发布流程脚本 |
| `test/` | 测试入口 `test/run.mjs` 及公共夹具 |
| `package.json` / `pnpm-workspace.yaml` / `tsconfig.json` | 根清单、工作区（`packages/*`、`services/*`、`examples/*/packages/*`）与 TS 配置 |

## 3. 分层架构与主要包

整体按「领域无关核心 → 编译 → 执行/运行时 → 领域作者包 → Provider → 宿主应用」分层，核心不感知视频、XML 或具体服务。

### 3.1 核心（领域无关）
- `kernel`：图编译器与构建状态机。从 Targets 出发链接 Author Graph 与 Run Graph，推导有限执行闭包；`BuildDefinition` + `BuildFact` 由 `BuildMachine` 重建状态并发出下一批命令。
- `core`、`protocol`、`facet`、`text`、`blob`、`atomic-file`、`validation`：基础类型、协议、Facet 描述、文本/二进制/原子文件工具。

### 3.2 源码与编译
- `source`：字节级 Source 契约（`SourceUnit`、`ResolvedSource`）；`source/text` 解析 `<?svml using="..."?>` 自描述头。
- `workspace`、`workspace-fs-node`：可替换的作者/运行源码与资源会话（文件系统、内存、远程等）。
- `markup`（`.svml` 前端）、`recipe`（`.svs` 参数配方前端）、`run` / `run-markup`（`.svrun` 运行源 → Run Graph）、`temporal-markup`、`compiler-markup-node`。
- `author`、`author-kit`、`elaborator`：作者模块链接与 Graph Fragment 静态展开（引用解析、类型检查、环检测）。
- `compiler`、`compiler-node`：编排 Workspace + 前端 + 清单 → Source 闭包 → Author Module/Graph。
- `loader`、`package-loader-node`：包贡献（Module/Facet）定义与 Node 端发行版包解析（`hypit.packageSources`）。
- `admission`：类型所有者校验。

### 3.3 执行与运行时
- `producer`：确定性 Producer SDK（只接收不可变类型化输入，外部工作以 Need 表达）。
- `executor`：`ProducerRegistry` + `EndpointRegistry`，执行 Kernel 生成的命令。
- `endpoint`、`endpoint-kit`：Provider 能力端点（含 HTTP 适配）。
- `runtime`、`runtime-kit`、`runtime-host-node`：运行时端口（BuildStore、OperationStore、ResourceStore、CredentialStore、命令执行器）。
- `runtime-local`：默认本地运行时（Worker、调度器、SQLite 活动工作列表），并贡献 `runtime`、`programs`、`paths` 命令。
- `store-sqlite`、`resource-store-fs`、`resource-store-s3`、`transport-aws-lambda`：存储与远程执行。
- `result`、`build-result*`、`artifact`：构建结果持久化（文件系统 / S3）与导出。
- `credential-store-env/file/local/os/platform`：凭据存储（环境变量、文件、系统钥匙串、Windows Credential），凭据不进入源码与构建状态。
- `project`：解析项目根（`package.json` 中 `hypit.project: true`）与 `.hypit/runtime` 运行时配置指针。

### 3.4 视频领域作者包（组件与轨道）
`timeline`、`timeline-author`、`temporal`、`spatial`、`media`、`media-track`、`visual-track`、`audio-track`、`typography-track`、`deck-track`、`caption`、`caption-fine`、`text-fine`、`film`、`composition`、`script`、`narrative*`、`speech*`、`sound`、`ranking`、`comment-sticker`、`screen-overlay`、`interview-emoji-reveal`、`fontsource`/`fonts-open`、`html-program`、`html-video`、`hyperframes`、`render-hyperframes`、`visual-ir`、`raster`、`image-*`、`depth-stack`、`performance`、`program-space`、`semantic-take-adjust` 等。带 `-studio` 后缀的包为对应组件在 Studio 中的可视化编辑扩展。

### 3.5 模型能力与 Provider
- 模型族/能力包：`seedance`(+kits)、`seedream`、`gpt-image`(+kits)、`nano-banana`、`grok-imagine`、`wan`、`pixverse`、`minimax-h3`、`elevenlabs-speech`、`fishaudio-speech`、`mimo-speech`、`whisperx`、`volcengine-matting`、`background-removal`、`generation`、`model-kit`。
- Provider 实现：`provider-hypihub`、`provider-tokendance`、`provider-beatapi`、`provider-hiapi`、`provider-monid`、`provider-pollo`（云端服务），`provider-html-local`、`provider-hyperframes-local`、`provider-image-opencv-local`、`provider-media-local`、`provider-whisperx-local`（本地，WhisperX 服务为 Python/uv 工程）。
- 本地工具：`media-local`（ffmpeg/ffprobe，`hypit media`）、`browser-capture`（浏览器截帧）、`yt-dlp`、`speech-estimate`。

### 3.6 宿主应用
- `cli`：领域无关命令引擎（检查、规划、构建、结果查询/导出、凭据输入、OAuth 等），命令实现见 `packages/cli/src/commands/`（environment/execution/results）。
- `video`、`video-cli`：官方视频发行版组装（Markup 前端、文件系统 Workspace、源包发现、初始 Runtime Profile、`hypit vocabulary`）。
- `studio`、`studio-adapter`、`studio-companion`：官方 Web Studio（`hypit studio`），打开 Run Source、实时编译预览、编辑语义/视觉投影、反馈服务、快照（`hypit snapshot --studio`）。

## 4. 入口与命令

`bin/hypit.mjs` 流程：
1. `--version` 直接输出版本；
2. 解析发行版物理根目录，`tsx` 注册以直接运行 TS 源码；
3. `installDistributionPackageResolution()` 安装包解析；
4. `createVideoDistribution()`（`@hypit/video`）组装发行版；
5. `runNodeCli()` → `runInstalledCliApplication()` 分发命令。

根 `package.json` 的 `hypit.cli.use` 声明命令贡献包：`runtime-local`、`browser-capture`、`yt-dlp`、`media-local`、`speech-estimate`、`whisperx`、`video`、`studio`。常用命令：`hypit studio`、`hypit runtime`、`hypit programs`、`hypit paths`、`hypit media`、`hypit snapshot`、`hypit vocabulary`，以及 CLI 核心的检查/规划/构建/结果类命令（`hypit --help` 查看完整列表）。

## 5. 数据流 / 流水线

```text
.svml / .svs / .svrun (Workspace)
   │ source/text 识别头部 → 选择前端 (markup / recipe / run-markup)
   ▼
compiler + author：Source 闭包 → Author Module → Author Graph
   ▼
run：Run Source + Author Graph → Run Graph（候选、操作、满足关系、Targets）
   ▼
kernel：规划执行闭包 → BuildDefinition；BuildMachine 发出命令
   ▼
executor / runtime-local：Producer（确定性）与 Endpoint/Provider（外部 Need：生成图像/视频/语音、ffmpeg、渲染）
   ▼
result：Build Result（公开 Outputs、文件归属、可复用于后续构建）→ 成片 / 导出
```

Studio 在同一编译结果上提供预览与编辑，源文件修改经 source-watch 重新编译。

## 6. 配置

- 项目：`package.json` 中 `"hypit": { "project": true }` 标识项目根。
- 运行时：`hypit.runtime.json`（如 `format: hypit.runtime-local@1`、`dataRoot: .hypit/runtimes/local`、`endpoints` 绑定 Provider 及并发），项目 `.hypit/runtime` 指向所选 Profile。
- 凭据：通过 credential-store（环境变量 / 系统凭据库）提供，不写入源码。
- 构建产物与运行数据默认在项目 `.hypit/` 下（已加入忽略，不入库）。

## 7. 构建与运行

```bash
pnpm install            # 安装依赖（Node >= 22.15，pnpm 10.33）
pnpm hypit --help       # 等价 node bin/hypit.mjs
pnpm studio             # 启动 Web Studio
pnpm check              # tsc 类型检查
pnpm test               # node test/run.mjs
pnpm docs:dev           # 本地文档站
pnpm build:public-types # 生成公共类型 dist/public
pnpm pack:distribution  # 打包发行版
```

## 8. 示例

见 `examples/README.md`：`ranking-football`（排名视频）、`podcast`（播客）、`interview`（街头采访，含字幕跟踪）、`complex-explainer`（137 秒复杂讲解，素材需另行下载，未入库）、`semantic-composition`（语义组合聊天动画）、`minimal-author-package`、`provider-package`。

## 9. 主要依赖

`tsx`、`typescript`、`@types/node`、`koffi`（Windows 凭据 FFI）、`cjs-module-lexer`；开发依赖 `vitepress`、`rollup`、`rollup-plugin-dts`、`linkedom`、`@fontsource*` 字体；外部工具 ffmpeg/ffprobe、浏览器、yt-dlp、WhisperX（Python/uv）。

## 10. 备份说明

本仓库为本地源码备份。被 git 忽略的内容（`node_modules`、`.hypit` 构建结果、`output`、`examples/complex-explainer/productions/explainer/assets` 素材等）未包含在内。
