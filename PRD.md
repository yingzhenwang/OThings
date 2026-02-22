# Stuff Manager PRD (Phase 1)

## 1. 产品定位
- 产品名称：Stuff Manager
- 目标用户：single-user（个人用户）
- 核心模式：local-first（本地优先）
- 平台：desktop + mobile web（现代浏览器）
- 数据引擎：SQLite via sql.js

## 2. 目标与非目标

### 2.1 Goals
1. 用最短路径完成 `Add / Find / Update / Delete items`。
2. 在 10k items 规模下保持可用性能（列表、过滤、搜索响应稳定）。
3. 提供可靠 reminders（date-only，本地日历日语义）。
4. 提供可解释、可恢复的数据导入导出（schemaVersion v1）。
5. 建立可维护架构：`Feature-first + Repository + typed contracts`。

### 2.2 Non-Goals (Phase 1)
1. 旧版本数据兼容迁移（clean break）。
2. QR 相关流程（生成/扫描）和照片上传流程。
3. 高级 analytics 与复杂 BI 报表。
4. telemetry/analytics 默认采集。
5. 本地数据库加密能力。

## 3. 用户画像与关键场景

### 3.1 画像
- 个人用户，管理家庭/个人资产。
- 主要在桌面端使用，移动端以查询为主。

### 3.2 Top Scenarios
1. 快速录入物品（名称、分类、数量、位置、状态）。
2. 按关键字/分类/位置筛选并定位物品。
3. 管理保修/维护提醒，避免过期。
4. 导出备份，导入恢复（仅 v1 格式）。

## 4. 功能范围（Phase 1）

### 4.1 In Scope
1. Items CRUD（平衡字段模型：core + details + custom key-value）。
2. Categories 管理。
3. Reminders（date-only）+ Browser Notification。
4. Search：normal 默认；LLM 为可选高级功能。
5. Reports：operational summaries（库存数量、价值、提醒概览）。
6. Import/Export：JSON 导入导出 + CSV 导出（JSON 必须 `schemaVersion`）。
7. Settings：theme/defaultView/itemsPerPage/searchMode/llmProvider/notifications。

### 4.2 Out of Scope
1. QR。
2. Photos。
3. 旧格式兼容与迁移工具。
4. CSV 导入（Phase 2 候选）。

## 5. 信息架构
1. Dashboard
2. Items
3. Categories
4. Reminders
5. Reports
6. Settings

## 6. 关键需求

### 6.1 Search
- 默认模式：normal search。
- LLM search：显式切换；失败必须用户可见；并发请求只保留最新响应。

### 6.2 Reminder
- dueDate 存储格式：`YYYY-MM-DD`。
- overdue/upcoming 判断按本地 start-of-day 计算。
- upcoming 与 overdue 集合互斥。

### 6.3 Settings
- 强类型序列化/反序列化，禁止泛字符串回写。
- 保存后无需整页 reload 即生效。

### 6.4 Import/Export
- 顶层强制 `schemaVersion`。
- Import 分两阶段：`preflight -> commit`。
- preflight 输出：新增数、冲突数、丢弃数、错误数。
- CSV 仅支持导出，不支持导入。
- CSV 导出范围固定为 `items`，不包含 `categories/reminders/settings`。

### 6.6 Reports 指标定义（Phase 1 固定）
- 总物品数：`SUM(items.quantity)`。
- 总价值：`SUM(items.quantity * items.purchase.price)`（仅统计 price 有效项）。
- 当月新增：当前本地自然月内 createdAt 的 item 数量。
- 提醒概览：`upcoming / overdue / completed` 三类计数。

### 6.5 Performance SLO（10k items 数据集）
- 首屏进入 Items 页面（已有本地 DB）P95 <= 2.0s。
- 切换分页（同筛选条件）P95 <= 300ms。
- 普通搜索（normal search）从输入到结果更新 P95 <= 400ms。
- 分类/位置/状态筛选组合变更到结果更新 P95 <= 400ms。
- 任一单次同步阻塞主线程不得超过 100ms（长任务需切分）。

## 7. 成功指标（Phase 1）
1. 主流程成功率：核心页面关键操作无阻断异常。
2. 功能可靠性：提醒日期边界、分页、设置生效、导入导出可复现。
3. 质量门槛：具备 unit + integration baseline。
4. 安全基线：LLM key 不持久化；无默认 telemetry。
5. 性能指标：满足 6.5 的 10k items SLO。

## 8. 发布门槛（Definition of Done）
1. 所有本 docs bundle 定义的 blocker 闭环：
   - DB init 失败可进入 initError 并 Retry 恢复。
   - LLM 失败可见且不影响 normal search。
   - Import 满足 preflight -> commit`，且 `schemaVersion 不匹配拒绝导入。
   - commit 不产生 orphan reminders。
2. 文档 bundle 完整并与实现一致。
3. 核心验收用例全部通过。
4. build 通过且无 blocker 级漏洞。

## 9. 风险与应对
1. 风险：sql.js 初始化失败。
   - 应对：显式 initError + retry。
2. 风险：LLM 服务不可用。
   - 应对：normal search 永远可用；LLM 错误可见且不影响主流程。
3. 风险：导入数据质量差。
   - 应对：safe parse + preflight + reject invalid payload。
