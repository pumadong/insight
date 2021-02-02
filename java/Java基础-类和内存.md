# 类信息

## 默认构造函数

```java
public class ConstructorDemo {

}
```

这是一个最简单的无显式构造函数的类，我们先编译成.class文件，然后查看.class文件的源码。

可以看到，编译器在编译成字节码时，帮我们生成了默认的构造函数。

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.cl.roadshow.java.lang.classinfo;

public class ConstructorDemo {
    public ConstructorDemo() {
    }
}
```

## 继承父类方法

```java
public class InheritanceDemo extends Parent {

}

abstract class Parent {
	public void show() {
		
	}
}
```



# 内存信息

