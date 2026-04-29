---
name: dart-flutter-patterns
description: 生产就绪的 Dart 和 Flutter 模式，涵盖空安全、不可变状态、异步组合、Widget 架构、主流状态管理框架（BLoC、Riverpod、Provider）、GoRouter 导航、Dio 网络请求、Freezed 代码生成及简洁架构。
origin: ECC
---

# Dart/Flutter 模式

## 何时使用

在以下情况使用本技能：
- 开始新的 Flutter 功能，需要状态管理、导航或数据访问的惯用模式
- 审查或编写 Dart 代码，需要关于空安全、密封类型或异步组合的指导
- 搭建新的 Flutter 项目，在 BLoC、Riverpod 或 Provider 之间做选择
- 实现安全的 HTTP 客户端、WebView 集成或本地存储
- 为 Flutter Widget、Cubit 或 Riverpod 提供者编写测试
- 使用 GoRouter 配合认证守卫进行路由

## 工作原理

本技能提供按关注点组织的即用型 Dart/Flutter 代码模式：
1. **空安全（Null safety）** — 避免 `!`，优先使用 `?.`/`??`/模式匹配
2. **不可变状态** — 密封类、`freezed`、`copyWith`
3. **异步组合** — 并发 `Future.wait`，`await` 后安全使用 `BuildContext`
4. **Widget 架构** — 提取为类（而非方法）、`const` 传播、局部重建
5. **状态管理** — BLoC/Cubit 事件、Riverpod notifier 和派生提供者
6. **导航** — GoRouter 配合通过 `refreshListenable` 实现的响应式认证守卫
7. **网络请求** — Dio 配合拦截器、一次性重试守卫的 token 刷新
8. **错误处理** — 全局捕获、`ErrorWidget.builder`、Crashlytics 接入
9. **测试** — 单元测试（BLoC test）、Widget 测试（ProviderScope 覆盖）、伪造（fake）优于 mock

## 示例

```dart
// 密封状态 — 防止不可能的状态
sealed class AsyncState<T> {}
final class Loading<T> extends AsyncState<T> {}
final class Success<T> extends AsyncState<T> { final T data; const Success(this.data); }
final class Failure<T> extends AsyncState<T> { final Object error; const Failure(this.error); }

// GoRouter 配合响应式认证重定向
final router = GoRouter(
  refreshListenable: GoRouterRefreshStream(authCubit.stream),
  redirect: (context, state) {
    final authed = context.read<AuthCubit>().state is AuthAuthenticated;
    if (!authed && !state.matchedLocation.startsWith('/login')) return '/login';
    return null;
  },
  routes: [...],
);

// Riverpod 派生提供者，使用安全的 firstWhereOrNull
@riverpod
double cartTotal(Ref ref) {
  final cart = ref.watch(cartNotifierProvider);
  final products = ref.watch(productsProvider).valueOrNull ?? [];
  return cart.fold(0.0, (total, item) {
    final product = products.firstWhereOrNull((p) => p.id == item.productId);
    return total + (product?.price ?? 0) * item.quantity;
  });
}
```

---

适用于 Dart 和 Flutter 应用程序的实用、生产就绪模式。尽可能与库无关，同时明确覆盖最常见的生态系统包。

---

## 1. 空安全基础

### 优先使用模式匹配而非叹号运算符

```dart
// 差 — 为 null 时运行时崩溃
final name = user!.name;

// 好 — 提供回退值
final name = user?.name ?? 'Unknown';

// 好 — Dart 3 模式匹配（复杂情况下优先使用）
final display = switch (user) {
  User(:final name, :final email) => '$name <$email>',
  null => 'Guest',
};

// 好 — 提前守卫返回
String getUserName(User? user) {
  if (user == null) return 'Unknown';
  return user.name; // 检查后提升为非空
}
```

### 避免过度使用 `late`

```dart
// 差 — 将空值错误推迟到运行时
late String userId;

// 好 — 可空并显式初始化
String? userId;

// 可接受 — 仅在首次访问前保证初始化时使用 late
// （例如，在 initState() 中，在任何 widget 交互之前）
late final AnimationController _controller;

@override
void initState() {
  super.initState();
  _controller = AnimationController(vsync: this, duration: const Duration(milliseconds: 300));
}
```

---

## 2. 不可变状态

### 用于状态层次结构的密封类（Sealed Classes）

```dart
sealed class UserState {}

final class UserInitial extends UserState {}

final class UserLoading extends UserState {}

final class UserLoaded extends UserState {
  const UserLoaded(this.user);
  final User user;
}

final class UserError extends UserState {
  const UserError(this.message);
  final String message;
}

// 穷举式 switch — 编译器强制所有分支
Widget buildFrom(UserState state) => switch (state) {
  UserInitial() => const SizedBox.shrink(),
  UserLoading() => const CircularProgressIndicator(),
  UserLoaded(:final user) => UserCard(user: user),
  UserError(:final message) => ErrorText(message),
};
```

### 使用 Freezed 实现零样板代码不可变性

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    @Default(false) bool isAdmin,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}

// 使用
final user = User(id: '1', name: 'Alice', email: 'alice@example.com');
final updated = user.copyWith(name: 'Alice Smith'); // 不可变更新
final json = user.toJson();
final fromJson = User.fromJson(json);
```

---

## 3. 异步组合

### 使用 Future.wait 实现结构化并发

```dart
Future<DashboardData> loadDashboard(UserRepository users, OrderRepository orders) async {
  // 并发运行 — 不要顺序 await
  final (userList, orderList) = await (
    users.getAll(),
    orders.getRecent(),
  ).wait; // Dart 3 记录解构 + Future.wait 扩展

  return DashboardData(users: userList, orders: orderList);
}
```

### Stream 模式

```dart
// 仓储层暴露响应式流用于实时数据
Stream<List<Item>> watchCartItems() => _db
    .watchTable('cart_items')
    .map((rows) => rows.map(Item.fromRow).toList());

// 在 widget 层 — 声明式，无需手动订阅
StreamBuilder<List<Item>>(
  stream: cartRepository.watchCartItems(),
  builder: (context, snapshot) => switch (snapshot) {
    AsyncSnapshot(connectionState: ConnectionState.waiting) =>
        const CircularProgressIndicator(),
    AsyncSnapshot(:final error?) => ErrorWidget(error.toString()),
    AsyncSnapshot(:final data?) => CartList(items: data),
    _ => const SizedBox.shrink(),
  },
)
```

### await 之后使用 BuildContext

```dart
// 重要 — 在 StatefulWidget 中任何 await 之后始终检查 mounted
Future<void> _handleSubmit() async {
  setState(() => _isLoading = true);
  try {
    await authService.login(_email, _password);
    if (!mounted) return; // ← 使用 context 前的守卫
    context.go('/home');
  } on AuthException catch (e) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(e.message)));
  } finally {
    if (mounted) setState(() => _isLoading = false);
  }
}
```

---

## 4. Widget 架构

### 提取为类而非方法

```dart
// 差 — 返回 widget 的私有方法，阻止优化
Widget _buildHeader() {
  return Container(
    padding: const EdgeInsets.all(16),
    child: Text(title, style: Theme.of(context).textTheme.headlineMedium),
  );
}

// 好 — 独立的 widget 类，支持 const 和元素复用
class _PageHeader extends StatelessWidget {
  const _PageHeader(this.title);
  final String title;

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      child: Text(title, style: Theme.of(context).textTh