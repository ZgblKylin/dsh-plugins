# AGENTS.md

本仓库是个人自用的 DSH 插件集合。所有插件使用现有 DSH/Cordis 接口编写，目标是能在官方原版 DSH（含 Desktop 2.x）中无额外依赖直接加载。

## 仓库布局

```
src/          插件源码，一个插件一个子目录
references/   deepseek-harness-desktop 官方仓库子模块，只读参考，永不修改
README.md     仓库说明
```

- `references/deepseek-harness-desktop/` 是 pinned 上游子模块，只用于查证规范；不要编辑其中的任何文件，不要从该目录向插件源码复制代码。
- 新增插件在 `src/<plugin-name>/` 下创建：`package.json`、`src/index.ts`（或 `index.js`）、`README.md`，必要时附 `tests/`。

## 规范状态：Fabric 是 Draft，不是可依赖目标

- `dsh-community-fabric` 的 manifest、capability、Host Descriptor 与统一事件模型目前**仅有 Draft 文档**，没有 runtime、SDK、正式 schema 或可加载插件。插件代码不得 import 它，不得声称"符合 Fabric 规范"作为兼容性承诺。
- 当前真正可用、可依赖的是官方现有接口：Cordis 插件模型（`@deepseek-ai/cordis`）与 Desktop 公开的 `desktopProfiles` / `desktopPnpm` Host service。
- 文档中的表述（"必须/应该/可以"）在 RFC 被接受前不构成稳定承诺；写 README 或注释时不要把它们当成已发布的 API。
- **Fabric 只作为插件开发的约束参考，不得进入插件交付内容**：插件代码、README、注释、`package.json` 与发布说明一律只提官方 DSH/Cordis 接口；不 import Fabric、不引用其 manifest/capability/事件模型、不声明"符合 Fabric 规范"。约束（如组合优先、声明清晰、兼容优先）以本文件下面的规范为准。

## 插件开发规范（基于官方 plugin-development.md 与生态倡议）

### 组合优先、声明清晰、兼容优先

1. **组合优先**：通过官方 slot、service 和 patch 组合能力；不要假设或覆盖其他插件的内部实现。
2. **声明清晰**：用 `inject` 显式声明依赖的 service 和 slot，不依赖运行时巧合。
3. **兼容优先**：升级保持向后兼容，不破坏已有组合。

### Cordis 基本规则

- 插件是一个返回 `{ name, inject, apply(ctx) }` 的对象（或 Service 子类）。`inject` 列出必需依赖；Cordis 会让插件等待这些服务出现后再激活。
- **注册是 effect**：任何注册（事件监听、工具、命令、定时器、流）都必须通过 `ctx.effect()` / `ctx.on()` 或返回 disposer 的官方 helper，保证 reload/teardown 可逆、顺序可控。
- **Waterfall 监听器必须调用 `next()`** 才能把（可能被包装的）结果传给下一个监听器；不调用即为短路。需要先于普通注册运行时才用 `prepend: true`。
- 事件按 `emit` / `waterfall` / `parallel` / `serial` 四种模式分发，模式是事件的公开契约，不得混用。
- 捕获行为优先走事件（拦截/策略），直接能力调用优先走 service 方法。

### 读取可选服务

- 用 `ctx.get('serviceName')` 读取可选服务并处理 `undefined`；只有硬依赖才放进 `inject`。
- 只有声明在 `inject` 里的服务才能作为 `ctx.serviceName` 访问，绝不访问未声明的服务。

### 跨环境插件（普通 DSH + Desktop）

- 同一插件若要在普通 `dsh web` 和 Desktop 都能运行，不要把 Desktop service 放进顶层 required `inject`。先注入普通依赖，再在 `apply` 中探测：
  - `ctx.get('desktopProfiles')` 存在 → 在 `ctx.inject(['desktopPnpm'], cb)` 中挂载 Desktop 实现；
  - 不存在 → 挂载普通 DSH fallback（fallback 仍是插件的权威实现）。
- Desktop-only 插件才把 `desktopProfiles` / `desktopPnpm` 放进顶层 `inject`。
- **不要**从 `process.argv`、`ctx.baseUrl`、settings 或 `$DSH_HOME` 推断 Desktop profile；Desktop 中以 `desktopProfiles.current` 为准。

### desktopPnpm 的正确用法

| 方法 | 用途 |
| --- | --- |
| `installPlugin(request)` | 唯一受支持的安装（add）路径：带 recovery receipt、profile 快照/WAL 生命周期。 |
| `runPlugin(args, invokingDir)` | 非安装变更：remove、update、依赖修复；会拒绝 `add`。 |
| `run(args)` | 低层 pnpm 操作，不保证 DSH profile 初始化与 bundle reconcile；仅当明确不需要时使用。 |

- 参数始终作为 argv 传递，不要拼接 shell 字符串，不要依赖 Windows `.cmd` shim。
- 一个 generation 同时只允许一个 package operation；服务会在 generation dispose 时终止仍在运行的 operation，插件卸载时必须在 effect disposer 中取消并等待 `done`。
- package 操作只从明确的用户动作发起：校验目标来源、安装前持久保存 recovery receipt、启动时 reconcile、设置自己的 timeout、同时检查 `exitCode` 和 `signal`。

### 不要依赖的接口

`desktopRuntime`、`desktopPnpmBootstrap`、Electron `BrowserWindow`、托盘注册表、private Node helper、`ELECTRON_RUN_AS_NODE` 和生成的 shim 都是 Desktop 内部实现，即使出现在类型声明或运行时上下文中也不属于第三方兼容 contract。

### 安全边界

- Capability/权限声明用于兼容判断、用户确认与审计，**不是安全沙箱**。受信任的同进程 JS 插件可以绕过 `ctx` 直接调操作系统接口。
- 不要在文档中把"声明通过"描述成安全审核或权限强制。

## 质量要求

- 每个插件至少验证：
  - 在普通 DSH 中无 Desktop service 时能加载（或按产品定义保持 pending）；
  - Desktop 中读取的 profile name/dir 与用户实际选择一致；
  - package operation 的取消、非零退出、spawn failure 与 teardown 路径；
  - 插件变更重启后 bundle 能进入下一次 Loader 组合。
- 为每个插件写简短 README：说明用途、依赖的 service/slot、配置项、已知限制。
- 插件行为有显著变化时更新 README，保持文档与代码同步。

## 参考文档（在 references/deepseek-harness-desktop/ 下）

- `docs/plugin-development.md` — 插件开发总览（现有接口 vs Draft 的区分）
- `dsh-plugin-desktop/docs/plugin-services.md` — `desktopProfiles` / `desktopPnpm` 契约
- `docs/plugin-ecosystem.md` — 生态倡议（组合优先 / 声明清晰 / 兼容优先）
- `deepseek-harness/docs/cordis-primer.md` — Cordis 五种核心思想、事件模式、waterfall 语义
- `dsh-community-fabric/` — 仅作前瞻参考，不作为实现依据
