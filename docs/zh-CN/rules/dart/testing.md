---
paths:
  - "**/*.dart"
  - "**/pubspec.yaml"
  - "**/analysis_options.yaml"
---
# Dart/Flutter 测试（Dart/Flutter Testing）

> 本文件在 [common/testing.md](../common/testing.md) 的基础上，补充 Dart 和 Flutter 专项测试内容。

## 测试框架（Test Framework）

- **flutter_test** / **dart:test** — 内置测试运行器
- **mockito**（配合 `@GenerateMocks`）或 **mocktail**（无需代码生成）用于 Mock
- **bloc_test** 用于 BLoC/Cubit 单元测试
- **fake_async** 用于在单元测试中控制时间
- **integration_test** 用于端到端设备测试

## 测试类型

| 类型 | 工具 | 位置 | 编写时机 |
|------|------|----------|---------------|
| 单元测试（Unit） | `dart:test` | `test/unit/` | 所有领域逻辑、状态管理器、仓库 |
| 组件测试（Widget） | `flutter_test` | `test/widget/` | 所有具有有意义行为的组件 |
| 黄金测试（Golden） | `flutter_test` | `test/golden/` | 设计关键型 UI 组件 |
| 集成测试（Integration） | `integration_test` | `integration_test/` | 在真机/模拟器上的关键用户流程 |

## 单元测试：状态管理器

### BLoC 配合 `bloc_test`

```dart
group('CartBloc', () {
  late CartBloc bloc;
  late MockCartRepository repository;

  setUp(() {
    repository = MockCartRepository();
    bloc = CartBloc(repository);
  });

  tearDown(() => bloc.close());

  blocTest<CartBloc, CartState>(
    'emits updated items when CartItemAdded',
    build: () => bloc,
    act: (b) => b.add(CartItemAdded(testItem)),
    expect: () => [CartState(items: [testItem])],
  );

  blocTest<CartBloc, CartState>(
    'emits empty cart when CartCleared',
    seed: () => CartState(items: [testItem]),
    build: () => bloc,
    act: (b) => b.add(CartCleared()),
    expect: () => [const CartState()],
  );
});
```

### Riverpod 配合 `ProviderContainer`

```dart
test('usersProvider loads users from repository', () async {
  final container = ProviderContainer(
    overrides: [userRepositoryProvider.overrideWithValue(FakeUserRepository())],
  );
  addTearDown(container.dispose);

  final result = await container.read(usersProvider.future);
  expect(result, isNotEmpty);
});
```

## 组件测试（Widget Tests）

```dart
testWidgets('CartPage shows item count badge', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        cartNotifierProvider.overrideWith(() => FakeCartNotifier([testItem])),
      ],
      child: const MaterialApp(home: CartPage()),
    ),
  );

  await tester.pump();
  expect(find.text('1'), findsOneWidget);
  expect(find.byType(CartItemTile), findsOneWidget);
});

testWidgets('shows empty state when cart is empty', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [cartNotifierProvider.overrideWith(() => FakeCartNotifier([]))],
      child: const MaterialApp(home: CartPage()),
    ),
  );

  await tester.pump();
  expect(find.text('Your cart is empty'), findsOneWidget);
});
```

## 优先使用 Fake 而非 Mock（Fakes Over Mocks）

对于复杂依赖，优先手写 fake：

```dart
class FakeUserRepository implements UserRepository {
  final _users = <String, User>{};
  Object? fetchError;

  @override
  Future<User?> getById(String id) async {
    if (fetchError != null) throw fetchError!;
    return _users[id];
  }

  @override
  Future<List<User>> getAll() async {
    if (fetchError != null) throw fetchError!;
    return _users.values.toList();
  }

  @override
  Stream<List<User>> watchAll() => Stream.value(_users.values.toList());

  @override
  Future<void> save(User user) async {
    _users[user.id] = user;
  }

  @override
  Future<void> delete(String id) async {
    _users.remove(id);
  }

  void addUser(User user) => _users[user.id] = user;
}
```

## 异步测试（Async Testing）

```dart
// 使用 fake_async 控制定时器和 Future
test('debounce triggers after 300ms', () {
  fakeAsync((async) {
    final debouncer = Debouncer(delay: const Duration(milliseconds: 300));
    var callCount = 0;
    debouncer.run(() => callCount++);
    expect(callCount, 0);
    async.elapse(const Duration(milliseconds: 200));
    expect(callCount, 0);
    async.elapse(const Duration(milliseconds: 200));
    expect(callCount, 1);
  });
});
```

## 黄金测试（Golden Tests）

```dart
testWidgets('UserCard golden test', (tester) async {
  await tester.pumpWidget(
    MaterialApp(home: UserCard(user: testUser)),
  );

  await expectLater(
    find.byType(UserCard),
    matchesGoldenFile('goldens/user_card.png'),
  );
});
```

当有意进行视觉变更时，运行 `flutter test --update-goldens` 更新基准图。

## 测试命名规范

使用描述性、以行为为中心的命名：

```dart
test('returns null when user does not exist', () { ... });
test('throws NotFoundException when id is empty string', () { ... });
testWidgets('disables submit button while form is invalid', (tester) async { ... });
```

## 测试组织结构

```
test/
├── unit/
│   ├── domain/
│   │   └── usecases/
│   └── data/
│       └── repositories/
├── widget/
│   └── presentation/
│       └── pages/
└── golden/
    └── widgets/

integration_test/
└── flows/
    ├── login_flow_test.dart
    └── checkout_flow_test.dart
```

## 覆盖率（Coverage）

- 业务逻辑（领域层 + 状态管理器）目标：80% 以上行覆盖率
- 所有状态转换必须有测试：loading → success、loading → error、重试
- 运行 `flutter test --coverage`，用覆盖率工具查看 `lcov.info`
- 覆盖率低于阈值时，CI 应阻断构建
