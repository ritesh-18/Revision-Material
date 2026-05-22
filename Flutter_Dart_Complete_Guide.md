# 🐦 Flutter & Dart — Complete Learning Guide (Beginner → Advanced)

> Detailed explanations + code examples for every topic in the roadmap.

---

## 📌 Table of Contents

1. [Phase 1 — Dart Language Fundamentals](#phase-1--dart-language-fundamentals)
2. [Phase 2 — Flutter Fundamentals](#phase-2--flutter-fundamentals)
3. [Phase 3 — Styling & Theming](#phase-3--styling--theming)
4. [Phase 4 — Navigation & Routing](#phase-4--navigation--routing)
5. [Phase 5 — State Management](#phase-5--state-management)
6. [Phase 6 — Networking & Backend](#phase-6--networking--backend)
7. [Phase 7 — Local Storage & Persistence](#phase-7--local-storage--persistence)
8. [Phase 8 — Animations](#phase-8--animations)
9. [Phase 9 — Custom Painting & Graphics](#phase-9--custom-painting--graphics)
10. [Phase 10 — Platform & Device Features](#phase-10--platform--device-features)
11. [Phase 11 — Architecture & Code Quality](#phase-11--architecture--code-quality)
12. [Phase 12 — Testing](#phase-12--testing)
13. [Phase 13 — Performance & Optimization](#phase-13--performance--optimization)
14. [Phase 14 — Deployment & CI/CD](#phase-14--deployment--cicd)
15. [Phase 15 — Advanced Topics](#phase-15--advanced-topics)

---

# Phase 1 — Dart Language Fundamentals

Dart is the language Flutter is built on. It's a strongly-typed, object-oriented language with C-style syntax, optimized for client-side development with features like JIT compilation (for hot reload during dev) and AOT compilation (for fast native release builds).

---

## 1.1 Basics

### Variables & Data Types

Dart has both primitive and reference types. All variables are objects (even `int`).

```dart
int age = 25;                  // integer
double price = 99.99;          // floating-point
String name = 'Alex';          // string
bool isActive = true;          // boolean
dynamic anything = 'string';   // type can change
anything = 42;                 // valid
var inferred = 'hello';        // type inferred as String at compile time
Object obj = 'anything';       // can hold any object, but type-checked
```

**`var` vs `dynamic`:**
- `var x = 5` → x is inferred as `int`. Reassigning to a String fails at compile time.
- `dynamic y = 5` → y has no fixed type. Reassigning to String compiles fine. Use sparingly.

### Type Inference & Type Safety

Dart's sound type system means you can't assign a value of one type to a variable of an incompatible type. Inference (`var`) makes code concise while still type-safe.

```dart
var count = 0;       // inferred int
// count = 'hi';     // ❌ compile error
```

### String Interpolation & Multiline Strings

```dart
String name = 'Alex';
print('Hello, $name!');                  // Hello, Alex!
print('Sum: ${2 + 3}');                  // Sum: 5
print('${name.toUpperCase()}');          // ALEX

String multiline = '''
This string spans
multiple lines.
''';

String raw = r'C:\Users\alex';           // raw string, no escapes
```

### Comments

```dart
// Single-line comment

/* Multi-line
   comment */

/// Doc comment — shows in IDE intellisense and dartdoc output.
/// Use for public APIs.
int add(int a, int b) => a + b;
```

### Constants — `const` vs `final`

| | `final` | `const` |
|---|---|---|
| Set once | ✅ | ✅ |
| Value known at | Runtime | Compile time |
| Example | `final time = DateTime.now();` | `const pi = 3.14;` |
| Deep immutability | No (object can mutate) | Yes (compile-time canonical) |

```dart
final list1 = [1, 2, 3];       // list reference is final, but contents mutable
list1.add(4);                   // ✅ allowed

const list2 = [1, 2, 3];        // entire list is immutable
// list2.add(4);                // ❌ runtime error
```

Use `const` constructors aggressively in Flutter — they let the framework reuse the same widget instance, avoiding rebuilds.

### Null Safety

Dart 2.12+ has sound null safety. Variables can't hold `null` unless their type ends with `?`.

```dart
String name = 'Alex';        // non-nullable; must always have a value
String? maybe;                // nullable; defaults to null

print(maybe?.length);         // null-safe access — returns null if maybe is null
print(maybe!.length);         // bang operator — assert non-null; throws if null

late String lateInit;         // promise to assign before reading
lateInit = 'value';
```

`late` defers initialization but treats the variable as non-nullable. Use for fields initialized in `initState`, dependency injection, etc.

---

## 1.2 Operators

### Arithmetic, Relational, Logical

```dart
5 + 3        // 8
10 ~/ 3      // 3   (integer division)
10 % 3       // 1
2.5 * 4      // 10.0

5 == 5       // true
5 != 4       // true
5 > 3 && 2 < 4   // true
!false       // true
```

### Assignment

```dart
int x = 5;
x += 3;      // x = x + 3 → 8
x ??= 10;    // assign only if x is null
```

### Null-Aware

```dart
String? name;
print(name ?? 'Guest');       // 'Guest' (if null)
name ??= 'Default';            // assign if null
print(name?.length);           // null-safe call
```

### Cascade `..`

Lets you chain operations on the same object without repeating its name.

```dart
final paint = Paint()
  ..color = Colors.red
  ..strokeWidth = 2
  ..style = PaintingStyle.stroke;
// Equivalent to:
// final paint = Paint();
// paint.color = Colors.red;
// paint.strokeWidth = 2;
```

### Spread `...`

Expands a collection inside another.

```dart
var a = [1, 2, 3];
var b = [0, ...a, 4];           // [0, 1, 2, 3, 4]
var c = [0, ...?nullable];      // null-aware spread
```

### Ternary

```dart
final label = isActive ? 'On' : 'Off';
```

---

## 1.3 Control Flow

```dart
// if/else
if (score > 90) {
  print('A');
} else if (score > 80) {
  print('B');
} else {
  print('C');
}

// switch (Dart 3 pattern matching)
switch (status) {
  case 'open':  print('open');
  case 'closed': print('closed');
  default: print('unknown');
}

// for
for (var i = 0; i < 5; i++) print(i);

// for-in
for (var item in [1, 2, 3]) print(item);

// forEach
[1, 2, 3].forEach(print);

// while
while (x > 0) x--;

// do-while
do { x++; } while (x < 10);

// break / continue
for (var i = 0; i < 10; i++) {
  if (i == 5) break;
  if (i.isOdd) continue;
  print(i);   // 0, 2, 4
}
```

---

## 1.4 Functions

### Parameters

```dart
// Positional, required
int add(int a, int b) => a + b;

// Optional positional [in square brackets]
String greet(String name, [String? greeting]) {
  return '${greeting ?? 'Hello'}, $name';
}

// Named parameters {in curly braces}
void user({required String name, int age = 18}) { ... }

user(name: 'Alex');                        // age defaults to 18
user(name: 'Alex', age: 25);
```

`required` keyword forces the caller to provide the parameter.

### Arrow Functions

For single-expression functions:
```dart
int square(int x) => x * x;
final isEven = (int n) => n % 2 == 0;
```

### Anonymous Functions & Lambdas

```dart
final numbers = [1, 2, 3];
numbers.forEach((n) => print(n));
numbers.where((n) => n.isEven);

final greet = (String name) {
  return 'Hi, $name';
};
```

### Higher-Order Functions

Functions that take or return other functions.

```dart
List<R> map<T, R>(List<T> list, R Function(T) fn) {
  return [for (final x in list) fn(x)];
}
```

### Closures

A closure captures variables from its lexical scope.

```dart
Function makeCounter() {
  int count = 0;
  return () => ++count;
}

final counter = makeCounter();
counter(); // 1
counter(); // 2
```

### Recursive Functions

```dart
int factorial(int n) => n <= 1 ? 1 : n * factorial(n - 1);
```

---

## 1.5 Collections

### List

```dart
List<int> fixed = List.filled(3, 0);        // [0, 0, 0]
List<int> growable = [1, 2, 3];
growable.add(4);
growable.removeAt(0);
growable.length;
growable.first; growable.last;
```

### Set (unique elements)

```dart
Set<String> tags = {'flutter', 'dart', 'flutter'};
print(tags); // {flutter, dart}
tags.add('mobile');
tags.contains('dart');
```

### Map (key-value)

```dart
Map<String, int> ages = {'alex': 25, 'bob': 30};
ages['alex'];                       // 25
ages['carol'] = 22;
ages.containsKey('alex');
ages.forEach((k, v) => print('$k: $v'));
```

### Iterable Methods

```dart
final nums = [1, 2, 3, 4, 5];

nums.map((n) => n * 2);             // (2, 4, 6, 8, 10)
nums.where((n) => n.isEven);        // (2, 4)
nums.reduce((a, b) => a + b);       // 15 (sum)
nums.fold(100, (acc, n) => acc + n); // 115 (with initial value)
nums.any((n) => n > 4);             // true
nums.every((n) => n > 0);           // true
nums.expand((n) => [n, n * 10]);    // (1, 10, 2, 20, ...)
nums.firstWhere((n) => n > 3);      // 4
nums.toList(); nums.toSet();
```

### Collection-if / Collection-for / Spread

```dart
final theme = 'dark';
final widgets = [
  Text('Header'),
  if (theme == 'dark') Icon(Icons.dark_mode),
  for (var i = 0; i < 3; i++) Text('Item $i'),
  ...extraWidgets,
];
```

These are huge for declarative UI — conditional widgets without ugly ternaries.

---

## 1.6 Object-Oriented Programming

### Classes & Objects

```dart
class User {
  String name;
  int age;

  User(this.name, this.age);

  void greet() => print('Hi, I am $name');
}

final u = User('Alex', 25);
u.greet();
```

### Constructors

```dart
class Point {
  final double x, y;

  // Default
  Point(this.x, this.y);

  // Named
  Point.origin() : x = 0, y = 0;
  Point.fromMap(Map m) : x = m['x'], y = m['y'];

  // const constructor (compile-time)
  const Point.zero() : x = 0, y = 0;

  // Factory — can return cached instance
  factory Point.cached(double x, double y) {
    return _cache['$x,$y'] ??= Point(x, y);
  }

  static final _cache = <String, Point>{};
}
```

### Initializer Lists

Initialize `final` fields before the body runs:

```dart
class Circle {
  final double radius;
  final double area;

  Circle(this.radius) : area = 3.14 * radius * radius {
    print('Circle created');
  }
}
```

### Getters & Setters

```dart
class Rectangle {
  double width, height;
  Rectangle(this.width, this.height);

  double get area => width * height;
  set scale(double factor) {
    width *= factor;
    height *= factor;
  }
}

final r = Rectangle(2, 3);
print(r.area);     // 6
r.scale = 2;        // width=4, height=6
```

### Static Members

Belong to the class, not instances.

```dart
class Counter {
  static int _count = 0;
  static void increment() => _count++;
  static int get value => _count;
}

Counter.increment();
print(Counter.value);  // 1
```

### Inheritance & `super`

```dart
class Animal {
  String name;
  Animal(this.name);
  void speak() => print('Some sound');
}

class Dog extends Animal {
  Dog(String name) : super(name);

  @override
  void speak() {
    super.speak();         // calls Animal.speak()
    print('Woof!');
  }
}
```

### Abstract Classes

Cannot be instantiated; can have abstract (declared but not implemented) methods.

```dart
abstract class Shape {
  double area();         // abstract — subclass must implement
  void describe() => print('Area: ${area()}');
}

class Square extends Shape {
  final double side;
  Square(this.side);
  @override
  double area() => side * side;
}
```

### Interfaces (`implements`)

Dart has no separate `interface` keyword — every class is implicitly an interface.

```dart
class Logger {
  void log(String msg) => print(msg);
}

class FakeLogger implements Logger {
  @override
  void log(String msg) {}     // must implement EVERY public method
}
```

**Difference:** `extends` inherits implementation; `implements` only inherits the contract — you must reimplement everything.

### Mixins (`with`)

A way to reuse code in multiple class hierarchies without inheritance.

```dart
mixin Swimmer {
  void swim() => print('Swimming');
}

mixin Flyer {
  void fly() => print('Flying');
}

class Duck extends Animal with Swimmer, Flyer {
  Duck() : super('Duck');
}

final d = Duck();
d.swim();
d.fly();
```

Mixins are perfect for cross-cutting concerns (logging, animation tickers in Flutter — `SingleTickerProviderStateMixin`).

### Extension Methods

Add functionality to existing types without modifying them.

```dart
extension StringX on String {
  bool get isEmail => contains('@') && contains('.');
  String capitalize() => '${this[0].toUpperCase()}${substring(1)}';
}

'alex@x.com'.isEmail;     // true
'alex'.capitalize();      // 'Alex'
```

Common use: `int` → `Duration` extension (`5.seconds`), `BuildContext` shortcuts.

---

## 1.7 Advanced Dart

### Generics

Type parameters for reusable, type-safe code.

```dart
class Box<T> {
  T value;
  Box(this.value);
}

final intBox = Box<int>(5);
final stringBox = Box<String>('hi');

// Generic function
T first<T>(List<T> list) => list[0];

// Bounded type parameter
class Repo<T extends Comparable> { ... }
```

### Enums (Basic & Enhanced)

```dart
// Basic
enum Status { open, closed, pending }

// Enhanced (Dart 2.17+) — like real classes
enum Priority {
  low(1, '🟢'),
  medium(2, '🟡'),
  high(3, '🔴');

  final int weight;
  final String emoji;
  const Priority(this.weight, this.emoji);

  bool get isCritical => weight >= 3;
}

print(Priority.high.emoji);     // 🔴
print(Priority.high.isCritical); // true
```

### Typedef & Function Types

```dart
typedef IntCallback = void Function(int);
typedef JsonMap = Map<String, dynamic>;

void onClick(IntCallback callback) { callback(42); }
```

### Exception Handling

```dart
try {
  final data = parseJson(input);
} on FormatException catch (e) {
  print('Format error: $e');
} on Exception catch (e) {
  print('Generic exception: $e');
} catch (e, stack) {
  print('Anything: $e\n$stack');
  rethrow;                       // pass it up
} finally {
  cleanup();
}
```

### Custom Exceptions

```dart
class NotFoundException implements Exception {
  final String message;
  NotFoundException(this.message);
  @override
  String toString() => 'NotFoundException: $message';
}

throw NotFoundException('user 42');
```

### Iterators & Generators

```dart
// Sync generator
Iterable<int> countTo(int n) sync* {
  for (var i = 1; i <= n; i++) yield i;
}

// Async generator (returns Stream)
Stream<int> ticker(int n) async* {
  for (var i = 1; i <= n; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i;
  }
}
```

### Pattern Matching (Dart 3.0+)

```dart
// Destructuring
var (x, y) = (1, 2);
var {'name': name, 'age': age} = json;

// Switch with patterns
String describe(Object obj) => switch (obj) {
  int n when n > 0 => 'positive int',
  int() => 'non-positive int',
  String s => 'string of length ${s.length}',
  List(length: 0) => 'empty list',
  List(length: var l) => 'list of $l items',
  _ => 'unknown',
};
```

### Records (Dart 3.0+)

Lightweight, immutable, structurally-typed multi-value type.

```dart
(int, String) pair = (42, 'answer');
print(pair.$1);    // 42
print(pair.$2);    // 'answer'

// Named fields
({String name, int age}) user = (name: 'Alex', age: 25);
print(user.name);

// Return multiple values from a function
(double, double) splitAmount(double total) => (total * 0.9, total * 0.1);
final (net, fee) = splitAmount(100);
```

### Sealed Classes (Dart 3.0+)

A class hierarchy with a known, fixed set of subtypes — enables exhaustive switch.

```dart
sealed class Result<T> {}
class Success<T> extends Result<T> { final T data; Success(this.data); }
class Failure<T> extends Result<T> { final String error; Failure(this.error); }

String render(Result<int> r) => switch (r) {
  Success(data: final v) => 'OK: $v',
  Failure(error: final e) => 'ERR: $e',
  // No default needed — compiler knows all subtypes
};
```

---

## 1.8 Asynchronous Dart

### Future & async/await

A `Future<T>` represents a value that will be available later.

```dart
Future<String> fetchUser() async {
  await Future.delayed(Duration(seconds: 1));
  return 'Alex';
}

void main() async {
  final name = await fetchUser();
  print(name);
}
```

### Future Methods

```dart
// .then() — callback style (avoid in favor of async/await)
fetchUser().then((name) => print(name)).catchError((err) => print(err));

// Future.wait — parallel
final results = await Future.wait([fetchUser(), fetchSettings()]);

// Future.any — first to complete
final firstDone = await Future.any([fastServer(), backupServer()]);

// whenComplete — runs always, like finally
await fetchUser().whenComplete(() => print('done'));
```

### Streams

A `Stream<T>` is a sequence of async values (a Future of many values over time).

```dart
Stream<int> tickStream() async* {
  for (var i = 1; i <= 5; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i;
  }
}

await for (final tick in tickStream()) {
  print(tick);
}
```

**Single-subscription** (default): only one listener; rewind not allowed.
**Broadcast**: multiple listeners, used for events like button clicks.

### StreamController

Create a stream programmatically:

```dart
final controller = StreamController<int>.broadcast();

controller.stream.listen((v) => print('A: $v'));
controller.stream.listen((v) => print('B: $v'));

controller.add(1);  // A: 1, B: 1
controller.add(2);
controller.close();
```

### Isolates (basic concept)

Dart is single-threaded — async/await yields control on the same thread. **Isolates** are real OS-level threads with **no shared memory** — they communicate via messages.

```dart
import 'dart:isolate';

Future<int> heavyCompute(int n) async {
  return await Isolate.run(() {
    int sum = 0;
    for (var i = 0; i < n; i++) sum += i;
    return sum;
  });
}
```

In Flutter, use `compute(fn, arg)` from `flutter/foundation.dart` — convenience wrapper to offload heavy work without blocking the UI.

---

# Phase 2 — Flutter Fundamentals

## 2.1 Flutter Basics

### Flutter Architecture Overview

Flutter has three layers:

1. **Framework (Dart)** — Widgets, Material/Cupertino, animations, rendering pipeline. This is what you write apps with.
2. **Engine (C++)** — Skia/Impeller rendering, Dart VM, text layout, platform channels. Compiled per platform.
3. **Embedder (Platform-specific)** — entry point on each OS (Android: `FlutterActivity`, iOS: `FlutterViewController`, web: canvas, etc.).

Flutter draws **every pixel itself** via Skia/Impeller — it does **not** use native UI components. This gives identical UI across platforms but means you re-implement OS-style widgets (Material on Android, Cupertino on iOS).

### Project Structure

```
my_app/
├── lib/                    # Dart source
│   └── main.dart
├── pubspec.yaml            # dependencies, assets, fonts
├── android/                # Android-specific (Gradle, manifest)
├── ios/                    # iOS-specific (Xcode project)
├── web/                    # Web entry
├── windows/ macos/ linux/  # Desktop entries
├── test/                   # Tests
└── assets/                 # Images, fonts (declared in pubspec)
```

### `pubspec.yaml`

```yaml
name: my_app
description: A new Flutter project
version: 1.0.0+1   # version+build

environment:
  sdk: '>=3.4.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.6
  http: ^1.2.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/data/config.json
  fonts:
    - family: Poppins
      fonts:
        - asset: assets/fonts/Poppins-Regular.ttf
        - asset: assets/fonts/Poppins-Bold.ttf
          weight: 700
```

### `main()` & `runApp()`

```dart
void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      theme: ThemeData(useMaterial3: true),
      home: const HomePage(),
    );
  }
}
```

`runApp()` attaches the widget tree to the screen and starts the framework.

### `MaterialApp` vs `CupertinoApp`

- **MaterialApp** — Material Design (Android-style). Provides theming, routing, localization, scaffolding.
- **CupertinoApp** — iOS look. Same role but with Cupertino widgets.

Most apps use `MaterialApp` and adapt iOS-feel selectively. Or use `Platform.isIOS` checks for adaptive widgets.

### Hot Reload vs Hot Restart

- **Hot Reload (r)** — preserves state. Re-runs `build()` methods. Fast (sub-second). Doesn't re-run `main()`.
- **Hot Restart (R)** — resets state. Reruns `main()`. Slower but needed when changing `main()`, top-level static fields, or enum definitions.
- **Cold start** — full rebuild & restart.

---

## 2.2 Widget Basics

### Everything is a Widget

Widgets describe what their view should look like given their current configuration and state. They are immutable blueprints — Flutter creates Element objects from them, which manage state and links to RenderObjects.

### `StatelessWidget` vs `StatefulWidget`

```dart
// Stateless — no internal mutable state
class Greeting extends StatelessWidget {
  final String name;
  const Greeting({super.key, required this.name});

  @override
  Widget build(BuildContext context) => Text('Hi, $name');
}

// Stateful — mutable state
class Counter extends StatefulWidget {
  const Counter({super.key});
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () => setState(() => count++),
      child: Text('Count: $count'),
    );
  }
}
```

Use Stateless when the widget is purely a function of its inputs. Use Stateful when you need internal state changes (animations, form inputs, counters).

### Widget Lifecycle

For `StatefulWidget`:

```dart
class _MyState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    // Called once when inserted into the tree.
    // Subscribe to streams, init controllers, fetch initial data.
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // Called when an InheritedWidget this widget depends on changes.
  }

  @override
  void didUpdateWidget(MyWidget old) {
    super.didUpdateWidget(old);
    // Called when parent rebuilds with a new widget config.
    if (old.url != widget.url) refetch();
  }

  @override
  Widget build(BuildContext context) { /* ... */ }

  @override
  void dispose() {
    // Cleanup: cancel subscriptions, dispose controllers.
    controller.dispose();
    super.dispose();
  }
}
```

### `BuildContext`

A handle to the location of a widget in the widget tree. Lets you:
- Look up ancestor widgets/state (`Theme.of(context)`, `Navigator.of(context)`).
- Find ancestor `InheritedWidget`s.

```dart
final theme = Theme.of(context);
final size = MediaQuery.sizeOf(context);
Navigator.of(context).push(...);
```

`context` is unique per widget instance — never store one across builds.

### Widget Tree, Element Tree, Render Tree

Three parallel trees:
- **Widget tree** — your declarative description (immutable).
- **Element tree** — long-lived, manages widget lifecycle, holds references.
- **RenderObject tree** — does layout & painting.

Flutter reconciles a new widget tree against the existing element tree (like React diffing). When possible, it updates the existing element with new widget config instead of recreating.

### Keys

Identify widgets across rebuilds. Critical when reordering or inserting items in lists.

- **ValueKey** — based on a value: `ValueKey(user.id)`.
- **ObjectKey** — identity of an object.
- **UniqueKey** — always unique (forces recreation each build).
- **GlobalKey** — globally identifies a widget; lets you access its state from anywhere. Use sparingly (form keys, animations).

```dart
ListView(
  children: items.map((item) => ListTile(
    key: ValueKey(item.id),
    title: Text(item.name),
  )).toList(),
);
```

Without keys, removing item 0 may cause Flutter to repaint everything wrong as it re-pairs widgets to elements.

---

## 2.3 Layout Widgets

### `Container`

Swiss-army knife: padding, margin, decoration, alignment, constraints, color.

```dart
Container(
  width: 200,
  height: 100,
  margin: EdgeInsets.all(16),
  padding: EdgeInsets.symmetric(horizontal: 12, vertical: 8),
  decoration: BoxDecoration(
    color: Colors.blue,
    borderRadius: BorderRadius.circular(12),
    boxShadow: [BoxShadow(color: Colors.black26, blurRadius: 4)],
  ),
  child: Text('Hi'),
);
```

For better performance, use `Padding`/`SizedBox`/`DecoratedBox` separately when you don't need everything.

### `Row` & `Column`

Arrange children horizontally (Row) or vertically (Column).

```dart
Column(
  mainAxisAlignment: MainAxisAlignment.center,        // along the column (vertical)
  crossAxisAlignment: CrossAxisAlignment.stretch,     // across (horizontal)
  mainAxisSize: MainAxisSize.min,                      // wrap content vs fill
  children: [
    Text('Header'),
    Text('Subtitle'),
  ],
);

Row(
  children: [
    Expanded(flex: 2, child: Container(color: Colors.red)),
    Expanded(flex: 1, child: Container(color: Colors.blue)),
  ],
);
```

`mainAxisAlignment`: `start, end, center, spaceBetween, spaceAround, spaceEvenly`.
`crossAxisAlignment`: `start, end, center, stretch, baseline`.

### `Stack` & `Positioned`

Overlap widgets like CSS absolute positioning.

```dart
Stack(
  children: [
    Image.network('background.jpg'),
    Positioned(
      bottom: 16,
      right: 16,
      child: FloatingActionButton(onPressed: () {}, child: Icon(Icons.add)),
    ),
    Align(
      alignment: Alignment.center,
      child: Text('Centered'),
    ),
  ],
);
```

### `Expanded` & `Flexible`

Used inside Row/Column to share available space.

```dart
Row(
  children: [
    Expanded(child: Text('long...')),     // fills remaining
    Text('fixed'),
  ],
);
```

- **Expanded** = `Flexible(fit: FlexFit.tight)` — must fill the space.
- **Flexible** with `FlexFit.loose` — can be smaller if content allows.
- **flex** ratio decides share if multiple Expanded.

### `SizedBox` & `ConstrainedBox`

```dart
SizedBox(width: 100, height: 50, child: Text('Box'));
SizedBox(height: 16);   // simple spacer between widgets

ConstrainedBox(
  constraints: BoxConstraints(maxWidth: 600),
  child: Text('Bounded'),
);
```

### `AspectRatio` & `FractionallySizedBox`

```dart
AspectRatio(aspectRatio: 16 / 9, child: VideoPlayer(...));

FractionallySizedBox(widthFactor: 0.5, child: Container(color: Colors.red));
```

### `Padding`, `Align`, `Center`

```dart
Padding(padding: EdgeInsets.all(16), child: Text('Padded'));
Align(alignment: Alignment.topRight, child: Icon(Icons.star));
Center(child: Text('Centered'));
```

### `Wrap` & `Flow`

`Wrap` lays out children in a row that wraps to the next line when full — perfect for chips/tags.

```dart
Wrap(
  spacing: 8,
  runSpacing: 8,
  children: tags.map((t) => Chip(label: Text(t))).toList(),
);
```

`Flow` is lower-level for custom flow layouts; rarely needed.

### `LayoutBuilder` & `MediaQuery`

```dart
// MediaQuery — global screen info
final size = MediaQuery.sizeOf(context);
final padding = MediaQuery.paddingOf(context); // status bar, notch

// LayoutBuilder — constraints from parent
LayoutBuilder(
  builder: (context, constraints) {
    return constraints.maxWidth > 600
      ? WideLayout()
      : NarrowLayout();
  },
);
```

`LayoutBuilder` is best for component-level responsive logic; `MediaQuery` for global queries.

---

## 2.4 Common UI Widgets

### Text & RichText

```dart
Text(
  'Hello',
  style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold, color: Colors.blue),
  textAlign: TextAlign.center,
  maxLines: 2,
  overflow: TextOverflow.ellipsis,
);

// Mixed styles
Text.rich(
  TextSpan(
    text: 'By ',
    children: [
      TextSpan(text: 'Alex', style: TextStyle(fontWeight: FontWeight.bold)),
      TextSpan(text: ' on '),
      TextSpan(text: '2026-05-22', style: TextStyle(color: Colors.grey)),
    ],
  ),
);
```

### Image

```dart
Image.network('https://...jpg');
Image.asset('assets/logo.png');
Image.file(File('/path/to/img.png'));
Image.memory(bytes);

// With placeholder & error widget
FadeInImage.assetNetwork(
  placeholder: 'assets/loading.gif',
  image: 'https://...jpg',
);

// Better — cached_network_image
CachedNetworkImage(
  imageUrl: url,
  placeholder: (_, __) => CircularProgressIndicator(),
  errorWidget: (_, __, ___) => Icon(Icons.error),
);
```

### Buttons

```dart
ElevatedButton(onPressed: () {}, child: Text('Primary'));
TextButton(onPressed: () {}, child: Text('Subtle'));
OutlinedButton(onPressed: () {}, child: Text('Outlined'));
IconButton(onPressed: () {}, icon: Icon(Icons.favorite));
FloatingActionButton(onPressed: () {}, child: Icon(Icons.add));

// onPressed: null  → disabled state
```

Customise via `style: ElevatedButton.styleFrom(...)` or theme.

### TextField & TextFormField

```dart
final controller = TextEditingController();

TextField(
  controller: controller,
  decoration: InputDecoration(
    labelText: 'Email',
    hintText: 'name@example.com',
    prefixIcon: Icon(Icons.email),
    border: OutlineInputBorder(),
  ),
  keyboardType: TextInputType.emailAddress,
  textInputAction: TextInputAction.next,
  obscureText: false,    // true for password
);

// TextFormField — for use inside a Form
TextFormField(
  validator: (v) => v == null || v.isEmpty ? 'Required' : null,
  onSaved: (v) => email = v!,
);
```

Always `controller.dispose()` in your widget's `dispose()`.

### Checkbox, Radio, Switch, Slider

```dart
Checkbox(value: isChecked, onChanged: (v) => setState(() => isChecked = v!));
Switch(value: isOn, onChanged: (v) => setState(() => isOn = v));
Slider(value: vol, min: 0, max: 100, onChanged: (v) => setState(() => vol = v));

Radio<String>(
  value: 'a',
  groupValue: selected,
  onChanged: (v) => setState(() => selected = v),
);
```

### DropdownButton

```dart
DropdownButton<String>(
  value: selected,
  items: ['Apple','Banana','Cherry'].map((f) =>
    DropdownMenuItem(value: f, child: Text(f))
  ).toList(),
  onChanged: (v) => setState(() => selected = v),
);
```

### Date/Time Pickers

```dart
final date = await showDatePicker(
  context: context,
  initialDate: DateTime.now(),
  firstDate: DateTime(2000),
  lastDate: DateTime(2100),
);

final time = await showTimePicker(
  context: context,
  initialTime: TimeOfDay.now(),
);
```

### Chip

```dart
Chip(label: Text('Flutter'), avatar: Icon(Icons.code), onDeleted: () {});
FilterChip(label: Text('Active'), selected: true, onSelected: (v) {});
ChoiceChip(label: Text('Option'), selected: false, onSelected: (v) {});
```

### Tooltip & Badge

```dart
Tooltip(message: 'Refresh', child: IconButton(...));

Badge.count(count: 5, child: Icon(Icons.notifications));
```

---

## 2.5 Scrollable Widgets

### `SingleChildScrollView`

Wraps a single (possibly tall) widget to make it scrollable.

```dart
SingleChildScrollView(
  child: Column(children: [/* many widgets */]),
);
```

Use when content size is known/small. For long lists, prefer `ListView` (lazy).

### `ListView`

```dart
// Pre-built list — eager (only short lists)
ListView(children: [Text('A'), Text('B'), Text('C')]);

// Lazy builder — only builds visible items (use for >20 items)
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, i) => ListTile(title: Text(items[i].name)),
);

// With separators
ListView.separated(
  itemCount: items.length,
  itemBuilder: (_, i) => ListTile(title: Text(items[i].name)),
  separatorBuilder: (_, __) => Divider(),
);
```

### `GridView`

```dart
GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    childAspectRatio: 0.8,
    crossAxisSpacing: 8,
    mainAxisSpacing: 8,
  ),
  itemCount: products.length,
  itemBuilder: (_, i) => ProductCard(products[i]),
);

// Or by max extent
GridView.extent(maxCrossAxisExtent: 200, children: [...]);
```

### `CustomScrollView` & Slivers

For complex scroll effects (collapsing app bars, mixed grids/lists):

```dart
CustomScrollView(
  slivers: [
    SliverAppBar(
      expandedHeight: 200,
      flexibleSpace: FlexibleSpaceBar(title: Text('My App')),
      pinned: true,
    ),
    SliverList(
      delegate: SliverChildBuilderDelegate(
        (context, i) => ListTile(title: Text('Item $i')),
        childCount: 50,
      ),
    ),
    SliverGrid(
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 2),
      delegate: SliverChildListDelegate([...]),
    ),
  ],
);
```

### `PageView`

Horizontal swipeable pages (onboarding, image carousels).

```dart
PageView(
  controller: PageController(),
  children: [Page1(), Page2(), Page3()],
  onPageChanged: (i) => print('Now on $i'),
);
```

### `ReorderableListView`

Drag-and-drop reorder.

```dart
ReorderableListView(
  onReorder: (oldIdx, newIdx) {
    setState(() {
      if (newIdx > oldIdx) newIdx--;
      final item = items.removeAt(oldIdx);
      items.insert(newIdx, item);
    });
  },
  children: [for (final i in items) ListTile(key: ValueKey(i), title: Text(i))],
);
```

### `RefreshIndicator`

Pull-to-refresh.

```dart
RefreshIndicator(
  onRefresh: () async => await fetchData(),
  child: ListView.builder(...),
);
```

---

## 2.6 Structural Widgets

### `Scaffold`

The page skeleton — provides slots for app bar, body, drawer, FAB, bottom bar, snackbar.

```dart
Scaffold(
  appBar: AppBar(title: Text('Home')),
  body: Center(child: Text('Content')),
  floatingActionButton: FloatingActionButton(onPressed: () {}, child: Icon(Icons.add)),
  drawer: Drawer(child: ...),
  bottomNavigationBar: NavigationBar(destinations: [...]),
);
```

### `AppBar`

```dart
AppBar(
  title: Text('Home'),
  leading: IconButton(icon: Icon(Icons.menu), onPressed: () {}),
  actions: [IconButton(icon: Icon(Icons.search), onPressed: () {})],
  backgroundColor: Colors.blue,
  elevation: 4,
  bottom: TabBar(tabs: [Tab(text: 'A'), Tab(text: 'B')]),
);
```

### `Drawer`

```dart
Drawer(
  child: ListView(
    children: [
      DrawerHeader(child: Text('Menu')),
      ListTile(leading: Icon(Icons.home), title: Text('Home'), onTap: () {}),
      ListTile(leading: Icon(Icons.settings), title: Text('Settings'), onTap: () {}),
    ],
  ),
);
```

### `BottomNavigationBar` / `NavigationBar`

```dart
// Material 3 — NavigationBar
NavigationBar(
  selectedIndex: currentIndex,
  onDestinationSelected: (i) => setState(() => currentIndex = i),
  destinations: [
    NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
    NavigationDestination(icon: Icon(Icons.search), label: 'Search'),
    NavigationDestination(icon: Icon(Icons.person), label: 'Profile'),
  ],
);
```

### `TabBar` & `TabBarView`

```dart
DefaultTabController(
  length: 3,
  child: Scaffold(
    appBar: AppBar(bottom: TabBar(tabs: [Tab(text: 'A'), Tab(text: 'B'), Tab(text: 'C')])),
    body: TabBarView(children: [PageA(), PageB(), PageC()]),
  ),
);
```

### `BottomSheet`

```dart
showModalBottomSheet(
  context: context,
  isScrollControlled: true,
  builder: (context) => Container(
    padding: EdgeInsets.all(16),
    child: Column(mainAxisSize: MainAxisSize.min, children: [...]),
  ),
);
```

Persistent (non-modal):
```dart
Scaffold.of(context).showBottomSheet(...);
```

### `ExpansionTile`

```dart
ExpansionTile(
  title: Text('Click to expand'),
  children: [Text('Hidden content')],
);
```

### `Card` & `ListTile`

```dart
Card(
  elevation: 4,
  child: ListTile(
    leading: CircleAvatar(child: Text('A')),
    title: Text('Alex'),
    subtitle: Text('alex@x.com'),
    trailing: Icon(Icons.chevron_right),
    onTap: () {},
  ),
);
```

### `Divider`

```dart
Divider(thickness: 1, color: Colors.grey);
VerticalDivider();
```

---

## 2.7 Dialogs & Overlays

### `AlertDialog`

```dart
showDialog(
  context: context,
  builder: (_) => AlertDialog(
    title: Text('Delete?'),
    content: Text('This cannot be undone.'),
    actions: [
      TextButton(onPressed: () => Navigator.pop(context, false), child: Text('Cancel')),
      TextButton(onPressed: () => Navigator.pop(context, true), child: Text('Delete')),
    ],
  ),
);

// Read the result
final confirmed = await showDialog<bool>(context: context, builder: ...);
if (confirmed == true) doDelete();
```

### `SnackBar`

```dart
ScaffoldMessenger.of(context).showSnackBar(
  SnackBar(
    content: Text('Saved!'),
    action: SnackBarAction(label: 'Undo', onPressed: () {}),
    behavior: SnackBarBehavior.floating,
    duration: Duration(seconds: 3),
  ),
);
```

Use `ScaffoldMessenger` (newer) — works across pages, unlike `Scaffold.of` for SnackBars.

### `Overlay` & `OverlayEntry`

Low-level — show widgets above everything (custom tooltips, banners).

```dart
final entry = OverlayEntry(
  builder: (_) => Positioned(top: 50, left: 50, child: Material(child: Text('Floating'))),
);
Overlay.of(context).insert(entry);
// later: entry.remove();
```

### `PopupMenuButton`

```dart
PopupMenuButton<String>(
  onSelected: (v) => print(v),
  itemBuilder: (_) => [
    PopupMenuItem(value: 'edit', child: Text('Edit')),
    PopupMenuItem(value: 'delete', child: Text('Delete')),
  ],
);
```

---

## 2.8 Forms & Validation

```dart
class LoginForm extends StatefulWidget {
  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  final _formKey = GlobalKey<FormState>();
  String email = '', password = '';

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      autovalidateMode: AutovalidateMode.onUserInteraction,
      child: Column(children: [
        TextFormField(
          decoration: InputDecoration(labelText: 'Email'),
          validator: (v) {
            if (v == null || v.isEmpty) return 'Required';
            if (!v.contains('@')) return 'Invalid email';
            return null;
          },
          onSaved: (v) => email = v!,
        ),
        TextFormField(
          obscureText: true,
          decoration: InputDecoration(labelText: 'Password'),
          validator: (v) => (v?.length ?? 0) < 8 ? 'Min 8 chars' : null,
          onSaved: (v) => password = v!,
        ),
        ElevatedButton(
          onPressed: () {
            if (_formKey.currentState!.validate()) {
              _formKey.currentState!.save();
              submit(email, password);
            }
          },
          child: Text('Login'),
        ),
      ]),
    );
  }
}
```

### `FocusNode` & `FocusScope`

Manage keyboard focus programmatically.

```dart
final emailFocus = FocusNode();
final passwordFocus = FocusNode();

TextField(focusNode: emailFocus, onSubmitted: (_) => passwordFocus.requestFocus());

// Dismiss keyboard
FocusScope.of(context).unfocus();
```

### Input Formatters

```dart
TextField(
  inputFormatters: [
    FilteringTextInputFormatter.digitsOnly,
    LengthLimitingTextInputFormatter(10),
  ],
);
```

For credit cards, phones, dates — use the `mask_text_input_formatter` package.

---

# Phase 3 — Styling & Theming

## 3.1 Styling

### `BoxDecoration`

Used inside `Container` (and many other widgets via `decoration:`).

```dart
Container(
  decoration: BoxDecoration(
    color: Colors.white,
    borderRadius: BorderRadius.circular(16),
    border: Border.all(color: Colors.blue, width: 2),
    boxShadow: [
      BoxShadow(color: Colors.black26, blurRadius: 8, offset: Offset(0, 4)),
    ],
    gradient: LinearGradient(colors: [Colors.blue, Colors.purple]),
    image: DecorationImage(image: NetworkImage(url), fit: BoxFit.cover),
  ),
);
```

### `ShapeDecoration`

For non-rectangular shapes:

```dart
Container(
  decoration: ShapeDecoration(
    color: Colors.blue,
    shape: StadiumBorder(),     // pill shape
  ),
);
```

### `TextStyle`

```dart
TextStyle(
  fontSize: 16,
  fontWeight: FontWeight.w600,
  fontFamily: 'Poppins',
  color: Colors.black87,
  letterSpacing: 0.5,
  height: 1.4,                  // line height multiplier
  decoration: TextDecoration.underline,
  fontStyle: FontStyle.italic,
);
```

### Google Fonts

```yaml
dependencies:
  google_fonts: ^6.1.0
```

```dart
import 'package:google_fonts/google_fonts.dart';

Text('Hello', style: GoogleFonts.poppins(fontSize: 24));

// Apply to whole app
MaterialApp(
  theme: ThemeData(
    textTheme: GoogleFonts.poppinsTextTheme(),
  ),
);
```

### Custom Fonts via pubspec

```yaml
flutter:
  fonts:
    - family: Poppins
      fonts:
        - asset: assets/fonts/Poppins-Regular.ttf
        - asset: assets/fonts/Poppins-Bold.ttf
          weight: 700
        - asset: assets/fonts/Poppins-Italic.ttf
          style: italic
```

```dart
Text('Hi', style: TextStyle(fontFamily: 'Poppins'));
```

### Gradients

```dart
LinearGradient(
  begin: Alignment.topLeft,
  end: Alignment.bottomRight,
  colors: [Colors.blue, Colors.purple],
);

RadialGradient(
  center: Alignment.center,
  radius: 0.5,
  colors: [Colors.yellow, Colors.red],
);

SweepGradient(colors: [Colors.red, Colors.blue, Colors.red]);
```

### Shadows & Clipping

```dart
// Card-style shadow
BoxShadow(color: Colors.black12, blurRadius: 10, spreadRadius: 0, offset: Offset(0, 4));

// Clip to rounded rect
ClipRRect(
  borderRadius: BorderRadius.circular(16),
  child: Image.network(url),
);

ClipOval(child: Image.network(avatarUrl));

ClipPath(clipper: MyCustomClipper(), child: ...);
```

---

## 3.2 Theming

### `ThemeData` & `ColorScheme`

Centralize visual style at the app level:

```dart
MaterialApp(
  theme: ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
    textTheme: const TextTheme(
      titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.bold),
      bodyMedium: TextStyle(fontSize: 14),
    ),
    elevatedButtonTheme: ElevatedButtonThemeData(
      style: ElevatedButton.styleFrom(
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      ),
    ),
  ),
  darkTheme: ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple, brightness: Brightness.dark),
  ),
  themeMode: ThemeMode.system,    // follow OS setting
);
```

### Material 3

`useMaterial3: true` opts into the Material You design system: dynamic color, new component styling (NavigationBar, FilledButton), expressive typography.

### Light & Dark Theme

```dart
themeMode: ThemeMode.light    // force light
themeMode: ThemeMode.dark     // force dark
themeMode: ThemeMode.system   // follow OS
```

Set both `theme` and `darkTheme` so the framework picks the right one.

### `Theme.of(context)`

Read current theme inside any widget:

```dart
final theme = Theme.of(context);
Text('Title', style: theme.textTheme.titleLarge);
Container(color: theme.colorScheme.primary);
```

### `ThemeExtension` — custom theme data

For app-specific design tokens not in `ThemeData`:

```dart
class BrandColors extends ThemeExtension<BrandColors> {
  final Color success;
  final Color warning;
  const BrandColors({required this.success, required this.warning});

  @override
  BrandColors copyWith({Color? success, Color? warning}) =>
    BrandColors(success: success ?? this.success, warning: warning ?? this.warning);

  @override
  BrandColors lerp(BrandColors? other, double t) {
    if (other == null) return this;
    return BrandColors(
      success: Color.lerp(success, other.success, t)!,
      warning: Color.lerp(warning, other.warning, t)!,
    );
  }
}

// Register
theme: ThemeData(extensions: [BrandColors(success: Colors.green, warning: Colors.orange)]);

// Use
final brand = Theme.of(context).extension<BrandColors>()!;
Container(color: brand.success);
```

### Dynamic Color (Material You)

Pull system wallpaper colors (Android 12+) for adaptive themes:

```yaml
dependencies:
  dynamic_color: ^1.7.0
```

```dart
DynamicColorBuilder(
  builder: (light, dark) => MaterialApp(
    theme: ThemeData(colorScheme: light ?? ColorScheme.fromSeed(seedColor: Colors.blue)),
    darkTheme: ThemeData(colorScheme: dark ?? ColorScheme.fromSeed(seedColor: Colors.blue, brightness: Brightness.dark)),
  ),
);
```

---

## 3.3 Responsive Design

### `MediaQuery`

```dart
final media = MediaQuery.of(context);
final size = media.size;
final orientation = media.orientation;
final isLandscape = orientation == Orientation.landscape;
final topPadding = media.padding.top;   // status bar / notch
final viewInsets = media.viewInsets;     // keyboard
```

Use `MediaQuery.sizeOf(context)` (cheaper, only rebuilds on size change) when possible.

### `LayoutBuilder`

Constraints from the immediate parent, ideal for component-level responsiveness:

```dart
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth > 600) return WideLayout();
    return NarrowLayout();
  },
);
```

### Responsive Breakpoints

```dart
enum DeviceType { mobile, tablet, desktop }

DeviceType deviceType(BuildContext context) {
  final w = MediaQuery.sizeOf(context).width;
  if (w < 600) return DeviceType.mobile;
  if (w < 1024) return DeviceType.tablet;
  return DeviceType.desktop;
}
```

Material spec breakpoints: 600, 905, 1240, 1440. Library: `responsive_framework`.

### `OrientationBuilder`

```dart
OrientationBuilder(
  builder: (context, orientation) =>
    orientation == Orientation.portrait ? PortraitView() : LandscapeView(),
);
```

### `FittedBox` & `FractionallySizedBox`

`FittedBox` scales/fits its child to available space:
```dart
FittedBox(
  fit: BoxFit.scaleDown,
  child: Text('Auto-scale this long text'),
);
```

### Adaptive Widgets

Use Cupertino/Material based on platform:

```dart
import 'package:flutter/cupertino.dart';

Widget adaptiveSwitch(bool v, ValueChanged<bool> onChanged) {
  return Platform.isIOS
    ? CupertinoSwitch(value: v, onChanged: onChanged)
    : Switch(value: v, onChanged: onChanged);
}

// Or use platform-aware widgets from `flutter/material.dart` like `Switch.adaptive`
Switch.adaptive(value: v, onChanged: onChanged);
```

---

# Phase 4 — Navigation & Routing

## 4.1 Navigator 1.0 — Imperative

### Basic push/pop

```dart
// Push a new route
Navigator.of(context).push(
  MaterialPageRoute(builder: (_) => DetailPage(id: 42)),
);

// Pop back
Navigator.of(context).pop();

// Pop with a return value
Navigator.of(context).pop('selected');
final result = await Navigator.of(context).push(...);

// Replace current page (no back arrow)
Navigator.of(context).pushReplacement(MaterialPageRoute(builder: (_) => HomePage()));

// Push and remove all previous
Navigator.of(context).pushAndRemoveUntil(
  MaterialPageRoute(builder: (_) => HomePage()),
  (_) => false,   // remove all
);
```

### Named Routes

```dart
MaterialApp(
  routes: {
    '/': (_) => HomePage(),
    '/profile': (_) => ProfilePage(),
    '/settings': (_) => SettingsPage(),
  },
  initialRoute: '/',
);

Navigator.of(context).pushNamed('/profile');
```

### Passing Arguments

```dart
Navigator.of(context).pushNamed('/detail', arguments: {'id': 42});

// In DetailPage
final args = ModalRoute.of(context)!.settings.arguments as Map;
print(args['id']);
```

Type-safe: use `onGenerateRoute`:
```dart
onGenerateRoute: (settings) {
  if (settings.name == '/detail') {
    final args = settings.arguments as DetailArgs;
    return MaterialPageRoute(builder: (_) => DetailPage(id: args.id));
  }
  return null;
}
```

### Custom Route Transitions

```dart
Navigator.of(context).push(
  PageRouteBuilder(
    pageBuilder: (_, __, ___) => DetailPage(),
    transitionsBuilder: (_, anim, __, child) =>
      SlideTransition(
        position: Tween(begin: Offset(1, 0), end: Offset.zero).animate(anim),
        child: child,
      ),
    transitionDuration: Duration(milliseconds: 300),
  ),
);
```

### Back-button handling — `PopScope`

```dart
PopScope(
  canPop: !hasUnsavedChanges,
  onPopInvoked: (didPop) async {
    if (!didPop) {
      final confirm = await showDialog<bool>(...);
      if (confirm == true) Navigator.of(context).pop();
    }
  },
  child: MyForm(),
);
```

`WillPopScope` is the older API; `PopScope` is preferred in Flutter 3.12+.

---

## 4.2 GoRouter — Declarative Routing

The recommended modern navigation package. Single source of truth for routes; URL-friendly; deep linking out of the box.

### Setup

```yaml
dependencies:
  go_router: ^14.0.0
```

```dart
final router = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(path: '/', builder: (_, __) => HomePage()),
    GoRoute(
      path: '/users/:id',
      builder: (_, state) => UserPage(id: state.pathParameters['id']!),
    ),
    GoRoute(
      path: '/search',
      builder: (_, state) {
        final q = state.uri.queryParameters['q'];
        return SearchPage(query: q);
      },
    ),
  ],
);

MaterialApp.router(routerConfig: router);
```

### `go` vs `push` vs `pushNamed`

```dart
context.go('/profile');            // replace stack
context.push('/profile');          // push on top
context.pushReplacement('/profile');
context.pop();                      // pop top
context.goNamed('profile', pathParameters: {'id': '1'});
```

### Nested routes & `ShellRoute`

For persistent bottom navigation:

```dart
GoRouter(routes: [
  ShellRoute(
    builder: (context, state, child) => ScaffoldWithNavBar(child: child),
    routes: [
      GoRoute(path: '/home', builder: (_, __) => HomeTab()),
      GoRoute(path: '/search', builder: (_, __) => SearchTab()),
      GoRoute(path: '/profile', builder: (_, __) => ProfileTab()),
    ],
  ),
]);
```

The `ShellRoute` builder receives `child` — the current sub-route. Wrap it in your Scaffold with bottom nav. Each tab keeps its own navigation stack with `StatefulShellRoute.indexedStack`.

### Guards & Redirects

```dart
GoRouter(
  redirect: (context, state) {
    final isLoggedIn = AuthService.instance.isLoggedIn;
    final isLoggingIn = state.matchedLocation == '/login';
    if (!isLoggedIn && !isLoggingIn) return '/login';
    if (isLoggedIn && isLoggingIn) return '/';
    return null;   // no redirect
  },
  routes: [...],
);
```

### `GoRouterObserver`

```dart
class MyObserver extends NavigatorObserver {
  @override
  void didPush(Route route, Route? prev) {
    print('pushed ${route.settings.name}');
  }
}

GoRouter(observers: [MyObserver()], routes: [...]);
```

### Deep linking

Configure platform sides:

**Android** (`AndroidManifest.xml`):
```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <category android:name="android.intent.category.BROWSABLE"/>
  <data android:scheme="https" android:host="example.com"/>
</intent-filter>
```

**iOS** (`Info.plist`): add associated domain `applinks:example.com`.

GoRouter will route incoming URLs through its config automatically.

---

## 4.3 Other Navigation Patterns

### Bottom Navigation + State Preservation

Without preservation, switching tabs rebuilds them. Use `IndexedStack` or `StatefulShellRoute`:

```dart
class HomeShell extends StatefulWidget {
  @override
  State<HomeShell> createState() => _HomeShellState();
}

class _HomeShellState extends State<HomeShell> {
  int _index = 0;
  final _pages = [HomeTab(), SearchTab(), ProfileTab()];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(index: _index, children: _pages),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _index,
        onDestinationSelected: (i) => setState(() => _index = i),
        destinations: [
          NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(icon: Icon(Icons.search), label: 'Search'),
          NavigationDestination(icon: Icon(Icons.person), label: 'Profile'),
        ],
      ),
    );
  }
}
```

`IndexedStack` keeps all children built but only shows one — preserves scroll position, form state, etc.

### Nested Navigators

A tab can have its own Navigator stack so pushes stay within the tab:

```dart
Navigator(
  key: tabANavigatorKey,
  onGenerateRoute: (settings) => MaterialPageRoute(builder: (_) => TabAHome()),
);
```

### Drawer Navigation

Combine with named routes:
```dart
ListTile(
  title: Text('Settings'),
  onTap: () {
    Navigator.pop(context);      // close drawer first
    context.go('/settings');
  },
);
```

---

# Phase 5 — State Management

The core challenge: how does data flow through the widget tree, and how do widgets rebuild when data changes? Pick one solution per project and stick with it.

## 5.1 Built-in State Management

### `setState()` — local state

The simplest state container. Use for ephemeral UI state (form input, animation toggles).

```dart
class _CounterState extends State<Counter> {
  int count = 0;

  void increment() => setState(() => count++);

  @override
  Widget build(BuildContext context) =>
    ElevatedButton(onPressed: increment, child: Text('$count'));
}
```

Limitations: state is tied to one widget; passing down requires constructor parameters; lifting state up to share between widgets gets messy fast.

### `InheritedWidget`

Foundation of all Flutter state propagation (Theme, MediaQuery use it). Lets descendants read data without prop-drilling.

```dart
class CounterScope extends InheritedWidget {
  final int count;
  final VoidCallback increment;

  const CounterScope({
    required this.count,
    required this.increment,
    required super.child,
    super.key,
  });

  static CounterScope of(BuildContext context) =>
    context.dependOnInheritedWidgetOfExactType<CounterScope>()!;

  @override
  bool updateShouldNotify(CounterScope old) => count != old.count;
}

// In a descendant
final scope = CounterScope.of(context);
Text('${scope.count}');
```

You rarely write `InheritedWidget` directly — use Provider/Riverpod instead, which build on it.

### `InheritedModel`

Like `InheritedWidget`, but lets dependents subscribe to specific aspects — only rebuilt when relevant data changes. Niche.

---

## 5.2 Provider

The most beginner-friendly third-party state package. Built on `InheritedWidget` + `ChangeNotifier`.

```yaml
dependencies:
  provider: ^6.1.0
```

### `ChangeNotifier` & `ChangeNotifierProvider`

```dart
class CounterModel extends ChangeNotifier {
  int _count = 0;
  int get count => _count;

  void increment() {
    _count++;
    notifyListeners();    // triggers rebuilds of consumers
  }
}

// Expose to subtree
ChangeNotifierProvider(
  create: (_) => CounterModel(),
  child: MyApp(),
);
```

### `Consumer` / `context.watch` / `context.read` / `context.select`

```dart
// Consumer — rebuilds only this subtree on change
Consumer<CounterModel>(
  builder: (context, model, _) => Text('${model.count}'),
);

// context.watch — subscribes; rebuilds entire widget on change
Text('${context.watch<CounterModel>().count}');

// context.read — one-time read; no subscription (use in callbacks)
ElevatedButton(
  onPressed: () => context.read<CounterModel>().increment(),
  child: Text('+'),
);

// context.select — subscribe to only one field
final count = context.select<CounterModel, int>((m) => m.count);
```

Rule: **watch** in `build`; **read** in event handlers.

### `MultiProvider`

```dart
MultiProvider(
  providers: [
    ChangeNotifierProvider(create: (_) => AuthModel()),
    ChangeNotifierProvider(create: (_) => CartModel()),
    Provider<ApiClient>(create: (_) => ApiClient()),
  ],
  child: MyApp(),
);
```

### `ProxyProvider`

Compose providers that depend on other providers:
```dart
ProxyProvider<ApiClient, UserRepository>(
  update: (_, api, __) => UserRepository(api),
);
```

### `StreamProvider` & `FutureProvider`

Expose async data:
```dart
StreamProvider<int>(create: (_) => Stream.periodic(Duration(seconds:1), (i) => i), initialData: 0);
FutureProvider<User?>(create: (_) => fetchUser(), initialData: null);
```

---

## 5.3 Riverpod (Modern Recommended)

Compile-safe, no `BuildContext` required, easy testing, supports async first-class.

```yaml
dependencies:
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.3.0   # for code gen
dev_dependencies:
  riverpod_generator: ^2.4.0
  build_runner: ^2.4.0
```

### Setup

```dart
void main() {
  runApp(const ProviderScope(child: MyApp()));
}
```

`ProviderScope` is required at the root — holds the state of all providers.

### Provider types

```dart
// Provider — immutable value (DI for services)
final apiProvider = Provider<ApiClient>((ref) => ApiClient());

// StateProvider — single mutable value
final counterProvider = StateProvider<int>((ref) => 0);

// FutureProvider — async value
final userProvider = FutureProvider<User>((ref) async {
  final api = ref.watch(apiProvider);
  return api.fetchUser();
});

// StreamProvider — stream of values
final messagesProvider = StreamProvider<List<Message>>((ref) =>
  FirebaseFirestore.instance.collection('messages').snapshots().map((s) => s.docs.map(...).toList())
);

// NotifierProvider — encapsulated state with methods
class CounterNotifier extends Notifier<int> {
  @override
  int build() => 0;
  void increment() => state++;
  void decrement() => state--;
}
final counterProvider = NotifierProvider<CounterNotifier, int>(CounterNotifier.new);

// AsyncNotifierProvider — async state with methods
class TodoListNotifier extends AsyncNotifier<List<Todo>> {
  @override
  Future<List<Todo>> build() async => api.fetchTodos();

  Future<void> add(Todo t) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => api.add(t));
  }
}
```

### Reading providers in widgets

Replace `StatelessWidget` with `ConsumerWidget`:

```dart
class Counter extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Column(children: [
      Text('$count'),
      ElevatedButton(
        onPressed: () => ref.read(counterProvider.notifier).increment(),
        child: Text('+'),
      ),
    ]);
  }
}
```

### `ref.watch` / `ref.read` / `ref.listen`

- `ref.watch(p)` — subscribe; rebuilds when value changes. Use in `build`.
- `ref.read(p)` — one-time read, no rebuild. Use in callbacks.
- `ref.listen(p, (prev, next) => ...)` — side effects on change (show snackbar on error).

### Family Modifiers

Parameterize providers:
```dart
final userProvider = FutureProvider.family<User, String>((ref, id) async {
  return api.fetchUser(id);
});

// Use
final user = ref.watch(userProvider('123'));
```

### AutoDispose

Free state when no longer watched:
```dart
final searchProvider = FutureProvider.autoDispose.family<List<Result>, String>(
  (ref, query) async => api.search(query),
);
```

### Combining providers

```dart
final filteredTodosProvider = Provider<List<Todo>>((ref) {
  final todos = ref.watch(todosProvider);
  final filter = ref.watch(filterProvider);
  return todos.where((t) => filter.matches(t)).toList();
});
```

When `todosProvider` or `filterProvider` changes, `filteredTodosProvider` recomputes automatically.

### Code generation (`@riverpod`)

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
part 'todo.g.dart';

@riverpod
Future<List<Todo>> todos(TodosRef ref) async {
  final api = ref.watch(apiProvider);
  return api.fetchTodos();
}

@riverpod
class TodoList extends _$TodoList {
  @override
  FutureOr<List<Todo>> build() async => api.fetchTodos();

  Future<void> add(Todo t) async { ... }
}
```

Run `dart run build_runner watch`. Generates type-safe `todosProvider` and `todoListProvider`.

---

## 5.4 Bloc / Cubit

Event-driven state management with explicit state transitions. Popular in larger codebases for predictability.

```yaml
dependencies:
  flutter_bloc: ^8.1.0
```

### Cubit — simpler

```dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
}
```

### Bloc — events + states

```dart
// Events
sealed class CounterEvent {}
class Increment extends CounterEvent {}
class Decrement extends CounterEvent {}

// State
class CounterState { final int count; CounterState(this.count); }

// Bloc
class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(CounterState(0)) {
    on<Increment>((event, emit) => emit(CounterState(state.count + 1)));
    on<Decrement>((event, emit) => emit(CounterState(state.count - 1)));
  }
}
```

### Widgets

```dart
// Provide
BlocProvider(
  create: (_) => CounterCubit(),
  child: const CounterPage(),
);

// Build
BlocBuilder<CounterCubit, int>(
  builder: (context, count) => Text('$count'),
);

// Listen (side effects)
BlocListener<CounterCubit, int>(
  listener: (context, count) {
    if (count == 10) ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Ten!')));
  },
  child: ...
);

// Combined
BlocConsumer<CounterCubit, int>(
  listener: (ctx, state) => ...,
  builder: (ctx, state) => ...,
);

// Trigger
context.read<CounterCubit>().increment();
```

### `MultiBlocProvider`

```dart
MultiBlocProvider(
  providers: [
    BlocProvider(create: (_) => AuthBloc()),
    BlocProvider(create: (_) => CartBloc()),
  ],
  child: MyApp(),
);
```

### `BlocObserver`

Global logger for all bloc transitions:
```dart
class AppBlocObserver extends BlocObserver {
  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    log('${bloc.runtimeType} $transition');
  }
}

void main() {
  Bloc.observer = AppBlocObserver();
  runApp(MyApp());
}
```

### Hydrated Bloc

Auto-persist Bloc state across restarts:
```yaml
dependencies:
  hydrated_bloc: ^9.1.0
```

```dart
class CounterCubit extends HydratedCubit<int> {
  CounterCubit() : super(0);
  void increment() => emit(state + 1);

  @override
  int fromJson(Map<String, dynamic> json) => json['count'] as int;
  @override
  Map<String, dynamic> toJson(int state) => {'count': state};
}
```

---

## 5.5 Other State Solutions

### GetX

Tightly coupled state + navigation + DI:
```dart
class Counter extends GetxController {
  var count = 0.obs;          // observable
  void increment() => count++;
}

Get.put(Counter());
Obx(() => Text('${Get.find<Counter>().count}'));
Get.to(() => OtherPage());
```

Powerful but opinionated; couples your app to GetX heavily — harder to migrate away from. Avoid for new code; Riverpod is the safer choice.

### MobX

Observable + reactions, like Vue/Knockout:
```dart
class Counter = _Counter with _$Counter;
abstract class _Counter with Store {
  @observable int count = 0;
  @action void increment() => count++;
}
```

Requires code gen. Smaller community than Riverpod/Bloc.

### Redux

Single store + reducer pattern. Verbose; falling out of favor in Flutter. Use if your team already uses Redux on web/iOS/Android.

### Signals (`signals` package)

Fine-grained reactivity from SolidJS:
```dart
final count = signal(0);
count.value++;

Watch((_) => Text('${count.value}'));
```

New in Flutter ecosystem; rapidly gaining traction.

---

# Phase 6 — Networking & Backend

## 6.1 HTTP & APIs

### `http` package — minimal

```yaml
dependencies:
  http: ^1.2.0
```

```dart
import 'package:http/http.dart' as http;

// GET
final res = await http.get(Uri.parse('https://api.example.com/users'));
if (res.statusCode == 200) {
  final data = jsonDecode(res.body);
}

// POST with JSON body
final res = await http.post(
  Uri.parse('https://api.example.com/users'),
  headers: {'Content-Type': 'application/json', 'Authorization': 'Bearer $token'},
  body: jsonEncode({'name': 'Alex'}),
);
```

### `dio` — feature-rich

```yaml
dependencies:
  dio: ^5.4.0
```

```dart
final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
  connectTimeout: Duration(seconds: 10),
  receiveTimeout: Duration(seconds: 10),
  headers: {'Accept': 'application/json'},
));

// GET
final res = await dio.get('/users', queryParameters: {'page': 1});
final users = res.data;

// POST
await dio.post('/users', data: {'name': 'Alex'});

// File upload
final form = FormData.fromMap({
  'file': await MultipartFile.fromFile(path, filename: 'avatar.jpg'),
});
await dio.post('/upload', data: form);
```

### Interceptors

Add cross-cutting concerns once:
```dart
dio.interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) {
    options.headers['Authorization'] = 'Bearer ${AuthStore.token}';
    return handler.next(options);
  },
  onError: (err, handler) async {
    if (err.response?.statusCode == 401) {
      await AuthStore.refreshToken();
      // Retry
      return handler.resolve(await dio.fetch(err.requestOptions));
    }
    return handler.next(err);
  },
));
```

### Cancel tokens

```dart
final cancelToken = CancelToken();
dio.get('/long-query', cancelToken: cancelToken);
// later:
cancelToken.cancel('User left page');
```

### Retry & timeout

`dio_retry`, `dio_smart_retry`, or roll your own with interceptor:
```dart
class RetryInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    final retries = err.requestOptions.extra['retries'] ?? 0;
    if (retries < 3 && err.type == DioExceptionType.connectionTimeout) {
      err.requestOptions.extra['retries'] = retries + 1;
      await Future.delayed(Duration(seconds: 2 << retries));
      return handler.resolve(await dio.fetch(err.requestOptions));
    }
    return handler.next(err);
  }
}
```

---

## 6.2 JSON Handling

### `dart:convert` — manual

```dart
import 'dart:convert';

final json = jsonEncode({'name': 'Alex'});       // String
final map = jsonDecode(jsonString);              // dynamic Map
```

### Manual Model Classes

```dart
class User {
  final int id;
  final String name;
  final String? email;

  User({required this.id, required this.name, this.email});

  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['id'] as int,
    name: json['name'] as String,
    email: json['email'] as String?,
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    if (email != null) 'email': email,
  };
}
```

### `json_serializable` — code gen

```yaml
dependencies:
  json_annotation: ^4.9.0
dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.7.0
```

```dart
import 'package:json_annotation/json_annotation.dart';
part 'user.g.dart';

@JsonSerializable()
class User {
  final int id;
  final String name;
  final String? email;

  User({required this.id, required this.name, this.email});

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}
```

Run `dart run build_runner watch` to generate `user.g.dart`.

### `freezed` — immutable models + unions

```yaml
dependencies:
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0
dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.7.0
```

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required int id,
    required String name,
    String? email,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}

// Use
final u = User(id: 1, name: 'Alex');
final u2 = u.copyWith(name: 'Alex Smith');   // generated copyWith
print(u == User(id: 1, name: 'Alex'));        // true — value equality
```

### Nested & nullable JSON

```dart
@JsonSerializable(explicitToJson: true)
class Post {
  final User author;
  final List<Comment> comments;
  Post({required this.author, this.comments = const []});
  factory Post.fromJson(Map<String, dynamic> json) => _$PostFromJson(json);
  Map<String, dynamic> toJson() => _$PostToJson(this);
}
```

`explicitToJson: true` makes nested objects serialize properly.

---

## 6.3 Real-time Communication

### WebSocket

```dart
import 'package:web_socket_channel/web_socket_channel.dart';

final channel = WebSocketChannel.connect(Uri.parse('wss://api.example.com/ws'));

channel.sink.add('hello');           // send
channel.stream.listen((msg) {        // receive
  print('Got: $msg');
});

channel.sink.close();
```

### Socket.IO

```yaml
dependencies:
  socket_io_client: ^2.0.0
```

```dart
import 'package:socket_io_client/socket_io_client.dart' as io;

final socket = io.io('https://server.com', {
  'transports': ['websocket'],
  'autoConnect': false,
});
socket.connect();
socket.on('chat:msg', (data) => print(data));
socket.emit('chat:send', {'text': 'hi'});
```

### Server-Sent Events (SSE)

One-way stream from server, simpler than WebSocket. Use `flutter_client_sse` or read the stream manually via `dio` with `responseType: ResponseType.stream`.

---

## 6.4 Firebase

### Setup with FlutterFire CLI

```bash
dart pub global activate flutterfire_cli
flutterfire configure
```

This generates `firebase_options.dart` and configures all platforms.

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(MyApp());
}
```

### Firebase Auth

```dart
import 'package:firebase_auth/firebase_auth.dart';

// Sign up
final cred = await FirebaseAuth.instance.createUserWithEmailAndPassword(
  email: email, password: password,
);

// Sign in
await FirebaseAuth.instance.signInWithEmailAndPassword(
  email: email, password: password,
);

// Sign out
await FirebaseAuth.instance.signOut();

// Listen to auth changes
FirebaseAuth.instance.authStateChanges().listen((User? user) {
  if (user == null) goToLogin();
  else goToHome();
});

// Google sign-in
final google = GoogleSignIn();
final account = await google.signIn();
final auth = await account!.authentication;
final credential = GoogleAuthProvider.credential(
  accessToken: auth.accessToken, idToken: auth.idToken,
);
await FirebaseAuth.instance.signInWithCredential(credential);
```

### Cloud Firestore

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

final db = FirebaseFirestore.instance;

// Add
await db.collection('users').doc(uid).set({'name': 'Alex'});

// Read once
final snap = await db.collection('users').doc(uid).get();
print(snap.data());

// Query
final q = await db.collection('posts')
  .where('authorId', isEqualTo: uid)
  .orderBy('createdAt', descending: true)
  .limit(20)
  .get();

// Realtime listener
db.collection('messages').snapshots().listen((snap) {
  for (final doc in snap.docs) print(doc.data());
});

// Update
await db.collection('users').doc(uid).update({'lastLogin': FieldValue.serverTimestamp()});

// Delete
await db.collection('users').doc(uid).delete();

// Transaction
await db.runTransaction((tx) async {
  final snap = await tx.get(db.collection('counters').doc('main'));
  tx.update(snap.reference, {'count': (snap.data()!['count'] as int) + 1});
});
```

### Firebase Storage

```dart
import 'package:firebase_storage/firebase_storage.dart';

final ref = FirebaseStorage.instance.ref('avatars/$uid.jpg');
await ref.putFile(File(localPath));
final url = await ref.getDownloadURL();
```

### FCM (Push Notifications)

```dart
import 'package:firebase_messaging/firebase_messaging.dart';

await FirebaseMessaging.instance.requestPermission();
final token = await FirebaseMessaging.instance.getToken();   // send to backend

FirebaseMessaging.onMessage.listen((message) {
  // foreground notification
  showLocalNotification(message);
});

FirebaseMessaging.onBackgroundMessage(backgroundHandler);
```

### Analytics & Crashlytics

```dart
FirebaseAnalytics.instance.logEvent(name: 'purchase', parameters: {'value': 9.99});
FirebaseCrashlytics.instance.recordError(error, stack);
FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterError;
```

### Remote Config

```dart
final rc = FirebaseRemoteConfig.instance;
await rc.setDefaults({'maintenance_mode': false});
await rc.fetchAndActivate();
final maint = rc.getBool('maintenance_mode');
```

### Security Rules

Lock down access at the Firestore/Storage layer:
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }
  }
}
```

Never trust the client — Firestore security rules are your primary defense.

---

## 6.5 Other Backend Options

### Supabase

Open-source Firebase alternative (Postgres-backed):
```yaml
dependencies:
  supabase_flutter: ^2.5.0
```

```dart
await Supabase.initialize(url: 'https://...supabase.co', anonKey: '...');
final supabase = Supabase.instance.client;

await supabase.auth.signUp(email: ..., password: ...);
final data = await supabase.from('posts').select().eq('userId', uid);
```

### Appwrite

Self-hostable BaaS — similar feature set; clean Flutter SDK.

### REST + Node/Express backend

Combine Dio + your own API (see Phase 6.1). Use `freezed`+`json_serializable` for typed models. Add an Interceptor for auth.

### GraphQL

```yaml
dependencies:
  graphql_flutter: ^5.1.0
```

```dart
final client = GraphQLClient(
  link: HttpLink('https://api.example.com/graphql'),
  cache: GraphQLCache(),
);

final res = await client.query(QueryOptions(
  document: gql('query { user(id: 1) { name email } }'),
));
```

For complex apps, use `ferry` (code-gen, typed) over `graphql_flutter`.

---

# Phase 7 — Local Storage & Persistence

## 7.1 Simple Storage

### `shared_preferences` — key-value

For small, non-sensitive prefs (theme mode, onboarding flag).

```yaml
dependencies:
  shared_preferences: ^2.2.0
```

```dart
final prefs = await SharedPreferences.getInstance();
await prefs.setString('username', 'alex');
await prefs.setBool('isDarkMode', true);
await prefs.setInt('launchCount', 5);
await prefs.setStringList('recent', ['a','b']);

final name = prefs.getString('username');
await prefs.remove('username');
await prefs.clear();
```

Synchronous reads after `.getInstance()` — fast. For complex objects, JSON-encode them first.

### `flutter_secure_storage` — encrypted

For tokens, credentials. Uses Keychain (iOS) / Keystore (Android).

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

```dart
const storage = FlutterSecureStorage();
await storage.write(key: 'jwt', value: token);
final jwt = await storage.read(key: 'jwt');
await storage.delete(key: 'jwt');
```

Don't put massive blobs here — it's slower than `shared_preferences`. Use it specifically for secrets.

---

## 7.2 Local Databases

### `sqflite` — raw SQLite

```yaml
dependencies:
  sqflite: ^2.3.0
  path: ^1.9.0
```

```dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

final dbPath = await getDatabasesPath();
final db = await openDatabase(
  join(dbPath, 'app.db'),
  version: 1,
  onCreate: (db, v) async {
    await db.execute('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');
  },
);

await db.insert('users', {'name': 'Alex'});
final rows = await db.query('users', where: 'name = ?', whereArgs: ['Alex']);
await db.update('users', {'name': 'A'}, where: 'id = ?', whereArgs: [1]);
await db.delete('users', where: 'id = ?', whereArgs: [1]);
```

Verbose; raw SQL strings; no compile-time type safety. Use only if you need raw SQL.

### Drift — type-safe SQLite ORM

```yaml
dependencies:
  drift: ^2.16.0
  drift_flutter: ^0.1.0
  path_provider: ^2.1.0
dev_dependencies:
  drift_dev: ^2.16.0
  build_runner: ^2.4.0
```

```dart
@DataClassName('User')
class Users extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().withLength(min: 1, max: 50)();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
}

@DriftDatabase(tables: [Users])
class AppDb extends _$AppDb {
  AppDb() : super(_openConnection());

  @override
  int get schemaVersion => 1;

  Future<List<User>> getAllUsers() => select(users).get();
  Future<void> addUser(UsersCompanion u) => into(users).insert(u);
  Stream<List<User>> watchAllUsers() => select(users).watch();   // reactive!
}
```

Built-in streams for reactive queries. Great if you like SQL + type safety.

### Isar — fast NoSQL embedded

```yaml
dependencies:
  isar: ^3.1.0
  isar_flutter_libs: ^3.1.0
  path_provider: ^2.1.0
dev_dependencies:
  isar_generator: ^3.1.0
  build_runner: ^2.4.0
```

```dart
@collection
class User {
  Id id = Isar.autoIncrement;
  @Index() late String name;
  late DateTime createdAt;
}

final isar = await Isar.open([UserSchema], directory: dir.path);

await isar.writeTxn(() async {
  await isar.users.put(User()..name = 'Alex'..createdAt = DateTime.now());
});

final all = await isar.users.where().findAll();
final found = await isar.users.filter().nameEqualTo('Alex').findFirst();
isar.users.watchLazy().listen((_) => print('changed'));
```

Extremely fast (faster than Hive for queries). Modern, popular.

### ObjectBox

Similar to Isar — very fast NoSQL. Slightly bigger binary; mature.

### Hive — lightweight key-value NoSQL

```yaml
dependencies:
  hive_flutter: ^1.1.0
```

```dart
await Hive.initFlutter();
final box = await Hive.openBox('settings');
box.put('theme', 'dark');
final theme = box.get('theme');
```

Good for small datasets. For larger / queryable data, prefer Isar/Drift.

---

## 7.3 File System

### `path_provider` — system paths

```yaml
dependencies:
  path_provider: ^2.1.0
```

```dart
import 'package:path_provider/path_provider.dart';

final docs = await getApplicationDocumentsDirectory();   // persistent app data
final cache = await getTemporaryDirectory();              // can be cleared by OS
final external = await getExternalStorageDirectory();     // Android only
```

### File operations

```dart
import 'dart:io';

final file = File('${docs.path}/notes.txt');
await file.writeAsString('Hello');
final content = await file.readAsString();
final bytes = await file.readAsBytes();
await file.delete();
print(await file.exists());
```

For streaming large files: `file.openRead()` / `file.openWrite()`.

### Caching images

```yaml
dependencies:
  cached_network_image: ^3.3.0
```

```dart
CachedNetworkImage(
  imageUrl: 'https://...',
  placeholder: (_, __) => CircularProgressIndicator(),
  errorWidget: (_, __, ___) => Icon(Icons.error),
);
```

Auto-caches to disk; serves from cache on subsequent requests; massive UX win.

---

# Phase 8 — Animations

## 8.1 Implicit Animations

"Tell Flutter the new value; it animates for you." Simplest path.

### `AnimatedContainer`

```dart
class _MyState extends State<MyWidget> {
  bool big = false;

  @override
  Widget build(BuildContext context) => GestureDetector(
    onTap: () => setState(() => big = !big),
    child: AnimatedContainer(
      duration: Duration(milliseconds: 300),
      curve: Curves.easeInOut,
      width: big ? 200 : 100,
      height: big ? 200 : 100,
      color: big ? Colors.blue : Colors.red,
    ),
  );
}
```

### Other Animated*

- `AnimatedOpacity` — fade.
- `AnimatedPadding` — padding changes.
- `AnimatedAlign` — alignment changes.
- `AnimatedDefaultTextStyle` — text style transitions.
- `AnimatedPositioned` (inside Stack).
- `AnimatedCrossFade` — switch between two children with a fade.
- `AnimatedSwitcher` — replace one child with another animated:

```dart
AnimatedSwitcher(
  duration: Duration(milliseconds: 300),
  child: showA ? Text('A', key: ValueKey('A')) : Text('B', key: ValueKey('B')),
);
```

### `TweenAnimationBuilder`

For one-shot animations with custom values:
```dart
TweenAnimationBuilder<double>(
  tween: Tween(begin: 0.0, end: 1.0),
  duration: Duration(seconds: 1),
  builder: (_, value, __) => Opacity(opacity: value, child: Text('Fade in')),
);
```

---

## 8.2 Explicit Animations

For full control — chained, paused, reversed animations.

### `AnimationController`

Drives a value from 0.0 → 1.0 over a duration.

```dart
class _MyState extends State<MyWidget> with SingleTickerProviderStateMixin {
  late AnimationController _ctrl;
  late Animation<double> _scale;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this, duration: Duration(seconds: 1));
    _scale = CurvedAnimation(parent: _ctrl, curve: Curves.elasticOut);
    _ctrl.forward();
  }

  @override
  void dispose() {
    _ctrl.dispose();           // CRITICAL — avoid leaks
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ScaleTransition(scale: _scale, child: FlutterLogo(size: 100));
  }
}
```

### `Tween` & `CurvedAnimation`

```dart
final colorAnim = ColorTween(begin: Colors.red, end: Colors.blue).animate(_ctrl);
final offsetAnim = Tween<Offset>(begin: Offset(0,-1), end: Offset.zero).animate(_ctrl);

// Curves
CurvedAnimation(parent: _ctrl, curve: Curves.easeInOut);
CurvedAnimation(parent: _ctrl, curve: Interval(0.0, 0.5, curve: Curves.bounceOut));
```

Common curves: `linear`, `easeIn`, `easeOut`, `easeInOut`, `bounceOut`, `elasticOut`, `decelerate`.

### `AnimatedBuilder` & `AnimatedWidget`

Rebuild only the animating part of your tree:
```dart
AnimatedBuilder(
  animation: _ctrl,
  builder: (_, child) => Transform.rotate(angle: _ctrl.value * 6.28, child: child),
  child: FlutterLogo(size: 100),   // doesn't rebuild — perf win
);
```

### Staggered animations

Run multiple animations with offset timing on one controller:
```dart
late Animation<double> fade = CurvedAnimation(parent: _ctrl, curve: Interval(0.0, 0.5));
late Animation<double> scale = CurvedAnimation(parent: _ctrl, curve: Interval(0.4, 1.0));
```

### TickerProviderStateMixin vs SingleTickerProviderStateMixin

- **SingleTickerProviderStateMixin** — one `AnimationController` per state.
- **TickerProviderStateMixin** — multiple controllers.

The "ticker" is what fires on each frame to drive animation values.

---

## 8.3 Advanced Animations

### Hero Animations

Same widget id "flies" between pages:

```dart
// Page A
Hero(tag: 'pic-1', child: Image.network(url));
// Page B (after Navigator.push)
Hero(tag: 'pic-1', child: Image.network(url));
```

Flutter automatically animates position/size between the matched tags.

### Custom `PageRouteBuilder`

```dart
Navigator.push(context, PageRouteBuilder(
  pageBuilder: (_, __, ___) => DetailPage(),
  transitionsBuilder: (_, anim, __, child) {
    return FadeTransition(opacity: anim, child: child);
  },
));
```

### `CustomPainter` with animation

```dart
class WavePainter extends CustomPainter {
  final double t;
  WavePainter(this.t);
  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()..color = Colors.blue;
    final path = Path();
    for (double x = 0; x <= size.width; x++) {
      final y = size.height/2 + 20*sin((x/30) + t*2*pi);
      x == 0 ? path.moveTo(x, y) : path.lineTo(x, y);
    }
    canvas.drawPath(path, paint);
  }
  @override
  bool shouldRepaint(WavePainter old) => old.t != t;
}

// In a stateful widget with AnimationController
AnimatedBuilder(
  animation: _ctrl,
  builder: (_, __) => CustomPaint(painter: WavePainter(_ctrl.value)),
);
```

### Lottie

Play After Effects animations (JSON files):
```yaml
dependencies:
  lottie: ^3.0.0
```

```dart
Lottie.asset('assets/loader.json', width: 200);
Lottie.network('https://...');
```

LottieFiles.com has thousands of free animations.

### Rive

Interactive animations with state machines:
```yaml
dependencies:
  rive: ^0.13.0
```

```dart
RiveAnimation.asset('assets/button.riv');
```

More performant than Lottie for complex interactive UI animations.

### `flutter_animate`

Chain & compose animations declaratively:
```yaml
dependencies:
  flutter_animate: ^4.5.0
```

```dart
Text('Hello').animate()
  .fadeIn(duration: 500.ms)
  .slide(begin: Offset(0, -0.2))
  .then(delay: 200.ms)
  .shake();
```

Hugely productive — covers 90% of animation needs without writing controllers.

### Physics-based animations

```dart
import 'package:flutter/physics.dart';

final spring = SpringSimulation(SpringDescription.withDampingRatio(mass:1, stiffness: 100, ratio: 0.5), 0, 1, 0);
_ctrl.animateWith(spring);
```

Used for "feel" — bouncy buttons, swipe-to-dismiss snap-back.

### Shared-element transitions

Hero is the simple version. For multiple coordinated transitions, use the `animations` package (`OpenContainer`):
```dart
OpenContainer(
  closedBuilder: (_, openContainer) => Card(child: Text('Open')),
  openBuilder: (_, __) => DetailPage(),
);
```

---

# Phase 9 — Custom Painting & Graphics

## 9.1 CustomPaint

The escape hatch for drawing anything. `Canvas` is the surface; `Paint` is the brush.

```dart
class MyPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue
      ..style = PaintingStyle.fill
      ..strokeWidth = 2;

    canvas.drawRect(Rect.fromLTWH(10, 10, 80, 80), paint);
    canvas.drawCircle(Offset(150, 50), 40, paint);
    canvas.drawLine(Offset(0, 100), Offset(size.width, 100), paint);

    // Path — custom shape
    final path = Path()
      ..moveTo(0, 200)
      ..quadraticBezierTo(size.width/2, 100, size.width, 200);
    canvas.drawPath(path, paint);
  }

  @override
  bool shouldRepaint(MyPainter old) => false;
}

// Use
CustomPaint(
  size: Size(300, 300),
  painter: MyPainter(),
);
```

### Drawing images

```dart
canvas.drawImage(uiImage, Offset(0, 0), Paint());
canvas.drawImageRect(uiImage, srcRect, dstRect, Paint());
```

Load `ui.Image`: use `ImageStream` or `instantiateImageCodec`.

### Clipping

```dart
canvas.save();
canvas.clipRRect(RRect.fromRectAndRadius(rect, Radius.circular(8)));
canvas.drawImage(img, Offset.zero, paint);
canvas.restore();
```

`ClipRect`, `ClipRRect`, `ClipPath` widgets clip child widgets without writing a custom painter.

### `shouldRepaint`

Return `true` if the painter should re-run. For animations driven externally (e.g., AnimationController), the parent rebuild handles it; return `false` here unless your painter has internal mutable inputs.

---

## 9.2 Advanced Graphics

### Path & PathMetrics

`PathMetrics` lets you traverse a path to position widgets along it (think text following a curve):

```dart
final path = Path()..addOval(Rect.fromLTWH(0, 0, 200, 200));
final metrics = path.computeMetrics().first;
final pos = metrics.getTangentForOffset(metrics.length * 0.5)!.position;
```

### Shaders & Gradients on Canvas

```dart
final paint = Paint()..shader = LinearGradient(
  colors: [Colors.blue, Colors.purple],
).createShader(Rect.fromLTWH(0, 0, 200, 200));

canvas.drawRect(Rect.fromLTWH(0, 0, 200, 200), paint);
```

### Charts from scratch

You can build any chart via CustomPainter. For complex needs, use packages:

### `fl_chart`

```yaml
dependencies:
  fl_chart: ^0.66.0
```

```dart
LineChart(LineChartData(
  lineBarsData: [
    LineChartBarData(spots: [FlSpot(0,1), FlSpot(1,3), FlSpot(2,2)]),
  ],
));
```

Covers line/bar/pie/scatter/radar charts.

### `syncfusion_flutter_charts`

Most feature-rich (paid for commercial use beyond free community license).

### Flame — 2D game engine

```yaml
dependencies:
  flame: ^1.18.0
```

A game engine with sprites, collisions, physics, audio. Used for 2D games but also rich interactive UIs.

```dart
class MyGame extends FlameGame {
  @override
  Future<void> onLoad() async {
    add(SpriteComponent(sprite: await loadSprite('player.png')));
  }
}

GameWidget(game: MyGame());
```

---

# Phase 10 — Platform & Device Features

## 10.1 Device Hardware

### Camera

For photo/video capture inside your app:
```yaml
dependencies:
  camera: ^0.10.0
```

```dart
final cameras = await availableCameras();
final controller = CameraController(cameras.first, ResolutionPreset.high);
await controller.initialize();
// Show preview
CameraPreview(controller);
// Take picture
final file = await controller.takePicture();
```

For simpler use cases (pick a photo from camera or gallery):
```yaml
dependencies:
  image_picker: ^1.0.0
```

```dart
final picker = ImagePicker();
final file = await picker.pickImage(source: ImageSource.camera);
// or ImageSource.gallery, pickVideo, pickMultiImage
```

### Microphone & Audio

```yaml
dependencies:
  record: ^5.0.0          # recording
  audioplayers: ^6.0.0    # playback (simple)
  just_audio: ^0.9.0      # playback (advanced)
```

```dart
// Record
final recorder = AudioRecorder();
if (await recorder.hasPermission()) {
  await recorder.start(const RecordConfig(), path: 'audio.m4a');
}
final path = await recorder.stop();

// Play
final player = AudioPlayer();
await player.play(UrlSource('https://...mp3'));
```

`just_audio` handles streaming, playlists, gapless playback, background audio.

### GPS & Location

```yaml
dependencies:
  geolocator: ^11.0.0
```

```dart
final perm = await Geolocator.requestPermission();
if (perm == LocationPermission.always || perm == LocationPermission.whileInUse) {
  final pos = await Geolocator.getCurrentPosition();
  print('${pos.latitude}, ${pos.longitude}');
}

// Continuous updates
Geolocator.getPositionStream().listen((pos) => print(pos));
```

Always declare location usage in `AndroidManifest.xml` and `Info.plist` with a description string explaining why.

### Sensors

```yaml
dependencies:
  sensors_plus: ^5.0.0
```

```dart
accelerometerEventStream().listen((event) {
  print('x:${event.x} y:${event.y} z:${event.z}');
});
gyroscopeEventStream().listen(...);
```

### Biometrics

```yaml
dependencies:
  local_auth: ^2.2.0
```

```dart
final auth = LocalAuthentication();
final canCheck = await auth.canCheckBiometrics;
final ok = await auth.authenticate(
  localizedReason: 'Please authenticate to unlock',
  options: AuthenticationOptions(biometricOnly: true),
);
```

### Misc — Flashlight, Vibration, Battery

```yaml
dependencies:
  torch_light: ^1.0.0
  vibration: ^2.0.0
  battery_plus: ^6.0.0
```

```dart
await TorchLight.enableTorch();
Vibration.vibrate(duration: 500);
final level = await Battery().batteryLevel;
```

---

## 10.2 OS & Platform Features

### Push Notifications

Local notifications (scheduled from device):
```yaml
dependencies:
  flutter_local_notifications: ^17.0.0
```

```dart
final plugin = FlutterLocalNotificationsPlugin();
await plugin.initialize(InitializationSettings(android: AndroidInitializationSettings('@mipmap/ic_launcher')));
await plugin.show(0, 'Title', 'Body',
  NotificationDetails(android: AndroidNotificationDetails('channel_id', 'channel name', importance: Importance.high)));
```

Remote notifications — use FCM (Phase 6.4).

### Background Tasks

```yaml
dependencies:
  workmanager: ^0.5.0                    # for periodic tasks
  flutter_background_service: ^5.0.0     # persistent background service
```

```dart
Workmanager().initialize(callbackDispatcher);
Workmanager().registerPeriodicTask("syncTask", "sync", frequency: Duration(hours: 1));

@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    await syncData();
    return true;
  });
}
```

Background work on mobile is heavily throttled by the OS — design for unpredictable scheduling.

### Deep Linking

```yaml
dependencies:
  app_links: ^4.0.0
```

```dart
final appLinks = AppLinks();
appLinks.uriLinkStream.listen((uri) {
  print('Got link: $uri');
  context.go(uri.path);
});
```

### Share

```yaml
dependencies:
  share_plus: ^9.0.0
```

```dart
await Share.share('Check this out: https://example.com');
await Share.shareXFiles([XFile('/path/to/img.jpg')]);
```

### URL Launcher

```yaml
dependencies:
  url_launcher: ^6.2.0
```

```dart
await launchUrl(Uri.parse('https://example.com'));
await launchUrl(Uri.parse('tel:+1234567890'));
await launchUrl(Uri.parse('mailto:alex@x.com?subject=Hi'));
await launchUrl(Uri.parse('sms:+1234567890'));
```

### Clipboard

```dart
import 'package:flutter/services.dart';

await Clipboard.setData(ClipboardData(text: 'copied!'));
final data = await Clipboard.getData('text/plain');
```

### App Badges

```yaml
dependencies:
  flutter_app_badger: ^1.5.0
```

```dart
FlutterAppBadger.updateBadgeCount(5);
FlutterAppBadger.removeBadge();
```

### In-App Review

```yaml
dependencies:
  in_app_review: ^2.0.0
```

```dart
final inAppReview = InAppReview.instance;
if (await inAppReview.isAvailable()) {
  inAppReview.requestReview();
}
```

### App Tracking Transparency (iOS)

Required for ad tracking on iOS 14.5+:
```yaml
dependencies:
  app_tracking_transparency: ^2.0.0
```

```dart
final status = await AppTrackingTransparency.requestTrackingAuthorization();
```

---

## 10.3 File & Media

### File picker

```yaml
dependencies:
  file_picker: ^8.0.0
```

```dart
final result = await FilePicker.platform.pickFiles(
  type: FileType.custom,
  allowedExtensions: ['pdf', 'doc'],
  allowMultiple: true,
);
if (result != null) {
  for (final file in result.files) print(file.path);
}
```

### QR Code Scanner

```yaml
dependencies:
  mobile_scanner: ^5.0.0
```

```dart
MobileScanner(
  onDetect: (capture) {
    for (final code in capture.barcodes) {
      print(code.rawValue);
    }
  },
);
```

### QR Code Generator

```yaml
dependencies:
  qr_flutter: ^4.1.0
```

```dart
QrImageView(data: 'https://example.com', size: 200);
```

### PDF Viewer

```yaml
dependencies:
  syncfusion_flutter_pdfviewer: ^25.0.0
```

```dart
SfPdfViewer.network('https://example.com/doc.pdf');
SfPdfViewer.asset('assets/sample.pdf');
SfPdfViewer.file(File(path));
```

### PDF Generation

```yaml
dependencies:
  pdf: ^3.10.0
  printing: ^5.12.0
```

```dart
final pdf = pw.Document();
pdf.addPage(pw.Page(build: (ctx) => pw.Center(child: pw.Text('Hello PDF'))));
await Printing.layoutPdf(onLayout: (_) async => pdf.save());
```

### Video Player

```yaml
dependencies:
  video_player: ^2.8.0
  chewie: ^1.7.0          # nicer controls
```

```dart
final controller = VideoPlayerController.networkUrl(Uri.parse('https://...mp4'));
await controller.initialize();
controller.play();

// With Chewie UI controls
ChewieController(videoPlayerController: controller, autoPlay: true);
```

### Image Editing & Cropping

```yaml
dependencies:
  image_cropper: ^7.0.0
  image: ^4.1.0     # for pixel-level manipulation
```

```dart
final cropped = await ImageCropper().cropImage(
  sourcePath: file.path,
  aspectRatio: CropAspectRatio(ratioX: 1, ratioY: 1),
);
```

---

## 10.4 Maps

### Google Maps

```yaml
dependencies:
  google_maps_flutter: ^2.6.0
```

Add API key in `AndroidManifest.xml` and `AppDelegate.swift`.

```dart
GoogleMap(
  initialCameraPosition: CameraPosition(target: LatLng(28.6, 77.2), zoom: 12),
  markers: {
    Marker(markerId: MarkerId('home'), position: LatLng(28.6, 77.2)),
  },
  polylines: {
    Polyline(polylineId: PolylineId('p'), points: [LatLng(...), LatLng(...)]),
  },
  onMapCreated: (controller) {},
);
```

### Mapbox

Self-hosted styles, often cheaper at scale:
```yaml
dependencies:
  mapbox_maps_flutter: ^1.0.0
```

### OpenStreetMap

Free, no API key:
```yaml
dependencies:
  flutter_map: ^6.1.0
```

```dart
FlutterMap(
  options: MapOptions(initialCenter: LatLng(28.6, 77.2), initialZoom: 12),
  children: [
    TileLayer(urlTemplate: 'https://tile.openstreetmap.org/{z}/{x}/{y}.png'),
    MarkerLayer(markers: [Marker(point: LatLng(...), child: Icon(Icons.location_on))]),
  ],
);
```

---

## 10.5 Payments & Monetization

### In-app purchases

```yaml
dependencies:
  in_app_purchase: ^3.1.0
```

```dart
final available = await InAppPurchase.instance.isAvailable();
final response = await InAppPurchase.instance.queryProductDetails({'premium_monthly'});
await InAppPurchase.instance.buyNonConsumable(purchaseParam: PurchaseParam(productDetails: product));

InAppPurchase.instance.purchaseStream.listen((purchases) {
  for (final p in purchases) {
    if (p.status == PurchaseStatus.purchased) {
      // Verify receipt on your server, then deliver
      InAppPurchase.instance.completePurchase(p);
    }
  }
});
```

Always verify receipts server-side — never trust the client.

### Razorpay / Stripe

```yaml
dependencies:
  razorpay_flutter: ^1.3.0
  flutter_stripe: ^10.0.0
```

```dart
// Stripe
await Stripe.instance.initPaymentSheet(paymentSheetParameters: SetupPaymentSheetParameters(
  paymentIntentClientSecret: clientSecret,
));
await Stripe.instance.presentPaymentSheet();
```

For card data, never collect it inside your app — always use the provider's hosted UI (Stripe Element / Payment Sheet) to stay PCI-compliant.

### AdMob

```yaml
dependencies:
  google_mobile_ads: ^5.0.0
```

```dart
MobileAds.instance.initialize();

BannerAd(
  adUnitId: 'ca-app-pub-...',
  request: AdRequest(),
  size: AdSize.banner,
  listener: BannerAdListener(),
).load();
```

Add a `BannerAd` widget to your tree; load it on init.

---

# Phase 11 — Architecture & Code Quality

## 11.1 Project Structure

### Feature-first (recommended for medium+ apps)

```
lib/
├── core/                    # shared infra
│   ├── network/
│   ├── storage/
│   └── theme/
├── features/
│   ├── auth/
│   │   ├── data/            # repositories, datasources
│   │   ├── domain/          # entities, use cases
│   │   └── presentation/    # widgets, providers
│   ├── home/
│   └── profile/
├── shared/                  # reusable widgets
└── main.dart
```

Each feature is self-contained — easy to assign teams to features, easy to extract into a package.

### Layer-first (small apps)

```
lib/
├── models/
├── services/
├── screens/
├── widgets/
└── main.dart
```

Simpler but doesn't scale — as you add features, files spread across folders.

### Barrel files

`index.dart` re-exports public API:
```dart
// features/auth/auth.dart
export 'presentation/login_page.dart';
export 'presentation/signup_page.dart';
export 'domain/auth_service.dart';

// Use elsewhere
import 'package:my_app/features/auth/auth.dart';
```

Reduces import noise. Use selectively — barrel files break tree-shaking if overused.

### Separation of concerns

- **UI widgets** — purely declarative, no business logic, no HTTP calls.
- **State / ViewModel / Notifier** — coordinates UI events, calls services.
- **Service / Repository** — talks to API/DB; abstracts data sources.
- **Models** — plain data classes (Freezed).

When you can swap the UI without touching services, your architecture is healthy.

---

## 11.2 Architecture Patterns

### MVC

- **Model** — data structures.
- **View** — widgets.
- **Controller** — handles user input, updates model.

Light separation; works for small apps. Boundaries blur in Flutter because widgets are both view and (with setState) controller.

### MVVM

- **View** — widget that observes a ViewModel.
- **ViewModel** — exposes state + commands; no Flutter imports.
- **Model** — domain entities.

```dart
class LoginViewModel extends ChangeNotifier {
  String _email = '';
  String _password = '';
  bool _loading = false;
  String? _error;

  bool get loading => _loading;
  String? get error => _error;

  set email(String v) { _email = v; notifyListeners(); }
  set password(String v) { _password = v; notifyListeners(); }

  Future<bool> submit() async {
    _loading = true; _error = null; notifyListeners();
    try {
      await authService.login(_email, _password);
      return true;
    } catch (e) {
      _error = e.toString();
      return false;
    } finally {
      _loading = false; notifyListeners();
    }
  }
}
```

The View is a thin Consumer/ConsumerWidget reading the ViewModel.

### Clean Architecture

Three layers + dependency rule: inner layers don't know about outer.

```
┌───────────────────────────────────────────┐
│            Presentation                     │   ← widgets, ViewModels
├───────────────────────────────────────────┤
│             Domain                          │   ← entities, use cases (no Flutter, no HTTP)
├───────────────────────────────────────────┤
│              Data                           │   ← repositories, datasources (HTTP, DB)
└───────────────────────────────────────────┘
```

- **Domain** defines **interfaces** (abstract repositories).
- **Data** implements those interfaces.
- **Presentation** depends on Domain only.

```dart
// Domain
abstract class UserRepository {
  Future<User> getUser(String id);
}
class GetUser {
  final UserRepository repo;
  GetUser(this.repo);
  Future<User> call(String id) => repo.getUser(id);
}

// Data
class UserRepositoryImpl implements UserRepository {
  final ApiClient api;
  UserRepositoryImpl(this.api);
  @override
  Future<User> getUser(String id) async {
    final dto = await api.fetchUser(id);
    return User(id: dto.id, name: dto.name);   // map DTO → entity
  }
}

// Presentation
class UserViewModel {
  final GetUser getUser;
  UserViewModel(this.getUser);
  Future<void> load(String id) async { _user = await getUser(id); }
}
```

Overkill for simple apps; lifesaver for large ones with multiple data sources or planned API changes.

### Repository Pattern

The single abstraction your app uses to talk to data, regardless of backend (REST, GraphQL, local cache):

```dart
abstract class TodoRepository {
  Future<List<Todo>> getAll();
  Future<Todo> getById(String id);
  Future<void> save(Todo t);
}

class TodoRepositoryImpl implements TodoRepository {
  final ApiClient api;
  final LocalDb db;
  TodoRepositoryImpl(this.api, this.db);

  @override
  Future<List<Todo>> getAll() async {
    try {
      final remote = await api.getTodos();
      await db.cacheTodos(remote);
      return remote;
    } catch (_) {
      return db.getCachedTodos();   // offline fallback
    }
  }
}
```

### UseCase Pattern

One class per business operation — explicit, composable, testable:

```dart
class LoginUser {
  final AuthRepository repo;
  LoginUser(this.repo);

  Future<Result<User>> call(String email, String password) async {
    if (!email.contains('@')) return Result.failure('Invalid email');
    return Result.success(await repo.login(email, password));
  }
}
```

ViewModels call use cases; tests mock the repo.

### Dependency Injection

Don't construct dependencies inside widgets — inject them.

**Service locator — `get_it`:**
```yaml
dependencies:
  get_it: ^7.7.0
```

```dart
final sl = GetIt.instance;

void setupDi() {
  sl.registerSingleton<ApiClient>(ApiClient());
  sl.registerSingleton<UserRepository>(UserRepositoryImpl(sl()));
  sl.registerFactory(() => UserViewModel(sl()));
}

// Use
final vm = sl<UserViewModel>();
```

**`injectable` — annotation-based codegen on top of get_it:**
```dart
@injectable
class UserRepository {
  final ApiClient api;
  UserRepository(this.api);
}
```

Run `build_runner` to generate registrations.

With Riverpod, providers themselves are DI — you usually don't need a separate service locator.

---

## 11.3 Code Quality

### Linting

```yaml
dev_dependencies:
  flutter_lints: ^4.0.0
  # or for stricter rules
  very_good_analysis: ^6.0.0
```

```yaml
# analysis_options.yaml
include: package:very_good_analysis/analysis_options.yaml
linter:
  rules:
    avoid_print: true
    prefer_const_constructors: true
```

Lints catch bugs and enforce style. Run `dart analyze` in CI.

### Formatting

```bash
dart format .
```

Set up your IDE to format on save. Don't argue about style in code review — let the formatter decide.

### DRY, SOLID

- **Single Responsibility** — one class, one reason to change.
- **Open/Closed** — extend behaviour without modifying existing code (use composition).
- **Liskov Substitution** — subclasses must be usable where parent is expected.
- **Interface Segregation** — many small interfaces beat one fat one.
- **Dependency Inversion** — depend on abstractions, not concretes (the basis of Clean Architecture).

Flutter is heavily compositional — favour composition over inheritance for widgets.

### Effective Dart

[dart.dev/effective-dart](https://dart.dev/effective-dart) — the official style guide. Key rules:

- Use lowerCamelCase for variables, UpperCamelCase for types.
- Prefer `final` over `var`.
- Use `???`, `?.` over manual null checks.
- Use named constructors for clarity.
- Prefer expression bodies (`=>`) for one-line functions.

### Null Safety Best Practices

- **Avoid `!` (bang)** — it's a runtime crash waiting to happen. Prefer `??` defaults or narrowing.
- **Avoid `late` unless necessary** — defers crashes to runtime.
- **Type your function returns** — `Future<void>` not `Future`.
- **Use sealed/freezed for state** — exhaustive switch + non-nullable fields make impossible states impossible.

---

# Phase 12 — Testing

Three layers, increasing scope and cost:

```
       /\         Integration (slow, real device)
      /  \
     /----\       Widget tests (mid, simulated)
    /------\
   /        \     Unit tests (fast, pure Dart)
  /----------\
```

## 12.1 Unit Testing

Tests pure Dart logic (services, models, view models) without Flutter.

```yaml
dev_dependencies:
  test: ^1.25.0
  mocktail: ^1.0.0       # mocking
```

```dart
import 'package:test/test.dart';

void main() {
  group('TaxCalculator', () {
    test('returns 10% of input', () {
      expect(calcTax(100), 10);
    });

    test('returns 0 for negative input', () {
      expect(calcTax(-1), 0);
    });

    test('throws on null input', () {
      expect(() => calcTax(null), throwsArgumentError);
    });
  });
}
```

Run with `flutter test test/calc_test.dart`.

### Mocking

Use `mocktail` (no code gen) or `mockito` (with code gen).

```dart
import 'package:mocktail/mocktail.dart';

class MockApi extends Mock implements ApiClient {}

void main() {
  late MockApi api;
  late UserRepository repo;

  setUp(() {
    api = MockApi();
    repo = UserRepository(api);
  });

  test('repository returns user from API', () async {
    when(() => api.fetchUser('1')).thenAnswer((_) async => User(id: '1', name: 'A'));
    final user = await repo.getUser('1');
    expect(user.name, 'A');
    verify(() => api.fetchUser('1')).called(1);
  });

  test('repository handles API error', () async {
    when(() => api.fetchUser(any())).thenThrow(Exception('boom'));
    expect(() => repo.getUser('1'), throwsA(isA<Exception>()));
  });
}
```

### Testing ChangeNotifier / Cubit / Notifier

```dart
test('counter increments', () {
  final cubit = CounterCubit();
  expect(cubit.state, 0);
  cubit.increment();
  expect(cubit.state, 1);
});

// Riverpod
test('notifier increments', () {
  final container = ProviderContainer();
  expect(container.read(counterProvider), 0);
  container.read(counterProvider.notifier).increment();
  expect(container.read(counterProvider), 1);
  container.dispose();
});
```

---

## 12.2 Widget Testing

Tests widgets in a simulated environment — fast, no real device.

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('counter increments on tap', (tester) async {
    await tester.pumpWidget(MaterialApp(home: CounterPage()));

    expect(find.text('0'), findsOneWidget);

    await tester.tap(find.byIcon(Icons.add));
    await tester.pump();   // rebuild after setState

    expect(find.text('1'), findsOneWidget);
  });
}
```

### `WidgetTester` & `pumpWidget`

- `pumpWidget(widget)` — render the widget.
- `pump()` — rebuild after a state change (sync).
- `pumpAndSettle()` — pump until no more animations are running (use cautiously — can hang if you have indefinite animations).

### Finders

```dart
find.byType(ElevatedButton);
find.byKey(ValueKey('submit'));
find.text('Login');
find.byIcon(Icons.search);
find.byTooltip('Refresh');
find.descendant(of: find.byType(Card), matching: find.text('Hello'));
```

### Interactions

```dart
await tester.tap(find.byKey(Key('submit')));
await tester.enterText(find.byType(TextField), 'hello');
await tester.drag(find.byType(ListView), Offset(0, -200));
await tester.longPress(find.byKey(Key('item-1')));
```

### Golden Tests

Compare a widget's rendered pixels against a saved image:

```dart
testWidgets('matches golden', (tester) async {
  await tester.pumpWidget(MyButton());
  await expectLater(find.byType(MyButton), matchesGoldenFile('my_button.png'));
});
```

Update goldens: `flutter test --update-goldens`. Useful for visual regression but flaky across platforms (font rendering differs).

---

## 12.3 Integration Testing

Tests the whole app on a real device / emulator.

```yaml
dev_dependencies:
  integration_test:
    sdk: flutter
```

```dart
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('login flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(Key('email')), 'a@x.com');
    await tester.enterText(find.byKey(Key('password')), 'secret123');
    await tester.tap(find.byKey(Key('login-btn')));
    await tester.pumpAndSettle();

    expect(find.text('Welcome'), findsOneWidget);
  });
}
```

Run with `flutter test integration_test/app_test.dart`.

These run against the actual backend (or a mock server) — slower but catch real-world bugs widget tests miss.

---

# Phase 13 — Performance & Optimization

## 13.1 Rendering Performance

### Flutter DevTools

The official perf toolkit. Open from your IDE or `dart devtools`:
- **Widget Inspector** — explore the tree, see widget sizes, constraints, baselines.
- **Performance / Timeline** — frame-by-frame timing; spot dropped frames.
- **CPU Profiler** — what code ran during a frame.
- **Memory** — heap snapshots, allocations over time.
- **Network** — HTTP requests + sizes.

Run app in **Profile mode** (`flutter run --profile`) for realistic perf — debug mode is slow due to assertions.

### Identifying Jank

A "frame budget" is ~16ms (60fps) or ~8ms (120fps). If a frame exceeds this, the UI stutters ("jank").

In DevTools Timeline:
- Look for bars exceeding 16ms.
- Identify the long-running function (`build()`, paint, layout).
- Common causes: large lists without `.builder`, complex widgets in scroll, sync work in build, image decode on main thread.

### `const` constructors everywhere

`const` widgets are canonicalized — Flutter reuses the exact same instance instead of rebuilding:

```dart
// ❌ new Text widget every build
Text('Hello')

// ✅ same instance reused
const Text('Hello')
```

Lint `prefer_const_constructors` makes this automatic.

### Avoiding unnecessary rebuilds

The biggest perf killer is rebuilding too much. Strategies:

1. **Lift state down, not up** — keep state in the smallest possible widget.
2. **Use `Consumer` / `Selector` (Provider) or `ref.watch` for narrow data** — only the consumer rebuilds.
3. **`context.select<Model, T>((m) => m.field)`** instead of `watch` to subscribe to one field.
4. **Pass `child` to AnimatedBuilder** — the child doesn't rebuild on each tick.
5. **`const` widgets** — skip identity check.

```dart
// ❌ entire Scaffold rebuilds when counter changes
Widget build(BuildContext context) {
  final count = context.watch<Counter>().value;
  return Scaffold(appBar: AppBar(title: Text('App')), body: Text('$count'));
}

// ✅ only the inner Text rebuilds
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(title: Text('App')),
    body: Consumer<Counter>(builder: (_, c, __) => Text('${c.value}')),
  );
}
```

### `RepaintBoundary`

Isolates a subtree's painting — repaints don't propagate above or below.

```dart
RepaintBoundary(
  child: ExpensiveCustomPaint(),
);
```

Use around heavy painted widgets that change independently from surroundings (animated icons, charts).

### `ListView.builder` vs `ListView`

`ListView(children: [...])` builds all children eagerly — fine for ≤20 items, disastrous for 1000.

`ListView.builder` lazily builds only visible items + a buffer. Always use builder for long lists.

### Slivers

For complex scrolling UIs (collapsing app bars, mixed lists/grids), use `CustomScrollView` + slivers. They share a single scroll controller and lazy build everything.

---

## 13.2 App Performance

### Image optimization

- **Use appropriate format** — JPEG for photos, PNG for sharp graphics, WebP for both (smaller).
- **Resize before display** — don't ship a 4000×4000 image and let Flutter scale it down per frame.
- **Use `ResizeImage`** to decode at target size:
  ```dart
  Image(image: ResizeImage(NetworkImage(url), width: 200));
  ```
- **`cached_network_image`** for disk caching.
- **Use `flutter_svg`** for vector assets — small, scale infinitely.

### Lazy loading & pagination

For long lists, load in chunks:

```dart
class PaginatedList extends StatefulWidget { ... }
class _State extends State<PaginatedList> {
  final _items = <Item>[];
  bool _loading = false;
  int _page = 1;

  Future<void> _loadMore() async {
    if (_loading) return;
    setState(() => _loading = true);
    final next = await api.getItems(page: _page);
    setState(() {
      _items.addAll(next);
      _page++;
      _loading = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _items.length + 1,
      itemBuilder: (_, i) {
        if (i == _items.length) {
          _loadMore();
          return Center(child: CircularProgressIndicator());
        }
        return ItemTile(_items[i]);
      },
    );
  }
}
```

Or use `infinite_scroll_pagination` for production-grade pagination.

### Isolates for heavy computation

```dart
import 'package:flutter/foundation.dart';

final result = await compute(parseHugeJson, jsonString);

// parseHugeJson must be a top-level or static function
List<Item> parseHugeJson(String s) {
  return (jsonDecode(s) as List).map((e) => Item.fromJson(e)).toList();
}
```

`compute` spawns an isolate, runs the function, returns the result. Use for >50ms work to avoid blocking the UI.

### Reducing startup time

- **Defer heavy work** — don't block `main()` on slow async ops; show a splash and load in parallel.
- **Lazy-init services** — instead of constructing everything at startup, lazy-load when first needed.
- **`flutter_native_splash`** for fast native splash before Flutter even initializes.

### Memory leaks

Common in Flutter:
- **Forgotten StreamSubscription / Listener** — always `cancel()` in `dispose()`.
- **`AnimationController` not disposed** — leaks the ticker.
- **`TextEditingController` not disposed**.
- **Static caches that grow unbounded** — use LRU.

Detect via DevTools Memory tab → heap snapshot diff.

### Profile / Release / Debug modes

| Mode | Use | Asserts | Tree shaking | JIT/AOT |
|---|---|---|---|---|
| Debug | dev (hot reload) | ✅ | ❌ | JIT |
| Profile | perf testing | ❌ | ✅ | AOT |
| Release | production | ❌ | ✅ | AOT |

Never benchmark in debug — it's 10x+ slower than release.

---

## 13.3 App Size Optimization

### Tree shaking

Flutter automatically strips unused code in release builds. Helped by:
- Avoid `dynamic` types (defeats type-based dead-code analysis).
- Avoid mirror packages.

### Deferred components (Android)

Split off optional code into a separate APK module, downloaded on demand:

```dart
import 'rare_feature.dart' deferred as rare;

await rare.loadLibrary();
rare.doRareThing();
```

Reduces initial download size.

### Asset optimization

- Compress images (TinyPNG, Squoosh).
- Use WebP over PNG/JPG.
- Delete unused assets — they bloat the bundle.
- Use SVG via `flutter_svg` for icons.

### Obfuscation

```bash
flutter build apk --obfuscate --split-debug-info=build/symbols
flutter build appbundle --obfuscate --split-debug-info=build/symbols
```

Renames Dart symbols, making reverse-engineering harder. Save the `symbols/` directory — required to symbolicate stack traces from production.

### Inspect APK size

```bash
flutter build apk --analyze-size
```

Generates a breakdown of contributors to your app size.

### Split APKs per ABI

```bash
flutter build apk --split-per-abi
```

Generates separate APKs for arm, arm64, x86_64. Each ~10MB smaller than the universal APK. The Play Store does this automatically with App Bundles — prefer **AAB** over APK.

---

# Phase 14 — Deployment & CI/CD

## 14.1 Build & Release

### Android

**1. Generate a keystore (one-time):**
```bash
keytool -genkey -v -keystore upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

**2. Create `android/key.properties` (don't commit):**
```
storePassword=...
keyPassword=...
keyAlias=upload
storeFile=/absolute/path/to/upload-keystore.jks
```

**3. Configure `android/app/build.gradle`:**
```gradle
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    buildTypes {
        release { signingConfig signingConfigs.release }
    }
}
```

**4. Build:**
```bash
flutter build apk --release             # APK for sideloading
flutter build appbundle --release       # AAB for Play Store (preferred)
```

**APK vs AAB:** Play Store requires AAB. It generates per-device APKs automatically (smaller downloads).

### iOS

**1. In Xcode:** open `ios/Runner.xcworkspace`, set bundle identifier, team, certificates.

**2. Provisioning profile** — manage in Apple Developer portal. For automatic signing, "Automatically manage signing" in Xcode.

**3. Build:**
```bash
flutter build ipa --release
```

**4. Upload via Transporter app** (Mac App Store) or `xcrun altool` / `fastlane pilot`.

You **need a Mac** to build iOS apps. Cloud Mac services (Codemagic, Bitrise, Mac Stadium) can substitute.

### Versioning

In `pubspec.yaml`:
```yaml
version: 1.2.3+45    # version+build
```

- `version` (`1.2.3`) = public version (CFBundleShortVersionString / versionName).
- `build` (`45`) = internal build number (CFBundleVersion / versionCode). Must increment for every store upload.

### Flavors / Build Variants

Different configs (dev, staging, prod) from one codebase.

```bash
flutter run --flavor dev -t lib/main_dev.dart
flutter build apk --flavor prod -t lib/main_prod.dart
```

Configure `android/app/build.gradle`:
```gradle
flavorDimensions "env"
productFlavors {
    dev { dimension "env"; applicationIdSuffix ".dev"; resValue "string", "app_name", "MyApp Dev" }
    prod { dimension "env"; resValue "string", "app_name", "MyApp" }
}
```

Each flavor can have its own icon, name, API endpoint.

### Environment variables

```yaml
dependencies:
  flutter_dotenv: ^5.1.0
```

```dart
await dotenv.load(fileName: '.env');
final apiUrl = dotenv.env['API_URL']!;
```

Or use `--dart-define`:
```bash
flutter build apk --dart-define=API_URL=https://api.example.com
```

```dart
const apiUrl = String.fromEnvironment('API_URL', defaultValue: 'http://localhost');
```

`--dart-define` is preferred for production — values are compiled in, not read from a file at runtime.

---

## 14.2 App Stores

### Google Play Store

**Tracks:**
- **Internal testing** — up to 100 testers; instant rollout.
- **Closed (alpha)** — invite list.
- **Open (beta)** — anyone with link.
- **Production** — public.

**Process:**
1. Create app in Play Console.
2. Fill in store listing (descriptions, screenshots, icon, feature graphic).
3. Upload AAB to chosen track.
4. Submit for review (typically <24h).
5. Promote between tracks as confidence grows.

### Apple App Store

**Process:**
1. Create app in App Store Connect.
2. Upload build via Xcode / Transporter / `fastlane pilot`.
3. Use **TestFlight** for beta — internal (Apple ID emails) or external (up to 10,000 testers, requires review).
4. Submit for App Store review (typically 24–48h).
5. Publish — instant once approved.

App review is stricter than Google's — prepare for rejection on edge cases (in-app purchase routing, account deletion requirement, privacy policy gaps).

### App store assets

- **Screenshots** — at least 2; multiple device sizes.
- **App icon** — 1024×1024 (master).
- **Feature graphic** (Play) — 1024×500.
- **Privacy policy URL** required.
- **Localized descriptions** for multiple markets.

### `flutter_launcher_icons`

Generate all icon sizes from one master image:
```yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.0

flutter_launcher_icons:
  android: true
  ios: true
  image_path: "assets/icon.png"
  adaptive_icon_background: "#FFFFFF"
  adaptive_icon_foreground: "assets/icon-foreground.png"
```

```bash
dart run flutter_launcher_icons
```

### `flutter_native_splash`

```yaml
dev_dependencies:
  flutter_native_splash: ^2.4.0

flutter_native_splash:
  color: "#ffffff"
  image: assets/splash.png
  android_12:
    image: assets/splash_android12.png
```

```bash
dart run flutter_native_splash:create
```

Generates native splash screens that show **before** Flutter initializes — avoids the white flash on cold start.

---

## 14.3 CI/CD

### GitHub Actions

`.github/workflows/build.yml`:
```yaml
name: Build
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          channel: stable
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test
      - run: flutter build apk --release
      - uses: actions/upload-artifact@v4
        with: { name: apk, path: build/app/outputs/flutter-apk/app-release.apk }
```

For iOS builds, use `runs-on: macos-latest`.

### Fastlane

Automates store upload + metadata.

`ios/fastlane/Fastfile`:
```ruby
lane :beta do
  build_app(workspace: "Runner.xcworkspace", scheme: "Runner")
  upload_to_testflight
end
```

`android/fastlane/Fastfile`:
```ruby
lane :internal do
  gradle(task: "bundleRelease")
  upload_to_play_store(track: 'internal', aab: '../build/app/outputs/bundle/release/app-release.aab')
end
```

Run `fastlane beta` / `fastlane internal`.

### Codemagic / Bitrise / CircleCI

Cloud CI/CD with first-class Flutter support — no need to maintain Mac runners.

Codemagic has a free tier for Flutter; `codemagic.yaml` defines pipelines.

### Firebase App Distribution

Send beta builds to testers without going through TestFlight / Play internal track:

```bash
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-release.apk \
  --app 1:...:android:... \
  --groups "qa-team"
```

Faster than store-based testing for early iterations.

---

# Phase 15 — Advanced Topics

## 15.1 Flutter Web & Desktop

### Flutter Web

`flutter build web` outputs a static site. Two rendering modes:
- **CanvasKit** (default in release) — Skia compiled to WASM; pixel-perfect but bigger download.
- **HTML** — uses DOM/CSS where possible; smaller; less accurate.

```bash
flutter build web --release --web-renderer canvaskit
flutter build web --release --web-renderer html
```

### Web-specific concerns

- **URL strategy** — by default Flutter uses `#/path` hash routing. For clean paths:
  ```dart
  import 'package:flutter_web_plugins/flutter_web_plugins.dart';
  void main() { usePathUrlStrategy(); runApp(MyApp()); }
  ```
  Requires server to fall back to `index.html` for SPA routing.
- **SEO** — Flutter renders to canvas; not great for SEO. For content-heavy sites, use SSR (jaspr) or another framework.
- **iframe widgets** — `HtmlElementView` to embed native HTML inside Flutter.

### Flutter Desktop (Windows / macOS / Linux)

```bash
flutter create --platforms=windows,macos,linux my_app
flutter run -d macos
flutter build macos
```

Most packages work but check platform support. UI scales differently — desktop has resizable windows, multiple windows, keyboard shortcuts, menus.

### Platform differences

```dart
import 'dart:io' show Platform;

if (Platform.isAndroid) ...
if (Platform.isIOS) ...
if (Platform.isMacOS) ...

// For web (dart:io not available)
import 'package:flutter/foundation.dart' show kIsWeb;
if (kIsWeb) ...
```

### `flutter_adaptive_scaffold`

Material 3 adaptive layouts for phone/tablet/desktop:
```dart
AdaptiveScaffold(
  destinations: [...],
  body: (_) => ...,
  smallBody: (_) => MobileLayout(),
  largeBody: (_) => TabletLayout(),
);
```

---

## 15.2 Platform Channels

When you need native APIs Flutter doesn't expose (e.g., a specific iOS framework, Android hardware feature), use platform channels.

### MethodChannel — call native code

```dart
const platform = MethodChannel('com.example/battery');

Future<int> getBatteryLevel() async {
  final result = await platform.invokeMethod<int>('getBatteryLevel');
  return result ?? 0;
}
```

**Android (Kotlin)** in `MainActivity.kt`:
```kotlin
class MainActivity: FlutterActivity() {
  override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
    super.configureFlutterEngine(flutterEngine)
    MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "com.example/battery")
      .setMethodCallHandler { call, result ->
        if (call.method == "getBatteryLevel") {
          val bm = getSystemService(Context.BATTERY_SERVICE) as BatteryManager
          val level = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
          result.success(level)
        } else result.notImplemented()
      }
  }
}
```

**iOS (Swift)** in `AppDelegate.swift`:
```swift
let controller = window?.rootViewController as! FlutterViewController
let channel = FlutterMethodChannel(name: "com.example/battery", binaryMessenger: controller.binaryMessenger)
channel.setMethodCallHandler { call, result in
  if call.method == "getBatteryLevel" {
    UIDevice.current.isBatteryMonitoringEnabled = true
    result(Int(UIDevice.current.batteryLevel * 100))
  } else { result(FlutterMethodNotImplemented) }
}
```

### EventChannel — stream from native

For continuous data (sensors, location updates):
```dart
const channel = EventChannel('com.example/sensors');
channel.receiveBroadcastStream().listen((data) => print(data));
```

### Writing a Flutter Plugin

For reusable native code:
```bash
flutter create --template=plugin -i swift -a kotlin my_plugin
```

Generates a plugin package structure with iOS, Android (and web/desktop) implementations.

### FFI — call C/C++ directly

For native binaries (cryptography, ML inference) without writing a plugin:
```yaml
dependencies:
  ffi: ^2.1.0
```

```dart
import 'dart:ffi';
final dylib = DynamicLibrary.open('libnative.so');
final add = dylib.lookupFunction<Int32 Function(Int32, Int32), int Function(int, int)>('add');
print(add(2, 3));
```

`ffigen` can auto-generate Dart bindings from C headers.

---

## 15.3 Flutter Internals

### Rendering pipeline

Every frame goes through phases:
1. **Build** — call `build()` on dirty widgets to produce widget tree.
2. **Layout** — walk RenderObject tree; each parent constrains its children, each child reports its size.
3. **Paint** — RenderObjects record paint commands into a layer tree.
4. **Composite** — engine combines layers and submits to GPU.

Performance optimization is mostly about minimizing work in steps 1–3.

### RenderObject tree

Below widgets and elements is the RenderObject tree — the actual layout engine. Most apps never touch it. To create a truly custom widget with custom layout, you'd write a `RenderBox` subclass.

### Element lifecycle

For each widget, there's an Element instance:
1. `mount` — first time inserted into tree.
2. `update(newWidget)` — when parent rebuilds with new widget config.
3. `deactivate` — temporarily removed (may be reactivated).
4. `unmount` — permanently removed.

`State.initState()` corresponds to mount; `dispose()` to unmount.

### InheritedWidget internals

The framework keeps a map from `InheritedWidget` type to widget. When a descendant calls `dependOnInheritedWidgetOfExactType`, the element registers as a dependent. On change, `updateShouldNotify` decides whether to rebuild dependents. This is how `Theme.of`, `MediaQuery.of`, Provider, Riverpod all work under the hood.

### Sliver protocol

Slivers exchange "slivers of work" with parent scrollables — they communicate via `SliverConstraints` and `SliverGeometry` instead of `BoxConstraints` and `Size`. This enables lazy materialization of off-screen items.

### Shader compilation

Flutter compiles its render commands to GLSL/SkSL shaders. First-frame shader compilation jank ("shader jank") was a historic complaint. Mitigations:
- **Shader warm-up** — pre-render typical UI in profile mode to capture used shaders.
- **Impeller** — new rendering engine that uses a precompiled shader pipeline.

### Impeller

Flutter's new rendering engine (default on iOS as of 3.10; opt-in on Android). Replaces Skia + runtime shader compilation with a Vulkan/Metal pipeline. Eliminates shader jank, improves perf consistency.

---

## 15.4 Internationalization (i18n)

### `flutter_localizations`

```yaml
dependencies:
  flutter_localizations:
    sdk: flutter
  intl: ^0.19.0
```

### ARB files

Define strings per language:

`lib/l10n/app_en.arb`:
```json
{
  "welcome": "Welcome",
  "greeting": "Hello, {name}!",
  "@greeting": { "placeholders": { "name": {"type": "String"} } },
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}"
}
```

`lib/l10n/app_es.arb`:
```json
{
  "welcome": "Bienvenido",
  "greeting": "Hola, {name}!"
}
```

### `l10n.yaml`

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

Run `flutter gen-l10n` to generate `AppLocalizations`. Use:

```dart
MaterialApp(
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
  home: HomePage(),
);

Text(AppLocalizations.of(context)!.welcome);
Text(AppLocalizations.of(context)!.greeting('Alex'));
```

### Pluralization & gender

`intl` supports ICU MessageFormat for plurals and gender-based variations — see the `itemCount` example above.

### RTL support

Flutter auto-flips most widgets for RTL languages (Arabic, Hebrew) when `Locale` is RTL. Use `Directionality.of(context)` to query. Test with:

```dart
MaterialApp(locale: Locale('ar'), ...)
```

Use `EdgeInsetsDirectional`, `AlignmentDirectional` instead of `EdgeInsets`/`Alignment` to respect RTL.

### Dynamic language switching

```dart
class App extends StatefulWidget {
  static _AppState of(BuildContext c) => c.findAncestorStateOfType<_AppState>()!;
  @override
  State<App> createState() => _AppState();
}

class _AppState extends State<App> {
  Locale _locale = Locale('en');
  void setLocale(Locale l) => setState(() => _locale = l);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(locale: _locale, ...);
  }
}

// To switch
App.of(context).setLocale(Locale('es'));
```

Persist the choice in `shared_preferences`.

---

## 15.5 Accessibility

Make your app usable for people with disabilities — also helps with automated testing.

### `Semantics` widget

```dart
Semantics(
  label: 'Add to cart',
  button: true,
  child: IconButton(icon: Icon(Icons.add), onPressed: ...),
);
```

Most Flutter widgets emit reasonable semantics automatically. Add `Semantics` when:
- A widget shows graphic info that screen reader needs (charts, custom icons).
- You wrap multiple widgets and want them announced as one.

### Screen reader support

- **Android TalkBack** / **iOS VoiceOver** read semantics aloud.
- Test by enabling them in Settings.
- Use `ExcludeSemantics` for purely decorative content.
- Use `MergeSemantics` to combine multiple widgets into one announced unit.

### Accessible colors & contrast

WCAG AA requires 4.5:1 contrast for normal text, 3:1 for large text. Tools: WebAIM Contrast Checker.

Flutter's `ThemeData.colorScheme` is generally well-balanced; verify your custom colors.

### Touch target sizing

Material guidelines: 48×48 dp minimum tap target.

```dart
IconButton(
  iconSize: 24,
  padding: EdgeInsets.all(12),    // total 48dp
  ...
);
```

`InkWell`/`GestureDetector` should also be at least 48dp.

### Dynamic text sizing

Respect the user's OS font scale:
```dart
final scale = MediaQuery.textScalerOf(context);
```

Use `Text` (which scales) over manually sized text. Layouts should accommodate ~200% scale — test with `MediaQuery(data: MediaQueryData(textScaler: TextScaler.linear(2))...)`.

---

## 15.6 Security

### Certificate pinning

Reject server certs that don't match a known fingerprint — defends against MITM with rogue CAs:

```dart
final dio = Dio();
(dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
  final client = HttpClient();
  client.badCertificateCallback = (cert, host, port) {
    final fingerprint = sha256.convert(cert.der).toString();
    return fingerprint == 'YOUR_PINNED_HASH';
  };
  return client;
};
```

For most apps this is over-engineered; defend against MITM via TLS + HSTS. Pin only for high-value targets (banking, healthcare).

### Obfuscation

```bash
flutter build apk --obfuscate --split-debug-info=build/symbols
```

Combine with ProGuard rules (Android) for Java-side obfuscation. Apple App Store auto-strips iOS symbols.

Symbols saved in `build/symbols/` are required to symbolicate crashes — upload to Crashlytics / your APM.

### Secure storage for tokens

Use `flutter_secure_storage` (Phase 7.1). Never store tokens in `shared_preferences`, plain files, or app state without encryption.

### Jailbreak / root detection

```yaml
dependencies:
  flutter_jailbreak_detection: ^1.10.0
```

```dart
final jailbroken = await FlutterJailbreakDetection.jailbroken;
if (jailbroken) {
  // restrict sensitive features
}
```

Note: not foolproof — determined attackers can bypass. Use as one layer of defense, not the only one.

### Preventing screenshots (Android)

```dart
import 'package:flutter_windowmanager/flutter_windowmanager.dart';

await FlutterWindowManager.addFlags(FlutterWindowManager.FLAG_SECURE);
```

Useful for screens showing sensitive info (banking, OTPs).

iOS doesn't allow apps to fully block screenshots — only obscure content during app switcher with `WidgetsBindingObserver` + a cover view.

### ProGuard / R8 rules (Android)

`android/app/proguard-rules.pro`:
```
-keep class io.flutter.app.** { *; }
-keep class io.flutter.plugin.** { *; }
-dontwarn io.flutter.**

# Keep crash reporting symbols
-keepattributes SourceFile,LineNumberTable
```

Plugins often need extra rules — check their docs.

---

## 🚀 Recommended Learning Path

```
1. Dart basics + OOP + null safety + async/await
2. Flutter widgets + layout + navigation
3. State management (Provider → Riverpod)
4. HTTP + JSON + Firebase OR REST backend
5. Local storage + caching
6. Theming + responsive design
7. Animations (implicit → explicit → advanced)
8. Architecture (feature-first + Repository pattern + DI)
9. Testing (unit → widget → integration)
10. Performance + DevTools
11. Deployment + CI/CD
12. Advanced topics as needed
```

Build a **real app at each stage** — todo, weather, chat, e-commerce, social. Reading docs doesn't make you a Flutter dev; shipping does.

---

## 📦 Must-Know Packages Cheat Sheet

| Category | Packages |
|---|---|
| State Management | `flutter_riverpod`, `flutter_bloc`, `provider`, `getx` |
| Networking | `dio`, `http`, `retrofit` |
| JSON | `freezed`, `json_serializable` |
| Navigation | `go_router` |
| Local DB | `isar`, `drift`, `hive`, `sqflite` |
| Auth | `firebase_auth`, `supabase_flutter` |
| Storage | `shared_preferences`, `flutter_secure_storage` |
| UI | `cached_network_image`, `shimmer`, `flutter_svg`, `lottie` |
| Animation | `flutter_animate`, `rive` |
| Maps | `google_maps_flutter`, `flutter_map` |
| DI | `get_it`, `injectable` |
| Testing | `mocktail`, `flutter_test`, `integration_test` |
| Dev Tools | `very_good_analysis`, `flutter_launcher_icons`, `flutter_native_splash` |

---

## ✅ Phase Tracker

| Phase | Topic | Confident | Needs Review |
|---|---|---|---|
| 1 | Dart Fundamentals | ☐ | ☐ |
| 2 | Flutter Fundamentals | ☐ | ☐ |
| 3 | Styling & Theming | ☐ | ☐ |
| 4 | Navigation & Routing | ☐ | ☐ |
| 5 | State Management | ☐ | ☐ |
| 6 | Networking & Backend | ☐ | ☐ |
| 7 | Local Storage | ☐ | ☐ |
| 8 | Animations | ☐ | ☐ |
| 9 | Custom Painting | ☐ | ☐ |
| 10 | Platform Features | ☐ | ☐ |
| 11 | Architecture & Code Quality | ☐ | ☐ |
| 12 | Testing | ☐ | ☐ |
| 13 | Performance | ☐ | ☐ |
| 14 | Deployment & CI/CD | ☐ | ☐ |
| 15 | Advanced Topics | ☐ | ☐ |

---

*Flutter & Dart Complete Guide | Beginner → Advanced | Build, ship, iterate 🚀*