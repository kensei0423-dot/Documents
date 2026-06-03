# Red Pay — PayPal Android 集成完整指南

> 面向第三方开发者。本文档覆盖 App 端 SDK 调用、服务端 API、前后端交互逻辑及完整请求示例。

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
│                  Android App (Kotlin)                    │
│                                                          │
│  PayPalManager          CardManager                      │
│  └─ PayPalWebCheckoutClient  └─ CardClient               │
│                                                          │
│  PaymentBottomSheet (Compose UI)                         │
│  └─ PayPalButton / PayLaterButton (官方按钮组件)           │
│  └─ PayPalMessageView (PLM 分期提示)                       │
└──────────────┬───────────────────────────────────────────┘
               │ HTTP (OkHttp) → http://10.0.2.2:3002
               ▼
┌──────────────────────────────────────────────────────────┐
│                Node.js Server (Express)                  │
│  POST /api/orders/create      — 创建 PayPal 订单          │
│  POST /api/orders/capture     — 捕获支付                  │
│  POST /api/orders/charge-vault — Vault 直接扣款           │
│  GET  /api/orders/:id/auth-check — 3DS 认证结果检查       │
│  GET  /api/vault-tokens       — 查询已保存 vault tokens   │
│  GET  /api/config             — 获取服务端配置            │
└──────────────┬───────────────────────────────────────────┘
               │ HTTPS (fetch)
               ▼
┌──────────────────────────────────────────────────────────┐
│         PayPal Sandbox API (api-m.sandbox.paypal.com)    │
│  POST /v2/checkout/orders                                │
│  POST /v2/checkout/orders/:id/capture                    │
│  GET  /v2/checkout/orders/:id                            │
└──────────────────────────────────────────────────────────┘
```

**关键常量（[PayPalManager.kt](../app/src/main/java/com/demo/red/PayPalManager.kt)）**

| 常量 | 值 | 说明 |
|------|-----|------|
| `PAYPAL_CLIENT_ID` | `AUbSpUcLC...` | PayPal Sandbox App 的 Client ID |
| `SERVER_URL` | `http://10.0.2.2:3002` | Android 模拟器访问本机服务的地址 |
| `RETURN_URL` | `com.demo.red` | Deep Link 前缀，用于 Chrome Tab 回调 |

> **真机调试**：将 `SERVER_URL` 改为电脑局域网 IP，例如 `http://192.168.1.100:3002`。

---

## 2. 环境配置

### 2.1 App 端依赖（[app/build.gradle.kts](../app/build.gradle.kts)）

```kotlin
// PayPal Web Checkout（PayPal / Pay Later — Chrome Custom Tab）
implementation("com.paypal.android:paypal-web-payments:2.3.0")

// PayPal Card Payments（ACDC — 原生卡号输入）
implementation("com.paypal.android:card-payments:2.3.0")

// PayPal Pay Later Messaging（PLM 分期提示组件）
implementation("com.paypal.messages:paypal-messages:1.3.0")

// PayPal 官方品牌按钮
implementation("com.paypal.android:payment-buttons:2.0.0")

// HTTP 客户端
implementation("com.squareup.okhttp3:okhttp:4.12.0")
```

### 2.2 AndroidManifest — Deep Link Intent Filter

Chrome Tab 完成后通过 Deep Link 回调 App，必须在 `AndroidManifest.xml` 的 `MainActivity` 中添加：

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTop"
    android:exported="true">

    <!-- PayPal 结账回调 -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="com.demo.red" android:host="paypalpay" />
    </intent-filter>

    <!-- ACDC 3DS 回调 -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="com.demo.red" android:host="card" />
    </intent-filter>
</activity>
```

> `android:launchMode="singleTop"` 确保 Deep Link 回调进入已有 Activity 的 `onNewIntent`，而不是创建新实例。

### 2.3 服务端环境变量（`.env`）

```bash
PAYPAL_CLIENT_ID=你的_PayPal_Client_ID
PAYPAL_SECRET=你的_PayPal_Secret
PORT=3002
```

---

## 3. 服务端启动

```bash
cd demo/red
npm install
node server.js
# 服务运行在 http://localhost:3002
# 管理后台：http://localhost:3002/admin
```

---

## 4. 服务端 API 文档

### 4.1 GET `/api/config`

获取服务端当前配置（App 启动时调用）。

**Response**

```json
{
  "useVault": true,
  "force3ds": false,
  "buyerCountry": "US",
  "showInstallment": true,
  "showSavePayLater": false,
  "currency": "USD",
  "clientId": "AUbSpUcLC..."
}
```

| 字段 | 说明 |
|------|------|
| `useVault` | 是否开启 Vault 直接扣款模式 |
| `force3ds` | 是否强制所有卡支付走 3DS |
| `buyerCountry` | PLM 分期消息的买家国家 |
| `currency` | 订单币种（USD / EUR / GBP / HKD）|

---

### 4.2 POST `/api/orders/create`

创建 PayPal 订单。根据支付方式不同，请求参数有所区别。

**URL**: `POST http://10.0.2.2:3002/api/orders/create`

**Content-Type**: `application/json`

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

> `saveCard: true` 时，服务端在 `payment_source.card.attributes.vault` 中注入 `store_in_vault: "ON_SUCCESS"`，同时绑定 `customer.id: "DEMO-CUSTOMER-001"`。

#### 场景 E：Card Vault 一键扣款（无感支付）

```json
{
  "amount": 120.00,
  "cardVaultId": "6DN69898PK337544H"
}
```

> 传入 `cardVaultId` 时，服务端使用 `payment_source.card.vault_id` + `stored_credential` 构建订单，PayPal 直接完成支付，App 只需等待 `status: "COMPLETED"` 即可，无需再调 capture。

---

### 4.3 POST `/api/orders/capture`

捕获已 APPROVED 的订单（PayPal / ACDC 流程在 SDK 回调成功后调用）。

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

> 若本次支付同时保存了支付方式，Response 中会带回 `vaultId`、`vaultType`、`vaultEmail`（PayPal 类型）或 `cardLast4`、`cardBrand`（Card 类型），App 需持久化到本地 SharedPreferences。

**Response（pending）**

```json
{
  "id": "2XE51484PK337544H",
  "status": "PENDING"
}
```

---

### 4.4 GET `/api/orders/:orderId/auth-check`

**仅 ACDC 流程使用。** 在 `cardManager.approveOrder` 成功回调后、调用 capture 之前，检查 3DS 认证结果。

**Request**: `GET /api/orders/2XE51484PK337544H/auth-check`

**Response**

```json
{
  "orderId": "2XE51484PK337544H",
  "liabilityShift": "POSSIBLE",
  "threeDSecure": {
    "authentication_status": "Y",
    "enrollment_status": "Y"
  },
  "paymentSource": { ... }
}
```

| `liabilityShift` 值 | 处理方式 |
|---------------------|---------|
| `POSSIBLE` 或空 | 继续 capture |
| `NO` | 拒绝：3DS 验证失败 |
| `UNKNOWN` | 拒绝：3DS 无法验证 |

---

### 4.5 POST `/api/orders/charge-vault`

PayPal Vault 一键扣款（不经过 Chrome Tab）。

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
  "id": "9KL23456CD789012E",
  "status": "COMPLETED"
}
```

> Vault 扣款订单在创建时即为 `COMPLETED`，无需再调 capture。

---

### 4.6 GET `/api/vault-tokens`

查询服务端保存的所有 vault tokens（App 启动时同步，用于判断是否展示一键支付 UI）。

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
    "email": "",
    "cardLast4": "1111",
    "cardBrand": "VISA",
    "customerId": "DEMO-CUSTOMER-001",
    "type": "card",
    "savedAt": "2026-06-01T13:00:00.000Z"
  }
]
```

---

### 4.7 GET `/api/orders/:orderId`

查询 PayPal 订单原始详情（调试用）。

**Response**: PayPal v2 Orders API 原始 JSON。

---

## 5. App 端 PayPal SDK 使用

### 5.1 SDK 初始化

在 `MainActivity.onCreate()` 中初始化两个 Manager：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    payPalManager = PayPalManager(this)   // PayPal / Pay Later
    cardManager = CardManager(this)       // 信用卡 ACDC
}
```

两个 Manager 均以 `Context` 为构造参数，内部创建对应 SDK Client：

```kotlin
// PayPalManager.kt
private val config = CoreConfig(PAYPAL_CLIENT_ID, environment = Environment.SANDBOX)
private val client = PayPalWebCheckoutClient(context, config, RETURN_URL)

// CardManager.kt
private val config = CoreConfig(PAYPAL_CLIENT_ID, environment = Environment.SANDBOX)
private val cardClient = CardClient(context, config)
```

---

### 5.2 Intent 路由（必须）

SDK 通过 Deep Link 将结果从 Chrome Tab 回传。必须在 `onResume` 和 `onNewIntent` 中都调用 `handleIntent`：

```kotlin
override fun onNewIntent(intent: android.content.Intent) {
    super.onNewIntent(intent)
    setIntent(intent)                        // 更新 Activity 的 intent
    cardManager.handleIntent(this, intent)   // 处理 3DS 回调
    payPalManager.handleIntent(intent)       // 处理 PayPal / Vault 回调
}

override fun onResume() {
    super.onResume()
    cardManager.handleIntent(this, intent)
    payPalManager.handleIntent(intent)
    payPalManager.cancelIfWaiting(intent)    // 用户直接关闭 Chrome Tab 时触发 onCancel
}
```

> **为什么 `onResume` 也要调？** 当 Chrome Tab 关闭后、Deep Link 没有触发时（用户按返回键），Activity 会走 `onResume`。此时调用 `handleIntent` 可以检测到 `NoResult`，配合 `cancelIfWaiting` 正确触发 `onCancel` 回调。

---

### 5.3 PayPalManager — PayPal 钱包 & Pay Later 结账

#### startCheckout()

```kotlin
payPalManager.startCheckout(
    activity = this@MainActivity,
    orderID = orderID,                              // 服务端创建的订单 ID
    fundingSource = PayPalWebCheckoutFundingSource.PAYPAL,  // 或 .PAY_LATER
    onSuccess = { id ->
        // Chrome Tab 完成，id = PayPal 订单 ID
        // 下一步：调 POST /api/orders/capture
    },
    onError = { msg ->
        // 错误信息
    },
    onCancel = {
        // 用户取消
    }
)
```

**内部流程**

```
App                      PayPalWebCheckoutClient         Chrome Tab (PayPal)
 │                              │                               │
 ├─ startCheckout() ──────────► │                               │
 │                              ├─ client.start() ─────────────► 打开 Chrome Tab
 │                              │   └─ PayPalPresentAuthChallengeResult.Success
 │                              │                               │
 │  ◄── 用户完成 PayPal 登录并确认付款 ──────────────────────────┤
 │                              │                               │
 ├─ onNewIntent(deepLink) ─────► │                               │
 │  handleIntent(intent)        ├─ client.finishStart(intent)
 │                              │   └─ PayPalWebCheckoutFinishStartResult.Success
 │                              │       └─ orderId
 └─ onSuccess(orderId) ◄────────┘
```

---

### 5.4 CardManager — 信用卡 ACDC（含 3DS）

#### approveOrder()

```kotlin
cardManager.approveOrder(
    activity = this@MainActivity,
    orderID = orderID,
    cardNumber = "4111111111111111",
    expirationMonth = "12",
    expirationYear = "2027",
    cvv = "123",
    force3ds = false,          // true = SCA_ALWAYS，false = SCA_WHEN_REQUIRED
    onSuccess = { id ->
        // 支付批准，id = 订单 ID
        // 下一步：调 GET /api/orders/:id/auth-check，再调 POST /api/orders/capture
    },
    onError = { msg -> },
    onCancel = { }
)
```

**3DS 触发流程**

```
App                   CardClient                  Browser（3DS 页面）
 │                        │                               │
 ├─ approveOrder() ──────► │                               │
 │                        ├─ cardClient.approveOrder()
 │                        │   └─ CardApproveOrderResult.AuthorizationRequired
 │                        ├─ cardClient.presentAuthChallenge()  ──────────────► 打开 3DS 页面
 │  hasPendingChallenge=true                               │
 │                        │  ◄── 用户完成 3DS 验证 ─────────┤
 ├─ onResume() ──────────► │                               │
 │  handleIntent(intent)   ├─ cardClient.finishApproveOrder(intent)
 │                        │   └─ CardFinishApproveOrderResult.Success
 └─ onSuccess(orderId) ◄──┘
```

**无 3DS 流程（直接完成）**

```
App                   CardClient
 │                        │
 ├─ approveOrder() ──────► │
 │                        ├─ cardClient.approveOrder()
 │                        │   └─ CardApproveOrderResult.Success（直接返回）
 └─ onSuccess(orderId) ◄──┘
```

#### Sealed Class 处理（approveOrder 回调）

```kotlin
// CardManager 内部
cardClient.approveOrder(request, CardApproveOrderCallback { result ->
    when (result) {
        is CardApproveOrderResult.Success -> {
            // 无需 3DS，直接成功
            onSuccess(result.orderId ?: orderID)
        }
        is CardApproveOrderResult.Failure -> {
            onError(result.error.errorDescription ?: "Card payment failed")
        }
        is CardApproveOrderResult.AuthorizationRequired -> {
            // 需要 3DS，弹出认证页面
            when (val cr = cardClient.presentAuthChallenge(activity, result.authChallenge)) {
                is CardPresentAuthChallengeResult.Success -> hasPendingChallenge = true
                is CardPresentAuthChallengeResult.Failure -> onError(...)
            }
        }
    }
})
```

#### finishApproveOrder 结果处理（handleIntent 内）

```kotlin
val result = cardClient.finishApproveOrder(intent) // null 表示本次 intent 无关
when (result) {
    is CardFinishApproveOrderResult.Success  -> onSuccess(result.orderId ?: orderId ?: "")
    is CardFinishApproveOrderResult.Failure  -> onError(result.error.errorDescription)
    CardFinishApproveOrderResult.Canceled    -> onCancel()
    CardFinishApproveOrderResult.NoResult    -> { /* 恢复 pending 回调，等下次 intent */ }
}
```

---

### 5.5 PLM — Pay Later 分期提示组件

在 Compose 中通过 `AndroidView` 嵌入官方 `PayPalMessageView`：

```kotlin
AndroidView(
    factory = { ctx ->
        com.paypal.messages.PayPalMessageView(ctx).also { view ->
            view.post {
                view.setConfig(
                    PayPalMessageConfig(
                        data = PayPalMessageData(
                            clientID = PAYPAL_CLIENT_ID,
                            amount = 120.0,
                            buyerCountry = "US",              // 控制展示哪个国家的分期
                            environment = PayPalEnvironment.SANDBOX,
                        ),
                        viewStateCallbacks = PayPalMessageViewStateCallbacks(
                            onError = { error -> },
                            onSuccess = { }
                        )
                    )
                )
            }
        }
    },
    modifier = Modifier.fillMaxWidth().height(56.dp),
)
```

---

### 5.6 官方品牌按钮（PaymentButtons）

在 Compose 中通过 `AndroidView` 使用：

```kotlin
// PayPal 金色按钮
AndroidView(
    factory = { ctx ->
        PayPalButton(ctx).apply {
            color = PayPalButtonColor.GOLD
            shape = PaymentButtonShape.ROUNDED
            size = PaymentButtonSize.MEDIUM
        }
    },
    modifier = Modifier.height(48.dp).width(160.dp),
)

// Pay Later 按钮
AndroidView(
    factory = { ctx ->
        PayLaterButton(ctx).apply {
            color = PayPalButtonColor.GOLD
            shape = PaymentButtonShape.ROUNDED
            size = PaymentButtonSize.MEDIUM
        }
    },
    modifier = Modifier.height(48.dp).width(160.dp),
)
```

> 按钮本身只负责渲染 UI，点击事件由外层 `Button` 的 `onClick` 处理，按钮不直接触发支付。

---

## 6. 支付流程详解（前后端交互）

### 6.1 PayPal 钱包支付（标准流程）

```
App                     Server                    PayPal API
 │                         │                           │
 ├─ POST /api/orders/create ►│                           │
 │  { amount, isCard:false } │                           │
 │                          ├─ POST /v2/checkout/orders ►│
 │                          │◄─ { id, status:"PAYER_ACTION_REQUIRED" }
 │◄─ { id } ───────────────┤                           │
 │                         │                           │
 ├─ payPalManager.startCheckout(orderID=id) ───────────────────────────────►
 │  (打开 Chrome Tab，用户登录 PayPal 并确认)                                │
 │◄── Deep Link 回调 (onSuccess, orderId) ─────────────────────────────────┤
 │                         │                           │
 ├─ POST /api/orders/capture ►│                           │
 │  { orderID }             │                           │
 │                          ├─ GET  /v2/checkout/orders/:id (验证 APPROVED)
 │                          ├─ POST /v2/checkout/orders/:id/capture ►│
 │                          │◄─ { status:"COMPLETED", ... }           │
 │◄─ { status:"COMPLETED" } ┤                           │
 │  → 展示支付成功           │                           │
```

### 6.2 信用卡 ACDC（无 3DS）

```
App                     Server                    PayPal API
 │                         │                           │
 ├─ POST /api/orders/create ►│                           │
 │  { amount, isCard:true }  │                           │
 │◄─ { id } ───────────────┤                           │
 │                         │                           │
 ├─ cardManager.approveOrder(orderID, card数据)          │
 │  └─ CardApproveOrderResult.Success（SDK 直接返回）     │
 │                         │                           │
 ├─ GET /api/orders/:id/auth-check ►│                   │
 │  liabilityShift = "POSSIBLE"     │                   │
 │                         │                           │
 ├─ POST /api/orders/capture ►│                           │
 │◄─ { status:"COMPLETED" } ┤                           │
 │  → 展示支付成功           │                           │
```

### 6.3 信用卡 ACDC（有 3DS）

```
App                     Server                    PayPal API    3DS 页面
 │                         │                           │             │
 ├─ POST /api/orders/create ─►│──► POST /v2/checkout/orders ──────────►│
 │◄─ { id } ───────────────┤                           │             │
 │                         │                           │             │
 ├─ cardManager.approveOrder(orderID, card数据, force3ds=true)        │
 │  └─ CardApproveOrderResult.AuthorizationRequired                   │
 │  └─ cardClient.presentAuthChallenge() ─────────────────────────────► 打开
 │  hasPendingChallenge = true           │             │             │
 │                         │            │  用户完成 3DS ►│◄────────────┤
 │◄── Deep Link onResume ──┤            │              │             │
 │  cardManager.handleIntent()          │              │             │
 │  └─ CardFinishApproveOrderResult.Success            │             │
 │                         │                           │             │
 ├─ GET /api/orders/:id/auth-check ─►│                 │             │
 │  liabilityShift = "POSSIBLE"       │                │             │
 │                         │                           │             │
 ├─ POST /api/orders/capture ─►│──► POST /v2/checkout/orders/:id/capture
 │◄─ { status:"COMPLETED" } ┤                           │             │
```

### 6.4 PayPal Vault 一键扣款

```
App                     Server                    PayPal API
 │                         │                           │
 │  （已有 savedVaultId，直接发起）                       │
 ├─ POST /api/orders/charge-vault ►│                   │
 │  { vaultId, amount }     │                           │
 │                          ├─ POST /v2/checkout/orders │
 │                          │  payment_source.paypal.vault_id = vaultId
 │                          │◄─ { status:"COMPLETED" }  │
 │◄─ { orderId, status:"COMPLETED" }                    │
 │  → 展示支付成功（无需 Chrome Tab）                     │
```

### 6.5 Card Vault 一键扣款

```
App                     Server                    PayPal API
 │                         │                           │
 │  （已有 savedCardVaultId，直接发起）                   │
 ├─ POST /api/orders/create ►│                          │
 │  { amount, cardVaultId }  │                          │
 │                          ├─ POST /v2/checkout/orders │
 │                          │  payment_source.card.vault_id = cardVaultId
 │                          │  stored_credential.payment_initiator = "MERCHANT"
 │                          │◄─ { id, status:"COMPLETED" }
 │◄─ { id } ───────────────┤                           │
 │  → 展示支付成功（无需 capture）                        │
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
     "customer_type": "CONSUMER",
     "permit_multiple_payment_tokens": true
   }
   ```
4. 用户在 Chrome Tab 完成支付后，App 调用 `POST /api/orders/capture`
5. Capture Response 中包含：
   ```json
   { "vaultId": "6DN69898...", "vaultType": "paypal", "vaultEmail": "buyer@example.com" }
   ```
6. App 将 `vaultId` 和 `vaultEmail` 存入 `SharedPreferences`（key: `paypal_vault_id`、`paypal_vault_email`）

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
4. ACDC 流程完成后，App 调用 `POST /api/orders/capture`
5. Capture Response 中包含：
   ```json
   { "vaultId": "7FM34567...", "vaultType": "card", "cardLast4": "1111", "cardBrand": "VISA" }
   ```
6. App 将数据存入 `SharedPreferences`

### 7.3 App 端 Vault 状态管理

```kotlin
// App 启动时同步（MainActivity.kt）
LaunchedEffect(Unit) {
    val prefs = context.getSharedPreferences("red_prefs", Context.MODE_PRIVATE)

    // 与服务端同步：如果服务端没有 vault tokens，清除本地缓存
    val serverTokens = getVaultTokens()  // GET /api/vault-tokens
    if (serverTokens.isEmpty()) {
        prefs.edit()
            .remove("paypal_vault_id").remove("paypal_vault_email")
            .remove("card_vault_id").remove("card_vault_last4").remove("card_vault_brand")
            .apply()
    } else {
        savedVaultId = prefs.getString("paypal_vault_id", null)
        savedVaultEmail = prefs.getString("paypal_vault_email", null)
        savedCardVaultId = prefs.getString("card_vault_id", null)
        savedCardLast4 = prefs.getString("card_vault_last4", null)
        savedCardBrand = prefs.getString("card_vault_brand", null)
    }
}
```

### 7.4 UI 展示逻辑

| 条件 | 展示内容 |
|------|---------|
| `useVaultMode=true` + `savedVaultId != null` | PayPal 区域显示已保存邮箱，勾选后直接扣款 |
| `useVaultMode=true` + `savedCardVaultId != null` | Card 区域显示 `VISA ****1111`，勾选后直接扣款 |
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

---

### 场景 B：PayPal 钱包结账 + 保存支付方式（Vault）

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

纯卡支付**不传 `payment_source`**，3DS 由 App 端 `CardClient` 内部处理：

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
      ]
    }
  ]
}
```

---

### 场景 D：信用卡 ACDC + 保存卡（Card Vault）

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
      ]
    }
  ],
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
      ]
    }
  ],
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

> `PayPal-Request-Id` 是幂等键，Card Vault 扣款必须携带，防止重复扣款。

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

| 场景 | `payment_source` 结构 | `shipping_preference` | 需要 Chrome Tab |
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
| 到期日 | — | 任意未来日期，如 `12/2027` |
| CVV | — | 任意 3 位，如 `123` |

> 完整 Sandbox 测试卡号参考：[PayPal Developer - Test Cards](https://developer.paypal.com/tools/sandbox/card-testing/)

---

## 10. 管理后台

访问 `http://localhost:3002/admin` 可实时控制以下配置（App 下次启动生效）：

| 功能 | 说明 |
|------|------|
| **Vault 模式开关** | 开启后已保存支付方式的用户走直接扣款，无需跳转 |
| **Force 3DS** | 强制所有卡支付走 3DS（SCA_ALWAYS）|
| **Buyer Country** | 控制 PLM 分期消息展示的国家 |
| **Currency** | 订单币种（USD / EUR / GBP / HKD）|
| **Installment UI** | 显示/隐藏分期付款明细展示 |
| **Save my Pay Later** | 显示/隐藏 Pay Later 保存复选框 |
| **PayPal Account** | 多账号切换（NL / US 等）|
| **Order Lookup** | 输入订单 ID 查看 PayPal 原始订单详情 |
| **Vault Tokens** | 查看并删除已保存的支付方式 |

---

## 附录：App 完整支付调用示意

```kotlin
// 伪代码 — 展示完整调用链路

// 1. 用户点击 "Buy Now" → 弹出 PaymentBottomSheet
// 2. 用户选择支付方式 → 点击支付按钮
// 3. 触发 onPayPalPay 回调

scope.launch {
    // Step 1: 创建订单
    val orderID = createOrder(amount, payLater, savePayPal, saveCard, isCard, shipping)
    // → POST /api/orders/create

    when (paymentMethod) {
        PAYPAL, PAY_LATER -> {
            // Step 2a: 启动 PayPal Chrome Tab
            payPalManager.startCheckout(
                activity, orderID, fundingSource,
                onSuccess = { id ->
                    // Step 3a: 捕获支付
                    val result = captureOrder(id)  // POST /api/orders/capture
                    if (result.status == "COMPLETED") showSuccess()
                },
                onError = { showError() },
                onCancel = { dismissProcessing() }
            )
        }
        CARD -> {
            // Step 2b: ACDC 卡支付
            cardManager.approveOrder(
                activity, orderID, cardNumber, expMonth, expYear, cvv, force3ds,
                onSuccess = { id ->
                    // Step 3b: 检查 3DS 结果
                    val authCheck = checkAuth(id)  // GET /api/orders/:id/auth-check
                    if (authCheck.liabilityShift != "NO") {
                        // Step 4b: 捕获支付
                        captureOrder(id)  // POST /api/orders/capture
                        showSuccess()
                    }
                },
                onError = { showError() },
                onCancel = { dismissProcessing() }
            )
        }
    }
}
```
