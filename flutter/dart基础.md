# 变量

下面是声明变量并赋值的示例：

```dart
var name = 'Bob';
```

变量是一个引用。上面名字为 `name` 的变量引用了 一个内容为 “Bob” 的 String 对象。

### 默认值

没有初始化的变量自动获取一个默认值为 `null`。类型为数字的 变量如何没有初始化其值也是 null，不要忘记了 数字类型也是对象。

### 可选的类型

在声明变量的时候，你可以选择加上具体 类型：

```dart
String name = 'Bob';
```

添加类型可以更加清晰的表达你的意图。 IDE 编译器等工具有可以使用类型来更好的帮助你， 可以提供代码补全、提前发现 bug 等功能。

### Final and const

如果你以后不打算修改一个变量，使用 `final` 或者 `const`。 一个 final 变量只能赋值一次；一个 const 变量是编译时常量。 （Const 变量同时也是 final 变量。） 顶级的 final 变量或者类中的 final 变量在 第一次使用的时候初始化。

注意： 实例变量可以为 final 但是不能是 const 。

```dart
final name = 'Bob'; // Or: final String name = 'Bob';
// name = 'Alice';  // Uncommenting this causes an error
```

`const` 变量为编译时常量。 如果 const 变量在类中，请定义为 `static const`。 可以直接定义 const 和其值，也 可以定义一个 const 变量使用其他 const 变量的值来初始化其值。

```dart
const bar = 1000000;       // Unit of pressure (dynes/cm2)
const atm = 1.01325 * bar; // Standard atmosphere
```

`const` 关键字不仅仅只用来定义常量。 有可以用来创建不变的值， 还能定义构造函数为 const 类型的，这种类型 的构造函数创建的对象是不可改变的。任何变量都可以有一个不变的值

```dart
/ Note: [] creates an empty list.
// const [] creates an empty, immutable list (EIA).
var foo = const [];   // foo is currently an EIA.
final bar = const []; // bar will always be an EIA.
const baz = const []; // baz is a compile-time constant EIA.

// You can change the value of a non-final, non-const variable,
// even if it used to have a const value.
foo = [];

// You can't change the value of a final or const variable.
// bar = []; // Unhandled exception.
// baz = []; // Unhandled exception.
```

# 内置的类型

Dart 内置支持下面这些类型：

- numbers
- strings
- booleans
- lists (也被称之为 arrays)
- maps
- runes (用于在字符串中表示 Unicode 字符)
- symbols

你可以直接使用字母量来初始化上面的这些类型。 例如 'this is a string' 是一个字符串字面量， true 是一个布尔字面量。

由于 Dart 中每个变量引用的都是一个对象 – 一个类的实例， 你通常使用构造函数来初始化变量。 一些内置的类型具有自己的构造函数。例如， 可以使用 Map()构造函数来创建一个 map， 就像这样 new Map()。

### Numbers

Dart 支持两种类型的数字：

- int

  整数值，其取值通常位于 -2 53次方 和 2 53次方 之间。

- double

  64-bit (双精度) 浮点数

int 和 double 都是 num 的子类。 num 类型定义了基本的操作符，例如 +, -, /, 和 *， 还定义了 abs()、 ceil()、和 floor() 等 函数。 (位操作符，例如 >> 定义在 int 类中。)