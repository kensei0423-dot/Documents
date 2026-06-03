# Red Pay Demo — 项目概览（iOS）

这是一个 PayPal iOS 支付 Demo，模拟类似小红书/电商 App 的购物支付体验，与 Android 版本共用同一套 Node.js 后端。

---

## 📁 项目结构

| 部分 | 说明 |
|------|------|
| `server.js` | Node.js 后端（与 Android 共用），运行在 port 3002 |
| `RedIOS/` | iOS 原生应用（Swift + SwiftUI） |
| `db.json` | 本地数据库（配置 + vault tokens，与 Android 共享） |
| `.env` | 环境变量（PayPal 账号凭证） |
| `paypal-ios-main/` | PayPal iOS SDK 源码参考 |
| `paypal-messages-ios-develop/` | PayPal Messages（PLM）SDK 源码参考 |

---

## 🖥️ 后端 API（server.js）

| 接口 | 功能 |
|------|------|
| `GET /api/config` | 获取配置（useVault、force3ds、currency、**clientId** 等） |
| `POST /api/orders/create` | 创建 PayPal 订单 |
| `POST /api/orders/capture` | 捕获订单，自动保存 vault token |
| `POST /api/orders/charge-vault` | Vault 直接扣款（无需跳转 PayPal） |
| `GET/POST/DELETE /api/vault-tokens` | 管理已保存的支付方式 |
| `GET /admin` | 管理后台页面 |

---

## 📱 iOS 应用（Swift + SwiftUI）

支持 3 种支付方式：

- **PayPal** — `ASWebAuthenticationSession` 内嵌浏览器，支持 Vault 保存
- **Pay Later (BNPL)** — 分期付款，集成 PLM 消息组件
- **Card (ACDC)** — 原生卡号输入，支持 3DS 验证 + Vault 保存

核心文件：

| 文件 | 说明 |
|------|------|
| `ContentView.swift` | 商品页面 + 支付流程控制 |
| `PaymentSheetView.swift` | 支付底部弹窗 UI |
| `PayPalManager.swift` | PayPal Web Checkout 管理 |
| `CardManager.swift` | ACDC 卡支付 + 3DS 管理 |
| `NetworkClient.swift` | URLSession 网络层 |
| `PLMView.swift` | Pay Later Messaging 组件 |

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

## 🔧 SDK 版本（SPM）

| 包 | 版本 |
|----|------|
| `paypal-ios` → `PayPalWebPayments` | 2.x |
| `paypal-ios` → `CardPayments` | 2.x |
| `paypal-ios` → `PaymentButtons` | 2.x |
| `paypal-messages-ios` → `PayPalMessages` | 1.2.0 |

---

## 🚀 启动方式

- **后端**：`cd demo/red && node server.js`（port 3002）
- **iOS**：用 Xcode 打开 `demo/red-ios/RedIOS.xcodeproj`，选择模拟器运行
- **管理后台**：浏览器访问 `http://localhost:3002/admin`

---

# PayPal iOS SDK 在 Red 项目中的完整使用说明

---

## 一、SDK 依赖配置

文件：`project.yml`（xcodegen）或 Xcode → Add Package Dependencies

```yaml
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
        product: CardPayments        # ACDC 原生卡支付（含 3DS）
      - package: PayPal
        product: PaymentButtons      # 官方品牌按钮
      - package: PayPalMessages
        product: PayPalMessages      # Pay Later Messaging 消息组件
```

---

## 二、Info.plist 配置

文件：`RedIOS/Info.plist`（xcodegen 自动注入）

```xml
<!-- 允许本地 HTTP（连接本地 server） -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsLocalNetworking</key>
    <true/>
</dict>
```

> ⚠️ **无需注册 URL Scheme。** iOS SDK 使用 `ASWebAuthenticationSession`，callback scheme `sdk.ios.paypal://` 由 SDK 内部管理，不经过 App 的 URL 路由系统。这与 Android 需要配置 `intent-filter` 有根本区别。

---

## 三、全局常量与动态 clientId

文件：`Constants.swift`

```swift
let PAYPAL_CLIENT_ID = "AUbSpUcLCXCQMo5Tqqz4sv4d56ALQxpYua7eaPHNChuy..."  // 初始 fallback
let SERVER_URL = "http://localhost:3002"   // 模拟器访问宿主机 localhost
```

> ⚠️ **关键**：iOS App 启动后从 `/api/config` 获取服务端当前激活账号的 `clientId`，并重新初始化 SDK。这确保 SDK 与服务端使用同一个 PayPal 账号，否则会出现 `NOT_ENABLED_TO_VAULT_PAYMENT_SOURCE` 错误。

```swift
// ContentView.swift — loadConfig() 中
if let clientId = config.clientId, !clientId.isEmpty {
    PayPalManager.configure(clientID: clientId)
    CardManager.configure(clientID: clientId)
}
```

---

## 四、模块一：PayPal Web Checkout（ASWebAuthenticationSession）

文件：`PayPalManager.swift`

### 4.1 初始化（支持动态 clientId）

```swift
class PayPalManager {
    static var shared = PayPalManager(clientID: PAYPAL_CLIENT_ID)

    // 服务端切换账号后重新初始化
    static func configure(clientID: String) {
        shared = PayPalManager(clientID: clientID)
    }

    private init(clientID: String) {
        let config = CoreConfig(clientID: clientID, environment: .sandbox)
        client = PayPalWebCheckoutClient(config: config)
    }
}
```

### 4.2 发起支付（async/await，无需回调）

```swift
// 构建请求（orderId + 资金来源）
let request = PayPalWebCheckoutRequest(
    orderID: orderID,
    fundingSource: .paypal    // 或 .paylater
)

// 启动 ASWebAuthenticationSession（SDK 内部管理，App 无需处理 URL）
let result = try await client.start(request: request)
// result.orderID → 调用 POST /api/orders/capture
// result.payerID → PayPal 买家标识（可选）
```

### 4.3 无需 Intent 路由

Android 必须在 `onNewIntent` 和 `onResume` 中手动处理回调：

```kotlin
// Android — 必须实现
override fun onNewIntent(intent: Intent) { payPalManager.handleIntent(intent) }
override fun onResume() { payPalManager.handleIntent(intent) }
```

**iOS 完全不需要**。`ASWebAuthenticationSession` 拦截 `sdk.ios.paypal://` 回调后，直接触发 `async throws` 的返回值，无需任何 `handleIntent` 逻辑。

### 4.4 支付完整流程

文件：`ContentView.swift`

```swift
// 1. 后端创建订单
let orderID = try await NetworkClient.shared.createOrder(
    amount: 120.0, payLater: false, savePayPal: true, isCard: false, shipping: shipping
)

// 2. 启动 ASWebAuthenticationSession
let result = try await PayPalManager.shared.startCheckout(
    orderID: orderID,
    fundingSource: .paypal
)

// 3. 后端 capture 订单
let capture = try await NetworkClient.shared.captureOrder(orderID: result.orderID)

// 4. 从 capture 响应中提取 vault token 并保存到 UserDefaults
if let vaultId = capture.vaultId {
    UserDefaults.standard.set(vaultId, forKey: "paypal_vault_id")
}
```

---

## 五、模块二：Card Payments（ACDC 原生卡支付）

文件：`CardManager.swift`

### 5.1 初始化

```swift
class CardManager {
    static var shared = CardManager(clientID: PAYPAL_CLIENT_ID)

    static func configure(clientID: String) {
        shared = CardManager(clientID: clientID)
    }

    private init(clientID: String) {
        let config = CoreConfig(clientID: clientID, environment: .sandbox)
        cardClient = CardClient(config: config)
    }
}
```

### 5.2 发起卡支付（3DS 自动处理）

```swift
// 构建 Card 对象
let card = Card(
    number: "4111111111111111",
    expirationMonth: "01",
    expirationYear: "2027",
    securityCode: "123"
)

// 设置 3DS 策略（SCA_ALWAYS 或 SCA_WHEN_REQUIRED）
let request = CardRequest(
    orderID: orderID,
    card: card,
    sca: force3ds ? .scaAlways : .scaWhenRequired
)

// 发起支付（async/await，3DS 由 SDK 内部自动处理）
let result = try await cardClient.approveOrder(request: request)
// result.didAttemptThreeDSecureAuthentication → 是否触发了 3DS
```

> ⚠️ **iOS 与 Android 最大区别**：Android 需要两步（`approveOrder` → `AuthorizationRequired` → `presentAuthChallenge` → `onResume` → `finishApproveOrder`）。iOS 只需一步，SDK 内部自动检测是否需要 3DS 并通过 `ASWebAuthenticationSession` 完成验证，`didAttemptThreeDSecureAuthentication` 标识是否发生了 3DS。

### 5.3 无需手动处理 3DS 回调

Android 必须在 `onNewIntent` 中手动调用 `finishApproveOrder(intent)`：

```kotlin
// Android — 必须实现
fun handleIntent(activity: Activity, intent: Intent) {
    val result = cardClient.finishApproveOrder(intent)
    when (result) {
        is CardFinishApproveOrderResult.Success -> onSuccess(result.orderId)
        // ...
    }
}
```

**iOS 完全不需要**。`approveOrder` 是单一 async 调用，3DS 完成后直接返回结果。

### 5.4 卡支付完整流程

文件：`ContentView.swift`

```swift
// 1. 后端创建订单（含 saveCard 参数）
let orderID = try await NetworkClient.shared.createOrder(
    amount: 120.0, isCard: true, saveCard: true
)

// 2. 发起 ACDC 支付（3DS 自动处理）
let result = try await CardManager.shared.approveOrder(
    orderID: orderID,
    card: card,
    force3ds: force3ds
)

// 3. 如果触发了 3DS，先验证 liabilityShift
if result.didAttemptThreeDSecureAuthentication {
    let auth = try await NetworkClient.shared.authCheck(orderID: orderID)
    if auth.liabilityShift == "NO" { throw NSError(...) }     // 3DS 验证失败
    if auth.liabilityShift == "UNKNOWN" { throw NSError(...) } // 3DS 不可用
}

// 4. 后端 capture，自动保存 card vault token
let capture = try await NetworkClient.shared.captureOrder(orderID: orderID)
```

### 5.5 3DS 结果验证

文件：`NetworkClient.swift` + `ContentView.swift`

```swift
// GET /api/orders/{orderId}/auth-check
let auth = try await NetworkClient.shared.authCheck(orderID: orderID)

switch auth.liabilityShift {
case "NO":      throw NSError(...)   // 3DS 验证失败
case "UNKNOWN": throw NSError(...)   // 3DS 不可用
default:        break                // POSSIBLE 或 nil → 继续 capture
}
```

---

## 六、模块三：Pay Later Messaging（PLM）

文件：`PLMView.swift`

> PLM 独立为单独文件，原因：`PaymentButtons` 模块间接引入 `CorePayments.Environment` 类型，与 `PayPalMessages` 的 `Environment` 产生命名冲突。独立文件只 import `PayPalMessages`，避免歧义。

```swift
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

> PLM 组件会根据 `amount + buyerCountry` 自动显示分期信息文案。
>
> ⚠️ PLM 必须放在 `PaymentOptionRow` 外部，否则 SwiftUI 手势系统会拦截 "Learn more" 点击事件。

---

## 七、模块四：Payment Buttons（官方品牌按钮）

文件：`PaymentSheetView.swift`

iOS SDK 提供 `UIViewRepresentable` 包装，可直接在 SwiftUI 中使用：

```swift
// PayPal 金色按钮
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

> 按钮本身只负责渲染品牌 UI，点击事件由 trailing closure 处理，不直接触发支付逻辑。

---

## 八、Vault（保存支付方式）机制

### 8.1 保存 PayPal 账号

**后端 `server.js`** — 创建订单时加入 vault 参数：

```json
"payment_source": {
  "paypal": {
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
```

后端 capture 后自动提取 vault token，保存到 `db.json`。

**iOS 端（`ContentView.swift`）** — capture 响应中提取并存入 `UserDefaults`：

```swift
if capture.vaultType == "paypal" {
    UserDefaults.standard.set(capture.vaultId, forKey: "paypal_vault_id")
    UserDefaults.standard.set(capture.vaultEmail, forKey: "paypal_vault_email")
}
```

### 8.2 Vault 直接扣款（无需跳转 PayPal）

**后端（`server.js`）**：

```json
"payment_source": {
  "paypal": { "vault_id": "VAULT_TOKEN_ID" }
}
```

**iOS 端（`ContentView.swift`）**：

```swift
if let vaultId = useVaultId {
    let (orderId, newVaultId) = try await NetworkClient.shared.chargeVault(
        vaultId: vaultId, amount: productPrice, shipping: shipping
    )
    // 直接完成，无需 ASWebAuthenticationSession
    if let newId = newVaultId { savedVaultId = newId }
    finish(orderId: orderId)
}
```

### 8.3 启动时从服务端同步 Vault 状态

```swift
// ContentView.swift — loadConfig() 中
let tokens = try await NetworkClient.shared.getVaultTokens()
if tokens.isEmpty {
    // 清除 UserDefaults 所有 vault 数据
} else {
    // 直接用服务端数据填充（server 是 source of truth）
    for token in tokens {
        if token.type == "paypal" {
            savedVaultId    = token.vaultId
            savedVaultEmail = token.email
        } else if token.type == "card" {
            savedCardVaultId = token.vaultId
            savedCardLast4   = token.cardLast4
            savedCardBrand   = token.cardBrand
        }
    }
}
```

> iOS 和 Android 共用同一个服务端，Vault token 可以跨平台共用：iOS 保存的 token，Android 启动时也能加载到，反之亦然。

---

## 九、完整数据流图

```text
用户点击 Buy Now
    │
    ▼
PaymentSheetView（选择支付方式）
    │
    ├── PayPal / Pay Later
    │       │
    │       ├── POST /api/orders/create（含 savePayPal/experience_context）
    │       ├── PayPalManager.startCheckout() → ASWebAuthenticationSession
    │       ├── 用户在内嵌浏览器完成 PayPal 登录授权
    │       ├── SDK 内部拦截 sdk.ios.paypal:// 回调
    │       ├── async result 直接返回（无需 handleIntent）
    │       └── POST /api/orders/capture → 提取 vault token → UserDefaults
    │
    ├── Card (ACDC)
    │       │
    │       ├── POST /api/orders/create（含 saveCard/card vault 参数）
    │       ├── CardManager.approveOrder() → SDK 调用 confirmPaymentSource
    │       ├── [如需 3DS] SDK 内部自动打开 ASWebAuthenticationSession
    │       ├── 3DS 完成后 async result 直接返回（无需 handleIntent）
    │       ├── GET /api/orders/{id}/auth-check（验证 liabilityShift）
    │       └── POST /api/orders/capture → 提取 card vault token → UserDefaults
    │
    └── Vault 直接扣款（已保存账号）
            │
            └── POST /api/orders/charge-vault（直接完成，无跳转，无 capture）
```

---

## 附录：iOS vs Android 关键差异速查

| 方面 | Android | iOS |
|------|---------|-----|
| PayPal 结账方式 | Chrome Custom Tab | `ASWebAuthenticationSession` |
| URL Scheme 注册 | ✅ 必须（AndroidManifest） | ❌ 不需要 |
| Intent 路由 | `onNewIntent` + `handleIntent` | 无，async/await 直接返回 |
| 3DS 流程 | 两步手动处理 | SDK 内部自动完成 |
| 并发模型 | Kotlin Coroutines | Swift async/await |
| 本地存储 | `SharedPreferences` | `UserDefaults` |
| clientId 来源 | 硬编码常量 | 启动后从 `/api/config` 动态更新 |
| 服务端地址（模拟器） | `http://10.0.2.2:3002` | `http://localhost:3002` |
| Vault 数据源 | 服务端同步（直接填充） | 服务端同步（直接填充） |
