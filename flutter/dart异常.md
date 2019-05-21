# dart异常

例外是指示发生意外事件的错误。 如果未捕获异常，则会引发引发异常的隔离，并且通常会隔离隔离及其程序。

与Java相比，Dart的所有异常都是未经检查的异常。 方法不会声明它们可能引发的异常，并且您不需要捕获任何异常。

Dart提供了Exception和Error类型，以及许多预定义的子类型。 当然，您可以定义自己的例外情况。 但是，Dart程序可以抛出任何非null对象 - 不仅仅是Exception和Error对象 - 作为异常。

**抛出异常：**

```dart
throw FormatException('Expected at least 1 section');
```

抛出任意对象作为异常：

```dart
throw 'Out of llamas!';
```

**捕获异常：**

捕获或捕获异常会阻止异常传播（除非您重新抛出异常）。 捕获异常使您有机会处理它：

```dart
try {
  breedMoreLlamas();
} on OutOfLlamasException {
  buyMoreLlamas();
}
```

要处理可能抛出多种类型异常的代码，可以指定多个catch子句。 与抛出对象的类型匹配的第一个catch子句处理异常。 如果catch子句未指定类型，则该子句可以处理任何类型的抛出对象：

```dart
ry {
  breedMoreLlamas();
} on OutOfLlamasException {
  // A specific exception
  buyMoreLlamas();
} on Exception catch (e) {
  // Anything else that is an exception
  print('Unknown exception: $e');
} catch (e) {
  // No specified type, handles all
  print('Something really unknown: $e');
}
```

如前面的代码所示，您可以使用on或catch或两者。 需要指定异常类型时使用。 在异常处理程序需要异常对象时使用catch。

您可以为catch（）指定一个或两个参数。 第一个是抛出的异常，第二个是堆栈跟踪（StackTrace对象）。

```dart
try {
  // ···
} on Exception catch (e) {
  print('Exception details:\n $e');
} catch (e, s) {
  print('Exception details:\n $e');
  print('Stack trace:\n $s');
}
```

要部分处理异常，同时允许它传播，请使用rethrow关键字。

```dart
void misbehave() {
  try {
    dynamic foo = true;
    print(foo++); // Runtime error
  } catch (e) {
    print('misbehave() partially handled ${e.runtimeType}.');
    rethrow; // Allow callers to see the exception.
  }
}

void main() {
  try {
    misbehave();
  } catch (e) {
    print('main() finished handling ${e.runtimeType}.');
  }
}
```

**Finally：**

无论是否抛出异常，要确保某些代码运行，请使用finally子句。 如果没有catch子句与异常匹配，则在finally子句运行后传播异常：

```dart
try {
  breedMoreLlamas();
} finally {
  // Always clean up, even if an exception is thrown.
  cleanLlamaStalls();
}
```

finally子句在任何匹配的catch子句之后运行：

```dart
try {
  breedMoreLlamas();
} catch (e) {
  print('Error: $e'); // Handle the exception first.
} finally {
  cleanLlamaStalls(); // Then clean up.
}
```

