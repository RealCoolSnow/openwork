# Openwork 项目功能模块与执行链路总结

## 1. 项目概览
Openwork 是一个基于 Electron 的桌面端自动化助手应用。它提供了一个 React构建的用户界面，通过 `node-pty` 调用底层的 OpenCode CLI 来执行用户的自然语言任务。应用支持多种 AI 模型提供商（如 Anthropic, OpenAI, Google, xAI 等），并将敏感的 API 密钥安全地存储在操作系统及其钥匙串中。

## 2. 核心架构
项目采用 Monorepo 结构，主要包含两个部分：
- **apps/desktop**: Electron 桌面应用程序（包含 Main, Preload, 和 Renderer 进程）。
- **packages/shared**: 共享的 TypeScript 类型定义。

### 技术栈
- **Electron**: 应用外壳和主进程逻辑。
- **React (Vite)**: 用户界面渲染。
- **Node-pty**: 伪终端集成，用于执行 CLI 工具。
- **Zustand**: 前端状态管理。
- **Keytar**: 安全存储 API 密钥。

## 3. 功能模块

### 3.1 任务管理 (Task Management)
- **任务创建与执行**: 用户输入 Prompt，主进程通过 IPC 启动任务。
- **生命周期控制**: 支持任务的开始 (`task:start`)、取消 (`task:cancel`) 和中断 (`task:interrupt`)。
- **历史记录**: 自动保存任务的对话记录、状态和结果，支持从历史记录中查看和恢复任务。
- **会话恢复**: 支持基于之前的 Session ID 继续对话 (`session:resume`)。

### 3.2 权限控制 (Permission System)
- **敏感操作拦截**: 当 CLI 试图执行文件读写或命令执行等敏感操作时，会触发权限请求。
- **用户交互**: 主进程拦截请求并通过 IPC 发送到前端，用户在 UI 上点击“允许”或“拒绝”。
- **结果回传**: 用户的决策通过 IPC (`permission:respond`) 回传给 CLI 继续执行。

### 3.3 设置与安全 (Settings & Security)
- **API 密钥管理**: 支持添加、删除和验证多平台 API Key。密钥不以明文存储在配置文件中，而是使用系统的钥匙串服务。
- **应用配置**: 管理调试模式、Onboarding 状态、模型选择 (Model Selection) 等。
- **本地模型支持**: 支持配置 Ollama 连接本地大模型。

### 3.4 CLI 适配层 (OpenCode Adapter)
- 封装了对 OpenCode CLI 的调用。
- 处理标准输入/输出流，将 CLI 的文本输出解析为结构化的消息流推送到前端。

## 4. 任务执行链路 (Execution Flow)

一个典型的任务执行流程如下：

1.  **用户输入**: 用户在 React 前端输入自然语言指令（Prompt）。
2.  **IPC 调用**: 前端通过 `window.accomplish.task.start` 调用主进程的 `task:start` 把手。
3.  **任务初始化**:
    - 主进程生成 `taskId`。
    - 验证用户配置和 Prompt。
    - 初始化权限 API 服务。
4.  **CLI 启动**:
    - `TaskManager` 通过 `node-pty` 启动 OpenCode CLI 子进程。
    - 注入必要的环境变量和上下文。
5.  **实时交互**:
    - **输出流**: CLI 的输出被 `opencode/adapter.ts` 捕获，转换为 `task:update` 或 `task:progress` 事件发送回前端展示。
    - **权限请求**: 若 CLI 需要权限，触发 `onPermissionRequest` -> 前端弹窗 -> 用户操作 -> `permission:respond` -> 恢复 CLI 执行。
    - **调试日志**: 若开启 Debug 模式，详细日志会推送到前端控制台。
6.  **任务结束**:
    - 任务完成或失败后，CLI 退出。
    - 主进程更新任务状态为 `completed` 或 `failed`。
    - 保存完整的任务记录到本地存储。
