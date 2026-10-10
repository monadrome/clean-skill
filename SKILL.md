---
name: clean-skill
description: Read-only audit of which agent skills are actually used, judged by usage frequency rather than token cost, with concrete recommendations on what to hide, turn off, or keep. Runs only when you invoke it.
disable-model-invocation: true
---

### 任务

你在一台没有任何上下文的机器上工作。机器上装了一个或多个 coding agent（Claude Code、Codex 等），skill 很多，每个 skill 的名字和描述会进入每一轮对话的上下文，但没人知道哪些真的在用。

**这是只读审计。** 不改任何配置、不删任何文件、不装卸插件。产出一份报告，然后停下来等用户决定。

你要回答的是使用频次问题，按 skill 逐个回答：

1. 有没有人用过——模型、用户、哪个 agent 都算。
2. 用的话，是用户手动敲 `/name`，还是模型自己调用？
3. 模型调用时，是模型自己判断要用，还是用户在消息里点了名？

然后把答案落成两类建议：每个 agent 上该给模型什么可见性档位；作用域该不该从全局降到项目。

**立场：判断依据是使用频次，不是 token 成本。** token 只说明成本，不说明需求。天天用的贵 skill 该留，从没用过的便宜 skill 才该砍。报告里 token 或字符数不能作为任何「留/砍」的理由。

### 0. 先冻结数据

Claude Code 按保留期自动清理会话记录，清理会在审计过程中发生（作者机器上实测：审计开始约 20 分钟后 `~/.claude/.last-cleanup` 时间戳前移，会话 `.jsonl` 从 263 个降到 215 个）。所以动手解析前的第一件事：把会话记录、`~/.claude/history.jsonl`、`~/.claude.json`、各级 settings 复制到临时目录（只拷 `.jsonl` / `.json` 即可），记下快照时间，后面所有数字都从快照算。报告交付后删掉快照，因为里面有完整会话内容。其他 agent 的日志若体积大且确认没有自动清理，可以直接读，但要写明读取时间。

### 1. 数据在哪：每条线索都要先在本机验证

下面的路径、字段、机制是在 Claude Code 2.1.284、Codex 0.154.0、Kimi Code 2.1.1 上核实过的。别的版本可能已经变了，所以每一条都要先抽样确认：存在吗、格式还是这样吗、时间跨度多长。确认不了的，在报告里标「未验证」，不要照抄。

查 Claude Code 内部机制时：它是 Bun 打包的单文件二进制（`~/.local/share/claude/versions/<版本>`），可以搜字符串。在 200MB 的文件上跑带长上下文的 `grep -E` 会超时，用 Python `mmap` + `find()` 定位，再截前后几百字节看。

#### 1.1 Claude Code：模型实际看到了什么（曝光，所有结论的分母）

会话记录（见 1.2）里 `type=="attachment"` 且 `attachment.type=="skill_listing"` 的记录，字段有 `names`（数组）、`content`（每行 `- name: description`）、`skillCount`、`isInitial`。`isInitial=true` 是本会话的完整清单，之后的是增量。

- 某行只有 `- name`、没有冒号和描述：说明本会话没把描述给模型看（清单超预算被降级，或设成了 name-only）。
- 某行描述和名字一样（`- foo: foo`）：说明 SKILL.md 没有 frontmatter，模型拿不到可匹配的内容。
- 磁盘上有什么，和模型看到什么，不是一回事。分母用这批记录。

#### 1.2 Claude Code：谁触发了 skill

会话文件：`~/.claude/projects/<cwd 转义>/<session>.jsonl`；子 agent 的记录在 `<session>/subagents/*.jsonl`。目录名转义是有损的（`/` 和 `.` 都变 `-`），项目归属以记录里的 `cwd` 字段为准，不要反解目录名。

- **模型触发**：`type=="assistant"` 的 `message.content[]` 里出现 `{"type":"tool_use","name":"Skill","input":{"skill":"<名>","args":…}}`。按 `tool_use.id` 去重（fork、resume 出来的会话会复制历史）。`input.skill` 可能是短名，要按本机清单解析成全名。
- **用户手动敲**：`type=="user"` 记录里，`message.content` 是字符串（或文本数组，且不是 tool_result），并且含有 `<command-name>/<名></command-name>`。按 `uuid` 去重。`/model`、`/rename` 这类内置命令也会产生这个标签，一部分落在 `type=="system"`、`subtype=="local_command"` 的记录里，统计时都要排除。
- 作者机器上实测：用户敲 `/skill` 不会额外产生一条 Skill tool_use，两条路径互不重叠，可以分开计数。
- **保留期**：由 settings 里的 `cleanupPeriodDays` 决定，不设就是默认 30 天。按文件 mtime 滚动删除，所以被 resume 过的会话里可能留着比 30 天更早的记录；子 agent 文件跟着父会话一起留或一起删。
- **陷阱**：assistant 记录上的 `attributionSkill` 是给每条消息打的归属标签，不是调用次数。拿它来数会把一次调用放大成几百次（作者机器上 `query-flight` 有 4 次 Skill 调用，却有 811 条带这个标签的记录）。

#### 1.3 Claude Code：用户输入历史（长窗口）

`~/.claude/history.jsonl`，每行 `{display, pastedContents, project, sessionId, timestamp(毫秒)}`。默认的保留期清理不会修剪它：二进制里修剪 history 的那段代码只在 hipaa 标志打开时执行，作者机器上实测保留了 6 个月。这是查「用户敲过什么」最可靠的长窗口来源。

- `display` 是用户敲的原文，短名不会被展开。要还原全名：在同一个 `sessionId` 的会话记录里找 ±120 秒内的 `<command-name>`。
- 以 `/` 开头不一定是命令：粘贴的文件路径、错拼、中文都会出现。只认能和 1.5 清单对上的。
- **内置命令及其别名优先于 skill 匹配。** 否则按「唯一短名」匹配会出错。2.1.284 里 `/new` 是 `/clear` 的别名，`/continue` 是 `/resume` 的别名；作者机器上 `/new` 有 105 条，差点全被算成项目命令 `opsx:new`。内置命令名和别名在二进制里能搜到（`type:"local"` 或 `type:"local-jsx"` 的 `name:"…"` 后面跟着 `aliases:[…]`）。
- 项目级 skill 只有在该条记录的 `project` 落在那个项目里时才能计入。
- 按当时敲的名字归属。不要因为某个 skill 现在坏了，就把历史上的使用改记到别的同名 skill 头上。

#### 1.4 Claude Code：终身计数器

`~/.claude.json` 里的 `skillUsage`（`{name: {usageCount, lastUsedAt}}`）、`pluginUsage`（`{"name@marketplace": {...}}`）、`numStartups`。

- 用户敲 `/name` 和模型调用 Skill 工具走的是同一个计数函数，所以它分不出是谁触发的。同一个名字 60 秒内只记一次，只能当下限。
- 它的价值是窗口最长：`usageCount==0` 而且 history 里也没敲过，是「模型和用户都没用过」的最强证据。
- `usageCount>0` 但 history 里没敲过：如果这个 skill 是在 history 起点之后才安装的，这些使用就来自模型侧；否则也可能是 history 起点之前敲的。另外，非交互调用会不会写进 history，未验证。
- `pluginUsage` 还会被 hook、MCP 工具、LSP 累加。数值大不代表插件里的 skill 被用过（作者机器上某插件 `pluginUsage` 是 628，它的 skill 终身合计只有 7 次）。插件在安装或启用时就会写入 `lastUsedAt`，不能拿它当使用证据。

#### 1.5 skill 从哪来（决定能不能管）

- 用户级：`~/.claude/skills/<name>/SKILL.md`。项目级：`<项目>/.claude/skills/`。嵌套子目录里的 `.claude/skills` 会带上目录前缀作命名空间（作者机器上的 `refs:capture-to-refs`）。
- 旧式命令：`~/.claude/commands/**/*.md`、`<项目>/.claude/commands/`。子目录变成冒号命名空间（`opsx/new.md` → `opsx:new`），同样会进 skill 清单。
- 插件：`~/.claude/plugins/installed_plugins.json` 记录每次安装的 `scope`、`installPath`、`projectPath`、`installedAt`。进清单时名字是 `<manifest 里的 name>:<skill>`。如果 `.claude-plugin/plugin.json` 里有 `skills` 数组，以数组为准，路径可以任意深；只扫 `skills/` 一层会漏（作者机器上某插件的 25 个 skill 都藏在多层目录里）。
- `~/.claude/skills/<dir>/` 底下如果有 `.claude-plugin/plugin.json`，它就不是 skill：Claude Code 会把它当插件 `<name>@skills-dir` 自动加载。二进制里 `plugin init` 的帮助文字和 `pluginUsage` 的 key 都能印证这一点。
- 内置 skill 在磁盘上没有文件，只出现在 `skill_listing` 里，名字能在二进制里搜到字面量。
- 很多条目是指向共享目录的 symlink：要解析 realpath，并单独列出悬空链接、空目录、缺 SKILL.md 的条目。这些条目什么都不加载，不能和「零触发」混在一起。

#### 1.6 其他 agent

先确认哪些 agent 真的在用。很多 `~/.<agent>/skills` 只是安装器统一建的 symlink，没有任何会话日志：作者机器上有 43 个 `~/.<agent>/skills` 软链到 `~/.agents/skills`，在常见位置能找到可用会话日志的 agent 只有 Claude Code、Codex、Kimi 这 3 个。只有有日志的 agent 能统计频次；其余的在报告里写「暴露但无数据」，不要猜。

对每个有日志的 agent，要找到三样东西：给模型列了哪些 skill、用户显式调用了哪个、模型自己加载了哪个。缺哪样，就在报告里写明缺了什么、因此哪个结论下不了。

- **Codex 0.154.0**：日志在 `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`，每行有 `type`（`session_meta` / `response_item` / `event_msg` 等）和 `payload`。
  - 清单：消息文本里的 `<skills_instructions>` 段，每条是 `- name: desc (file: <path>)`。新版会先声明根目录别名（`` - `r1` = `/Users/x/.agents/skills` ``），条目里再写 `r1/foo/SKILL.md`。不展开别名就去匹配路径，插件 skill 会全部显示成从没加载过（作者第一遍就踩了这个坑）。
  - 用户显式调用：用户写 `$name`，会在 user 角色的消息里注入 `<skill><name>…</name><path>…</path>`。
  - 模型自己加载：Codex 没有专门的 Skill 工具，模型用 `exec` / `exec_command` 去 `cat` 或 `sed` SKILL.md。只有读的路径恰好是本会话清单里给出的路径才算加载，其余是在浏览或开发 skill 源码（作者机器上 983 次 SKILL.md 读取里有 120 次不在清单里）。同一会话同一 skill 只记一次；会话里用 `apply_patch` 改过这个 skill 的，算开发，不算使用。
  - 去重：`~/.codex/external_agent_session_imports.json` 记录了从 Claude Code 导入的会话（作者机器上 93 个），这些会话的 `thread_source` 为空，必须排除，否则会把 Claude Code 的使用重复算一遍。`thread_source=="subagent"` 的子线程要单独算或排除。
- **Kimi Code 2.1.1**：`~/.kimi-code/sessions/<workdir>/<session>/agents/<agent>/wire.jsonl`。加载事件写成 `<skill-loaded name="…" trigger="model-tool|user-slash" source="user|builtin" dir="…">`。同一次加载会出现在好几种事件里，只数 `type=="context.append_message"` 的那条。

#### 1.7 谁在管这些文件（决定改动会不会被覆盖）

- skills CLI：`~/.agents/.skill-lock.json` 记录 `source`、`installedAt`。这份锁文件会过期（作者机器上 48 条里有 19 条已经不在磁盘上）。
- 厂商 CLI（例如 lark-cli）会直接往共享目录装一整批 skill。
- cc-switch：`~/.cc-switch/cc-switch.db` 的 `skills` 表有各 agent 的启用标志和 `installed_at`（秒）。`installed_at` 是导入批次的时间，不是真实安装时间（作者机器上 72 行只对应 2 个时间点）。
- 共享目录 + symlink：删掉源文件，所有 agent 一起失去这个 skill。
- 安装时间普遍不可靠：文件的 birthtime 在重装时会被刷新（作者机器上某批 skill 的目标目录 birthtime 是 09-22，但 09-07 就已经出现在会话清单里了）。唯一硬的证据是「第一次出现在会话清单里」的时间，比这更早的存在只能从文件或锁的时间推断，算弱下限。

### 2. 怎么数

以模型看到的名字为主键；跨 agent 时按同名或同一 realpath 关联。每个 skill 至少分开记下面这些数，不许合并成一个「使用次数」：

- **模型自主**：模型调用了 skill，并且触发它的那条用户消息里没有出现这个名字。
- **被点名后调用**：那条消息里出现了名字。匹配要用 ASCII 边界：Python 的 `\w` 会匹配汉字，`用blog-writer写` 里的名字会被漏掉。拿不准时，宁可算成被点名，也不要算成自主。
- **用户敲 /name**：长窗口用 history，短窗口用会话记录还原全名。两个来源重叠的那段时间里应该对得上，对不上的要给出解释。
- **曝光**：进了多少个会话的清单、其中多少个会话带着描述。没有曝光的零触发不说明任何问题。
- **跨几个自然月**（看趋势）、**跨几个项目**（看作用域；同一仓库的多个 worktree 算一个项目）。
- **终身计数器**的值和最后使用时间，作为补充单独列出，不要加到上面的数里。

**自检**：挑一个你确定用过的 skill，确认它在上面至少有一个数不是零。如果所有 skill 都是零，几乎可以肯定是解析写错了，而不是真的没人用。作者实测时踩过的坑：内置命令的别名被算成 skill；粘贴的路径被当成命令；导入的会话被重复计数；Codex 的根目录别名没展开，插件 skill 全部显示为零。

### 3. 从频次到档位

档位要落到本机这个 agent 实际支持的机制上，见第 5 节。

| 观察到的使用 | 档位（Claude Code 的叫法） | 理由 |
|---|---|---|
| 模型自主 ≥1 | `on` | 描述在起作用 |
| 模型只在被点名后调用，自主为 0 | `name-only` | 模型只需要知道名字，用户点名时找得到就够了 |
| 只有用户敲过，模型从没调用 | `user-invocable-only` | 对模型完全隐藏，`/name` 仍然可用 |
| 用户和模型都是 0，且排除了第 4 节的干扰 | `off` | 没人用 |

### 4. 零次不等于没人要：先排除这些情况

- **模型根本没见过**：零曝光，或者描述每次都被隐藏。这时模型侧的结论不成立。
- **描述写坏了**：没有 frontmatter、描述就是名字本身。建议修描述，而不是关掉。
- **太新**：第一次进清单不到 30 天，或者是窗口中途才出现的。归为观察期。
- **项目休眠**：所属项目在所有窗口里都没有会话。这种 skill 只在该项目里加载，不影响别处。

每条结论都要标证据强度，并在报告里写明口径。作者实测用的口径：**强**：闲置 ≥90 天，带描述曝光 ≥20 个会话，且不是窗口中途才出现的。**中**：闲置 ≥30 天，带描述曝光 ≥10 个会话。**弱**：其余情况。

**作用域**：用户级 skill 的使用（≥3 次）全部集中在同一个项目，建议降为项目级。

**跨 agent**：同一份 SKILL.md 在一个 agent 里被频繁自动加载、在另一个 agent 里从没被选中，这是正常现象，因为背后的模型不同。档位按每个 agent 自己的数据定。某个文件还被其他 agent 用着，就不要建议删除它。

### 5. 机制约束：只给本机能落地的建议

每条建议都要写清楚：改哪个文件、哪个键、改成什么值、为什么选这一档而不是相邻的那一档。做不到的，写明原因。

#### 5.1 Claude Code 2.1.284（已验证）

- 键是 `skillOverrides`，取值 `on` / `name-only` / `user-invocable-only` / `off`，不写就是 on。按 skill 名作 key，可以写在 `~/.claude/settings.json`、`<项目>/.claude/settings.json`、`<项目>/.claude/settings.local.json`。三处按 key 合并，优先级 local > project > user。`/skills` 界面里的改动写进 `settings.local.json`。
- 作者机器上实测：在某项目的 `settings.local.json` 里把一个 skill 设成 `off` 之后，该项目后来的 43 个会话（主目录 11 个、worktree 32 个）清单里都不再出现它；内容逐字节相同、但没设 override 的对照 skill，在这 43 个会话里全部出现。说明 override 确实生效，而且 worktree 继承主仓库的设置。
- 由此推出「降为项目级」可以纯靠配置做到：全局设 `off`，再在目标项目的 local 里设 `on`。这是读合并逻辑推出来的，这个组合本身没有实测过。
- **`skillOverrides` 对插件 skill 完全无效。** 解析函数遇到 `source==="plugin"` 直接返回 `"on"`；`/skills` 界面会提示 "A plugin skill is managed with the plugin, not here." 插件 skill 只有两条路：
  - 整个插件关掉：`enabledPlugins: {"<name>@<marketplace>": false}`。如果插件是在项目的 `.claude/settings.json` 里启用的，`false` 必须写进 `.claude/settings.local.json`，写在用户级会被项目级覆盖。
  - 插件作者在 SKILL.md 的 frontmatter 里写 `disable-model-invocation: true`。这个判断排在插件判断之前，效果等同于 `user-invocable-only`，对插件 skill 同样生效。只有能改插件源码时才能这么做；团队共享的插件改了，会影响所有人。
- `~/.claude/skills/` 下的 skills-dir 插件：二进制里写明它是从这个目录自动加载的（`plugin init` 的帮助："auto-loads next session as `<name>@skills-dir`"），所以把目录移出去应该就停止加载。这是推断，没有实测。在 `enabledPlugins` 里写 `<name>@skills-dir: false` 是否生效，同样未验证。
- 旧式 commands 同样受 `skillOverrides` 控制（已验证）。内置 skill 不要单独动，除非用户要求；要整体关掉用 `disableBundledSkills: true`。
- 清单超出预算时进入 priority 模式：按带 7 天半衰期的使用分排序，排在后面的降成只有名字。所以零使用的 skill 天然会被挤成只剩名字，它们的「零触发」是弱证据。
- **谁会改写配置**：cc-switch 的数据库里存着一份包含 `skillOverrides` 的 Claude 通用配置（作者机器上当前值和 `settings.json` 一致），而它的各供应商配置大多只有 `env`，说明 `settings.json` 很可能是由这两部分拼出来的。只改 `settings.json` 不改 cc-switch，有被覆盖的风险。具体在什么时候写回，未实测。

#### 5.2 Codex 0.154.0

- **对模型隐藏、`$name` 仍可用**：在 skill 目录下的 `agents/openai.yaml` 里写 `policy: {allow_implicit_invocation: false}`。依据是 Codex 自带的 `~/.codex/skills/.system/skill-creator/references/openai_yaml.md`，原文："When false, the skill is not injected into the model context by default, but can still be invoked explicitly via `$skill`." 这个文件在 skill 目录里面，改共享目录里的 skill，会影响所有读这个目录的工具。
- **完全关掉**：用 `config.toml` 的 `[[skills.config]]` 条目，每条必须恰好有 `path` 或 `name` 其中一个。二进制里有 "ignoring skills.config entry without a path or name selector" 和 "…with both path and name selectors" 这两条日志；app-server 有对应的 `skills/config/write` 请求，报错文字是 "requires exactly one of path or name"。我没找到描述字段语义的一手文档，所以稳妥的做法是用 Codex 自带的 skill 管理界面改，改完回读 `config.toml` 确认写进去的是什么。
- **`disable-model-invocation` 不能跨 agent 通用**：Codex 自带的插件校验脚本（`plugin-creator/scripts/validate_plugin.py`）要求插件里的 skill 这个字段必须为 false。普通 skill 在运行时怎么处理这个字段，未确认。

#### 5.3 Kimi Code 2.1.1

- frontmatter 里的 `disable-model-invocation` 会被解析成 `disableModelInvocation`（二进制里的 `METADATA_ALIASES`），它自带的几个只由用户调用的 skill 就用了这个字段。隐藏效果没有实测。
- 写进 `~/.agents/skills` 下共享的 SKILL.md，会同时影响 Claude Code 和 Kimi。但那个文件归厂商 CLI 管，下次更新就会被覆盖。

### 6. 报告：规定内容，不规定形式

形式按这台机器扫出来的情况决定，表格、列表、段落都行。下面这些内容必须在。

**必须包含**

1. **数据可信度，放在最前面**：每个数据源的路径、条数、时间跨度、快照时间；哪些结论只有短窗口支撑；审计过程中有没有数据被删；缺了哪个数据源、因此哪个结论下不了。
2. **频次分布的全局事实**：进过模型清单的 skill 一共多少个，其中多少个被模型自主触发过、多少个只被用户敲过、多少个在所有信号上都是零。
3. **每个被提到的 skill，都要分开列出**：模型自主、被点名后调用、用户敲 /name、曝光（会话数 / 带描述的会话数）、跨几个月、跨几个项目。来源相同、结论相同的可以合并成一组，但组内每一项的数字或数字范围要给出来。
4. **每条建议都要能直接执行**：改哪个文件、哪个键、改成什么值；为什么是这一档而不是相邻那一档；证据强度。
5. **优先级**：按「证据强度 × 一次改动覆盖的 skill 数」排序，不按 token 排。用户看完要知道先动哪几个。
6. **明确点名不动哪些**：保持 on 的、在观察期的、内置的、已经处理过的。
7. **说清楚哪些做不到、为什么**：插件 skill、共享文件、会被管理工具覆盖的配置等。

**必须分开，不许混为一谈**：模型自主 / 被点名后调用 / 用户敲；没人要 / 描述写坏了 / 根本没机会曝光；长窗口证据 / 短窗口证据；能用 `skillOverrides` 管的 / 插件 / 内置；本 agent 的结论 / 其他 agent 的结论。

**禁止出现**：会话内容原文（只出 skill 名、计数、日期）；估算值或示例值冒充实测值，每个数字都要来自你实际解析的输出；为了结构好看编出来的内容，某一类为空就写明为什么空；拿 token 或字符成本作为留或砍的理由；「建议考虑……」「可以关注……」这类落不了地的措辞。

### 7. 执行纪律

- 只读。报告交付后停下，由用户决定。
- 用户确认要改时：只做字段级合并，绝不整文件重写；先备份，写完回读校验；有管理工具的（cc-switch、`/skills`、`/plugin`），优先走工具。
- 大文件逐行流式解析，坏行计数后跳过；不要整体读进内存，也不要靠肉眼看。
- 扫描脚本和中间结果放在临时目录；报告交付后删除快照。
