# NH 支付系统规格 v1

> 来源:Claude App 设计对话窗 · 归档 2026-05
> 状态:Phase 0 待落地 / Phase 1 待 Sdn Bhd 注册
> 关联:state/nh-product-spec-v2.md

## 0. 核心结论(5 句话)

1. 4 种支付(USDT / 信用卡 / 二维码 / 网页转账)用同一套 webhook + 统一订单表架构
2. 公司主体选择:**Sdn Bhd 是 NH 多国 SaaS 唯一真路**(Enterprise 天花板低)
3. 没公司前 Phase 0 跑:NOWPayments(USDT) + Lemon Squeezy(信用卡) 覆盖 90% 用户
4. 小商家场景(客人没 USDT 没信用卡):个人收款码 + 双门铃监测(SMS + 截图 AI OCR)
5. 关掉"网页银行转账"选项,引导扫二维码(本质同操作 UX 一致)

## 1. 共通架构机制

下单 → 生成 order_id → `xr_payment_orders` 表 status=pending → PSP webhook 回调 → 验签 + 幂等 → status=paid → 触发 fulfillment(写 `xr_user_quotas` + `xr_quota_transactions`) → 推 toast "X 次已到账"

**4 件必做:**
- HMAC 验签(防伪造 webhook)
- event_id 幂等去重(防重复发奖)
- 30 分钟超时取消(释放库存)
- fallback 轮询(webhook 漏掉时每分钟查 PSP API)

## 2. 4 种支付方式 · Phase 1 推荐组合(注册公司后)

| 支付 | 工具 | 到账时间 | 费率 | 备注 |
|---|---|---|---|---|
| USDT | CryptoMUS / NOWPayments | 1-3 分钟 | 0.4-1% | TRC20 最便宜,$20+ 包才开 USDT |
| 信用卡 | Paddle (MoR) | 秒级 | 5%+5%(含税) | 替代:Stripe(2.9%+$0.30,需 Sdn Bhd) |
| 二维码 | Xendit | 秒级-30秒 | 1.5-2% | 一家覆盖 MY/TH/ID/PH/VN/KH 6 国 |
| 网页转账 | **取消选项** | — | — | 改引导扫二维码,UX 一致 |

## 3. 三阶段路径

### 3.1 Phase 0 · 没注册公司前(NH 现状)

**适用:** 个人身份能签的工具

- **USDT** → NOWPayments(个人可注册,需 KYC 验证)
- **信用卡** → Lemon Squeezy 或 Gumroad(MoR 帮你处理税)
  - Lemon Squeezy: 5% + 50¢/笔
  - Gumroad: 10%/笔(贵但简单)
- **关闭** 网页转账 + 二维码本地银行

**最简组合:** NOWPayments + Lemon Squeezy → 覆盖 90% 用户

### 3.2 Phase 0.5 · 小商家场景(客人没 USDT 没信用卡)

**适用:** 东南亚 80% SMB 客户,只会扫手机银行二维码

**唯一方案:** NH 老板自开各国个人账户(柬埔寨 ABA / 马来 Maybank / 泰国 KBank / 印尼 GoPay / 菲律宾 GCash / 越南 Momo),每国一个码贴出去

**双门铃监测:**

**门铃 1 · 银行 SMS 转发**
- 老板手机收入账 SMS
- 装 SMS forwarder app(Android)自动转给 NH 服务器
- 服务器靠**金额尾数**(订单 $5.07 的 07 是订单号尾)匹配订单 → 自动发货
- **30 秒-2 分钟到账**

**门铃 2 · 截图 + AI OCR 兜底**
- SMS 漏了/老板手机断网,让客人传截图
- AI(Google Vision / GPT-4V)读金额、时间、成功字样
- 对上自动发货,AI 拿不准 admin 5 分钟内审

**类比:家里装两个门铃,哪个响都发货,都不响才让 admin 看。**

**5 个必须知道的坑:**
1. 老板手机要 24/7 在线 — 双轨必须,SMS 单跑不够
2. 金额尾数撞车 — 订单号写转账备注里二次匹配
3. 每国一个账户 — 老板亲自办或当地朋友合作分成
4. 税务雷区 — 个人大额流水各国税局都盯,Sdn Bhd 一下来立刻切公司账户
5. PS 截图诈骗 — AI 必查"水印 + 金额 + 时间戳 + 参考号"四件套缺一不批

### 3.3 Phase 1 · Sdn Bhd 注册后(目标态)

**升级路径:**
- USDT: NOWPayments → 自建 TronGrid 监听(省 0.5% 费率)
- 信用卡: Lemon Squeezy → Stripe(5% → 2.9%+$0.30) + Paddle 备份
- 二维码: 接入 Xendit(覆盖 6 国)
- 多币种公司账户: Maybank / OCBC / HSBC / CIMB(USD/SGD/EUR/JPY)

**总费率从 ~5-10% 降到 ~2-3%。**

## 4. 关键盲点 6 件事(各 PSP 共通)

1. **退款** — USDT 不可退,manual 退;Stripe/Paddle 退款扣已收费率
2. **Webhook 失败** — PSP 重试 3-5 次后放弃,NH 必须 fallback 轮询每分钟查 PSP API
3. **重复 webhook** — 同一笔回调多次很常见,必须幂等(用 event_id 去重)
4. **超时未付** — 30 分钟自动 cancel 释放库存
5. **PSP 资金沉淀风险** — 钱在 PSP 账上,跑路风险存在,**至少 2 家 PSP 备份**
6. **多币种结算** — 各 PSP 结算到 NH 的币种不一(USDT 拿 USDT, Paddle 拿 USD, Xendit 拿本地币),签约看清

## 5. 公司主体 · Sdn Bhd vs Enterprise(多国付款 5 维度)

| 维度 | Enterprise | Sdn Bhd |
|---|---|---|
| PSP 准入 | Stripe 能签 sole proprietor;Paddle/Xendit/2C2P 多数拒收 | 所有大 PSP 默认接受 |
| 多币种账户 | 个人账户能开,大额国际汇入触发银行 AML | 公司 multi-currency 开箱即用 |
| 信用卡风控 | 个人主体 Stripe/Paddle 易被冻结审查 | 公司主体风控分高,稳定 |
| 税务 | 全部并入个人所得,最高 28% | 公司税 17-24%,可留存盈利 |
| 法律责任 | 客户起诉直接诉个人,无限责任 | 有限责任,公司隔离 |

**类比:** Enterprise 是路边摊,Sdn Bhd 是登记店面。国际信用卡公司只敢把钱交给登记店面;路边摊收外汇被银行抽查"这钱哪来的"概率极高,账户随时被冻。

**结论:** NH 多国 SaaS,Sdn Bhd 唯一真路。Enterprise 流量大了一定撞墙。

**外国人注册马来西亚 Sdn Bhd 要点:**
- 100% 外国股东持股可(SaaS 不在限制行业)
- 1 个本地常驻董事(代理 RM 200-400/月)
- 注册地址(虚拟办公 RM 50-150/月)
- 公司秘书(法律强制 RM 100-300/月)
- SSM 注册费 RM 1,000 + 代理服务费 RM 1,500-3,000

**总成本: 一次性 RM 2,500-5,000 + 月固定 RM 350-850**

## 6. 数据层 Schema(Phase 1 新加)

```sql
-- 统一支付订单表(所有支付方式共用)
xr_payment_orders
  id, user_id, sku_id, amount, currency,
  payment_method (usdt/card/qr/sms_match/screenshot_ocr),
  psp_provider (cryptomus/lemonsqueezy/stripe/paddle/xendit/manual),
  psp_order_ref, psp_event_id (for idempotency),
  status (pending/paid/expired/cancelled/refunded/disputed),
  created_at, paid_at, expires_at (default created_at + 30min)

-- 支付审计日志(每个 webhook 都记)
xr_payment_webhook_logs
  id, order_id, psp_provider, event_id, raw_payload,
  signature_valid (bool), processed_at, idempotent_skip (bool)

-- 个人收款码场景(Phase 0.5)
xr_payment_proof_uploads
  id, order_id, screenshot_url, ocr_result_json,
  ai_decision (auto_pass/auto_reject/manual_review),
  admin_decision, admin_id, decided_at

-- SMS 转发匹配(Phase 0.5)
xr_payment_sms_logs
  id, raw_sms, parsed_amount, parsed_sender, parsed_ref,
  matched_order_id, matched_at
```

## 7. 待验证清单(开发前必查)

- [ ] Lemon Squeezy 当前柬埔寨/马来西亚个人注册政策(被 Stripe 收购后可能变)
- [ ] NOWPayments 柬埔寨个人 KYC 通过率
- [ ] Stripe Malaysia 申请 Sdn Bhd 准入门槛(月流水要求)
- [ ] Paddle MoR 在东南亚的实际抽成(可能 5%+5% 是估值,实测可能更高)
- [ ] Xendit Cambodia KHQR 是否已接入(2026 状态)
- [ ] 各国 SMS forwarder app 是否合规(部分国家禁止短信转发)

## 8. 落地路线

### Phase 0 立即可做(无需公司)
- 注册 NOWPayments + Lemon Squeezy
- 落地 xr_payment_orders 表
- 写 2 个 webhook endpoint
- 用户端商店 UI 集成 2 种支付按钮

### Phase 0.5 小商家场景(可选)
- 准备各国个人收款码(柬埔寨 ABA + 马来 Maybank 先行)
- 部署 SMS forwarder + AI OCR 服务
- 落地 xr_payment_proof_uploads + xr_payment_sms_logs 表
- admin 审核 panel

### Phase 1 Sdn Bhd 后升级
- 启动 SSM 名字搜索 + Company Secretary 选定
- 注册下来后申请 Stripe/Paddle/Xendit
- 切换 Lemon Squeezy → Stripe + Paddle
- 加入 Xendit 二维码覆盖 6 国
- 切换个人收款 → 公司多币种账户

## 9. 风险防御(各阶段)

- **PSP 跑路**: 至少 2 家备份,资金不沉淀超过 30 天
- **退款**: 各 PSP 退款流程不同,USDT 完全 manual,信用卡走 PSP API
- **欺诈**: 同 IP / 同 device fingerprint 短时间多笔下单 → 风控触发
- **chargeback**: Stripe/Paddle 1% 内可承受,>1% 账户被审查
- **AML**: 个人账户大额国际汇入触发,Sdn Bhd 后切公司账户
