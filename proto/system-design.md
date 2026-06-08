# System Design — Live MCN 營運系統（純文字稿）

> **讀者**：其他工程師。本文是 pitch 的文字底稿，之後依此產出 **HTML 架構圖 + 說明文件**。
> **來源場景**：以 `proto/user-story.md` 的「同場多品牌直播檔次」為唯一 worked example，所有資料模型/流程/結算都對得上那組假資料（LS-0918）。
> **與 `pages/page3.html` 的關係**：page3 是給自己人看的三層架構鳥瞰；本文是它底下的工程規格——同一套五大資料庫，但展開到欄位、關係、狀態機、結算邏輯與落地選型。

---

## 1. 設計原則（不可違反的硬約束）

| 原則 | 對系統的意義 | 來源 |
|---|---|---|
| **B2B-only · 不碰 C 端** | 系統不接金流、不持顧客個資、不負客服。消費者側全在渠道/品牌之間。 | brief §7.10 |
| **Agent / Net 認列** | 只認佣金為營收，不認 GMV 全額。資料模型必須能單獨算出「佣金」這個數。 | brief §2、§7.1 |
| **平台中立** | 不綁定任何收單系統（就醬播只是選項之一）。對接層走 adapter，先期可手動匯入。 | brief §7.9 |
| **含稅 / 未稅雙向** | 每個價格帶帶 `tax_inclusive` flag，系統自動換算。 | brief §2 |
| **AI 不做決策** | AI 做記錄、案型模仿、評分、決策建議；人拍板。條例歸案是基礎。 | brief §7.7、§12 |
| **直播主分級內部 only** | S/A/B/C 是 DB 欄位，對外不可見（外部只說 Tag 化配對）。 | brief §7.8、附錄 B-10 |

---

## 2. 概念資料模型（與工具無關）

對應 page3「五大資料庫」，但展開為實體 + 關係。**核心是 `Session`（直播檔次）這個聚合根**，它把五庫 join 起來。

### 2.1 實體與關鍵欄位

```
Venue 場地         venue_id(PK) · name · capacity · layout · photos[] · is_self_owned=false
Brand 品牌         brand_id(PK) · name · tags[] · contact · terms(json)
Product 商品       product_id(PK) · brand_id(FK) · name · category_tags[]
Quote 報價         quote_id(PK) · product_id(FK) · tax_inclusive(bool) · supply_price · suggested_price · valid_range
Creator 直播主     creator_id(PK) · name · tags[] · grade(S/A/B/C) · channels[]        ← grade 內部 only
DealType 案型      deal_id(PK) · type(buy_n_get_m|bundle|threshold_gift) · rule(json) · ai_source_ref

Session 直播檔次   session_id(PK) · venue_id(FK) · creator_id(FK) · scheduled_at · status   ← 聚合根
LineItem 選品行    line_id(PK) · session_id(FK) · product_id(FK) · deal_id(FK) · unit_price · tax_inclusive
SettlementLine 結算行
                   settle_id(PK) · session_id(FK) · brand_id(FK)            ← 歸戶 = (Session × Brand)
                   · gmv · returns · net_gmv · take_rate · commission
                   · commission_base(gross|net)                            ← B1/B2 可配置，不寫死
DisputeTicket 爭議單
                   dispute_id(PK) · settle_id(FK) · shipped_qty · reported_qty · diff · status
Scorecard 場後評分 score_id(PK) · session_id(FK) · brand_id(FK) · ai_grade · feedback
```

### 2.2 關係（文字版 ER，供之後畫 HTML 圖）

```
Brand ─1:N─ Product ─1:N─ Quote                  （品牌庫）
Creator ······ tags/grade                         （直播主庫）
Venue                                             （場地庫）
DealType ······ AI 可生成                          （案型庫）

Session ─N:1─ Venue
Session ─N:1─ Creator
Session ─1:N─ LineItem ─N:1─ Product
                       └─N:1─ DealType
Session ─1:N─ SettlementLine ─N:1─ Brand          （結算庫；每場每品牌一行）
SettlementLine ─1:0..1─ DisputeTicket
Session × Brand ─1:1─ Scorecard
```

> **關鍵設計決定**：結算的歸戶粒度是 **(Session × Brand)**，不是 Session、也不是 Brand。
> 這一刀直接解掉 **B4（同場多品牌歸戶）**——LS-0918 一場三品牌 = 三條 SettlementLine。
> 「同場單品牌」自動是退化情況（一條），日後不需重構。先用最複雜形狀設計。

---

## 3. 直播檔次狀態機（Session.status）

```
 草稿 ──→ 已排程 ──→ 直播中 ──→ 對帳中 ──→ 已結算
draft   scheduled    live      reconciling  settled
  │         │
  └────→ 取消 cancelled

  對帳中 期間：若任一 SettlementLine 出現 出貨數 ≠ 回報數 → 開 DisputeTicket（B5）
              dispute: 開立 → 對帳中 → 裁決 → 結案
              所有 DisputeTicket 結案後，Session 才可 → 已結算
```

- LS-0918 目前在 **對帳中**：綠研所出貨 215 ≠ 回報 210 → 爭議單 `#D-0918-01`「對帳中」，卡住整場結算。
- 結算時機（**B1**）= 「commission 在哪個狀態被認列」：首批下訂時 / 退貨期後 / 分兩次——做成設定，不寫死。

---

## 4. 流程（直播前 / 中 / 後）對應模組與資料寫入

| 階段 | 動作 | 模組 | 寫入 |
|---|---|---|---|
| **前** | 品牌進件、報價建檔、打 Tag | 品牌管理 | Brand · Product · Quote |
| 前 | Tag 配對 Bella × 三批商品 | 配對引擎 | （讀）Creator · Product |
| 前 | 案型設計（AI 模仿）+ 銷量預估 | 案型 + AI | DealType |
| 前 | 選品、定價（含稅/未稅換算） | 選品 | LineItem |
| 前 | 排場地、排程、建 Session | 排期 | Session(草稿→已排程) · LineItem |
| **中** | 場控、流程、即時數據 | 場控 | Session(直播中) |
| **後** | 雙邊回報、對帳、開爭議單 | 對帳 | SettlementLine · DisputeTicket |
| 後 | 退貨歸屬計算 | 結算 | SettlementLine.returns（B2） |
| 後 | 多品牌分開歸戶、開佣金 | 結算 | SettlementLine.commission（B1/B4） |
| 後 | AI 場後評分、數據回饋 | AI | Scorecard |
| 後 | 三視角報表 | 報表 | （讀）全庫 |

---

## 5. 結算與佣金模型（Agent / Net）

```
每條 SettlementLine（場次 × 品牌）：
  net_gmv   = gmv − returns
  base      = (commission_base == gross) ? gmv : net_gmv        ← B1/B2 可配置
  commission= base × take_rate

我方營收認列 = Σ commission（不認 GMV）          → LS-0918：25,440
營業稅 5% 基數 = Σ commission                    → 小且乾淨
直播主團隊淨分潤 = net_gmv − 品牌供貨成本 − commission
品牌方 = supply_price × 售出數 − returns
```

LS-0918 worked example（以 net 為基數）：

| 品牌 | gmv | returns | net_gmv | commission(8%) |
|---|---|---|---|---|
| 綠研所 | 180,000 | 12,000 | 168,000 | 13,440 |
| 膚光 | 95,000 | 5,000 | 90,000 | 7,200 |
| 暖窩 | 60,000 | 0 | 60,000 | 4,800 |
| **Σ** | 335,000 | 17,000 | 318,000 | **25,440** |

含稅/未稅：每條價格帶 `tax_inclusive` flag，結算前統一換算到同一基準（×1.05 / ÷1.05），避免混算。

---

## 6. 平台中立整合層（adapter）

```
[ 就醬播 ] ┐
[ 收單系統 B ] ├─→ Adapter（正規化為 SettlementLine 草稿）─→ 結算庫
[ 收單系統 C ] ┘
[ 手動匯入 CSV ] ────────────────────────────────────────┘   ← 先期預設
```

- 先期**不對接 API**，走手動匯入（brief §9：先期可手動匯入）。
- 要對接時，每個收單系統寫一個 adapter，把它的銷售/退貨資料正規化成我方 SettlementLine——核心模型不變。
- 系統**永不**接觸 C 端金流/個資；adapter 只收「銷售收斂量」這類 B2B 對帳數字。

---

## 7. AI 治理層

| 能力 | 做什麼 | 工具方向 |
|---|---|---|
| 案型模仿 | LLM + few-shot 學專家案型（需先有零散商品清單） | Claude / Claude Code |
| 銷量預估 | 依品類/直播主歷史給預估 | LLM + 歷史資料 |
| 場後評分 | 每品牌獨立 Scorecard + 數據回饋 | LLM |
| 決策諮詢 | 跨資料源查詢，提供決策資訊與建議（**不拍板**） | 自架 agent + 知識庫 |
| 條例歸案 | 原則/合約條款/TBD 決議入庫，living spec | Notion / Obsidian |
| 會議紀錄 | 轉錄、摘要、行動項 | Fireflies / Otter |

---

## 8. 系統邊界（明確不做）

- ❌ C 端下單、收款、發票、退款、客服 → 渠道/品牌
- ❌ 持有消費者個資
- ❌ 跨境實際金流、報關（只諮詢）→ 品牌（出貨）↔ 直播主（收貨）
- ❌ 養護自有場地（只記錄容納/格局/照片）
- ❌ AI 自動拍板決策

---

## 9. 落地選項對比：自建 vs 拼裝現成工具

技術可行性整體 🟢-🟡，**無真正 blocker**（brief §9）；最大不確定性在商業條款（B1-B5），不是工程題。因此**不該一開始就重金自建**。

| 子系統 | 拼裝現成工具 | 自建 | 建議 |
|---|---|---|---|
| 五大資料庫 CRUD | Airtable / NocoDB 🟢 | 過度投資 | **拼裝** |
| Session 聚合 + 狀態機 | Airtable 自動化勉強 🟡 | 清楚但要工 🟢 | 拼裝起步 → 痛了再自建薄層 |
| 結算 / 對帳 / 多品牌歸戶 (B1-B5) | 現成工具弱 🔴 | 必要 🟢 | **自建薄層**（核心價值所在） |
| 含稅/未稅換算 | 公式欄位可 🟡 | 簡單 🟢 | 隨結算層自建 |
| 平台中立 adapter | — | 每系統一個 🟡 | 延後，先手動匯入 |
| 三視角 UI（本原型） | 工具內建視圖醜 🟡 | 自建前端 🟢 | 階段 2 自建（先用原型對齊） |
| CRM（品牌/直播主） | HubSpot / Notion 🟢 | — | **拼裝** |
| AI 治理 | Claude Code + Notion + Fireflies 🟢 | — | **拼裝** |

### 建議路線（分階段，對齊 brief 時程）

- **Phase 0 · MVP（2026-07）**：五庫 + Session 用 Airtable/NocoDB；知識庫 Notion；會議 Fireflies；AI 用 Claude Code。手動匯入銷售資料。目標：跑得動。
- **Phase 1 · 試營運（2026-08）**：**自建結算薄層**——多品牌歸戶 (B4)、含稅未稅、B1/B2 可配置佣金基數、爭議單 (B5)。這是現成工具做不好、又是我們營收命脈的部分。跑通 1-2 場、發現坑修坑。
- **Phase 2 · 對外開跑（2026-09+）**：自建三視角前端（本原型即藍圖）；視需要加收單系統 adapter。

---

## 10. 風險／待決掛鉤（哪個設計卡在哪條 TBD）

| 設計點 | 卡在 | 影響 |
|---|---|---|
| `commission_base` gross vs net | **B1 / B2** | 佣金金額、認列時點、現金流 |
| 退貨損益歸屬欄位 | **B2** | SettlementLine 要不要記「誰吃退貨」 |
| 補單 take rate | **B3** | LineItem 是否區分首單/補單費率 |
| (Session × Brand) 歸戶 | **B4** | 已採用為核心粒度（本文的關鍵決定） |
| DisputeTicket 狀態流 | **B5** | 裁決流程定不下來 → 結算卡關 |
| 直播主視角資料可見度 | **S1** | 隱藏讓價空間，防跳單（見原型直播主視角） |

---

## 11. 下一步

1. 本文 review 定稿後 → 產出 **HTML 架構圖**（沿用 page3 視覺：三層 + 五庫 + Session 聚合根 + 狀態機 + 結算流），作為對工程師的 pitch 主畫面。
2. 圖旁附 **說明文件**（即本文精簡版 + worked example）。
3. 鎖定 **Phase 1 自建結算薄層**的 schema 與 API，進入實作規劃。
