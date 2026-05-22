# 📱 Mobile App Development — Best Practices (Complete Guide)
> Applies to Flutter, React Native, and native iOS/Android development

---

## 📌 Table of Contents
1. [Project Structure & Architecture](#1-project-structure--architecture)
2. [UI & UX Design Practices](#2-ui--ux-design-practices)
3. [State Management](#3-state-management)
4. [Networking & API Calls](#4-networking--api-calls)
5. [Local Storage & Data Persistence](#5-local-storage--data-persistence)
6. [Performance Optimization](#6-performance-optimization)
7. [Security](#7-security)
8. [Error Handling & Logging](#8-error-handling--logging)
9. [Navigation](#9-navigation)
10. [Testing](#10-testing)
11. [Accessibility](#11-accessibility)
12. [Offline Support](#12-offline-support)
13. [Push Notifications](#13-push-notifications)
14. [App Release & CI/CD](#14-app-release--cicd)
15. [Code Quality & Team Practices](#15-code-quality--team-practices)

---

## 1. Project Structure & Architecture

### ✅ DO
- **Choose an architecture before writing a single line** — Clean Architecture, MVVM, or MVC. Stick to it.
- **Organise by feature, not by type** for medium-large apps.
  ```
  ✅ feature-first (recommended for large apps)
  lib/
  ├── features/
  │   ├── auth/
  │   │   ├── data/        ← API calls, models, local DB
  │   │   ├── domain/      ← business logic, use cases
  │   │   └── presentation/← screens, widgets, state
  │   ├── home/
  │   └── profile/
  └── core/
      ├── network/
      ├── theme/
      └── utils/

  ✅ layer-first (ok for small apps)
  lib/
  ├── models/
  ├── services/
  ├── screens/
  └── widgets/
  ```
- **Separate concerns** — UI should never contain business logic or direct API calls.
- **Use a Repository pattern** — your screens talk to repositories, not directly to APIs or databases.
- **Use Dependency Injection** — don't instantiate services deep inside widgets; inject them from outside.
- **Create a `core/` or `shared/` folder** for utilities, constants, extensions, and shared widgets.
- **Keep `main.dart` minimal** — only initialisation, providers, and routing setup.

### ❌ DON'T
- Don't put API calls directly inside UI widgets or screens.
- Don't create one giant `utils.dart` file with everything in it.
- Don't skip architecture because "it's a small app" — small apps grow.
- Don't hardcode values (colors, strings, URLs) inline across the codebase.

---

## 2. UI & UX Design Practices

### ✅ DO
- **Follow platform conventions** — use Material Design for Android, Cupertino for iOS where appropriate.
- **Design for all screen sizes from day one** — phone, large phone, tablet.
  ```dart
  // Always use MediaQuery or LayoutBuilder, never hardcoded pixel values
  double width = MediaQuery.of(context).size.width;
  // ✅ responsive sizing
  double cardWidth = width > 600 ? 300 : width * 0.9;
  ```
- **Use relative sizing** (`Expanded`, `Flexible`, `FractionallySizedBox`) instead of fixed widths/heights.
- **Always handle all UI states**: loading, empty, error, and success.
  ```
  Every screen needs these 4 states:
  ┌─────────────┬────────────────────────────────┐
  │ Loading     │ Show shimmer or spinner        │
  │ Empty       │ Show helpful empty state UI    │
  │ Error       │ Show error message + retry btn │
  │ Success     │ Show actual content            │
  └─────────────┴────────────────────────────────┘
  ```
- **Show loading feedback** for every async action (button tap → API call).
- **Disable buttons while a request is in progress** to prevent double submissions.
- **Use shimmer/skeleton screens** instead of plain spinners for content that has a known shape.
- **Use consistent spacing** — define a spacing scale (4, 8, 12, 16, 24, 32px) and stick to it.
- **Support both light and dark themes** from the beginning, not as an afterthought.
- **Minimum touch target size: 48×48dp** for all interactive elements (Apple HIG / Material guidelines).
- **Show confirmation dialogs** before destructive actions (delete, logout).
- **Use optimistic UI** for actions that are very likely to succeed (like/unlike, toggle).

### ❌ DON'T
- Don't hardcode pixel values for layout — screens break on different devices.
- Don't show a blank white screen while loading.
- Don't silently fail — always inform the user when something goes wrong.
- Don't block the entire screen with a full-screen loader for minor operations.
- Don't use tiny text or color contrast that fails WCAG AA (4.5:1 ratio).

---

## 3. State Management

### ✅ DO
- **Choose one state management solution** and use it consistently — don't mix Provider + Bloc + GetX.
- **Keep UI state local** when it's only needed in one widget (`setState` is fine for that).
- **Keep global/shared state in a state manager** (Riverpod, Bloc, etc.), not in a widget.
- **Separate server state from client state** — use React Query / Riverpod AsyncNotifier for data that comes from APIs; use local state for UI-only things (form inputs, animation toggles).
- **Make state immutable** — use `copyWith` or `Freezed` instead of mutating objects.
  ```dart
  // ❌ BAD — mutating state
  user.name = 'New Name';
  setState(() {});

  // ✅ GOOD — immutable update
  setState(() {
    user = user.copyWith(name: 'New Name');
  });
  ```
- **Name your states clearly** — `AuthState.loading`, `AuthState.authenticated`, `AuthState.unauthenticated`, `AuthState.error`.
- **Handle loading and error states in every async state** — not just the success case.
- **Dispose controllers and subscriptions** when widgets are removed from the tree.

### ❌ DON'T
- Don't put business logic inside `setState` blocks.
- Don't store derived/computed values in state — compute them on the fly.
- Don't use global variables as a substitute for state management.
- Don't forget to cancel streams and subscriptions on `dispose`.

---

## 4. Networking & API Calls

### ✅ DO
- **Use a single HTTP client instance** (like Dio) configured with base URL, timeout, and interceptors.
  ```dart
  // ✅ Configure once, reuse everywhere
  final dio = Dio(BaseOptions(
    baseUrl: AppConfig.baseUrl,
    connectTimeout: Duration(seconds: 10),
    receiveTimeout: Duration(seconds: 15),
  ));
  ```
- **Use interceptors** for:
  - Adding auth tokens to every request
  - Refreshing tokens on 401 automatically
  - Logging requests/responses in debug mode
  - Global error handling
- **Always handle all HTTP error codes** — 400, 401, 403, 404, 422, 500, and network errors.
- **Show user-friendly error messages**, not raw error objects or stack traces.
- **Model every API response** — parse into typed Dart objects, never work with raw `Map<String, dynamic>` in your UI.
- **Implement retry logic** for network failures (with exponential backoff for transient errors).
- **Set timeouts** — never let a request hang indefinitely.
- **Cancel in-flight requests** when the user navigates away from a screen.
- **Use HTTPS only** — never allow plain HTTP in production.
- **Validate responses** — check that the structure you expect is actually what arrived.

### ❌ DON'T
- Don't create a new HTTP client instance on every API call.
- Don't expose raw exception messages (e.g. `SocketException` text) to the user.
- Don't make API calls directly from widget `build()` methods.
- Don't log sensitive data (tokens, passwords, PII) from interceptors.
- Don't trust or use data from the API without validation/parsing.

---

## 5. Local Storage & Data Persistence

### ✅ DO
- **Choose the right storage tool for the job:**
  ```
  ┌──────────────────────┬───────────────────────────────────┐
  │ Use Case             │ Tool                              │
  ├──────────────────────┼───────────────────────────────────┤
  │ Simple key-value     │ shared_preferences                │
  │ Sensitive data       │ flutter_secure_storage            │
  │ Structured/relational│ Drift (SQLite), sqflite           │
  │ Fast NoSQL/objects   │ Isar, Hive, ObjectBox             │
  │ Large files/media    │ File system (path_provider)       │
  └──────────────────────┴───────────────────────────────────┘
  ```
- **Never store sensitive data in SharedPreferences** — it's unencrypted plain storage.
- **Store tokens and credentials only in secure storage** (Keychain on iOS, Keystore on Android).
- **Define a clear data migration strategy** — what happens when your local schema changes after an app update?
- **Cache API responses locally** to support offline viewing of previously loaded data.
- **Set expiry on cached data** — don't serve month-old cached data as if it's fresh.
- **Clean up old/expired data** regularly to avoid storage bloat.

### ❌ DON'T
- Don't store JWT tokens or passwords in SharedPreferences or plain files.
- Don't store large binary data (images, videos) in SQLite — use the file system and store only the path.
- Don't assume local storage operations are instant — they're async, treat them as such.
- Don't skip migrations when changing your database schema.

---

## 6. Performance Optimization

### ✅ DO
- **Use `const` constructors everywhere possible** — they prevent widget rebuilds.
  ```dart
  // ✅ Flutter skips rebuilding this entirely if parent rebuilds
  const Text('Hello World')
  const SizedBox(height: 16)
  const Icon(Icons.home)
  ```
- **Use `ListView.builder` instead of `ListView`** for any list that could have more than ~20 items.
- **Use `RepaintBoundary`** around complex widgets that animate or update frequently.
- **Cache network images** with `cached_network_image` — never re-download the same image.
- **Optimise images** before displaying:
  - Use correct image resolution (don't load a 4000×3000 image for a 50×50 thumbnail)
  - Use WebP format where supported
  - Specify `cacheWidth` / `cacheHeight` on Image widgets
- **Paginate long lists** — never load 10,000 items at once.
- **Use `compute()`** (Isolate) for heavy operations: JSON parsing of large payloads, image processing, encryption.
- **Measure before optimising** — use Flutter DevTools Profiler to identify actual bottlenecks.
- **Avoid building expensive widgets in `build()`** — extract them or memoize.
- **Use `AutomaticKeepAliveClientMixin`** for tabs/pages that shouldn't be rebuilt when switching tabs.

### ❌ DON'T
- Don't build large widget trees synchronously in `build()`.
- Don't use `Column` inside `SingleChildScrollView` with unlimited-height children without a height constraint.
- Don't use `Opacity(opacity: 0)` to hide widgets — use `Visibility` or conditional rendering instead.
- Don't do heavy computation (sorting, parsing) on the main thread — use `compute()`.
- Don't load full-resolution images when thumbnails are needed.

---

## 7. Security

### ✅ DO
- **Store all secrets (API keys, tokens) securely:**
  - User tokens → `flutter_secure_storage`
  - Build-time secrets → environment variables, not committed to git
  - Never hardcode API keys in source code
- **Implement certificate pinning** for sensitive apps (banking, health) to prevent MITM attacks.
- **Obfuscate release builds:**
  ```bash
  # Flutter
  flutter build apk --obfuscate --split-debug-info=./debug-info
  ```
- **Validate ALL user input** on both client and server — client validation is UX, server validation is security.
- **Use HTTPS exclusively** — enforce it, never fall back to HTTP.
- **Implement proper session management:**
  - Short-lived access tokens (15 min – 1 hour)
  - Longer-lived refresh tokens with rotation
  - Clear all tokens on logout
- **Implement jailbreak/root detection** for high-security apps.
- **Prevent screenshots** for sensitive screens (payment, password) on both platforms.
- **Apply ProGuard/R8 rules** on Android to prevent reverse engineering.
- **Audit third-party packages** — only use well-maintained packages with active security patches.
- **Use `android:allowBackup="false"`** in AndroidManifest to prevent data backup leaks.

### ❌ DON'T
- Don't store tokens in SharedPreferences, AsyncStorage, or plain files.
- Don't log sensitive user data (passwords, tokens, PII) — even in debug mode.
- Don't ship debug builds to production.
- Don't trust data coming from the client — always re-validate on the server.
- Don't use outdated packages with known CVEs.
- Don't include `.env` files or `google-services.json` with production secrets in version control.

---

## 8. Error Handling & Logging

### ✅ DO
- **Handle errors at every layer:**
  ```
  Network Layer   → catch SocketException, TimeoutException, HTTP errors
  Repository Layer → wrap in custom Result/Either type
  State Layer     → expose error state to UI
  UI Layer        → show user-friendly message + retry option
  ```
- **Use a Result/Either type** to make errors explicit instead of throwing exceptions across the app.
  ```dart
  // ✅ Makes it impossible to forget error handling
  sealed class Result<T> {
    const Result();
  }
  class Success<T> extends Result<T> {
    final T data;
    const Success(this.data);
  }
  class Failure<T> extends Result<T> {
    final String message;
    const Failure(this.message);
  }
  ```
- **Set up a global error catcher** — in Flutter, use `FlutterError.onError` and `PlatformDispatcher.instance.onError`.
- **Use crash reporting in production** — Firebase Crashlytics or Sentry. Every uncaught error should be reported.
- **Log structured data** — include timestamps, screen names, user IDs (anonymised), and error codes.
- **Use log levels** — `debug` (dev only), `info`, `warning`, `error`, `fatal`.
- **Never swallow exceptions silently** — a bare `catch (e) {}` is almost always wrong.
- **Include error recovery** — every error screen should have a "Retry" or "Go Home" button.

### ❌ DON'T
- Don't show raw stack traces or technical error messages to users.
- Don't use `print()` for logging in production — use a proper logger.
- Don't catch generic `Exception` and ignore it silently.
- Don't forget to report errors to your crash monitoring tool.

---

## 9. Navigation

### ✅ DO
- **Use a dedicated routing package** (GoRouter for Flutter) — not ad-hoc navigation scattered in the codebase.
- **Define all routes in one place** — a `routes.dart` or `app_router.dart` file.
- **Implement deep linking from day one** — every main screen should be reachable via a URL.
- **Protect routes that require authentication:**
  ```dart
  redirect: (context, state) {
    final isLoggedIn = ref.read(authProvider).isAuthenticated;
    if (!isLoggedIn && state.matchedLocation != '/login') {
      return '/login';
    }
    return null;
  }
  ```
- **Handle the back button correctly** — especially on Android; always test the back stack behaviour.
- **Preserve scroll position and state** when navigating back to a list/tab.
- **Pass only IDs between routes**, not full objects — fetch the data in the destination screen.

### ❌ DON'T
- Don't push the same route multiple times on rapid taps — debounce navigation triggers.
- Don't pass large objects through navigation arguments — serialise to ID and refetch.
- Don't use context for navigation in business logic or service classes.

---

## 10. Testing

### ✅ DO
- **Write tests at all three levels:**
  ```
  Unit Tests       → business logic, repositories, use cases, pure functions
  Widget Tests     → individual screens and components
  Integration Tests→ critical user flows (login, checkout, etc.)
  ```
- **Test the unhappy path** — test errors, empty states, and edge cases, not just the happy path.
- **Mock external dependencies** (APIs, databases) in unit and widget tests.
- **Name tests descriptively:**
  ```dart
  // ✅ Clear intent
  test('returns error when network is unavailable', () { ... });
  test('shows empty state when list is empty', () { ... });

  // ❌ Meaningless
  test('test 1', () { ... });
  ```
- **Run tests in CI** — tests that only run locally are unreliable.
- **Write tests for every bug you fix** — regression tests prevent the same bug reappearing.
- **Aim for high coverage on business logic** — 80%+ on domain/data layers is a good target.

### ❌ DON'T
- Don't test implementation details — test behaviour and outcomes.
- Don't skip tests because "it's a mobile app" — mobile apps crash too.
- Don't write tests only after the app is complete — write them alongside features.

---

## 11. Accessibility

### ✅ DO
- **All interactive elements must have labels** — icons, image buttons, and custom widgets need semantic descriptions.
  ```dart
  // ✅ Screen readers will announce "Close button"
  IconButton(
    icon: Icon(Icons.close),
    tooltip: 'Close',
    onPressed: () => Navigator.pop(context),
  )
  ```
- **Meet colour contrast ratios:**
  - Normal text: minimum 4.5:1 contrast ratio
  - Large text (18pt+): minimum 3:1
- **Support dynamic text sizes** — never hardcode font sizes that ignore system text scale.
  ```dart
  // ✅ Respects user's accessibility font size
  Text('Hello', style: Theme.of(context).textTheme.bodyLarge)
  // ❌ Ignores system font scale
  Text('Hello', style: TextStyle(fontSize: 16))
  ```
- **Minimum tap target 48×48dp** for all buttons and interactive areas.
- **Test with a screen reader** — TalkBack (Android), VoiceOver (iOS) — on real devices.
- **Support keyboard navigation** on desktop and tablet.
- **Avoid conveying information by colour alone** — add icons or labels alongside colour indicators.
- **Don't auto-play audio/video** without user consent.

### ❌ DON'T
- Don't rely solely on gestures for critical actions — provide button alternatives.
- Don't use placeholder text as a label — it disappears when the user types.
- Don't disable the system font size scaling (`textScaleFactor`).

---

## 12. Offline Support

### ✅ DO
- **Design for offline first** — assume connectivity can drop at any time.
- **Detect network status** and clearly communicate it to the user.
  ```dart
  // Use connectivity_plus to listen for network changes
  // Show a banner when the user goes offline
  ```
- **Cache the last successful response** so users can still see data when offline.
- **Queue write operations** when offline and sync when connectivity returns.
- **Show clear offline indicators** — a banner, icon, or message that says "You're offline".
- **Distinguish between "no internet" and "server error"** — show different messages.
- **Test offline behaviour explicitly** — turn off WiFi and test every key flow.

### ❌ DON'T
- Don't crash when there's no internet — handle `SocketException` gracefully.
- Don't show a loading spinner forever when the device is offline — detect and show appropriate message.
- Don't silently discard user actions when offline — queue them.

---

## 13. Push Notifications

### ✅ DO
- **Request notification permission at the right moment** — not on app first open. Ask in context, when the user is about to receive a relevant notification.
- **Handle all three notification states:**
  ```
  App in foreground  → show in-app notification / snackbar
  App in background  → system notification tray
  App terminated     → cold start from notification tap
  ```
- **Always deep link from a notification** to the relevant content (not just the home screen).
- **Include notification payload** with enough data to navigate correctly without an extra API call.
- **Allow users to control notification preferences** inside the app settings.
- **Test on real devices** — notification behaviour differs significantly between simulators and real hardware.
- **Handle token refresh** — FCM tokens can expire; always save the latest token to your backend.

### ❌ DON'T
- Don't send irrelevant notifications — notification spam leads to uninstalls.
- Don't request notification permission on the very first screen.
- Don't send notifications without a clear way for the user to opt out.

---

## 14. App Release & CI/CD

### ✅ DO
- **Use build flavors/schemes** for dev, staging, and production environments.
  ```
  Each flavor has its own:
  - Base API URL
  - Firebase project
  - App icon (add "DEV" badge on dev builds)
  - Bundle ID / package name
  ```
- **Automate your build and release pipeline** — no manual builds for production.
- **Sign release builds correctly:**
  - Android: Keystore file, stored securely (not in git)
  - iOS: Distribution certificate + provisioning profile
- **Version your app semantically** — `MAJOR.MINOR.PATCH` + build number.
- **Maintain a `CHANGELOG.md`** — track what changed in every release.
- **Use TestFlight / Firebase App Distribution** for beta testing before production releases.
- **Monitor crash-free rate** after every release — roll back if it drops below acceptable level.
- **Test release builds before submitting** — debug and release builds can behave differently (obfuscation, tree shaking).
- **Set up app store review guidelines compliance checks** before submission.

### ❌ DON'T
- Don't commit keystore files or signing credentials to version control.
- Don't manually build and upload production releases — automate it.
- Don't submit to the app store without testing on real devices.
- Don't ship a new release without checking the crash dashboard 30 minutes after rollout.

---

## 15. Code Quality & Team Practices

### ✅ DO
- **Enforce a linting config** — `flutter_lints` at minimum, `very_good_analysis` for strict projects.
  ```yaml
  # analysis_options.yaml
  include: package:very_good_analysis/analysis_options.yaml
  ```
- **Run `dart format` automatically** — enforce consistent formatting via pre-commit hooks or CI.
- **Write self-documenting code** — clear names over comments. Add comments only when the "why" isn't obvious.
- **Document public APIs** with doc comments (`///`).
- **Use constants for magic values:**
  ```dart
  // ❌ BAD
  await Future.delayed(Duration(milliseconds: 300));
  if (score > 1000) { ... }

  // ✅ GOOD
  const kAnimationDuration = Duration(milliseconds: 300);
  const kHighScoreThreshold = 1000;
  ```
- **Keep functions and methods small** — a function that does one thing is easier to test and read.
- **Keep widget `build()` methods short** — extract sub-widgets into their own classes or methods.
- **Review code before merging** — every PR should be reviewed by at least one other developer.
- **Write meaningful commit messages** — use Conventional Commits format:
  ```
  feat: add biometric authentication
  fix: resolve crash on empty cart screen
  refactor: extract payment logic into PaymentService
  ```
- **Use feature flags** for incomplete features in the main branch — ship dark code safely.
- **Keep dependencies updated** — run `flutter pub outdated` regularly; patch security vulnerabilities promptly.
- **Document your architecture decisions** — a simple `docs/architecture.md` saves hours of onboarding.

### ❌ DON'T
- Don't merge code that breaks linting or tests.
- Don't use abbreviations or single-letter variable names (except loop indices).
- Don't leave TODO/FIXME comments older than one sprint unresolved.
- Don't copy-paste the same code in 3 places — extract it.
- Don't leave commented-out code in the repository — version control is for history.

---

## 🚦 Quick Reference Checklist

Use this before every feature PR and before every release:

### Feature Checklist
- [ ] All 4 UI states handled: loading, empty, error, success
- [ ] No hardcoded strings, colors, or values
- [ ] No API calls inside widgets
- [ ] Tokens/secrets not in SharedPreferences
- [ ] `const` used wherever possible
- [ ] Images cached, not re-downloaded
- [ ] Errors reported to Crashlytics / Sentry
- [ ] Unit tests written for business logic
- [ ] Tested on both Android and iOS
- [ ] Tested with slow/no internet connection
- [ ] Accessibility: labels on all icons, contrast checked

### Release Checklist
- [ ] Tested on real device (not just simulator)
- [ ] Release build tested (not debug)
- [ ] All environment variables set for production
- [ ] Version number and build number bumped
- [ ] Changelog updated
- [ ] Crash monitoring enabled
- [ ] App store screenshots and metadata updated if needed
- [ ] Beta tested via TestFlight / App Distribution

---

> 💡 **Golden Rule**: Build the app you would want to use yourself. If something feels slow, broken, confusing, or insecure to you as the developer — it definitely feels that way to your users.

---
*Mobile App Best Practices | Flutter & General | Production-ready development guide 🚀*
