如何自定义插件

定义task:

自定义的task并不会直接执行。



gradle构建流程

# 问题？

1. TaskContainer create与TaskContainer register的关系
2. 定义的task 不通过命令的情况下，不会直接执行action。

## 如何打包一个插件

打包一个插件一共有三种方式，

1. build script

   直接在构建脚本中插入插件源代码，这样的好处是，无需执行任何操作既可以自动编译插件并将其包含在构建脚本的类路径中。但是该脚本在构建搅拌之外不可见，因此不能再定义该构建脚本的外部重用该插件。

2. buildSrc project

   您可以将插件的源代码放在rootProjectDir / buildSrc / src / main / java目录（或rootProjectDir / buildSrc / src / main / groovy或rootProjectDir / buildSrc / src / main / kotlin中，具体取决于您喜欢的语言）。 Gradle将负责编译和测试插件，并使其在构建脚本的类路径中可用。 该插件对构建使用的每个构建脚本都是可见的。 但是，它在构建外部不可见，因此您不能在定义该构建的外部重用该插件。

3. Standalone project

   您可以为插件创建一个单独的项目。 这个项目产生并发布了一个JAR，您可以在多个版本中使用它并与他人共享。 通常，此JAR可能包含一些插件，或将几个相关的任务类捆绑到一个库中。 或两者的某种组合。

# 如何定义一个task?

# 如何为Task添加配置？



