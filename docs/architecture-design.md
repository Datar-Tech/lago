# Lago 架構設計文件

> 版本：v1.43.0 | 最後更新：2026-03-18

---

## 目錄

1. [全端架構設計 (Full Stack Design)](#1-全端架構設計)
2. [資料庫 Schema (DB Schema)](#2-資料庫-schema)
3. [排程任務 (Cron Jobs)](#3-排程任務-cron-jobs)
4. [Webhook 設計](#4-webhook-設計)
5. [事件處理管線 (Event Processing Pipeline)](#5-事件處理管線)
6. [環境設定 (Environment Configuration)](#6-環境設定)
7. [擴展建議 (Scaling Guidelines)](#7-擴展建議)

---

## 1. 全端架構設計

### 1.1 整體架構圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                        外部世界 (Internet)                           │
│                                                                     │
│   客戶端 App          第三方付款商             管理後台              │
│  (API Calls)        (Stripe/PayPal)         (Dashboard)            │
└────────┬───────────────────┬──────────────────────┬────────────────┘
         │                   │                      │
         ▼                   ▼                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Traefik (反向代理 / TLS)                        │
│               api.lago.dev    app.lago.dev                         │
└────────┬───────────────────────────────────┬────────────────────────┘
         │                                   │
         ▼                                   ▼
┌─────────────────┐                ┌──────────────────┐
│   Rails API     │                │  React Frontend  │
│  (Port 3000)    │                │   (Port 80/nginx) │
│                 │                │                  │
│ - REST API      │                │ - 訂閱管理        │
│ - GraphQL       │                │ - 帳單查看        │
│ - Event Ingest  │                │ - 用量儀表板      │
│ - Webhook Mgmt  │                │ - 客戶管理        │
└────────┬────────┘                └──────────────────┘
         │
         ├─────────────────┬────────────────────┐
         │                 │                    │
         ▼                 ▼                    ▼
┌──────────────┐  ┌──────────────────┐  ┌─────────────┐
│  PostgreSQL  │  │  Redis (3實例)    │  │  Redpanda   │
│  (Port 5432) │  │                  │  │  (Kafka相容) │
│              │  │ - Primary: 隊列   │  │  Port 9092  │
│ - 主資料庫    │  │ - Cache: 快取    │  │             │
│ - 分區表      │  │ - Store: 狀態    │  │ Kafka Topics│
│ - pg_partman │  └──────────────────┘  └──────┬──────┘
└──────────────┘                               │
                                               ▼
                                    ┌──────────────────────┐
                                    │  Events Processor    │
                                    │     (Go 1.24)        │
                                    │                      │
                                    │ - 事件豐富化          │
                                    │ - 費用計算            │
                                    │ - 訂閱匹配            │
                                    └──────────┬───────────┘
                                               │
                                               ▼
                                    ┌──────────────────────┐
                                    │     ClickHouse       │
                                    │  (分析資料倉儲)        │
                                    │  Port 9000 / 8123    │
                                    └──────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      Sidekiq Workers (背景任務)                      │
│                                                                     │
│  api-worker (預設)     api-billing-worker    api-webhook-worker     │
│  api-events-worker     api-pdfs-worker       api-clock-worker       │
│  api-analytics-worker  api-ai-agent-worker                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                   支援服務 (Supporting Services)                     │
│                                                                     │
│   Clockwork Scheduler    Gotenberg (PDF)    Mailhog (Dev Email)    │
│   PgHero (DB Monitor)    Redpanda Console   Traefik Dashboard      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 技術棧

| 層級 | 技術 | 版本 | 說明 |
|------|------|------|------|
| **前端** | React + TypeScript | - | SPA，nginx 服務 |
| **後端 API** | Ruby on Rails | - | REST + GraphQL |
| **背景任務** | Sidekiq | - | Redis 驅動的任務隊列 |
| **排程器** | Clockwork | - | Ruby 排程框架 |
| **事件處理** | Go 1.24 | 1.24 | 高吞吐量事件管線 |
| **主資料庫** | PostgreSQL | 15 | 含 pg_partman 分區 |
| **快取/隊列** | Redis | - | 3 個獨立實例 |
| **訊息佇列** | Redpanda (Kafka) | - | 事件串流 |
| **分析倉儲** | ClickHouse | - | 時序資料分析 |
| **PDF 生成** | Gotenberg | 7.8.2 | 發票 PDF |
| **反向代理** | Traefik | - | TLS + 路由 |

### 1.3 各服務職責

#### Rails API (`lago-api`)
- **HTTP API**：客戶、訂閱、方案、發票的 CRUD
- **Event Ingestion**：接收使用量事件，推送至 Kafka `events-raw` topic
- **Webhook 管理**：設定 webhook endpoint、簽名、重試
- **付款整合**：Stripe、PayPal 等 inbound webhook 處理
- **Background Jobs**：透過 Sidekiq 執行所有異步任務

#### Events Processor (`Go`)
- **事件豐富化**：從 Kafka 消費 raw events，查詢 billable metric、subscription 資料
- **費用計算**：依 aggregation type（count/sum/max/distinct）計算費用
- **自訂表達式**：支援 lago-expression 語言
- **輸出**：將豐富化事件推送至下游 topics
- **快取**：使用 Redis 維護 charge usage 狀態

#### Frontend (`lago-front`)
- React SPA，透過 REST API 與後端溝通
- 功能：帳單設定、訂閱管理、用量查看、發票下載

---

## 2. 資料庫 Schema

### 2.1 核心資料模型

```
┌─────────────────────────────────────────────────────────────────────┐
│                         核心實體關係圖                               │
└─────────────────────────────────────────────────────────────────────┘

Organization (組織/租戶)
    │
    ├── Customer (客戶)
    │       ├── Subscription (訂閱)
    │       │       ├── Plan (方案)
    │       │       │       ├── Charge (收費項目)
    │       │       │       │       └── BillableMetric (可計費指標)
    │       │       │       └── UsageThreshold (用量門檻)
    │       │       └── SubscriptionActivity
    │       │
    │       ├── Invoice (發票)
    │       │       ├── InvoiceSubscription
    │       │       ├── Fee (費用明細)
    │       │       └── PaymentRequest
    │       │
    │       ├── CreditNote (折讓單)
    │       ├── Coupon → CustomerCoupon (優惠券)
    │       └── Wallet (錢包)
    │               └── WalletTransaction (錢包交易)
    │
    ├── Event (使用量事件) ── 分區表 (by organization_id + timestamp)
    ├── EnrichedEvent    ── 分區表 (monthly range by timestamp)
    │
    ├── WebhookEndpoint (webhook 端點設定)
    │       └── Webhook (webhook 發送紀錄)
    │
    └── InboundWebhook (inbound webhook 接收紀錄)
```

### 2.2 主要資料表

#### organizations
```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY,
    name            VARCHAR NOT NULL,
    api_key         VARCHAR UNIQUE,
    hmac_key        VARCHAR,                    -- webhook HMAC 簽名金鑰
    webhook_url     VARCHAR,                    -- 舊版 webhook URL
    timezone        VARCHAR DEFAULT 'UTC',
    invoice_footer  TEXT,
    logo_url        VARCHAR,
    legal_name      VARCHAR,
    legal_number    VARCHAR,
    tax_identification_number VARCHAR,
    email           VARCHAR,
    country         VARCHAR(2),
    currency        VARCHAR(3) DEFAULT 'USD',
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);
```

#### customers
```sql
CREATE TABLE customers (
    id                       UUID PRIMARY KEY,
    organization_id          UUID NOT NULL REFERENCES organizations(id),
    external_id              VARCHAR NOT NULL,
    name                     VARCHAR,
    email                    VARCHAR,
    phone                    VARCHAR,
    country                  VARCHAR(2),
    currency                 VARCHAR(3),
    timezone                 VARCHAR,
    billing_configuration    JSONB,             -- 付款商設定
    metadata                 JSONB,
    created_at               TIMESTAMP NOT NULL,
    updated_at               TIMESTAMP NOT NULL,
    UNIQUE(organization_id, external_id)
);
```

#### plans
```sql
CREATE TABLE plans (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR NOT NULL,
    code                VARCHAR NOT NULL,
    description         TEXT,
    interval            VARCHAR NOT NULL,       -- weekly/monthly/quarterly/yearly
    amount_cents        BIGINT NOT NULL DEFAULT 0,
    amount_currency     VARCHAR(3) NOT NULL,
    pay_in_advance      BOOLEAN DEFAULT false,
    bill_charges_monthly BOOLEAN DEFAULT false,
    trial_period        DECIMAL,
    created_at          TIMESTAMP NOT NULL,
    updated_at          TIMESTAMP NOT NULL
);
```

#### subscriptions
```sql
CREATE TABLE subscriptions (
    id                          UUID PRIMARY KEY,
    organization_id             UUID NOT NULL REFERENCES organizations(id),
    customer_id                 UUID NOT NULL REFERENCES customers(id),
    plan_id                     UUID NOT NULL REFERENCES plans(id),
    external_id                 VARCHAR,
    status                      VARCHAR NOT NULL,   -- pending/active/terminated/canceled
    started_at                  TIMESTAMP,
    ending_at                   TIMESTAMP,
    canceled_at                 TIMESTAMP,
    terminated_at               TIMESTAMP,
    trial_ended_at              TIMESTAMP,
    billing_time                VARCHAR,            -- calendar/anniversary
    current_billing_period_started_at  TIMESTAMP,
    current_billing_period_ending_at   TIMESTAMP,
    created_at                  TIMESTAMP NOT NULL,
    updated_at                  TIMESTAMP NOT NULL
);
```

#### billable_metrics
```sql
CREATE TABLE billable_metrics (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR NOT NULL,
    code                VARCHAR NOT NULL,
    description         TEXT,
    aggregation_type    INTEGER NOT NULL,       -- count/sum/max/count_unique/latest
    field_name          VARCHAR,               -- 用於 sum/max/latest 的欄位
    weighted_interval   VARCHAR,               -- weighted_sum 時間權重
    expression          TEXT,                  -- 自訂計算表達式
    created_at          TIMESTAMP NOT NULL,
    updated_at          TIMESTAMP NOT NULL,
    UNIQUE(organization_id, code)
);
```

#### charges
```sql
CREATE TABLE charges (
    id                  UUID PRIMARY KEY,
    plan_id             UUID NOT NULL REFERENCES plans(id),
    billable_metric_id  UUID NOT NULL REFERENCES billable_metrics(id),
    charge_model        VARCHAR NOT NULL,       -- standard/graduated/package/percentage/volume/custom
    properties          JSONB,                  -- 收費模型參數
    pay_in_advance      BOOLEAN DEFAULT false,
    prorated            BOOLEAN DEFAULT false,
    min_amount_cents    BIGINT DEFAULT 0,
    created_at          TIMESTAMP NOT NULL,
    updated_at          TIMESTAMP NOT NULL
);
```

#### events (分區表)
```sql
-- 依 organization_id hash 分區，提高查詢效率
CREATE TABLE events (
    id                          UUID NOT NULL,
    organization_id             UUID NOT NULL,
    external_subscription_id    VARCHAR NOT NULL,
    transaction_id              VARCHAR NOT NULL,
    code                        VARCHAR NOT NULL,
    timestamp                   TIMESTAMP NOT NULL,
    properties                  JSONB,
    metadata                    JSONB,
    ingested_at                 TIMESTAMP NOT NULL,
    created_at                  TIMESTAMP NOT NULL,
    PRIMARY KEY (id, organization_id)
) PARTITION BY HASH (organization_id);
```

#### enriched_events (分區表)
```sql
-- 依 timestamp 月份 range 分區，保留 14 個月
CREATE TABLE enriched_events (
    id                          UUID NOT NULL,
    organization_id             UUID NOT NULL,
    external_subscription_id    VARCHAR,
    subscription_id             UUID,
    charge_id                   UUID,
    charge_filter_id            UUID,
    code                        VARCHAR,
    aggregation_type            VARCHAR,
    value                       VARCHAR,
    grouped_by                  JSONB,
    timestamp                   TIMESTAMP NOT NULL,
    ingested_at                 TIMESTAMP,
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);
-- pg_partman 自動管理：每月建立分區，預建 3 個月，保留 14 個月
```

#### invoices
```sql
CREATE TABLE invoices (
    id                          UUID PRIMARY KEY,
    organization_id             UUID NOT NULL REFERENCES organizations(id),
    customer_id                 UUID NOT NULL REFERENCES customers(id),
    status                      INTEGER NOT NULL,   -- draft/finalized/voided/failed
    payment_status              INTEGER NOT NULL,   -- pending/succeeded/failed/overdue
    invoice_type                INTEGER NOT NULL,   -- subscription/add_on/credit/one_off/advance_charges/progressive_billing
    number                      VARCHAR,
    sequential_id               INTEGER,
    issuing_date                DATE,
    payment_due_date            DATE,
    amount_cents                BIGINT DEFAULT 0,
    taxes_amount_cents          BIGINT DEFAULT 0,
    total_amount_cents          BIGINT DEFAULT 0,
    currency                    VARCHAR(3),
    file                        VARCHAR,            -- PDF 檔案路徑
    taxes_rate                  DECIMAL,
    version_number              INTEGER DEFAULT 4,
    metadata                    JSONB,
    created_at                  TIMESTAMP NOT NULL,
    updated_at                  TIMESTAMP NOT NULL
);
```

#### fees
```sql
CREATE TABLE fees (
    id                      UUID PRIMARY KEY,
    invoice_id              UUID REFERENCES invoices(id),
    subscription_id         UUID REFERENCES subscriptions(id),
    charge_id               UUID REFERENCES charges(id),
    fee_type                INTEGER,           -- charge/add_on/subscription/credit/commitment
    amount_cents            BIGINT DEFAULT 0,
    taxes_amount_cents      BIGINT DEFAULT 0,
    total_amount_cents      BIGINT DEFAULT 0,
    currency                VARCHAR(3),
    units                   DECIMAL,
    unit_amount_cents       BIGINT,
    description             TEXT,
    properties              JSONB,
    grouped_by              JSONB,
    pay_in_advance          BOOLEAN DEFAULT false,
    invoiceable             BOOLEAN DEFAULT true,
    created_at              TIMESTAMP NOT NULL,
    updated_at              TIMESTAMP NOT NULL
);
```

#### webhook_endpoints
```sql
CREATE TABLE webhook_endpoints (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    webhook_url         VARCHAR NOT NULL,
    signature_algo      INTEGER DEFAULT 0,     -- 0=jwt, 1=hmac
    hmac_key            VARCHAR,
    created_at          TIMESTAMP NOT NULL,
    updated_at          TIMESTAMP NOT NULL
);
```

#### webhooks
```sql
CREATE TABLE webhooks (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    webhook_endpoint_id UUID REFERENCES webhook_endpoints(id),
    object_id           UUID,
    object_type         VARCHAR,
    webhook_type        VARCHAR,
    endpoint            VARCHAR,
    payload             TEXT,
    http_status         INTEGER,
    response            TEXT,
    status              INTEGER,               -- pending/succeeded/failed/retried
    retries             INTEGER DEFAULT 0,
    last_retried_at     TIMESTAMP,
    created_at          TIMESTAMP NOT NULL,
    updated_at          TIMESTAMP NOT NULL
);
```

#### inbound_webhooks
```sql
CREATE TABLE inbound_webhooks (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    source              VARCHAR NOT NULL,       -- stripe/paypal/etc.
    code                VARCHAR,
    payload             TEXT,
    status              INTEGER,               -- pending/processing/processed/failed
    http_status         INTEGER,
    processing_errors   JSONB,
    created_at          TIMESTAMP NOT NULL,
    updated_at          TIMESTAMP NOT NULL
);
```

#### wallets
```sql
CREATE TABLE wallets (
    id                      UUID PRIMARY KEY,
    organization_id         UUID NOT NULL,
    customer_id             UUID NOT NULL REFERENCES customers(id),
    name                    VARCHAR,
    status                  INTEGER,           -- active/terminated
    currency                VARCHAR(3),
    rate_amount             DECIMAL NOT NULL,  -- 充值匯率
    credits_balance         DECIMAL DEFAULT 0,
    balance_cents           BIGINT DEFAULT 0,
    consumed_credits        DECIMAL DEFAULT 0,
    depleted_ongoing_balance BOOLEAN DEFAULT false,
    expiration_at           TIMESTAMP,
    last_balance_sync_at    TIMESTAMP,
    last_consumed_credit_at TIMESTAMP,
    created_at              TIMESTAMP NOT NULL,
    updated_at              TIMESTAMP NOT NULL
);
```

### 2.3 資料庫分區策略

```
┌────────────────────────────────────────────────────────────┐
│                  PostgreSQL 分區策略                        │
├────────────────────────┬───────────────────────────────────┤
│ events                 │ HASH by organization_id           │
│                        │ 提升多租戶並發查詢效率             │
├────────────────────────┬───────────────────────────────────┤
│ enriched_events        │ RANGE by timestamp (每月)         │
│                        │ 由 pg_partman 自動管理             │
│                        │ 預建: 3 個月                      │
│                        │ 保留: 14 個月                     │
│                        │ 舊資料自動 DROP 或 DETACH          │
└────────────────────────┴───────────────────────────────────┘
```

---

## 3. 排程任務 (Cron Jobs)

### 3.1 排程架構

```
Clockwork Scheduler (api-clock container)
    │
    ├── 每分鐘     ──► Sidekiq Queue ──► Sidekiq Worker ──► 執行任務
    ├── 每5分鐘    ──► Sidekiq Queue ──► Sidekiq Worker ──► 執行任務
    ├── 每15分鐘   ──► Sidekiq Queue ──► Sidekiq Worker ──► 執行任務
    ├── 每小時     ──► Sidekiq Queue ──► Sidekiq Worker ──► 執行任務
    └── 每日       ──► Sidekiq Queue ──► Sidekiq Worker ──► 執行任務
```

### 3.2 任務清單

#### 高頻任務（每 1-5 分鐘）

| 頻率 | 任務名稱 | 說明 |
|------|----------|------|
| 每 1 分鐘 | `ProcessSubscriptionActivity` | 處理訂閱活動狀態變更 |
| 每 1 分鐘 | `RefreshFlaggedSubscriptions` | 更新標記訂閱的用量快取 |
| 每 5 分鐘 | `ActivateSubscriptions` | 啟動到期的待啟用訂閱 |
| 每 5 分鐘 | `RefreshDraftInvoices` | 更新草稿發票金額 |
| 每 5 分鐘 | `RefreshLifetimeUsages` | 更新終身用量統計（可停用） |
| 每 5 分鐘 | `RefreshWalletsOngoingBalance` | 更新錢包即時餘額 |

#### 每小時任務（在指定分鐘執行）

| 執行時間 | 任務名稱 | 說明 |
|----------|----------|------|
| :05 | `TerminateEndedSubscriptions` | 終止已到期的訂閱 |
| :05 | `PostValidateEvents` | 事後驗證事件資料 |
| :10 | `BillCustomers` | 觸發客戶計費流程 |
| :15 | `ApiKeysTrackUsage` | 統計 API Key 使用量 |
| :15 | `ComputeDailyUsage` | 計算每日用量報告 |
| :20 | `FinalizeInvoices` | 確認草稿發票為正式發票 |
| :25 | `MarkInvoicesAsPaymentOverdue` | 標記逾期未付發票 |
| :30 | `TerminateCoupons` | 終止過期優惠券 |
| :30 | `RetryGeneratingSubscriptionInvoices` | 重試失敗的訂閱發票生成 |
| :35 | `BillEndedTrialSubscriptions` | 計費試用期結束的訂閱 |
| :45 | `TerminateWallets` | 終止過期錢包 |
| :45 | `ProcessDunningCampaigns` | 處理欠款追收流程 |
| :50 | `TerminationAlert` | 發送即將終止通知 |
| :50 | `TerminateExpiredWalletTransactionRules` | 終止過期錢包規則 |
| :55 | `TopUpWalletIntervalCredits` | 定期儲值錢包點數 |

#### 每 15 分鐘任務

| 任務名稱 | 說明 |
|----------|------|
| `RetryFailedInvoices` | 重試發票 PDF 生成失敗 |
| `RetryInboundWebhooks` | 重試處理失敗的 inbound webhooks |

#### 每日任務

| 執行時間 | 任務名稱 | 說明 |
|----------|----------|------|
| 01:00 | `CleanWebhooks` | 清除舊的 outbound webhook 記錄 |
| 01:10 | `CleanInboundWebhooks` | 清除舊的 inbound webhook 記錄 |

### 3.3 Sidekiq 隊列優先順序

```
Priority  Queue Name          用途
  1       high_priority       緊急任務（付款失敗通知等）
  2       default             一般任務
  3       mailers             Email 發送
  4       clock               Clockwork 觸發的任務
  5       providers           第三方整合
  6       webhook             Webhook 發送
  7       invoices            發票生成
  8       integrations        系統整合
  9       low_priority        低優先度背景任務
  10      long_running        長時間執行任務
```

---

## 4. Webhook 設計

### 4.1 整體流程

```
┌──────────────────────────────────────────────────────────────────┐
│                    Webhook 雙向設計                               │
└──────────────────────────────────────────────────────────────────┘

Outbound (Lago → 客戶系統)
━━━━━━━━━━━━━━━━━━━━━━━━━

  Lago 系統事件發生
      │
      ▼
  WebhookJob (Sidekiq)
      │
      ├── 讀取 WebhookEndpoint 設定
      ├── 建立 Webhook 記錄
      ├── 簽名 Payload
      └── HTTP POST → 客戶 endpoint
              │
              ├── 成功 (2xx) → 標記 succeeded
              └── 失敗 → 標記 failed，等待排程重試


Inbound (付款商 → Lago)
━━━━━━━━━━━━━━━━━━━━━━━

  Stripe/PayPal 付款事件
      │
      ▼
  Lago API Endpoint
      │
      ├── 驗證來源簽名
      ├── 儲存 InboundWebhook 記錄
      └── 非同步處理 (Sidekiq)
              │
              └── 更新付款狀態、觸發業務邏輯
```

### 4.2 Outbound Webhook

#### 簽名機制

**HMAC 模式（對稱加密）**
```
簽名演算法：HMAC-SHA256
金鑰來源：  organization.hmac_key（儲存於 DB，可自訂）
計算方式：  Base64.strict_encode64(
              OpenSSL::HMAC.digest("sha-256", hmac_key, payload)
            )
Header：    X-Lago-Signature: <base64_signature>
驗證方：    客戶用相同 hmac_key 驗證
```

**JWT 模式（非對稱加密）**
```
簽名演算法：RS256 (RSA + SHA-256)
金鑰來源：  LAGO_RSA_PRIVATE_KEY 環境變數
Payload：   { data: <webhook_payload>, iss: <LAGO_API_URL> }
Header：    X-Lago-Signature: <jwt_token>
驗證方：    客戶用 Lago 公鑰驗證（可從 API 取得公鑰）
```

#### Webhook 事件類型

| 類別 | 事件 |
|------|------|
| **發票** | `invoice.created`, `invoice.add_on_added`, `invoice.paid`, `invoice.payment_failure`, `invoice.drafted`, `invoice.voided`, `invoice.payment_overdue`, `invoice.payment_dispute_lost` |
| **訂閱** | `subscription.started`, `subscription.terminated`, `subscription.termination_alert`, `subscription.trial_ended`, `subscription.usage_threshold_reached` |
| **客戶** | `customer.created`, `customer.updated`, `customer.payment_provider_created`, `customer.payment_provider_error` |
| **錢包** | `wallet_transaction.created`, `wallet_transaction.updated`, `wallet_transaction.payment_failure` |
| **優惠券** | `coupon.terminated` |
| **信用票據** | `credit_note.created`, `credit_note.generated` |
| **付款請求** | `payment_request.created`, `payment_request.payment_failure` |

#### Webhook 重試策略

```
首次發送
    │
    ├── 成功 (2xx) ─────────────────────────────► 完成
    │
    └── 失敗 (非 2xx / timeout)
            │
            ▼
        標記 status = failed
            │
            ▼
        Clockwork 每 15 分鐘掃描失敗記錄
            │
            ▼
        重新加入 webhook 隊列
            │
            └── 重試次數由 LAGO_WEBHOOK_ATTEMPTS 控制
```

### 4.3 Inbound Webhook

#### 支援的付款商

| 付款商 | 驗證方式 | 說明 |
|--------|----------|------|
| Stripe | Stripe-Signature header | RSA 簽名驗證 |
| PayPal | PayPal-Auth-Algo | 非對稱驗證 |
| Adyen | HMAC | 共享金鑰驗證 |
| GoCardless | Webhook-Signature | HMAC-SHA256 |
| Cashfree | x-cashfree-signature | HMAC 驗證 |
| Moneyhash | 自訂 | API 金鑰驗證 |

#### Inbound 處理流程

```
付款商 POST /webhooks/{source}/{org_code}
    │
    ├── 1. 驗證 webhook 簽名
    ├── 2. 儲存至 inbound_webhooks 表（status: pending）
    ├── 3. 回應 200 OK（快速回應避免 timeout）
    │
    └── 非同步處理（Sidekiq）
            ├── status 更新為 processing
            ├── 解析事件類型
            ├── 執行業務邏輯（更新付款狀態等）
            └── status 更新為 processed / failed

失敗處理：
    ├── Clockwork 每 15 分鐘重試 status=failed 記錄
    └── Clockwork 每日 01:10 清除舊記錄
```

---

## 5. 事件處理管線

### 5.1 Kafka Topic 設計

```
客戶 API 呼叫
    │
    ▼
events-raw (輸入)
    │
    ▼
Events Processor (Go)
    │
    ├──► events_enriched             # 豐富化事件（含訂閱/指標資訊）
    ├──► events_enriched_expanded    # 展開事件（含 charge/filter 維度）
    ├──► events_charged_in_advance   # 預扣費事件（錢包扣點）
    ├──► activity_logs               # 系統活動稽核
    ├──► api_logs                    # API 請求日誌
    └──► events_dead_letter          # 處理失敗事件
              │
              ▼
    ClickHouse (分析倉儲)
```

### 5.2 Event 資料結構

**Raw Event**
```json
{
  "organization_id": "uuid",
  "external_subscription_id": "sub_xxx",
  "transaction_id": "txn_unique_id",
  "code": "api_calls",
  "properties": {
    "region": "us-east-1",
    "response_time_ms": 250
  },
  "precise_total_amount_cents": "1000",
  "timestamp": 1700000000,
  "ingested_at": "2024-01-01T00:00:00Z"
}
```

**Enriched Event**
```json
{
  "organization_id": "uuid",
  "subscription_id": "uuid",
  "external_subscription_id": "sub_xxx",
  "plan_id": "uuid",
  "charge_id": "uuid",
  "charge_filter_id": "uuid",
  "code": "api_calls",
  "aggregation_type": "sum",
  "value": "250",
  "grouped_by": {
    "region": "us-east-1"
  },
  "timestamp": 1700000000.0,
  "target_wallet_code": null
}
```

**Dead Letter Event**
```json
{
  "original_event": { ... },
  "error_code": "SUBSCRIPTION_NOT_FOUND",
  "error_message": "No active subscription found",
  "failed_at": "2024-01-01T00:00:00Z"
}
```

### 5.3 聚合類型

| 類型 | 說明 | 使用場景 |
|------|------|----------|
| `count` | 計算事件次數 | API 請求次數 |
| `sum` | 加總 field 值 | 資料傳輸量 |
| `max` | 取最大值 | 尖峰用量 |
| `count_unique` | 計算唯一值數量 | 活躍用戶數 |
| `latest` | 取最新值 | 當前座位數 |
| `weighted_sum` | 時間加權加總 | 平均使用量 |
| `custom` | 自訂表達式 | 複雜計費邏輯 |

---

## 6. 環境設定

### 6.1 Service Ports

| 服務 | Port | 說明 |
|------|------|------|
| Rails API | 3000 | HTTP API |
| Frontend | 80 | nginx |
| PostgreSQL | 5432 | 主資料庫 |
| Redis (Primary) | 6379 | Sidekiq 隊列 |
| Redpanda | 9092 | Kafka broker |
| ClickHouse HTTP | 8123 | HTTP 介面 |
| ClickHouse Native | 9000 | 原生協議 |
| Traefik Dashboard | 8080 | 代理管理 |
| Gotenberg | 3000 | PDF 生成（內部） |

### 6.2 關鍵環境變數

```bash
# === 基本設定 ===
LAGO_API_URL=https://api.lago.example.com
LAGO_FRONT_URL=https://app.lago.example.com
SECRET_KEY_BASE=<rails_secret>

# === 資料庫 ===
DATABASE_URL=postgresql://user:pass@postgres:5432/lago
POSTGRES_USER=lago
POSTGRES_PASSWORD=<password>
POSTGRES_DB=lago

# === Redis (3 個實例) ===
REDIS_URL=redis://redis:6379           # Sidekiq 隊列
LAGO_REDIS_CACHE_URL=redis://redis-cache:6379   # 快取
LAGO_REDIS_STORE_URL=redis://redis-store:6379   # 狀態儲存

# === Kafka/Redpanda ===
LAGO_KAFKA_BOOTSTRAP_SERVERS=redpanda:9092
LAGO_KAFKA_RAW_EVENTS_TOPIC=events-raw
LAGO_KAFKA_ENRICHED_EVENTS_TOPIC=events_enriched
LAGO_KAFKA_ENRICHED_EVENTS_EXPANDED_TOPIC=events_enriched_expanded
LAGO_KAFKA_EVENTS_CHARGED_IN_ADVANCE_TOPIC=events_charged_in_advance
LAGO_KAFKA_EVENTS_DEAD_LETTER_TOPIC=events_dead_letter
LAGO_KAFKA_CONSUMER_GROUP=lago-events-processor

# === 加密 ===
LAGO_ENCRYPTION_PRIMARY_KEY=<key>
LAGO_ENCRYPTION_DETERMINISTIC_KEY=<key>
LAGO_RSA_PRIVATE_KEY=<rsa_private_key>   # JWT webhook 簽名

# === 功能開關 ===
LAGO_SIDEKIQ_WEB=true                   # 啟用 Sidekiq UI
LAGO_CLICKHOUSE_ENABLED=true            # 啟用分析
LAGO_DISABLE_PDF_GENERATION=false       # 停用 PDF
LAGO_DISABLE_WALLET_REFRESH=false       # 停用錢包刷新
LAGO_WEBHOOK_ATTEMPTS=3                 # Webhook 重試次數

# === 專用 Worker 開關 ===
SIDEKIQ_EVENTS=true      # 事件處理 worker
SIDEKIQ_PDFS=true        # PDF 生成 worker
SIDEKIQ_BILLING=true     # 計費 worker
SIDEKIQ_WEBHOOK=true     # Webhook worker
SIDEKIQ_CLOCK=true       # 排程 worker
SIDEKIQ_ANALYTICS=true   # 分析 worker
SIDEKIQ_AI_AGENT=true    # AI 功能 worker
```

---

## 7. 擴展建議

### 7.1 生產環境建議資源

| 元件 | CPU | 記憶體 | 副本數 |
|------|-----|--------|--------|
| API | 4 cores | 4 Gi | 10–30 |
| Default Worker | 1.1 cores | 2 Gi | 3–5 |
| Events Worker | 500m | 1 Gi | 2–5 |
| PDF Worker | 1.1 cores | 1 Gi | 1 |
| Webhook Worker | 1.1 cores | 1 Gi | 3–10 |
| Clock Process | 100m | 812 Mi | **1（不可多開）** |
| Events Processor (Go) | 2 cores | 2 Gi | 1 |
| ClickHouse | 2 cores | 1 Gi | 2–4 |

> **注意**：`api-clock` 必須只有 1 個副本，否則排程任務會重複執行。

### 7.2 自行擴充建議

若要在此 repo 基礎上擴充功能，建議方向：

1. **新增計費模型**：在 `charges` 的 `charge_model` 新增類型，並在 Events Processor 加入對應聚合邏輯
2. **新增付款商整合**：在 `inbound_webhooks` 加入新的 `source`，實作對應的驗證與處理邏輯
3. **自訂 Webhook 事件**：在 Rails API 新增 webhook 觸發點，定義新的 `webhook_type`
4. **擴充分析能力**：透過 ClickHouse 新增自訂查詢，或整合 Metabase/Grafana
5. **新增排程任務**：在 Clockwork 設定檔新增任務，並實作對應的 Sidekiq Job

---

*本文件由 Claude Code 根據原始碼分析自動生成*
