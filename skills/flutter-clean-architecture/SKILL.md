---
name: flutter-clean-architecture
description: Use when writing or reviewing Flutter Clean Architecture.
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    category: flutter
    tags:
      - flutter
      - dart
      - clean-architecture
      - bloc
      - cubit
      - json-serializable
      - get-it
      - fpdart
      - dartz
---

# Flutter Clean Architecture (Feature-First)

## Overview

Enforces a strict, feature-first Flutter Clean Architecture using Dart 3 class modifiers, `Either<Failure, T>` functional error handling, `json_serializable` DTOs, and scoped `BlocProvider` dependency injection. Applies when creating, refactoring, or reviewing Flutter features, layers, Blocs/Cubits, repositories, use cases, or tests.

## When to Use

Use this skill whenever the user asks to:
* Scaffold or implement a new feature in a Flutter project.
* Write or refactor Data Layer classes (`DataSource`, `Service`, `@JsonSerializable()` DTOs, `RepositoryImpl`).
* Write or refactor Domain Layer classes (`Domain Model`, `Repository` contracts, `UseCase` classes).
* Write or refactor Presentation Layer classes (`Bloc`, `Cubit`, `sealed` States/Events, `<Feature>Page`, `<Feature>View`).
* Write unit tests or widget tests for a Flutter feature.
* Review Flutter code for architectural compliance.

## Directory Structure

Separate shared utilities and global dependency injection in `core/` from isolated `<feature>/` modules:

```text
lib/
├── core/
│   ├── di/
│   │   └── service_locator.dart # Global DI (GetIt.I) strictly for shared/core contracts
│   ├── error/
│   │   ├── exceptions.dart      # Common Exceptions thrown by datasources/services
│   │   └── failures.dart        # Common Failures (abstract base class) returned in Domain/Presentation
│   └── network/                 # Shared contracts & implementations (e.g., AppHttpClient)
└── features/
    └── <feature>/
        ├── data/
        │   ├── datasources/     # Contracts & implementations for CRUD/data persistence (API, Local DB)
        │   ├── services/        # Contracts & implementations for external gateways/SDKs (SMS, Email, etc.)
        │   ├── models/          # @JsonSerializable() DTOs with .toDomainModel() mapping
        │   └── repositories/    # Concrete implementations of domain repositories
        ├── domain/
        │   ├── models/          # Immutable domain entities/models
        │   ├── repositories/    # Abstract repository contracts
        │   └── usecases/        # Business logic orchestration (mandatory entry point for Blocs)
        └── presentation/
            ├── blocs/           # Blocs or Cubits (with sealed States/Events) scoped to the feature page
            ├── pages/           # Contains ONLY <Feature>Page (DI & BlocProvider wrapper)
            └── widgets/         # Contains <Feature>View and modular UI components composed inside it
```

## Core Architectural Rules & Dart 3 Modifiers

### 1. Class Modifiers (Mandatory)

* **Contracts / Interfaces:** Every contract (Domain Repositories, Data Sources, Services, and Shared Core Clients like `AppHttpClient`) **must** be declared as an `abstract interface class` to prevent inheritance (`extends`), enforce implementation (`implements`), and ensure every dependency is easily mockable or fakeable in tests.
* **Implementations:** Every concrete implementation (Repository implementations, Data Source/Service implementations, Use Cases, Models, Blocs/Cubits, concrete States/Events, and concrete Failures/Exceptions) **must** be declared as a `final class` to prevent subclassing and ensure strict encapsulation.
* **Shared Base Classes (`Failure`):** Base classes that share common properties or logic across subclasses—specifically `Failure` in `core/error/`—**must** be declared as an `abstract base class` so subclasses can use `extends` and inherit shared behavior.
* **Bloc States & Events (`sealed class`):** Base `State` and `Event` classes inside `presentation/blocs/` **must** be declared as a `sealed class`, with each concrete state or event subtype declared as a `final class`. This guarantees compile-time exhaustive pattern matching (`switch`) in the UI and Bloc event handlers.

### 2. Dependency Injection & Composition Strategy

* **Global DI (`core/di/`):** Service locator registration (`GetIt.I`) is reserved **strictly for shared cross-feature dependencies** (such as `AppHttpClient`, `AppLocalStorage`, or global auth/analytics contracts). Avoid naming custom clients `HttpClient` to prevent namespace collisions with `dart:io`. Do **not** register feature-specific Data Sources, Repositories, Use Cases, or Blocs in the global container.
* **Scoped Feature Composition (`<Feature>Page`):** Each feature composes its own dependency graph locally inside `presentation/pages/<feature>_page.dart` using `BlocProvider`. The `<Feature>Page` resolves shared global dependencies via `GetIt.I<...>()` (`get_it`), instantiates the feature's Data Sources/Services $\rightarrow$ Repository $\rightarrow$ Use Cases $\rightarrow$ Bloc/Cubit, and wraps `<Feature>View` (located in `presentation/widgets/<feature>_view.dart`).
* **Widget Testing Strategy (`<Feature>View`):** In widget tests, never pump `<Feature>Page` directly. Always pump `<Feature>View` wrapped in a `BlocProvider` providing a fake or mocked Bloc/Cubit.

### 3. Error Handling & Data Flow Pipeline (Futures & Streams)

Data and errors flow strictly through the layers as follows:

1. **Data Sources / Services:** Perform raw operations (HTTP calls, DB queries, WebSockets, SMS/Email gateways) inside `try/catch` blocks. They return raw Data Models (`DTOs`), `Stream<DTO>`, or primitives on success, or **throw custom `Exception`s** (defined in `core/error/exceptions.dart`) on failure.
2. **Repositories:** Wrap one-shot calls to Data Sources and Services inside `try/catch` blocks, catching `Exception`s, mapping DTOs to Domain Models via `.toDomainModel()`, and returning `Future<Either<Failure, T>>`. For reactive streams, Repositories transform `Stream<DTO>` into `Stream<Either<Failure, T>>`, catching stream errors and emitting them as `Left(Failure)` so the stream subscription never unexpectedly terminates on error.
3. **Use Cases:** Called exclusively by Blocs/Cubits. They execute business logic and return `Future<Either<Failure, T>>` or `Stream<Either<Failure, T>>`.
4. **Blocs / Cubits:** Must **always** call a `UseCase` (never a `Repository`, `DataSource`, or `Service` directly). They fold the `Either<Failure, T>` result (or use `emit.forEach` for `Stream<Either<Failure, T>>`) into `sealed` UI states.

## Procedure

When building or modifying a feature, execute the following steps in order:

### Step 1: Verify Shared Core Dependencies (`core/`)
Ensure base `Exception`, `Failure`, and shared contracts (`AppHttpClient`) exist in `core/`.

```dart
// core/error/exceptions.dart
final class ServerException implements Exception {
  final String message;
  const ServerException([this.message = 'A server error occurred.']);
}

// core/error/failures.dart
abstract base class Failure {
  final String message;
  const Failure(this.message);
}

final class ServerFailure extends Failure {
  const ServerFailure([super.message = 'Failed to communicate with the server.']);
}

// core/network/app_http_client.dart
abstract interface class AppHttpClient {
  Future<Map<String, dynamic>> post(String path, {Map<String, dynamic>? data});
}

// core/di/service_locator.dart
void setupSharedDependencies() {
  GetIt.I.registerLazySingleton<AppHttpClient>(() => AppHttpClientImpl());
}
```

### Step 2: Implement the Domain Layer (`domain/`)
Write pure Dart classes with zero dependencies on Flutter UI, external networking packages, or the Data layer:
1. **`domain/models/`:** Create immutable `final class` domain entities without JSON logic.
2. **`domain/repositories/`:** Create `abstract interface class` contracts returning `Future<Either<Failure, T>>` or `Stream<Either<Failure, T>>`.
3. **`domain/usecases/`:** Create `final class` use cases with a `call()` method that delegates to the repository contract.

```dart
// domain/models/user.dart
final class User {
  final String id;
  final String email;
  const User({required this.id, required this.email});
}

// domain/repositories/login_repository.dart
abstract interface class LoginRepository {
  Future<Either<Failure, User>> login({required String email, required String password});
  Stream<Either<Failure, User>> watchUserSession();
}

// domain/usecases/login_usecase.dart
final class LoginUseCase {
  final LoginRepository _repository;
  const LoginUseCase(this._repository);

  Future<Either<Failure, User>> call({required String email, required String password}) async {
    return await _repository.login(email: email, password: password);
  }
}
```

### Step 3: Implement the Data Layer (`data/`)
1. **`data/models/`:** Create `@JsonSerializable()` `final class` DTOs with `part '<name>.g.dart';` and a `.toDomainModel()` mapper method.
2. **`data/datasources/` or `data/services/`:**
   * Use **`datasources/`** (e.g., `LoginRemoteDataSource`, `UserLocalDataSource`) for CRUD data fetching/persistence.
   * Use **`services/`** (e.g., `EmailService`, `SMSService`, `LocationService`) for action-oriented external gateways or hardware SDKs.
   * Declare both an `abstract interface class` contract and a `final class` implementation. Wrap calls in `try/catch` and throw custom `Exception`s.
3. **`data/repositories/`:** Create the `final class` repository implementation. Catch `Exception`s from datasources/services, map DTOs via `.toDomainModel()`, and return `Either<Failure, T>`.

```dart
// data/models/user_dto.dart
import 'package:json_annotation/json_annotation.dart';
import '../../domain/models/user.dart';

part 'user_dto.g.dart';

@JsonSerializable()
final class UserDto {
  final String id;
  final String email;

  const UserDto({required this.id, required this.email});

  factory UserDto.fromJson(Map<String, dynamic> json) => _$UserDtoFromJson(json);

  Map<String, dynamic> toJson() => _$UserDtoToJson(this);

  User toDomainModel() => User(id: id, email: email);
}

// data/datasources/login_remote_datasource.dart
abstract interface class LoginRemoteDataSource {
  Future<UserDto> login({required String email, required String password});
  Stream<UserDto> watchUserSession();
}

final class LoginRemoteDataSourceImpl implements LoginRemoteDataSource {
  final AppHttpClient _client;
  const LoginRemoteDataSourceImpl(this._client);

  @override
  Future<UserDto> login({required String email, required String password}) async {
    try {
      final response = await _client.post('/login', data: {'email': email, 'password': password});
      return UserDto.fromJson(response);
    } catch (e) {
      throw const ServerException('Invalid credentials or server error.');
    }
  }

  @override
  Stream<UserDto> watchUserSession() async* {
    // Emits UserDto updates or throws ServerException on error
  }
}

// data/repositories/login_repository_impl.dart
final class LoginRepositoryImpl implements LoginRepository {
  final LoginRemoteDataSource _remoteDataSource;
  const LoginRepositoryImpl(this._remoteDataSource);

  @override
  Future<Either<Failure, User>> login({required String email, required String password}) async {
    try {
      final dto = await _remoteDataSource.login(email: email, password: password);
      return Right(dto.toDomainModel());
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    }
  }

  @override
  Stream<Either<Failure, User>> watchUserSession() async* {
    try {
      await for (final dto in _remoteDataSource.watchUserSession()) {
        yield Right(dto.toDomainModel());
      }
    } on ServerException catch (e) {
      yield Left(ServerFailure(e.message));
    }
  }
}
```

### Step 4: Implement the Presentation Layer (`presentation/`)
1. **`presentation/blocs/`:** Define `sealed class` States (and Events for Blocs) with `final class` subtypes. Inject Use Cases into the `final class` Bloc/Cubit.
2. **`presentation/pages/`:** Create `<Feature>Page` strictly to provide the `Bloc`/`Cubit` via `BlocProvider`, composing the feature dependencies locally and pulling shared contracts via `GetIt.I<...>()` before returning `<Feature>View`.
3. **`presentation/widgets/`:** Create `<Feature>View` and any child widgets inside `widgets/`.

```dart
// presentation/blocs/login_state.dart
sealed class LoginState {
  const LoginState();
}

final class LoginInitial extends LoginState {
  const LoginInitial();
}

final class LoginLoading extends LoginState {
  const LoginLoading();
}

final class LoginSuccess extends LoginState {
  final User user;
  const LoginSuccess(this.user);
}

final class LoginError extends LoginState {
  final String message;
  const LoginError(this.message);
}

// presentation/blocs/login_cubit.dart
final class LoginCubit extends Cubit<LoginState> {
  final LoginUseCase _loginUseCase;
  LoginCubit(this._loginUseCase) : super(const LoginInitial());

  Future<void> submitLogin(String email, String password) async {
    emit(const LoginLoading());
    final result = await _loginUseCase(email: email, password: password);
    result.fold(
      (failure) => emit(LoginError(failure.message)),
      (user) => emit(LoginSuccess(user)),
    );
  }
}

// presentation/pages/login_page.dart
final class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider<LoginCubit>(
      create: (_) => LoginCubit(
        LoginUseCase(
          LoginRepositoryImpl(
            LoginRemoteDataSourceImpl(
              GetIt.I<AppHttpClient>(),
            ),
          ),
        ),
      ),
      child: const LoginView(),
    );
  }
}

// presentation/widgets/login_view.dart
final class LoginView extends StatelessWidget {
  const LoginView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: BlocBuilder<LoginCubit, LoginState>(
        builder: (context, state) => switch (state) {
          LoginInitial() => const LoginFormWidget(),
          LoginLoading() => const Center(child: CircularProgressIndicator()),
          LoginSuccess(:final user) => Text('Welcome, ${user.email}'),
          LoginError(:final message) => Text('Error: $message'),
        },
      ),
    );
  }
}
```

## Pitfalls to Avoid

* **Never inject Repositories directly into Blocs/Cubits:** Blocs and Cubits must exclusively call Use Cases.
* **Never register feature-scoped classes in `GetIt.I`:** Only shared cross-feature dependencies (like `AppHttpClient`) belong in `core/di/`. Feature classes are composed inside `<Feature>Page`.
* **Never place `<Feature>View` inside `pages/`:** `pages/` holds only `<Feature>Page`. `<Feature>View` belongs in `presentation/widgets/<feature>_view.dart`.
* **Never pump `<Feature>Page` in widget tests:** Always pump `<Feature>View` wrapped in a `BlocProvider` with a fake or mocked Bloc/Cubit.
* **Never write manual JSON parsing in DTOs:** Always use `@JsonSerializable()` and run `build_runner`.
* **Never emit raw errors in Domain Streams:** Always return `Stream<Either<Failure, T>>` so stream subscriptions stay alive on failure.

## Verification

After generating or refactoring feature files:
1. Run `dart run build_runner build --delete-conflicting-outputs` to generate `*.g.dart` files for any `@JsonSerializable()` DTOs.
2. Run `flutter analyze` to verify there are no missing imports, type mismatches, or non-exhaustive `switch` statements on `sealed` Bloc states.
