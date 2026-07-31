# Database Conventions

## Drift Table — Required Fields for Every Synced Entity

```dart
// features/X/data/local/X_table.dart
class TransactionsTable extends Table {
  // --- Identity ---
  TextColumn get id => text()();                    // UUID v4
  TextColumn get userId => text()();

  // --- Business fields ---
  IntColumn get amount => integer()();              // IDR integer, no decimals
  TextColumn get note => text().nullable()();
  TextColumn get categoryId => text().nullable()(); // nullable at capture
  TextColumn get accountId => text()();
  BoolColumn get isReviewed => boolean().withDefault(const Constant(false))();

  // --- Soft delete (REQUIRED — hard delete forbidden) ---
  BoolColumn get isDeleted => boolean().withDefault(const Constant(false))();

  // --- Sync metadata (REQUIRED on every synced entity) ---
  BoolColumn get isSynced => boolean().withDefault(const Constant(false))();
  DateTimeColumn get updatedAt => dateTime()();
  TextColumn get idempotencyKey => text()();        // UUID v4, generated client-side

  @override
  Set<Column> get primaryKey => {id};
}
```

## JSON / API Model Convention

```dart
// features/X/data/models/transaction_model.dart
@freezed
class TransactionModel with _$TransactionModel {
  const factory TransactionModel({
    required String id,
    required String userId,
    required String accountId,
    required int amount,
    String? note,
    String? categoryId,
    @JsonKey(name: 'is_reviewed') required bool isReviewed,
    @JsonKey(name: 'transaction_date') required String transactionDate,
    @JsonKey(name: 'created_at') required String createdAt,
  }) = _TransactionModel;

  factory TransactionModel.fromJson(Map<String, dynamic> json) =>
      _$TransactionModelFromJson(json);
}
```

## Soft Delete Rules

- **Hard delete is forbidden** for any data synced to the server
- Every synced entity must have `isDeleted: bool` (default `false`)
- When a user deletes a record → set `isDeleted = true`, `isSynced = false`
- The server receives soft deletes via the normal sync push flow
- Drift queries must always filter `WHERE is_deleted = false`

```dart
final transactions = await (db.select(db.transactionsTable)
    ..where((t) => t.isDeleted.equals(false)))
    .get();
```

## Sync Engine

### Core Rules
1. **All writes always go to Drift first** — never directly to the server
2. **Mark `isSynced = false`** on every local write
3. **Sync triggers** when the app returns to the foreground and has connectivity
4. **Sync also triggers** immediately after a successful online write
5. **One batch request** per sync cycle — never N requests for N records
6. **Idempotent** — each mutation carries an `idempotency_key`; backend rejects duplicates
7. **Server Wins** for conflict resolution (MVP default)

### Sync Flow

```
PUSH: SELECT isSynced=false → POST /sync/push (batch) → UPDATE isSynced=true
PULL: GET /sync/pull?since={lastSyncAt} → merge into Drift → UPDATE lastSyncAt
```

### SyncService Sketch

```dart
// core/sync/sync_service.dart
class SyncService {
  Future<void> fullSync() async {
    if (!await _isOnline()) return;
    await _push();
    await _pull();
  }

  Future<void> _push() async {
    final pending = await _localDb.getPendingSync();
    if (pending.isEmpty) return;
    await _api.batchPush(pending);
    await _localDb.markSynced(pending.map((e) => e.id).toList());
  }

  Future<void> _pull() async {
    final lastSync = await _storage.getLastSyncAt();
    final changes = await _api.pullSince(since: lastSync);
    for (final item in changes) {
      await _localDb.upsert(item); // server wins
    }
    await _storage.setLastSyncAt(DateTime.now().toUtc().toIso8601String());
  }
}
```

### Connectivity Trigger

```dart
ConnectivityPlus().onConnectivityChanged.listen((result) {
  if (result != ConnectivityResult.none) {
    ref.read(syncServiceProvider).fullSync();
  }
});
```
