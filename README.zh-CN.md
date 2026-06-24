# 技能自进化系统 — OpenClaw 迁移实现规格

**让 Agent 拥有真正的自进化能力——不是 Prompt 技巧，是工程。**

这是一套完整的技能自进化系统规格，逆向自 Hermes Agent 生产实现，在 OpenClaw 上验证可用。

---

## 这是什么？

大多数 AI Agent 是无状态工具。每次对话结束后，记忆清零。这套规格描述了一个**技能自进化系统**，赋予 Agent：

- **持久化技能记忆** — 学到的程序性知识跨 session 存活
- **使用遥测** — Agent 知道自己实际用了哪些技能
- **生命周期治理（Curator）** — 自动归档死技能，防止技能库腐烂
- **省 token 加载** — 只把一行摘要注入上下文，相关时按需拉全文

> "瓶颈不是智能，是纪律设计。" — 规格第 8 节

---

## 四大支柱

| 支柱 | 作用 |
|------|------|
| **技能文件** | 磁盘上的纯文本 `SKILL.md` — 可读写、可版本管理、可备份 |
| **读写工具** | `skill_view`、`skill_manage`、`skills_list` — "学习"在机制上等于"写文件" |
| **系统提示词纪律** | 写死何时存、何时改 — 让 Agent 真的去维护，而非被动等指令 |
| **Curator** | 后台治理：去重、归档、剪枝 — 防止技能库腐烂成死垃圾堆 |

---

## 文件结构

```
$CLAW_HOME/skills/
├── <category>/
│   └── <skill-name>/
│       ├── SKILL.md              # 主文件（必需）
│       ├── references/           # 按需加载
│       ├── templates/
│       └── scripts/
├── .usage.json                   # 使用遥测 sidecar
├── .curator_state                # Curator 调度状态
└── .skills_prompt_snapshot.json  # 提示词快照缓存
```

---

## SKILL.md 格式

```markdown
---
name: deploy-netlify
description: "一句话说清触发条件和用途，注入到上下文的就是这一行"
version: 1.0.0
created_by: agent
tags: [deploy, netlify, static]
---

# 技能标题

## 何时使用（触发条件）

## 步骤
1. 带确切命令的编号步骤

## 坑 / Pitfalls
- 踩过的坑写在这里

## 验证
- 怎么确认成功
```

`created_by: agent` 字段是整个治理体系的地基。只有 agent 创建的技能才会被 Curator 触碰。内置和 hub 安装的技能永远豁免。

---

## 省 token 加载（规模化命门）

**错误做法：** 把所有技能全文塞进系统提示词 → 技能一多上下文崩。

**正确做法：**
1. 系统提示词只放一行式清单：每个技能一行 `name: description`
2. 完整正文永远不进系统提示词
3. Agent 调用 `skill_view(name)` → 按需加载
4. 快照缓存避免冷启动重新解析所有 SKILL.md

---

## Curator — 生命周期治理

多数实现会省掉这步。不做，技能库 3 个月烂成垃圾堆。

**触发方式：** 闲置触发，非守护进程。Agent 闲置超过 `min_idle_hours`，且距上次运行超过 `interval_hours`，才触发。

**状态机：**
```
active ──(stale_after_days 无活动)──> stale ──(archive_after_days 无活动)──> archived
```

永不删除。最狠到 `archived`，始终可恢复。

**真实默认值（直接来自生产）：**
```yaml
curator:
  enabled: true
  interval_hours: 168       # 每周一次
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    enabled: true           # 每次运行前 tar.gz 备份
```

**治理范围：** 只对 `is_managed`（即 `created_by: agent`）的技能下手。内置和 hub 技能不参与。

---

## 使用遥测（.usage.json）

每个技能一条记录：

```json
{
  "deploy-netlify": {
    "created_by": "agent",
    "use_count": 12,
    "view_count": 30,
    "patch_count": 2,
    "last_used_at": "2026-06-20T10:00:00Z",
    "state": "active",
    "pinned": false
  }
}
```

三类埋点：
- `skill_view` → `bump_view`
- 技能被实际采用/执行 → `bump_use`
- `skill_manage patch/edit` → `bump_patch`

---

## 实现检查清单

- [ ] 1. 定 SKILL.md 格式 + frontmatter schema（含 `created_by` 溯源字段）
- [ ] 2. 目录约定 + 启动加载器（只注入 description，不注入正文）
- [ ] 3. 快照缓存（manifest = mtime_ns + size），冷启动复用
- [ ] 4. 工具：`skill_view` / `skill_manage` / `skills_list`，全部原子写
- [ ] 5. 遥测 sidecar `.usage.json` + 三类埋点（view/use/patch）
- [ ] 6. 系统提示词触发纪律（规格 §6 片段）
- [ ] 7. Curator：闲置触发 + 状态机 + 默认值 + tar.gz 备份 + 永不删除
- [ ] 8. pin/unpin 豁免逻辑
- [ ] 9. CLI 动词：status / run / pause / resume / pin / unpin / archive / restore / prune / backup / rollback

---

## 完整规格

见 [SPEC.md](SPEC.md)，含所有代码示例、状态机、逆向自生产实现的默认值。

---

## License

MIT
