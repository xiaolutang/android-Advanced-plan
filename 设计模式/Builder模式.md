# Builder模式

**概念：**将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以有不同的表示。

**使用场景：**

1. 相同的方法，不同的执行顺序，产生不同的结果
2. 多个部件或零件，都可以装配到一个对象中，但产生的运行结果又不相同时
3. 产品类特别复杂，或者产品类中的调用顺序不同产生了不同的作用，这个时候使用建造者模式非常适合
4. 当初始化一个对象特别复杂，如参数的，且很多参数都具有默认参数时。

**uml图：**

![img](.\builder模式uml.png)

以简单的电脑创建过程为例：

```java
public abstract class Builder {
    public abstract void buildBoard(String board);
    public abstract void buildDisplay(String display);
    public abstract void buildOs();
    public abstract Computer create();
}
```

```java
public abstract class Computer {
    protected String mBoard;
    protected String mDisplay;
    protected String mOs;

    protected Computer() {
    }

    public void setmBoard(String mBoard) {
        this.mBoard = mBoard;
    }

    public void setmDisplay(String mDisplay) {
        this.mDisplay = mDisplay;
    }

    public abstract void setOs() ;

    @Override
    public String toString() {
        return "Computer{" +
                "mBoard='" + mBoard + '\'' +
                ", mDisplay='" + mDisplay + '\'' +
                ", mOs='" + mOs + '\'' +
                '}';
    }
}
```

```java
public class MacBook extends Computer {
    @Override
    public void setOs() {
        mOs = "我是苹果系统";
    }
}
```

```java
public class MacbookBuilder extends Builder {
    private Computer mC = new MacBook();
    @Override
    public void buildBoard(String board) {
        mC.setmBoard( board );
    }

    @Override
    public void buildDisplay(String display) {
        mC.setmDisplay( display );
    }

    @Override
    public void buildOs() {
        mC.setOs();
    }

    @Override
    public Computer create() {
        return mC;
    }
}
```

```java
public class Director {
    Builder mBuilder;

    public Director(Builder mBuilder) {
        this.mBuilder = mBuilder;
    }

    public void construct(String board,String display){
        mBuilder.buildBoard( board );
        mBuilder.buildDisplay( display );
        mBuilder.buildOs();
    }

    public static void main(String[] args){
        Builder builder = new MacbookBuilder();
        Director director = new Director( builder );
        director.construct( "XXX主板","XXX显示器" );
        System.out.print( "电脑信息："+builder.create().toString() );
    }
}
```

**运行结果：**

```
电脑信息：Computer{mBoard='XXX主板', mDisplay='XXX显示器', mOs='我是苹果系统'}
```

在开发的过程中经常将Director省略，使用链式调用builder的方式来进行构建。