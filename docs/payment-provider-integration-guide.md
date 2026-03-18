# 金流整合開發指引

> 適用版本：Lago v1.43.0+
> 技術棧：Ruby on Rails + Sidekiq + PostgreSQL

---

## 目錄

1. [整合架構概覽](#1-整合架構概覽)
2. [前置準備](#2-前置準備)
3. [資料庫層](#3-資料庫層)
4. [Model 層](#4-model-層)
5. [Service 層 — Outbound（Lago 發起付款）](#5-service-層--outbound)
6. [Controller 層 — Inbound（接收付款通知）](#6-controller-層--inbound)
7. [路由設定](#7-路由設定)
8. [Background Job](#8-background-job)
9. [Clockwork 排程整合](#9-clockwork-排程整合)
10. [Outbound Webhook 通知](#10-outbound-webhook-通知)
11. [環境變數設定](#11-環境變數設定)
12. [錯誤處理與重試機制](#12-錯誤處理與重試機制)
13. [測試指引](#13-測試指引)
14. [部署 Checklist](#14-部署-checklist)
15. [常見問題](#15-常見問題)

---

## 1. 整合架構概覽

### 1.1 雙向金流資料流

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Outbound（Lago → 金流商）                       │
│                                                                      │
│  Invoice 確認                                                         │
│      │                                                               │
│      ▼                                                               │
│  Payments::YourProviderService.create                                │
│      │                                                               │
│      ├── 呼叫金流商 API（建立收款）                                    │
│      ├── 儲存 provider_invoice_id、provider_payment_id               │
│      └── 更新 Invoice#payment_status = pending                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       Inbound（金流商 → Lago）                        │
│                                                                      │
│  金流商 POST /webhooks/your_provider/:org_code                        │
│      │                                                               │
│      ├── 1. 驗證簽名                                                  │
│      ├── 2. 儲存 InboundWebhook（status: pending）                    │
│      └── 3. 立即回 200                                                │
│                                                                      │
│  Sidekiq Job（非同步）                                                │
│      │                                                               │
│      ├── 4. 解析事件類型                                              │
│      ├── 5. 執行業務邏輯（更新發票狀態等）                              │
│      └── 6. 觸發 Lago Outbound Webhook 通知客戶系統                   │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 需要新增的檔案清單

```
api/
├── app/
│   ├── models/payment_providers/
│   │   └── your_provider.rb                          # STI Model
│   │
│   ├── services/payment_providers/
│   │   └── your_provider_service.rb                  # 付款商 CRUD 服務
│   │
│   ├── services/payments/
│   │   └── your_provider_service.rb                  # 發起付款邏輯
│   │
│   ├── controllers/webhooks/
│   │   └── your_provider_controller.rb               # Inbound webhook 接收
│   │
│   └── jobs/payment_providers/your_provider/
│       ├── handle_event_job.rb                       # 事件處理 Job
│       └── handle_payment_intent_event_job.rb        # 付款意圖事件（視需求）
│
├── db/migrate/
│   ├── YYYYMMDDHHMMSS_create_your_provider_customers.rb
│   └── YYYYMMDDHHMMSS_add_your_provider_to_customer_providers.rb
│
├── config/routes.rb                                  # 新增路由（修改）
└── spec/
    ├── models/payment_providers/your_provider_spec.rb
    ├── services/payment_providers/your_provider_service_spec.rb
    ├── services/payments/your_provider_service_spec.rb
    └── controllers/webhooks/your_provider_controller_spec.rb
```

---

## 2. 前置準備

### 2.1 了解金流商的 API 規格

開始實作前，確認以下資訊：

| 項目 | 說明 | 範例 |
|------|------|------|
| **API 認證方式** | Bearer Token / API Key / OAuth | `Authorization: Bearer sk_xxx` |
| **建立付款 API** | 如何透過 API 建立一筆收款 | `POST /v1/payments` |
| **Webhook 簽名方式** | 如何驗證 webhook 來源 | HMAC-SHA256 / RSA |
| **簽名 Header 名稱** | Webhook 請求中哪個 Header 含有簽名 | `X-Signature` |
| **事件類型命名** | 金流商的事件名稱規格 | `payment.succeeded` |
| **客戶識別欄位** | 金流商如何識別客戶 | `customer_id: "cust_xxx"` |
| **SDK 是否可用** | 是否有官方 Ruby gem | `gem 'your_provider_ruby'` |

### 2.2 本地開發環境準備

```bash
# 確認開發環境正常
cd lago
docker-compose -f docker-compose.dev.yml up -d

# 進入 Rails console 測試
docker exec -it lago-api rails console

# 確認現有付款商 model 可正常讀取（參考用）
PaymentProviders::StripeProvider.first
```

### 2.3 安裝金流商 SDK（如有需要）

```ruby
# api/Gemfile
gem 'your_provider_ruby', '~> 1.0'  # 官方 Ruby SDK

# 執行
bundle install
```

---

## 3. 資料庫層

### 3.1 核心資料表說明

Lago 使用 **STI（Single Table Inheritance）** 方式管理所有付款商設定，統一儲存在 `payment_providers` 表。

```
payment_providers 表（所有付款商共用）
    │
    ├── type: "PaymentProviders::StripeProvider"
    ├── type: "PaymentProviders::AdyenProvider"
    └── type: "PaymentProviders::YourProvider"    ← 你要新增的
```

客戶與付款商的關聯透過 `customer_payment_providers` 或 `customers.billing_configuration` 管理。

### 3.2 建立 Migration

#### Migration 1：新增付款商欄位（如有特殊欄位需求）

```ruby
# db/migrate/YYYYMMDDHHMMSS_add_your_provider_fields_to_payment_providers.rb
class AddYourProviderFieldsToPaymentProviders < ActiveRecord::Migration[7.0]
  def change
    # payment_providers 表已存在，只需新增特定欄位
    # 大多數設定存在 settings JSONB 欄位中，無需額外 migration

    # 若需要獨立欄位（通常不需要）：
    # add_column :payment_providers, :your_provider_specific_field, :string
  end
end
```

#### Migration 2：客戶付款商關聯表

```ruby
# db/migrate/YYYYMMDDHHMMSS_create_your_provider_customers.rb
# 注意：通常 customer_payment_providers 表已存在，
# 確認是否已有泛用設計，若已有則跳過此步驟

class CreateYourProviderCustomers < ActiveRecord::Migration[7.0]
  def change
    # 若 customer_payment_providers 已存在且為通用設計，則無需新建
    # 僅需確認 payment_provider_type 欄位可儲存新的 type 字串
  end
end
```

### 3.3 資料存儲設計

付款商的 API Key 和設定透過 **Active Record Encryption** 加密儲存：

```
payment_providers 表關鍵欄位：
┌─────────────────────┬────────────────────────────────────────┐
│ 欄位                 │ 說明                                   │
├─────────────────────┼────────────────────────────────────────┤
│ id                  │ UUID，主鍵                              │
│ organization_id     │ 關聯組織                                │
│ type                │ STI 類型識別，如 "PaymentProviders::..." │
│ code                │ 付款商代碼，如 "stripe"                 │
│ name                │ 顯示名稱                                │
│ secrets             │ 加密儲存（API Key / Secret 等）          │
│ settings            │ JSONB，非敏感設定                       │
│ created_at          │ 建立時間                                │
│ updated_at          │ 更新時間                                │
└─────────────────────┴────────────────────────────────────────┘
```

---

## 4. Model 層

### 4.1 PaymentProvider Model（STI）

```ruby
# app/models/payment_providers/your_provider.rb
module PaymentProviders
  class YourProvider < BaseProvider
    # ─── 付款商識別碼 ───────────────────────────────────────────────
    PROVIDER_CODE = 'your_provider'   # 全小寫，底線連接
    PROVIDER_NAME = 'Your Provider'

    # ─── 加密欄位（API Keys）─────────────────────────────────────────
    # 使用 Rails Active Record Encryption（LAGO_ENCRYPTION_PRIMARY_KEY）
    encrypts :api_key
    encrypts :api_secret

    # ─── 驗證 ────────────────────────────────────────────────────────
    validates :api_key, presence: true

    # ─── 讀取器（從 settings JSONB 或 secrets 欄位讀取）────────────────
    def api_key
      secrets&.dig('api_key')
    end

    def api_key=(value)
      self.secrets = (secrets || {}).merge('api_key' => value)
    end

    def api_secret
      secrets&.dig('api_secret')
    end

    def api_secret=(value)
      self.secrets = (secrets || {}).merge('api_secret' => value)
    end

    # ─── Webhook 簽名設定 ─────────────────────────────────────────────
    def webhook_secret
      settings&.dig('webhook_secret')
    end

    def webhook_secret=(value)
      self.settings = (settings || {}).merge('webhook_secret' => value)
    end
  end
end
```

### 4.2 CustomerPaymentProvider（客戶與付款商關聯）

```ruby
# app/models/payment_providers/your_provider_customer.rb
module PaymentProviders
  class YourProviderCustomer < BaseCustomer
    # 儲存客戶在金流商端的識別資訊
    # 例如：Stripe 的 customer_id = "cus_xxx"

    # 從 settings JSONB 讀取
    def provider_customer_id
      settings&.dig('provider_customer_id')
    end

    def provider_customer_id=(value)
      self.settings = (settings || {}).merge('provider_customer_id' => value)
    end

    # 付款方式 ID（如：預設信用卡 token）
    def provider_payment_methods
      settings&.dig('provider_payment_methods') || []
    end
  end
end
```

---

## 5. Service 層 — Outbound

當 Lago 要向金流商發起付款時（通常由 Invoice 確認後觸發）。

### 5.1 PaymentProvider Service（CRUD）

```ruby
# app/services/payment_providers/your_provider_service.rb
module PaymentProviders
  class YourProviderService < BaseService
    # ─── 建立或更新付款商設定 ────────────────────────────────────────
    def create_or_update(**args)
      payment_provider = PaymentProviders::YourProvider
        .find_or_initialize_by(organization: args[:organization])

      payment_provider.api_key       = args[:api_key] if args[:api_key]
      payment_provider.api_secret    = args[:api_secret] if args[:api_secret]
      payment_provider.webhook_secret = args[:webhook_secret] if args[:webhook_secret]
      payment_provider.code          = PaymentProviders::YourProvider::PROVIDER_CODE

      payment_provider.save!

      result.payment_provider = payment_provider
      result.success!
    rescue ActiveRecord::RecordInvalid => e
      result.record_validation_failure!(record: e.record)
    end

    # ─── 處理 Inbound Webhook ────────────────────────────────────────
    def handle_incoming_webhook(organization_id:, body:, signature:)
      organization = Organization.find(organization_id)
      payment_provider = organization.your_provider

      # 驗證 webhook 簽名
      unless valid_signature?(body, signature, payment_provider.webhook_secret)
        return result.not_allowed_failure!(code: 'webhook_signature_invalid')
      end

      # 儲存並非同步處理
      inbound_webhook = InboundWebhook.create!(
        organization:    organization,
        source:          PaymentProviders::YourProvider::PROVIDER_CODE,
        payload:         body,
        status:          :pending
      )

      PaymentProviders::YourProvider::HandleEventJob.perform_later(
        inbound_webhook: inbound_webhook
      )

      result.success!
    rescue ActiveRecord::RecordInvalid => e
      result.record_validation_failure!(record: e.record)
    end

    private

    # ─── 簽名驗證（依金流商規格實作）────────────────────────────────────
    def valid_signature?(payload, signature, secret)
      # 範例：HMAC-SHA256
      expected = OpenSSL::HMAC.hexdigest('sha256', secret, payload)
      ActiveSupport::SecurityUtils.secure_compare(expected, signature)

      # 若金流商使用 RSA 簽名，改用對應驗證邏輯
    end
  end
end
```

### 5.2 Payment Creation Service（發起收款）

```ruby
# app/services/payments/your_provider_service.rb
module Payments
  class YourProviderService < BaseService
    PENDING_STATUSES  = %w[pending processing].freeze
    SUCCESS_STATUSES  = %w[succeeded paid].freeze
    FAILED_STATUSES   = %w[failed declined].freeze

    def initialize(invoice)
      @invoice  = invoice
      @customer = invoice.customer
      super
    end

    # ─── 主要入口：向金流商建立一筆收款 ──────────────────────────────
    def create
      return result.success! if @invoice.payment_succeeded?

      provider_customer = @customer.payment_provider_customers
        .find_by(payment_provider_type: 'PaymentProviders::YourProvider')

      return result.not_found_failure!(resource: 'provider_customer') unless provider_customer

      # 呼叫金流商 API
      response = your_provider_client.create_payment(
        amount:      @invoice.total_amount_cents,
        currency:    @invoice.currency.downcase,
        customer_id: provider_customer.provider_customer_id,
        metadata: {
          lago_invoice_id:     @invoice.id,
          lago_customer_id:    @customer.id,
          lago_invoice_number: @invoice.number
        }
      )

      # 更新發票的付款狀態
      update_invoice_payment_status(response)

      result.success!
    rescue YourProviderSDK::AuthenticationError => e
      deliver_error_webhook(e)
      result.third_party_failure!(third_party: 'YourProvider', error: e)
    rescue YourProviderSDK::ApiError => e
      deliver_error_webhook(e)
      result.third_party_failure!(third_party: 'YourProvider', error: e)
    end

    private

    def your_provider_client
      @your_provider_client ||= begin
        provider = @customer.organization.your_provider
        YourProviderSDK::Client.new(api_key: provider.api_key)
      end
    end

    def update_invoice_payment_status(response)
      status = case response.status
               when *SUCCESS_STATUSES then :succeeded
               when *FAILED_STATUSES  then :failed
               else :pending
               end

      @invoice.update!(
        payment_status:              status,
        payment_provider_invoice_id: response.id,      # 金流商的交易 ID
        ready_for_payment_processing: false
      )

      # 觸發 Lago Outbound Webhook（通知使用 Lago 的客戶系統）
      if status == :succeeded
        SendWebhookJob.perform_later('invoice.paid', @invoice)
      elsif status == :failed
        SendWebhookJob.perform_later('invoice.payment_failure', @invoice)
      end
    end

    def deliver_error_webhook(error)
      SendWebhookJob.perform_later(
        'invoice.payment_failure',
        @invoice,
        provider_error: {
          message:       error.message,
          error_code:    error.respond_to?(:code) ? error.code : nil
        }
      )
    end
  end
end
```

---

## 6. Controller 層 — Inbound

接收金流商發送過來的 webhook 通知。

```ruby
# app/controllers/webhooks/your_provider_controller.rb
module Webhooks
  class YourProviderController < ApplicationController
    # 跳過 CSRF 保護（webhook 是外部請求）
    skip_before_action :verify_authenticity_token

    # 在 action 執行前找到對應的組織
    before_action :find_organization

    # ─── 主要 Webhook 接收端點 ────────────────────────────────────────
    def handle
      # 讀取原始 request body（注意：讀取後不可再次讀取）
      payload   = request.body.read
      signature = request.headers['X-Your-Signature-Header']

      result = PaymentProviders::YourProviderService.new
        .handle_incoming_webhook(
          organization_id: @organization.id,
          body:            payload,
          signature:       signature
        )

      if result.success?
        head :ok
      elsif result.error_code == 'webhook_signature_invalid'
        head :unauthorized
      else
        head :bad_request
      end
    end

    private

    def find_organization
      # URL 格式：/webhooks/your_provider/:organization_code
      @organization = Organization.find_by(api_key: params[:organization_code])

      head :not_found unless @organization
    end
  end
end
```

---

## 7. 路由設定

```ruby
# config/routes.rb

# 在既有的 webhooks namespace 中新增
namespace :webhooks do
  # 現有路由
  post 'stripe/:organization_code',       to: 'stripe#handle'
  post 'adyen/:organization_code',        to: 'adyen#handle'
  post 'gocardless/:organization_code',   to: 'gocardless#handle'
  post 'cashfree/:organization_code',     to: 'cashfree#handle'
  post 'moneyhash/:organization_code',    to: 'moneyhash#handle'

  # 新增你的金流商
  post 'your_provider/:organization_code', to: 'your_provider#handle'
end
```

完整的 Webhook URL 格式：
```
POST https://your-lago-api.com/webhooks/your_provider/{ORGANIZATION_API_KEY}
```

---

## 8. Background Job

處理 inbound webhook 的非同步 Job。

```ruby
# app/jobs/payment_providers/your_provider/handle_event_job.rb
module PaymentProviders
  module YourProvider
    class HandleEventJob < ApplicationJob
      queue_as :providers   # 使用 providers 隊列

      # 重試設定（Lago 預設 max_retries: 0，依賴 Clockwork 排程重試）
      retry_on ActiveRecord::RecordNotFound, wait: 5.seconds, attempts: 3

      def perform(inbound_webhook:)
        # 標記為處理中
        inbound_webhook.update!(status: :processing)

        payload    = JSON.parse(inbound_webhook.payload)
        event_type = extract_event_type(payload)

        case event_type
        when 'payment.succeeded', 'payment.completed'
          handle_payment_success(payload, inbound_webhook.organization)

        when 'payment.failed', 'payment.declined'
          handle_payment_failure(payload, inbound_webhook.organization)

        when 'customer.created'
          handle_customer_created(payload, inbound_webhook.organization)

        when 'refund.completed'
          handle_refund(payload, inbound_webhook.organization)

        else
          # 未知事件，記錄但不報錯
          Rails.logger.info(
            "YourProvider: Unhandled event type #{event_type}"
          )
        end

        # 標記為已處理
        inbound_webhook.update!(status: :processed)
      rescue StandardError => e
        # 標記為失敗，等待 Clockwork 重試
        inbound_webhook.update!(
          status:            :failed,
          processing_errors: { error: e.message, backtrace: e.backtrace.first(5) }
        )
        raise  # 讓 Sidekiq 記錄到 dead queue
      end

      private

      # ─── 付款成功 ────────────────────────────────────────────────────
      def handle_payment_success(payload, organization)
        provider_invoice_id = payload.dig('data', 'invoice_id')   # 依金流商格式調整

        invoice = Invoice
          .joins(:customer)
          .where(customers: { organization_id: organization.id })
          .find_by(payment_provider_invoice_id: provider_invoice_id)

        return unless invoice

        invoice.update!(payment_status: :succeeded)

        # 觸發 Lago Outbound Webhook → 通知客戶系統
        SendWebhookJob.perform_later('invoice.paid', invoice)
      end

      # ─── 付款失敗 ────────────────────────────────────────────────────
      def handle_payment_failure(payload, organization)
        provider_invoice_id = payload.dig('data', 'invoice_id')

        invoice = Invoice
          .joins(:customer)
          .where(customers: { organization_id: organization.id })
          .find_by(payment_provider_invoice_id: provider_invoice_id)

        return unless invoice

        invoice.update!(payment_status: :failed)

        SendWebhookJob.perform_later('invoice.payment_failure', invoice)
      end

      # ─── 客戶在金流商端建立成功 ─────────────────────────────────────
      def handle_customer_created(payload, organization)
        external_customer_id     = payload.dig('data', 'metadata', 'lago_customer_id')
        provider_customer_id     = payload.dig('data', 'id')

        customer = organization.customers.find_by(id: external_customer_id)
        return unless customer

        provider_customer = customer.payment_provider_customers
          .find_or_initialize_by(
            payment_provider_type: 'PaymentProviders::YourProvider'
          )

        provider_customer.update!(
          settings: provider_customer.settings.merge(
            'provider_customer_id' => provider_customer_id
          )
        )

        SendWebhookJob.perform_later('customer.payment_provider_created', customer)
      end

      # ─── 退款 ────────────────────────────────────────────────────────
      def handle_refund(payload, organization)
        # 實作退款邏輯（更新 CreditNote 狀態等）
      end

      def extract_event_type(payload)
        # 依金流商 webhook 格式調整路徑
        payload['event_type'] || payload.dig('data', 'type')
      end
    end
  end
end
```

---

## 9. Clockwork 排程整合

Lago 的重試機制透過 Clockwork 觸發，不需要額外設定。現有的通用重試任務已涵蓋你的整合：

```ruby
# 現有 Clockwork 排程（已包含你的金流商，無需修改）

# 每 15 分鐘：重試失敗的 inbound webhooks
every(15.minutes, 'retry_inbound_webhooks') do
  RetryInboundWebhooksJob.perform_later
  # 這會掃描所有 status=failed 的 inbound_webhooks 並重試
  # 包含你的 your_provider 來源
end

# 每日 01:10：清理舊的 inbound webhook 記錄
every(1.day, 'clean_inbound_webhooks', at: '01:10') do
  CleanInboundWebhooksJob.perform_later
end
```

### 補充：若需要主動輪詢（Pull-based）

若金流商不支援 Webhook（只能輪詢），需新增 Clockwork 任務：

```ruby
# 在 Clockwork 設定中新增
every(5.minutes, 'sync_your_provider_payments') do
  PaymentProviders::YourProvider::SyncPaymentsJob.perform_later
end
```

---

## 10. Outbound Webhook 通知

當付款狀態變更時，Lago 自動向你的客戶系統發送 Webhook。以下是與金流商整合相關的事件：

| 事件 | 觸發時機 | Payload 重點 |
|------|----------|-------------|
| `invoice.paid` | 付款成功 | `invoice.payment_status = "succeeded"` |
| `invoice.payment_failure` | 付款失敗 | `invoice.payment_status = "failed"` |
| `invoice.payment_overdue` | 超過付款期限 | `invoice.payment_due_date` |
| `customer.payment_provider_created` | 客戶在金流商端建立成功 | `customer.provider_id` |
| `customer.payment_provider_error` | 客戶金流商設定失敗 | `error` 詳情 |
| `payment_request.payment_failure` | 收款請求失敗 | `payment_request` 詳情 |

這些 Webhook 的簽名方式在 `webhook_endpoints` 表中設定（HMAC 或 JWT），詳見架構設計文件。

---

## 11. 環境變數設定

在 `.env` 或 Docker Compose 中新增：

```bash
# ─── 你的金流商 API 設定 ────────────────────────────────────────────
# 注意：這些值在 UI 或 API 設定付款商時由使用者輸入
# 存入 DB 時透過 LAGO_ENCRYPTION_PRIMARY_KEY 加密
# 以下為開發環境的 API Key（不要 hardcode 到程式碼）

# ─── 加密金鑰（已有，確認設定正確）────────────────────────────────────
LAGO_ENCRYPTION_PRIMARY_KEY=your-32-char-primary-key
LAGO_ENCRYPTION_DETERMINISTIC_KEY=your-32-char-deterministic-key
LAGO_ENCRYPTION_KEY_DERIVATION_SALT=your-32-char-salt

# ─── Webhook 相關 ─────────────────────────────────────────────────
LAGO_WEBHOOK_ATTEMPTS=3          # Outbound Webhook 重試次數
LAGO_RSA_PRIVATE_KEY=...         # JWT 簽名（已有）

# ─── Sidekiq Worker（建議為付款啟用獨立 Worker）────────────────────
SIDEKIQ_BILLING=true             # 計費處理 Worker
SIDEKIQ_WEBHOOK=true             # Webhook 發送 Worker
```

---

## 12. 錯誤處理與重試機制

### 12.1 錯誤分類

```ruby
# Outbound（Lago 呼叫金流商）錯誤處理
begin
  response = your_provider_client.create_payment(...)
rescue YourProviderSDK::AuthenticationError
  # API Key 錯誤 → 不應重試，立刻通知
  deliver_error_webhook(e)
  result.third_party_failure!(...)

rescue YourProviderSDK::RateLimitError
  # 被限流 → 應重試，加入 Sidekiq 延遲隊列
  self.class.perform_in(5.minutes, invoice.id)

rescue YourProviderSDK::NetworkError, Timeout::Error
  # 網路問題 → 由 Clockwork 每小時 :10 執行 BillCustomers 重試
  invoice.update!(ready_for_payment_processing: true)
end
```

### 12.2 Inbound 重試流程

```
金流商發送 webhook
    │
    ▼
HandleEventJob 執行失敗（raise error）
    │
    ▼
inbound_webhook.status = :failed
    │
    ▼
Clockwork 每 15 分鐘掃描 status=failed 記錄
    │
    ▼
RetryInboundWebhooksJob.perform_later
    │
    ├── 重新加入 Sidekiq 隊列
    └── HandleEventJob 重新執行
```

### 12.3 Idempotency（冪等性）

**重要**：金流商 Webhook 可能重複發送，必須確保重複處理不會造成問題：

```ruby
def handle_payment_success(payload, organization)
  provider_invoice_id = payload.dig('data', 'invoice_id')
  invoice = Invoice.find_by(payment_provider_invoice_id: provider_invoice_id)

  return unless invoice
  # ↑ 找不到就跳過，避免重複處理

  # 使用 update_columns 搭配條件，避免重複觸發 callbacks
  return if invoice.payment_succeeded?  # ← 已成功則跳過
  # ↑ 冪等性檢查：避免重複發送 invoice.paid webhook

  invoice.update!(payment_status: :succeeded)
  SendWebhookJob.perform_later('invoice.paid', invoice)
end
```

---

## 13. 測試指引

### 13.1 測試架構

```ruby
# spec/services/payments/your_provider_service_spec.rb
require 'rails_helper'

RSpec.describe Payments::YourProviderService do
  subject(:service) { described_class.new(invoice) }

  let(:organization)    { create(:organization) }
  let(:customer)        { create(:customer, organization: organization) }
  let(:invoice)         { create(:invoice, customer: customer, total_amount_cents: 10_000) }
  let(:payment_provider) do
    create(:your_provider, organization: organization, api_key: 'test_key')
  end
  let(:provider_customer) do
    create(:your_provider_customer,
           customer: customer,
           payment_provider: payment_provider,
           settings: { 'provider_customer_id' => 'cust_test123' })
  end

  before { provider_customer }  # 確保關聯存在

  describe '#create' do
    context '付款成功' do
      before do
        allow(YourProviderSDK::Client).to receive(:new)
          .and_return(instance_double(YourProviderSDK::Client,
            create_payment: OpenStruct.new(id: 'pay_xxx', status: 'succeeded')
          ))
      end

      it '將發票狀態更新為 succeeded' do
        result = service.create
        expect(result).to be_success
        expect(invoice.reload.payment_status).to eq('succeeded')
      end

      it '觸發 invoice.paid webhook' do
        expect(SendWebhookJob).to receive(:perform_later)
          .with('invoice.paid', invoice)
        service.create
      end
    end

    context '付款失敗' do
      before do
        allow(YourProviderSDK::Client).to receive(:new)
          .and_return(instance_double(YourProviderSDK::Client,
            create_payment: OpenStruct.new(id: 'pay_xxx', status: 'failed')
          ))
      end

      it '將發票狀態更新為 failed' do
        service.create
        expect(invoice.reload.payment_status).to eq('failed')
      end
    end

    context 'API Key 錯誤' do
      before do
        allow(YourProviderSDK::Client).to receive(:new)
          .and_raise(YourProviderSDK::AuthenticationError, 'Invalid API key')
      end

      it '回傳 third_party_failure' do
        result = service.create
        expect(result).not_to be_success
      end
    end
  end
end
```

### 13.2 Webhook 簽名測試

```ruby
# spec/controllers/webhooks/your_provider_controller_spec.rb
require 'rails_helper'

RSpec.describe Webhooks::YourProviderController, type: :controller do
  let(:organization)     { create(:organization) }
  let(:payment_provider) do
    create(:your_provider,
           organization: organization,
           settings: { 'webhook_secret' => 'whsec_test' })
  end
  let(:payload) { { event_type: 'payment.succeeded', data: { invoice_id: 'pay_xxx' } }.to_json }

  before { payment_provider }

  describe 'POST #handle' do
    context '有效簽名' do
      let(:signature) do
        OpenSSL::HMAC.hexdigest('sha256', 'whsec_test', payload)
      end

      it '回傳 200 並建立 InboundWebhook 記錄' do
        request.headers['X-Your-Signature-Header'] = signature
        expect {
          post :handle,
               params: { organization_code: organization.api_key },
               body: payload,
               format: :json
        }.to change(InboundWebhook, :count).by(1)

        expect(response).to have_http_status(:ok)
      end
    end

    context '無效簽名' do
      it '回傳 401' do
        request.headers['X-Your-Signature-Header'] = 'invalid_signature'
        post :handle,
             params: { organization_code: organization.api_key },
             body: payload,
             format: :json

        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

### 13.3 本地手動測試

```bash
# 1. 模擬金流商發送 webhook（先計算簽名）
PAYLOAD='{"event_type":"payment.succeeded","data":{"invoice_id":"inv_xxx"}}'
SECRET='your_webhook_secret'
SIGNATURE=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

# 2. 發送測試請求
curl -X POST \
  http://localhost:3000/webhooks/your_provider/YOUR_ORG_API_KEY \
  -H "Content-Type: application/json" \
  -H "X-Your-Signature-Header: $SIGNATURE" \
  -d "$PAYLOAD"

# 預期：HTTP 200

# 3. 確認 InboundWebhook 記錄
docker exec -it lago-api rails runner \
  "puts InboundWebhook.last.attributes.to_yaml"

# 4. 確認 Sidekiq Job 執行
# 在 http://localhost:3000/sidekiq 查看 Jobs 狀態
```

---

## 14. 部署 Checklist

在上線前確認以下項目：

### 程式碼

- [ ] `PaymentProviders::YourProvider` Model 建立完成
- [ ] `PaymentProviders::YourProviderService` 建立完成（含 `handle_incoming_webhook`）
- [ ] `Payments::YourProviderService` 建立完成（含 `create`）
- [ ] `Webhooks::YourProviderController` 建立完成（含簽名驗證）
- [ ] `PaymentProviders::YourProvider::HandleEventJob` 建立完成
- [ ] 路由已新增
- [ ] 所有事件類型都有 idempotency 保護
- [ ] 錯誤都有適當的 rescue 和 webhook 通知

### 資料庫

- [ ] Migration 已執行（`rails db:migrate`）
- [ ] STI type 字串與 Model class 名稱一致

### 測試

- [ ] Unit tests 通過（Model、Service）
- [ ] Integration tests 通過（Controller + Job）
- [ ] 簽名驗證測試通過（有效 / 無效簽名）
- [ ] 冪等性測試通過（重複 webhook 不重複處理）

### 環境設定

- [ ] 加密金鑰環境變數已設定（`LAGO_ENCRYPTION_PRIMARY_KEY` 等）
- [ ] 金流商的 API Key 透過 Lago UI 或 API 設定，**不存在程式碼中**
- [ ] 金流商的 Webhook URL 已設定為 `https://your-lago.com/webhooks/your_provider/{ORG_API_KEY}`
- [ ] Webhook Secret 已在金流商後台設定，並儲存至 Lago

### 監控

- [ ] Sentry 或其他錯誤追蹤已設定
- [ ] `inbound_webhooks` 表的 `failed` 記錄有警報機制
- [ ] Sidekiq Dead Queue 有監控

---

## 15. 常見問題

### Q1：Webhook 一直收到 401

**原因**：簽名驗證失敗。
**排查**：
1. 確認 `webhook_secret` 已正確儲存到 DB
2. 確認簽名計算時使用的是原始 `request.raw_post`，不是解析後的 params
3. 確認 Header 名稱正確（大小寫敏感）
4. 部分金流商在計算簽名時包含 timestamp，需特別處理

```ruby
# 錯誤示範：使用 parsed params（已被修改）
def valid_signature?(payload, ...)   # payload = params.to_json  ← 錯

# 正確：使用原始 body
payload = request.raw_post   # ← 正確
```

---

### Q2：InboundWebhook 一直是 failed 狀態

**原因**：HandleEventJob 執行時拋出例外。
**排查**：
1. 查看 `processing_errors` 欄位的錯誤訊息
2. 查看 Sidekiq Dead Queue
3. 確認 `provider_invoice_id` 欄位有資料（對帳關鍵）

```sql
-- 查看失敗的 webhook 和錯誤訊息
SELECT id, source, status, processing_errors, created_at
FROM inbound_webhooks
WHERE status = 3  -- failed
ORDER BY created_at DESC
LIMIT 10;
```

---

### Q3：付款成功但 invoice.paid webhook 沒有發出

**原因**：`SendWebhookJob` 沒有被呼叫，或 webhook endpoint 未設定。
**排查**：
1. 確認程式碼中有呼叫 `SendWebhookJob.perform_later('invoice.paid', invoice)`
2. 確認組織有設定 webhook endpoint（在 Lago UI → Developer → Webhooks）
3. 查看 `webhooks` 表的記錄，確認 status

---

### Q4：同一張發票收到重複付款

**原因**：缺少冪等性保護。
**解法**：在 `handle_payment_success` 中加入狀態檢查：

```ruby
return if invoice.payment_succeeded?   # 已成功，跳過
```

---

### Q5：如何在 Lago UI 中顯示新的付款商選項

需要同時修改前端 (`lago-front` submodule)，新增對應的付款商設定 UI 元件。
若只需要 API 整合，可跳過前端，直接透過 Lago REST API 設定付款商：

```bash
# 透過 API 設定付款商
curl -X POST https://your-lago.com/api/v1/payment_providers/your_provider \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "your_provider": {
      "api_key": "your_api_key",
      "webhook_secret": "your_webhook_secret"
    }
  }'
```

---

*本文件由 Claude Code 根據 Lago v1.43.0 原始碼架構分析產生*
