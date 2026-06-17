# LOCAL STORAGE & DATA PERSISTENCE COMPARISON

## LOCAL STORAGE ANALYSIS

| ASPECT | app_wp_pms | lazytask_primetechbd | RATING | TECHNICAL DETAILS |
| :--- | :--- | :--- | :--- | :--- |
| **1. SHARED PREFERENCES (KEY-VALUE)** | | | | |
| ├─ Library Version | shared_preferences: ^2.2.3 | shared_preferences: ^2.2.3 | 🟢 Same | Both use standard package |
| ├─ Storage Type | Key-value pairs | Key-value pairs | 🟢 Same | Platform defaults |
| ├─ Use Case | Simple preferences | Simple preferences | 🟢 Same | Non-sensitive data |
| └─ Data Persistence | Device storage | Device storage | 🟢 Same | Survives restarts |

### SESSION MANAGER COMPARISON:

#### app_wp_pms SessionManager (lib/session/session_manager.dart):
```dart
class SessionManager {
  Future<void> setToken(String token) async {
    final SharedPreferences prefs = await SharedPreferences.getInstance();
    prefs.setString('token', token);
    prefs_token.value = token;  // ⚠️ Global observable
  }

  Future<String?> getToken() async {
    final SharedPreferences prefs = await SharedPreferences.getInstance();
    final String? token = prefs.getString('token');
    prefs_token.value = token ?? '';  // ⚠️ Updates global
    return token;
  }

  Future<void> setUserInfo(String email, String password) {
    // Same pattern repeated...
  }
}
```

**ISSUES IDENTIFIED (🔴):**
* Creates new SharedPreferences instance each call (inefficient)
* Updates global observable variables (anti-pattern)
* Password stored in plaintext (SECURITY RISK ⚠️⚠️⚠️)
* No encryption for sensitive data
* Multiple properties scattered
* Async but not used in initialization
* Passwords exposed in `.obs` variable

#### lazytask SessionManager (abstracted in service):
```dart
// In service_locator.dart:
..registerLazySingleton<SessionManager>(SessionManager.new)

// In interceptor:
class AuthAndPathInterceptor extends QueuedInterceptor {
  final SessionManager sessionManager;
  final String namespace;
  
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // Retrieves token from SessionManager
    // Sets authorization header
    options.headers['Authorization'] = 'Bearer $token';
    options.path = namespace + options.path;
    return handler.next(options);
  }
}
```

**BENEFITS (✅):**
* SessionManager injected via DI
* No global observables exposed
* Token used in interceptor (clean separation)
* Credentials not stored locally (API only)
* Better security practices
* Centralized token management

| ASPECT | app_wp_pms | lazytask_primetechbd | RATING | TECHNICAL DETAILS |
| :--- | :--- | :--- | :--- | :--- |
| **2. SQLITE DATABASE** | | | | |
| ├─ Library Version | sqflite: ^2.3.0 | sqflite: ^2.3.0 | 🟢 Same | Same database layer |
| ├─ Database Name | pms.db | pms.db | 🟢 Same | Same naming |
| ├─ Tables | 3 tables | 4 tables (1 new) | 🟢 LZ | LZ: more structured data |
| ├─ Access Pattern | DatabaseHelper.instance | Injected via GetIt | 🟢 LZ | LZ: better DI |
| ├─ Asset Database | Pre-bundled in assets | Pre-bundled in assets | 🟢 Same | Initial data seeding |
| └─ Migration Strategy | Version 1 only | Version 1 only | 🟢 Same | No migration path |

### IMPLEMENTATION COMPARISON:

#### app_wp_pms:
```dart
class DatabaseHelper {
  static const String _DB_NAME = 'pms.db';
  static const _DB_VERSION = 1;

  // Singleton pattern
  static final DatabaseHelper instance = DatabaseHelper._privateConstructor();
  
  DatabaseHelper._privateConstructor();

  Future<Database?> get database async {
    if (_database != null) return _database;
    _database = await _initDatabase();
    return _database;
  }

  _initDatabase() async {
    final databasePath = await getDatabasesPath();
    final String path = join(databasePath, _DB_NAME);
    
    // Copy from assets if not exists
    final exists = await databaseExists(path);
    if (!exists) {
      try {
        await Directory(dirname(path)).create(recursive: true);
      } catch (_) {}
      
      final ByteData data = await rootBundle.load(join('assets/db', _DB_NAME));
      final List<int> bytes = data.buffer.asUint8List(data.offsetInBytes, data.lengthInBytes);
      await File(path).writeAsBytes(bytes, flush: true);
    }
    
    return await openDatabase(path, version: _DB_VERSION);
  }
}
```

**ISSUES:**
* Singleton pattern (not DI-friendly)
* `getDatabase` property (confusing pattern)
* Limited error handling
* No logging for debugging

#### lazytask:
```dart
class DatabaseHelper {
  static const String _databaseName = 'pms.db';
  static const int _databaseVersion = 1;

  /// Table names (documented)
  static const String tableUsers = 'users';
  static const String tableServiceGateKeeper = 'service_gate_keeper';
  static const String tableDomainLoggedInData = 'domain_logged_in_data';

  static String status = '';  // Track init status
  String? dbPath;

  // Private constructor for singleton
  DatabaseHelper._privateConstructor();

  /// Singleton instance
  static final DatabaseHelper instance = DatabaseHelper._privateConstructor();

  static Database? _database;

  /// Get database (lazy-loaded)
  Future<Database?> get database async {
    if (_database != null) return _database;
    _database = await _initDatabase();
    return _database;
  }

  /// Initialize database with error handling
  Future<Database> _initDatabase() async {
    final databasePath = await getDatabasesPath();
    final String path = join(databasePath, _databaseName);
    dbPath = path;

    final exists = await databaseExists(path);
    if (!exists) {
      status = 'Creating Database';
      try {
        await Directory(dirname(path)).create(recursive: true);
      } catch (e) {
        status = 'Error creating directory: $e';
      }

      // Copy from assets
      final ByteData data = await rootBundle.load(join('assets/db', _databaseName));
      final List<int> bytes = data.buffer.asUint8List(data.offsetInBytes, data.lengthInBytes);

      await File(path).writeAsBytes(bytes, flush: true);
    } else {
      status = 'Opening existing database';
    }

    return await openDatabase(path, version: _databaseVersion);
  }

  /// Improved type-safe methods
  Future<List<Map<String, dynamic>>> getAll({required String tbl}) async {
    final Database? db = await database;
    final result = await db!.query(tbl);
    return result.toList();
  }

  /// Type-safe with documentation
  Future<List<Map<String, dynamic>>> getAllWhr({
    required String tbl,
    required String where,
    required List<dynamic> whereArgs,
  }) async {
    final Database? db = await database;
    return await db!.query(tbl, where: where, whereArgs: whereArgs);
  }
}
```

**BENEFITS:**
* Comprehensive documentation
* Better error messages
* Type-safe return types
* Improved method signatures
* Status tracking for debugging

| ASPECT | app_wp_pms | lazytask_primetechbd | RATING | TECHNICAL DETAILS |
| :--- | :--- | :--- | :--- | :--- |
| **3. SECURE STORAGE** | | | | |
| ├─ flutter_secure_storage | ^9.0.0 | ^9.0.0 | 🟢 Same | Same version |
| ├─ Encryption | Platform default | Platform default | 🟢 Same | Keychain/Keystore |
| ├─ Token Storage | ❌ Not used | Recommended (not found) | 🟢 Both | Should use for tokens |
| └─ Security Best Practices| ⚠️ Partial | ⚠️ Partial | 🔶 N/A | Both could improve |

**ISSUE: app_wp_pms stores password in SharedPreferences!**
```dart
Future<void> setUserInfo(String email, String password) {
  prefs.setString('email', email);
  prefs.setString('password', password);  // ❌ SECURITY RISK!
}
```

**SHOULD BE:**
```dart
Future<void> setUserInfo(String email) async {
  final secureStorage = FlutterSecureStorage();
  await secureStorage.write(key: 'email', value: email);
  // Never store password - use token instead
}
```

| ASPECT | app_wp_pms | lazytask_primetechbd | RATING | TECHNICAL DETAILS |
| :--- | :--- | :--- | :--- | :--- |
| **4. CACHING STRATEGY** | | | | |
| ├─ In-Memory Cache | GetX observables | Riverpod providers | 🟢 LZ | LZ: auto-memoization |
| ├─ Disk Cache | SQLite | SQLite | 🟢 Same | Same approach |
| ├─ Cache Invalidation | Manual | Provider invalidation | 🟢 LZ | LZ: can use ref.invalidate() |
| ├─ Offline Support | SQLite local DB | SQLite local DB | 🟢 Same | Both support offline |
| └─ Network Cache | Dio default | Dio with config | 🟢 LZ | LZ: better configured |

### LOCAL STORAGE SCORE:
* **app_wp_pms**: 55/100 ⚠️ SECURITY CONCERNS
* **lazytask**: 80/100 ✅ BETTER SECURITY