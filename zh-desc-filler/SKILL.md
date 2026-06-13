---
name: zh-desc-filler
description: "自动为已安装 skill 补上中文描述（description_zh）。扫描单个或全部已安装 skill 的 SKILL.md，检查 description 是否为纯英文且缺少 description_zh 字段，如是则自动翻译并写入。适用于用户说「给 XX skill 补上中文描述」「加 description_zh」「这个 skill 是干嘛的」「看不懂这个 skill」等场景。"
description_zh: 自动为已安装的 skill 补上中文描述（description_zh），支持单个或批量扫描。
version: "1.2.0"
last_updated: "2026-06-13"
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

> 扫描单个或全部已安装 skill 的 SKILL.md，为纯英文 description 自动翻译并写入 description_zh 字段。

---

## 前置检查

开始前先确认环境：

```
~/.workbuddy/skills/ 目录存在？
  ├─ 否 → ❌ 无可操作的 skill，直接报告退出
  └─ 是 → ✅ 继续
```

---

## 工作流程

按以下优先级顺序执行：

### 步骤 1：确定范围

分析用户输入，确定要处理的 skill 范围：

| 用户输入示例 | 范围 | 动作 |
|------------|------|------|
| "给 **XXX** 补上中文描述" | 单个 skill | 只处理指定名称的 skill |
| "扫描所有缺中文描述的 skill" | 全部 | 扫描全部 `~/.workbuddy/skills/*/SKILL.md` |
| "检查 skill 的中文描述" | 全部 | 同批量扫描 |
| "这个 skill 是干嘛的" + 上下文中有明确 skill 名 | 单个 | 处理上下文中引用的 skill |
| "给这个 skill 加 description_zh" + 上下文 | 单个 | 同上 |

**单 skill 模式：**
- 用 Glob 搜索 `~/.workbuddy/skills/*/SKILL.md` 匹配目录名
- 如未找到精确匹配 → 用 Bash `ls ~/.workbuddy/skills/` 列出全部 skill，让用户确认
- 找到后记录完整路径 `~/.workbuddy/skills/<skill-name>/SKILL.md`

**批量扫描模式：**
- 用 Glob `~/.workbuddy/skills/*/SKILL.md` 或 Bash `find ~/.workbuddy/skills -maxdepth 2 -name SKILL.md` 扫描全部

> ⚠️ **用户确认检查点**：批量扫描模式下，收集完待修改清单后，先展示给用户确认：
> ```
> 发现 N 个 skill 缺中文描述，将自动补全：
>   - skill-A
>   - skill-B
>   - ...
> 是否继续？(Y/N)
> ```
> 用户确认后执行步骤 2，否则退出。

### 步骤 2：读取并分析 frontmatter

对每个待检查的 SKILL.md：

1. **用 Read 工具**读取整个文件内容
2. **定位 YAML frontmatter** — 提取第一个 `---` 到第二个 `---` 之间的内容
3. **识别关键字段**：

```yaml
---
name: xxx                  # 识别
description: "..."         # 识别（可能是英文或中文）
description_zh: "..."      # 不一定存在
# 其他字段忽略
---
```

**frontmatter 解析失败的异常处理：**
- 找不到 `---` 定界符 → 记录为"格式异常"并跳过，不中断流程
- 有第一个 `---` 但无第二个 → 同上处理
- frontmatter 能解析但 `name` 字段不存在 → 记录警告但继续（用目录名作为 fallback）

### 步骤 3：判断是否需要补翻译

**跳过条件（任一满足则跳过，不做任何修改）：**

1. **description 本身已含中文**（用正则匹配 CJK 统一表意文字范围 `\p{Han}`）→ 说明原生就是中文 skill，不需要翻译
2. **description_zh 字段已存在** → 即使内容简短也跳过（只补缺失的，不覆盖已有的）
3. **description 为空** → 跳过，无意义
4. **SKILL.md 格式异常**（无法解析 frontmatter）→ 跳过并记录警告

**处理条件（全部满足才处理）：**
- description 是纯英文（不含任何 CJK 字符）
- description_zh 不存在
- description 不为空

### 步骤 4：生成翻译并写入

#### 4a. 读取 description 内容
- 从 frontmatter 中提取 `description` 字段的完整值
- 注意 YAML 值可能有引号包裹、多行折叠语法（`|` / `>-`）等

#### 4b. 生成翻译
用自身能力生成准确的中文翻译，**遵循以下规则**：

```
✅ 保留专有名词：skill 名、技术框架名、人名、产品名不翻译
✅ 技术术语要准："frontmatter"→"frontmatter"而非"前沿领域"
✅ 保持技术语境，不文艺化
✅ 如果 description 是触发式短语（如 "Use when..."），翻译成与之对等的触发式中文（如 "适用场景"）
```

#### 4c. 写入 SKILL.md
在 YAML frontmatter 中，在 `description` 行**之后**、其他字段**之前**插入一行：

```yaml
---
name: some-skill
description: Some English description text here
description_zh: 这里放中文翻译
version: 1.0.0
---
```

**写入规则表：**

| description 格式 | description_zh 格式 | 示例 |
|-----------------|-------------------|------|
| `description: plain text` | `description_zh: 中文` | 不加引号 |
| `description: "quoted text"` | `description_zh: "中文"` | 双引号匹配 |
| `description: 'single quoted'` | `description_zh: '中文'` | 单引号匹配 |
| `description: \|` 多行块 | `description_zh: \|` 多行块 | 语法匹配 |
| `description: >-` 折叠 | `description_zh: >-` 折叠 | 语法匹配 |

**写入方法：**
用 Edit 工具，在 frontmatter 中定位 `description:` 行，在其后插入 `description_zh:` 行。注意保留原文件的缩进和空格风格。

#### 4d. 验证写入
写入后，用 Read 工具重新读取 frontmatter，确认：
- ✅ `description_zh` 行已正确插入
- ✅ 原 `description` 行未被修改
- ✅ 其他字段未被影响

### 步骤 5：报告结果

处理完成后，向用户呈现简洁报告：

**单 skill 模式：**
```
✅ 已为 <skill 名> 补上 description_zh：
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

---

## 边界条件速查

| 场景 | 处理方式 |
|------|---------|
| `~/.workbuddy/skills/` 目录不存在 | ❌ 直接报告退出，告知用户无已安装 skill |
| 找不到指定 skill 名 | 列出全部 skill 让用户确认，或询问是否批量扫描 |
| frontmatter 缺少 `name` 字段 | 用目录名作为 fallback，记录警告 |
| 无法解析 frontmatter | 跳过并记录"格式异常"，不中断流程 |
| 文件被锁定（权限问题） | 记录错误，跳过该文件，继续处理其他 |
| 批量扫描大数量（50+） | 确认后逐个处理，每 10 个输出一次进度 |
| description 含代码/URL/人名 | 保留不翻译，在翻译结果中用括号标注原文 |
| 写入后验证失败 | 记录"写入未验证"警告，不撤回已写入的内容 |
| description 含特殊 YAML 字符（:、# 等） | 用引号包裹 description_zh 值 |

---

## 翻译质量规则

1. **保留专有名词不翻译**：
   - ✅ `frontmatter` → `frontmatter`，不是"前沿领域"
   - ✅ `code review` → `code review`，不是"代码审查"
   - ✅ 人名/产品名/公司名保留原文

2. **技术语境准确**：
   - `trigger when` → `触发条件：`（不要直译成"当...时触发"）
   - `use this when` → `适用场景：`（不要直译）

3. **简洁一致**：中文描述不要超过 80 字，单行完成

4. **触发短语还原**：如果 description 是触发式短语（如 "Trigger: deploy, publish"），中文版本保持同等模式（`触发：部署、发布`）

---

## 后置自检清单

完成每项修改后，对修改过的 SKILL.md 逐项检查：

- [ ] `description_zh` 行已插入 ✅/❌
- [ ] `description` 行未被修改 ✅/❌
- [ ] YAML frontmatter 结构未被破坏 ✅/❌
- [ ] 文件尾部的正文内容未被修改 ✅/❌
- [ ] 翻译无错别字 ✅/❌

如任一检查项失败 → 记录警告，告知用户手动检查该文件

---

## 注意事项

- **仅修改 description 为纯英文且 description_zh 缺失的 skill**，不对已有内容做任何覆盖
- 不要修改 SKILL.md 中除插入 `description_zh` 行之外的任何内容
- 插入位置严格在 `description` 行之后、其他字段之前
- 如果单个 skill 的处理失败（文件锁定、权限等），记录错误但继续处理其他 skill，不要中断整个流程
