# API Integration (Dio)

Flutter is a bridge. All data comes from the API, all mutations go to the API. Dio is the HTTP client. Interceptors handle auth tokens and error transformation. The repository pattern abstracts the HTTP layer from the presentation layer.

---

## Dio Setup

Single Dio instance configured with base URL, timeouts, and interceptors. Exposed as a Riverpod provider.

```dart
// lib/core/network/dio_provider.dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:walking_for_me/core/network/auth_interceptor.dart';
import 'package:walking_for_me/core/network/error_interceptor.dart';
import 'package:walking_for_me/core/network/logging_interceptor.dart';

final dioProvider = Provider<Dio>((ref) {
  final dio = Dio(
    BaseOptions(
      baseUrl: const String.fromEnvironment(
        'API_BASE_URL',
        defaultValue: 'http://10.0.2.2:3000', // Android emulator -> host localhost
      ),
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 15),
      sendTimeout: const Duration(seconds: 10),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    ),
  );

  dio.interceptors.addAll([
    AuthInterceptor(ref: ref, dio: dio),
    ErrorInterceptor(),
    if (kDebugMode) LoggingInterceptor(),
  ]);

  return dio;
});
```

### Platform Base URL

```dart
// Android emulator: 10.0.2.2 maps to host machine's localhost
// iOS simulator: localhost works directly
// Physical device: use actual network IP or deployed API URL

String get apiBaseUrl {
  if (kIsWeb) return 'http://localhost:3000';
  if (Platform.isAndroid) return 'http://10.0.2.2:3000';
  return 'http://localhost:3000'; // iOS simulator
}
```

---

## Auth Interceptor

Injects the access token into every request. On 401 response, attempts a token refresh and retries the original request.

```dart
// lib/core/network/auth_interceptor.dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:walking_for_me/features/auth/data/token_storage.dart';

class AuthInterceptor extends Interceptor {
  AuthInterceptor({required this.ref, required this.dio});

  final Ref ref;
  final Dio dio;
  bool _isRefreshing = false;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final tokenStorage = ref.read(tokenStorageProvider);
    final accessToken = await tokenStorage.getAccessToken();

    if (accessToken != null) {
      options.headers['Authorization'] = 'Bearer $accessToken';
    }

    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode != 401) {
      return handler.next(err);
    }

    // Avoid infinite refresh loop
    if (_isRefreshing) {
      return handler.next(err);
    }

    _isRefreshing = true;

    try {
      final tokenStorage = ref.read(tokenStorageProvider);
      final refreshToken = await tokenStorage.getRefreshToken();

      if (refreshToken == null) {
        _logout();
        return handler.next(err);
      }

      // Call refresh endpoint with a clean Dio instance (no interceptors)
      final refreshDio = Dio(BaseOptions(baseUrl: dio.options.baseUrl));
      final response = await refreshDio.post(
        '/api/auth/refresh',
        data: {'refreshToken': refreshToken},
      );

      final newAccessToken = response.data['accessToken'] as String;
      final newRefreshToken = response.data['refreshToken'] as String;

      await tokenStorage.saveTokens(
        accessToken: newAccessToken,
        refreshToken: newRefreshToken,
      );

      // Retry the original request with the new token
      final retryOptions = err.requestOptions;
      retryOptions.headers['Authorization'] = 'Bearer $newAccessToken';

      final retryResponse = await dio.fetch(retryOptions);
      return handler.resolve(retryResponse);
    } on DioException {
      _logout();
      return handler.next(err);
    } finally {
      _isRefreshing = false;
    }
  }

  void _logout() {
    ref.read(tokenStorageProvider).clearTokens();
    ref.read(authStateProvider.notifier).logout();
  }
}
```

---

## Error Interceptor

Transforms Dio exceptions into app-specific, user-friendly errors.

```dart
// lib/core/network/error_interceptor.dart
import 'package:dio/dio.dart';
import 'package:walking_for_me/core/errors/app_exception.dart';

class ErrorInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final appException = _mapToAppException(err);
    handler.next(
      DioException(
        requestOptions: err.requestOptions,
        response: err.response,
        type: err.type,
        error: appException,
      ),
    );
  }

  AppException _mapToAppException(DioException err) {
    // No internet / timeout
    if (err.type == DioExceptionType.connectionTimeout ||
        err.type == DioExceptionType.receiveTimeout ||
        err.type == DioExceptionType.sendTimeout) {
      return const NetworkException('Connection timed out. Please try again.');
    }

    if (err.type == DioExceptionType.connectionError) {
      return const NetworkException('No internet connection.');
    }

    // Server responded with error
    final statusCode = err.response?.statusCode;
    final data = err.response?.data;

    // Try to extract error message from API response
    final serverMessage = data is Map<String, dynamic>
        ? data['message'] as String? ?? data['error'] as String?
        : null;

    return switch (statusCode) {
      400 => ValidationException(serverMessage ?? 'Invalid request.'),
      401 => const UnauthorizedException('Session expired. Please log in again.'),
      403 => const ForbiddenException('You do not have permission for this action.'),
      404 => NotFoundException(serverMessage ?? 'Resource not found.'),
      409 => ConflictException(serverMessage ?? 'Conflict. Please refresh and try again.'),
      422 => ValidationException(serverMessage ?? 'Validation failed.'),
      429 => const RateLimitException('Too many requests. Please wait a moment.'),
      >= 500 => const ServerException('Server error. Please try again later.'),
      _ => AppException(serverMessage ?? 'An unexpected error occurred.'),
    };
  }
}
```

### App Exception Hierarchy

```dart
// lib/core/errors/app_exception.dart

class AppException implements Exception {
  const AppException(this.message);
  final String message;

  @override
  String toString() => message;
}

class NetworkException extends AppException {
  const NetworkException(super.message);
}

class UnauthorizedException extends AppException {
  const UnauthorizedException(super.message);
}

class ForbiddenException extends AppException {
  const ForbiddenException(super.message);
}

class NotFoundException extends AppException {
  const NotFoundException(super.message);
}

class ValidationException extends AppException {
  const ValidationException(super.message);
}

class ConflictException extends AppException {
  const ConflictException(super.message);
}

class RateLimitException extends AppException {
  const RateLimitException(super.message);
}

class ServerException extends AppException {
  const ServerException(super.message);
}
```

---

## API Response Models

Every API response is a Dart model with `fromJson` factory and `toJson` method.

```dart
// lib/features/walks/data/models/walk_model.dart

class Walk {
  const Walk({
    required this.id,
    required this.title,
    required this.distanceKm,
    required this.durationMinutes,
    required this.startedAt,
    this.completedAt,
    required this.status,
  });

  final String id;
  final String title;
  final double distanceKm;
  final int durationMinutes;
  final DateTime startedAt;
  final DateTime? completedAt;
  final WalkStatus status;

  factory Walk.fromJson(Map<String, dynamic> json) {
    return Walk(
      id: json['id'] as String,
      title: json['title'] as String,
      distanceKm: (json['distanceKm'] as num).toDouble(),
      durationMinutes: json['durationMinutes'] as int,
      startedAt: DateTime.parse(json['startedAt'] as String),
      completedAt: json['completedAt'] != null
          ? DateTime.parse(json['completedAt'] as String)
          : null,
      status: WalkStatus.values.byName(json['status'] as String),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'distanceKm': distanceKm,
      'durationMinutes': durationMinutes,
      'startedAt': startedAt.toIso8601String(),
      'completedAt': completedAt?.toIso8601String(),
      'status': status.name,
    };
  }
}

enum WalkStatus { planned, active, completed, cancelled }
```

### Paginated Response Model

```dart
// lib/core/models/paginated_response.dart

class PaginatedResponse<T> {
  const PaginatedResponse({
    required this.items,
    required this.total,
    required this.page,
    required this.pageSize,
    this.nextCursor,
  });

  final List<T> items;
  final int total;
  final int page;
  final int pageSize;
  final String? nextCursor;

  bool get hasMore => nextCursor != null;

  factory PaginatedResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Map<String, dynamic>) fromJsonT,
  ) {
    return PaginatedResponse(
      items: (json['items'] as List<dynamic>)
          .map((e) => fromJsonT(e as Map<String, dynamic>))
          .toList(),
      total: json['total'] as int,
      page: json['page'] as int,
      pageSize: json['pageSize'] as int,
      nextCursor: json['nextCursor'] as String?,
    );
  }
}
```

---

## Repository Pattern

Abstract interface defines the contract. Implementation uses Dio.

```dart
// lib/features/walks/data/repositories/walk_repository.dart

// Abstract interface
abstract class WalkRepository {
  Future<List<Walk>> getWalks({String? status, int page = 1});
  Future<Walk> getWalkById(String id);
  Future<Walk> createWalk(CreateWalkRequest request);
  Future<Walk> updateWalk(String id, UpdateWalkRequest request);
  Future<void> deleteWalk(String id);
}

// Implementation
class WalkRepositoryImpl implements WalkRepository {
  WalkRepositoryImpl(this._dio);
  final Dio _dio;

  @override
  Future<List<Walk>> getWalks({String? status, int page = 1}) async {
    final response = await _dio.get(
      '/api/walks',
      queryParameters: {
        if (status != null) 'status': status,
        'page': page,
      },
    );

    final data = response.data as List<dynamic>;
    return data
        .map((json) => Walk.fromJson(json as Map<String, dynamic>))
        .toList();
  }

  @override
  Future<Walk> getWalkById(String id) async {
    final response = await _dio.get('/api/walks/$id');
    return Walk.fromJson(response.data as Map<String, dynamic>);
  }

  @override
  Future<Walk> createWalk(CreateWalkRequest request) async {
    final response = await _dio.post(
      '/api/walks',
      data: request.toJson(),
    );
    return Walk.fromJson(response.data as Map<String, dynamic>);
  }

  @override
  Future<Walk> updateWalk(String id, UpdateWalkRequest request) async {
    final response = await _dio.put(
      '/api/walks/$id',
      data: request.toJson(),
    );
    return Walk.fromJson(response.data as Map<String, dynamic>);
  }

  @override
  Future<void> deleteWalk(String id) async {
    await _dio.delete('/api/walks/$id');
  }
}
```

### Request Models

```dart
class CreateWalkRequest {
  const CreateWalkRequest({
    required this.title,
    required this.distanceKm,
    this.notes,
  });

  final String title;
  final double distanceKm;
  final String? notes;

  Map<String, dynamic> toJson() => {
        'title': title,
        'distanceKm': distanceKm,
        if (notes != null) 'notes': notes,
      };
}
```

---

## Riverpod Provider for Repository

```dart
// lib/features/walks/data/repositories/walk_repository.dart (bottom of file)

final walkRepositoryProvider = Provider<WalkRepository>((ref) {
  final dio = ref.watch(dioProvider);
  return WalkRepositoryImpl(dio);
});
```

---

## Token Storage

Secure storage for JWT tokens:

```dart
// lib/features/auth/data/token_storage.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final tokenStorageProvider = Provider<TokenStorage>((ref) {
  return TokenStorage();
});

class TokenStorage {
  static const _accessTokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';

  final _storage = const FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );

  Future<String?> getAccessToken() =>
      _storage.read(key: _accessTokenKey);

  Future<String?> getRefreshToken() =>
      _storage.read(key: _refreshTokenKey);

  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await Future.wait([
      _storage.write(key: _accessTokenKey, value: accessToken),
      _storage.write(key: _refreshTokenKey, value: refreshToken),
    ]);
  }

  Future<void> clearTokens() async {
    await Future.wait([
      _storage.delete(key: _accessTokenKey),
      _storage.delete(key: _refreshTokenKey),
    ]);
  }
}
```

---

## Complete Flow Example

From provider to screen, end to end:

```dart
// 1. Provider fetches data via repository
final walksProvider = FutureProvider<List<Walk>>((ref) async {
  final repository = ref.watch(walkRepositoryProvider);
  return repository.getWalks();
});

// 2. Screen watches provider and handles AsyncValue
class WalksScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final walksAsync = ref.watch(walksProvider);

    return Scaffold(
      body: walksAsync.when(
        data: (walks) => WalksList(walks: walks),
        loading: () => const LoadingView(),
        error: (error, _) => ErrorView(
          message: error is AppException ? error.message : 'Unexpected error',
          onRetry: () => ref.invalidate(walksProvider),
        ),
      ),
    );
  }
}

// 3. Mutation: create a walk
Future<void> _createWalk(WidgetRef ref, CreateWalkRequest request) async {
  final repository = ref.read(walkRepositoryProvider);
  await repository.createWalk(request);
  ref.invalidate(walksProvider); // Refresh the list
}
```

---

## Anti-Patterns

### 1. Direct Dio call in widget
```dart
// BAD
final response = await Dio().get('http://localhost:3000/api/walks');
// GOOD
final walks = ref.watch(walksProvider);
```

### 2. Business logic in repository
```dart
// BAD: calculating in Flutter
Future<double> getTotalDistance() async {
  final walks = await getWalks();
  return walks.fold(0.0, (sum, w) => sum + w.distanceKm);
}
// GOOD: API returns the aggregate
Future<WalkStats> getWalkStats() async {
  final response = await _dio.get('/api/walks/stats');
  return WalkStats.fromJson(response.data);
}
```

### 3. Missing error handling
```dart
// BAD: no try-catch, no error interceptor
final response = await _dio.get('/api/walks');
// GOOD: ErrorInterceptor transforms all errors automatically
// Widget handles errors via AsyncValue.when(error: ...)
```

### 4. Hardcoded base URL
```dart
// BAD
Dio(BaseOptions(baseUrl: 'http://192.168.1.5:3000'))
// GOOD
Dio(BaseOptions(baseUrl: apiBaseUrl)) // platform-aware
```
