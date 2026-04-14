# Screen Blueprint (Primary Production Unit)

This is the primary production unit of the Flutter Agent. Every new screen follows this blueprint exactly.

## File Structure

```
lib/features/{feature}/
  data/
    models/{feature}_model.dart
    repositories/{feature}_repository.dart
  presentation/
    providers/{feature}_provider.dart
    screens/{screen}_screen.dart
    widgets/{widget}_widget.dart
```

Every feature is self-contained. No cross-feature imports at the presentation layer.

---

## Screen Template

Every screen is a `ConsumerWidget` (or `ConsumerStatefulWidget` if local animation/controller state is needed). The screen fetches data through a Riverpod provider and handles all three async states.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:walking_for_me/core/extensions/context_extensions.dart';
import 'package:walking_for_me/core/widgets/loading_view.dart';
import 'package:walking_for_me/core/widgets/error_view.dart';
import 'package:walking_for_me/core/widgets/empty_view.dart';
import 'package:walking_for_me/core/widgets/responsive_layout.dart';
import 'package:walking_for_me/features/walks/presentation/providers/walks_provider.dart';
import 'package:walking_for_me/features/walks/presentation/widgets/walk_list_tile.dart';
import 'package:walking_for_me/features/walks/presentation/widgets/walk_grid_card.dart';

class WalksScreen extends ConsumerWidget {
  const WalksScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final walksAsync = ref.watch(walksProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text(context.l10n.walks_title),
      ),
      body: walksAsync.when(
        data: (walks) {
          if (walks.isEmpty) {
            return EmptyView(
              icon: Icons.directions_walk,
              title: context.l10n.walks_empty_title,
              subtitle: context.l10n.walks_empty_subtitle,
            );
          }

          return RefreshIndicator(
            onRefresh: () => ref.refresh(walksProvider.future),
            child: ResponsiveLayout(
              mobile: _MobileLayout(walks: walks),
              tablet: _TabletLayout(walks: walks),
            ),
          );
        },
        loading: () => const LoadingView(),
        error: (error, stack) => ErrorView(
          message: error.toString(),
          onRetry: () => ref.invalidate(walksProvider),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => context.push('/walks/new'),
        child: const Icon(Icons.add),
      ),
    );
  }
}

class _MobileLayout extends StatelessWidget {
  const _MobileLayout({required this.walks});
  final List<Walk> walks;

  @override
  Widget build(BuildContext context) {
    return ListView.separated(
      padding: const EdgeInsets.all(16),
      itemCount: walks.length,
      separatorBuilder: (_, __) => const SizedBox(height: 8),
      itemBuilder: (context, index) => WalkListTile(walk: walks[index]),
    );
  }
}

class _TabletLayout extends StatelessWidget {
  const _TabletLayout({required this.walks});
  final List<Walk> walks;

  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      padding: const EdgeInsets.all(24),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
        childAspectRatio: 1.5,
      ),
      itemCount: walks.length,
      itemBuilder: (context, index) => WalkGridCard(walk: walks[index]),
    );
  }
}
```

---

## ConsumerStatefulWidget (When Needed)

Use `ConsumerStatefulWidget` ONLY when you need:
- `TextEditingController`, `ScrollController`, `AnimationController`
- `TickerProviderStateMixin` for animations
- `initState`/`dispose` lifecycle

```dart
class WalkDetailScreen extends ConsumerStatefulWidget {
  const WalkDetailScreen({super.key, required this.walkId});
  final String walkId;

  @override
  ConsumerState<WalkDetailScreen> createState() => _WalkDetailScreenState();
}

class _WalkDetailScreenState extends ConsumerState<WalkDetailScreen> {
  late final ScrollController _scrollController;

  @override
  void initState() {
    super.initState();
    _scrollController = ScrollController();
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final walkAsync = ref.watch(walkDetailProvider(widget.walkId));

    return Scaffold(
      body: walkAsync.when(
        data: (walk) => CustomScrollView(
          controller: _scrollController,
          slivers: [
            SliverAppBar(
              expandedHeight: 200,
              flexibleSpace: FlexibleSpaceBar(
                title: Text(walk.title),
              ),
            ),
            SliverPadding(
              padding: const EdgeInsets.all(16),
              sliver: SliverToBoxAdapter(
                child: _WalkContent(walk: walk),
              ),
            ),
          ],
        ),
        loading: () => const LoadingView(),
        error: (error, stack) => ErrorView(
          message: error.toString(),
          onRetry: () => ref.invalidate(walkDetailProvider(widget.walkId)),
        ),
      ),
    );
  }
}
```

---

## Route Registration

Every screen must be registered in the router. Add the route to the appropriate section in `lib/core/routing/app_router.dart`:

```dart
GoRoute(
  path: '/walks',
  name: 'walks',
  builder: (context, state) => const WalksScreen(),
  routes: [
    GoRoute(
      path: 'new',
      name: 'walks-new',
      builder: (context, state) => const NewWalkScreen(),
    ),
    GoRoute(
      path: ':walkId',
      name: 'walks-detail',
      builder: (context, state) {
        final walkId = state.pathParameters['walkId']!;
        return WalkDetailScreen(walkId: walkId);
      },
    ),
  ],
),
```

---

## i18n Strings

Every user-facing string must be added to both ARB files before the screen is complete.

**`lib/l10n/app_en.arb`:**
```json
{
  "walks_title": "My Walks",
  "walks_empty_title": "No walks yet",
  "walks_empty_subtitle": "Start your first walk by tapping the + button",
  "walks_error_load": "Could not load walks"
}
```

**`lib/l10n/app_tr.arb`:**
```json
{
  "walks_title": "Yuruyuslerim",
  "walks_empty_title": "Henuz yuruyus yok",
  "walks_empty_subtitle": "Ilk yuruyusunuzu baslatmak icin + tusuna dokunun",
  "walks_error_load": "Yuruyusler yuklenemedi"
}
```

---

## Provider for the Screen

Every screen has a corresponding provider that fetches data from the repository:

```dart
// lib/features/walks/presentation/providers/walks_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:walking_for_me/features/walks/data/repositories/walk_repository.dart';
import 'package:walking_for_me/features/walks/data/models/walk_model.dart';

final walksProvider = FutureProvider<List<Walk>>((ref) async {
  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalks();
});

final walkDetailProvider = FutureProvider.family<Walk, String>((ref, walkId) async {
  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalkById(walkId);
});
```

---

## Loading / Error / Empty State Widgets

These are shared across all screens. They live in `lib/core/widgets/`:

```dart
// lib/core/widgets/loading_view.dart
class LoadingView extends StatelessWidget {
  const LoadingView({super.key});

  @override
  Widget build(BuildContext context) {
    return const Center(
      child: CircularProgressIndicator(),
    );
  }
}

// lib/core/widgets/error_view.dart
class ErrorView extends StatelessWidget {
  const ErrorView({
    super.key,
    required this.message,
    this.onRetry,
  });

  final String message;
  final VoidCallback? onRetry;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              Icons.error_outline,
              size: 64,
              color: context.colors.error,
            ),
            const SizedBox(height: 16),
            Text(
              message,
              textAlign: TextAlign.center,
              style: context.text.bodyLarge,
            ),
            if (onRetry != null) ...[
              const SizedBox(height: 24),
              FilledButton.icon(
                onPressed: onRetry,
                icon: const Icon(Icons.refresh),
                label: Text(context.l10n.common_retry),
              ),
            ],
          ],
        ),
      ),
    );
  }
}

// lib/core/widgets/empty_view.dart
class EmptyView extends StatelessWidget {
  const EmptyView({
    super.key,
    required this.icon,
    required this.title,
    this.subtitle,
    this.action,
  });

  final IconData icon;
  final String title;
  final String? subtitle;
  final Widget? action;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(icon, size: 80, color: context.colors.outline),
            const SizedBox(height: 16),
            Text(
              title,
              style: context.text.titleMedium,
              textAlign: TextAlign.center,
            ),
            if (subtitle != null) ...[
              const SizedBox(height: 8),
              Text(
                subtitle!,
                style: context.text.bodyMedium?.copyWith(
                  color: context.colors.onSurfaceVariant,
                ),
                textAlign: TextAlign.center,
              ),
            ],
            if (action != null) ...[
              const SizedBox(height: 24),
              action!,
            ],
          ],
        ),
      ),
    );
  }
}
```

---

## Checklist

Before a screen is considered complete, every item must be checked:

| # | Item | Details |
|---|------|---------|
| 1 | **File location** | `lib/features/{feature}/presentation/screens/{screen}_screen.dart` |
| 2 | **ConsumerWidget** | Uses `ConsumerWidget` (or `ConsumerStatefulWidget` only if controllers needed) |
| 3 | **Provider** | Screen has a corresponding provider in `providers/` |
| 4 | **AsyncValue** | `.when(data:, loading:, error:)` pattern used |
| 5 | **Loading state** | Shows `LoadingView` or shimmer skeleton |
| 6 | **Error state** | Shows `ErrorView` with retry button |
| 7 | **Empty state** | Shows `EmptyView` with icon, title, subtitle |
| 8 | **Responsive** | `ResponsiveLayout` with separate mobile and tablet builders |
| 9 | **i18n** | All strings use `context.l10n.key`. Both EN and TR ARB files updated |
| 10 | **Route** | Registered in `app_router.dart` with name and path |
| 11 | **Theme** | Colors from `Theme.of(context)`, no hardcoded colors |
| 12 | **RefreshIndicator** | Pull-to-refresh on list screens |
| 13 | **Navigation** | `context.push`/`context.go` for navigation, not `Navigator.push` |
| 14 | **SafeArea** | Handled (Scaffold does it by default, check custom layouts) |
| 15 | **Tests** | Widget test file created at `test/features/{feature}/presentation/screens/{screen}_screen_test.dart` |

---

## Anti-Patterns (Never Do These)

### 1. Business logic in widget
```dart
// BAD: calculating in the widget
final bmi = weight / (height * height);
// GOOD: API returns the calculated value, widget only displays it
Text(walk.bmiFormatted)
```

### 2. Raw setState for API data
```dart
// BAD: fetching data with setState
void initState() {
  super.initState();
  setState(() => _isLoading = true);
  api.getWalks().then((data) {
    setState(() { _walks = data; _isLoading = false; });
  });
}
// GOOD: use a Riverpod provider
final walksAsync = ref.watch(walksProvider);
```

### 3. Hardcoded strings
```dart
// BAD
Text('My Walks')
// GOOD
Text(context.l10n.walks_title)
```

### 4. Hardcoded colors
```dart
// BAD
Container(color: Color(0xFF2196F3))
// GOOD
Container(color: context.colors.primary)
```

### 5. Navigator.push instead of go_router
```dart
// BAD
Navigator.push(context, MaterialPageRoute(builder: (_) => WalkDetailScreen()));
// GOOD
context.push('/walks/${walk.id}');
```

### 6. Ignoring error/empty states
```dart
// BAD: only handles data
body: ListView.builder(itemCount: walks.length, ...)
// GOOD: handles all three states
body: walksAsync.when(data: ..., loading: ..., error: ...)
```

### 7. One giant widget
```dart
// BAD: 500-line build method
// GOOD: extract private widgets or separate widget files
class _WalkHeader extends StatelessWidget { ... }
class _WalkStats extends StatelessWidget { ... }
```
