# LazyTasks (PMS App) - Comprehensive Development Document

This document provides a complete technical analysis and structural overview of the **LazyTasks (PMS App)** project. It details the architecture, design patterns, state management, dependency injection, data handling, and code quality standards implemented across the codebase.

---

## 1. Architectural Pattern & Domain-Driven Design (DDD)
The project strictly adheres to a **Feature-First Clean Architecture** pattern combined with **Domain-Driven Design (DDD)** principles. The codebase is divided into two primary directories: `core` and `features`.

Each feature (e.g., `authentication`, `dashboard_page`, `my_task_page`, `my_zen`) operates as an isolated module containing its own micro-architecture:
- **Domain Layer:** Contains core business logic, entities, use cases, and repository interfaces (e.g., `features/my_task_page/domain/entities`, `usecases`, `repositories`). This layer is framework-agnostic.
- **Data Layer:** Handles external boundaries—remote/local data sources, DTOs (Data Transfer Objects), and concrete repository implementations.
- **Presentation Layer:** Manages UI components and State Notifiers.

---

## 2. Directory Structure Representation
Here is a high-level representation of the project's folder architecture, visualizing how Clean Architecture is rigorously enforced:

```text
lib/
 ├── core/                            # Application-wide generic configurations & utilities
 │    ├── database/                   # Local database configurations (Sqflite database_helper)
 │    ├── network/                    # Global network handlers
 │    │    └── dio/                   # Retrofit API Client & Dio interceptors
 │    ├── utils/                      # Helpers, Services, and global Configurations
 │    │    ├── constants/             # App-wide constants
 │    │    └── services/              # Contains service_locator.dart (GetIt configuration)
 │    └── widgets/                    # Globally shared, reusable UI components
 │
 ├── features/                        # Feature-driven modular separation
 │    ├── authentication/             # Example of a fully implemented Clean Architecture feature
 │    │    ├── data/                  # 1. Data Layer
 │    │    │    ├── data_sources/     # Remote (API) & Local API endpoints
 │    │    │    ├── models/           # Freezed & JSON Serializable DTOs
 │    │    │    └── repositories/     # Concrete implementations of Domain Repository interfaces
 │    │    ├── domain/                # 2. Domain Layer (Business Logic)
 │    │    │    ├── entities/         # Pure abstract models
 │    │    │    ├── repositories/     # Repository Interfaces (Contracts)
 │    │    │    └── usecases/         # Isolated functional logic units (e.g., LoginUseCase)
 │    │    └── presentation/          # 3. Presentation Layer (UI & State)
 │    │         ├── pages/            # Main Scaffold screen files
 │    │         ├── providers/        # Riverpod AsyncNotifiers & Sealed States
 │    │         └── widgets/          # Decoupled component-level UI widgets
 │    │
 │    ├── dashboard_page/             # Other isolated features built with the exact same structure...
 │    ├── my_task_page/
 │    ├── my_zen/
 │    └── ...
 │
 ├── l10n/                            # Localization (.arb files)
 ├── generated/                       # Auto-generated code files (Freezed/JSON Serializable outputs)
 ├── main.dart                        # Bootstrapping, DI initialization, and app execution entry point
 ├── my_app.dart                      # Core app context, Theme data, and Localization bindings
 └── translation.dart                 # GetX specific fallback translation handler
```

---

## 3. Deep Dive: State Management (Riverpod + Sealed Classes)
The project utilizes **Riverpod** (`flutter_riverpod`) but heavily extends it using robust Dart 3+ language features (like `sealed classes`) to enforce strict type-safety and eliminate unhandled states.

- **State Notifier Architecture:** Instead of using basic `StateProvider`, the project uses `AsyncNotifierProvider` (`AuthNotifier extends AsyncNotifier<AuthState>`). This seamlessly wraps the custom states within Riverpod's internal `AsyncData`, `AsyncLoading`, and `AsyncError` envelopes.
- **Sealed State Classes:** For example, in the Authentication feature (`auth_state.dart`), the state is defined as a `sealed class AuthState`. It dictates explicitly defined states:
  - `AuthInitial`
  - `AuthLoading`
  - `Authenticated(UserModel user)`
  - `Unauthenticated`
  - `AuthError(String message)`
- **Safety:** Using `sealed` classes ensures that when the UI checks the state (via `switch` statements), the compiler enforces exhaustive checking. There is zero chance of an unaccounted UI state.
- **Decoupled Execution:** The Notifiers act as the pure middleman. E.g., `AuthNotifier` never processes raw JSON. It calls `LoginUseCase`, which calls `AuthRepository`, ensuring absolute separation of concerns.

---

## 4. Deep Dive: UI Component Decoupling & Shared Widgets
To combat Flutter's notorious "Nested Hell", the presentation layer is highly fragmented.

- **Component Isolation:** Inside a feature's presentation folder, you'll find `pages` and `widgets` isolated explicitly (e.g., `features/authentication/presentation/widgets/login_widgets.dart`).
- **Granular Widgets:** Main page files (like `login_page.dart`) are kept extremely thin. They only house the primary Scaffold structure. Internal layout items (like a Login Title, Email Input Field, or Submit Buttons) are extracted into independent, stateless/stateful widgets residing in the `widgets/` subdirectory.
- **Core Shared Widgets:** Cross-feature reusable components (like standardized App Bars, generic Error Modals like `unauthorize_user_modal.dart`, Custom Buttons, or the `RetryModal`) are hoisted globally to `lib/core/widgets/`.

---

## 5. Dependency Injection (DI) & Service Location
Dependency Injection is robustly handled using **GetIt** (`get_it`) combined with annotation-driven **Injectable** (`injectable`).
- `setupServiceLocator()` dynamically links interface contracts to concrete implementations (e.g., binding `AuthRepository` to `AuthRepositoryImpl`).
- It manages object lifecycles as well, utilizing `registerLazySingleton` to ensure memory efficiency—instantiating services and HTTP clients only when explicitly requested.

---

## 6. Advanced Networking & API Integration
Network requests are managed securely using **Dio** and code-generated **Retrofit** clients (`lazy_tasks_api_client.dart`).
- **Pipelined Interceptors:**
  - `LoggingInterceptor`: Dumps granular request/response logs into the debug console.
  - `ConnectivityInterceptor`: Proactively checks network status via `connectivity_plus`. It halts API execution if offline and forces a `RetryModal` lock screen.
  - `AuthAndPathInterceptor`: Dynamically handles base URLs (fetched from `SessionManager`), injects JWT Bearer tokens, and automatically prepends standard WordPress REST namespaces (e.g., `/wp-json/lazytasks/api/v2/`).
  - `ErrorInterceptor`: A global catch-block for timeouts, 404s, and automatic session invalidation upon receiving HTTP 401.

---

## 7. Persistence & Deep Data Modeling
- **Data Modeling:** All models are constructed using **Freezed** and **JSON Serializable**. This enforces absolute immutability, built-in structural equality (`==`), and compile-safe JSON mapping.
- **Local Persistence Layers:**
  - **Relational Data:** `Sqflite` provides a structured SQLite database for heavy offline caching (initialized natively in `main.dart`).
  - **Key-Value Store:** `Shared Preferences` handles localized configurations (App Locale, Base URL configuration).
  - **Cryptographic Storage:** `Flutter Secure Storage` persists the highly sensitive JWT Authorization tokens in platform-native keychains.

---

## 8. Routing & Application Context Handling
Instead of a monolithic declarative router, the app utilizes **GetX Routing Mechanism** (`GetMaterialApp`) overlaid with named routes.
- `Get.to()`, `Get.back()`, and `Get.offAll()` govern navigation dynamically without requiring explicit `BuildContext` passing.
- A global `appNavigatorKey` binds the native Navigator state allowing asynchronous callbacks (like FCM triggers or dio interceptors) to push screens seamlessly.

---

## 9. Deep Tooling: FCM, Rich Text & Calendar Engines
- **Firebase Cloud Messaging (FCM):** The `NotificationService` and `PushNotificationService` modules handle granular push notification states (foreground, background, and terminated app states). Local notifications bridge FCM payloads to native OS popups.
- **Rich Document Viewing:** The app deploys `flutter_quill` for WYSIWYG text formatting, alongside parsing engines (`vsc_quill_delta_to_html`, `flutter_widget_from_html`) to natively render rich HTML outputs dynamically fetched from APIs.
- **Complex UI Rendering:** `CalendarControllerProvider` wraps the top-level app context enabling deeply integrated timeline scheduling (`calendar_view` & `syncfusion_flutter_calendar`).

---

## 10. Internationalization & Custom Theming
- **Multi-lingual Native Support:** The application sets up `GlobalMaterialLocalizations` and supports dynamic language swapping (`en`, `bn` through `.arb` configurations and `Languages()` GetX translations class).
- **Theme Engine:** `GoogleFonts.interTextTheme()` strictly enforces the `Inter` font family natively overriding the Material Design default typography across all Scaffold and Appbar widget trees globally.

---

## 11. Deep Dive: SOLID Principles Implementation
The LazyTasks codebase strictly adheres to the 5 pillars of object-oriented design (SOLID), which ensures long-term scalability and prevents tightly coupled legacy code:

- **S - Single Responsibility Principle (SRP):**
  - *Implementation:* Every class and file has exactly one reason to change. 
  - *Example:* The `AuthNotifier` strictly manages state. It does not parse JSON or hit endpoints. `LoginUseCase` strictly executes the business rule for logging in. `login_page.dart` strictly renders the UI scaffold, while inner widgets are offloaded to `login_widgets.dart`.

- **O - Open/Closed Principle (OCP):**
  - *Implementation:* Entities and network layers are open for extension but closed for modification.
  - *Example:* The Dio network client is extended through `Interceptors` (`ConnectivityInterceptor`, `ErrorInterceptor`). If a new authentication header logic is needed tomorrow, a new interceptor is simply chained into `network_module.dart` without ever altering the core `Dio` setup or the API client classes.

- **L - Liskov Substitution Principle (LSP):**
  - *Implementation:* Derived classes or interface implementations must be perfectly substitutable for their base types.
  - *Example:* In `service_locator.dart`, `AuthRepository` (interface) is registered. However, the concrete class `AuthRepositoryImpl` is provided. If the backend changes entirely tomorrow (e.g., from REST to GraphQL), you can swap in `AuthRepositoryGraphQLImpl` without changing a single line of code in the UseCases or the UI.

- **I - Interface Segregation Principle (ISP):**
  - *Implementation:* Clients should not be forced to depend upon interfaces they do not use.
  - *Example:* Instead of a monolithic `AppRepository` interface that holds 50+ methods, the domain layer breaks them down explicitly. `ILicenseRepository` handles license tasks, `MyTaskRepository` handles tasks, and `AuthRepository` handles auth. The UI only injects the interfaces it actually requires.

- **D - Dependency Inversion Principle (DIP):**
  - *Implementation:* High-level modules do not depend on low-level modules. Both depend on abstractions.
  - *Example:* The UI layer (`AuthNotifier`) completely ignores how data is fetched. It depends purely on the `LoginUseCase` abstraction. The `LoginUseCase` depends on the abstract `AuthRepository`. The lower-level concrete `AuthRemoteDataSource` fetches data and fulfills the repository contract. No layers are tightly coupled to the actual HTTP client directly.

---

## 12. Process & Data Execution Flow
1. **Bootstrapping (`main.dart`):** Loads EasyLoading configs, forces Portrait mode, boots `Sqflite`, and injects the `GetIt` registry.
2. **Context Binding (`my_app.dart`):** Traps the application inside `CalendarControllerProvider`, applies localized dictionaries, mounts the custom `Inter` typography, and routes to `SplashScreen`.
3. **Execution Pipeline:** 
   `User Input -> UI -> Riverpod Notifier -> Domain UseCase -> Domain Interface -> Data Repository -> Network/Dio (intercepted and logged) -> JSON parsing via Freezed -> UI State Update`.

---

## Conclusion
The **LazyTasks** application is an enterprise-grade architecture. Its adherence to **Domain-Driven Design (DDD)** isolates complex logic into explicit modules. Utilizing **Riverpod** alongside **GetIt** and **Freezed** drastically prevents state corruption and limits runtime errors. Deep OS integrations (FCM, global connectivity interception, and secure tokenization) signify a highly scalable and resilient engineering standard.
