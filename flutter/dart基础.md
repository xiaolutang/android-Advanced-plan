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

整数是没有小数点的数字。 以下是定义整数文字的一些示例：

```dart
var x = 1;
var hex = 0xDEADBEEF;
```

如果数字包含小数那么他是double类型

```dart
var y = 1.1;
var exponents = 1.42e5;
```

从Dart 2.1开始，必要时整数文字会自动转换为double：

```dart
double z = 1; // Equivalent to double z = 1.0.
```

如何将数字与字符串之间互相转换

```dart
// String -> int
var one = int.parse('1');
assert(one == 1);

// String -> double
var onePointOne = double.parse('1.1');
assert(onePointOne == 1.1);

// int -> String
String oneAsString = 1.toString();
assert(oneAsString == '1');

// double -> String
String piAsString = 3.14159.toStringAsFixed(2);
assert(piAsString == '3.14');
```

int类型指定传统的按位移位（<<，>>），AND（＆）和OR（|）运算符。 例如：

```dart
assert((3 << 1) == 6); // 0011 << 1 == 0110
assert((3 >> 1) == 1); // 0011 >> 1 == 0001
assert((3 | 4) == 7); // 0011 | 0100 == 0111
```

文字数字是编译时常量。 许多算术表达式也是编译时常量，只要它们的操作数是编译为数字的编译时常量。

```dart
const msPerSecond = 1000;
const secondsUntilRetry = 5;
const msUntilRetry = secondsUntilRetry * msPerSecond;
```

### Strings

Dart字符串是一系列UTF-16代码单元。 您可以使用单引号或双引号来创建字符串：

```dart
var s1 = 'Single quotes work well for string literals.';
var s2 = "Double quotes work just as well.";
var s3 = 'It\'s easy to escape the string delimiter.';
var s4 = "It's even easier to use the other delimiter.";
```

您可以使用$ {expression}将表达式的值放在字符串中。 如果表达式是标识符，则可以跳过{}。 为了获得对应于对象的字符串，Dart调用对象的toString（）方法。

```dart
var s = 'string interpolation';

assert('Dart has $s, which is very handy.' ==
    'Dart has string interpolation, ' +
        'which is very handy.');
assert('That deserves all caps. ' +
        '${s.toUpperCase()} is very handy!' ==
    'That deserves all caps. ' +
        'STRING INTERPOLATION is very handy!');
```

**注意**：==运算符测试两个对象是否相同。 如果两个字符串包含相同的代码单元序列，则它们是等效的。

您可以使用相邻的字符串文字或+运算符来连接字符串：

```dart
var s1 = 'String '
    'concatenation'
    " works even over line breaks.";
assert(s1 ==
    'String concatenation works even over '
        'line breaks.');

var s2 = 'The + operator ' + 'works, as well.';
assert(s2 == 'The + operator works, as well.');
```

创建多行字符串的另一种方法：使用带有单引号或双引号的三重引号

```dart
var s1 = '''
You can create
multi-line strings like this one.
''';

var s2 = """This is also a
multi-line string.""";
```

您可以通过在其前面加上r来创建“原始”字符串：

```dart
var s = r'In a raw string, not even \n gets special treatment.';
```

文字字符串是编译时常量，只要任何插值表达式是编译时常量，其值为null或数值，字符串或布尔值

```dart
const aConstNum = 0;
const aConstBool = true;
const aConstString = 'a constant string';

// These do NOT work in a const string.
var aNum = 0;
var aBool = true;
var aString = 'a string';
const aConstList = [1, 2, 3];

const validConstString = '$aConstNum $aConstBool $aConstString';
// const invalidConstString = '$aNum $aBool $aString $aConstList';
```

###  Booleans

为了表示布尔值，Dart有一个名为bool的类型。 只有两个对象具有bool类型：boolean literals true和false，它们都是编译时常量。

Dart的类型安全意味着您不能使用if（nonbooleanValue）或assert（nonbooleanValue）之类的代码。 相反，明确检查值，如下所示:

```dart
// Check for an empty string.
var fullName = '';
assert(fullName.isEmpty);

// Check for zero.
var hitPoints = 0;
assert(hitPoints <= 0);

// Check for null.
var unicorn;
assert(unicorn == null);

// Check for NaN.
var iMeantToDoThis = 0 / 0;
assert(iMeantToDoThis.isNaN);
```

### Lists

也许几乎每种编程语言中最常见的集合是数组或有序的对象组。 在Dart中，数组是List对象，因此大多数人只是将它们称为列表。

Dart列表文字看起来像JavaScript数组文字。 这是一个简单的Dart列表：

```dart
var list = [1, 2, 3];
```

列表使用从零开始的索引，其中0是第一个元素的索引，list.length  -  1是最后一个元素的索引。 您可以获得列表的长度并像在JavaScript中一样引用列表元素：

```dart
var list = [1, 2, 3];
assert(list.length == 3);
assert(list[1] == 2);

list[1] = 1;
assert(list[1] == 1);
```

要创建一个编译时常量的列表，请在列表文字之前添加const：

```dart
var constantList = const [1, 2, 3];
// constantList[1] = 1; // Uncommenting this causes an error.
```

###  Sets

Dart中的一组是一组无序的独特物品。 对集合的Dart支持由set literals和Set类型提供。

这是一个简单的Dart集，使用set literal创建：

```dart
var halogens = {'fluorine', 'chlorine', 'bromine', 'iodine', 'astatine'};
```

要创建一个空集，请使用前面带有类型参数的{}，或者将{}赋给类型为Set的变量：

```dart
var names = <String>{};
// Set<String> names = {}; // This works, too.
// var names = {}; // Creates a map, not a set.
```

使用add（）或addAll（）方法为set添加元素：

```dart
var elements = <String>{};
elements.add('fluorine');
elements.addAll(halogens);
```

使用.length来获取set的项目数：

```dart
var elements = <String>{};
elements.add('fluorine');
elements.addAll(halogens);
assert(elements.length == 5);
```

要创建一个编译时常量的集合，请在set 之前添加const：

```dart
final constantSet = const {
  'fluorine',
  'chlorine',
  'bromine',
  'iodine',
  'astatine',
};
// constantSet.add('helium'); // Uncommenting this causes an error.
```

###  Maps

通常，map是关联键和值的对象。 键和值都可以是任何类型的对象。 每个键只出现一次，但您可以多次使用相同的值。

下面是创建map的示例：

```dart
var gifts = {
  // Key:    Value
  'first': 'partridge',
  'second': 'turtledoves',
  'fifth': 'golden rings'
};

var nobleGases = {
  2: 'helium',
  10: 'neon',
  18: 'argon',
};
```

您可以使用Map构造函数创建相同的对象：

```dart
var gifts = Map();
gifts['first'] = 'partridge';
gifts['second'] = 'turtledoves';
gifts['fifth'] = 'golden rings';

var nobleGases = Map();
nobleGases[2] = 'helium';
nobleGases[10] = 'neon';
nobleGases[18] = 'argon';
```

像在JavaScript中一样，将新的键值对添加到现有map：

```dart
var gifts = {'first': 'partridge'};
gifts['fourth'] = 'calling birds'; // Add a key-value pair
```

以与在JavaScript中相同的方式从map中获取值：

```dart
var gifts = {'first': 'partridge'};
assert(gifts['first'] == 'partridge');
```

如果您查找不在map中的键，则会返回null：

```dart
var gifts = {'first': 'partridge'};
assert(gifts['fifth'] == null);
```

可以使用.length获取map的长度

```dart
var gifts = {'first': 'partridge'};
gifts['fourth'] = 'calling birds';
assert(gifts.length == 2);
```

要创建一个编译时常量的映射，请在map 之前添加const：

```dart
final constantMap = const {
  2: 'helium',
  10: 'neon',
  18: 'argon',
};

// constantMap[2] = 'Helium'; // Uncommenting this causes an error.
```

### Runes

在Dart中，Runes是字符串的UTF-32代码点。

Unicode为世界上所有书写系统中使用的每个字母，数字和符号定义唯一的数值。 由于Dart字符串是UTF-16代码单元的序列，因此在字符串中表示32位Unicode值需要特殊语法。

表达Unicode代码点的常用方法是\ uXXXX，其中XXXX是4位十六进制值。 例如，心脏角色（♥）是\ u2665。 要指定多于或少于4个十六进制数字，请将值放在大括号中。 例如，笑的表情符号（😆）是\ u {1f600}。

String类有几个属性可用于提取符文信息。 codeUnitAt和codeUnit属性返回16位代码单元。 使用runes属性获取字符串的符文。

以下示例说明了符文，16位代码单元和32位代码点之间的关系。 

```dart
main() {
  var clapping = '\u{1f44f}';
  print(clapping);
  print(clapping.codeUnits);
  print(clapping.runes.toList());

  Runes input = new Runes(
      '\u2665  \u{1f605}  \u{1f60e}  \u{1f47b}  \u{1f596}  \u{1f44d}');
  print(new String.fromCharCodes(input));
}
```

**注意：**使用列表操作操作符文时要小心。 这种方法很容易分解，具体取决于特定的语言，字符集和操作。

###  Symbols

Symbol对象表示Dart程序中声明的运算符或标识符。 您可能永远不需要使用符号，但它们对于按名称引用标识符的API非常有用，因为缩小会更改标识符名称而不会更改标识符符号。

要获取标识符的符号，请使用符号文字，它只是＃后跟标识符：

```dart
#radix
#bar
```