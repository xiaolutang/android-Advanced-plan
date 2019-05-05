# 类

Dart是一种面向对象的语言，具有类和基于mixin的继承。 每个对象都是一个类的实例，所有类都来自Object。 基于Mixin的继承意味着虽然每个类（除了Object）只有一个超类，但是一个类体可以在多个类层次结构中重用。dart是单继承。

## 类的使用

### 使用class的成员

对象具有由函数和数据（分别为方法和实例变量）组成的成员。 调用方法时，可以在对象上调用它：该方法可以访问该对象的函数和数据。

使用点（.）符号来引用实例变量或方法。比如：

```dart
var p = Point(2, 2);

// Set the value of the instance variable y.
p.y = 3;

// Get the value of y.
assert(p.y == 3);

// Invoke distanceTo() on p.
num distance = p.distanceTo(Point(4, 4));
```

可以使用 ?. 来代替 .  来避免当左边操作数为null时的异常

```dart
// If p is non-null, set its y value to 4.
p?.y = 4;
```

### 使用构造函数

您可以使用构造函数创建对象。 构造函数名称可以是ClassName或ClassName.identifier。 例如，以下代码使用Point（）和Point.fromJson（）构造函数创建Point对象。

```dart
var p1 = Point(2, 2);
var p2 = Point.fromJson({'x': 1, 'y': 2});
```

以下代码具有相同的效果，但在构造函数名称之前使用可选的new关键字：

```dart
var p1 = new Point(2, 2);
var p2 = new Point.fromJson({'x': 1, 'y': 2});
```

​	注意：new 关键字在dart2是可选的

有些类提供常量构造函数。 要使用常量构造函数创建编译时常量，请将const关键字放在构造函数名称之前

```dart
var p = const ImmutablePoint(2, 2);
```

构造两个相同的编译时常量会产生一个规范的实例（这两个实例是同一个对象）：

```dart
var a = const ImmutablePoint(1, 1);
var b = const ImmutablePoint(1, 1);

assert(identical(a, b)); // They are the same instance!
```

在常量上下文中，可以在构造函数或文字之前省略const。 例如，查看此代码，该代码创建一个const映射

```dart
// Lots of const keywords here.
const pointAndLine = const {
  'point': const [const ImmutablePoint(0, 0)],
  'line': const [const ImmutablePoint(1, 10), const ImmutablePoint(-2, 11)],
};
```

您可以省略除const关键字的第一次使用之外的所有内容：

```dart
// Only one const, which establishes the constant context.
const pointAndLine = {
  'point': [ImmutablePoint(0, 0)],
  'line': [ImmutablePoint(1, 10), ImmutablePoint(-2, 11)],
};
```

如果常量构造函数在常量上下文之外并且在没有const的情况下调用，则会创建一个非常量对象：

```dart
var a = const ImmutablePoint(1, 1); // Creates a constant
var b = ImmutablePoint(1, 1); // Does NOT create a constant

assert(!identical(a, b)); // NOT the same instance!
```

### 获取对象的类型

要在运行时获取对象的类型，可以使用Object的runtimeType属性，该属性返回Type对象。

```dart
print('The type of a is ${a.runtimeType}');
```

## 类的实现

### 类的定义

以下是声明实例变量的示例：

```dart
class Point {
  num x; // Declare instance variable x, initially null.
  num y; // Declare y, initially null.
  num z = 0; // Declare z, initially 0.
}
```

所有未初始化的实例变量的值都为null。

所有实例变量都生成一个隐式getter方法。 非最 final 例变量也会生成隐式setter方法。

```dart
class Point {
  num x;
  num y;
}

void main() {
  var point = Point();
  point.x = 4; // Use the setter method for x.
  assert(point.x == 4); // Use the getter method for x.
  assert(point.y == null); // Values default to null.
}
```

如果初始化声明它的实例变量（而不是构造函数或方法），则在创建实例时设置该值，该实例在构造函数及其初始化列表执行之前执行

### 构造函数

通过创建与其类同名的函数来声明构造函数（另外，可选地，如命名构造函数中所述的附加标识符）。 最常见的构造函数形式，即生成构造函数，创建一个类的新实例：

```dart
class Point {
  num x, y;

  Point(num x, num y) {
    // There's a better way to do this, stay tuned.
    this.x = x;
    this.y = y;
  }
}
```

this关键字引用当前实例。

​	注意：仅在存在名称冲突时使用此选项。 否则，Dart风格省略了这一点

将构造函数参数赋值给实例变量的模式是如此常见，Dart具有语法糖，使其变得简单：

```dart
class Point {
  num x, y;

  // Syntactic sugar for setting x and y
  // before the constructor body runs.
  Point(this.x, this.y);
}
```

#### 默认构造函数

如果您未声明构造函数，则会为您提供默认构造函数。 默认构造函数没有参数，并在超类中调用无参数构造函数。

#### 构造函数不是继承的

子类不从其超类继承构造函数。 声明没有构造函数的子类只有默认（无参数，无名称）构造函数（没有理解到这个是干什么的，是不是意味着dart不像java会强制调用父类的构造方法。在dart中不会调用父类的构造方法，如果要调用需要自己实现？）

#### 命名构造函数

使用命名构造函数为类实现多个构造函数或提供额外的清晰度：

```dart
class Point {
  num x, y;

  Point(this.x, this.y);

  // Named constructor
  Point.origin() {
    x = 0;
    y = 0;
  }
}
```

请记住，构造函数不是继承的，这意味着超类的命名构造函数不会被子类继承。 如果希望使用超类中定义的命名构造函数创建子类，则必须在子类中实现该构造函数。

#### 调用非默认的超类构造函数

默认情况下，子类中的构造函数调用超类的未命名的无参数构造函数。 超类的构造函数在构造函数体的开头被调用。 如果还使用初始化列表，则在调用超类之前执行。 总之，执行顺序如下：

初始化列表
超类的无参数构造函数
主类的无参数构造函数
如果超类没有未命名的无参数构造函数，则必须手动调用超类中的一个构造函数。 在冒号（:)之后指定超类构造函数，就在构造函数体（如果有）之前。

在下面的示例中，Employee类的构造函数为其超类Person调用命名构造函数。 

```dart
class Person {
  String firstName;

  Person.fromJson(Map data) {
    print('in Person');
  }
}

class Employee extends Person {
  // Person does not have a default constructor;
  // you must call super.fromJson(data).
  Employee.fromJson(Map data) : super.fromJson(data) {
    print('in Employee');
  }
}

main() {
  var emp = new Employee.fromJson({});

  // Prints:
  // in Person
  // in Employee
  if (emp is Person) {
    // Type check
    emp.firstName = 'Bob';
  }
  (emp as Person).firstName = 'Bob';
}
```

因为在调用构造函数之前会计算超类构造函数的参数，所以参数可以是一个表达式，例如函数调用：

```dart
class Employee extends Person {
  Employee() : super.fromJson(getDefaultData());
  // ···
}
```

​	警告：超类构造函数的参数无权访问它。 例如，参数可以调用静态方法，但不能调用实例方法。

#### 初始化列表

除了调用超类构造函数之外，还可以在构造函数体运行之前初始化实例变量。 用逗号分隔初始化程序

```dart
// Initializer list sets instance variables before
// the constructor body runs.
Point.fromJson(Map<String, num> json)
    : x = json['x'],
      y = json['y'] {
  print('In Point.fromJson(): ($x, $y)');
}
```

在开发期间，您可以使用初始化列表中的assert来验证输入。

```dart
Point.withAssert(this.x, this.y) : assert(x >= 0) {
  print('In Point.withAssert(): ($x, $y)');
}
```

设置最终字段时，初始化程序列表很方便。 以下示例初始化初始化列表中的三个最终字段。

```dart
import 'dart:math';

class Point {
  final num x;
  final num y;
  final num distanceFromOrigin;

  Point(x, y)
      : x = x,
        y = y,
        distanceFromOrigin = sqrt(x * x + y * y);
}

main() {
  var p = new Point(2, 3);
  print(p.distanceFromOrigin);
}
```

#### 重定向构造函数

有时构造函数的唯一目的是重定向到同一个类中的另一个构造函数。 重定向构造函数的主体是空的，构造函数调用出现在冒号（:)之后。

```dart
class Point {
  num x, y;

  // The main constructor for this class.
  Point(this.x, this.y);

  // Delegates to the main constructor.
  Point.alongXAxis(num x) : this(x, 0);
}
```

#### 常量构造函数

如果您的类生成永远不会更改的对象，则可以使这些对象成为编译时常量。 为此，请定义const构造函数并确保所有实例变量都是final。

```dart
class ImmutablePoint {
  static final ImmutablePoint origin =
      const ImmutablePoint(0, 0);

  final num x, y;

  const ImmutablePoint(this.x, this.y);
}
```

#### 工厂构造函数

在实现并不总是创建其类的新实例的构造函数时，请使用factory关键字。 例如，工厂构造函数可能从缓存中返回实例，或者它可能返回子类型的实例。

以下示例演示了从缓存中返回对象的工厂构造函数

```dart
class Logger {
  final String name;
  bool mute = false;

  // _cache is library-private, thanks to
  // the _ in front of its name.
  static final Map<String, Logger> _cache =
      <String, Logger>{};

  factory Logger(String name) {
    if (_cache.containsKey(name)) {
      return _cache[name];
    } else {
      final logger = Logger._internal(name);
      _cache[name] = logger;
      return logger;
    }
  }

  Logger._internal(this.name);

  void log(String msg) {
    if (!mute) print(msg);
  }
}
```

像调用任何其他构造函数一样调用工厂构造函数

```dart
var logger = Logger('UI');
logger.log('Button clicked');
```

## 方法

方法是为对象提供行为的函数。

### 实例方法

对象的实例方法可以访问实例变量和它。 以下示例中的distanceTo（）方法是实例方法的示例：

```dart
import 'dart:math';

class Point {
  num x, y;

  Point(this.x, this.y);

  num distanceTo(Point other) {
    var dx = x - other.x;
    var dy = y - other.y;
    return sqrt(dx * dx + dy * dy);
  }
}
```

### Getters and setters

getter和setter是提供对象属性的读写访问权限的特殊方法。 回想一下，每个实例变量都有一个隐式getter，如果合适的话还有一个setter。 您可以使用get和set关键字通过实现getter和setter来创建其他属性

```dart
class Rectangle {
  num left, top, width, height;

  Rectangle(this.left, this.top, this.width, this.height);

  // Define two calculated properties: right and bottom.
  num get right => left + width;
  set right(num value) => left = value - width;
  num get bottom => top + height;
  set bottom(num value) => top = value - height;
}

void main() {
  var rect = Rectangle(3, 4, 20, 15);
  assert(rect.left == 3);
  rect.right = 12;
  assert(rect.left == -8);
}
```

使用getter和setter，您可以从实例变量开始，稍后使用方法包装它们，而无需更改客户端代码。

​	注意：无论是否明确定义了getter，增量（++）等运算符都以预期的方式工作。 为避免任何意外的副作用，操作员只需调用一次getter，将其值保存在临时变量中

### 抽象方法

实例，getter和setter方法可以是抽象的，定义一个接口，但将其实现留给其他类。 抽象方法只能存在于抽象类中。

要使方法成为抽象，请使用分号（;）而不是方法体：

```dart
abstract class Doer {
  // Define instance variables and methods...

  void doSomething(); // Define an abstract method.
}

class EffectiveDoer extends Doer {
  void doSomething() {
    // Provide an implementation, so the method is not abstract here...
  }
}
```

## 抽象类

使用abstract修饰符定义抽象类 - 无法实例化的类。 抽象类对于定义接口非常有用，通常还有一些实现。 如果希望抽象类看起来是可实例化的，请定义工厂构造函数。

抽象类通常有抽象方法。 这是一个声明具有抽象方法的抽象类的示例：

```dart
// This class is declared abstract and thus
// can't be instantiated.
abstract class AbstractContainer {
  // Define constructors, fields, methods...

  void updateChildren(); // Abstract method.
}
```

## 隐式接口

每个类都隐式定义一个接口，该接口包含该类的所有实例成员及其实现的任何接口。 如果要在不继承B实现的情况下创建支持B类API的A类，则A类应实现B接口。

类通过在implements子句中声明它们然后提供接口所需的API来实现一个或多个接口。 例如：

```dart
// A person. The implicit interface contains greet().
class Person {
  // In the interface, but visible only in this library.
  final _name;

  // Not in the interface, since this is a constructor.
  Person(this._name);

  // In the interface.
  String greet(String who) => 'Hello, $who. I am $_name.';
}

// An implementation of the Person interface.
class Impostor implements Person {
  get _name => '';

  String greet(String who) => 'Hi $who. Do you know who I am?';
}

String greetBob(Person person) => person.greet('Bob');

void main() {
  print(greetBob(Person('Kathy')));
  print(greetBob(Impostor()));
}
```

## 继承

使用extends创建子类，使用super来引用超类

```dart
class Television {
  void turnOn() {
    _illuminateDisplay();
    _activateIrSensor();
  }
  // ···
}

class SmartTelevision extends Television {
  void turnOn() {
    super.turnOn();
    _bootNetworkInterface();
    _initializeMemory();
    _upgradeApps();
  }
  // ···
}
```

### 重写方法

子类可以覆盖实例方法，getter和setter。 您可以使用@override注释来指示您有意覆盖成员

```dart
class SmartTelevision extends Television {
  @override
  void turnOn() {...}
  // ···
}
```

要在类型安全的代码中缩小方法参数或实例变量的类型，可以使用covariant关键字。

```dart
class Animal {
  void chase(Animal x) { ... }
}

class Mouse extends Animal { ... }

class Cat extends Animal {
  void chase(covariant Mouse x) { ... }
}
```

### 重写操作符

您可以覆盖下表中显示的运算符。 例如，如果定义Vector类，则可以定义一个+方法来添加两个向量

​	注意：您可能已经注意到！=不是可覆盖的运算符。 表达式e1！= e2只是!（e1 == e2）的语法糖。

```dart
class Vector {
  final int x, y;

  Vector(this.x, this.y);

  Vector operator +(Vector v) => Vector(x + v.x, y + v.y);
  Vector operator -(Vector v) => Vector(x - v.x, y - v.y);

  // Operator == and hashCode not shown. For details, see note below.
  // ···
}

void main() {
  final v = Vector(2, 3);
  final w = Vector(2, 2);

  assert(v + w == Vector(4, 5));
  assert(v - w == Vector(0, 1));
}
```

如果覆盖==，则还应该覆盖Object的hashCode getter。

### noSuchMethod()

要在代码尝试使用不存在的方法或实例变量时检测或做出反应，您可以覆盖noSuchMethod（）：

```dart
class A {
  // Unless you override noSuchMethod, using a
  // non-existent member results in a NoSuchMethodError.
  @override
  void noSuchMethod(Invocation invocation) {
    print('You tried to use a non-existent member: ' +
        '${invocation.memberName}');
  }
}
```

除非满足以下条件之一，否则不能调用未实现的方法：

接收器具有静态类型动态。

接收器有一个定义未实现方法的静态类型（抽象是OK），接收器的动态类型有一个noSuchMethod（）实现，它与Object类中的实现不同。(不是太懂)