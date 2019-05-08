# 函数、方法（ Functions）

Dart是一种真正的面向对象语言，因此即使是函数也是对象并且具有类型Function。 这意味着函数可以分配给变量或作为参数传递给其他函数。 您也可以调用Dart类的实例，就好像它是一个函数一样。

这是一个实现函数的例子：

```dart
bool isNoble(int atomicNumber) {
  return _nobleGases[atomicNumber] != null;
}
```

虽然Effective Dart建议为公共API键入注释，但如果省略类型，该函数仍然有效：

```dart
isNoble(atomicNumber) {
  return _nobleGases[atomicNumber] != null;
}
```

对于只包含一个表达式的函数，可以使用简写语法：

```dart
bool isNoble(int atomicNumber) => _nobleGases[atomicNumber] != null;
```

=> expr语法是{return expr;的简写;}。 =>表示法有时称为箭头语法。

**注意：**箭头（=>）和分号（;）之间只能出现表达式而非语句。 例如，您不能在其中放置if语句，但可以使用条件表达式。

函数可以有两种类型的参数：必需和可选。 首先列出所需参数，然后列出任何可选参数。 命名的可选参数也可以标记为@required。 

## 可选参数

可选参数可以是位置参数，也可以是命名参数，但不能同时存在

### 可选的命名参数

调用函数时，可以使用paramName：value指定命名参数。 例如：

```dart
enableFlags(bold: true, hidden: false);
```

定义函数时，使用{param1，param2，...}指定命名参数

```dart
/// Sets the [bold] and [hidden] flags ...
void enableFlags({bool bold, bool hidden}) {...}
```

Flutter实例创建表达式可能变得复杂，因此窗口小部件构造函数仅使用命名参数。 这使得实例创建表达式更易于阅读。

您可以使用@required在任何Dart代码（不仅仅是Flutter）中注释命名参数，以指示它是必需参数。 例如：

```dart
const Scrollbar({Key key, @required Widget child})
```

构建Scrollbar时，分析器会在子参数不存在时报告问题。

required在meta中定义。 可以直接导入包：meta / meta.dart，也可以导入另一个导出meta的包，例如Flutter的包：flutter / material.dart。

### 可选的位置参数

在[]中包装一组函数参数将它们标记为可选的位置参数：

```dart
String say(String from, String msg, [String device]) {
  var result = '$from says $msg';
  if (device != null) {
    result = '$result with a $device';
  }
  return result;
}
```

这是一个在没有可选参数的情况下调用此函数的示例：

```dart
assert(say('Bob', 'Howdy') == 'Bob says Howdy');
```

以下是使用第三个参数（可选位置参数）调用此函数的示例：

```dart
assert(say('Bob', 'Howdy', 'smoke signal') ==
    'Bob says Howdy with a smoke signal');
```

### 默认参数值

您的函数可以使用=来定义命名和位置参数的默认值。 默认值必须是编译时常量。 如果未提供默认值，则默认值为null。

以下是为命名参数设置默认值的示例：

```dart
/// Sets the [bold] and [hidden] flags ...
void enableFlags({bool bold = false, bool hidden = false}) {...}

// bold will be true; hidden will be false.
enableFlags(bold: true);
```

下一个示例显示如何设置位置参数的默认值：

```dart
String say(String from, String msg,
    [String device = 'carrier pigeon', String mood]) {
  var result = '$from says $msg';
  if (device != null) {
    result = '$result with a $device';
  }
  if (mood != null) {
    result = '$result (in a $mood mood)';
  }
  return result;
}

assert(say('Bob', 'Howdy') ==
    'Bob says Howdy with a carrier pigeon');
```

您还可以将lists或map作为默认值传递。 以下示例定义了一个函数doStuff（），该函数指定list参数的默认列表和gifts参数的默认映射。

```dart
void doStuff(
    {List<int> list = const [1, 2, 3],
    Map<String, String> gifts = const {
      'first': 'paper',
      'second': 'cotton',
      'third': 'leather'
    }}) {
  print('list:  $list');
  print('gifts: $gifts');
}
```

## main（）函数

每个应用程序都必须具有顶级main（）函数，该函数用作应用程序的入口点。 main（）函数返回void，并为参数提供可选的List <String>参数。

以下是Web应用程序的main（）函数示例：

```dart
void main() {
  querySelector('#sample_text_id')
    ..text = 'Click me!'
    ..onClick.listen(reverseText);
}
```

**注意：**前面代码中的..语法称为级联。 使用级联，您可以对单个对象的成员执行多个操作

以下是带参数的命令行应用程序的main（）函数示例：

```dart
// Run the app like this: dart args.dart 1 test
void main(List<String> arguments) {
  print(arguments);

  assert(arguments.length == 2);
  assert(int.parse(arguments[0]) == 1);
  assert(arguments[1] == 'test');
}
```

## 一等方法对象(Functions as first-class objects)

可以把方法当做参数调用另外一个方法。例如：

```dart
//多个参数可以这样玩吗？
void printElement(int element) {
  print(element);
}

var list = [1, 2, 3];

// Pass printElement as a parameter.
list.forEach(printElement);
```

您还可以为变量分配函数，例如：

```dart
var loudify = (msg) => '!!! ${msg.toUpperCase()} !!!';
assert(loudify('hello') == '!!! HELLO !!!');
```

此示例使用匿名函数。

## 匿名函数

大多数函数都被命名，例如main（）或printElement（）。 您还可以创建一个名为匿名函数的无名函数，有时也可以创建一个lambda或闭包。 您可以为变量分配匿名函数，以便例如可以在集合中添加或删除它。

匿名函数看起来类似于命名函数 - 零个或多个参数，在逗号和括号之间用逗号和可选类型注释分隔。

后面的代码块包含函数的主体：

([[*Type*] *param1*[, …]]) { 
  *codeBlock*; 
}; 

以下示例使用无类型参数item定义匿名函数。 为列表中的每个项调用的函数将打印一个包含指定索引处的值的字符串。

```dart
var list = ['apples', 'bananas', 'oranges'];
list.forEach((item) {
  print('${list.indexOf(item)}: $item');
});
```

如果函数只包含一个语句，则可以使用箭头表示法缩短它。 以下行粘贴和上面的代码是等效的

```dart
list.forEach(
    (item) => print('${list.indexOf(item)}: $item'));
```

## 静态作用域（Lexical scope）

Dart 是静态作用域语言，这意味着变量的范围是静态确定的，只需通过代码的布局。 您可以“向外跟随花括号”以查看变量是否在范围内。

以下是每个范围级别包含变量的嵌套函数示例：

```dart
bool topLevel = true;

void main() {
  var insideMain = true;

  void myFunction() {
    var insideFunction = true;

    void nestedFunction() {
      var insideNestedFunction = true;

      assert(topLevel);
      assert(insideMain);
      assert(insideFunction);
      assert(insideNestedFunction);
    }
  }
}
```

**注意**nestedFunction（）可以使用每个级别的变量，一直到顶级。

## 词法闭包（Lexical closures）

闭包是一个函数对象，它可以访问其词法范围中的变量，即使该函数在其原始范围之外使用也是如此。

函数可以关闭周围范围中定义的变量。 在以下示例中，makeAdder（）捕获变量addBy。 无论返回的函数在哪里，它都会记住addBy。

```dart
/// Returns a function that adds [addBy] to the
/// function's argument.
Function makeAdder(num addBy) {
  return (num i) => addBy + i;
}

void main() {
  // Create a function that adds 2.
  var add2 = makeAdder(2);

  // Create a function that adds 4.
  var add4 = makeAdder(4);

  assert(add2(3) == 5);
  assert(add4(3) == 7);
}
```

## 测试函数是否相等

以下是测试顶级函数，静态方法和实例方法的相等性的示例：

```dart
void foo() {} // A top-level function

class A {
  static void bar() {} // A static method
  void baz() {} // An instance method
}

void main() {
  var x;

  // Comparing top-level functions.
  x = foo;
  assert(foo == x);
	print('${foo == x}');
  // Comparing static methods.
  x = A.bar;
  assert(A.bar == x);
print('${A.bar == x}');
  // Comparing instance methods.
  var v = A(); // Instance #1 of A
  var w = A(); // Instance #2 of A
  var y = w;
  x = w.baz;

  // These closures refer to the same instance (#2),
  // so they're equal.
  assert(y.baz == x);
print('${y.baz == x}');
  // These closures refer to different instances,
  // so they're unequal.
  assert(v.baz != w.baz);
  print('${v.baz != w.baz}');
}
```

## 返回值

所有函数都返回一个值。 如果未指定返回值，则该语句返回null; 隐式附加到函数体。

```dart
foo() {}

assert(foo() == null);
```