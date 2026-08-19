# AGENTS.md

本仓库是个人自用的 DSH 插件集合。所有插件使用现有 DSH/Cordis 接口编写，目标是能在官方原版 DSH（含 Desktop 2.x）中无额外依赖直接加载。

## 仓库布局

```
src/          插件源码，一个插件一个子目录
references/   deepseek-harness-desktop 官方仓库子模块，只读参考，永不修改
third_party/  第三方插件源码（git 克隆后本地编译安装用），含各自的 .git
.dsh/skills/  项目级 DSH skill（每个 skill 一个子目录，内含 SKILL.md）
README.md     仓库说明
```

- `references/deepseek-harness-desktop/` 是 pinned 上游子模块，只用于查证规范；不要编辑其中的任何文件，不要从该目录向插件源码复制代码。
- 新增插件在 `src/<plugin-name>/` 下创建：`package.json`、`src/index.ts`（或 `index.js`）、`README.md`，必要时附 `tests/`。
- 第三方源码插件放在 `third_party/<plugin>/`（保留 `.git` 便于更新）；编译产物 tgz 放在 `third_party/` 下，已被 `.gitignore` 的 `*.tgz` 排除。

## 项目 Skills

- **`dsh-plugin-install`**：安装/卸载 DSH 插件到 profile 的标准流程。涉及插件安装、卸载、更新、源码编译安装时，先加载该 skill。其内容基于 `dsh plugin --profile <profile> add --help`（受管安装器转发的 pnpm add）的权威安装方式列表：npm 包 / tag / 版本 / 版本范围 / git 简写 / git URL / 本地 tgz / tarball URL / 目录。
- Skill 存放于 `.dsh/skills/<skill-name>/SKILL.md`（frontmatter 含 `name`、`description`、`whenToUse`），由 DSH 的 `skill-filesystem` provider 自动发现。

## 规范状态：Fabric 是 Draft，Market 已实现

- `dsh-community-fabric` 的 manifest、capability、Host Descriptor 与统一事件模型目前**仅有 Draft 文档**，没有 runtime、SDK、正式 schema 或可加载插件。插件代码不得 import 它，不得声称"符合 Fabric 规范"作为兼容性承诺。
- **`dsh-community-market` 与 Fabric 不同：它已完成并内置于 DSH Desktop**，其 `docs/` 是已实现的公开契约（`catalog-provider-contract` 为已实现的公开 v1 契约），对插件作者有实际约束；但它不发明新插件格式——市场只消费 npm package，`desktopProfiles` / `desktopPnpm` 仍是受管安装的底层服务。
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

### 插件市场（Community Market）约束

Market 是 DSH Desktop 内置的开放插件市场，只消费 npm package（经 `desktopProfiles` / `desktopPnpm` 受管安装），**不发明新插件格式**；本仓库插件如要在市场中被收录/安装，需满足其已实现的公开契约：

- **包必须可被受管安装器接纳**：发布到 npm 的 package 应使用精确稳定的 SemVer 版本；不得依赖 GitHub URL、版本范围、`latest` tag 或 prerelease 作为安装目标。
- **不得定义安装 lifecycle script**：manifest 中不得出现 `preinstall`、`install`、`postinstall` 或 `prepare`（受管安装器会拒绝这类 package）。
- **运行时兼容**：声明的 DSH/Cordis 依赖需与 Desktop 内置 DSH runtime（当前基于 `0.1.0-rc.7`）兼容；`engines.node` 需接受 Desktop 内置的 Node.js runtime。
- **需要 DSH bundle 证据**：若要作为 bundle 被加载，`dsh.bundle.patch` 必须指向 package 内真实存在的文件，且不得越出 package 目录。
- **安全合规**：package 需由官方 npm 提供 HTTPS tarball 与合法 SHA-512 integrity。
- **收录 ≠ 安全审核**：目录条目被展示或出现在"可安装"页，不代表任何一方审核、推荐或背书；插件 README 与文档不得暗示这一点。

### bundle 加载（`dsh.bundle.patch`）注意事项

- `dsh.bundle.patch` 是**官方契约**（非社区私有约定）：npm 包的 manifest 声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`，即成为一个可安装的 profile bundle 层。
- 官方依据：`packages/boot/app-boot/src/profile.ts` 的 `DshBundleManifest`；`docs/user/develop/basic/publish.md`（bundle 教程）；`apps/cli/reference/README.md`（`dsh plugin add` 后按该声明 reconcile `dsh.profile.bundles`）；`docs/architecture.md`（"`dsh.bundle` points at a bundle's patch file"）；官方内置 `packages/bundle/{base,web-app,headless}/` 均为此格式。
- 流程：`dsh plugin --profile <name> add <pkg>` 安装后，若 manifest 声明了 `dsh.bundle`，CLI 自动把该包追加进 `dsh.profile.bundles`；profile 启动时按列表顺序应用各 bundle 的 patch 层。
- 因此**同时带 client 半的插件**（`dsh.client`）也走 bundle 通道即可：patch 里 `insert` 自己的 Loader row（`name` 为包名），Node 半侧扫描该 entry 的 `dsh.client` 并服务浏览器 bundle，无需手工改 profile 的 `cordis.patch.yml`。
- patch 文件必须随发布包含（`files` 白名单加 `cordis.patch.yml`），且路径不得越出 package 目录（Market 安装器会校验）。
- 未声明 `dsh.bundle` 的包仍可安装，但只作为普通依赖、不激活任何层（CLI 打印一次性警告）。

## 质量要求

- 每个插件至少验证：
  - 在普通 DSH 中无 Desktop service 时能加载（或按产品定义保持 pending）；
  - Desktop 中读取的 profile name/dir 与用户实际选择一致；
  - package operation 的取消、非零退出、spawn failure 与 teardown 路径；
  - 插件变更重启后 bundle 能进入下一次 Loader 组合。
- 为每个插件写简短 README：说明用途、依赖的 service/slot、配置项、已知限制。
- 插件行为有显著变化时更新 README，保持文档与代码同步。

## 沙箱与提权

- 构建、安装、运行、调试等操作可能被 dsh 沙箱拦截（如 `pnpm` 经 Desktop 的 Electron 包装器执行时，named-pipe IPC 在受限模式下被禁止）。
- 被拦截时**不要尝试用非标手段绕过**（改路径、直接调底层二进制、禁用安全检查、手工模拟产物等）。
- 正确做法：**通过工具（pwsh 等）以 `sandbox_permissions` 向用户申请提权**，使用最窄的足够宽模式（通常 `danger-full-access`），并附一句理由；由用户在审批弹窗中决定是否放行。
- 提权只针对被拦截的那条命令；审批被拒或环境限制（如沙箱端口/连接数限制）依然存在时，如实报告并换用合规手段（如降低并发重试），不反复尝试绕过。

## Git 提交规范

- 采用 Conventional Commits：`feat:` / `fix:` / `chore:` / `docs:` / `build:` / `refactor:`，正文按需使用。
- 外层仓库与 `src/<plugin>/` 子模块各自独立提交；改动跨越两者时分别编写 commit message。
- **不主动执行 `git commit`**：dsh 沙箱限制下由 dsh 生成的提交无法引用用户 GPG 签名，直接提交会绕过用户的签名配置。
- 完成代码改动后，将变更添加到暂存区（`git add`），编写 commit message，并在回复中提醒用户手动执行 `git commit`（以便 GPG 签名与提交钩子生效）。
- 提交前检查 `git status` 确认暂存范围正确；不要提交 `node_modules/`、`lib/` 等构建产物（已由 `.gitignore` 排除）。

### 预提交 checklist（每次提交前逐项核对）

**官方指南（DSH/Cordis）**
- [ ] 插件遵循 Cordis 插件模型：`inject` 显式声明依赖，注册均为 effect（`ctx.effect`/`ctx.on`/disposer）。
- [ ] 未依赖任何 Desktop 内部接口（`desktopRuntime`、`desktopPnpmBootstrap`、Electron、生成的 shim 等）。
- [ ] 跨环境插件：普通 DSH fallback 为权威实现，Desktop 探测走 `ctx.get`/嵌套 `ctx.inject`，不靠 `process.argv`/`ctx.baseUrl`/settings/`$DSH_HOME` 推断 profile。
- [ ] `package.json` 的 `dsh` 段、`exports`、构建产物路径与官方 client/bundle 契约一致。
- [ ] README 反映当前行为；显著行为变化已同步更新。

**社区约定（生态倡议 / Fabric 约束参考）**
- [ ] 组合优先：未覆盖或假设其他插件内部实现；通过官方 slot/service/patch 组合。
- [ ] 声明清晰：依赖的 service/slot 已显式声明，无运行时巧合依赖。
- [ ] 兼容优先：未破坏已有组合；升级保持向后兼容。
- [ ] Fabric 仅作约束参考：插件代码/README/package.json 未引用或 import Fabric 的 Draft 接口。

**插件市场（Community Market）约束**
- [ ] 版本为精确稳定 SemVer；无 GitHub URL、版本范围、`latest` tag 或 prerelease 安装目标。
- [ ] 未定义 `preinstall`/`install`/`postinstall`/`prepare` lifecycle script。
- [ ] `engines.node` 与 Desktop 内置 Node 兼容（当前 LTS ≥ 24）。
- [ ] `dsh.bundle.patch`（若声明）指向 package 内真实文件且不越界。
- [ ] `files` 白名单正确，发布 tarball 只含必要构建产物；`private` 标记与发布意图一致。
- [ ] README 未暗示"被收录/审核/推荐"。

## 参考文档（在 references/deepseek-harness-desktop/ 下）

- `docs/plugin-development.md` — 插件开发总览（现有接口 vs Draft 的区分）
- `dsh-plugin-desktop/docs/plugin-services.md` — `desktopProfiles` / `desktopPnpm` 契约
- `docs/plugin-ecosystem.md` — 生态倡议（组合优先 / 声明清晰 / 兼容优先）
- `deepseek-harness/docs/cordis-primer.md` — Cordis 五种核心思想、事件模式、waterfall 语义
- `deepseek-harness/docs/user/develop/basic/publish.md` — bundle 打包/发布教程（`dsh.bundle.patch` 官方依据）
- `deepseek-harness/apps/cli/reference/README.md` — profile 组合、`dsh plugin add` 与 bundles reconcile 行为
- `deepseek-harness/packages/boot/app-boot/src/profile.ts` — `DshBundleManifest` 与 profile/bundle 加载契约
- `deepseek-harness/packages/bundle/` — 官方内置 bundle（base / web-app / headless）的 patch 层实例
- `dsh-community-fabric/` — 仅作前瞻参考，不作为实现依据
- `dsh-community-market/docs/catalog-provider-contract.md` — 目录提供方契约（已实现的公开 v1）
- `dsh-community-market/docs/install-and-uninstall.md` — 安装/卸载边界与受管安装器规则
- `dsh-community-market/docs/market-shell.md` — 市场壳设计与产品边界
- `dsh-community-market/docs/catalog-adapter-guide.md` — 目录适配器接入指南
