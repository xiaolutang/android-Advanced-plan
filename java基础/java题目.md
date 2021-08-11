# java Exception 和 Error 的区别？

- 首先 Exception 和 Error 都是继承于 Throwable 类，在 Java 中只有 Throwable 类型的实例才可以被抛出（throw）或者捕获（catch），Exception 和 Error 体现了JAVA 这门语言对于异常处理的两种方式。
- Exception 是 Java 程序运行中可预料的异常情况，我们可以获取到这种异常，并且对这种异常进行业务外的处理。它分为检查性异常和非检查性（RuntimeException）异常。两个根本的区别在于，检查性异常必须在编写代码时，使用 try catch 捕获（比如：IOException异常）。非检查性异常 在代码编写时，可以忽略捕获操作（比如：ArrayIndexOutOfBoundsException），这种异常是在代码编写或者使用过程中通过规范可以避免发生的。
- Error 是 Java 程序运行中不可预料的异常情况，这种异常发生以后，会直接导致 JVM 不可处理或者不可恢复的情况。所以这种异常不可能抓取到，比如 OutOfMemoryError、NoClassDefFoundError等。

# NoClassDefFoundError 和 ClassNotFoundException 有什么区别？

ClassNotFoundException 产生的原因是：

Java 支持使用反射方式在运行时动态加载类，例如使用 Class.forName 方法来动态地加载类时，可以将类名作为参数传递给上述方法从而将指定类加载到 JVM 内存中，如果这个类在类路径中没有被找到，那么此时就会在运行时抛出ClassNotFoundException 异常。

原因：

常见问题在于类名书写错误。
当一个类已经被某个类加载器加载到内存中了，此时另一个类加载器又尝试着动态地从同一个包中加载这个类。通过控制动态类加载过程，可以避免上述情况发生。
NoClassDefFoundError 产生的原因是：

如果 JVM 或者 ClassLoader 实例尝试加载（可以通过正常的方法调用，也可能是使用 new 来创建新的对象）类的时候却找不到类的定义。要查找的类在编译的时候是存在的，运行的时候却找不到了。这个时候就会导致 NoClassDefFoundError。

原因：

打包过程漏掉了部分类。
jar包出现损坏或者篡改。
解决方法：
查找那些在开发期间存在于类路径下，但在运行期间却不在类路径下的类。