---
paths:
  - "**/*.dart"
  - "**/pubspec.yaml"
  - "**/analysis_options.yaml"
---
# Dart/Flutter 编码风格（Dart/Flutter Coding Style）

> 本文件在 [common/coding-style.md](../common/coding-style.md) 的基础上，补充 Dart 和 Flutter 专项内容。

## 格式化（Formatting）

- 所有 `.dart` 文件使用 **dart format** — 在 CI 中强制执行（`dart format --set-exit-if-changed .`）
- 行长：80 个字符（dart format 默认值）
- 多行参数/参数列表末尾加尾随逗号（trailing comma），以改善 diff 和格式化效果

## 不可变性（Immutability）

- 局部变量优先使用 `final`，编译时常量使用 `const`
- 所有字段为 `final` 的地方使用 `const` 构造函数
- 公共 API 返回不可修改的集合（`List.unmodifiable`、`Map.unmodifiable`）
- 不可变状态类中的状态变更使用 `copyWith()`

```dart
// BAD
var count = 0;
List<String> items = ['a', 'b'];

// GOOD
final count = 0;
const items = ['a', 'b'];
```

## 命名规范（Naming）

遵循 Dart 约定：
- 变量、参数和命名构造函数：`camelCase`
- 类、枚举、typedef 和扩展：`PascalCase`
- 文件名和库名：`snake_case`
- 顶层 `const` 常量：`SCREAMING_SNAKE_CASE`
- 私有成员以 `_` 开头
- 扩展（Extension）名称描述其扩展的类型：`StringExtensions`，而非 `MyHelpers`

## 空安全（Null Safety）

- 避免使用 `!`（bang 操作符）— 优先使用 `?.`、`??`、`if (x != null)` 或 Dart 3 模式匹配；仅在 null 值属于编程错误且崩溃是正确行为时才使用 `!`
- 避免使用 `late`，除非确保在首次使用前已初始化（优先使用 nullable 或构造函数初始化）
- 必须提供的构造函数参数使用 `required`

```dart
// BAD — 如果 user 为 null 会在运行时崩溃
final name = user!.name;

// GOOD — 空感知操作符
final name = user?.name ?? 'Unknown';

// GOOD — Dart 3 模式匹配（穷举式，编译器检查）
final name = switch (user) {
  User(:final name) => name,
  null => 'Unknown',
};

// GOOD — 提前返回空值守卫
String getUserName(User? user) {
  if (user == null) return 'Unknown';
  return user.name; // 守卫后提升为非空
}
```

## 密封类型与模式匹配（Sealed Types and Pattern Matching，Dart 3+）

使用密封类（sealed class）对封闭状态层次结构建模：

```dart
sealed class AsyncState<T> {
  const AsyncState();
}

final class Loading<T> extends AsyncState<T> {
  const Loading();
}

final class Success<T> extends AsyncState<T> {
  const Success(this.data);
  final T data;
}

final class Failure<T> extends AsyncState<T> {
  const Failure(this.error);
  final Object error;
}
```

对密封类型始终使用穷举 `switch`，不加 default/通配符：

```dart
// BAD
if (state is Loading) { ... }

// GOOD
return switch (state) {
  Loading() => const CircularProgressIndicator(),
  Success(:final data) => DataWidget(data),
  Failure(:final error) => ErrorWidget(error.toString()),
};
```

## 错误处理（Error Handling）

- 在 `on` 子句中指定异常类型 — 永远不要使用裸 `catch (e)`
- 永远不要捕获 `Error` 子类型 — 它们表示编程缺陷
- 对可恢复错误使用 `Result` 风格的类型或密封类
- 避免将异常用于控制流

```dart
// BAD
try {
  await fetchUser();
} catch (e) {
  log(e.toString());
}

// GOOD
try {
  await fetchUser();
} on NetworkException catch (e) {
  log('Network error: ${e.message}');
} on NotFoundException {
  handleNotFound();
}
```

## 异步 / Future（Async / Futures）

- 始终 `await` Future，或显式调用 `unawaited()` 标明有意的即发即忘（fire-and-forget）
- 如果函数从不 `await` 任何内容，不要将其标记为 `async`
- 使用 `Future.wait` / `Future.any` 处理并发操作
- 在任何 `await` 之后使用 `BuildContext` 前，检查 `context.mounted`（Flutter 3.7+）

```dart
// BAD — 忽略 Future
fetchData(); // 即发即忘但未标明意图

// GOOD
unawaited(fetchData()); // 显式即发即忘
await fetchData();      // 或正确地 await
```

## 导入（Imports）

- 全部使用 `package:` 导入 — 跨功能或跨层代码禁止使用相对导入（`../`）
- 排序：`dart:` → 外部 `package:` → 内部 `package:`（同一包）
- 无未使用的导入 — `dart analyze` 通过 `unused_import` 强制执行

## 代码生成（Code Generation）

- 生成文件（`.g.dart`、`.freezed.dart`、`.gr.dart`）必须统一提交或加入 `.gitignore`——每个项目选择一种策略
- 永远不要手动编辑生成的文件
- 生成器注解（`@JsonSerializable`、`@freezed`、`@riverpod` 等）只放在规范源文件上
