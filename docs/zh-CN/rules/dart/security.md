---
paths:
  - "**/*.dart"
  - "**/pubspec.yaml"
  - "**/AndroidManifest.xml"
  - "**/Info.plist"
---
# Dart/Flutter 安全（Dart/Flutter Security）

> 本文件在 [common/security.md](../common/security.md) 的基础上，补充 Dart、Flutter 及移动端专项安全内容。

## 密钥管理（Secrets Management）

- 永远不要在 Dart 源代码中硬编码 API key、token 或凭证
- 使用 `--dart-define` 或 `--dart-define-from-file` 进行编译时配置（这些值并非真正保密 — 服务端密钥应使用后端代理）
- 使用 `flutter_dotenv` 或同类工具，将 `.env` 文件列入 `.gitignore`
- 运行时密钥存储在平台安全存储中：`flutter_secure_storage`（iOS 的 Keychain，Android 的 EncryptedSharedPreferences）

```dart
// BAD
const apiKey = 'sk-abc123...';

// GOOD — 编译时配置（不保密，只是可配置）
const apiKey = String.fromEnvironment('API_KEY');

// GOOD — 从安全存储读取运行时密钥
final token = await secureStorage.read(key: 'auth_token');
```

## 网络安全（Network Security）

- 强制使用 HTTPS — 生产环境不允许 `http://` 调用
- 在 Android `network_security_config.xml` 中配置，阻止明文传输
- 在 `Info.plist` 中设置 `NSAppTransportSecurity`，禁止任意加载
- 所有 HTTP 客户端设置请求超时 — 不要使用默认值
- 对高安全性端点考虑证书固定（certificate pinning）

```dart
// Dio 配置超时和 HTTPS 强制
final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
  connectTimeout: const Duration(seconds: 10),
  receiveTimeout: const Duration(seconds: 30),
));
```

## 输入验证（Input Validation）

- 在发送到 API 或存储前，验证并净化所有用户输入
- 永远不要将未净化的输入传递给 SQL 查询 — 使用参数化查询（sqflite、drift）
- 在导航前净化深度链接（deep link）URL — 验证 scheme、host 和路径参数
- 使用 `Uri.tryParse` 并在导航前验证

```dart
// BAD — SQL 注入
await db.rawQuery("SELECT * FROM users WHERE email = '$userInput'");

// GOOD — 参数化查询
await db.query('users', where: 'email = ?', whereArgs: [userInput]);

// BAD — 未验证的深度链接
final uri = Uri.parse(incomingLink);
context.go(uri.path); // 可能导航到任意路由

// GOOD — 验证后的深度链接
final uri = Uri.tryParse(incomingLink);
if (uri != null && uri.host == 'myapp.com' && _allowedPaths.contains(uri.path)) {
  context.go(uri.path);
}
```

## 数据保护（Data Protection）

- token、PII 和凭证只存储在 `flutter_secure_storage`
- 永远不要将敏感数据明文写入 `SharedPreferences` 或本地文件
- 登出时清除认证状态：token、缓存的用户数据、cookie
- 对敏感操作使用生物认证（`local_auth`）
- 避免记录敏感数据 — 不要 `print(token)` 或 `debugPrint(password)`

## Android 专项

- 在 `AndroidManifest.xml` 中只声明必要的权限
- 仅在必要时导出 Android 组件（`Activity`、`Service`、`BroadcastReceiver`）；不需要导出的组件加 `android:exported="false"`
- 审查 intent filter — 带隐式 intent filter 的导出组件可被任意应用访问
- 对显示敏感数据的界面使用 `FLAG_SECURE`（防止截图）

```xml
<!-- AndroidManifest.xml — 限制导出组件 -->
<activity android:name=".MainActivity" android:exported="true">
    <!-- 只有启动 Activity 需要 exported=true -->
</activity>
<activity android:name=".SensitiveActivity" android:exported="false" />
```

## iOS 专项

- 在 `Info.plist` 中只声明必要的用途描述（`NSCameraUsageDescription` 等）
- 将密钥存储在 Keychain — `flutter_secure_storage` 在 iOS 上使用 Keychain
- 使用 App Transport Security（ATS）— 禁止任意加载
- 对敏感文件启用数据保护（data protection）权利

## WebView 安全

- 使用 `webview_flutter` v4+（`WebViewController` / `WebViewWidget`）— 旧版 `WebView` 组件已移除
- 除非明确需要，禁用 JavaScript（`JavaScriptMode.disabled`）
- 在加载前验证 URL — 永远不要从深度链接加载任意 URL
- 除非绝对必要且经过仔细沙箱隔离，不要向 JavaScript 暴露 Dart 回调
- 使用 `NavigationDelegate.onNavigationRequest` 拦截并验证导航请求

```dart
// webview_flutter v4+ API（WebViewController + WebViewWidget）
final controller = WebViewController()
  ..setJavaScriptMode(JavaScriptMode.disabled) // 除非需要，默认禁用
  ..setNavigationDelegate(
    NavigationDelegate(
      onNavigationRequest: (request) {
        final uri = Uri.tryParse(request.url);
        if (uri == null || uri.host != 'trusted.example.com') {
          return NavigationDecision.prevent;
        }
        return NavigationDecision.navigate;
      },
    ),
  );

// 在组件树中：
WebViewWidget(controller: controller)
```

## 混淆与构建安全（Obfuscation and Build Security）

- 在 release 构建中启用混淆：`flutter build apk --obfuscate --split-debug-info=./debug-info/`
- 将 `--split-debug-info` 的输出排除在版本控制之外（仅用于崩溃符号化）
- 确保 ProGuard/R8 规则不会意外暴露序列化类
- 在发布前运行 `flutter analyze` 并处理所有警告
