# Quick App 技术规范（渲染稳定性要求）

> 本文件为 Quick App 构建仪表盘时必须遵守的技术规范，确保页面渲染稳定，不出现白屏。

---

## 1. 全局背景色

必须在全局样式中同时设置 `html`、`body`、`#root` 的背景色，不能只设在组件的 `div` 上。

原因：应用运行在 iframe 沙箱中，iframe 默认背景为白色。如果组件渲染失败被卸载，只设在 div 上的背景色会一起消失，用户只看到白屏。

## 2. 图表库依赖

如果使用 Highcharts：

- 雷达图（polar chart）和气泡图（bubble chart）都依赖 `highcharts-more` 模块
- 正确导入路径为 `highcharts/highcharts-more`，不是 `highcharts/modules/more`
- 热力图依赖 `highcharts/modules/heatmap`
- 缺少任何一个模块会导致图表初始化失败，触发整页白屏

## 3. 错误隔离

- 每个图表组件必须用 ErrorBoundary 或 try-catch 包裹
- 单个图表渲染失败时，显示该图表的错误提示，其余图表正常展示
- 禁止因一个图表崩溃导致整个页面白屏

## 4. CSV 数据解析

- 必须处理 UTF-8 BOM 头（`\uFEFF`），读取后先清除
- 兼容 `\r\n` 和 `\n` 两种换行符
- 自动定位包含 `name` 和 `tech_depth` 字段的表头行（CSV 前面可能有额外的元数据行）
- 所有评分字段（tech_depth 等）解析为数字类型，非数字值视为 0
- 跳过空行

## 5. 空数据保护

- 数据加载中时显示加载提示（如 spinner 或文字）
- 数据加载失败时显示明确的错误信息（如"数据加载失败，请检查 CSV 文件"）
- 数据为空（0 条记录）时显示提示信息
- 以上三种情况均不能显示空白页面

