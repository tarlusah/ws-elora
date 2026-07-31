# Coding Style & Conventions

## Feature Architecture — Repository → Provider → Screen

```
Repository  →  Provider (Notifier)  →  Screen (ConsumerWidget)
```

- **Repository**: handles all API calls and local DB access. Returns models or throws `AppException`.
- **Provider**: `Notifier` or `AsyncNotifier` holding feature state as a `@freezed` class. Calls the repository only.
- **Screen**: `ConsumerWidget` that watches providers and renders UI. Zero business logic.

```dart
// CORRECT
final state = ref.watch(captureProvider);

// WRONG — never call Dio or Drift directly from a screen or provider
final response = await dio.post('/transactions', data: {...});
```

## Provider Pattern

```dart
@riverpod
class CaptureNotifier extends _$CaptureNotifier {
  @override
  CaptureState build() => CaptureState.initial();

  Future<void> save(CreateTransactionRequest request) async {
    state = state.copyWith(isLoading: true, error: null);
    try {
      final result = await ref.read(transactionRepositoryProvider).create(request);
      state = state.copyWith(isLoading: false, lastSaved: result);
    } on AppException catch (e) {
      state = state.copyWith(isLoading: false, error: e);
    }
  }
}
```

## State Model Convention

Use `@freezed` for ALL state and data models:

```dart
@freezed
class CaptureState with _$CaptureState {
  const factory CaptureState({
    @Default('') String amount,
    String? selectedAccountId,
    String? selectedCategoryId,
    @Default(false) bool isLoading,
    AppException? error,
  }) = _CaptureState;

  factory CaptureState.initial() => const CaptureState();
}
```

## Error Handling

All repository methods must throw `AppException` on failure. Providers catch `AppException` and store it in state. Never let `DioException` or other raw exceptions leak into the provider layer.

```dart
// core/errors/app_exception.dart
class AppException implements Exception {
  final String code;       // e.g. 'VALIDATION_ERROR', 'UNAUTHORIZED'
  final String message;
  final int statusCode;
  final Map<String, dynamic>? details;
}
```

Display errors via `AppSnackbar.showError(context, e.message)`.

## Naming Conventions

| Type | Convention | Example |
|---|---|---|
| File | `snake_case.dart` | `capture_provider.dart` |
| Class | `PascalCase` | `CaptureNotifier` |
| Variable / method | `camelCase` | `isSynced`, `saveTransaction()` |
| Constant | `camelCase` with `const` | `const maxBatchSize = 500` |
| Freezed model | Entity name + `Model` | `TransactionModel` |
| Provider | Feature name + `Provider` | `captureProvider` |
| Notifier | Feature name + `Notifier` | `CaptureNotifier` |
| Repository | Feature name + `Repository` | `TransactionRepository` |
| Screen | Feature name + `Screen` | `CaptureScreen` |

## Forbidden Practices

- ❌ Calling Dio directly from a widget or provider — always go through a Repository
- ❌ Hard deleting synced data — always soft delete
- ❌ Storing JWT or secrets in `SharedPreferences` — use `flutter_secure_storage`
- ❌ Using `StatefulWidget` when a `ConsumerWidget` + provider is sufficient
- ❌ Using `Navigator.push()` — always use `context.go()` or `context.push()`
- ❌ Making N API requests for N records — always batch
- ❌ Running sync when offline — always check `connectivity_plus` first
- ❌ Storing raw API response state in a provider — re-read from local DB instead
- ❌ Calling Drift outside of repository classes
- ❌ Letting `DioException` escape the repository — map it to `AppException` before throwing
