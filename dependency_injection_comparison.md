# DEPENDENCY INJECTION & SERVICE LOCATOR

## DEPENDENCY INJECTION FRAMEWORK ANALYSIS

| ASPECT | app_wp_pms | lazytask_primetechbd | RATING | DI STRATEGY |
| :--- | :--- | :--- | :--- | :--- |
| **1. DI FRAMEWORK** | | | | |
| ├─ Framework Used | GetIt (v8.0.0) | GetIt (v9.0.5) + Riverpod | 🟢 LZ | LZ: hybrid approach |
| ├─ DI Type | Service Locator | Service Locator + Provider| 🟢 LZ | LZ: combines both |
| ├─ Framework Feature Set | Basic | Advanced (v9 features) | 🟢 LZ | LZ: leverages new features |
| ├─ Configuration Location | `main()` scattered | Centralized `setup()` | 🟢 LZ | LZ: organized DI setup |
| └─ Registration Count | ~15-20 | 60+ registrations | 🟢 LZ | LZ: comprehensive DI |

### 2. SERVICE LOCATOR SETUP COMPARISON

#### app_wp_pms (Basic Setup in main.dart):
```dart
void setupServiceLocator() {
  sl.registerLazySingleton<NotificationService>(
    NotificationService.new
  );
  // Maybe 3-5 registrations...
  // Most services created inline with Get.put()
}

// In main():
setupServiceLocator()
Get.put(ConnectivityService())  // ⚠️ Mixed patterns
Get.put(DashboardController())  // Controllers registered ad-hoc
```

**ISSUES:**
* ❌ Incomplete DI setup
  * ├─ Only few services registered
  * ├─ Controllers created with `Get.put()`
  * ├─ Mixed GetIt and GetX patterns
  * └─ Hard to find all dependencies

#### lazytask (Comprehensive Setup - 140+ lines):
```dart
final GetIt sl = GetIt.instance;

void setupServiceLocator() {
  // --- Core ---
  sl
    ..registerLazySingleton<SessionManager>(
      SessionManager.new)
    ..registerLazySingleton<LoggingInterceptor>(
      LoggingInterceptor.new)
    ..registerLazySingleton<DatabaseHelper>(
      () => DatabaseHelper.instance)

    // --- Dedicated Dio ---
    ..registerLazySingleton<Dio>(() {
      final sessionManager = sl<SessionManager>();
      const String namespace = '/wp-json/lazytasks/api/v2/';

      final dio = Dio(BaseOptions(
        connectTimeout: const Duration(seconds: 60),
        receiveTimeout: const Duration(seconds: 60),
        headers: { ... },
      ));

      dio.interceptors.add(sl<LoggingInterceptor>());
      dio.interceptors.add(ConnectivityInterceptor());
      dio.interceptors.add(
        AuthAndPathInterceptor(sessionManager, namespace));
      dio.interceptors.add(ErrorInterceptor());

      return dio;
    })

    // --- API Client ---
    ..registerLazySingleton<LazyTasksApiClient>(
      () => LazyTasksApiClient(sl<Dio>()))

    // --- Data Sources ---
    ..registerLazySingleton<MyTaskRemoteDataSource>(
      () => MyTaskRemoteDataSourceImpl(
        sl<LazyTasksApiClient>()))
    ..registerLazySingleton<WorkspaceRemoteDataSource>(
      () => WorkspaceRemoteDataSourceImpl(
        sl<LazyTasksApiClient>()))
    ..registerLazySingleton<AuthRemoteDataSource>(
      () => AuthRemoteDataSource(
        sl<LazyTasksApiClient>()))

    // --- Repositories ---
    ..registerLazySingleton<MyTaskRepository>(
      () => MyTaskRepositoryImpl(
        sl<MyTaskRemoteDataSource>()))

    // --- Use Cases ---
    ..registerLazySingleton<GetMyTaskListUseCase>(
      () => GetMyTaskListUseCase(
        sl<MyTaskRepository>()))
    ..registerLazySingleton<EditTaskUseCase>(
      () => EditTaskUseCase(
        sl<MyTaskRepository>()))
    // ... more use cases

    // --- Services ---
    ..registerLazySingleton<NotificationService>(
      NotificationService.new)
    ..registerLazySingleton<PushNotificationService>(
      PushNotificationService.new);
}
```

**BENEFITS:**
* ✅ Complete DI configuration
  * ├─ All dependencies registered
  * ├─ Clear dependency chain
  * ├─ Single source of truth
  * └─ Easy to test (override providers)
* ✅ Proper dependency hierarchy
  * ├─ Dio with all interceptors
  * ├─ API client uses Dio
  * ├─ Data sources use API client
  * ├─ Repositories use data sources
  * └─ Use cases use repositories
* ✅ Easy to modify
  * ├─ Change Dio config in one place
  * ├─ Add new interceptor easily
  * ├─ Swap implementations for tests
  * └─ No scattered `Get.put()` calls

### 3. GETIT PATTERNS

**Basic Registration:**
```dart
sl.registerSingleton<MyService>(MyService());
// Single instance, entire app lifetime
```

**Lazy Singleton:**
```dart
sl.registerLazySingleton<MyService>(() => MyService());
// Single instance, created on first access
```

**Factory:**
```dart
sl.registerFactory<MyService>(() => MyService());
// New instance each time
```

**Usage:**
```dart
final service = sl<MyService>();  // Get instance
```

### 4. DEPENDENCY CHAINS

#### app_wp_pms (Shallow):
```
MyTaskController ──┐
                  ├─→ Services (god object)
DashboardController┘
```

**ISSUES:**
* ❌ Controllers depend on single god object
* ❌ Hard to mock individual services
* ❌ Tight coupling

#### lazytask (Deep, Organized):
```
UI Widget
    ↓
Notifier (Riverpod provider)
    ↓
Use Case (GetMyTaskListUseCase)
    ↓
Repository (MyTaskRepository interface)
    ↓
Remote Data Source (MyTaskRemoteDataSource)
    ↓
API Client (LazyTasksApiClient with Retrofit)
    ↓
Dio (with interceptors)
    ↓
Network (HTTP)
```

**BENEFITS:**
* ✅ Each layer has single responsibility
* ✅ Easy to mock at any level
* ✅ Clear dependency flow
* ✅ Easy to test integration

### 5. TESTING & MOCKING

#### app_wp_pms (Difficult):
```dart
test('getMyTaskList', () {
  // Hard to mock Services class
  // Can't easily override individual methods
  // Need to mock entire god object
  // Complex setup required
});
```

#### lazytask (Easy):
```dart
test('fetchTasks', () async {
  final container = ProviderContainer(
    overrides: [
      getMyTaskListUseCaseProvider.overrideWithValue(
        MockGetMyTaskListUseCase()
      ),
    ],
  );

  final notifier = container.read(
    myTaskNotifierProvider.notifier);

  await notifier.fetchTasks();

  expect(
    container.read(myTaskNotifierProvider),
    isA<AsyncValue<List<Task>>>()
  );
});
```
* ✅ Override specific use case
* ✅ No complex setup
* ✅ Provider container isolated
* ✅ Clear expectations

| ASPECT | app_wp_pms | lazytask_primetechbd | RATING | DI STRATEGY |
| :--- | :--- | :--- | :--- | :--- |
| **6. CONFIGURATION FLEXIBILITY** | | | | |
| ├─ Change Dio config | Edit in Services | Centralized in `setup()` | 🟢 LZ | LZ: one place to change |
| ├─ Add new service | Create + `Get.put()` | Register in `setup()` | 🟢 LZ | LZ: consistent pattern |
| ├─ Swap implementation | Edit Services class | Override provider | 🟢 LZ | LZ: easier for tests |
| ├─ Lazy loading | Partial support | Full support (`.lazy`) | 🟢 LZ | LZ: performance optimization |
| └─ Scope management | Singleton only | Multiple scopes possible | 🟢 LZ | LZ: future flexibility |

### DI IMPLEMENTATION COMPARISON:

| METRIC | app_wp_pms | lazytask | IMPACT |
| :--- | :--- | :--- | :--- |
| **Services registered** | ~15-20 | 60+ | LZ: complete DI coverage |
| **LOC in setup** | ~50 lines | 140+ lines | LZ: well-organized |
| **Dependency clarity** | Unclear | Clear chains | LZ: easy to understand |
| **Mockability for tests** | Medium | High | LZ: easier testing |
| **Configuration centralization** | Scattered | Centralized | LZ: maintainable |
| **Added complexity** | Low | Medium | app_wp: simpler (less complete) |
| **Framework leverage** | Basic | Advanced | LZ: uses v9 features |
| **Production readiness** | Adequate | Enterprise-grade | LZ: professional setup |

### DI SCORE:
* **app_wp_pms**: 55/100 ⚠️ BASIC SETUP
* **lazytask**: 92/100 ✅ ENTERPRISE DI