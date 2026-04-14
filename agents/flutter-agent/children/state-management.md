# State Management (Riverpod)

All async data flows through Riverpod. No raw `setState` for API data. `setState` is only acceptable for local UI state (animation toggles, form field visibility).

---

## Provider Types

### 1. Provider (Synchronous, Read-Only)

For computed values, configuration, and dependency injection.

```dart
// Dependency injection: expose a repository instance
final walkRepositoryProvider = Provider<WalkRepository>((ref) {
  final dio = ref.watch(dioProvider);
  return WalkRepositoryImpl(dio);
});

// Computed value from another provider
final completedWalksCountProvider = Provider<int>((ref) {
  final walks = ref.watch(walksProvider).valueOrNull ?? [];
  return walks.where((w) => w.isCompleted).length;
});
```

### 2. FutureProvider (Async, Read-Only)

For one-time async data fetches. The most common provider for API data.

```dart
// Simple fetch
final walksProvider = FutureProvider<List<Walk>>((ref) async {
  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalks();
});

// Auto-dispose: free resources when no widget is listening
final walksProvider = FutureProvider.autoDispose<List<Walk>>((ref) async {
  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalks();
});
```

### 3. FutureProvider.family (Parameterized Async)

For fetching data that depends on a parameter (ID, filter, page).

```dart
// Single parameter
final walkDetailProvider = FutureProvider.family<Walk, String>((ref, walkId) async {
  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalkById(walkId);
});

// Usage in widget
final walkAsync = ref.watch(walkDetailProvider(walkId));

// Multiple parameters: use a record or custom class
final walksByFilterProvider = FutureProvider.family<List<Walk>, ({String status, int page})>(
  (ref, params) async {
    final repository = ref.watch(walkRepositoryProvider);
    return repository.getWalks(status: params.status, page: params.page);
  },
);

// Usage
final walks = ref.watch(walksByFilterProvider((status: 'active', page: 1)));
```

### 4. StateNotifierProvider (Mutable State)

For complex local state that changes over time. Forms, filters, multi-step flows.

```dart
// State class (immutable)
@immutable
class WalkFilterState {
  const WalkFilterState({
    this.status = WalkStatus.all,
    this.dateRange,
    this.searchQuery = '',
    this.sortBy = WalkSortBy.newest,
  });

  final WalkStatus status;
  final DateTimeRange? dateRange;
  final String searchQuery;
  final WalkSortBy sortBy;

  WalkFilterState copyWith({
    WalkStatus? status,
    DateTimeRange? dateRange,
    String? searchQuery,
    WalkSortBy? sortBy,
  }) {
    return WalkFilterState(
      status: status ?? this.status,
      dateRange: dateRange ?? this.dateRange,
      searchQuery: searchQuery ?? this.searchQuery,
      sortBy: sortBy ?? this.sortBy,
    );
  }
}

// Notifier
class WalkFilterNotifier extends StateNotifier<WalkFilterState> {
  WalkFilterNotifier() : super(const WalkFilterState());

  void setStatus(WalkStatus status) {
    state = state.copyWith(status: status);
  }

  void setSearchQuery(String query) {
    state = state.copyWith(searchQuery: query);
  }

  void setDateRange(DateTimeRange? range) {
    state = state.copyWith(dateRange: range);
  }

  void setSortBy(WalkSortBy sortBy) {
    state = state.copyWith(sortBy: sortBy);
  }

  void reset() {
    state = const WalkFilterState();
  }
}

// Provider
final walkFilterProvider =
    StateNotifierProvider<WalkFilterNotifier, WalkFilterState>((ref) {
  return WalkFilterNotifier();
});

// Filtered walks provider that reacts to filter changes
final filteredWalksProvider = FutureProvider<List<Walk>>((ref) async {
  final filter = ref.watch(walkFilterProvider);
  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalks(
    status: filter.status,
    dateRange: filter.dateRange,
    search: filter.searchQuery,
    sortBy: filter.sortBy,
  );
});
```

### 5. StreamProvider (Real-Time Data)

For WebSocket streams, real-time updates, connectivity status.

```dart
// Connectivity stream
final connectivityProvider = StreamProvider<ConnectivityResult>((ref) {
  return Connectivity().onConnectivityChanged;
});

// Usage
final connectivity = ref.watch(connectivityProvider);
connectivity.when(
  data: (result) => result == ConnectivityResult.none
      ? const OfflineBanner()
      : const SizedBox.shrink(),
  loading: () => const SizedBox.shrink(),
  error: (_, __) => const SizedBox.shrink(),
);
```

### 6. StateProvider (Simple Mutable Value)

For a single value that changes. Tab index, toggle, selected item.

```dart
final selectedTabProvider = StateProvider<int>((ref) => 0);
final isDarkModeProvider = StateProvider<bool>((ref) => false);

// Usage
final tabIndex = ref.watch(selectedTabProvider);
ref.read(selectedTabProvider.notifier).state = 1;
```

---

## AsyncValue Pattern

The core pattern for handling async data in widgets. Every FutureProvider and StreamProvider returns an `AsyncValue`.

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  final walksAsync = ref.watch(walksProvider);

  return walksAsync.when(
    data: (walks) => WalksList(walks: walks),
    loading: () => const LoadingView(),
    error: (error, stack) => ErrorView(
      message: error.toString(),
      onRetry: () => ref.invalidate(walksProvider),
    ),
  );
}
```

### Preserving Previous Data During Refresh

When refreshing, `when` defaults to showing loading again. To keep old data visible during refresh:

```dart
// Option 1: skipLoadingOnRefresh (built-in)
walksAsync.when(
  skipLoadingOnRefresh: true, // keeps showing old data during refresh
  data: (walks) => WalksList(walks: walks),
  loading: () => const LoadingView(),
  error: (error, stack) => ErrorView(...),
);

// Option 2: use hasValue + isRefreshing
if (walksAsync.isRefreshing) {
  // show a subtle indicator (e.g., linear progress bar at top)
}
final walks = walksAsync.valueOrNull;
if (walks != null) {
  // render data even during refresh
}
```

### Combining Multiple Providers

When a screen needs data from multiple providers:

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  final walksAsync = ref.watch(walksProvider);
  final profileAsync = ref.watch(profileProvider);

  // Both must be loaded
  if (walksAsync.isLoading || profileAsync.isLoading) {
    return const LoadingView();
  }

  if (walksAsync.hasError) {
    return ErrorView(message: walksAsync.error.toString());
  }

  final walks = walksAsync.requireValue;
  final profile = profileAsync.requireValue;

  return DashboardContent(walks: walks, profile: profile);
}
```

---

## ref.watch vs ref.read

### ref.watch -- Reactive, Rebuild on Change

Use inside `build` method. Widget rebuilds when the provider value changes.

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  // CORRECT: watch in build -- rebuilds when data changes
  final walks = ref.watch(walksProvider);
  final filter = ref.watch(walkFilterProvider);

  return ...;
}
```

### ref.read -- One-Time Read, No Rebuild

Use inside callbacks, event handlers, `onPressed`. Never in `build`.

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  return ElevatedButton(
    onPressed: () {
      // CORRECT: read in callback
      ref.read(walkFilterProvider.notifier).reset();
    },
    child: Text('Reset'),
  );
}
```

### Common Mistake

```dart
// BAD: watch in callback -- causes unnecessary rebuilds
onPressed: () {
  ref.watch(walkFilterProvider.notifier).reset(); // WRONG
}

// BAD: read in build -- won't rebuild when data changes
final walks = ref.read(walksProvider); // WRONG in build method
```

---

## Invalidation and Refresh

### ref.invalidate -- Discard and Refetch

Marks the provider as dirty. Next time a widget reads it, it will refetch.

```dart
// After creating a new walk, invalidate the list to refetch
Future<void> _createWalk(WidgetRef ref, Walk walk) async {
  await ref.read(walkRepositoryProvider).createWalk(walk);
  ref.invalidate(walksProvider); // will refetch on next read
}
```

### ref.refresh -- Invalidate + Immediately Read

Forces an immediate refetch and returns the new Future.

```dart
// Pull-to-refresh
RefreshIndicator(
  onRefresh: () => ref.refresh(walksProvider.future),
  child: ...,
);
```

### When to Use Which

| Scenario | Use |
|----------|-----|
| After a mutation (create/update/delete) | `ref.invalidate` |
| Pull-to-refresh (need to await completion) | `ref.refresh(provider.future)` |
| User taps retry button | `ref.invalidate` |

---

## Provider Lifecycle and Scoping

### autoDispose

Providers are kept alive by default. Use `autoDispose` to free resources when no widget listens:

```dart
// Disposed when the screen is popped
final walkDetailProvider = FutureProvider.autoDispose.family<Walk, String>(
  (ref, walkId) async {
    final repository = ref.watch(walkRepositoryProvider);
    return repository.getWalkById(walkId);
  },
);

// Keep alive for a duration even after listeners are gone
final walksProvider = FutureProvider.autoDispose<List<Walk>>((ref) async {
  // Keeps data cached for 30 seconds after last listener
  ref.keepAlive();
  final link = ref.keepAlive();
  Timer(const Duration(seconds: 30), link.close);

  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalks();
});
```

### Provider Scoping with ProviderScope

Override providers for different parts of the widget tree (useful for testing):

```dart
// In tests: override repository with a mock
ProviderScope(
  overrides: [
    walkRepositoryProvider.overrideWithValue(MockWalkRepository()),
  ],
  child: const WalksScreen(),
);
```

---

## Provider Organization

### File structure

```
lib/features/walks/
  presentation/
    providers/
      walks_provider.dart          # FutureProvider for walk list
      walk_detail_provider.dart    # FutureProvider.family for single walk
      walk_filter_provider.dart    # StateNotifierProvider for filters
```

### Naming Convention

| Provider Type | Naming Pattern | Example |
|--------------|----------------|---------|
| FutureProvider (list) | `{feature}Provider` | `walksProvider` |
| FutureProvider.family (detail) | `{feature}DetailProvider` | `walkDetailProvider` |
| StateNotifierProvider | `{feature}{Purpose}Provider` | `walkFilterProvider` |
| StateProvider | `{purpose}Provider` | `selectedTabProvider` |
| StreamProvider | `{feature}StreamProvider` | `walkUpdatesStreamProvider` |

---

## Anti-Patterns

### 1. Provider doing business logic
```dart
// BAD: calculating in the provider
final bmiProvider = Provider<double>((ref) {
  final profile = ref.watch(profileProvider).valueOrNull;
  return profile!.weight / (profile.height * profile.height);
});

// GOOD: API returns calculated data
final profileProvider = FutureProvider<Profile>((ref) async {
  final repo = ref.watch(profileRepositoryProvider);
  return repo.getProfile(); // API returns bmi field
});
```

### 2. Deeply nested provider chains
```dart
// BAD: 5 providers chained together
// GOOD: flatten the chain, max 2-3 levels
```

### 3. Forgetting autoDispose on detail screens
```dart
// BAD: detail data stays in memory after navigating away
final walkDetailProvider = FutureProvider.family<Walk, String>(...);

// GOOD: dispose when the detail screen is popped
final walkDetailProvider = FutureProvider.autoDispose.family<Walk, String>(...);
```
