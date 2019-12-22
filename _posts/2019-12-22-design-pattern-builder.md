---
title: 构建者设计模式
categories:
  - design pattern
tags:
  - builder
---
最近在学习SpringSecurity的源码，发现源码中有大量Builder类，要能读懂源码及源码前后的设计思想，得先了解设计模式，所以打算开始系统的学习Java中的设计模式，23种设计模式这是第一篇。

当要创建的对象包含很多属性时，Factory 和 Abstract Factory 设计模式存在三个主要问题。
- 从客户端程序传递到工厂类的参数太多，容易出错，因为大多数时候，参数的类型是相同的，从客户端很难维护参数的顺序
- 有些参数可能是可选的，但是在工厂模式我们被迫传入所有参数，可选参数则需要传入null值
- 如果对象很重，而且它的创建很复杂，那么所有这些复杂性都将成为工厂类的一部分，这是令人困惑的

我们可以通过为构造函数提供所需的参数，然后通过不同的`setter`方法设置可选参数来解决大量参数的问题。 这种方法的问题在于，除非显式设置所有属性，否则对象状态将不一致。
Builder模式通过提供一种逐步构建对象的方法，并提供一种实际返回最终对象的方法，解决了大量可选参数和不一致状态的问题。

我们看看如何在java中实现Builder设计模式。
- 首先，你需要创建一个静态内部类，然后将外部类中的所有参数复制到Builder类中。 我们应该遵循变量命名原则，如果类名是`Computer`那么建筑类应该命名为`ComputerBuilder`。
- Builder类应该有一个公共构造函数，该构造函数具有所有必需的属性作为参数
- Builder类应该具有设置可选参数的方法，并且在设置可选属性之后应该返回相同的Builder对象
- 最后一步是提供一个`build()`方法，该方法将返回客户端程序所需的对象。 为此，我们需要在类中有一个私有构造函数，用Builder类作为参数

下面是构建者模式示例代码，其中我们有一个`Computer`类和`ComputerBuilder`类来构建它。
```java
package com.xym.design.builder;

public class Computer {

    //required parameters
    private String HDD;
    private String RAM;

    //optional parameters
    private boolean isGraphicsCardEnabled;
    private boolean isBluetoothEnabled;


    public String getHDD() {
        return HDD;
    }

    public String getRAM() {
        return RAM;
    }

    public boolean isGraphicsCardEnabled() {
        return isGraphicsCardEnabled;
    }

    public boolean isBluetoothEnabled() {
        return isBluetoothEnabled;
    }

    private Computer(ComputerBuilder builder) {
        this.HDD=builder.HDD;
        this.RAM=builder.RAM;
        this.isGraphicsCardEnabled=builder.isGraphicsCardEnabled;
        this.isBluetoothEnabled=builder.isBluetoothEnabled;
    }

    @Override
    public String toString() {
        return "Computer{" +
                "HDD='" + HDD + '\'' +
                ", RAM='" + RAM + '\'' +
                ", isGraphicsCardEnabled=" + isGraphicsCardEnabled +
                ", isBluetoothEnabled=" + isBluetoothEnabled +
                '}';
    }

    //Builder Class
    public static class ComputerBuilder{

        // required parameters
        private String HDD;
        private String RAM;

        // optional parameters
        private boolean isGraphicsCardEnabled;
        private boolean isBluetoothEnabled;

        public ComputerBuilder(String hdd, String ram){
            this.HDD=hdd;
            this.RAM=ram;
        }

        public ComputerBuilder setGraphicsCardEnabled(boolean isGraphicsCardEnabled) {
            this.isGraphicsCardEnabled = isGraphicsCardEnabled;
            return this;
        }

        public ComputerBuilder setBluetoothEnabled(boolean isBluetoothEnabled) {
            this.isBluetoothEnabled = isBluetoothEnabled;
            return this;
        }

        public Computer build(){
            return new Computer(this);
        }

    }

}

```
**注意，Computer类只有getter方法，没有公共构造函数。 所以获得Computer对象的唯一方法是通过ComputerBuilder类。**
下面是一个构建者模式示例测试程序，演示如何使用Builder类获取对象。
```java
package com.xym.design.builder;

public class TestBuilderPattern {

    public static void main(String[] args) {
        Computer comp = new Computer.ComputerBuilder(
                "500 GB", "2 GB").setBluetoothEnabled(true)
                .setGraphicsCardEnabled(true).build();
        System.out.println(comp);
    }

}
```

JDK中的一些构建者模式示例如下:
- java.lang.StringBuilder#append() (unsynchronized)
- java.lang.StringBuffer#append() (synchronized)
以上就是java中的构建者设计模式。