# 集成Tinker遇到的问题

## 官方项目编译问题：

> 错误: 无法访问Keep
>   找不到android.support.annotation.Keep的类文件
> 1 个错误
>
> FAILURE: Build failed with an exception.
>
> * What went wrong:
> Execution failed for task ':app:compileDebugJavaWithJavac'.
>
> > Compilation failed; see the compiler error output for details.

![image-20201204100329456](D:\android-Advanced-plan\Android\热修复\官方项目编译问题.png)

是因为项目是androidx，但是远程依赖还是support 。在项目gradle.properties添加

```groovy
android.enableJetifier=true
```

即可解决

## 集成时遇到的问题

> A problem occurred configuring project ':app'.
>
> > No such property: variantConfiguration for class: com.android.build.gradle.internal.variant.ApplicationVariantData

![image-20201204101248673](D:\android-Advanced-plan\Android\热修复\集成时编译问题.png)

原因是本地项目gradle tool 版本与不兼容，参考官方示例项目更改即可解决。

有问题的版本：

![image-20201204101213473](D:\android-Advanced-plan\Android\热修复\有问题的gradle版本.png)

解决后的版本：

![image-20201204101444959](D:\android-Advanced-plan\Android\热修复\解决问题的gradle版本.png)

## 官方demo构建基准补丁apk dex时遇到的问题

> com.tencent.tinker.android.dex.DexException: com.tencent.tinker.android.dex.DexException: Unexpected magic: [100, 101, 120, 10, 48, 51, 56, 0]

![1607350217773](官方demo构建补丁dex时的问题.png)

在生成基准apk的时候不要使用android studio的三角形直接运行，会有问题。使用 gradlew assembleDebug 进行构建。

