# MyClaudeCode 使用指南

> 基于 Claude Code v2.1.88 深度魔改版，移除遥测、远程控制、模型白名单、强制更新等限制。

---

## 一、首次环境准备（只需一次）

### 1.1 安装 Node.js 依赖

项目只有两个开发依赖：`esbuild`（打包器）和 `typescript`（类型检查）。

**在 VS Code 中：** `Ctrl+Shift+P` → `Tasks: Run Task` → `① 安装依赖 (npm install)`

或手动在终端执行：
```bash
cd /home/slime/env/MyClaudeCode
npm install
```

安装完成后 `node_modules/` 目录出现，IDE 的红色波浪线会大量消失。

> ⚠️ **注意**：项目运行时依赖（axios、react、ink 等）来自全局安装的 `@anthropic-ai/claude-code` 包，不在此项目的 `package.json` 中。IDE 里仍会有部分"找不到模块"的报错（如 `axios`），这是正常的——构建时 esbuild 会用 `--packages=external` 跳过它们。

### 1.2 配置环境变量

在 `~/.bashrc` 或 `~/.zshrc` 中添加：

```bash
# 自定义 API 代理端点（兼容 Anthropic 消息格式的代理）
export ANTHROPIC_BASE_URL=https://your-proxy.example.com/v1
export ANTHROPIC_API_KEY=sk-your-custom-key

# 可选：指定默认模型（支持任意模型名，白名单已移除）
export ANTHROPIC_MODEL=claude-opus-4-5

# 可选：完全禁用所有非必要网络请求
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

---

## 二、日常工作流（VS Code 任务）

`Ctrl+Shift+P` → `Tasks: Run Task`，或直接 `Ctrl+Shift+B` 触发默认构建任务。

| 任务 | 说明 |
|------|------|
| `① 安装依赖` | `npm install`，首次或 package.json 变更后执行 |
| `② 构建` | `node scripts/build.mjs` → 生成 `dist/cli.js` |
| `③ 类型检查` | `tsc --noEmit`，检查类型错误，不产生输出文件 |
| `④ 运行构建结果` | `node dist/cli.js`，测试构建是否正常 |
| `⑤ 安装到全局 claude` | 将 `dist/cli.js` 覆盖到全局 `@anthropic-ai/claude-code` 包 |
| `⑥ 构建 + 安装到全局 (一键)` | 先构建再覆盖，常用快捷流程 |

### 标准修改流程

1. 在 `src/` 里修改源码
2. 运行任务 `⑥ 构建 + 安装到全局` 
3. 打开新终端验证：`claude --version` 或 `claude -p "hello"`

---

## 三、构建系统说明

### 构建流程

```
src/  ──(复制)──▶  build-src/  ──(esbuild 打包)──▶  dist/cli.js
```

`build.mjs` 做了三件事：
1. 将 `src/` 复制到临时目录 `build-src/`（不修改源文件）
2. 替换 `MACRO.VERSION` 等编译时常量、`feature()` → `false`
3. 用 esbuild 打包为单文件 `dist/cli.js`（运行时依赖标记为 external）

> `build-src/` 是临时构建目录，已加入 `.vscode/settings.json` 的排除列表，不会干扰 IDE。

### 为什么 IDE 有大量报错

这是**预期行为**。IDE 报错原因：

| 报错类型 | 原因 | 影响构建？ |
|---------|------|-----------|
| `找不到模块 "axios"` 等 | 运行时依赖不在 package.json | **不影响**，esbuild 用 `--packages=external` |
| `找不到名称 "process"` | 装包前 `@types/node` 未加载 | 装包后消失 |
| `找不到模块 "react"` 等 | 同上 | **不影响** |

**只要构建成功（任务 ② 输出 `✅ Build succeeded`），就可以正常使用。**

---

## 四、TypeScript 项目配置说明

### tsconfig.json 关键配置

| 配置项 | 值 | 说明 |
|-------|-----|------|
| `target` | `ES2022` | 编译目标 |
| `module` | `ESNext` | 模块格式 |
| `moduleResolution` | `bundler` | 适配 esbuild |
| `types` | `["node"]` | 启用 Node.js 全局类型 |
| `paths.bun:bundle` | `stubs/bun-bundle.ts` | 将 Bun 内置模块映射到 stub |
| `skipLibCheck` | `true` | 跳过第三方库类型检查（减少噪音）|
| `strict` | `false` | 关闭严格模式（原始代码风格）|

### stubs/ 目录作用

```
stubs/
├── bun-bundle.ts    # 替代 bun:bundle，提供 feature() 函数（始终返回 false）
├── global.d.ts      # 声明 MACRO.VERSION 等全局编译时常量的类型
└── macros.ts        # 运行时 MACRO 值（构建时被替换为字面量）
```

---

## 五、版本管理（Git）

完全通过 IDE 的 Git 面板操作，无需命令行。

### 保存改动

1. IDE 左侧 Source Control 图标
2. 看到修改的文件列表 → 点击 `+` 暂存（Stage）
3. 写 Commit Message → 点击 `✓ Commit`

### 打版本标记（Tag）

`Ctrl+Shift+P` → `Git: Create Tag`，输入名称如 `v2.1.88-patched`

### 回退到某个版本

Source Control → `...` → `Checkout to...` → 选择 tag 或 commit

### 查看改动了哪些文件

Source Control 面板 → `...` → `View History`，或点击任意文件看 diff

---

## 六、魔改内容速查

所有修改均在 `src/` 中，搜索 `// PATCHED:` 可定位所有改动点。

| 文件 | 功能 | 改动 |
|------|------|------|
| `services/analytics/sink.ts` | 事件上报 | → noop，不上报任何数据 |
| `services/analytics/datadog.ts` | Datadog 遥测 | → 空函数，禁用 |
| `services/analytics/firstPartyEventLogger.ts` | OTEL 日志 | → 返回 false，连带禁用 GrowthBook |
| `services/remoteManagedSettings/index.ts` | 远程设置拉取 | → 立即返回 |
| `services/remoteManagedSettings/syncCache.ts` | 远程设置缓存 | → 永远不读磁盘缓存 |
| `services/policyLimits/index.ts` | 组织策略限制 | → 永远不拉取，永远允许 |
| `utils/autoUpdater.ts` | 版本强制退出 | → 立即返回，不检查 |
| `utils/config.ts` | 自动更新开关 | → 永远禁用 |
| `components/AutoUpdater.tsx` | 后台 npm 轮询 | → 立即返回 |
| `utils/model/modelAllowlist.ts` | 模型白名单 | → 始终返回 true |
| `utils/model/providers.ts` | URL 是否 1P | → 始终返回 true（支持自定义代理）|
| `constants/oauth.ts` | OAuth URL 白名单 | → 删除白名单限制 |
| `utils/permissions/permissionSetup.ts` | bypassPermissions gate | → 硬编码为 false（始终允许）|

---

## 七、故障排查

**构建失败 `❌ Build failed after all rounds`**
→ 查看 `build-src/` 目录下报错的文件，通常是新增的第三方导入找不到

**运行时 `Error: Cannot find module 'xxx'`**
→ 该运行时依赖不在全局 `@anthropic-ai/claude-code/node_modules/` 中，检查全局安装是否完整

**`claude` 命令仍是旧版**
→ 确认运行了任务 `⑤ 安装到全局` 或手动执行了 `cp dist/cli.js ...`

**IDE 报大量红色错误**
→ 正常现象，先确认 `npm install` 已执行；构建通过即可使用
