# NH 商业模式 v2 + 用户端 IA 完整规格

> 来源:Claude App 设计对话窗 · 归档 2026-05
> 状态:Phase 1 MVP 待开发

## 0. 核心结论(5 句话)

1. NH 不是 CRM,是**痛点驱动的 AI 工具集成 OS**(用户卡哪 NH 帮哪)
2. 商业模式不订阅,卖功能次数(超市模式:商品 + 价格 + 购物车)
3. 任务奖励直接送某 SKU 次数(不送积分),用户立刻能用
4. 现有 xr_quota 体系不重构,只加 3 张新表 + 扩展字段
5. NH 护城河 = **中间层**(痛点检测 + 智能推销 + 闭环串联),不是商店本身

## 1. NH 真实定位

**3 句话哲学:**
- 用户没有 → NH 辅助直到有
- 用户不会 → NH 辅助直到会
- 用户有要求 → NH 实现直到满意

**类比:** Shopify(工具市场抽成) + Notion AI(AI 嵌入闭环) + 闭环 OS(内核) 三合一。

## 2. 6 阶段闭环工作流(用户主线)

| # | 阶段 | 用户做 | 系统帮 | 支撑工具(情境触发) |
|---|---|---|---|---|
| 1 | 搜索 | 关键词+地区 | 抓 3000 名单 | 关键词库,地区设置 |
| 2 | 过滤 | 勾选条件 | AI 评分筛 200 精准 | AI 评分模型 |
| 3 | 外联 | 选渠道 | AI 写话术+批量发+追踪 | 内容助手,素材库 |
| 4 | 跟进 | Pipeline 拖卡片 | 提醒+回复建议 | 素材库,行业学堂 |
| 5 | 成交 | 标记+录金额 | 报价/合同/付款 | 报价模板 |
| 6 | 复购/裂变 | 选老客户 | AI 唤醒+佣金跟踪 | 系统全自动 |

**设计:** 支撑工具情境式触发,不在主菜单单独列。

## 3. 用户端 IA

### 3.1 Dashboard(状态驱动)
3 块从上到下:
- 顶部"你现在该做什么"大卡片(检测当前状态自动给指令+主行动按钮)
- 中部 6 阶段进度条(高亮当前)
- 下部当前阶段数据 + 今日必做

### 3.2 主菜单 13 项 / 3 组
- 主流程(6): Dashboard / 搜索任务 / 名单库 / 外联中心 / Pipeline / 裂变
- 支撑工具(4,默认折叠): 内容助手 / 素材库 / 行业学堂 / 关键词库
- 后台(3): 数据 / 套餐 / 设置

### 3.3 Onboarding 3 步 wizard
1. 你做什么生意?(行业模板)
2. 你想找什么客户?(关键词+地区)
3. 完成 → 自动跑搜索 30s 给 3000 名单 → Dashboard 显示"你刚搜到,下一步过滤"

### 3.4 Pipeline(统一对象 + Stage 视图)
1 customer 表 + stage 字段(lead/following/won/lost),3 视图切换:看板 / 列表 / 详情。
取代原 leads/followup/deals 三模块。

## 4. 商品 3 层结构

```
类目(Category)
  └── SKU(Tier - 后端不同 API)
        └── 包装(Pack - 大小不同次数)
```

**例 - 搜索类目:**
- 基础(Google Maps · 60% 准确): 小 100/$5, 中 500/$20(划算), 大 2000/$60(最划算)
- 标准(Apollo · 85% 准确): 小 100/$15, 中 500/$60, 大 2000/$200
- 高级(决策人+动态 · 95% 准确): 小 50/$30, 中 200/$100, 大 1000/$400

类目: 搜索 / 图生成 / 视频 / 外联 / 数字人 / 组合优惠包(待扩展)

## 5. 商业模式 v2

### 5.1 核心机制
- 充值买:USD 标价,多币种切换,第一次付款锁币种
- 任务挣:完成任务直送某 SKU 次数(不送积分)
- 用一次扣一次,按 SKU 记账,不同档次不互通

### 5.2 多币种
- USD 默认,IP 自动检测推荐本地币种
- 第一次付款锁定币种(防 VPN 套利)
- admin 月调汇率,小 N 每月从央行自动拉建议

### 5.3 库存感知商店
每商品卡片显示用户当前持有量:
- 绿(充足) - 含 7 天用量预测
- 黄(快用完) - 提示该补
- 灰(没买过) - 推荐试小包

stock-row 告诉 3 件事: 剩多少 / 用得多快 / 还能用多久

### 5.4 我的次数专页
顶部 stat: 需补货 / 充足 / 没买过
按类目分组所有 SKU 库存 + 用量趋势 + 一键补货按钮

### 5.5 购物车多选打折 + 小 R 推销
admin 配 3 类规则:
- 件数: 2+/3+/4+ → 5/8/12% off
- 总额: ≥$50/$100 → 10%/15% off
- 类目组合: 搜索+外联 / 图+视频 → 额外 5%

小 R 智能推销:基于用量+剩余次数主动推 cross-sell

## 6. 5 个关键决策(已锁定)

| # | 决策 | 方案 |
|---|---|---|
| 1 | 汇率 | admin 月调 + 小 N 自动从央行拉建议 |
| 2 | 币种 | 浏览自由切;第一次付款锁定那国币种 |
| 3 | 任务奖励 | admin 可调;注册送少,第一笔交易再追加 |
| 4 | 过期 | 任务积分 12 月不动过期;充值/购买的次数永不过期 |
| 5 | 打折 | 必须打折,admin 配规则 + 小 R 主动推销 |

## 7. Admin 后台 3 张缺的 panel

### 7.1 币种汇率配置
- 基准 USD + 货币列表(RM/THB/IDR/PHP/VND...)
- 每行: 当前汇率 + 上次更新 + 小 N 建议(↑↓)
- 套用全部建议 + IP 自动检测 toggle + 第一次付款锁币 toggle

### 7.2 购物车折扣规则
- 件数折扣 / 总额折扣 / 类目组合奖励
- 启用小 R 智能推销 toggle

### 7.3 任务奖励 SKU 字段扩展
在现有 OnboardingTasksPanel 加列:
- xr_onboarding_task_defs 加 reward_sku_id + reward_amount
- SKU 选择器(从所有 SKU 列表选)+ 次数 input + 启用 toggle

## 8. 后端中间层 5 个核心服务

### 8.1 痛点检测引擎
信号: 配额耗尽/转化突降/操作失败 3 次/低于行业基准
触发: 卡住瞬间推,不是事后
输出: 推荐工具 + 当前数据 + 预期 ROI

### 8.2 小 R 智能推销
输入: 当前剩余次数 + 7 天用量 + 当前购物车
Phase 1 规则触发,Phase 2 ML 预测

### 8.3 IP + 第一次付款锁币种
IP 检测推荐 → 第一次付款记录国别币种 → 后续锁定

### 8.4 配额预测算法
当前余额 / 7 天日均用量 = 还能用多少天
推动 stock-row 状态切换

### 8.5 任务完成 → 自动发奖事件链
监听任务事件 → 读 reward_sku_id+reward_amount → 写 xr_user_quotas + xr_quota_transactions(source="task_reward") → UI 弹奖励 toast

## 9. 闭环 ↔ 商店串联(关键回路)

用户在闭环某关卡卡住 → 痛点检测推商品 → 买 → 回闭环

例:
1. 搜索阶段(结果 < 50 条)
2. 痛点检测识别"搜索能力不足"
3. 推商店"搜索·标准/高级" + 预期 ROI
4. 用户买 → 自动跳回搜索阶段使用新次数
5. 转化数据回流痛点检测模型,下次更精准

**没这层,商店和主流程是孤岛,NH 沦为普通 Shopify。**

## 10. 现有 admin v2 盘点(CC grep 验证)

**Main HEAD:** ffbbcc3 (FollowupPagePC v4)

**Admin v2 panels 32 个分类:**
Plans / Packs / Quota / Referral / Promo / Invoices / Onboarding / Health / Audit / Alerts / Funnel / Growth / Domain

**xr_ 表 ~40 张分组:**
- 用户/认证: profiles, user_plans, user_quotas, quota, quota_transactions
- 钱/计划: plans, plan_versions, plan_prices, plan_quotas, pack_skus, pack_purchases, promo_codes, invoices, price_ab_tests
- 业务: orders, gifts, gift_catalog, gift_codes, tasks, messages, xiaor_messages
- 增长: search_tasks, search_leads, referrals, outreach_log, user_recall_log, user_streaks, daily_checkins, effort_scores
- 内容: content_briefs, content_generations, content_materials, content_schedule, content_sessions, content_history, content_auth, content_device, content_session
- 系统: system_flags, system_health, api_health, features, kpi_snapshots, admin_broadcasts, notifications, otp_codes, push_subscriptions, holiday_extensions, experience_pack_campaigns, daily_free_quotas, onboarding_task_defs, urgent_followups, user_goals, user_predictions, user_onboarding_progress, weekly_reports, domain_ssl

**关键缺失:**
- 无 Wallet/Credit/Exchange/Cost/Topup 类 panel
- 无独立 wallet/credit ledger 或 exchange-rate 表
- 无"折抵/抵扣/auto-credit"代码(0 命中)

**结论:** 现有架构走配额制,商业模式 v2 走 C 路径(保留 quota 体系,加 3 张新表+扩展字段)。

## 11. 落地路线

### Phase 1 MVP(必须)

**数据层 A:**
- 扩展 xr_pack_skus: 加 category, tier, pack_size, pack_label
- 扩展 xr_onboarding_task_defs: 加 reward_sku_id, reward_amount
- 新加 3 张:
  - xr_currency_rates(基准 USD + 各币种汇率 + 更新时间)
  - xr_cart_discount_rules(件数/总额/类目组合规则)
  - xr_user_currency_lock(用户首次付款币种锁定)

**Admin 后台:** 币种汇率 + 折扣规则 + 任务奖励 SKU 扩展(3 张缺的 panel)

**用户端:**
- NH 商店主页(3 层 + 库存感知 + 多币种)
- 购物车(多选打折 + 小 R 推销卡)
- 我的次数专页
- 任务奖励到账 toast

**简单版后端 B:**
- IP+付款锁币种
- 配额预测算法
- 任务完成事件链
- 简单痛点检测(1-2 触发点)
- 简单 cross-sell(规则触发)

### Phase 2 优化
- 智能小 R 推销(ML 用户行为模型)
- 痛点检测引擎(全维度信号)
- ROI 报告
- Onboarding wizard 完整体验
- 闭环 ↔ 商店深度串联
- 6 阶段闭环工作流完整落地

## 12. 已产出 mockup 列表(本对话窗)

**用户端 (8 张):**
1. NH 商店主页(3 层 + 库存感知 stock-row)
2. 购物车 + 小 R 智能推销
3. 我的次数库存页
4. Dashboard 状态驱动
5. Pipeline 看板
6. 6 阶段闭环 + 支撑工具关系图
7. Sidebar 菜单 before/after 对比
8. 商业模式 v2 骨架图

**Admin 后台 (3 张):**
1. 币种汇率配置 panel
2. 购物车折扣规则 panel
3. 任务奖励 SKU 字段扩展 panel

## 13. 待 Ray 拍板

- 折扣百分比具体数字(mock 5/8/12/10/15 是占位)
- 各 SKU 具体次数和定价(待对接 API 衡量真实成本)
- 任务奖励具体 SKU 选择(注册送什么/邀友/成交)
- 行业模板列表(onboarding 第 1 步选项)
- 商业模式 v2 是否完全替代订阅,还是订阅作为大客户保留

## 14. 风险防御(Phase 2 处理)

- 退款政策
- VPN 套利防御
- 机器人薅羊毛(任务奖励大量注册)
- API 供应商绑架(同类 2-3 个供应商可切换)
- AI 内容审核(话术不当生成)
