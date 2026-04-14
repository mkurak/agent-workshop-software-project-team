# Responsive Design (Mobile + Tablet)

Every screen must work on both mobile phones and tablets. We use a breakpoint system with a `ResponsiveLayout` widget and helper utilities. No screen is mobile-only.

---

## Breakpoints

```dart
// lib/core/responsive/breakpoints.dart

abstract class Breakpoints {
  /// Mobile: 0 - 599px
  static const double mobile = 0;

  /// Landscape / small tablet: 600 - 905px
  static const double landscape = 600;

  /// Tablet: 906px+
  static const double tablet = 906;
}
```

---

## ResponsiveLayout Widget

The core widget for switching between mobile and tablet layouts. Every screen uses this.

```dart
// lib/core/widgets/responsive_layout.dart
import 'package:flutter/material.dart';
import 'package:walking_for_me/core/responsive/breakpoints.dart';

class ResponsiveLayout extends StatelessWidget {
  const ResponsiveLayout({
    super.key,
    required this.mobile,
    this.tablet,
  });

  /// Mobile layout (required, also used as fallback for tablet if tablet is null)
  final Widget mobile;

  /// Tablet layout (optional, falls back to mobile)
  final Widget? tablet;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        if (constraints.maxWidth >= Breakpoints.tablet && tablet != null) {
          return tablet!;
        }
        return mobile;
      },
    );
  }
}
```

### Usage in Screens

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  final walksAsync = ref.watch(walksProvider);

  return Scaffold(
    appBar: AppBar(title: Text(context.l10n.walks_title)),
    body: walksAsync.when(
      data: (walks) => ResponsiveLayout(
        mobile: _MobileWalkList(walks: walks),
        tablet: _TabletWalkGrid(walks: walks),
      ),
      loading: () => const LoadingView(),
      error: (e, _) => ErrorView(message: e.toString()),
    ),
  );
}
```

---

## responsiveValue Helper

For inline responsive values (padding, font sizes, grid columns) without needing a full layout switch.

```dart
// lib/core/responsive/responsive_value.dart
import 'package:flutter/material.dart';
import 'package:walking_for_me/core/responsive/breakpoints.dart';

T responsiveValue<T>(
  BuildContext context, {
  required T mobile,
  T? landscape,
  T? tablet,
}) {
  final width = MediaQuery.sizeOf(context).width;

  if (width >= Breakpoints.tablet) {
    return tablet ?? landscape ?? mobile;
  }
  if (width >= Breakpoints.landscape) {
    return landscape ?? mobile;
  }
  return mobile;
}
```

### Usage

```dart
@override
Widget build(BuildContext context) {
  final padding = responsiveValue<double>(
    context,
    mobile: 16,
    tablet: 32,
  );

  final columns = responsiveValue<int>(
    context,
    mobile: 1,
    landscape: 2,
    tablet: 3,
  );

  final titleStyle = responsiveValue<TextStyle>(
    context,
    mobile: context.text.titleMedium!,
    tablet: context.text.titleLarge!,
  );

  return Padding(
    padding: EdgeInsets.all(padding),
    child: GridView.builder(
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: columns,
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
      ),
      itemCount: items.length,
      itemBuilder: (context, index) => ItemCard(item: items[index]),
    ),
  );
}
```

---

## Context Extensions

Convenient extensions for quick device checks in any widget.

```dart
// lib/core/extensions/context_extensions.dart
import 'package:flutter/material.dart';
import 'package:walking_for_me/core/responsive/breakpoints.dart';

extension ResponsiveExtension on BuildContext {
  double get screenWidth => MediaQuery.sizeOf(this).width;
  double get screenHeight => MediaQuery.sizeOf(this).height;

  bool get isMobile => screenWidth < Breakpoints.landscape;
  bool get isLandscape =>
      screenWidth >= Breakpoints.landscape && screenWidth < Breakpoints.tablet;
  bool get isTablet => screenWidth >= Breakpoints.tablet;

  /// Content max width (centered on large screens)
  double get contentMaxWidth => isMobile ? double.infinity : 800;
}
```

### Usage

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: Text(context.l10n.settings_title),
      // Show actions inline on tablet, in menu on mobile
      actions: context.isTablet
          ? [
              IconButton(onPressed: _onEdit, icon: const Icon(Icons.edit)),
              IconButton(onPressed: _onShare, icon: const Icon(Icons.share)),
            ]
          : [
              PopupMenuButton<String>(
                onSelected: _onMenuAction,
                itemBuilder: (_) => [
                  const PopupMenuItem(value: 'edit', child: Text('Edit')),
                  const PopupMenuItem(value: 'share', child: Text('Share')),
                ],
              ),
            ],
    ),
  );
}
```

---

## LayoutBuilder for Container-Based Responsiveness

Use `LayoutBuilder` when responsiveness should depend on the parent container width, not the screen width. This is essential for reusable widgets.

```dart
class WalkCard extends StatelessWidget {
  const WalkCard({super.key, required this.walk});
  final Walk walk;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        // Compact card if container is narrow
        if (constraints.maxWidth < 300) {
          return _CompactWalkCard(walk: walk);
        }
        // Full card if container is wide
        return _FullWalkCard(walk: walk);
      },
    );
  }
}
```

---

## Common Responsive Patterns

### List on Mobile, Grid on Tablet

```dart
ResponsiveLayout(
  mobile: ListView.builder(
    padding: const EdgeInsets.all(16),
    itemCount: walks.length,
    itemBuilder: (_, i) => WalkListTile(walk: walks[i]),
  ),
  tablet: GridView.builder(
    padding: const EdgeInsets.all(24),
    gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
      crossAxisCount: 2,
      crossAxisSpacing: 16,
      mainAxisSpacing: 16,
      childAspectRatio: 1.4,
    ),
    itemCount: walks.length,
    itemBuilder: (_, i) => WalkGridCard(walk: walks[i]),
  ),
);
```

### Master-Detail on Tablet

On mobile, the list and detail are separate screens. On tablet, they are side by side.

```dart
class WalksScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final walksAsync = ref.watch(walksProvider);
    final selectedId = ref.watch(selectedWalkIdProvider);

    return Scaffold(
      body: walksAsync.when(
        data: (walks) => ResponsiveLayout(
          mobile: WalkListView(
            walks: walks,
            onTap: (walk) => context.push('/walks/${walk.id}'),
          ),
          tablet: Row(
            children: [
              SizedBox(
                width: 360,
                child: WalkListView(
                  walks: walks,
                  selectedId: selectedId,
                  onTap: (walk) {
                    ref.read(selectedWalkIdProvider.notifier).state = walk.id;
                  },
                ),
              ),
              const VerticalDivider(width: 1),
              Expanded(
                child: selectedId != null
                    ? WalkDetailPanel(walkId: selectedId)
                    : const Center(child: Text('Select a walk')),
              ),
            ],
          ),
        ),
        loading: () => const LoadingView(),
        error: (e, _) => ErrorView(message: e.toString()),
      ),
    );
  }
}
```

### Centered Content on Large Screens

Prevent content from stretching too wide on tablets:

```dart
class ContentConstraint extends StatelessWidget {
  const ContentConstraint({super.key, required this.child, this.maxWidth = 800});
  final Widget child;
  final double maxWidth;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: ConstrainedBox(
        constraints: BoxConstraints(maxWidth: maxWidth),
        child: child,
      ),
    );
  }
}

// Usage
ContentConstraint(
  child: ListView(
    children: [
      ProfileHeader(profile: profile),
      ProfileStats(stats: stats),
    ],
  ),
),
```

### Responsive Padding

```dart
EdgeInsets get screenPadding => EdgeInsets.symmetric(
      horizontal: responsiveValue(context, mobile: 16.0, tablet: 32.0),
      vertical: responsiveValue(context, mobile: 16.0, tablet: 24.0),
    );
```

### Responsive Dialog

Dialogs should be constrained on tablet:

```dart
Future<void> _showWalkDialog(BuildContext context) {
  return showDialog(
    context: context,
    builder: (context) => Dialog(
      child: ConstrainedBox(
        constraints: const BoxConstraints(maxWidth: 500),
        child: const NewWalkForm(),
      ),
    ),
  );
}
```

---

## Orientation Handling

Lock to portrait on phones, allow landscape on tablets:

```dart
// In main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Allow all orientations initially
  await SystemChrome.setPreferredOrientations([
    DeviceOrientation.portraitUp,
    DeviceOrientation.portraitDown,
    DeviceOrientation.landscapeLeft,
    DeviceOrientation.landscapeRight,
  ]);

  runApp(const ProviderScope(child: App()));
}

// In a widget that needs to restrict orientation
class _PhoneOnlyPortrait extends StatefulWidget { ... }

class _PhoneOnlyPortraitState extends State<_PhoneOnlyPortrait> {
  @override
  void initState() {
    super.initState();
    if (!context.isTablet) {
      SystemChrome.setPreferredOrientations([DeviceOrientation.portraitUp]);
    }
  }

  @override
  void dispose() {
    SystemChrome.setPreferredOrientations(DeviceOrientation.values);
    super.dispose();
  }
}
```

---

## Anti-Patterns

### 1. Fixed pixel widths
```dart
// BAD
SizedBox(width: 375, child: ...)
// GOOD
Expanded(child: ...) or ConstrainedBox(constraints: BoxConstraints(maxWidth: 600))
```

### 2. Only testing on mobile
```dart
// BAD: works on phone, broken on tablet
// GOOD: test with ResponsiveLayout providing both mobile and tablet builders
```

### 3. Using screen width for container-aware widgets
```dart
// BAD: uses screen width even though the widget might be in a sidebar
if (MediaQuery.of(context).size.width > 600) ...
// GOOD: use LayoutBuilder for container-relative decisions
LayoutBuilder(builder: (context, constraints) {
  if (constraints.maxWidth > 600) ...
})
```
