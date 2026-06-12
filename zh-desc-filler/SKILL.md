---
name: zh-desc-filler
description: "自动为已安装 skill 补上中文描述（description_zh）。扫描单个或全部已安装 skill 的 SKILL.md，检查 description 是否为纯英文且缺少 description_zh 字段，如是则自动翻译并写入。适用于用户说「给 XX skill 补上中文描述」「加 description_zh」「这个 skill 是干嘛的」「看不懂这个 skill」等场景。"
description_zh: 自动为已安装的 skill 补上中文描述（description_zh），支持单个或批量扫描。
agent_created: true
trigger_keywords:
  - 补中文描述
  - 加 description_zh
  - 扫描缺中文描述
  - 看不懂这个 skill
  - 这个 skill 是干嘛的
  - 检查 skill 中文
  - zh-desc-filler
  - 补上中文翻译
---

# zh-desc-filler — 自动补全 Skill 中文描述

## 工作流程

按以下优先级顺序执行：

### 步骤 1：确定范围

分析用户输入，确定要处理的 skill 范围：

| 用户输入示例 | 范围 | 动作 |
|------------|------|------|
| "给 **XXX** 补上中文描述" | 单个 skill | 只处理指定名称的 skill |
| "扫描所有缺中文描述的 skill" | 全部 | 扫描全部 ~/.workbuddy/skills/*/SKILL.md |
| "检查 skill 的中文描述" | 全部 | 同批量扫描 |
| "这个 skill 是干嘛的" + 上下文中有明确 skill 名 | 单个 | 处理上下文中引用的 skill |
| "给这个 skill 加 description_zh" + 上下文 | 单个 | 同上 |

**单 skill 模式**：先用 Glob 或 Bash 在 `~/.workbuddy/skills/` 下搜索匹配的目录名，找到准确的 SKILL.md 路径。

**批量扫描模式**：直接在 `~/.workbuddy/skills/*/SKILL.md` 遍历。

### 步骤 2：读取并分析 frontmatter

对每个待检查的 SKILL.md，读取 YAML frontmatter（第一个 `---` 到第二个 `---` 之间的内容），识别以下字段：

```
---
name: xxx
description: 字段内容（可能是英文或中文）
description_zh: 字段内容（可能不存在）
---
```

### 步骤 3：判断是否需要补翻译

**跳过条件（任一满足则跳过，不做任何修改）：**

1. **description 本身已含中文**（用正则匹配 CJK 统一表意文字范围 `\p{Han}`）→ 说明原生就是中文 skill，不需要翻译
2. **description_zh 字段已存在** → 即使内容简短也跳过（用户之前已明确要求只补缺失的）
3. **description 为空** → 跳过
4. **SKILL.md 格式异常**（无法解析 frontmatter）→ 跳过并记录警告

**处理条件（全部满足才处理）：**
- description 是纯英文（不含任何 CJK 字符）
- description_zh 不存在
- description 不为空

### 步骤 4：生成翻译并写入

1. 读取 `description` 的完整英文内容
2. 用模型自身能力生成准确的中文翻译（注意保持技术术语准确，不要直译）
3. 在 SKILL.md 的 YAML frontmatter 中，在 `description` 行**之后**、其他字段**之前**插入一行 `description_zh: 中文翻译`
4. 写入文件

**插入格式示例：**

```yaml
---
name: some-skill
description: Some English description text here
description_zh: 这里放中文翻译
version: 1.0.0
---
```

### 步骤 5：报告结果

处理完成后，向用户呈现简洁报告：

**单 skill 模式：**
```
✅ 已为 XXX skill 补上 description_zh：
   "英文描述 → 中文翻译"
```

**批量扫描模式：**
```
📊 扫描完成！共检查 X 个 skill：
  ✅ 已补翻译：A、B、C...（N 个）
  ⏭️ 已有 description_zh：D、E、F...（N 个）
  ⏭️ description 已是中文：G、H...（N 个）
  ⚠️ 跳过（异常）：I...（N 个）
```

## 注意事项

- **仅修改 description 为纯英文且 description_zh 缺失的 skill**，不对已有内容做任何覆盖
- 不要修改 SKILL.md 中除插入 `description_zh` 行之外的任何内容
- 当 `description` 的 YAML 值使用引号包裹（如 `description: "text"`），翻译后的 `description_zh` 也应使用相同格式（双引号包裹）
- 当 `description` 使用多行折叠语法（`|` 或 `>-`），`description_zh` 也使用相同语法
- 如果单个 skill 的处理失败（文件锁定、权限等），记录错误但继续处理其他 skill，不要中断整个流程
