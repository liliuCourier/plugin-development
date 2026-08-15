---
name: plugin-development
description: Create, modify, debug, or extend dynamic Cordis Plugins, including Host Services and Events, Client Slot and theme UI, Package-private Client-to-Host calls, dynamic Tools, version updates, approval failures, and runtime diagnostics. Use this Skill to route a user request to the correct platform and Inspect Provider, then define, run, repair, or roll back the Plugin. 动手前必须先在 GitHub 与插件市场中检索现成实现，参考分析后再决定复用或从零开发，避免重复造轮子和踩坑。
---

# Develop Dynamic Cordis Plugins

## 0. 需求澄清与轮子检索（MUST，动手前的前置流程）

收到任何插件开发需求后，**在写任何代码之前**，必须按以下四步流程走完，任何一步未完成不得动手：

**第 1 步：先 grill 用户（需求澄清）**

- 模型优先根据用户提及的功能和要求，设计问题逐层追问用户，把功能、需求的内容确定下来。
- 可参考 GitHub 上"基于需求问用户"的做法（例如 `grill-me` 这类需求盘问 skill 的工作流，见 PANGKAIFENG/ai-product-manager-skills、feiskyer/codex-settings、infometa/workbuddyskills 等仓库）：像面试官一样一轮轮问清目标场景、核心功能、边界与明确不做的事、与现有插件的关系、交付形态。
- 本轮结束产出一份简短的"已确认需求"清单，再进入检索。

**第 2 步：基于需求检索现成实现**

- 用 `web_search` 按上一步确认的需求关键词，在 GitHub（代码、README、Issues、PR）和插件市场/注册表（npm registry、Koishi/Cordis 插件市场、DSH 官方插件目录等）中做功能、需求匹配检索，并加上 `plugin`、`cordis`、`koishi`、`npm` 等限定词组合搜索。
- 找到相似或相近的仓库后逐一分析：架构、API 设计、优缺点、已知坑，判断可复用程度。

**第 3 步：再次 grill 用户（基于检索结果的第二轮澄清）**

- 把检索结论摆给用户，再一次明确：哪些需求可以参考/复用现成的轮子，哪些是切实需要新实现的功能需求。
- 与用户确认复用决策（直接使用 / 在其基础上改造扩展 / 必须从零实现）后再继续。

**第 4 步：产出设计**

- 生成一份完整的"待实现功能与需求设计"：功能清单、复用点与新建点、技术方案；必要时给出**一系列备选方案**供用户选择，并说明各自的取舍。

检索结论与最终设计必须写进交付回复的开头，让用户看到完整的澄清与调研过程。目的：先问清需求，再避免重复造轮子和踩坑，优先复用成熟方案。

First determine whether a capability belongs on Host or Client, then query the real interface before writing code. Never infer a complete API from a Service name, Event payload, Slot props, theme token, or example.

## Standard workflow

1. Call `cordis_inspect_list` to obtain the Providers, methods, and schemas currently registered on Host and Client.
2. Select the smallest set of `cordis_inspect_query` calls needed to read the exact Services, Events, Builtins, Slots, Theme tokens, or Tools that the implementation will use.
3. For a new Plugin, design its first Package. To modify an existing Plugin, first use `cordis_inspect_self(pluginId, packageId)` to read the base source and diagnostics.
4. Write plain JavaScript in `code.host`, `code.client`, or both, then call `cordis_define`.
5. Call `cordis_run` with the final `pluginId` and `packageId` returned by define.
6. Handle approval, waiting, Client loading, and render failures from the Run card, steering messages, or `cordis_inspect_self`.
7. Use `cordis_stop` to disable the Plugin temporarily. Use `cordis_undefine` only when it is no longer needed.

Do not wait in the same turn for user approval or asynchronous browser results. After `cordis_run` returns `awaiting-approval` or `starting`, end the current Tool flow and wait for the system to report the final outcome through state updates and steering.

## Tool usage guidance

| Tool | Use it when | Do not |
| --- | --- | --- |
| `cordis_inspect_list` | Discover current Host/Client Providers and method schemas in one call; refresh after the runtime capability directory changes | Hard-code Provider names and skip list; treat a manifest as business data |
| `cordis_inspect_query` | Confirm exact Service methods, Event modes, Builtins, Slots, tokens, or Tool schemas before writing code | Use it instead of calling a real Service from the Plugin; assume a Client query will finish without a responding page |
| `cordis_inspect_self` | List current Plugins, inspect version pointers, or read exact Package source and runtime diagnostics | Fetch all source just to build a list; use it to modify or start a Plugin |
| `cordis_define` | Create a Plugin's first version or append an immutable Package to an existing Plugin; let the user preview the code first | Expect define to execute `apply`, request approval, or update current |
| `cordis_run` | Activate an exact Package; use `run` for first activation, restart, or rollback, and `update` to switch versions | Use `run` to switch versions implicitly; treat pending or starting as success |
| `cordis_stop` | Pause current effects while preserving Packages, grants, and version pointers for later use | Use stop to mean permanent deletion |
| `cordis_undefine` | Permanently remove a Plugin and all of its Packages and clear historical business views | Call it while rollback, inspection, or restart is still needed |

## Choose a platform

| Requirement | Preferred platform | Inspect first |
| --- | --- | --- |
| Files, commands, processes, or networking | Host | `fs`, `bash`, `subprocess`, `pty`, and `web` in `Service.listService` |
| Agents, durable Session data, or Host lifecycle | Host | The relevant Service and `Event.listEvents` |
| Register a dynamic Tool callable in the next model step | Host | `harness` in `Builtin.listBuiltins`, plus `Tool.listTools` |
| Page theme, layout, or current page state | Client | `Theme.listTokens` and Client `Service.listService` |
| Conversation Snapshot or session/workspace lists | Client | The target Slot's standard props and owner props |
| Settings pages, sidebars, input areas, overlays, or Tool cards | Client | `Slots.listSubTree` |
| Fetch on Host and display on Client | Both | Host Service + `harness.handle`; Client Slot + `host.call` |

Prefer the capability closest to the data owner. If Slot props already provide the Conversation Snapshot, do not fetch it again through Host. If only the Package's own styles need to change, do not override the global theme. If only a small entry point is needed, do not replace an entire product UI region.

## Provider navigation

Select methods from the actual `cordis_inspect_list` result. Common initial methods include:

- `Service.listService`: without `service`, returns every callable Service with its purpose and exact method signatures. Query the selected `service` again for access rules, structured method descriptions/parameters/returns, and only its referenced types.
- `Event.listEvents`: without `event`, returns every Event with its purpose, dispatch mode, and exact listener signature. Query the selected `event` again for its structured listener contract and only its referenced types; a Waterfall listener must call `next()`.
- `Builtin.listBuiltins`: returns evaluator-provided symbols and signatures that cannot be obtained through `ctx.get()`.
- `Slots.listSubTree`: without `root`, returns compact live trees with each Slot's purpose, kind, scope, registration keys, replacement risk, and children. With an exact `root`, it also returns that selected Slot's full contract, props, and current occupants while keeping descendants compact.
- `Theme.listTokens`: returns theme tokens that may currently be queried and overridden; it does not modify the theme.
- `Tool.listTools`: returns Tool schemas actually visible to the current Agent, including dynamically registered Tools.

Provider names, methods, and inputs must come from the current list result. The Service/Event Catalog describes which interfaces this version permits; it does not guarantee that a Service is currently mounted. At runtime, use real Services and Events rather than caching or displaying Catalog query results.

## Execution environment

Both `code.host` and `code.client` are plain JavaScript function bodies that return a Cordis Plugin. They are not compiled by TypeScript, JSX, or a bundler.

Do not use:

- `import`, `require`, TypeScript types, `as`, decorators, or JSX;
- globals not confirmed by `Builtin.listBuiltins`;
- guessed access to `window`, `document`, `process`, `Buffer`, `fetch`, or native timers.

Client React code must use `React.createElement(...)`.

Correct:

```js
return {
  apply(ctx) {
    const slots = ctx.get('slots')
    if (slots === undefined) return
    slots.inject('tool.view.cordis', () => slots.register(
      { name: 'tool.view.cordis', key: 'self' },
      () => React.createElement('div', null, 'Hello'),
    ))
  },
}
```

Incorrect:

```jsx
return {
  apply(ctx) {
    return <div>Hello</div>
  },
}
```

JSX is not the only problem in this example. `apply()` registers lifecycle contributions and cannot return a React Element as the Plugin result. UI must be registered in a queried Slot.

## Access Services

Read optional capabilities with `ctx.get(name)` by default and handle their absence:

```js
return {
  apply(ctx) {
    const service = ctx.get('serviceName')
    if (service === undefined) return
    service.someMethod()
  },
}
```

Declare `inject` only when a Service is a hard dependency and the Plugin must enter waiting until Cordis reactivates it after the Service appears:

```js
return {
  inject: ['requiredService'],
  apply(ctx) {
    ctx.requiredService.someMethod()
  },
}
```

Do not overuse `inject` merely to avoid an `undefined` check. Do not access `ctx.requiredService` without declaring the injection; the Guard rejects undeclared dependencies.

## Manage side effects

Every contribution must be removed after the Plugin is stopped, updated, or removed. Prefer Cordis lifecycle APIs:

- Use `ctx.on()` to register Event listeners.
- Use `ctx.effect()` to own an external subscription that returns a disposer.
- Retain disposers returned by Cordis Service, Tool, Slot, timer, and theme APIs.
- Do not create process-wide or page-wide side effects at module scope or outside `apply()`.

Recommended:

```js
return {
  apply(ctx) {
    const service = ctx.get('serviceName')
    if (service === undefined) return
    ctx.effect(() => service.subscribe((value) => {
      console.log(value)
    }))
  },
}
```

If `subscribe()` does not return a disposer, first query whether the Service provides a supported cleanup mechanism. Do not assume unload automatically removes arbitrary third-party callbacks.

## Host and Client timers

On both platforms, the timer is a Service named `timer` with the same interface; it is not a Builtin. Query `{ "service": "timer" }` through the corresponding platform's `Service.listService` before using it. Declare `inject: ['timer']` before using the timer mixin.

One-shot delay:

```js
return {
  inject: ['timer'],
  apply(ctx) {
    const onClick = () => {
      ctx.timeout(() => console.log('done'), 300)
    }
    // Pass onClick to a queried Slot UI.
  },
}
```

Periodic work in a React component:

```js
return {
  inject: ['timer'],
  apply(ctx) {
    function Clock() {
      React.useEffect(() => ctx.interval(() => console.log('tick'), 1000), [])
      return React.createElement('div', null, 'Running')
    }
    // Register Clock in a queried Slot.
  },
}
```

Incorrect:

```js
return {
  apply(ctx) {
    ctx.timeout(() => console.log('invalid'), 300)
  },
}
```

```js
setTimeout(() => console.log('invalid'), 300)
```

The first example does not declare the timer hard dependency. The second uses a global timer that does not exist.

## Listen to Events

Query the Event Provider first to confirm the platform, parameter order, return value, and `mode`.

Ordinary emit Event:

```js
return {
  apply(ctx) {
    ctx.on('some/event', (payload) => {
      console.log(payload)
    })
  },
}
```

The last parameter of a Waterfall Event is `next`. Unless the listener intentionally stops downstream processing, it must call and return it:

```js
return {
  apply(ctx) {
    ctx.on('some/waterfall', (payload, next) => {
      console.log(payload)
      return next()
    })
  },
}
```

## Register Client UI

Query `Slots.listSubTree` without `root` to choose a target from the compact purpose and topology tree, then query the exact Slot with `root` before writing its registration. The exact result determines:

- the Slot's purpose in the layout;
- whether its registration protocol is `single`, `list`, `keyed`, or `chain`;
- registration options;
- scope standard props and business owner props;
- current occupants, replacement risks, and descendant Slots.

Use `ctx.get('slots')` and handle its absence. Then use `slots.inject` to wait for the Slot declaration and call `slots.register` inside the callback:

```js
return {
  apply(ctx) {
    const slots = ctx.get('slots')
    if (slots === undefined) return
    slots.inject('target.slot', () => slots.register(
      { name: 'target.slot', id: 'my-view' },
      (props) => React.createElement('div', null, String(props.someValue)),
    ))
  },
}
```

`ctx.get('slots')` does not require an injection. Do not rewrite it as `ctx.slots` unless `inject: ['slots']` is declared:

```js
return {
  apply(ctx) {
    ctx.slots.register({ name: 'target.slot' }, () => null)
  },
}
```

Do not guess an `id`, `key`, selector, or props before querying the Slot protocol. Do not default to root-level `root`, `sidebar`, `conversation`, or `details` Slots; replacing an entire occupant also removes the descendant Slots it declares.

### Settings pages

A full settings UI should usually register its own section through `settings.section` to obtain a complete content area. `settings.general.item` is only appropriate for one compact, general-purpose preference. Query the actual subtree, options, and props for both, then select the narrowest entry point that is still sufficient.

Dynamic Plugins are temporary and process-local, so their settings UI does not need persistent storage. Do not add durable settings or another persistence mechanism for it. Register the UI in the appropriate settings Slot and keep any transient interaction state in memory for the lifetime of the Plugin.

### Session and page data

A session-scoped Slot may provide `useSession`, `useSessions`, `useWorkspaces`, `useProjection`, input state, or actions through standard props. Follow the query result and prefer owner or standard props directly; do not add a Host RPC for data already present there.

Select only the fields that the UI actually needs. Do not copy or render an entire Conversation Snapshot, Session, Tool call, or Slot props object.

### Cordis Run-specific panel

To place interactive UI in the latest `cordis_run` card, register `tool.view.cordis` with `key: 'self'`:

When the feature needs user interaction tied to this Package's result, this region is often a good fit because it keeps the controls in the conversation flow beside the Run card. It is not the default target for every Client UI: settings, sidebars, message actions, and overlays should use their own queried Slots when those locations better match the feature.

```js
return {
  apply(ctx) {
    const slots = ctx.get('slots')
    if (slots === undefined) return
    slots.inject('tool.view.cordis', () => slots.register(
      { name: 'tool.view.cordis', key: 'self' },
      (props) => React.createElement('div', null, `Package ${props.packageId}`),
    ))
  },
}
```

At runtime, `self` binds to `pluginId + packageId`. Do not include `pluginRunId` in the key. When the same Package runs multiple times, the latest Run card hosts the UI and older cards automatically degrade.

### Ordinary Tool cards

To customize the call card for an ordinary model Tool, query `tool.call.toolview`. Its key is the Tool name; registering an existing key may replace the product's default card. When customizing only a newly added Tool, first verify its schema with `Tool.listTools`, then query the complete `ToolCallOwnerProps`.

### Overlays and local entry points

- For toasts, status notices, and frame-wide overlays, query `shell.overlay` first; observe its pointer-events and ordering rules.
- When the selected target is a global overlay Slot, decide whether the UI should be draggable, how the user shows and hides it, and which existing layers it must cover or remain below.
- For small sidebar actions, prefer additive inner Slots such as `sidebar.footer.action`; do not replace the entire sidebar.
- For supplementary content after a conversation turn, query `conversation.chat.turnTail` and register according to its returned chain selector and fallback rules.

## Themes and styles

Determine the scope of the change first:

1. Global theme: first query `Theme.listTokens`, then query `{ "service": "theme" }` through Client `Service.listService`. Supply light and dark values for each override as required by the query, and retain the returned disposer.
2. The Package's own components: use `styles.insert(css)` and prefer theme CSS variables for colors.
3. New visible content: choose a Slot first, then decide between local CSS and global tokens.

Do not manipulate `document.body`, `window`, or hard-coded product DOM selectors. The theme Service changes tokens but does not create UI. Slots create UI but do not replace the theme system.

## Call Host from Client

Host registers a Package-private method with `harness.handle(method, handler)`, and Client invokes it with `host.call(method, args)`. This is Client→Host JSON RPC.

Host:

```js
return {
  apply(ctx) {
    harness.handle('read-state', async (args) => {
      return { value: args.key }
    })
  },
}
```

Client:

```js
return {
  async apply(ctx) {
    const result = await host.call('read-state', { key: 'demo' })
    console.log(result.value)
  },
}
```

Arguments and return values must be lossless JSON. Do not pass functions, React elements, class instances, Contexts, Services, or other runtime objects; return `null` when there is no response data. Do not register a public Remote Service or use `ctx.remote` for Package-private communication.

## Register a dynamic model Tool

Host can use `harness` to register a Tool callable in the next model step. First query the current `harness` signature with Host `Builtin.listBuiltins`, then inspect existing Tool names and schemas with `Tool.listTools` to avoid conflicts.

Tool arguments and return values must be JSON-compatible. `execute` owns the business result; render and presentation own only what the model and native UI see. Tool registration must belong to the current Plugin Fiber so it is automatically removed after stop or update.

## Handle internal live data

Service instances, Event payloads, Slot props, Session and Conversation Snapshots, Tool state, and other DSH/Cordis objects are internal live data.

Do not:

- call `JSON.stringify` or `structuredClone` on these objects or their descendants;
- recursively enumerate, fully copy, or display them as a whole;
- place Host objects in the Package's long-lived state or RPC return values.

Read only the leaf fields required by the current feature. Extract the minimum strings, numbers, booleans, and other scalar values before constructing owned JSON.

## Versions, approval, and repair

- A Plugin is the stable instance identified by `pluginId`.
- A Package is an immutable code version identified by `packageId`.
- Every activation attempt has its own `pluginRunId`.
- `currentPackageId` is the latest successful version; it does not imply that the Plugin is currently running.
- `nextPackageId` is the target awaiting approval, activating, awaiting Client activation, or most recently failed.

Choose the `cordis_run` mode as follows:

| Current state | Target | mode |
| --- | --- | --- |
| No current | Any Package under the Plugin | `run` |
| Has current | The same Package | `run` |
| Has current | A different Package | `update` |
| Update failed | `nextPackageId` | `update` to retry |
| Update failed | `currentPackageId` | `run` to roll back |

An unauthorized Client Package returns `awaiting-approval`. A single check mark authorizes only the current Package; double check marks authorize future versions of the same Plugin. A grant remains after a technical runtime failure. An authorized Package returns `starting` and completes asynchronously in the browser.

After a technical failure:

1. Use `cordis_inspect_self(pluginId, packageId)` to read the failed version's source and exact diagnostics.
2. If the error involves an unknown capability, list and query the corresponding Provider again.
3. Define a new Package under the same Plugin; do not overwrite the failed Package.
4. Run again with the new `packageId` and the correct mode.

Do not retry automatically after the user rejects approval. A failed update does not automatically restore the old physical Run; explicitly run current when recovery is required.

## Modify @pluginId

When the user identifies a target with `@pluginId`, do not create another Plugin. The injected context contains only identity, version pointers, and the default base Package, not source code.

Modify it as follows:

1. Read the base Package with `cordis_inspect_self(pluginId, packageId)`.
2. Preserve the Host or Client half that does not need to change and modify only the target code.
3. Call `cordis_define` with `plugin.kind: 'existing'` and the original `pluginId`.
4. Use the returned `packageId`; when current exists, activate the new version with `update` in the usual case.

If the reference is unavailable, explain that the Plugin was removed, belongs to another Session, or was lost on process restart. Do not create a same-named replacement.

## Common failure checks

| Failure | Check first |
| --- | --- |
| `service "x" is not declared` | Whether code uses `ctx.x` without declaring `inject: ['x']` on the Plugin object; switch to `ctx.get('x')` with an absence check or declare a true hard dependency |
| `cannot get property "timer" without inject` | Query the timer Service and declare `inject: ['timer']` |
| Client parse failure | Whether the code uses JSX, TypeScript, import, or an unavailable global |
| Slot registration failure | Whether the live subtree was queried, the Slot exists, and options, key, or selector satisfy the returned protocol |
| UI loads but the page reports an error | Inspect the `client-render` diagnostic and stack; the error belongs to an exact Run, so define a new Package to repair it |
| `host.call` failure | The Host handler name, current `pluginRunId`, JSON arguments, and real Service dependencies inside the handler |
| Update failure | Preserve current/next semantics; repair next and update, or run current to roll back |

## DSH 静态（文件系统）插件安装排查（2026-08 实踩）

DSH profile 插件的正式安装入口是 `dsh plugin --profile <name> <args...>`（`apps/cli/src/plugin.ts`）：把剩余参数原样转发给 pnpm（cwd = profile 目录，Windows 走 shell），成功后再 reconcile `dsh.profile.bundles`。手工安装时按此规则排查：

| 错误/现象 | 原因 | 解法 |
| --- | --- | --- |
| `[EPERM] ... open 'C:\Users\20112\.dsh\...'` | dsh/pnpm 写 `~/.dsh`（工作区外），被会话文件沙箱拒绝 | 用 `sandbox_permissions` 升级重跑同一条命令 |
| `ERR_PNPM_UNEXPECTED_STORE`：node_modules 来自 store A，pnpm 要用 store B | profile 依赖之前用 `--store-dir D:\...\.pkg-cache\pnpm-store\v11` 装的，默认 store 不一致；`npm_config_store_dir` 环境变量传不进 `dsh plugin` 的 spawnSync | 直接在 `dsh plugin ... add <pkg> --store-dir "D:\desktop\DsHarness\.pkg-cache\pnpm-store\v11"` 透传；或先 `pnpm config set store-dir` |
| `[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: sharp@...` | pnpm 默认阻止依赖的构建脚本，且 `dsh plugin` 因此返回非零、**不执行 reconcile** | 在 profile 的 `pnpm-workspace.yaml` 加 `allowBuilds: { sharp: true }`（pnpm 会自动写入 `sharp: set this to true or false` 占位，改成 `true`），再重跑 |
| 命令超时（如 sharp 首次构建拉二进制） | pnpm 静默挂起 | 用 `run_in_background` 跑，完成后用 `job_output` 收结果；装完的包在 profile 自己的 `node_modules`（`~/.dsh/profiles/<name>/node_modules`），不是共享的 `~/.dsh/profiles/node_modules`（后者是 `healProfilesModuleFallback` 给 DSH 核心包建的符号链接层） |
| 装进依赖了但 profile 不加载 | 无 `dsh.bundle.patch` 的插件（如 `dsh-vision-router`）不会进 `dsh.profile.bundles`，`reconcilePlugins` 只收 bundle | 在 `~/.dsh/profiles/<name>/cordis.patch.yml` 的 `insert` 列表加 `- id: <name>` / `name: <name>`（格式同 `dsh-chat-outline` 示例），再用 `dsh --profile <name> --dump-config` 验证合成树 |
| 验证插件解析 | — | 用 `createRequire('<profile>/package.json').resolve('<pkg>')` 从 profile 锚点解析，确认 Node 能找到 index.js 及全部依赖 |
| `notes/list: transport failure for /api/notes/list: HTTP 404`（host Remote 服务在浏览器调用 404） | 插件的 `@deepseek-ai/*` 依赖从**仓库自己的 node_modules**（npm install 时被 npm 实体化出的拷贝）解析，与 dsh web 进程里 gateway 用的**官方 runtime 包**是两份不同实例；`@Remote` 装饰器把方法标记写进**模块私有 WeakMap**，gateway 的 SRC `collectSrcClaims` 用官方 `remoteMethods()` 读官方 WeakMap → 读不到插件拷贝写的标记 → `claimsEndpoint` 返回 false → 客户端 RPC 退回 HTTP 传输 → 404 | 让插件与 gateway **共享同一份** `@deepseek-ai/*`：删掉仓库真实 `node_modules`，重建为指向 `~/.dsh/profiles/node_modules` 的 **junction**（`New-Item -ItemType Junction`），再重启 dsh web 使 host 插件用官方包重新加载。离线验证脚本在仓库目录能过是因为它只解析仓库侧那一份，自洽不代表运行时两方同源 |
| host Remote 修复后仍 404 | 运行中的 dsh web 进程已缓存旧模块（link 指向的仓库代码 + 旧 node_modules 解析），文件系统修复不会让已加载模块换源 | 重启 dsh web（会断开当前会话），或在仓库目录跑 `verify-host.mjs` 确认 junction 下解析路径已指向官方 runtime 包后再重启 |

安装完成 ≠ 生效：profile 配置是启动时读取的，正在运行的 `dsh web` 需要重启才加载新插件；重启会断开当前会话。

## 维护说明

本文件 = 官方 `cordis-plugin-development` 技能底稿 + 用户自定义强化规则（第 0 节"需求澄清与轮子检索"四步流程）。官方底稿更新时，同步替换对应章节；自己的约定（命名规范、常用骨架、踩过的坑）继续追加到对应小节或新增小节。本文件是长期资产，每次对话中确认的新规则都要回写到这里。
