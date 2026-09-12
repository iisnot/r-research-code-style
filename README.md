# r-research-code-style

一份面向科研场景的 **R 代码写作风格指南**。适用于农业、气候、空间、计量、DSSAT、SEM、机器学习与 ggplot2 工作流。

它的立场是：**定义代码应该"读起来是什么样"，而不规定具体技术实现**。技术选型留给执行时判断，避免风格规范变成实现约束。

## 安装

### WorkBuddy / Claude Code（用户级）

把整个 `r-research-code-style` 文件夹复制到：

```
~/.workbuddy/skills/
```

即最终路径为 `~/.workbuddy/skills/r-research-code-style/SKILL.md`。Windows 下 `~` 对应 `C:\Users\<用户名>`。

### 项目级（随项目分发给合作者）

复制到项目根目录的 `.workbuddy/skills/` 下，团队内共享同一套代码风格。

## 内容概要

| 章节 | 说明 |
|------|------|
| Guiding Principle | 原则优先于处方：只约束表达，不约束实现 |
| Dependency Judgment Order | 四档取用顺序：base/已加载 → 成熟包 → 陌生包（先读源码）→ 自写 |
| Replacing An Implementation | 替换实现是语义变更，须核对单位、参数顺序、基准、NA、返回类型、向量化 |
| Voice And Tone | 注释用研究者的因果口吻，记录代码中看不见的信息 |
| Core Rules | 命名、参数置顶、可追溯性、单位显式、一致性的硬性风格条款 |
| Script Structure | 0–8 段的固定脚本骨架 |
| Naming Style | 六类对象命名对照表 |
| Readability Rhythm | 版面节奏：换行、对齐、留白、反炫技 |
| Narrative Requirements By Stage | 六个阶段各自的"要说清楚什么" |
| Paths And Outputs | 路径只定义一次的写法 |
| Review Checklist | 审查代码时按六档优先级报告 |
| What This Skill Deliberately Does Not Specify | 显式留白清单 |

## 使用建议

加载后，在写 / 审 / 重构 R 代码时会被约束：中文注释密度、对象命名的可追溯性、分段结构、以及"不把工作逻辑改写成另一套技术方案"的边界。

如需调整语气或增删规则，直接编辑 `SKILL.md` 对应章节即可——该文件不含脚本依赖，纯文本规则。
