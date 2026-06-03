# Red Pay Demo — 项目概览

这是一个 PayPal Android 支付 Demo，模拟类似小红书/电商 App 的购物支付体验。

---

## 📁 项目结构

| 部分 | 说明 |
|------|------|
| `server.js` | Node.js 后端，运行在 port 3002 |
| `app/` | Android 原生应用（Kotlin + Jetpack Compose） |
| `db.json` | 本地数据库（配置 + vault tokens） |
| `.env` | 环境变量（PayPal 账号凭证） |
| `paypal-android-main/` | PayPal Android SDK 源码参考 |
| `paypal-ios-main/` | PayPal iOS SDK 源码参考 |

---

## 🖥️ 后端 API（server.js）

| 接口 | 功能 |
|------|------|
| `GET /api/config` | 获取配置（useVault、force3ds、currency 等） |
| `POST /api/orders/create` | 创建 PayPal 订单 |
| `POST /api/orders/capture` | 捕获订单，自动保存 vault token |
| `POST /api/orders/charge-vault` | Vault 直接扣款（无需跳转 PayPal） |
| `GET/POST/DELETE /api/vault-tokens` | 管理已保存的支付方式 |
| `GET /admin` | 管理后台页面 |
| `GET /plm` | Pay Later Messaging WebView 页面 |

---

## 📱 Android 应用（Kotlin + Compose）

支持 3 种支付方式：

- **PayPal** — Chrome Custom Tab 跳转，支持 Vault 保存
- **Pay Later (BNPL)** — 分期付款，集成 PLM 消息组件
- **Card (ACDC)** — 原生卡号输入，支持 3DS 验证 + Vault 保存

核心文件：

- `MainActivity.kt` — 商品页面 + 支付流程控制
- `PaymentBottomSheet.kt` — 支付底部弹窗 UI
- `PayPalManager.kt` — PayPal Web Checkout 管理
- `CardManager.kt` — ACDC 卡支付 + 3DS 管理

---

## ⚙️ 当前配置（db.json）

| 配置项 | 当前值 |
|--------|--------|
| 活跃账号 | HK（港区账号） |
| 货币 | USD |
| Vault 模式 | ✅ 开启 |
| Force 3DS | ✅ 开启 |
| Buyer Country | US |
| 分期 UI | ❌ 关闭 |

4 个 PayPal 沙盒账号：NL、C2、HK、US

---

## 🔧 SDK 版本（build.gradle.kts）

| 模块 | 版本 |
|------|------|
| `paypal-web-payments` | 2.3.0 |
| `card-payments` | 2.3.0 |
| `paypal-messages` | 1.3.0 |
| `payment-buttons` | 2.0.0 |

---

## 🚀 启动方式

- **后端**：`cd demo/red && node server.js`（port 3002）
- **Android**：用 Android Studio 打开 `demo/red/` 目录，连接模拟器运行
- **管理后台**：浏览器访问 `http://localhost:3002/admin`

---

# PayPal Mobile SDK 在 Red 项目中的完整使用说明

## 一、SDK 依赖配置

文件：`app/build.gradle.kts`

```kotlin
// PayPal Web Checkout（Chrome Custom Tab 跳转）
implementation("com.paypal.android:paypal-web-payments:2.3.0")

// ACDC 原生卡支付（含 3DS）
implementation("com.paypal.android:card-payments:2.3.0")

// Pay Later Messaging 消息组件
implementation("com.paypal.messages:paypal-messages:1.3.0")

// PayPal 官方品牌按钮
implementation("com.paypal.android:payment-buttons:2.0.0")
```

---

## 二、AndroidManifest 配置

文件：`app/src/main/AndroidManifest.xml`

```xml
<!-- 网络权限 -->
<uses-permission android:name="android.permission.INTERNET" />

<!-- 允许明文 HTTP（连接本地 server） -->
<application android:usesCleartextTraffic="true">

<!-- Activity 必须设置 singleTop，防止 Chrome Tab 回调时重建 -->
<activity
    android:launchMode="singleTop"
    android:windowSoftInputMode="adjustResize">

    <!-- PayPal 回调 Deep Link：scheme = com.demo.red -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="com.demo.red" />
    </intent-filter>
```

> ⚠️ 关键：`android:scheme` 必须与 `server.js` 中的 `RETURN_URL` 和 `PayPalManager.kt` 中的 `RETURN_URL` 完全一致。

---

## 三、全局常量

文件：`PayPalManager.kt`（第 16-18 行）

```kotlin
const val PAYPAL_CLIENT_ID = "AUbSpUcLCXCQMo5Tqqz4sv4d56ALQxpYua7eaPHNChuy..."
const val SERVER_URL = "http://10.0.2.2:3002"   // 模拟器访问宿主机 localhost
const val RETURN_URL = "com.demo.red"            // Deep Link scheme
```

---

## 四、模块一：PayPal Web Checkout（Chrome Custom Tab）

文件：`PayPalManager.kt`

### 4.1 初始化

```kotlin
// 创建 CoreConfig（clientId + 环境）
val config = CoreConfig(PAYPAL_CLIENT_ID, environment = Environment.SANDBOX)

// 创建 PayPalWebCheckoutClient（context + config + returnUrl）
val client = PayPalWebCheckoutClient(context, config, RETURN_URL)
```

### 4.2 发起支付

```kotlin
// 构建请求（orderId + 资金来源）
val request = PayPalWebCheckoutRequest(
    orderID,
    fundingSource = PayPalWebCheckoutFundingSource.PAYPAL  // 或 PAY_LATER
)

// 启动 Chrome Custom Tab
client.start(activity, request, PayPalWebStartCallback { result ->
    when (result) {
        is PayPalPresentAuthChallengeResult.Success -> { /* Chrome Tab 已打开 */ }
        is PayPalPresentAuthChallengeResult.Failure -> { /* 启动失败 */ }
    }
})
```

### 4.3 处理回调（onNewIntent + onResume）

文件：`MainActivity.kt`（第 59-72 行）

```kotlin
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    setIntent(intent)
    payPalManager.handleIntent(intent)  // 处理 PayPal 回调
}

override fun onResume() {
    super.onResume()
    payPalManager.handleIntent(intent)
    payPalManager.cancelIfWaiting(intent)  // 用户关闭 Chrome Tab 时取消
}
```

文件：`PayPalManager.kt`（第 55-101 行）

```kotlin
fun handleIntent(intent: Intent) {
    // 处理普通支付回调
    client.finishStart(intent)?.let { result ->
        when (result) {
            is PayPalWebCheckoutFinishStartResult.Success -> onSuccess(result.orderId)
            is PayPalWebCheckoutFinishStartResult.Failure -> onError(result.error)
            is PayPalWebCheckoutFinishStartResult.Canceled -> onCancel()
            PayPalWebCheckoutFinishStartResult.NoResult -> { /* 保持等待 */ }
        }
    }

    // 处理 Vault 回调
    client.finishVault(intent)?.let { result ->
        when (result) {
            is PayPalWebCheckoutFinishVaultResult.Success -> onSuccess(result.approvalSessionId)
            // ...
        }
    }
}

// 用户关闭 Chrome Tab 未完成支付时取消
fun cancelIfWaiting(intent: Intent) {
    val hasPayPalData = intent.data?.toString()?.contains(RETURN_URL) == true
    if (!hasPayPalData && pendingOnCancel != null) {
        onCancel()
    }
}
```

### 4.4 支付流程（MainActivity.kt 第 362-418 行）

```kotlin
// 1. 后端创建订单
val orderID = createOrder(amount, payLater = false, savePayPal = true)

// 2. 启动 Chrome Tab
payPalManager.startCheckout(
    activity = this,
    orderID = orderID,
    fundingSource = PayPalWebCheckoutFundingSource.PAYPAL,
    onSuccess = { id ->
        // 3. 后端 capture 订单
        val result = captureOrder(id, context)
        // 4. 从 capture 响应中提取 vault token 并保存
        val vaultId = result.getString("vaultId")
    },
    onError = { msg -> /* 处理错误 */ },
    onCancel = { /* 用户取消 */ }
)
```

---

## 五、模块二：Card Payments（ACDC 原生卡支付）

文件：`CardManager.kt`

### 5.1 初始化

```kotlin
val config = CoreConfig(PAYPAL_CLIENT_ID, environment = Environment.SANDBOX)
val cardClient = CardClient(context, config)
```

### 5.2 发起卡支付

```kotlin
// 构建 Card 对象
val card = Card(
    number = "4111111111111111",
    expirationMonth = "01",
    expirationYear = "2027",
    securityCode = "123",
)

// 设置 3DS 策略
val sca = if (force3ds) SCA.SCA_ALWAYS else SCA.SCA_WHEN_REQUIRED

// 构建请求（orderId + card + returnUrl + sca）
val request = CardRequest(orderID, card, "$RETURN_URL://card", sca = sca)

// 发起支付
cardClient.approveOrder(request, CardApproveOrderCallback { result ->
    when (result) {
        is CardApproveOrderResult.Success -> onSuccess(result.orderId)
        is CardApproveOrderResult.Failure -> onError(result.error)
        is CardApproveOrderResult.AuthorizationRequired -> {
            // 需要 3DS 验证，弹出 3DS 页面
            cardClient.presentAuthChallenge(activity, result.authChallenge)
            hasPendingChallenge = true
        }
    }
})
```

### 5.3 处理 3DS 回调

文件：`CardManager.kt`（第 82-122 行）

```kotlin
fun handleIntent(activity: Activity, intent: Intent) {
    val result = cardClient.finishApproveOrder(intent)
    when (result) {
        is CardFinishApproveOrderResult.Success -> onSuccess(result.orderId)
        is CardFinishApproveOrderResult.Failure -> onError(result.error)
        CardFinishApproveOrderResult.Canceled -> onCancel()
        CardFinishApproveOrderResult.NoResult -> {
            if (wasChallengePending) onCancel()  // 3DS 页面被关闭
        }
    }
}
```

### 5.4 卡支付完整流程（MainActivity.kt 第 323-361 行）

```kotlin
// 1. 后端创建订单（含 saveCard 参数）
val orderID = createOrder(amount, isCard = true, saveCard = true)

// 2. 发起 ACDC 支付
cardManager.approveOrder(
    activity = this,
    orderID = orderID,
    cardNumber = "4111111111111111",
    expirationMonth = "01",
    expirationYear = "2027",
    cvv = "123",
    force3ds = true,
    onSuccess = { id ->
        // 3. 检查 3DS 结果（auth-check）
        checkAuthAndCapture(id, context)
        // 4. 后端 capture，自动保存 card vault token
    }
)
```

### 5.5 3DS 结果验证（MainActivity.kt 第 565-579 行）

```kotlin
private suspend fun checkAuthAndCapture(orderID: String, context: Context?) {
    val authResp = // GET /api/orders/{orderId}/auth-check
    val liabilityShift = authResp.optString("liabilityShift", "")
    when (liabilityShift) {
        "NO"      -> throw Exception("3DS 验证失败")
        "UNKNOWN" -> throw Exception("3DS 不可用")
        else      -> captureOrder(orderID, context)  // POSSIBLE → 继续 capture
    }
}
```

---

## 六、模块三：Pay Later Messaging（PLM）

文件：`PaymentBottomSheet.kt`（第 315-342 行）

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
                            buyerCountry = "US",
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

PLM 组件会根据 `amount` + `buyerCountry` 自动显示分期信息文案。

---

## 七、模块四：Payment Buttons（官方品牌按钮）

文件：`PaymentBottomSheet.kt`（第 546-567 行）

```kotlin
// PayPal 按钮
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

---

## 八、Vault（保存支付方式）机制

### 8.1 保存 PayPal 账号

后端 `server.js`（第 221-237 行）— 创建订单时加入 vault 参数：

```json
"payment_source": {
  "paypal": {
    "attributes": {
      "customer": { "id": "DEMO-CUSTOMER-001" },
      "vault": { "store_in_vault": "ON_SUCCESS", "usage_type": "MERCHANT" }
    }
  }
}
```

后端 capture 后自动提取 vault token（`server.js` 第 353-372 行），保存到 `db.json`。

Android 端（`MainActivity.kt` 第 595-611 行）— capture 响应中提取并存入 SharedPreferences：

```kotlin
context.getSharedPreferences("red_prefs", MODE_PRIVATE).edit()
    .putString("paypal_vault_id", vaultId)
    .putString("paypal_vault_email", vaultEmail)
    .apply()
```

### 8.2 Vault 直接扣款（无需跳转 PayPal）

后端（`server.js` 第 90-152 行）：

```json
"payment_source": {
  "paypal": { "vault_id": "VAULT_TOKEN_ID" }
}
```

Android 端（`MainActivity.kt` 第 304-311 行）：

```kotlin
if (useVaultId != null) {
    val result = chargeVault(useVaultId, productPrice, shipping)
    // 直接完成，无需 Chrome Tab
}
```

---

## 九、完整数据流图

```text
用户点击 Buy Now
    │
    ▼
PaymentBottomSheet（选择支付方式）
    │
    ├── PayPal / Pay Later
    │       │
    │       ├── POST /api/orders/create（含 savePayPal/experience_context）
    │       ├── PayPalManager.startCheckout() → Chrome Custom Tab
    │       ├── 用户在 PayPal 页面授权
    │       ├── Deep Link 回调 → onNewIntent → PayPalManager.handleIntent()
    │       └── POST /api/orders/capture → 提取 vault token → SharedPreferences
    │
    ├── Card (ACDC)
    │       │
    │       ├── POST /api/orders/create（含 saveCard/card vault 参数）
    │       ├── CardManager.approveOrder() → PayPal SDK 处理卡数据
    │       ├── [如需 3DS] CardClient.presentAuthChallenge() → 3DS WebView
    │       ├── 3DS 回调 → onNewIntent → CardManager.handleIntent()
    │       ├── GET /api/orders/{id}/auth-check（验证 liabilityShift）
    │       └── POST /api/orders/capture → 提取 card vault token
    │
    └── Vault 直接扣款（已保存账号）
            │
            └── POST /api/orders/charge-vault（直接完成，无跳转）
```
