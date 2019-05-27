# 异步支持

Dart库中包含许多返回Future或Stream对象的函数。 这些函数是异步的：它们在设置可能耗时的操作（例如I / O）后返回，而不等待该操作完成。

async和await关键字支持异步编程，允许您编写看起来类似于同步代码的异步代码。

## 处理Future

当您需要完成Future的结果时，您有两个选择：

1. 使用async和await。
2. 使用Future API，

使用async和await的代码是异步的，但它看起来很像同步代码。 例如，这里有一些代码使用await来等待异步函数的结果：

```dart
await lookUpVersion();
```

要使用await，代码必须位于异步函数中 - 标记为async的函数：

```dart
Future checkVersion() async {
  var version = await lookUpVersion();
  // Do something with version
}
```

**注意：**虽然异步函数可能执行耗时的操作，但它不会等待这些操作。 相反，异步函数只在遇到第一个await表达式（详细信息）时才会执行。 然后它返回一个Future对象，仅在await表达式完成后才恢复执行。

使用try，catch，最后在使用await的代码中处理错误：

```dart
try {
  version = await lookUpVersion();
} catch (e) {
  // React to inability to look up the version
}
```

您可以在异步函数中多次使用等待。 例如，以下代码等待三次函数结果：

```dart
var entrypoint = await findEntrypoint();
var exitCode = await runExecutable(entrypoint, args);
await flushThenExit(exitCode);
```

在await表达式中，表达式的值通常是Future; 如果不是，那么该值将自动包装在Future中。 此Future对象表示返回对象的promise。 await表达式的值是返回的对象。 await表达式使执行暂停，直到该对象可用。

如果在使用await时遇到编译时错误，请确保await在异步函数中。 例如，要在应用程序的main（）函数中使用await，main（）的主体必须标记为async：

```dart
Future main() async {
  checkVersion();
  print('In main: version is ${await lookUpVersion()}');
}
```

## 声明异步函数

异步函数是一个函数，其主体用async修饰符标记。

将async关键字添加到函数使其返回Future。 例如，考虑这个同步函数，它返回一个String：

```dart
String lookUpVersion() => '1.0.0';
```

如果将其更改为异步函数 - 例如，因为将来的实现将非常耗时 - 返回的值是Future：

```dart
Future<String> lookUpVersion() async => '1.0.0';
```

请注意，函数的主体不需要使用Future API。 如有必要，Dart会创建Future对象。

如果您的函数没有返回有用的值，请使其返回类型为Future <void>。