# Red Pay — PayPal iOS 集成完整指南

> 面向第三方开发者。本文档覆盖 App 端 SDK 调用、服务端 API、前后端交互逻辑及完整请求示例。
> 服务端与 Android 版本共用同一套 `server.js`，本文档仅记录 iOS 端差异。

---

## 目录

1. [项目架构概览](#1-项目架构概览)
2. [环境配置](#2-环境配置)
3. [服务端启动](#3-服务端启动)
4. [服务端 API 文档](#4-服务端-api-文档)
5. [App 端 PayPal SDK 使用](#5-app-端-paypal-sdk-使用)
6. [支付流程详解（前后端交互）](#6-支付流程详解前后端交互)
7. [Vault 保存与一键扣款](#7-vault-保存与一键扣款)
8. [PayPal API Request 完整示例](#8-paypal-api-request-完整示例)
9. [测试卡号参考](#9-测试卡号参考)
10. [管理后台](#10-管理后台)

---

## 1. 项目架构概览

```
┌──────────────────────────────────────────────────────────┐
│                   iOS App (Swift / SwiftUI)               │
│                                                           │
│  PayPalManager          CardManager                       │
│  └─ PayPalWebCheckoutClient  └─ CardClient                │
│                                                           │
│  PaymentSheetView (SwiftUI)                               │
│  └─ PayPalButton.Representable / PayPalPayLaterButton     │
│  └─ PLMView → PayPalMessageView.Representable (PLM)       │
└──────────────┬────────────────────────────────────────────┘
               │ HTTP (URLSession) → http://localhost:3002
               ▼
┌──────────────────────────────────────────────────────────┐
│                Node.js Server (Express)                   │
│  POST /api/orders/create      — 创建 PayPal 订单           │
│  POST /api/orders/capture     — 捕获支付                   │
│  POST /api/orders/charge-vault — Vault 直接扣款            │
│  GET  /api/orders/:id/auth-check — 3DS 认证结果检查        │
│  GET  /api/vault-tokens       — 查询已保存 vault tokens    │
│  GET  /api/config             — 获取服务端配置（含 clientId）│
└──────────────┬────────────────────────────────────────────┘
               │ HTTPS (fetch)
               ▼
┌──────────────────────────────────────────────────────────┐
│         PayPal Sandbox API (api-m.sandbox.paypal.com)     │
│  POST /v2/checkout/orders                                 │
│  POST /v2/checkout/orders/:id/capture                     │
│  GET  /v2/checkout/orders/:id                             │
└──────────────────────────────────────────────────────────┘
```

**关键常量（[Constants.swift](../RedIOS/Constants.swift)）**

| 常量 | 值 | 说明 |
|------|-----|------|
| `PAYPAL_CLIENT_ID` | `AUbSpUcLC...` | 初始 fallback，启动后从 `/api/config.clientId` 覆盖 |
| `SERVER_URL` | `http://localhost:3002` | 模拟器访问本机服务；真机改为局域网 IP |

> **关键设计**：iOS App 启动时调用 `/api/config`，拿到 `clientId` 后重新初始化 `PayPalManager` 和 `CardManager`，确保 SDK 与服务端使用同一个 PayPal 账号。
>
> **真机调试**：将 `SERVER_URL` 改为电脑局域网 IP，例如 `http://192.168.1.100:3002`。

---

## 2. 环境配置

### 2.1 App 端依赖（SPM — [project.yml](../project.yml)）

通过 Swift Package Manager 添加：

```yaml
# project.yml (xcodegen)
packages:
  PayPal:
    url: https://github.com/paypal/paypal-ios
    from: "2.0.0"
  PayPalMessages:
    url: https://github.com/paypal/paypal-messages-ios
    from: "1.2.0"

targets:
  RedIOS:
    dependencies:
      - package: PayPal
        product: PayPalWebPayments   # PayPal / Pay Later 结账
      - package: PayPal
        product: CardPayments        # ACDC 信用卡原生输入
      - package: PayPal
        product: PaymentButtons      # 官方品牌按钮
      - package: PayPalMessages
        product: PayPalMessages      # PLM 分期提示组件
```

或在 Xcode 中：`File → Add Package Dependencies` 添加以上两个仓库。

### 2.2 Info.plist — 允许本地 HTTP

iOS 默认只允许 HTTPS，需在 `Info.plist` 开启本地网络权限（xcodegen 已自动注入）：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsLocalNetworking</key>
    <true/>
</dict>
```

> **无需注册 URL Scheme。** iOS SDK 使用 `ASWebAuthenticationSession`，PayPal 结账和 3DS 回调由 SDK 内部拦截（callback scheme `sdk.ios.paypal://`），不经过 App 的 URL 路由系统。这是与 Android 最核心的差异。

### 2.3 服务端环境变量（`.env`）

```bash
PAYPAL_CLIENT_ID=你的_PayPal_Client_ID
PAYPAL_SECRET=你的_PayPal_Secret
PORT=3002
```

---

## 3. 服务端启动

```bash
cd demo/red          # iOS 与 Android 共用同一个 server.js
npm install
node server.js
# 服务运行在 http://localhost:3002
# 管理后台：http://localhost:3002/admin
```

---

## 4. 服务端 API 文档

服务端与 Android 版本完全共用，接口定义相同。以下仅标注 iOS 端调用入口。

### 4.1 GET `/api/config`

App 启动时调用，获取运行配置及当前激活账号的 `clientId`。

**iOS 调用**：`NetworkClient.shared.getConfig()` → [ContentView.swift](../RedIOS/ContentView.swift) `loadConfig()`

**Response**

```json
{
  "useVault": true,
  "force3ds": false,
  "buyerCountry": "US",
  "showInstallment": true,
  "showSavePayLater": false,
  "currency": "USD",
  "clientId": "AWrRw3Nan..."
}
```

| 字段 | 说明 |
|------|------|
| `useVault` | 是否开启 Vault 直接扣款模式 |
| `force3ds` | 是否强制所有卡支付走 3DS（SCA_ALWAYS）|
| `buyerCountry` | PLM 分期消息的买家国家 |
| `currency` | 订单币种（USD / EUR / GBP / HKD）|
| `clientId` | **iOS 专用**：用于重新初始化 SDK，确保账号一致 |

> **iOS 特有逻辑**：拿到 `clientId` 后立即调用：
> ```swift
> PayPalManager.configure(clientID: clientId)
> CardManager.configure(clientID: clientId)
> ```

---

### 4.2 POST `/api/orders/create`

创建 PayPal 订单。根据支付方式不同，请求参数有所区别。

**URL**: `POST http://localhost:3002/api/orders/create`

**Content-Type**: `application/json`

**iOS 调用**：`NetworkClient.shared.createOrder(...)`

#### 场景 A：PayPal 钱包结账（含 Pay Later）

```json
{
  "amount": 120.00,
  "payLater": false,
  "savePayPal": false,
  "isCard": false,
  "shipping": {
    "name": "Alex Lu",
    "line1": "3 San Jose",
    "city": "Laguna",
    "state": "NM",
    "zip": "87026-5026"
  }
}
```

**Response**

```json
{
  "id": "2XE51484PK337544H",
  "status": "PAYER_ACTION_REQUIRED",
  "links": [
    {
      "href": "https://www.sandbox.paypal.com/checkoutnow?token=2XE51484PK337544H",
      "rel": "payer-action",
      "method": "GET"
    }
  ]
}
```

#### 场景 B：PayPal 钱包结账 + 保存支付方式（Vault）

```json
{
  "amount": 120.00,
  "payLater": false,
  "savePayPal": true,
  "isCard": false
}
```

> `savePayPal: true` 时，服务端在 `payment_source.paypal.attributes.vault` 中注入 `store_in_vault: "ON_SUCCESS"`。

#### 场景 C：信用卡支付（ACDC）

```json
{
  "amount": 120.00,
  "isCard": true,
  "saveCard": false
}
```

#### 场景 D：信用卡支付 + 保存卡（Card Vault）

```json
{
  "amount": 120.00,
  "isCard": true,
  "saveCard": true
}
```

#### 场景 E：Card Vault 一键扣款（无感支付）

```json
{
  "amount": 120.00,
  "cardVaultId": "6DN69898PK337544H"
}
```

> 传入 `cardVaultId` 时，服务端使用 `payment_source.card.vault_id` + `stored_credential` 构建订单，PayPal 直接完成支付，无需再调 capture。

---

### 4.3 POST `/api/orders/capture`

捕获订单（PayPal / ACDC 流程在 SDK async 回调成功后调用）。

**iOS 调用**：`NetworkClient.shared.captureOrder(orderID:)`

**Request**

```json
{
  "orderID": "2XE51484PK337544H"
}
```

**Response（成功）**

```json
{
  "id": "2XE51484PK337544H",
  "status": "COMPLETED",
  "purchase_units": [
    {
      "payments": {
        "captures": [
          {
            "id": "5TY98765AB123456C",
            "status": "COMPLETED",
            "amount": { "currency_code": "USD", "value": "120.00" }
          }
        ]
      }
    }
  ],
  "vaultId": "6DN69898PK337544H",
  "vaultType": "paypal",
  "vaultEmail": "buyer@example.com",
  "cardLast4": null,
  "cardBrand": null
}
```

> 若本次支付同时保存了支付方式，Response 中会带回 `vaultId`、`vaultType`、`vaultEmail`（PayPal 类型）或 `cardLast4`、`cardBrand`（Card 类型），iOS App 持久化到 `UserDefaults`。

iOS 判断 capture 结果的逻辑（对标 Android）：

```swift
// CaptureResult.captureStatus 取 purchase_units[0].payments.captures[0].status
switch capture.captureStatus {
case "COMPLETED": finish(orderId: orderId)       // 成功
case "PENDING":   showPending = true              // 橙色 Order Pending 弹窗
default:          throw NSError(...)              // 失败
}
```

**Response（pending）**

```json
{ "id": "2XE51484PK337544H", "status": "PENDING" }
```

---

### 4.4 GET `/api/orders/:orderId/auth-check`

**仅 ACDC 流程使用。** `CardManager.approveOrder` 成功（`didAttemptThreeDSecureAuthentication = true`）后、调用 capture 之前，检查 3DS 认证结果。

**iOS 调用**：`NetworkClient.shared.authCheck(orderID:)`

**Request**: `GET /api/orders/2XE51484PK337544H/auth-check`

**Response**

```json
{
  "orderId": "2XE51484PK337544H",
  "liabilityShift": "POSSIBLE",
  "threeDSecure": {
    "authentication_status": "Y",
    "enrollment_status": "Y"
  }
}
```

| `liabilityShift` 值 | iOS 处理 |
|---------------------|---------|
| `POSSIBLE` 或 `nil` | 继续 capture |
| `NO` | throw：3DS 验证失败 |
| `UNKNOWN` | throw：3DS 无法验证 |

---

### 4.5 POST `/api/orders/charge-vault`

PayPal Vault 一键扣款（不弹出 PayPal 页面）。

**iOS 调用**：`NetworkClient.shared.chargeVault(vaultId:amount:shipping:)`

**Request**

```json
{
  "vaultId": "6DN69898PK337544H",
  "amount": 120.00,
  "shipping": {
    "name": "Alex Lu",
    "line1": "3 San Jose",
    "city": "Laguna",
    "state": "NM",
    "zip": "87026-5026"
  }
}
```

**Response**

```json
{
  "orderId": "9KL23456CD789012E",
  "status": "COMPLETED"
}
```

> Vault 扣款在创建时即为 `COMPLETED`，无需再调 capture。响应可能包含 `vaultId` 字段，iOS 需更新 `savedVaultId`。

---

### 4.6 GET `/api/vault-tokens`

查询服务端保存的所有 vault tokens（App 启动时调用，**直接从服务端数据填充状态**）。

**iOS 调用**：`NetworkClient.shared.getVaultTokens()` → 返回 `[VaultToken]`

**Response**

```json
[
  {
    "vaultId": "6DN69898PK337544H",
    "email": "buyer@example.com",
    "customerId": "DEMO-CUSTOMER-001",
    "type": "paypal",
    "savedAt": "2026-06-01T12:00:00.000Z"
  },
  {
    "vaultId": "7FM34567EF890123G",
    "cardLast4": "1111",
    "cardBrand": "VISA",
    "customerId": "DEMO-CUSTOMER-001",
    "type": "card",
    "savedAt": "2026-06-01T13:00:00.000Z"
  }
]
```

> **iOS 与 Android 的关键差异**：iOS 直接用服务端返回的数据填充 `savedVaultId`、`savedVaultEmail` 等状态，`UserDefaults` 仅作离线 fallback。这样 iOS 与 Android 创建的 vault token 可以互相共用，服务端是唯一数据源。

---

### 4.7 GET `/api/orders/:orderId`

查询 PayPal 订单原始详情（调试用）。

**Response**: PayPal v2 Orders API 原始 JSON。

---

## 5. App 端 PayPal SDK 使用

### 5.1 SDK 初始化

iOS SDK 使用静态工厂模式，支持动态切换 `clientId`：

```swift
// PayPalManager.swift
class PayPalManager {
    static var shared = PayPalManager(clientID: PAYPAL_CLIENT_ID)

    static func configure(clientID: String) {
        shared = PayPalManager(clientID: clientID)
    }

    private init(clientID: String) {
        let config = CoreConfig(clientID: clientID, environment: .sandbox)
        client = PayPalWebCheckoutClient(config: config)
    }
}

// CardManager.swift — 结构相同
class CardManager {
    static var shared = CardManager(clientID: PAYPAL_CLIENT_ID)
    static func configure(clientID: String) { ... }
}
```

**App 启动后**（`loadConfig()` 中）：

```swift
if let clientId = config.clientId, !clientId.isEmpty {
    PayPalManager.configure(clientID: clientId)   // 用服务端账号重新初始化
    CardManager.configure(clientID: clientId)
}
```

> **为什么要动态配置？** 服务端可能切换不同 PayPal Sandbox 账号（如 HK / NL）。SDK 必须使用与服务端相同的 `clientId` 才能处理订单，否则 `confirmPaymentSource` 会因账号不匹配返回 `UNPROCESSABLE_ENTITY`。

---

### 5.2 无需 Intent 路由（iOS vs Android 最大差异）

Android 必须在 `onResume` 和 `onNewIntent` 中手动路由 Deep Link：

```kotlin
// Android — 必须处理
override fun onNewIntent(intent: Intent) {
    cardManager.handleIntent(this, intent)
    payPalManager.handleIntent(intent)
}
```

**iOS 不需要任何路由代码。** `ASWebAuthenticationSession` 内部监听 `sdk.ios.paypal://` scheme，PayPal 结账和 3DS 结果直接通过 `async throws` 返回：

```swift
// iOS — 直接拿结果，无需 handleIntent
let result = try await PayPalManager.shared.startCheckout(orderID: orderID, fundingSource: .paypal)
// result.orderID → 调 captureOrder
```

**原理**：iOS SDK 创建 `ASWebAuthenticationSession` 时传入 `callbackURLScheme: "sdk.ios.paypal"`。PayPal 服务器收到 `integration_artifact=MOBILE_SDK` 参数后，将回调 URL 改为 `sdk.ios.paypal://...`。`ASWebAuthenticationSession` 拦截后直接触发 completion handler，App 无感知。

---

### 5.3 PayPalManager — PayPal 钱包 & Pay Later 结账

```swift
// 一行调用，async throws，结果直接返回
let result = try await PayPalManager.shared.startCheckout(
    orderID: orderID,
    fundingSource: .paypal    // 或 .paylater
)
// result.orderID → POST /api/orders/capture
// result.payerID → 可选，PayPal 买家标识
```

**内部流程**

```
ContentView                 PayPalWebCheckoutClient      ASWebAuthenticationSession
 │                                   │                           │
 ├─ startCheckout() ─────────────── ► │                           │
 │                                   ├─ client.start(request:) ─►│
 │                                   │   将 checkoutnow URL      │  弹出系统浏览器
 │                                   │   + integration_artifact  │  用户登录 PayPal
 │                                   │   传入 Session             │  并确认付款
 │                                   │                           │
 │                                   │  ◄── PayPal 重定向回 ──── ┤
 │                                   │     sdk.ios.paypal://...  │  Session 自动拦截
 │                                   │                           │
 │◄─ async result (orderID, payerID) ┤
 │  → 调 captureOrder
```

---

### 5.4 CardManager — 信用卡 ACDC（含 3DS）

```swift
let card = Card(
    number: "4111111111111111",
    expirationMonth: "12",
    expirationYear: "2027",
    securityCode: "123"
)

let result = try await CardManager.shared.approveOrder(
    orderID: orderID,
    card: card,
    force3ds: false    // true = SCA_ALWAYS，false = SCA_WHEN_REQUIRED
)
// result.didAttemptThreeDSecureAuthentication → 是否触发了 3DS
// → 若 true：先调 authCheck，再调 captureOrder
// → 若 false：直接调 captureOrder
```

**无 3DS 流程**

```
ContentView              CardClient (iOS SDK)
 │                              │
 ├─ approveOrder() ────────────► │
 │                              ├─ confirmPaymentSource() → APPROVED
 └─ async result (didAttempt=false) ◄──
```

**有 3DS 流程（SDK 自动处理）**

```
ContentView              CardClient                ASWebAuthenticationSession
 │                              │                           │
 ├─ approveOrder() ────────────► │                           │
 │                              ├─ confirmPaymentSource()   │
 │                              │   → PAYER_ACTION_REQUIRED │
 │                              ├─ 自动启动 Session ────────► 打开 3DS 页面
 │                              │                           │ 用户完成验证
 │                              │  ◄── 3DS 完成回调 ─────── ┤
 └─ async result (didAttempt=true) ◄──
```

> **iOS 3DS 与 Android 最大不同**：Android 是两步（`approveOrder` → `AuthorizationRequired` → `presentAuthChallenge` → `onResume` → `finishApproveOrder`）。iOS 是一步（`approveOrder` 内部自动完成所有步骤），`didAttemptThreeDSecureAuthentication` 标识是否发生了 3DS。

---

### 5.5 PLM — Pay Later 分期提示组件

PLM 使用独立文件 [PLMView.swift](../RedIOS/PLMView.swift) 封装，避免与 `CorePayments.Environment` 产生类型冲突：

```swift
// PLMView.swift — 只 import PayPalMessages，不 import PaymentButtons
import SwiftUI
import PayPalMessages

struct PLMView: View {
    let clientID: String
    let amount: Double
    let buyerCountry: String

    var body: some View {
        let data = PayPalMessageData(
            clientID: clientID,
            environment: .sandbox,
            amount: amount
        )
        let _ = { data.buyerCountry = buyerCountry }()
        let config = PayPalMessageConfig(data: data, style: PayPalMessageStyle())

        PayPalMessageView.Representable(
            config: config,
            stateDelegate: nil,
            eventDelegate: nil
        )
    }
}
```

在 `PaymentSheetView` 中使用：

```swift
PLMView(
    clientID: PAYPAL_CLIENT_ID,
    amount: amount,
    buyerCountry: buyerCountry   // 来自 /api/config
)
.frame(height: 56)
```

> **与 Android 的区别**：Android 用 `AndroidView` 包装原生 View；iOS 用 `UIViewRepresentable`（SDK 内部已封装为 `PayPalMessageView.Representable`）。PLM 必须放在 `PaymentOptionRow` 外部，避免 SwiftUI 手势系统拦截 "Learn more" 点击事件。

---

### 5.6 官方品牌按钮（PaymentButtons）

```swift
// PayPal 金色按钮（SwiftUI UIViewRepresentable 包装）
PayPalButton.Representable(
    color: .gold,
    edges: .softEdges,
    size: .collapsed
) {
    handlePayTap()   // 点击回调
}
.frame(width: 160, height: 48)

// Pay Later 按钮
PayPalPayLaterButton.Representable(
    color: .gold,
    edges: .softEdges,
    size: .collapsed
) {
    handlePayTap()
}
.frame(width: 160, height: 48)
```

> 按钮本身只负责渲染品牌 UI，点击回调由 trailing closure 处理，不直接触发支付。

---

## 6. 支付流程详解（前后端交互）

### 6.1 PayPal 钱包支付（标准流程）

```
ContentView             Server                    PayPal API
 │                         │                           │
 ├─ POST /api/orders/create ►│                           │
 │  { amount, isCard:false } │                           │
 │                          ├─ POST /v2/checkout/orders ►│
 │                          │◄─ { id, PAYER_ACTION_REQUIRED }
 │◄─ { id } ───────────────┤                           │
 │                         │                           │
 ├─ payPalManager.startCheckout(orderID, .paypal)
 │  [ASWebAuthenticationSession — 用户在系统浏览器完成 PayPal 登录]
 │◄── async result (orderID) ─────────────────────────────────────
 │                         │                           │
 ├─ POST /api/orders/capture ►│                           │
 │  { orderID }             │                           │
 │                          ├─ POST /v2/checkout/orders/:id/capture
 │                          │◄─ { status:"COMPLETED" }  │
 │◄─ { status:"COMPLETED" } ┤                           │
 │  → 展示支付成功           │                           │
```

### 6.2 信用卡 ACDC（无 3DS）

```
ContentView             Server                    PayPal API
 │                         │                           │
 ├─ POST /api/orders/create ►│──► POST /v2/checkout/orders
 │◄─ { id } ───────────────┤                           │
 │                         │                           │
 ├─ cardManager.approveOrder(orderID, card, force3ds=false)
 │  [SDK 内部调 confirmPaymentSource → APPROVED]
 │◄── async result (didAttempt=false) ────────────────────
 │                         │                           │
 ├─ GET /api/orders/:id/auth-check  (可选，liabilityShift 为空 → 跳过)
 ├─ POST /api/orders/capture ►│──► POST /v2/checkout/orders/:id/capture
 │◄─ { status:"COMPLETED" } ┤                           │
 │  → 展示支付成功           │                           │
```

### 6.3 信用卡 ACDC（有 3DS）

```
ContentView             Server                    PayPal API    3DS 页面
 │                         │                           │             │
 ├─ POST /api/orders/create ─►│──► POST /v2/checkout/orders ──────────►│
 │◄─ { id } ───────────────┤                           │             │
 │                         │                           │             │
 ├─ cardManager.approveOrder(orderID, card, force3ds=true)
 │  [SDK 内部: confirmPaymentSource → PAYER_ACTION_REQUIRED]
 │  [SDK 内部: ASWebAuthenticationSession 自动打开 3DS 页面 ──────────►]
 │                         │            │  用户完成 3DS ►│◄────────────┤
 │◄── async result (didAttempt=true) ─────────────────────────────────
 │                         │                           │             │
 ├─ GET /api/orders/:id/auth-check ─►│                 │             │
 │  liabilityShift = "POSSIBLE"       │                │             │
 │                         │                           │             │
 ├─ POST /api/orders/capture ─►│──► POST /v2/checkout/orders/:id/capture
 │◄─ { status:"COMPLETED" } ┤                           │             │
 │  → 展示支付成功           │                           │             │
```

### 6.4 PayPal Vault 一键扣款

```
ContentView             Server                    PayPal API
 │                         │                           │
 │  （已有 savedVaultId，直接发起，无需弹出 PayPal 页面）  │
 ├─ POST /api/orders/charge-vault ►│                   │
 │  { vaultId, amount }     │                           │
 │                          ├─ POST /v2/checkout/orders │
 │                          │  payment_source.paypal.vault_id
 │                          │◄─ { status:"COMPLETED" }  │
 │◄─ { orderId, status:"COMPLETED" }                    │
 │  → 展示支付成功           │                           │
```

### 6.5 Card Vault 一键扣款

```
ContentView             Server                    PayPal API
 │                         │                           │
 │  （已有 savedCardVaultId，直接发起）                  │
 ├─ POST /api/orders/create ►│──► POST /v2/checkout/orders
 │  { amount, cardVaultId } │  payment_source.card.vault_id
 │                          │  stored_credential.payment_initiator = "MERCHANT"
 │                          │◄─ { id, status:"COMPLETED" }
 │◄─ { id } ───────────────┤                           │
 │  → 展示支付成功（无需 capture）                       │
```

---

## 7. Vault 保存与一键扣款

### 7.1 PayPal Vault 保存流程

1. 用户勾选 "Save my PayPal"
2. App 调用 `POST /api/orders/create` 时传 `savePayPal: true`
3. 服务端在 `payment_source.paypal.attributes.vault` 中加入：
   ```json
   {
     "store_in_vault": "ON_SUCCESS",
     "usage_type": "MERCHANT",
     "customer_type": "CONSUMER"
   }
   ```
4. 用户通过 `ASWebAuthenticationSession` 完成 PayPal 登录和确认
5. App 调用 `POST /api/orders/capture`
6. Capture Response 中包含：
   ```json
   { "vaultId": "6DN69898...", "vaultType": "paypal", "vaultEmail": "buyer@example.com" }
   ```
7. App 将数据存入 `UserDefaults`（key: `paypal_vault_id`、`paypal_vault_email`）

### 7.2 Card Vault 保存流程

1. 用户勾选 "Save card for future payments"
2. App 调用 `POST /api/orders/create` 时传 `saveCard: true`
3. 服务端在 `payment_source.card.attributes.vault` 中加入：
   ```json
   {
     "store_in_vault": "ON_SUCCESS",
     "usage_type": "MERCHANT",
     "customer_type": "CONSUMER"
   }
   ```
4. ACDC 流程完成（`approveOrder` async 返回）后调用 `POST /api/orders/capture`
5. Capture Response 中包含：
   ```json
   { "vaultId": "7FM34567...", "vaultType": "card", "cardLast4": "1111", "cardBrand": "VISA" }
   ```
6. App 将数据存入 `UserDefaults`

### 7.3 App 端 Vault 状态管理

```swift
// ContentView.swift — loadConfig() 中
let tokens = try await NetworkClient.shared.getVaultTokens()  // GET /api/vault-tokens
if tokens.isEmpty {
    // 清除 UserDefaults 所有 vault 数据
    ["paypal_vault_id", "paypal_vault_email",
     "card_vault_id", "card_vault_last4", "card_vault_brand"].forEach {
        UserDefaults.standard.removeObject(forKey: $0)
    }
} else {
    // 直接用服务端数据填充状态（server 是 source of truth）
    for token in tokens {
        if token.type == "paypal" {
            savedVaultId    = token.vaultId
            savedVaultEmail = token.email    // 服务端有邮箱则直接用
        } else if token.type == "card" {
            savedCardVaultId = token.vaultId
            savedCardLast4   = token.cardLast4
            savedCardBrand   = token.cardBrand
        }
    }
}
```

> **iOS 与 Android 行为一致**：两端均直接从服务端数据填充 vault 状态，本机存储（`UserDefaults` / `SharedPreferences`）仅作网络不可用时的离线 fallback。因此 **iOS 和 Android 的 vault token 可以互相共用**——无论哪个平台保存，另一个平台启动时都能从服务端加载到。

### 7.4 UI 展示逻辑

| 条件 | 展示内容 |
|------|---------|
| `useVaultMode=true` + `savedVaultId != nil` + email 非空 | PayPal 区域显示已保存邮箱 checkbox，勾选后直接扣款 |
| `useVaultMode=true` + `savedCardVaultId != nil` | Card 区域显示 `VISA ****1111`，勾选后直接扣款 |
| 无 vault 数据 | 显示正常支付流程 + "Save my PayPal" / "Save card" 复选框 |

> `useVaultMode` 由服务端 `/api/config` 控制，可在管理后台实时切换。

---

## 8. PayPal API Request 完整示例

> 以下是服务端实际发送给 `https://api-m.sandbox.paypal.com/v2/checkout/orders` 的请求体，按场景分类。所有场景均需先在服务端获取 Access Token：
>
> ```
> POST /v1/oauth2/token
> Authorization: Basic Base64(clientId:secret)
> Body: grant_type=client_credentials
> ```

---

### 场景 A：PayPal 钱包结账（不保存）

```json
POST https://api-m.sandbox.paypal.com/v2/checkout/orders
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "intent": "CAPTURE",
  "purchase_units": [
    {
      "amount": {
        "currency_code": "USD",
        "value": "120.00",
        "breakdown": {
          "item_total": { "currency_code": "USD", "value": "120.00" }
        }
      },
      "description": "Pro Wireless Earphones",
      "items": [
        {
          "name": "Pro Wireless Earphones",
          "description": "Premium wireless earphones with noise cancellation",
          "url": "https://www.example.com/products/pro-wireless-earphones",
          "image_url": "https://image.16pic.com/00/93/65/16pic_9365586_s.png",
          "unit_amount": { "currency_code": "USD", "value": "120.00" },
          "quantity": "1",
          "category": "PHYSICAL_GOODS"
        }
      ],
      "shipping": {
        "name": { "full_name": "Alex Lu" },
        "address": {
          "address_line_1": "3 San Jose",
          "admin_area_2": "Laguna",
          "admin_area_1": "NM",
          "postal_code": "87026-5026",
          "country_code": "US"
        }
      }
    }
  ],
  "payment_source": {
    "paypal": {
      "experience_context": {
        "return_url": "com.demo.red://paypalpay",
        "cancel_url": "com.demo.red://paypalpay",
        "user_action": "PAY_NOW",
        "shipping_preference": "SET_PROVIDED_ADDRESS"
      }
    }
  }
}
```

> **iOS 注意**：`return_url` / `cancel_url` 对 iOS SDK 无效。`PayPalWebCheckoutClient` 传入 `integration_artifact=MOBILE_SDK`，PayPal 会使用 `sdk.ios.paypal://` 作为实际回调，`experience_context` 中的 URL 被忽略。

---

### 场景 B：PayPal 钱包结账 + 保存支付方式（Vault）

```json
{
  "intent": "CAPTURE",
  "purchase_units": [ { "...": "同场景 A，含 shipping 字段" } ],
  "payment_source": {
    "paypal": {
      "attributes": {
        "customer": { "merchant_customer_id": "DEMO-CUSTOMER-001" },
        "vault": {
          "store_in_vault": "ON_SUCCESS",
          "usage_type": "MERCHANT",
          "customer_type": "CONSUMER"
        }
      },
      "experience_context": {
        "return_url": "com.demo.red://paypalpay",
        "cancel_url": "com.demo.red://paypalpay",
        "user_action": "PAY_NOW",
        "shipping_preference": "SET_PROVIDED_ADDRESS"
      }
    }
  }
}
```

---

### 场景 C：信用卡 ACDC（不保存）

纯卡支付**不传 `payment_source`**，3DS 由 `CardClient` 内部自动处理：

```json
{
  "intent": "CAPTURE",
  "purchase_units": [ { "...": "同场景 A，含 items，无需 shipping" } ]
}
```

---

### 场景 D：信用卡 ACDC + 保存卡（Card Vault）

```json
{
  "intent": "CAPTURE",
  "purchase_units": [ { "...": "同场景 C" } ],
  "payment_source": {
    "card": {
      "attributes": {
        "customer": { "merchant_customer_id": "DEMO-CUSTOMER-001" },
        "vault": {
          "store_in_vault": "ON_SUCCESS",
          "usage_type": "MERCHANT",
          "customer_type": "CONSUMER"
        }
      }
    }
  }
}
```

---

### 场景 E：Card Vault 一键扣款（merchant-initiated）

```json
POST https://api-m.sandbox.paypal.com/v2/checkout/orders
Authorization: Bearer <access_token>
Content-Type: application/json
PayPal-Request-Id: card-vault-1748870400000-x7k3m2

{
  "intent": "CAPTURE",
  "purchase_units": [ { "...": "含 amount 和 items" } ],
  "payment_source": {
    "card": {
      "vault_id": "7FM34567EF890123G",
      "stored_credential": {
        "payment_initiator": "MERCHANT",
        "payment_type": "UNSCHEDULED",
        "usage": "SUBSEQUENT"
      }
    }
  }
}
```

---

### 场景 F：PayPal Vault 一键扣款（含收货地址）

```json
POST https://api-m.sandbox.paypal.com/v2/checkout/orders
Authorization: Bearer <access_token>
Content-Type: application/json
PayPal-Request-Id: vault-1748870400000-a9b3c1

{
  "intent": "CAPTURE",
  "purchase_units": [
    {
      "amount": { "currency_code": "USD", "value": "120.00", "breakdown": { "item_total": { "currency_code": "USD", "value": "120.00" } } },
      "description": "Pro Wireless Earphones",
      "items": [ { "...": "同场景 A items" } ],
      "shipping": {
        "name": { "full_name": "Alex Lu" },
        "address": {
          "address_line_1": "3 San Jose",
          "admin_area_2": "Laguna",
          "admin_area_1": "NM",
          "postal_code": "87026-5026",
          "country_code": "US"
        }
      }
    }
  ],
  "payment_source": {
    "paypal": {
      "vault_id": "6DN69898PK337544H",
      "experience_context": {
        "shipping_preference": "SET_PROVIDED_ADDRESS"
      }
    }
  }
}
```

---

### 各场景 `payment_source` 结构对比

| 场景 | `payment_source` 结构 | `shipping_preference` | 需要 PayPal 页面 |
|------|----------------------|----------------------|:--------------:|
| PayPal 普通结账 | `paypal.experience_context` | `SET_PROVIDED_ADDRESS` | ✅ |
| PayPal + 保存 | `paypal.attributes.vault` + `experience_context` | `SET_PROVIDED_ADDRESS` | ✅ |
| ACDC 纯卡 | 无 `payment_source` | — | ❌ |
| ACDC + 保存卡 | `card.attributes.vault` | — | ❌ |
| Card Vault 扣款 | `card.vault_id` + `stored_credential` | — | ❌ |
| PayPal Vault 扣款 | `paypal.vault_id` + `experience_context` | `SET_PROVIDED_ADDRESS` | ❌ |

---

## 9. 测试卡号参考

| 卡号 | 类型 | 说明 |
|------|------|------|
| `4111 1111 1111 1111` | VISA | 无 3DS，直接通过 |
| `4012 0000 3333 0026` | VISA | 触发 3DS 验证 |
| `5455 3307 6000 0018` | Mastercard | 无 3DS |
| 到期日 | — | 任意未来日期，如 `12/2027`（输入 `1227`，UI 自动格式化为 `12/27`）|
| CVV | — | 任意 3 位，如 `123` |

> 完整 Sandbox 测试卡号参考：[PayPal Developer - Test Cards](https://developer.paypal.com/tools/sandbox/card-testing/)

---

## 10. 管理后台

访问 `http://localhost:3002/admin` 可实时控制以下配置（iOS App 下次启动生效）：

| 功能 | 说明 |
|------|------|
| **Vault 模式开关** | 开启后已保存支付方式的用户走直接扣款，不弹出 PayPal 页面 |
| **Force 3DS** | 强制所有卡支付走 3DS（SCA_ALWAYS）|
| **Buyer Country** | 控制 PLM 分期消息展示的国家 |
| **Currency** | 订单币种（USD / EUR / GBP / HKD）|
| **Installment UI** | 显示 / 隐藏 Pay Later 分期付款明细展示 |
| **Save my Pay Later** | 显示 / 隐藏 Pay Later 保存复选框 |
| **PayPal Account** | 多账号切换（HK / NL 等）— 切换后 iOS App 重启生效（SDK 重新初始化 clientId）|
| **Order Lookup** | 输入订单 ID 查看 PayPal 原始订单详情 |
| **Vault Tokens** | 查看并删除已保存的支付方式 |

---

## 附录：iOS vs Android 关键差异速查

| 方面 | Android | iOS |
|------|---------|-----|
| PayPal 结账方式 | Chrome Custom Tab | `ASWebAuthenticationSession` |
| URL Scheme 注册 | ✅ 必须（AndroidManifest intent-filter）| ❌ 不需要 |
| Intent 路由 | `onNewIntent` + `handleIntent` | 无，async/await 直接返回 |
| 3DS 流程 | 两步：`approveOrder` → `presentAuthChallenge` → `finishApproveOrder` | 一步：`approveOrder` 内部自动完成 |
| PLM SDK | `com.paypal.messages:paypal-messages:1.3.0` | `paypal-messages-ios` SPM |
| 并发模型 | Kotlin Coroutines | Swift async/await |
| 本地存储 | `SharedPreferences` | `UserDefaults` |
| Vault 数据源 | **服务端直接填充**（跨平台共用）| **服务端直接填充**（跨平台共用）|
| SDK clientId | 硬编码常量 | 启动后从 `/api/config.clientId` 动态更新 |
| 服务端地址 | `http://10.0.2.2:3002`（模拟器） | `http://localhost:3002`（模拟器）|
