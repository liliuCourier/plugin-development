# plugin-development

DSH (DeepSeek Harness) 用户级技能：Cordis 插件开发的完整规则手册 + "先澄清需求、先检索轮子"的强制工作流。

## 这是什么

一个注入后即用的 DSH skill，解决"每次新对话都要重新解释插件实现原理和规则"的问题。包含：

1. **第 0 节 · 需求澄清与轮子检索（MUST 流程）**：写任何代码前必须走完四步——
   - 先 grill 用户，问清功能/需求（参考 grill-me 类工作流）
   - 基于需求在 GitHub / 插件市场检索现成实现并分析
   - 拿着检索结果再次向用户确认哪些复用、哪些新建
   - 产出完整功能与需求设计（必要时多方案+取舍）
2. **官方底稿全文**：cordis-plugin-development 技能内容（Host/Client 分工、Provider 导航、生命周期清理、事件、UI Slot、Host↔Client RPC、版本/审批/回滚、常见报错排查等）
3. **维护说明**：新规则随时回写，官方底稿更新时同步

## 安装

把 skills/plugin-development 目录复制到 DSH 用户级技能目录（\/skills，Windows 默认 %USERPROFILE%\.dsh\skills）：

``powershell
New-Item -ItemType Directory -Force -Path "C:\Users\20112\.dsh\skills" | Out-Null
Copy-Item -Recurse -Force .\skills\plugin-development "C:\Users\20112\.dsh\skills\"
``

也可以复制到项目级目录 <项目根>/.dsh/skills/，只对单个项目生效。

## 使用

- **自动触发**：新对话中提到 Cordis 插件相关的创建/修改/调试/扩展任务时，模型会自动加载
- **手动触发**：对话中输入 /plugin-development

## 目录结构

``
plugin-development/
└── skills/
    └── plugin-development/
        └── SKILL.md      # 技能本体（frontmatter + 规则正文）
``

## 维护

- 修改技能内容：编辑 skills/plugin-development/SKILL.md 后复制回 ~/.dsh/skills/，或直接用本技能自己的"维护说明"规则让助手回写。
- 官方 cordis-plugin-development 更新时，同步替换对应章节。

## License

MIT