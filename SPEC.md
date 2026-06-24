# 技能自进化系统 — OpenClaw 迁移实现规格

> 本规格逆向自 Hermes Agent 真实源码（agent/curator.py、tools/skill_usage.py、
> agent/prompt_builder.py、agent/skill_utils.py），所有默认值、状态机、判定逻辑
> 均与生产实现一致，不是泛化描述。照此实现即可在 OpenClaw 复刻同等能力。

---

## 0. 心智模型

"自进化"不动模型权重。它 = 文件化的程序性记忆 + 读写工具 + 提示词触发纪律 + 后台治理（Curator）。

四个支柱缺一不可：
1. 技能是磁盘上的纯文本文件 → 可读写、可版本管理、可备份。
2. 一组工具让 agent 增删改查这些文件 → "学习"在机制上等于"写文件"。
3. 系统提示词写死何时存、何时改 → 让 agent 真的去维护，而非被动等指令。
4. Curator 后台进程做去重/归档/剪枝 → 防止技能库腐烂成死垃圾堆（多数方案活不久就死在这一步）。

省 token 的关键设计：平时只把每个技能的一行 description 注入上下文，真正相关时才按需拉全文。

---

## 1. 技能文件格式

### 目录布局
```
$CLAW_HOME/skills/
├── <category>/                  # 可选分类子目录
│   └── <skill-name>/
│       ├── SKILL.md             # 主文件（必需）
│       ├── references/          # 按需加载的参考资料
│       ├── templates/
│       └── scripts/
├── .usage.json                  # 使用遥测 sidecar（见 §4）
├── .curator_state               # curator 调度状态（见 §5）
└── .skills_prompt_snapshot.json # 提示词快照缓存（见 §3）
```

### SKILL.md 结构
```markdown
---
name: deploy-netlify          # 小写、连字符，≤64 字符，全局唯一
description: "一句话说清触发条件和用途，注入到上下文的就是这一行"
version: 1.0.0
created_by: agent             # 关键溯源字段：agent 创建的才归 curator 管
tags: [deploy, netlify, static]
---

# 技能标题

## 何时使用（触发条件）
- 明确列出什么场景该加载本技能

## 步骤
1. 带确切命令的编号步骤
2. ...

## 坑 / Pitfalls
- 踩过的坑写在这里，下次不重复犯

## 验证
- 怎么确认成功
```

溯源字段 created_by 是整个治理体系的地基。只有 created_by: agent（或 usage 记录里
agent_created: true）的技能才会被 curator 触碰。内置技能和从 hub 安装的技能永远豁免。

---

## 2. 工具接口（暴露给 agent）

### 2.1 skill_view(name, file_path=None)
- 不带 file_path：返回 SKILL.md 全文 + linked_files 字典（列出可用的
  references/templates/scripts）。
- 带 file_path：返回该附属文件内容（限定在 references/ templates/ scripts/ assets/ 下）。
- 副作用：调用即 bump_view（见 §4）。

### 2.2 skill_manage(action, ...)
| action | 参数 | 说明 |
|--------|------|------|
| create | name, content, category | 新建 SKILL.md，写入 created_by: agent |
| patch | name, old_string, new_string, replace_all | 首选的小修改方式，find-replace |
| edit | name, content | 全文重写，仅大改用 |
| delete | name, absorbed_into | 删除；absorbed_into 区分"合并"还是"剪枝" |
| write_file | name, file_path, file_content | 写附属文件 |
| remove_file | name, file_path | 删附属文件 |

### 2.3 skills_list(category=None)
返回所有技能的 name + description（不含正文），供 agent 浏览。

实现要点：所有写操作用原子写（临时文件 + os.replace + fsync），避免半写坏文件。

---

## 3. 上下文注入与省 token（核心）

这是规模化的命门。错误做法是把所有技能全文塞进系统提示词——技能一多就爆。

### 3.1 注入策略
- 系统提示词里只放一个技能清单：每个技能一行 name: description。
- 完整正文永远不进系统提示词，只在 agent 主动 skill_view 时才加载。
- 提示词里写一条硬规则：开工前扫清单，相关就 skill_view 加载后再动手。

### 3.2 快照缓存（冷启动加速）
Hermes 用一个 manifest 快照避免每次启动都重新解析所有 SKILL.md：

```python
# 为每个 SKILL.md / DESCRIPTION.md 记录 [mtime_ns, size]
def build_skills_manifest(skills_dir) -> dict[str, list[int]]:
    manifest = {}
    for filename in ("SKILL.md", "DESCRIPTION.md"):
        for path in iter_skill_files(skills_dir, filename):
            st = path.stat()
            manifest[str(path.relative_to(skills_dir))] = [st.st_mtime_ns, st.st_size]
    return manifest

# 启动时：若磁盘快照的 manifest 与当前一致，直接复用解析好的 description，
# 不一致才重新解析并回写快照。
```
快照文件 payload：{version, manifest, skills:[...], category_descriptions:{...}}。
版本号变了或 manifest 不匹配就作废重建。

---

## 4. 使用遥测 sidecar（.usage.json）

curator 的决策依据。每个技能一条记录：

```json
{
  "deploy-netlify": {
    "created_by": "agent",
    "use_count": 12,
    "view_count": 30,
    "patch_count": 2,
    "last_used_at": "2026-06-20T10:00:00Z",
    "last_viewed_at": "2026-06-23T08:00:00Z",
    "last_patched_at": "2026-06-15T09:00:00Z",
    "created_at": "2026-05-01T00:00:00Z",
    "state": "active",
    "pinned": false
  }
}
```

埋点：
- skill_view → bump_view（view_count++、last_viewed_at）
- 技能被实际执行/采用 → bump_use
- skill_manage patch/edit → bump_patch

```python
def activity_count(record) -> int:
    return sum(int(record.get(k) or 0) for k in ("use_count","view_count","patch_count"))
```

判定函数：
```python
def is_agent_created(skill_name) -> bool:
    off_limits = bundled_names() | hub_installed_names()
    return skill_name not in off_limits

def is_managed(record) -> bool:           # curator 是否接管
    return record.get("created_by") == "agent" or record.get("agent_created") is True
```

遥测追踪所有技能（含内置），但 curator 只对 is_managed 为真的下手。遥测是可观测性，
治理范围是另一回事，两者分开。

---

## 5. Curator — 后台技能生命周期治理

### 5.1 触发方式（无 daemon）
Hermes 用闲置触发而非定时守护进程：
- 每轮交互结束检查：agent 空闲，且距上次 curator 运行超过 interval_hours。
- 满足则 spawn 一个 fork 出来的 review agent，用辅助模型（不是主对话模型，绝不污染
  主会话的 prompt cache）跑一遍审查。
- 还要满足 min_idle_hours：agent 闲置不够久不跑，避免打断连续工作。

### 5.2 生命周期状态机
```
active  ──(stale_after_days 无活动)──>  stale  ──(再 archive_after_days 无活动)──>  archived
```
- 三个合法状态：active / stale / archived。
- 永不删除，最狠到 archived，可 restore。
- pinned 技能豁免一切自动转换。
- 内置 / hub 技能不参与（is_managed 过滤）。

### 5.3 真实默认值（直接抄）
```yaml
curator:
  enabled: true              # 默认开
  interval_hours: 168        # 7 天跑一次
  min_idle_hours: 2          # agent 至少闲置 2h 才触发
  stale_after_days: 30       # 30 天无活动 → stale
  archive_after_days: 90     # 再 90 天无活动 → archived
  backup:
    enabled: true            # 跑之前先 tar.gz 备份整个 skills/
```

注意时钟语义：内置技能首次纳管时 archive 倒计时从"当下"起算，不是从 epoch 起算——
避免一个长期没用的老技能在标志位刚打开就被立刻归档。

### 5.4 review agent 能做什么
fork 出来的审查 agent 通过 skill_manage 可以：pin、archive、consolidate（合并重复
技能）、patch。它拿到的是技能清单 + 使用遥测，自己判断哪些该合并、哪些过时。

### 5.5 状态持久化（.curator_state）
```json
{
  "last_run_at": "...", "last_run_duration_seconds": 42,
  "last_run_summary": "...", "last_report_path": "...",
  "paused": false, "run_count": 7
}
```
原子写（mkstemp + fsync + os.replace）。

### 5.6 CLI 动词（建议在 OpenClaw 实现的对应命令）
status / run / pause / resume / pin / unpin / archive / restore / prune / backup / rollback

---

## 6. 系统提示词片段（直接嵌入 OpenClaw 的 system prompt）

```
## 技能（强制）
回复前扫一遍下面的技能清单。只要某个技能与任务相关或部分相关，你必须用
skill_view(name) 加载它并遵循其中的步骤。宁可加载用不上，也不要漏掉关键步骤、
坑或既定流程。技能里编码了用户偏好的做法和质量标准——即使是你会做的任务也要加载，
因为技能定义了"这里应该怎么做"。

完成困难/多步任务（5+ 步）、克服了报错、用户纠正过的做法奏效、或发现了非平凡
流程后，主动用 skill_manage(action='create') 把方法存成技能。

用到某个技能时若发现它过时、缺步骤或有错，立刻用 skill_manage(action='patch')
修正——别等人催。没人维护的技能会变成负债。

<available_skills>
  {category}:
    - {name}: {description}
    ...
</available_skills>
```

---

## 7. 实现检查清单（OpenClaw 落地顺序）

- [ ] 1. 定 SKILL.md 格式 + frontmatter schema（含 created_by 溯源字段）
- [ ] 2. 目录约定 + 启动加载器（只注入 description，不注入正文）
- [ ] 3. 快照缓存（manifest = mtime_ns + size），冷启动复用
- [ ] 4. 工具：skill_view / skill_manage / skills_list，全部原子写
- [ ] 5. 遥测 sidecar .usage.json + 三类埋点（view/use/patch）
- [ ] 6. 系统提示词触发纪律（§6 片段）
- [ ] 7. Curator：闲置触发 + 状态机 + 默认值（§5.3）+ tar.gz 备份 + 永不删除
- [ ] 8. pin/unpin 豁免逻辑
- [ ] 9. CLI 动词（§5.6）

---

## 8. 三个最容易被省掉、但决定成败的点

1. 省 token 加载（§3）：不做就是技能一多上下文崩，这是规模化命门。
2. Curator 治理（§5）：不做技能库 3 个月烂成垃圾堆，自进化变自污染。
3. 溯源 + 永不删除（§1、§5.2）：保证治理只碰 agent 自建技能、且最狠到归档可恢复，
   用户和内置技能绝对安全——这是敢让它自动跑的前提。

技术上没有任何专有训练依赖，全是工程。门槛在纪律设计，不在算法。
