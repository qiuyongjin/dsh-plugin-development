# dsh-plugin-development

一个自包含的插件开发技能(Skill),用于在 **deepseek-harness**(下文简称 dsh)仓库中
开发、扩展和调试 Cordis 插件。技能本身不绑定任何特定 Agent:支持技能机制的 Agent
(Claude Code、Cursor 等)均可加载,也可以当作独立的开发指南直接阅读。

在 dsh 中,**一切皆是 Cordis 插件**:模型适配器、工具注册表、会话日志、agent
loop 本身都是插件——不存在需要打补丁的"特权核心",扩展产品的方式就是在其旁挂载
一个新插件。本技能把编写这类插件所需的全部规则封装在自身目录内(`SKILL.md` 与
`references/` 下的五个参考文件),做到自包含:不依赖外部文档即可按规范工作。

## 何时使用

当任务涉及以下任何内容时,Agent 会自动加载本技能(在支持技能机制的 Agent 中):

- **插件开发**:新建或修改 Cordis 插件、bundle、profile 层、cordis.yml 配置行
- **扩展点**:tool 注册(`ctx.tools` / `defineTool`)、service 与 `ctx.*` 键、
  事件监听、provider / capability 接缝、prompt section、commands
- **相关关键词**:插件、plugin、扩展点、extension point、capability、service、
  tool registration、cordis.yml、bundle、profile、`ctx.xxx`
- **修改 agent loop** 或注册新的 session 事件之前——确保变更挂接到已文档化的
  扩展点上,而不是硬改核心

也可以手动调用:在支持技能命令的 Agent 中输入 `/dsh-plugin-development`。

## 目录结构

```
dsh-plugin-development/
├── SKILL.md                      # 主技能:8 步开发流程 + 正确性清单
└── references/
    ├── plugin-templates.md       # 每种插件形态的代码模板
    ├── extension-points.md       # 扩展点地图、事件域与分发模式、命名规则
    ├── package-checklist.md      # package.json / tsconfig / 注册 / README 清单
    ├── tool-contract.md          # 工具 execute 契约、策略门、渲染意图规则
    └── testing-policy.md         # 测试分层、REAL 组合要求、快照义务
```

## 核心内容一览

| 主题 | 要点 | 参考 |
|---|---|---|
| 插件形态 | 函数插件(仅命名导出,无 default)、Service 子类、能力接缝(Definition / Provider / Consumer)、客户端插件 | `plugin-templates.md` |
| 命名与放置 | 按角色选 `packages/<group>/`;`ctx` 键单复数区分单一引擎与注册表;角色后缀要诚实 | `extension-points.md` |
| 注册机制 | 所有注册都是 effect(返回 disposer);依赖用 `inject` 声明;waterfall 监听器必须调用 `next()` 委托 | `extension-points.md` |
| 模型可见即记录 | 一切进入模型请求的内容必须能从会话日志重建,新输入要配套新 session event | `extension-points.md` |
| 配置 | schemastery `Config` 校验、加载时快速失败、无硬编码部署参数 | `SKILL.md` Step 5 |
| 挂载运行 | 树内走 workspace + profile;树外发布 npm 包或 bundle;patch 语义(整行替换 / `insert:`) | `SKILL.md` Step 6 |
| 测试 | 单元测试、逐文件 100% 覆盖率门、无密钥快照、REAL 组合 e2e(自跳过)、HMR 安全测试 | `testing-policy.md` |
| 文档 | 包 README 结构、Agent Note、`./invariant` 义务 | `package-checklist.md` |
| 工具契约 | 参数由 `defineTool` 预校验、入参只读、返回规范 JSON、尊重 `exec.signal` | `tool-contract.md` |

## 安装

本技能不限定于任何特定 Agent。当前放置在用户级技能目录,遵循该约定的 Agent
(如 Claude Code)会自动发现;也可以按你所用 Agent 的技能机制放置(例如项目内
`.claude/skills/`、`.cursor/` 等):

```bash
# 当前放置位置(用户级技能目录)
~/.claude/skills/dsh-plugin-development/
```

> **路径约定**:技能当前位于 `~/.claude/skills`(仓库之外),因此技能内提到的仓库路径
> 一律以纯文本(而非链接)书写,相对于 **deepseek-harness 仓库根目录**(含
> `AGENTS.md`、`packages/`、`docs/` 的目录)。

## 维护与迭代

修改本技能时请遵守其自身的两条规则:

1. **保持自包含**——新规则必须写进 `SKILL.md` 或 `references/`,而不是靠外部链接
   承载;技能内的参考文件从仓库根文档内联而来,**当两者冲突时以仓库原文为准**。
2. **内容同步**——`references/` 里的每份文件都标注了权威来源(如
   `docs/architecture.md`、`docs/cookbook/adding-a-package.md`),仓库文档更新后
   应同步更新对应参考文件。

## 参考实现(可对照学习的典型包)

| 模式 | 包 |
|---|---|
| 函数插件 + waterfall 恢复 + 持久事件 | `packages/llm/llm-retry` |
| Service 子类 + session 事件 + prompt section + 工具 | `packages/plan/plan-mode` |
| 工具三件套(Definition / provider / Consumer) | `packages/shell/shell` + `packages/shell/bash-local` + `packages/shell/tool-bash` |
| Provider 注册表 | `packages/skill/skill` + `packages/skill/skill-filesystem` |
| 配置丰富的适配器 | `packages/llm/llm-deepseek` |
| 无密钥驱动的插件树挂载 | `examples/headless-agent` |
| web profile 上的 patch overlay | `examples/web-cordis/cordis.yml` |
