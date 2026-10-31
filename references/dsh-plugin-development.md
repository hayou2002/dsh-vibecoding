# DSH 插件与扩展开发

把「在 DeepSeek Harness 上做扩展」的现场知识固定成一条可执行路径：**先判形态 → 再摸 profile → 再跑最小可运行 → 最后才打包安装**。不要一上手就写包清单。

**来源可信度分层**（按此顺序取证，冲突时上位覆盖下位）：

| 层级 | 来源 | 用途 |
|---|---|---|
| 1 权威 | 当前宿主的已安装包文档 `node_modules/@deepseek-ai/*/README.zh.md` | 本机版本的真实字段、API、默认值与限制 |
| 2 权威 | 官方文档/源码仓库中的 `docs/`（`cordis-primer`、`config-catalog`、`cookbook/adding-a-package`）、changelog | 概念模型与作者格式 |
| 3 线索 | 社区文章、教程、issue | 上手顺序与踩坑；**必须在本机版本复现后才采信** |

已安装包文档描述的是**宿主当前版本**的行为；`docs/` 与社区文章描述的是某一时刻的形态，常用于「这个 API 怎么用」的对照，不是「这个版本为什么坏了」的证据。

## 0. 先判形态：五类扩展点，别选错层

| 形态 | 载体 | 典型需求 | 关键接口 |
|---|---|---|---|
| 函数插件 | 一个 cordis 插件模块（`apply(ctx, config)`） | 改行为、加服务、挂事件、启停功能 | `export const name` / `inject` / `Config`（schemastery）、`apply` |
| 工具插件 | `@deepseek-ai/dsh-tools` 的 `ctx.tools.register(defineTool(...))` | 给模型加一个可调用的能力 | `defineTool`、参数 schema DSL、`output.render` |
| 客户端/UI 插件 | 带 `dsh.client`（如 `{ platform: 'web' }`）的包 | 侧栏页、配置页、卡片外观、展示元信息 | `DshClientManifest`、`tool.call.toolview`、导出的 `./icon` 与 locale `meta` |
| 组合包 bundle | 包清单 + `cordis.patch.yml`（可多层） | 把一组插件与配置一次装进 profile | `dsh.bundle.patch`、`dsh.profile.bundles` |
| 非插件扩展点 | 技能 / Agent 预设 / MCP 服务器 / 配置覆盖 | 只想加流程、加工具面、加外部服务 | 优先用这些，成本更低 |

**选择顺序是成本顺序**：能用一个技能或一段 profile 配置解决，就不要写插件；插件里能用函数插件解决，就不要做客户端包；只有当需要跨会话持久、影响工具面或改宿主行为时才上升形态。

## 1. 摸清现场（只读先行，不改任何文件）

```bash
echo "$DSH_HOME"                      # 宿主主目录
ls "$DSH_HOME/profiles/"              # 已有 profile
ls "$DSH_HOME/profiles/<name>/"       # cordis.yml / cordis.patch.yml / package.json / pnpm-lock.yaml
python - <<'PY'
import json, os, pathlib
p = pathlib.Path(os.environ["DSH_HOME"], "profiles", "<name>", "package.json")
d = json.loads(p.read_text(encoding="utf-8"))
print(json.dumps(d.get("dsh", {}).get("profile", {}).get("bundles", []), ensure_ascii=False, indent=2))
PY
```

要看清的五件事：

1. **profile 名与目录**：`$DSH_HOME/profiles/<name>`；`dsh --profile <name>` 启动它，`dsh web` 是 web profile。
2. **组合包层**：`package.json` 里有序的 `dsh.profile.bundles` 决定叠加顺序，顺序影响配置优先级。
3. **覆盖层**：`cordis.patch.yml` 是用户覆盖，比组合包层优先级高；插件启停通常只改这里的 `disabled`。
4. **配置树**：`cordis.yml` 是 Cordis 的配置项列表（每项 `name` 可为相对路径或 npm 包名），loader 逐条挂载。
5. **运行实例状态**：能连上时读 `pluginInventory/list`，按加载顺序看条目 id、模块标识、有效启用状态与根 Fiber 阶段（`pending` / `loading` / `active` / `failed` / `unloading` / `null`）。它是**只读快照**，不能用来改插件，也没有历史。

现场与文档冲突时以现场为准；现场与默认值冲突且无法安全判断时，只出只读报告，不猜着改。

## 2. 最小可运行：先在 profile 里直挂，不进包

不打包、不发布，先在**临时 profile** 里把行为跑通，能省掉一整轮打包-安装-重启的往返。

```ts
// tmp/hello-plugin/hello.ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello'

export function apply(ctx: Context) {
  ctx.logger.info('hello from dsh plugin')
}
```

```yaml
# tmp/hello-plugin/cordis.yml —— Cordis 配置项列表，逐条挂载
- name: './hello.ts'
```

```bash
cd tmp/hello-plugin
dsh --profile headless "报一下当前时间"      # 全新持久会话，打印最终答案后退出
```

带配置与依赖注入的函数插件形态：

```ts
import type { Context } from '@deepseek-ai/cordis'
import z from '@deepseek-ai/schemastery'

export const name = 'my-plugin'
export const inject = ['tools']                 // 声明依赖的 ctx 服务
export interface Config { greeting: string }
export const Config: z<Config> = z.object({ greeting: z.string() })

export function apply(ctx: Context, config: Config) {
  ctx.logger.info(config.greeting)
}
```

**最小化的判据**：日志/工具出现、退出码为 0、行为可复现。跑不通就先解决加载（路径、模块解析、`inject` 缺服务、TypeScript 装载），不要接着写清单。

## 3. 工具插件：把能力交给模型

在任何 agent 调用工具的地方挂载 `@deepseek-ai/dsh-tools`：它提供 `ctx.tools` 注册表；**注册一个工具就足以让它可见**，注册表会把 schema 送进系统提示词组装。

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

declare const ctx: Context

ctx.tools.register(defineTool({
  name: 'read_file',
  description: 'Read a file from disk.',
  parameters: {
    path: { type: 'string', required: true, description: 'Absolute file path' },
    offset: { type: 'number' },
    limit: { type: 'number' },
  },
  output: {
    schema: { type: 'string' },
    render: (_args, value) => [{ type: 'text', text: value }],
  },
  async execute(args, exec) {
    // args 已按 schema 收窄：{ path: string; offset?: number; limit?: number }
    return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
  },
}))
```

要点：

- **参数在执行前校验**，无效输入变成普通错误结果，不是崩溃。
- `execute` 只返回 `output.schema` 声明的 JSON 值；渲染与记录由 `render` 决定。规范值属于执行过程，Session 记录的是渲染后的内容与可选展示元数据——需要禁止程序化读取时要拦执行或替换值，而不是改渲染。
- schema DSL 支持 `string`、`number`、`integer`、`boolean`、`null`、`array`、`object`、仅供作者使用的 `json`、恰好匹配一个分支的 `oneOf`；`InferValue` 在 16 层容器内保留精确类型，之后加宽为 `JsonValue`。
- **改动工具面之前先想权限**：`ctx.tools.restrict(filter)` 对继承的全局工具加允许/拒绝掩码（掩码取交集，作用域注册仍可见，dispose 时解除）；`ctx.tools.guard(guard)` 在 `tools/pre-execute` waterfall 之后注册**单调**守卫，返回的理由会拒绝调用且后续监听器无法翻案。
- 相关流水线事件：`tools/pre-execute`（允许/拒绝/询问）、`tools/execute`（为超时或重试包装分发）、`tools/post-execute`（检查或替换结果）、`tools/result`（观测冻结的最终结果）。
- **呈现模式**：`mode: native`（默认，每个可见 schema 直给模型）或 `mode: ptc`（只有 `run_code` 加一份生成 SDK；要求已组合的 `ctx.ptcRuntime` 且有该语言的 SDK 渲染器）。`maxParallelSubCalls` 默认 `10`，设 `1` 恢复严格串行。
- **卡片外观**：工具可以保留纯函数 `presentCall()` / `presentResult()` 给宿主本地消费者；内置 Web Client 不消费这些值，而是按 `tool.call.toolview` 选 renderer，从原始参数、结果内容、失败状态与持久 metadata 派生卡片属性。
- `run_code` 程序**启动时固定**当时的工作目录；后续工具调用会跟随目录切换，运行中的程序仍保留启动目录。

## 4. 打包成组合包

包根 `package.json` 里声明清单（类型 `DshPackageManifest`）：

```json
{
  "name": "example-dsh-plugin",
  "version": "1.0.0",
  "engines": { "node": ">=24", "dsh": "0.1.5-alpha.1" },
  "dsh": {
    "manifestVersion": 1,
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": { "platform": "web" }
  }
}
```

| 字段 | 说明 |
|---|---|
| `name` / `version` | 必填；本地 profile 读取方接受 `Partial<DshPackageManifest>` |
| `dsh.manifestVersion` | 声明的清单格式版本，当前为 `1`；**独立于** npm 版本与 Session 格式版本 |
| `dsh.bundle.patch` | 相对包根的 patch 文件路径，或**有序路径列表**；launcher 按顺序作为同一组合包层应用 |
| `dsh.client` | 客户端领域清单（如 `{ platform: 'web' }`） |
| `engines.dsh` | SemVer 范围，可与 `engines.node` / `engines.npm` 并列；版本范围写窄会挡住安装 |
| `icon` / 导出 `./icon` | 包根清单 icon、导出 `./icon`，或子路径插件的 `<子路径>/icon`，用于卡片与详情 |
| locale `meta` | 展示标题与描述先读导出 locale `meta`，缺失时回退包根 `package.json`（**子路径插件从不读取包根**），标题最终用完整包名或模块名 |

作者格式（展示元信息、包结构）以官方仓库 `docs/cookbook/adding-a-package.zh.md` 为准。**注意**：`docs/`、`examples/`、`.agents/notes/` 这些路径在已安装的 `resources/dsh/` 里并不存在（那里只有 `node_modules` 与锁文件），不要按文档路径去找文件；要引用它们时说清来源，并用已安装 README 的字段表核对。

## 5. 安装、启停与合并语义

三条可用路径，按现场条件选：

1. **Web 侧栏 → 插件**：管理当前 profile 已安装的组合包，启停组合包及其行、安装随附但默认关闭的官方组合包、看 pnpm 输出、停止一次运行、卸载（需确认）。首次打开通过 `api-remotes` 读清单；**没有受管 profile 的 Host 上该页不可用**。
2. **`plugin_manager` 工具**：操作与上面一致，在 Creator 模式启用、其他预设默认禁用；**每个工具操作要求 `danger-full-access` 或本次调用批准**，批准不改变会话权限模式。
3. **CLI 转发 pnpm**：`dsh plugin --profile <name> <pnpm args>`，在 profile 目录里管理该 profile 的插件依赖。

不用 Agent 预设的部署，在 profile patch 里启用插件管理工具：

```yaml
- id: tool-plugin-manager
  disabled: false
```

合并与优先级（改这里最容易出错）：

- **插件开关**只更新 profile `cordis.patch.yml` 中**最后一条**匹配覆盖项的 `disabled`，没有匹配项就追加；匹配依据条目 id 加覆盖项声明的模块名。
- 如果组合包层与其余 profile patch **已经**让条目处于所请求的状态，只含 `id` 与 `disabled` 的覆盖项会被**移除**且不追加——profile patch 只记录与组合包不同的设置（比较不计 home 与单次启动 patch）。
- **组合包开关**改的是 `package.json` 的有序 `dsh.profile.bundles`：关闭保留依赖；开启会**追加到列表末尾**，可能改变配置优先级；新安装的组合包默认启用；home 与单次启动 patch 优先级更高。
- 已选但加载不了的组合包仍在 `listBundles` 中并带 `error`；`enabled` 表示**保存的选择**，不是加载成功；损坏的组合包无法启用。
- **YAML 启用 HMR 时**配置变化立即生效；未启用时运行中的组合保留到重启。profile 改动影响**使用该 profile 的全部会话**。

改完的验收不是「点了开关」，而是：`pluginInventory/list` 里该条目的有效状态与根 Fiber 阶段符合预期，日志无 `failed`。

## 6. 调试与观测

| 现象 | 先看什么 |
|---|---|
| 插件没生效 | `pluginInventory/list` 的有效启用状态（含被禁用祖先组）与 Fiber 阶段；是否只在别的 profile 里装了 |
| 改了配置没反应 | 该 profile 是否启用 HMR；是否需要重启；改动是否只是「与组合包相同」因而被移除 |
| 加载报错 | 组合包 `error` 字段、pnpm 输出、模块标识是否与 `package.json` 声明一致 |
| 工具不出现在模型侧 | 是否 `ctx.tools.register` 到了同一个 ctx；`mode` 是否为 `ptc` 导致只暴露 `run_code`；是否被 `restrict` 掩码挡掉 |
| 权限/审批异常 | 工具操作是否要求 `danger-full-access` 或单次批准；依赖构建脚本是否单独批准过 |

## 7. 验收清单（可判定，不要用「看起来好了」结案）

- [ ] 形态选择有理由（为什么不是技能/预设/MCP/纯配置）。
- [ ] 最小可运行在**当前宿主版本**上真实跑通，记录了命令与退出码。
- [ ] 包清单字段与已安装 README 的字段表逐项对上；`engines.dsh` 覆盖当前宿主版本。
- [ ] 安装后 `pluginInventory/list` 状态正确，无 `failed`。
- [ ] 工具类插件：参数校验、错误结果、`output.schema` 与 `render` 一致；权限掩码与守卫按预期。
- [ ] 客户端类插件：标题/描述/icon 的解析路径符合包结构（子路径插件别指望读包根）。
- [ ] 卸载或关闭能回到干净状态；profile patch 没有留下多余覆盖项。
- [ ] 报告写清改了什么、验证了什么、**未覆盖**的部分与遗留风险。

## 8. 边界与坑

- **宿主代码在宿主进程内运行，不受工作区沙箱限制**：插件能力等于宿主能力，写文件、发网络、改配置都按宿主权限生效。给用户交付前说清这一点。
- **依赖构建脚本仍需单独批准**；工具操作需要 `danger-full-access` 或单次批准。
- **profile 是共享面**：启停会影响该 profile 的**全部**会话，且跨会话持久。
- `desktop` 名称保留给 Electron，CLI 拒绝对它的启动与配置 dump。
- **不要手改生成物**（`moduleFallback`、`configTrees`、`sessionFormatMigration`、锁文件）；要改就改 `cordis.yml` / `cordis.patch.yml` / `package.json` 声明层。
- 组合包层顺序影响优先级：开新包可能悄悄改变旧覆盖项的胜出者，装完要复核。
- 社区文章（教程、实战帖）能给上手顺序，但**未经本机验证不作为结论**；与本机 README 冲突时以 README 为准。
- 版本漂移：`engines.dsh`、字段默认值与「实验性」状态都会随版本变；写之前用当前安装包的 README 与配置目录核对，别沿用旧文章。

## 9. 来源

- 本机 `resources/dsh/node_modules/@deepseek-ai/` 下的 `dsh`、`dsh-package-manifest`、`dsh-plugin-manager`、`dsh-host-plugin-inventory`、`dsh-tools`、`dsh-client-ui-plugin-manager` 的 `README.zh.md`（当前宿主版本，权威）。
- 官方仓库 `docs/cordis-primer.md`、`docs/config-catalog.zh.md`、`docs/cookbook/adding-a-package.zh.md`（作者格式与配置穷尽表；本地安装树中不存在，作对照用）。
- 社区教程与实战文章（Cordis 插件骨架、`cordis.yml` 直挂、headless 跑通顺序）：仅作线索，采信前在本机复现。
