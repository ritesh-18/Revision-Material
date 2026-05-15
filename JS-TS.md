# JavaScript & TypeScript – Complete Revision Guide
### From Fundamentals to Advanced | Interview Ready

---

## Table of Contents

### Part 1 – JavaScript
1. [How JavaScript Works](#1-how-javascript-works)
2. [Variables – var, let, const](#2-variables--var-let-const)
3. [Data Types](#3-data-types)
4. [Type Coercion](#4-type-coercion)
5. [Operators](#5-operators)
6. [Functions](#6-functions)
7. [Scope & Closures](#7-scope--closures)
8. [Hoisting](#8-hoisting)
9. [The `this` Keyword](#9-the-this-keyword)
10. [Prototypes & Inheritance](#10-prototypes--inheritance)
11. [Classes](#11-classes)
12. [Arrays & Array Methods](#12-arrays--array-methods)
13. [Objects](#13-objects)
14. [Destructuring & Spread/Rest](#14-destructuring--spreadrest)
15. [Iterators & Generators](#15-iterators--generators)
16. [Promises & Async/Await](#16-promises--asyncawait)
17. [Event Loop & Concurrency](#17-event-loop--concurrency)
18. [Error Handling](#18-error-handling)
19. [Modules (ESM & CommonJS)](#19-modules-esm--commonjs)
20. [Maps, Sets, WeakMap, WeakSet](#20-maps-sets-weakmap-weakset)
21. [Symbols & Well-Known Symbols](#21-symbols--well-known-symbols)
22. [Proxy & Reflect](#22-proxy--reflect)
23. [Memory Management & Garbage Collection](#23-memory-management--garbage-collection)
24. [Design Patterns in JS](#24-design-patterns-in-js)

### Part 2 – TypeScript
25. [What is TypeScript & Why](#25-what-is-typescript--why)
26. [Basic Types](#26-basic-types)
27. [Type Inference](#27-type-inference)
28. [Interfaces vs Type Aliases](#28-interfaces-vs-type-aliases)
29. [Union & Intersection Types](#29-union--intersection-types)
30. [Literal Types & Template Literal Types](#30-literal-types--template-literal-types)
31. [Enums](#31-enums)
32. [Functions in TypeScript](#32-functions-in-typescript)
33. [Generics](#33-generics)
34. [Utility Types](#34-utility-types)
35. [Type Narrowing & Type Guards](#35-type-narrowing--type-guards)
36. [Classes in TypeScript](#36-classes-in-typescript)
37. [Decorators](#37-decorators)
38. [Modules & Namespaces](#38-modules--namespaces)
39. [Declaration Files (.d.ts)](#39-declaration-files-dts)
40. [tsconfig.json Deep Dive](#40-tsconfigjson-deep-dive)
41. [Advanced Types](#41-advanced-types)
42. [TypeScript with Node.js / NestJS Patterns](#42-typescript-with-nodejs--nestjs-patterns)

### Part 3 – 100 Interview Questions (No Solutions)
[Go to Questions](#part-3--100-interview-questions)

---

# PART 1 – JAVASCRIPT

---

## 1. How JavaScript Works

### The Engine
JavaScript runs inside an engine (V8 in Chrome/Node.js, SpiderMonkey in Firefox). The engine:
1. **Parses** source code into an AST (Abstract Syntax Tree).
2. **Compiles** AST to bytecode (JIT – Just In Time compilation).
3. **Executes** the bytecode.

### Execution Context
Every time code runs, an Execution Context is created. There are two types:
- **Global Execution Context (GEC):** Created when the script first runs. Creates `window` (browser) or `global` (Node.js).
- **Function Execution Context (FEC):** Created each time a function is called.

Each Execution Context has:
- **Variable Environment** – stores variables, functions.
- **Scope Chain** – reference to outer scopes.
- **`this` binding** – depends on how the function is called.

### Call Stack
JavaScript is single-threaded. The call stack tracks function execution.

```
Global Execution Context (always at bottom)
  → functionA Execution Context
    → functionB Execution Context  ← currently executing
```

```javascript
function greet(name) {
  return `Hello, ${name}`;
}

function main() {
  const msg = greet('Arjun');
  console.log(msg);
}

main();
// Call Stack:
// 1. main() pushed
// 2. greet() pushed
// 3. greet() returns, popped
// 4. console.log() pushed, popped
// 5. main() returns, popped
```

### JS is Single-Threaded but Non-Blocking
JS handles async operations via the **Event Loop** (covered in detail in Section 17).

---

## 2. Variables – var, let, const

### Declarations Compared

| Feature | var | let | const |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisted | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| Redeclarable | Yes | No | No |
| Reassignable | Yes | Yes | No |
| Global property | Yes | No | No |

### var – Function Scoped, Hoisted

```javascript
function example() {
  console.log(x); // undefined (hoisted, not error)
  var x = 10;
  console.log(x); // 10

  if (true) {
    var y = 20; // leaks outside the if block
  }
  console.log(y); // 20 — var doesn't respect block scope
}
```

### let – Block Scoped, TDZ

```javascript
{
  // console.log(a); // ReferenceError: Cannot access 'a' before initialization (TDZ)
  let a = 5;
  console.log(a); // 5
}
// console.log(a); // ReferenceError: a is not defined
```

**Temporal Dead Zone (TDZ):** The period between entering the block and the `let`/`const` declaration. Accessing the variable in TDZ throws a ReferenceError.

### const – Block Scoped, Cannot Reassign

```javascript
const PI = 3.14;
// PI = 3.15; // TypeError: Assignment to constant variable

// const for objects – binding is const, content is mutable
const user = { name: 'Arjun' };
user.name = 'Raj'; // ALLOWED – we changed the property, not the binding
// user = {}; // TypeError – cannot reassign the binding

// const for arrays
const arr = [1, 2, 3];
arr.push(4); // ALLOWED
// arr = []; // TypeError
```

### The Classic var Loop Bug

```javascript
// Problem with var
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3 — all share the same 'i'

// Fix with let
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2 — each iteration gets its own 'i'
```

---

## 3. Data Types

### Primitive Types (7)
Stored by value. Immutable.

```javascript
let num = 42;               // number
let big = 9007199254740993n; // bigint
let str = 'hello';          // string
let bool = true;            // boolean
let nothing = null;         // null (intentional absence)
let notDefined;             // undefined (uninitialized)
let id = Symbol('id');      // symbol (unique identifier)
```

### Reference Types
Stored by reference (heap). Objects, Arrays, Functions are all objects.

```javascript
const obj = { a: 1 };
const arr = [1, 2, 3];
const fn = function() {};

// Reference copy
const a = { x: 1 };
const b = a;       // b points to SAME object
b.x = 99;
console.log(a.x);  // 99 — a was mutated via b

// Value copy (primitives)
let x = 10;
let y = x;
y = 20;
console.log(x); // 10 — unaffected
```

### typeof

```javascript
typeof 42          // 'number'
typeof 'hello'     // 'string'
typeof true        // 'boolean'
typeof undefined   // 'undefined'
typeof null        // 'object' ← famous bug in JS
typeof {}          // 'object'
typeof []          // 'object' (use Array.isArray())
typeof function(){} // 'function'
typeof Symbol()    // 'symbol'
typeof 42n         // 'bigint'
```

### null vs undefined

```javascript
let a;              // undefined — declared but not assigned
let b = null;       // null — explicitly set to "no value"

console.log(a == b);  // true  (loose equality, both "empty")
console.log(a === b); // false (strict equality, different types)
```

---

## 4. Type Coercion

JavaScript automatically converts types in certain operations. This is a major source of bugs.

### Implicit Coercion

```javascript
// String + Number → String concatenation
'5' + 3      // '53' (3 converted to string)
'5' - 3      // 2   (coerced to number for math)
'5' * '2'    // 10
true + 1     // 2 (true → 1)
false + 1    // 1 (false → 0)
null + 1     // 1 (null → 0)
undefined + 1 // NaN (undefined → NaN)
'' + false   // 'false' (false → 'false')
```

### Loose vs Strict Equality

```javascript
// == does type coercion before comparing
0 == false      // true  (false → 0)
'' == false     // true  ('' → 0, false → 0)
null == undefined // true (special case)
1 == '1'        // true  ('1' → 1)
NaN == NaN      // false (NaN is never equal to anything)

// === never coerces
0 === false     // false
'' === false    // false
1 === '1'       // false

// Rule: ALWAYS use === unless you have specific reason
```

### Falsy Values
These values coerce to `false` in boolean context:

```javascript
false, 0, -0, 0n, '', null, undefined, NaN

// Everything else is truthy, including:
'0', [], {}, -1, Infinity
```

### Explicit Coercion

```javascript
// To Number
Number('42')     // 42
Number('')       // 0
Number(null)     // 0
Number(undefined)// NaN
Number(true)     // 1
+'42'            // 42 (unary plus)

// To String
String(42)       // '42'
(42).toString()  // '42'

// To Boolean
Boolean(0)       // false
Boolean('hi')    // true
!!0              // false (double NOT)
!!'hi'           // true
```

---

## 5. Operators

### Nullish Coalescing (??)
Returns right side only if left is `null` or `undefined` (not falsy).

```javascript
const name = null ?? 'Default';    // 'Default'
const count = 0 ?? 42;            // 0 — 0 is NOT null/undefined
const val = '' ?? 'fallback';     // '' — '' is NOT null/undefined

// Compare with ||
const a = 0 || 42;   // 42 — 0 is falsy
const b = 0 ?? 42;   // 0  — 0 is not null/undefined
```

### Optional Chaining (?.)
Safely access nested properties without throwing.

```javascript
const user = null;
console.log(user?.profile?.avatar); // undefined (no error)
console.log(user?.getName());       // undefined (no error)
console.log(user?.['email']);       // undefined (no error)

// Without optional chaining:
console.log(user && user.profile && user.profile.avatar); // old way
```

### Logical Assignment

```javascript
// ||= assigns if left is falsy
let a = null;
a ||= 'default';  // a = 'default'

// &&= assigns if left is truthy
let b = 'value';
b &&= 'updated'; // b = 'updated'

// ??= assigns if left is null/undefined
let c = 0;
c ??= 42; // c = 0 (0 is not null/undefined)
```

### Spread & Rest (covered in Section 14)

---

## 6. Functions

### Function Declaration vs Expression vs Arrow

```javascript
// 1. Function Declaration — hoisted fully
function greet(name) {
  return `Hello ${name}`;
}

// 2. Function Expression — not hoisted
const greet2 = function(name) {
  return `Hello ${name}`;
};

// 3. Arrow Function — no own 'this', not hoistable
const greet3 = (name) => `Hello ${name}`;

// Arrow with block body
const add = (a, b) => {
  const result = a + b;
  return result; // explicit return needed with {}
};
```

### Parameters

```javascript
// Default parameters
function greet(name = 'World') {
  return `Hello, ${name}`;
}

// Rest parameters (must be last)
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4); // 10

// Arguments object (only in regular functions, not arrows)
function oldStyle() {
  console.log(arguments); // array-like object
}
```

### Pure vs Impure Functions

```javascript
// Pure: same input → same output, no side effects
function add(a, b) {
  return a + b;
}

// Impure: depends on/modifies external state
let counter = 0;
function increment() {
  counter++; // side effect – modifies external state
  return counter;
}
```

### Higher-Order Functions
Functions that take or return other functions.

```javascript
// Takes a function
[1, 2, 3].map(x => x * 2); // [2, 4, 6]

// Returns a function
function multiplier(factor) {
  return (number) => number * factor;
}
const double = multiplier(2);
const triple = multiplier(3);
double(5); // 10
triple(5); // 15
```

### IIFE (Immediately Invoked Function Expression)

```javascript
(function() {
  // Runs immediately, creates own scope
  const privateVar = 'only here';
})();

// Arrow version
(() => {
  console.log('IIFE');
})();
```

### Currying

```javascript
// Regular function
function add(a, b, c) {
  return a + b + c;
}

// Curried version
function curriedAdd(a) {
  return function(b) {
    return function(c) {
      return a + b + c;
    };
  };
}

curriedAdd(1)(2)(3); // 6

// Arrow curried
const curry = a => b => c => a + b + c;
curry(1)(2)(3); // 6

// Practical use
const addTax = rate => price => price + price * rate;
const addGST = addTax(0.18);
addGST(100); // 118
addGST(200); // 236
```

---

## 7. Scope & Closures

### Scope Types

```javascript
// Global Scope
var globalVar = 'I am global';

function outer() {
  // Function Scope
  let outerVar = 'I am outer';

  function inner() {
    // Function Scope (nested)
    let innerVar = 'I am inner';
    console.log(outerVar); // Can access — closure
    console.log(globalVar); // Can access — scope chain
  }

  inner();
  // console.log(innerVar); // ReferenceError
}

{
  // Block Scope (let/const only)
  let blockVar = 'block';
  const blockConst = 'const';
  var leaks = 'I leak';
}
// console.log(blockVar); // ReferenceError
console.log(leaks); // 'I leak' — var ignores blocks
```

### Scope Chain
When JS looks for a variable, it searches from current scope up to global.

```javascript
const x = 'global';

function outer() {
  const x = 'outer';

  function inner() {
    const x = 'inner';
    console.log(x); // 'inner' — found in own scope
  }

  function middle() {
    console.log(x); // 'outer' — not in own scope, goes up
  }

  inner();
  middle();
}

outer();
```

### Closures

A closure is a function that **remembers** the variables from its outer scope even after the outer function has returned.

```javascript
function makeCounter() {
  let count = 0; // this variable lives in the closure

  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
}

const counter = makeCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.decrement(); // 1
counter.getCount();  // 1
// count is not accessible from outside — it's private
```

**Practical Uses of Closures:**

```javascript
// 1. Data encapsulation / private variables
function createPerson(name) {
  let _age = 0; // private
  return {
    getName: () => name,
    getAge: () => _age,
    birthday: () => _age++,
  };
}

// 2. Memoization
function memoize(fn) {
  const cache = {};
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache[key]) return cache[key];
    cache[key] = fn(...args);
    return cache[key];
  };
}

const memoFib = memoize(function fib(n) {
  if (n <= 1) return n;
  return memoFib(n - 1) + memoFib(n - 2);
});

// 3. Partial application
function partial(fn, ...presetArgs) {
  return function(...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

const log = (level, message) => `[${level}] ${message}`;
const logError = partial(log, 'ERROR');
logError('Something failed'); // '[ERROR] Something failed'
```

---

## 8. Hoisting

Hoisting is JS's behavior of moving declarations to the top of their scope before execution.

### What Gets Hoisted?

```javascript
// var declarations — hoisted, initialized to undefined
console.log(a); // undefined (not ReferenceError)
var a = 5;

// Function declarations — hoisted fully (with body)
greet(); // 'Hello' — works before declaration
function greet() { console.log('Hello'); }

// let / const — hoisted but NOT initialized (TDZ)
// console.log(b); // ReferenceError: Cannot access before initialization
let b = 10;

// Function expressions — NOT hoisted (only the var is)
// sayHi(); // TypeError: sayHi is not a function
var sayHi = function() { console.log('Hi'); };

// Arrow functions — NOT hoisted
// hi(); // TypeError
const hi = () => console.log('Hi');
```

### What Actually Happens (Behind the Scenes)

```javascript
// What you write:
console.log(x);
var x = 5;

// How JS sees it:
var x; // declaration moved to top
console.log(x); // undefined
x = 5; // assignment stays in place
```

---

## 9. The `this` Keyword

`this` refers to the object that is currently executing the function. Its value depends on **how** the function is called.

### Rules for `this`

```javascript
// 1. Global context — this = global object (window/global)
console.log(this); // Window in browser, {} in Node.js module

// 2. Method call — this = the object before the dot
const user = {
  name: 'Arjun',
  greet() {
    console.log(this.name); // 'Arjun'
  }
};
user.greet();

// 3. Regular function call — this = undefined (strict mode) or global
function standalone() {
  console.log(this); // undefined in strict mode
}

// 4. Arrow function — NO own this, inherits from enclosing scope
const obj = {
  name: 'Arjun',
  greet: () => {
    console.log(this.name); // undefined — arrow inherits outer 'this' (global)
  },
  greetCorrect() {
    const arrow = () => console.log(this.name); // 'Arjun' — inherits method's 'this'
    arrow();
  }
};

// 5. Constructor call — this = newly created object
function Person(name) {
  this.name = name; // this = new object
}
const p = new Person('Arjun');
console.log(p.name); // 'Arjun'

// 6. Explicit binding — call, apply, bind
function greet(greeting) {
  return `${greeting}, ${this.name}`;
}
const person = { name: 'Arjun' };

greet.call(person, 'Hello');   // 'Hello, Arjun'
greet.apply(person, ['Hi']);   // 'Hi, Arjun'
const bound = greet.bind(person);
bound('Hey');                  // 'Hey, Arjun'
```

### call vs apply vs bind

```javascript
function introduce(greeting, punctuation) {
  return `${greeting}, I am ${this.name}${punctuation}`;
}

const user = { name: 'Arjun' };

// call — passes args one by one, executes immediately
introduce.call(user, 'Hello', '!'); // 'Hello, I am Arjun!'

// apply — passes args as array, executes immediately
introduce.apply(user, ['Hi', '.']); // 'Hi, I am Arjun.'

// bind — returns NEW function with 'this' bound, doesn't execute
const boundFn = introduce.bind(user, 'Hey');
boundFn('?'); // 'Hey, I am Arjun?'
```

### Lost this Problem & Fix

```javascript
const timer = {
  seconds: 0,
  start() {
    // Problem: setInterval callback loses 'this'
    setInterval(function() {
      this.seconds++; // 'this' is window/undefined here!
    }, 1000);

    // Fix 1: Arrow function
    setInterval(() => {
      this.seconds++; // 'this' = timer object ✓
    }, 1000);

    // Fix 2: bind
    setInterval(function() {
      this.seconds++;
    }.bind(this), 1000);
  }
};
```

---

## 10. Prototypes & Inheritance

### Prototype Chain
Every object has a hidden `[[Prototype]]` property pointing to another object. When you access a property, JS looks up the chain until it finds it or reaches `null`.

```javascript
const animal = {
  breathe() { return 'breathing'; }
};

const dog = Object.create(animal); // dog's prototype = animal
dog.bark = function() { return 'woof'; };

dog.bark();    // 'woof' — found on dog
dog.breathe(); // 'breathing' — not on dog, found on animal (prototype)
dog.toString() // found on Object.prototype (top of chain)

// Check prototype
Object.getPrototypeOf(dog) === animal; // true
```

### Constructor Functions & prototype

```javascript
function Animal(name) {
  this.name = name; // instance property
}

Animal.prototype.speak = function() { // shared method
  return `${this.name} makes a sound`;
};

const dog = new Animal('Rex');
const cat = new Animal('Whiskers');

dog.speak(); // 'Rex makes a sound'
cat.speak(); // 'Whiskers makes a sound'

// Both share the same speak function — memory efficient
dog.speak === cat.speak; // true
```

### Prototypal Inheritance

```javascript
function Animal(name) {
  this.name = name;
}
Animal.prototype.eat = function() { return `${this.name} eats`; };

function Dog(name, breed) {
  Animal.call(this, name); // call parent constructor
  this.breed = breed;
}

// Set up prototype chain
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // fix constructor reference

Dog.prototype.bark = function() { return `${this.name} barks`; };

const d = new Dog('Rex', 'Labrador');
d.eat();  // 'Rex eats' — inherited
d.bark(); // 'Rex barks' — own
```

### hasOwnProperty

```javascript
const obj = { a: 1 };
'a' in obj;                // true (includes prototype chain)
obj.hasOwnProperty('a');   // true (own property only)
obj.hasOwnProperty('toString'); // false (on prototype)
```

---

## 11. Classes

ES6 Classes are syntactic sugar over prototype-based inheritance.

```javascript
class Animal {
  // Constructor
  constructor(name, sound) {
    this.name = name;
    this.sound = sound;
  }

  // Instance method (on prototype)
  speak() {
    return `${this.name} says ${this.sound}`;
  }

  // Static method (on class itself, not instances)
  static create(name, sound) {
    return new Animal(name, sound);
  }

  // Getter
  get info() {
    return `Animal: ${this.name}`;
  }

  // Setter
  set info(value) {
    this.name = value;
  }
}

const dog = new Animal('Rex', 'woof');
dog.speak(); // 'Rex says woof'
Animal.create('Cat', 'meow'); // static
dog.info; // 'Animal: Rex'
```

### Inheritance with extends & super

```javascript
class Dog extends Animal {
  constructor(name, breed) {
    super(name, 'woof'); // MUST call super() before using 'this'
    this.breed = breed;
  }

  // Override parent method
  speak() {
    return `${super.speak()} (loudly)`; // call parent method
  }

  fetch() {
    return `${this.name} fetches the ball`;
  }
}

const d = new Dog('Rex', 'Labrador');
d.speak();  // 'Rex says woof (loudly)'
d.fetch();  // 'Rex fetches the ball'
d instanceof Dog;    // true
d instanceof Animal; // true
```

### Private Fields (ES2022)

```javascript
class BankAccount {
  #balance = 0; // private field — truly inaccessible from outside

  deposit(amount) {
    if (amount > 0) this.#balance += amount;
  }

  get balance() {
    return this.#balance;
  }
}

const acc = new BankAccount();
acc.deposit(1000);
acc.balance; // 1000
// acc.#balance; // SyntaxError
```

---

## 12. Arrays & Array Methods

### Core Methods (Immutable — return new array)

```javascript
const nums = [1, 2, 3, 4, 5];

// map — transform each element
nums.map(n => n * 2);          // [2, 4, 6, 8, 10]

// filter — keep elements that pass test
nums.filter(n => n % 2 === 0); // [2, 4]

// reduce — reduce to single value
nums.reduce((acc, n) => acc + n, 0); // 15
nums.reduce((acc, n) => acc + n);    // 15 (no initial value, uses first element)

// find — first element that matches
nums.find(n => n > 3);         // 4

// findIndex — index of first match
nums.findIndex(n => n > 3);    // 3

// some — at least one matches
nums.some(n => n > 4);         // true

// every — all match
nums.every(n => n > 0);        // true

// includes — check if value exists
nums.includes(3);              // true

// flat — flatten nested arrays
[1, [2, [3]]].flat();   // [1, 2, [3]]
[1, [2, [3]]].flat(Infinity); // [1, 2, 3]

// flatMap — map then flat(1)
[1, 2, 3].flatMap(n => [n, n * 2]); // [1, 2, 2, 4, 3, 6]
```

### Mutating Methods (Modify original)

```javascript
const arr = [1, 2, 3];

arr.push(4);       // [1, 2, 3, 4] — add to end
arr.pop();         // removes last
arr.shift();       // removes first
arr.unshift(0);    // add to start
arr.splice(1, 1);  // remove 1 element at index 1
arr.sort((a, b) => a - b); // sort ascending
arr.reverse();     // reverse in place
```

### Useful Patterns

```javascript
// Unique values
const unique = [...new Set([1, 2, 2, 3, 3])]; // [1, 2, 3]

// Flatten and sort
const nested = [[3,1],[2,4]];
nested.flat().sort((a,b) => a - b); // [1, 2, 3, 4]

// Group by (ES2024 Object.groupBy)
const people = [{name:'A',age:20},{name:'B',age:20},{name:'C',age:25}];
Object.groupBy(people, p => p.age);
// { 20: [{name:'A',...},{name:'B',...}], 25: [{name:'C',...}] }

// Reduce to object (group by manually)
people.reduce((acc, p) => {
  acc[p.age] = acc[p.age] || [];
  acc[p.age].push(p);
  return acc;
}, {});
```

---

## 13. Objects

### Creation

```javascript
// Object literal
const obj = { name: 'Arjun', age: 25 };

// Object.create
const proto = { greet() { return `Hi, ${this.name}`; } };
const obj2 = Object.create(proto);
obj2.name = 'Arjun';

// Constructor
function User(name) { this.name = name; }
const u = new User('Arjun');

// Class
class Product { constructor(name) { this.name = name; } }
```

### Object Methods

```javascript
const obj = { a: 1, b: 2, c: 3 };

Object.keys(obj);    // ['a', 'b', 'c']
Object.values(obj);  // [1, 2, 3]
Object.entries(obj); // [['a',1],['b',2],['c',3]]

// From entries back to object
Object.fromEntries([['a',1],['b',2]]); // { a: 1, b: 2 }

// Merge objects
const merged = Object.assign({}, obj, { d: 4 });
const merged2 = { ...obj, d: 4 }; // spread (preferred)

// Freeze — no modifications
const frozen = Object.freeze({ x: 1 });
frozen.x = 99; // silently fails (throws in strict mode)

// Seal — no add/delete, can modify existing
const sealed = Object.seal({ x: 1 });
sealed.x = 99; // allowed
sealed.y = 2;  // silently fails

// Check property
'a' in obj;                    // true
obj.hasOwnProperty('a');       // true
Object.prototype.hasOwnProperty.call(obj, 'a'); // safe version
```

### Computed Properties & Shorthand

```javascript
const key = 'name';
const value = 'Arjun';

const obj = {
  [key]: value,            // computed: { name: 'Arjun' }
  [key + 'Upper']: value.toUpperCase(), // 'nameUpper': 'ARJUN'
};

// Shorthand property
const x = 1, y = 2;
const point = { x, y }; // { x: 1, y: 2 }

// Shorthand method
const calc = {
  add(a, b) { return a + b; }, // instead of add: function(a,b)
};
```

### Deep Clone

```javascript
// Shallow clone (only one level deep)
const clone1 = Object.assign({}, original);
const clone2 = { ...original };

// Deep clone options:
// 1. JSON (loses functions, undefined, Date objects)
const deep1 = JSON.parse(JSON.stringify(original));

// 2. structuredClone (modern, handles most types)
const deep2 = structuredClone(original);
```

---

## 14. Destructuring & Spread/Rest

### Object Destructuring

```javascript
const user = { name: 'Arjun', age: 25, city: 'Bengaluru' };

// Basic
const { name, age } = user;

// Rename
const { name: userName, age: userAge } = user;

// Default value
const { country = 'India' } = user; // 'India' if undefined

// Nested
const data = { user: { profile: { avatar: 'url' } } };
const { user: { profile: { avatar } } } = data;

// In function parameters
function greet({ name, age = 0 }) {
  return `${name} is ${age}`;
}
greet(user);

// Rest
const { name: n, ...rest } = user;
// rest = { age: 25, city: 'Bengaluru' }
```

### Array Destructuring

```javascript
const colors = ['red', 'green', 'blue'];

const [first, second] = colors;
const [head, ...tail] = colors; // tail = ['green', 'blue']

// Skip elements
const [,, third] = colors; // 'blue'

// Default values
const [a = 'default'] = [];  // 'default'

// Swap variables
let x = 1, y = 2;
[x, y] = [y, x]; // x=2, y=1

// From function return
function getCoords() { return [10, 20]; }
const [lat, lng] = getCoords();
```

### Spread Operator

```javascript
// Arrays
const a = [1, 2, 3];
const b = [4, 5, 6];
const combined = [...a, ...b]; // [1,2,3,4,5,6]

// Objects
const defaults = { color: 'blue', size: 'M' };
const custom = { size: 'L', weight: 'light' };
const merged = { ...defaults, ...custom };
// { color: 'blue', size: 'L', weight: 'light' } — custom overrides defaults

// Function arguments
Math.max(...[1, 5, 3, 9, 2]); // 9

// Clone
const original = { x: 1 };
const copy = { ...original }; // shallow clone
```

### Rest Parameters

```javascript
// In function — collect remaining args into array
function sum(first, second, ...rest) {
  return first + second + rest.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4, 5); // 15
```

---

## 15. Iterators & Generators

### Iterators
An object is iterable if it has a `[Symbol.iterator]` method returning an iterator.

```javascript
// Custom iterable
const range = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      }
    };
  }
};

for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}

[...range]; // [1, 2, 3, 4, 5]
```

### Generators
Functions that can pause and resume. Marked with `*`.

```javascript
function* counter(start = 0) {
  while (true) {
    yield start++; // pause here, return value
  }
}

const gen = counter(5);
gen.next(); // { value: 5, done: false }
gen.next(); // { value: 6, done: false }
gen.next(); // { value: 7, done: false }

// Finite generator
function* range(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}

[...range(1, 5)]; // [1, 2, 3, 4, 5]

// Generator with return
function* gen2() {
  yield 1;
  return 'done'; // { value: 'done', done: true }
  yield 2;       // never reached
}

// Practical: async-like flow control
function* fetchUserFlow() {
  const userId = yield getUserId();
  const user = yield getUser(userId);
  const posts = yield getPosts(user);
  return posts;
}
```

---

## 16. Promises & Async/Await

### Callback Hell (the problem)

```javascript
getUser(id, function(user) {
  getPosts(user.id, function(posts) {
    getComments(posts[0].id, function(comments) {
      // deeply nested, hard to read and error-handle
    });
  });
});
```

### Promises

A Promise is an object representing eventual completion or failure of an async operation.

**States:** Pending → Fulfilled (resolved) | Rejected

```javascript
// Creating a Promise
const fetchUser = (id) => new Promise((resolve, reject) => {
  setTimeout(() => {
    if (id > 0) resolve({ id, name: 'Arjun' });
    else reject(new Error('Invalid ID'));
  }, 1000);
});

// Consuming
fetchUser(1)
  .then(user => {
    console.log(user); // { id: 1, name: 'Arjun' }
    return user.name; // pass to next .then
  })
  .then(name => console.log(name)) // 'Arjun'
  .catch(err => console.error(err.message))
  .finally(() => console.log('Done')); // always runs
```

### Promise Methods

```javascript
const p1 = Promise.resolve(1);
const p2 = Promise.resolve(2);
const p3 = Promise.reject(new Error('oops'));

// Promise.all — all must succeed, fails fast
Promise.all([p1, p2])
  .then(([v1, v2]) => console.log(v1, v2)); // 1, 2

Promise.all([p1, p3])
  .catch(err => console.log(err.message)); // 'oops' — one failed

// Promise.allSettled — waits for all, never rejects
Promise.allSettled([p1, p3])
  .then(results => {
    results.forEach(r => {
      if (r.status === 'fulfilled') console.log(r.value);
      else console.log(r.reason.message);
    });
  });

// Promise.race — first to settle wins
Promise.race([
  new Promise(res => setTimeout(() => res('slow'), 2000)),
  new Promise(res => setTimeout(() => res('fast'), 100)),
]).then(v => console.log(v)); // 'fast'

// Promise.any — first to FULFILL wins (ignores rejections)
Promise.any([p3, p1, p2])
  .then(v => console.log(v)); // 1 (first fulfilled)
```

### Async/Await

Syntactic sugar over Promises. Makes async code read like synchronous.

```javascript
async function loadUserData(userId) {
  try {
    const user = await fetchUser(userId);    // waits for promise
    const posts = await fetchPosts(user.id); // waits for promise
    const comments = await fetchComments(posts[0].id);
    return { user, posts, comments };
  } catch (error) {
    console.error('Failed:', error.message);
    throw error; // re-throw if needed
  }
}

// Always returns a Promise
loadUserData(1).then(data => console.log(data));
```

### Parallel vs Sequential

```javascript
// SEQUENTIAL (slow) — each waits for the previous
async function sequential() {
  const user = await fetchUser(1);    // 1000ms
  const posts = await fetchPosts(1);  // 1000ms
  // Total: 2000ms
}

// PARALLEL (fast) — all start at once
async function parallel() {
  const [user, posts] = await Promise.all([
    fetchUser(1),    // starts immediately
    fetchPosts(1),   // starts immediately
  ]);
  // Total: ~1000ms (max of both)
}
```

### Common Async Mistakes

```javascript
// Mistake 1: Forgetting await
async function wrong() {
  const data = fetchUser(1); // data is a Promise, not the user!
  console.log(data.name); // undefined
}

// Mistake 2: await in forEach (doesn't work as expected)
async function wrongLoop() {
  const ids = [1, 2, 3];
  ids.forEach(async (id) => {
    const user = await fetchUser(id); // forEach doesn't await these
    console.log(user);
  });
  // forEach returns before any of the async operations finish
}

// Fix: use for...of
async function correctLoop() {
  const ids = [1, 2, 3];
  for (const id of ids) {
    const user = await fetchUser(id); // awaited correctly
    console.log(user);
  }
}

// Or use Promise.all for parallel
async function parallelLoop() {
  const ids = [1, 2, 3];
  const users = await Promise.all(ids.map(id => fetchUser(id)));
}
```

---

## 17. Event Loop & Concurrency

### What is the Event Loop?

JavaScript is single-threaded but non-blocking. The Event Loop enables async operations.

**Components:**
- **Call Stack** – Executes synchronous code.
- **Web APIs / Node APIs** – Handle async operations (setTimeout, fetch, I/O).
- **Callback Queue (Macrotask Queue)** – Holds callbacks from setTimeout, setInterval, I/O.
- **Microtask Queue** – Holds Promise callbacks, queueMicrotask. **Higher priority than macrotask.**

### Event Loop Algorithm

```
1. Execute all synchronous code (drain call stack)
2. Process ALL microtasks (Promise .then, queueMicrotask)
3. Render (browsers only)
4. Process ONE macrotask (setTimeout, setInterval)
5. Repeat from step 2
```

### Priority: Microtasks > Macrotasks

```javascript
console.log('1 - sync');

setTimeout(() => console.log('4 - macrotask (setTimeout)'), 0);

Promise.resolve()
  .then(() => console.log('2 - microtask (Promise)'))
  .then(() => console.log('3 - microtask (Promise chain'));

queueMicrotask(() => console.log('2.5 - microtask'));

console.log('1.5 - sync');

// Output:
// 1 - sync
// 1.5 - sync
// 2 - microtask (Promise)
// 2.5 - microtask
// 3 - microtask (Promise chain)
// 4 - macrotask (setTimeout)
```

### Macrotasks vs Microtasks

| Macrotasks | Microtasks |
|---|---|
| setTimeout | Promise.then/catch/finally |
| setInterval | queueMicrotask |
| setImmediate (Node) | MutationObserver (browser) |
| I/O callbacks | |
| UI rendering | |

### setTimeout(fn, 0) is not instant

```javascript
console.log('start');
setTimeout(() => console.log('timeout'), 0); // queued as macrotask
console.log('end');
// Output: start, end, timeout
```

---

## 18. Error Handling

### try / catch / finally

```javascript
function divide(a, b) {
  if (b === 0) throw new Error('Division by zero');
  return a / b;
}

try {
  const result = divide(10, 0);
} catch (error) {
  console.error(error.message); // 'Division by zero'
  console.error(error.stack);   // stack trace
} finally {
  console.log('Always runs'); // cleanup here
}
```

### Error Types

```javascript
new Error('generic');
new TypeError('wrong type');
new RangeError('out of range');
new ReferenceError('variable not found');
new SyntaxError('invalid syntax');

// Custom Error
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

try {
  throw new ValidationError('Email is invalid', 'email');
} catch (e) {
  if (e instanceof ValidationError) {
    console.log(e.field); // 'email'
  }
}
```

### Async Error Handling

```javascript
// With async/await — use try/catch
async function fetchData() {
  try {
    const data = await riskyFetch();
    return data;
  } catch (error) {
    // handle error
    throw error; // re-throw if needed
  }
}

// Global unhandled rejections (Node.js)
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
});
```

---

## 19. Modules (ESM & CommonJS)

### CommonJS (Node.js traditional)

```javascript
// exporting
module.exports = { add, subtract };
module.exports.add = add; // named
exports.add = add; // shorthand (same as above)

// importing
const { add } = require('./math');
const math = require('./math');
```

### ES Modules (ESM – modern)

```javascript
// Named exports
export const PI = 3.14;
export function add(a, b) { return a + b; }
export class Calculator { ... }

// Default export (one per file)
export default class App { ... }

// Re-export
export { add } from './math';
export * from './helpers';

// Importing
import { add, PI } from './math';
import App from './App'; // default
import * as math from './math'; // namespace
import { add as sum } from './math'; // alias

// Dynamic import (lazy loading)
const module = await import('./heavy-module');
```

### Key Differences

| Feature | CommonJS | ESM |
|---|---|---|
| Syntax | require/module.exports | import/export |
| Loading | Synchronous | Asynchronous |
| Tree-shaking | No | Yes |
| Top-level await | No | Yes |
| Default in | Node.js | Browsers + Modern Node |

---

## 20. Maps, Sets, WeakMap, WeakSet

### Map vs Object

```javascript
const map = new Map();

// Keys can be ANY type (objects, functions, primitives)
map.set('name', 'Arjun');
map.set(42, 'the answer');
map.set({}, 'object key');

map.get('name');       // 'Arjun'
map.has('name');       // true
map.size;              // 3
map.delete('name');
map.clear();

// Iteration
for (const [key, value] of map) { ... }
map.forEach((value, key) => { ... });

// From object
const obj = { a: 1, b: 2 };
const fromObj = new Map(Object.entries(obj));

// Use Map over Object when:
// - keys are not strings/symbols
// - need insertion order
// - frequent add/delete
```

### Set

```javascript
const set = new Set([1, 2, 2, 3, 3, 3]);
set.size; // 3 — duplicates removed

set.add(4);
set.has(3);    // true
set.delete(2);

// Convert to array
[...set]; // [1, 3, 4]

// Use cases
const unique = new Set(array); // remove duplicates
const union = new Set([...setA, ...setB]);
const intersection = new Set([...setA].filter(x => setB.has(x)));
const difference = new Set([...setA].filter(x => !setB.has(x)));
```

### WeakMap & WeakSet

Hold **weak references** — don't prevent garbage collection.

```javascript
// WeakMap: keys must be objects
const wm = new WeakMap();
let obj = {};
wm.set(obj, 'metadata');
obj = null; // obj can be GC'd; entry auto-removed from WeakMap

// Use case: attach private data to objects without memory leaks
const _private = new WeakMap();
class User {
  constructor(name) {
    _private.set(this, { password: 'secret' });
    this.name = name;
  }
  getPassword() {
    return _private.get(this).password;
  }
}

// WeakSet: objects only
const ws = new WeakSet();
ws.add(someObject);
```

---

## 21. Symbols & Well-Known Symbols

```javascript
// Symbols are unique, primitive values
const id1 = Symbol('id');
const id2 = Symbol('id');
id1 === id2; // false — every Symbol is unique

// Use as unique object keys (won't clash)
const USER_ID = Symbol('userId');
const obj = { [USER_ID]: 123, name: 'Arjun' };
obj[USER_ID]; // 123

// Not enumerable by default
Object.keys(obj); // ['name'] — Symbol hidden
Object.getOwnPropertySymbols(obj); // [Symbol(userId)]

// Well-Known Symbols — customize built-in behaviors
class MyArray {
  static [Symbol.hasInstance](instance) {
    return Array.isArray(instance);
  }
}
[] instanceof MyArray; // true

class Range {
  constructor(from, to) { this.from = from; this.to = to; }
  [Symbol.iterator]() { // make iterable
    let n = this.from;
    return { next: () => n <= this.to ? { value: n++, done: false } : { done: true } };
  }
}

[...new Range(1, 5)]; // [1, 2, 3, 4, 5]
```

---

## 22. Proxy & Reflect

### Proxy
Intercepts and customizes operations on objects.

```javascript
const user = { name: 'Arjun', age: 25 };

const proxy = new Proxy(user, {
  // Intercept property reads
  get(target, prop) {
    console.log(`Reading ${prop}`);
    return prop in target ? target[prop] : `Property ${prop} not found`;
  },

  // Intercept property writes
  set(target, prop, value) {
    if (prop === 'age' && typeof value !== 'number') {
      throw new TypeError('Age must be a number');
    }
    target[prop] = value;
    return true; // must return true
  },

  // Intercept has (in operator)
  has(target, prop) {
    return prop in target;
  }
});

proxy.name;     // logs 'Reading name', returns 'Arjun'
proxy.email;    // 'Property email not found'
proxy.age = 30; // sets age
proxy.age = 'thirty'; // TypeError
```

### Reflect
Provides default implementations of Proxy traps. Use inside Proxy handlers.

```javascript
const proxy2 = new Proxy(user, {
  get(target, prop, receiver) {
    console.log(`Get: ${prop}`);
    return Reflect.get(target, prop, receiver); // default behavior
  },
  set(target, prop, value, receiver) {
    console.log(`Set: ${prop} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  }
});
```

---

## 23. Memory Management & Garbage Collection

### How GC Works
JavaScript uses **Mark and Sweep** algorithm:
1. GC marks all reachable objects from root (global, stack).
2. Sweeps (deletes) everything not marked.

```javascript
// Memory leak examples:
// 1. Accidental globals
function leak() {
  leakyVar = 'oops'; // no var/let/const — becomes global
}

// 2. Forgotten timers
const interval = setInterval(() => {
  bigData.process(); // bigData won't be GC'd
}, 1000);
clearInterval(interval); // must clear when done

// 3. DOM references
let element = document.getElementById('btn');
document.body.removeChild(element);
// element variable still holds reference — leak!
element = null; // fix

// 4. Closures holding references
function outer() {
  const bigArray = new Array(1000000);
  return function inner() {
    return bigArray.length; // inner keeps bigArray alive
  };
}
```

### WeakMap/WeakSet for Memory Efficiency
(See Section 20 — keys are weakly referenced, allowing GC when no other references exist.)

---

## 24. Design Patterns in JS

### Singleton

```javascript
class DatabaseConnection {
  static #instance = null;

  constructor(url) {
    if (DatabaseConnection.#instance) {
      return DatabaseConnection.#instance;
    }
    this.url = url;
    this.connected = false;
    DatabaseConnection.#instance = this;
  }

  static getInstance(url) {
    if (!DatabaseConnection.#instance) {
      new DatabaseConnection(url);
    }
    return DatabaseConnection.#instance;
  }
}

const db1 = DatabaseConnection.getInstance('postgres://...');
const db2 = DatabaseConnection.getInstance('postgres://...');
db1 === db2; // true — same instance
```

### Observer / EventEmitter

```javascript
class EventEmitter {
  #listeners = new Map();

  on(event, listener) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, []);
    }
    this.#listeners.get(event).push(listener);
    return this;
  }

  emit(event, ...args) {
    this.#listeners.get(event)?.forEach(listener => listener(...args));
    return this;
  }

  off(event, listener) {
    const listeners = this.#listeners.get(event) || [];
    this.#listeners.set(event, listeners.filter(l => l !== listener));
    return this;
  }
}

const emitter = new EventEmitter();
emitter.on('data', payload => console.log(payload));
emitter.emit('data', { user: 'Arjun' });
```

### Factory

```javascript
class UserFactory {
  static create(role) {
    switch (role) {
      case 'admin': return new AdminUser();
      case 'guest': return new GuestUser();
      default: throw new Error(`Unknown role: ${role}`);
    }
  }
}

const admin = UserFactory.create('admin');
```

### Decorator Pattern (not TS decorators)

```javascript
function withLogging(fn) {
  return function(...args) {
    console.log(`Calling ${fn.name} with`, args);
    const result = fn(...args);
    console.log(`Result:`, result);
    return result;
  };
}

const add = (a, b) => a + b;
const loggedAdd = withLogging(add);
loggedAdd(2, 3); // logs call and result
```

---

# PART 2 – TYPESCRIPT

---

## 25. What is TypeScript & Why

TypeScript is a **statically typed superset of JavaScript** developed by Microsoft. It compiles down to plain JavaScript.

**Why use TypeScript:**
- Catch errors at **compile time** instead of runtime.
- Better IDE support (autocomplete, refactoring).
- Self-documenting code (types as documentation).
- Safer refactoring in large codebases.
- Required for frameworks like Angular; strongly recommended for NestJS.

```typescript
// JS: runtime error
function greet(name) {
  return name.toUpperCase();
}
greet(42); // runtime error: name.toUpperCase is not a function

// TS: compile time error
function greet(name: string): string {
  return name.toUpperCase();
}
greet(42); // Error: Argument of type 'number' is not assignable to parameter of type 'string'
```

---

## 26. Basic Types

```typescript
// Primitives
let age: number = 25;
let name: string = 'Arjun';
let isActive: boolean = true;
let n: null = null;
let u: undefined = undefined;
let big: bigint = 9007199254740991n;
let sym: symbol = Symbol('id');

// Arrays
let nums: number[] = [1, 2, 3];
let strs: Array<string> = ['a', 'b']; // generic syntax

// Tuple — fixed length, known types at each position
let point: [number, number] = [10, 20];
let entry: [string, number] = ['age', 25];

// any — disables type checking (avoid!)
let data: any = 'could be anything';
data = 42; // allowed

// unknown — safer than any, must narrow before use
let value: unknown = getData();
if (typeof value === 'string') {
  value.toUpperCase(); // only allowed after check
}

// void — function returns nothing
function log(msg: string): void {
  console.log(msg);
}

// never — function never returns (throws or infinite loop)
function fail(msg: string): never {
  throw new Error(msg);
}

// object — non-primitive
let obj: object = { a: 1 };

// Object with shape
let user: { name: string; age: number } = { name: 'Arjun', age: 25 };
```

---

## 27. Type Inference

TypeScript infers types when you initialize variables. You don't always need to annotate.

```typescript
let x = 42;         // inferred: number
let s = 'hello';    // inferred: string
let arr = [1,2,3];  // inferred: number[]
let obj = { a: 1 }; // inferred: { a: number }

// Function return type inferred
function add(a: number, b: number) {
  return a + b; // return type inferred as number
}

// contextual typing
window.addEventListener('click', (e) => {
  // e inferred as MouseEvent
  console.log(e.clientX, e.clientY);
});

// When to annotate explicitly:
// 1. Function parameters (always)
// 2. When inference is too wide
// 3. Public API boundaries
// 4. Complex objects
```

---

## 28. Interfaces vs Type Aliases

### Interface

```typescript
interface User {
  id: number;
  name: string;
  email?: string; // optional
  readonly createdAt: Date; // cannot be changed after creation
}

// Extending interface
interface AdminUser extends User {
  role: 'admin';
  permissions: string[];
}

// Declaration merging (only interfaces)
interface Window {
  myPlugin: () => void;
}
interface Window {
  anotherPlugin: () => void;
}
// Merged: Window has both myPlugin and anotherPlugin
```

### Type Alias

```typescript
type User = {
  id: number;
  name: string;
};

// Can represent any type (not just objects)
type ID = string | number;
type Status = 'active' | 'inactive' | 'pending';
type Callback = (error: Error | null, data?: any) => void;

// Intersection (extending)
type AdminUser = User & {
  role: 'admin';
  permissions: string[];
};
```

### When to Use Which

| Feature | Interface | Type |
|---|---|---|
| Declaration merging | ✅ Yes | ❌ No |
| Extending | `extends` | `&` |
| Primitives/unions | ❌ No | ✅ Yes |
| Computed properties | Limited | ✅ Yes |
| Preferred for | Object shapes, OOP, libraries | Unions, computed, complex |

**General rule:** Use `interface` for object shapes (especially class contracts). Use `type` for everything else.

---

## 29. Union & Intersection Types

### Union ( | )
Value can be one of several types.

```typescript
type StringOrNumber = string | number;
let id: StringOrNumber = 'abc';
id = 123; // both allowed

// Discriminated unions — add a 'kind' or 'type' field
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2;
    case 'rectangle': return shape.width * shape.height;
    case 'triangle': return 0.5 * shape.base * shape.height;
  }
}
```

### Intersection ( & )
Value must satisfy ALL types simultaneously.

```typescript
type Serializable = { serialize(): string };
type Loggable = { log(): void };

type LoggableUser = User & Serializable & Loggable;
// Must have all User fields AND serialize() AND log()

// Practical example
type WithTimestamps<T> = T & {
  createdAt: Date;
  updatedAt: Date;
};

type UserWithTimestamps = WithTimestamps<User>;
```

---

## 30. Literal Types & Template Literal Types

### Literal Types
Restrict to exact values.

```typescript
type Direction = 'up' | 'down' | 'left' | 'right';
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;
type Status = 'pending' | 'active' | 'inactive';

function move(direction: Direction) {
  // only 'up','down','left','right' accepted
}

// widening
let dir = 'up'; // type: string (widened)
const dir2 = 'up'; // type: 'up' (literal, because const)

// Force literal type
let dir3 = 'up' as const; // type: 'up'
```

### Template Literal Types

```typescript
type EventName = 'click' | 'focus' | 'blur';
type EventHandler = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur'

type PropKey = 'name' | 'age';
type GetterKey = `get${Capitalize<PropKey>}`;
// 'getName' | 'getAge'

// Practical: typed CSS properties
type CSSValue = 'px' | 'em' | 'rem' | '%';
type CSSProp = `${number}${CSSValue}`;
// '16px', '1.5em', '100%' etc.
```

---

## 31. Enums

```typescript
// Numeric enum (default)
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right  // 3
}

Direction.Up;   // 0
Direction[0];   // 'Up' (reverse mapping)

// String enum (preferred — more debuggable)
enum Status {
  Pending = 'PENDING',
  Active = 'ACTIVE',
  Inactive = 'INACTIVE',
}

Status.Pending; // 'PENDING'

// Const enum — inlined at compile time (no JS object generated)
const enum Size {
  Small = 1,
  Medium = 2,
  Large = 3,
}
// Compiled: uses values directly (e.g., 1 instead of Size.Small)

// Alternative: use const object (recommended over enum by many)
const STATUS = {
  Pending: 'PENDING',
  Active: 'ACTIVE',
} as const;

type Status = typeof STATUS[keyof typeof STATUS];
// 'PENDING' | 'ACTIVE'
```

---

## 32. Functions in TypeScript

```typescript
// Parameter types + return type
function add(a: number, b: number): number {
  return a + b;
}

// Optional parameters
function greet(name: string, greeting?: string): string {
  return `${greeting ?? 'Hello'}, ${name}`;
}

// Default parameters
function createUser(name: string, role: string = 'user') {
  return { name, role };
}

// Rest parameters
function sum(...numbers: number[]): number {
  return numbers.reduce((a, b) => a + b, 0);
}

// Function type
type Transformer<T, R> = (input: T) => R;
const stringify: Transformer<number, string> = (n) => n.toString();

// Overloads — multiple signatures, one implementation
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value);
}

// void vs never
function logMessage(msg: string): void { console.log(msg); }
function throwError(msg: string): never { throw new Error(msg); }
```

---

## 33. Generics

Generics allow writing reusable code that works with multiple types.

### Basic Generics

```typescript
// Without generics: loses type info
function identity(arg: any): any { return arg; }

// With generics: type-safe
function identity<T>(arg: T): T { return arg; }

identity<string>('hello'); // return type: string
identity(42);              // inferred: identity<number>

// Generic array
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

first([1, 2, 3]);     // type: number | undefined
first(['a', 'b']);    // type: string | undefined
```

### Generic Constraints

```typescript
// Constraint: T must have a .length property
function logLength<T extends { length: number }>(arg: T): void {
  console.log(arg.length);
}

logLength('hello');    // ok
logLength([1,2,3]);   // ok
logLength(42);        // Error: number doesn't have 'length'

// Key constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'Arjun', age: 25 };
getProperty(user, 'name'); // type: string
getProperty(user, 'age');  // type: number
// getProperty(user, 'email'); // Error: 'email' not in User
```

### Generic Interfaces & Classes

```typescript
// Generic interface
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Partial<T>): Promise<T>;
  update(id: string, data: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}

// Generic class
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  isEmpty(): boolean { return this.items.length === 0; }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
numStack.pop(); // type: number | undefined
```

### Generic Default Types

```typescript
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

const response: ApiResponse = { data: null, status: 200, message: 'ok' };
const typed: ApiResponse<User> = { data: user, status: 200, message: 'ok' };
```

---

## 34. Utility Types

TypeScript has built-in utility types that transform existing types.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial<T> — all properties optional
type UpdateUser = Partial<User>;
// { id?: number; name?: string; email?: string; ... }

// Required<T> — all properties required
type RequiredUser = Required<Partial<User>>;

// Readonly<T> — all properties read-only
type ImmutableUser = Readonly<User>;

// Pick<T, K> — select specific properties
type UserProfile = Pick<User, 'id' | 'name' | 'email'>;
// { id: number; name: string; email: string }

// Omit<T, K> — exclude specific properties
type PublicUser = Omit<User, 'password'>;
// { id: number; name: string; email: string; createdAt: Date }

// Record<K, V> — object with specific key/value types
type UserMap = Record<string, User>;
type StatusMap = Record<'active' | 'inactive', number>;

// Exclude<T, U> — remove types from union
type NonString = Exclude<string | number | boolean, string>;
// number | boolean

// Extract<T, U> — keep types in union
type OnlyString = Extract<string | number | boolean, string | symbol>;
// string

// NonNullable<T> — remove null and undefined
type DefinedValue = NonNullable<string | null | undefined>;
// string

// ReturnType<T> — get return type of function
function createUser() { return { id: 1, name: 'Arjun' }; }
type UserReturn = ReturnType<typeof createUser>;
// { id: number; name: string }

// Parameters<T> — get parameter types of function
type CreateUserParams = Parameters<typeof createUser>;

// InstanceType<T> — get instance type of class
class UserClass { name = 'Arjun'; }
type UserInstance = InstanceType<typeof UserClass>;

// Awaited<T> — unwrap Promise type
type ResolvedUser = Awaited<Promise<User>>; // User
```

---

## 35. Type Narrowing & Type Guards

Narrowing = refining a type to something more specific.

```typescript
// typeof narrowing
function process(value: string | number) {
  if (typeof value === 'string') {
    return value.toUpperCase(); // TS knows: string
  }
  return value.toFixed(2); // TS knows: number
}

// instanceof narrowing
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.log(error.message); // TS knows: Error
  } else {
    console.log(String(error));
  }
}

// in operator narrowing
type Dog = { breed: string; bark(): void };
type Cat = { indoor: boolean; meow(): void };

function handlePet(pet: Dog | Cat) {
  if ('bark' in pet) {
    pet.bark(); // TS knows: Dog
  } else {
    pet.meow(); // TS knows: Cat
  }
}

// Equality narrowing
function compare(a: string | null, b: string | null) {
  if (a === b) {
    a.toUpperCase(); // TS knows: both are string (null === null → not string)
  }
}

// Custom type guard (is)
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

function process(data: unknown) {
  if (isUser(data)) {
    console.log(data.name); // TS knows: User
  }
}

// Assertion function
function assertUser(value: unknown): asserts value is User {
  if (!isUser(value)) throw new Error('Not a user');
}

// Discriminated union narrowing
type ApiResult =
  | { status: 'success'; data: User }
  | { status: 'error'; message: string };

function handle(result: ApiResult) {
  if (result.status === 'success') {
    console.log(result.data.name); // TS knows: success branch
  } else {
    console.log(result.message); // TS knows: error branch
  }
}
```

---

## 36. Classes in TypeScript

```typescript
class User {
  // Access modifiers
  public name: string;           // accessible anywhere (default)
  private password: string;      // only inside class
  protected email: string;       // inside class + subclasses
  readonly id: number;           // can't be changed after construction
  static count: number = 0;      // shared across all instances

  // Shorthand constructor (declares and initializes)
  constructor(
    public name: string,
    private password: string,
    protected email: string,
    readonly id: number = ++User.count,
  ) {}

  // Method
  greet(): string {
    return `Hi, I'm ${this.name}`;
  }

  // Getter
  get maskedPassword(): string {
    return '****';
  }

  // Static method
  static getCount(): number {
    return User.count;
  }
}

// Abstract class — can't be instantiated directly
abstract class Animal {
  constructor(public name: string) {}

  abstract makeSound(): string; // must be implemented by subclasses

  move(): string {
    return `${this.name} is moving`;
  }
}

class Dog extends Animal {
  makeSound(): string {
    return 'Woof!';
  }
}

// Implementing interfaces
interface Serializable {
  serialize(): string;
}

interface Loggable {
  log(): void;
}

class Product implements Serializable, Loggable {
  constructor(public name: string, public price: number) {}

  serialize(): string {
    return JSON.stringify({ name: this.name, price: this.price });
  }

  log(): void {
    console.log(this.serialize());
  }
}
```

---

## 37. Decorators

Decorators are special functions that can modify classes, methods, properties, or parameters. Heavily used in NestJS.

```typescript
// Enable in tsconfig: "experimentalDecorators": true

// Class decorator
function Entity(tableName: string) {
  return function(target: Function) {
    target.prototype.tableName = tableName;
  };
}

@Entity('users')
class User {
  name = 'Arjun';
}

// Method decorator
function Log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${propertyKey} with`, args);
    const result = originalMethod.apply(this, args);
    console.log(`Result:`, result);
    return result;
  };
  return descriptor;
}

class Calculator {
  @Log
  add(a: number, b: number): number {
    return a + b;
  }
}

// Property decorator
function Required(target: any, propertyKey: string) {
  // attach metadata
}

// Parameter decorator
function Inject(token: string) {
  return function(target: any, propertyKey: string | undefined, index: number) {
    // used by DI containers like NestJS
  };
}

// NestJS-style decorators
@Controller('users')
class UserController {
  constructor(private userService: UserService) {}

  @Get(':id')
  @UseGuards(AuthGuard)
  getUser(@Param('id') id: string) {
    return this.userService.findById(id);
  }
}
```

---

## 38. Modules & Namespaces

### Modules (Preferred — same as ESM)

```typescript
// user.model.ts
export interface User { id: number; name: string; }
export type UserId = number | string;
export class UserService { ... }
export default class App { ... }

// Barrel file (index.ts)
export { User, UserService } from './user.model';
export * from './auth.model';

// Importing
import { User, UserService } from './user.model';
import type { User } from './user.model'; // type-only import (erased at compile time)
```

### Namespaces (legacy, avoid in modern TS)

```typescript
namespace Validation {
  export interface Validator {
    validate(value: string): boolean;
  }

  export class EmailValidator implements Validator {
    validate(email: string): boolean {
      return /\S+@\S+\.\S+/.test(email);
    }
  }
}

const validator = new Validation.EmailValidator();
```

---

## 39. Declaration Files (.d.ts)

Declaration files describe the shape of existing JS libraries for TypeScript.

```typescript
// math.d.ts — describing a JS library
declare module 'math-lib' {
  export function add(a: number, b: number): number;
  export function subtract(a: number, b: number): number;
  export const PI: number;
}

// Declaring global variables
declare const __DEV__: boolean;
declare function require(module: string): any;

// Augmenting existing types
declare global {
  interface Window {
    analytics: { track: (event: string) => void };
  }

  interface Array<T> {
    last(): T | undefined;
  }
}

// @types packages — most popular JS libraries have these
// npm install -D @types/express @types/lodash
```

---

## 40. tsconfig.json Deep Dive

```json
{
  "compilerOptions": {
    // Target JS version
    "target": "ES2020",

    // Module system
    "module": "commonjs",      // for Node.js
    "module": "ESNext",        // for modern bundlers

    // Enable ESM-style imports
    "moduleResolution": "node",

    // Strict mode (highly recommended — enables many checks)
    "strict": true,
    // strict = shorthand for all of these:
    "strictNullChecks": true,         // null/undefined are not assignable to other types
    "strictFunctionTypes": true,       // stricter function type checking
    "strictBindCallApply": true,       // strict bind/call/apply
    "noImplicitAny": true,            // error on implicit 'any'
    "noImplicitThis": true,           // error on 'this: any'
    "alwaysStrict": true,             // emit 'use strict'

    // Additional checks
    "noUnusedLocals": true,           // error on unused variables
    "noUnusedParameters": true,       // error on unused params
    "noImplicitReturns": true,        // all code paths must return
    "noFallthroughCasesInSwitch": true, // no switch fallthrough

    // Output
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,              // generate .d.ts files
    "sourceMap": true,                // generate source maps

    // Decorators (NestJS requires these)
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,

    // Path aliases
    "baseUrl": ".",
    "paths": {
      "@modules/*": ["src/modules/*"],
      "@common/*": ["src/common/*"]
    },

    // Library typings included
    "lib": ["ES2020", "DOM"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 41. Advanced Types

### Conditional Types

```typescript
// T extends U ? X : Y
type IsString<T> = T extends string ? true : false;
IsString<string>; // true
IsString<number>; // false

// infer — extract a type within a conditional
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;
UnpackPromise<Promise<string>>; // string
UnpackPromise<number>;          // number

type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

// Distributive conditional types
type ToArray<T> = T extends any ? T[] : never;
ToArray<string | number>; // string[] | number[]
```

### Mapped Types

```typescript
// Transform all properties
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type Optional<T> = {
  [K in keyof T]?: T[K];
};

type Mutable<T> = {
  -readonly [K in keyof T]: T[K]; // remove readonly
};

type Required<T> = {
  [K in keyof T]-?: T[K]; // remove optional
};

// Remap keys
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number }
```

### Template Literal in Mapped Types

```typescript
type EventMap<T extends string> = {
  [K in T as `on${Capitalize<K>}`]: () => void;
};

type ButtonEvents = EventMap<'click' | 'hover' | 'focus'>;
// { onClick: () => void; onHover: () => void; onFocus: () => void }
```

### Recursive Types

```typescript
// JSON type
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

// Deep partial
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Tree node
type TreeNode<T> = {
  value: T;
  children?: TreeNode<T>[];
};
```

---

## 42. TypeScript with Node.js / NestJS Patterns

### Common Patterns in NestJS

```typescript
// DTOs with class-validator
import { IsEmail, IsString, MinLength, IsOptional } from 'class-validator';

export class CreateUserDto {
  @IsString()
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsOptional()
  @IsString()
  role?: string;
}

// Generic Repository interface
interface IRepository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id' | 'createdAt'>): Promise<T>;
  update(id: ID, data: Partial<T>): Promise<T | null>;
  delete(id: ID): Promise<boolean>;
}

// Service with typed return
@Injectable()
export class UserService {
  async findUser(id: string): Promise<User | null> {
    return this.repository.findById(id);
  }

  async createUser(dto: CreateUserDto): Promise<Omit<User, 'password'>> {
    const user = await this.repository.create(dto);
    const { password, ...safeUser } = user;
    return safeUser;
  }
}

// Typed config
interface AppConfig {
  database: {
    host: string;
    port: number;
    name: string;
  };
  jwt: {
    secret: string;
    expiresIn: string;
  };
  redis: {
    url: string;
  };
}

// Type-safe environment
function getConfig(): AppConfig {
  return {
    database: {
      host: process.env.DB_HOST!,
      port: Number(process.env.DB_PORT),
      name: process.env.DB_NAME!,
    },
    jwt: {
      secret: process.env.JWT_SECRET!,
      expiresIn: process.env.JWT_EXPIRES_IN ?? '15m',
    },
    redis: {
      url: process.env.REDIS_URL!,
    },
  };
}

// Discriminated union for API responses
type ApiResponse<T> =
  | { success: true; data: T; statusCode: number }
  | { success: false; error: string; statusCode: number };

function handleResponse<T>(response: ApiResponse<T>) {
  if (response.success) {
    return response.data; // TS knows: T
  }
  throw new Error(response.error); // TS knows: error branch
}
```

---

# PART 3 – 100 INTERVIEW QUESTIONS

*No solutions — use this for self-assessment and revision practice.*

---

## JavaScript – Fundamentals (Q1–Q20)

1. What is the difference between `var`, `let`, and `const`? When would you use each?
2. Explain the Temporal Dead Zone (TDZ) with an example.
3. What is hoisting? What gets hoisted and what does not?
4. What is the difference between `null` and `undefined`?
5. What does `typeof null` return and why?
6. Explain type coercion in JavaScript. Give 3 unexpected examples.
7. What is the difference between `==` and `===`? When is `==` acceptable to use?
8. List all falsy values in JavaScript.
9. What is the difference between `||` and `??`?
10. What is optional chaining (`?.`) and how does it differ from `&&` chaining?
11. Explain the difference between a primitive type and a reference type.
12. What happens when you assign an object to another variable in JavaScript?
13. How does `const` work with objects and arrays?
14. What is the difference between `function declaration` and `function expression`?
15. Can you call a function before it's declared? Explain why or why not.
16. What is an IIFE and what is it used for?
17. What is a pure function? Give an example of a pure and impure function.
18. What is the difference between `arguments` and rest parameters?
19. What is currying? Write a curried `add(1)(2)(3)` function.
20. What does the `in` operator do? How is it different from `hasOwnProperty`?

---

## JavaScript – Core Concepts (Q21–Q50)

21. Explain the scope chain with a nested function example.
22. What is a closure? Write a function that uses closure to create a counter.
23. What is the difference between `.call()`, `.apply()`, and `.bind()`?
24. How does `this` behave in regular functions vs arrow functions?
25. How do you fix the "lost `this`" problem in callbacks?
26. What is the prototype chain? How does property lookup work?
27. How does prototypal inheritance differ from classical inheritance?
28. What does `Object.create(null)` give you and when would you use it?
29. What is the difference between `Object.freeze()` and `Object.seal()`?
30. How do you deep clone an object in JavaScript? What are the limitations of `JSON.parse(JSON.stringify())`?
31. What is the difference between `for...in` and `for...of`?
32. Explain the Map vs Object trade-offs. When would you choose Map?
33. What is a WeakMap? How is it different from a Map?
34. What is a Symbol? Why would you use it as an object key?
35. What is the Event Loop? Describe the difference between the Call Stack, Microtask Queue, and Macrotask Queue.
36. What is the order of execution between `setTimeout(fn, 0)`, a Promise `.then()`, and synchronous code?
37. What are microtasks vs macrotasks? Give examples of each.
38. What is a generator function? How does `yield` work?
39. How do you make a custom iterable? What is `Symbol.iterator`?
40. What is the difference between `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`?
41. What happens if you forget `await` before an async function call?
42. Why doesn't `await` inside `forEach` work as expected? How do you fix it?
43. What is the difference between sequential and parallel async execution? Write both versions.
44. How do you handle errors in async/await code?
45. What is the Proxy object used for? Give a practical example.
46. Explain the Observer/EventEmitter pattern. Implement a basic version.
47. What is a memory leak? Give 3 common causes and how to avoid them.
48. What is the difference between CommonJS and ES Modules?
49. What is tree-shaking and which module system supports it?
50. What is the difference between `structuredClone()` and spread clone?

---

## JavaScript – Advanced (Q51–Q60)

51. What is the difference between `Object.keys()`, `Object.values()`, `Object.entries()`, and `Object.fromEntries()`?
52. How does `Array.prototype.reduce` work? Rewrite `map` and `filter` using `reduce`.
53. What is `flatMap`? How is it different from `map` followed by `flat`?
54. How would you implement memoization? Write a generic `memoize` function.
55. What is the Singleton pattern? Implement it using a class with a private static field.
56. Explain the difference between composition and inheritance. Why do many prefer composition?
57. What is debounce vs throttle? Implement both.
58. What is the difference between `setTimeout(fn, 0)` and `setImmediate(fn)` in Node.js?
59. What is `process.nextTick` in Node.js and how does it compare to Promise microtasks?
60. How would you implement a simple pub/sub system in JavaScript?

---

## TypeScript – Fundamentals (Q61–Q80)

61. What is the difference between `any` and `unknown`? When should you use each?
62. What is the difference between `void` and `never` return types?
63. What is a Tuple type? How is it different from an array type?
64. What is the difference between an `interface` and a `type alias`? When would you prefer each?
65. What is declaration merging in TypeScript? When is it useful?
66. What is `readonly` and how is it different from `const`?
67. What is a union type? What is a discriminated union and why is it useful?
68. What is an intersection type (`&`)? Write an example combining two interfaces.
69. What is `keyof`? Write a function using `keyof` to safely access object properties.
70. What is `typeof` in TypeScript type position? How is it different from JS `typeof`?
71. What is type inference? When should you add explicit type annotations?
72. What are generic constraints (`T extends SomeType`)? Write an example.
73. Write a generic `Stack<T>` class with `push`, `pop`, and `peek` methods.
74. What is the difference between `Partial<T>`, `Required<T>`, `Readonly<T>`, and `Pick<T, K>`?
75. What does `Omit<T, K>` do? When would you use it? Give a practical example.
76. What is `ReturnType<T>` and how would you use it?
77. What is `NonNullable<T>`? How is it useful with API responses?
78. What are conditional types in TypeScript? Write a type that unwraps a Promise.
79. What is a mapped type? Write a `Nullable<T>` mapped type.
80. What is a template literal type? Write a type that creates getter names from property names.

---

## TypeScript – Advanced (Q81–Q100)

81. What are TypeScript decorators? What are the four types of decorators?
82. What do `experimentalDecorators` and `emitDecoratorMetadata` do in `tsconfig`?
83. What is `strictNullChecks`? What changes in your code when you enable it?
84. What is the `!` non-null assertion operator? When should (and shouldn't) you use it?
85. What is a type guard? Write a custom type guard function using `is`.
86. What is an assertion function (`asserts value is T`)? How is it different from a type guard?
87. What is the difference between structural typing and nominal typing? Which does TypeScript use?
88. What is `infer` in TypeScript? Write a type that extracts the first argument type of a function.
89. What are recursive types? Write a `DeepPartial<T>` type.
90. What is `as const`? How does it affect type inference?
91. How do you type an object where you know the value type but not all keys (`Record<string, T>`)?
92. What is the `satisfies` operator in TypeScript? How is it different from type assertion (`as`)?
93. How do you extend a third-party library's types using declaration merging?
94. What is a `.d.ts` file? When would you create one manually?
95. What is the difference between `import type` and regular `import`?
96. What is `namespace` in TypeScript? Is it recommended in modern TypeScript? Why or why not?
97. How do you type `this` in a function in TypeScript?
98. How do you make a class property truly private in TypeScript vs using `private` keyword?
99. What is the `Awaited<T>` utility type and when do you need it?
100. You have a function that accepts `unknown`. Write the full chain of type narrowing to safely extract a `User` object with `id: number` and `name: string`, including a custom type guard and error handling.

---

*Best of luck! Revise all sections, then attempt the questions without referring back.*
