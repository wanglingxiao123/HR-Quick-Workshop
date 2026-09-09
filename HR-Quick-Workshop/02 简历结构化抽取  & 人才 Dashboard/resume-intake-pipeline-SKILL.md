---
name: resume-intake-pipeline
display_name: "简历入库 + 结构化提取"
description: "从 Quick Suite Space 中扫描新增简历（PDF/DOCX/MD），自动提取结构化字段并生成/更新 talent_pool.csv。支持定时执行和手动触发。触发词：处理新简历、简历入库、resume intake、简历提取、更新人才库。"
icon: "📋"
trigger: 处理新简历 简历入库 resume intake 简历提取 更新人才库 人才库更新
inputs:
  - name: space_name
    description: "存放简历文件的 Quick Suite Space 名称"
    type: string
    required: true
    default: "简历"
  - name: dataset_url
    description: "QuickSight Dataset 编辑页面 URL（用于提醒用户更新数据源）"
    type: url
    required: false
    default: ""
tools: [list_quick_suite_directory, list_space_documents, get_document_download_url, download_file, read_quick_suite_file, file_read_pdf, file_read_docx, file_read, run_python, file_write, folder_create, open_in_session_tab, send_notification, create_task_group, start_task, get_task_group_result, inspect_task]
id: fc918bbac6b9431c800e62781fa9d160
---

## Overview

本 Skill 实现 HR 人才库的简历入库自动化流程：扫描 Quick Suite Space 中的新增简历文件，通过 LLM 提取结构化字段（姓名、技能、工作年限、学历等），追加到 `talent_pool.csv`，并通知用户更新 QuickSight Dataset。

适用场景：
- 定时 Agent 每天自动扫描处理新简历
- 用户手动说"处理新简历"立即触发
- 与需求 4（市场人才趋势分析）共享结构化数据

## 文件存储路径规范（关键）

`agent_files/` 是 Amazon Quick App 内置存储，跨 Session 全局持久化，无需用户手动授权。读写方式：`run_python` 生成内容字符串，`file_write` 工具写入路径。**禁止**用 Python `open()` 写入 agent_files（沙箱报 PermissionError）。

```
agent_files/
├── resume_data/
│   └── talent_pool.csv          ← 人才库主文件
└── resume_tracker/
    └── processed.json           ← 增量追踪：记录已处理文件名+日期
```

## Workflow

### Step 0: 检查 Space 是否已附加（前置守卫）
- **Mode**: `deterministic`
- **检查方式**：查看系统上下文中的 `<attached_spaces>` 列表
- **通过条件**：`space_name`（默认"简历"）出现在已附加 Space 名称列表中
- **不通过时**：**立即停止，不执行后续步骤**，向用户发送以下提示：

> ⚠️ 未检测到已附加的简历 Space。请先在对话侧边栏中附加包含简历文件的 Space（名称应为"**{{space_name}}**"），否则无法读取简历文件。附加后请重新触发此操作。

此步骤是必要的前置守卫——没有 Space 就无法扫描文件，后续所有步骤都依赖它。

### Step 1: 加载已处理记录
- **Mode**: `deterministic`
- **Tool**: `file_read`
- **Input**: `agent_files/resume_tracker/processed.json`
- **Output**: 已处理文件名列表（JSON 对象，key 为文件名，value 为处理日期）
- **Validate**: JSON 解析成功
- **On failure**: 如果文件不存在，说明是首次运行，初始化为空对象 `{}`

### Step 2: 扫描 Space 获取当前文件列表
- **Mode**: `deterministic`
- **Tool**: `list_quick_suite_directory`
- **Input**: `space_name` = `{{space_name}}`
- **Output**: Space 中所有文件名列表
- **Validate**: 返回非空列表
- **On failure**: 提示用户确认 Space 名称是否正确

### Step 3: 计算增量（新增简历）
- **Mode**: `deterministic`
- **Tool**: `run_python`
- **Input**: Step 1 的已处理记录 + Step 2 的当前文件列表
- **Output**: 新增简历文件名列表
- **Validate**: 列表可以为空（无新简历）

过滤逻辑：只处理 .pdf / .docx / .md 结尾的文件，排除已处理过的文件。

**如果没有新增简历**：跳过 Step 4，直接进入 Step 5（依然向用户返回当前 CSV）。

### Step 4: 提取结构化字段
- **Mode**: `agentic`
- **Tool**: `read_quick_suite_file` + `file_read_pdf` / `file_read_docx` + LLM 提取
- **Input**: Step 3 的新增简历文件名列表
- **Output**: 结构化 JSON 数组
- **Validate**: 每条记录包含所有必需字段
- **On failure**: 记录失败的文件名，跳过继续处理其他文件

并行策略：
- 新增 5 份以内：当前 Agent 直接逐份处理
- 新增超过 5 份：使用 `create_task_group` + `start_task` 并行处理，每个 task 处理 5 份

**每份简历的下载流程（含降级策略）：**

**主路径**：`read_quick_suite_file(space_name="{{space_name}}", document_name=文件名)`

如果返回 `null`，先重试一次。若仍为 `null`，执行降级路径：
1. `list_space_documents(space_id)` 获取完整文档列表，找到目标文件的 `documentId`
   - space_id 从 `<attached_spaces>` 中读取，或用 `search_spaces(query=space_name)` 查询
2. `get_document_download_url(space_id, document_id)` 获取预签名 S3 URL
3. `download_file(url=presignedS3Url, filename=文件名)` 下载到 `workspace/downloads/`

**批量降级优化**：同一批次多个文件失败时，一次 `list_space_documents` 获取所有 documentId，再并行 `get_document_download_url` + `download_file`，避免重复查询。

下载成功后按文件类型读取：
- `.pdf` → `file_read_pdf`
- `.docx` → `file_read_docx`
- `.md` → `file_read`

LLM 提取字段：

必需字段：name, gender（无法判断填"未知"）, age（按毕业年份估算）, education_level（本科/硕士/博士）, university, major, work_years（从最早工作时间算到当前年份）, current_role, current_company, career_path（如"公司A → 公司B → 公司C"）, skills（英文逗号分隔）, industry, resume_source（自动填充文件名）

可选字段：certificates, target_position, expected_city, expected_salary

### Step 5: 合并写入 CSV
- **Mode**: `deterministic`
- **Tool**: `file_read` + `run_python` + `file_write`
- **Input**: Step 4 的结构化 JSON（若无新简历则为空） + 现有 `agent_files/resume_data/talent_pool.csv`（如存在）
- **Output**: 更新后的 `talent_pool.csv`（新增时追加，无新增时不修改内容）
- **Validate**: CSV 行数 = 原有行数 + 新增记录数

CSV 字段顺序：name, gender, age, education_level, university, major, work_years, current_role, current_company, career_path, skills, industry, certificates, target_position, expected_city, expected_salary, resume_source

编码：**utf-8-sig（带 BOM）**，列名使用英文，确保 Excel 和 QuickSight 正确识别中文。

合并算法：用 `file_read` 读取现有 CSV 内容 → `run_python` 解析并追加新行（`pd.concat([existing_df, new_df], ignore_index=True)`）→ `file_write` 写回 agent_files 路径。**不能**用 Python `open()` 直接写入 agent_files。

### Step 6: 更新已处理记录
- **Mode**: `deterministic`
- **Tool**: `file_write`
- **Input**: 更新后的 processed.json（新增已成功处理的文件名和日期）
- **Output**: 写入 `agent_files/resume_tracker/processed.json`
- **Validate**: 文件写入成功
- **On failure**: 重试一次

### Step 7: 向用户返回结果（必须执行）
- **Mode**: `deterministic`
- **Tool**: `open_in_session_tab` + `send_notification`
- **注意**：**本步骤无论是否有新增简历，都必须执行**。哪怕 Space 中没有新文件，也要打开现有 CSV 让用户确认当前人才库状态。

执行内容：
1. `open_in_session_tab("agent_files/resume_data/talent_pool.csv")` — 在 Session Tab 中打开 CSV
2. `send_notification` — 发送通知

通知内容需包含：
- 本次新增处理了多少份简历（0 也要明确说明"无新增"）
- 处理失败了多少份（如有）
- 人才库当前总人数
- 如果提供了 dataset_url，附上 QuickSight Dataset 更新步骤指引

## Output

- `agent_files/resume_data/talent_pool.csv` — 结构化人才库数据（主文件，跨 Session 持久化）
- `agent_files/resume_tracker/processed.json` — 已处理文件记录（增量检测用）
- Session Tab 展示 + 通知消息

## Lessons Learned

### Do
- **每次运行结束都必须打开 CSV**，无论有无新简历，用户需要看到当前人才库状态
- **先检查 Space 是否已附加**，没有 Space 立即停止并提示用户，不要继续执行
- 使用 `list_quick_suite_directory` 获取文件列表，返回简洁的文件名列表
- CSV 用 utf-8-sig 编码（带 BOM），确保 Excel 和 QuickSight 正确识别中文
- 并行任务数超过 5 时使用 task group
- 技能字段统一用英文逗号加空格分隔，方便后续分析
- 写入 agent_files 路径必须用 `file_write` 工具，不能用 Python `open()`（沙箱限制）
- 同一批次多个文件下载失败时，一次性调用 `list_space_documents` 批量获取 document_id，再并行 `get_document_download_url` + `download_file`

### Don't
- 不要在 CSV 中使用中文列名，QuickSight 对英文列名支持更好
- 不要假设所有简历格式一致，PDF 和 DOCX 结构差异大，需要通用的 LLM 提取
- 不要一次性处理超过 50 份简历而不通知用户确认
- **不要因为"没有新简历"就跳过 Step 7**——用户仍然需要看到 CSV

### Common Failures
- **Space 未附加**：Step 0 守卫会拦截，提示用户在侧边栏附加 Space
- Space 名称错误：`list_quick_suite_directory` 返回 null，提示用户确认 Space 名称
- **`read_quick_suite_file` 返回 null**：先重试一次，仍失败则走降级路径：`list_space_documents` → `get_document_download_url` → `download_file`
- PDF 提取乱码：某些扫描版 PDF 文本质量差，记录为失败并建议上传 DOCX
- 字段缺失：用"未知"填充，不阻塞流程
- 同一人多份简历：暂不去重，保留所有版本

### When to Ask the User
- Space 名称无法识别时
- 新增简历超过 50 份时（确认是否批量处理）
- 提取字段与现有 CSV 结构不匹配时
