---
title: 适配器设计模式
categories: 
  - design pattern
tags: 
  - adapter
---
适配器设计模式是结构型设计模式之一，它使得两个不相关的接口可以一起工作。 连接这些不相关接口的对象称为 Adapter。
适配器设计模式的一个现实生活中的例子是移动充电器。 手机电池需要5伏电压充电，但正常的插座产生120V(美国)或240V(印度)或220V(中国)。 所以这款手机充电器就像一个连接手机充电口和墙上插座的适配器。
首先 我们新建两个类`Volt`(测量电压)和`Socket`(固定输出电压120V)

```java
package com.xym.design.adapter;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class Volt {
    private int volts;
}
```
```java
package com.xym.design.adapter;

public class Socket {

    public Volt getVolt(){
        return new Volt(120);
    }
}
```

现在我们要建立一个适配器，可以产生3伏，12伏和默认120伏。 因此，首先我们将使用这些方法创建一个适配器接口。
```java
package com.xym.design.adapter;

public interface SocketAdapter {
    Volt get120Volt();

    Volt get12Volt();

    Volt get3Volt();
}
```

在实现 Adapter 模式时，有两种方式——类适配器和对象适配器，这两种方式产生的结果是一样的。
- 类适配器 该方式使用Java的继承并扩展源接口，在我们的例子中是 Socket 类
- 对象适配器 该方式使用Java的组合，并且适配器包含源对象

适配器的类适配器方式实现:
```java
package com.xym.design.adapter;

//Using inheritance for adapter pattern
public class SocketClassAdapterImpl extends Socket implements SocketAdapter{

    @Override
    public Volt get120Volt() {
        return getVolt();
    }

    @Override
    public Volt get12Volt() {
        Volt v= getVolt();
        return convertVolt(v,10);
    }

    @Override
    public Volt get3Volt() {
        Volt v= getVolt();
        return convertVolt(v,40);
    }

    private Volt convertVolt(Volt v, int i) {
        return new Volt(v.getVolts()/i);
    }

}
```
适配器的对象适配器实现:
```java
public class SocketObjectAdapterImpl implements SocketAdapter{

    //Using Composition for adapter pattern
    private Socket sock = new Socket();

    @Override
    public Volt get120Volt() {
        return sock.getVolt();
    }

    @Override
    public Volt get12Volt() {
        Volt v= sock.getVolt();
        return convertVolt(v,10);
    }

    @Override
    public Volt get3Volt() {
        Volt v= sock.getVolt();
        return convertVolt(v,40);
    }

    private Volt convertVolt(Volt v, int i) {
        return new Volt(v.getVolts()/i);
    }
}
```
**注意，两个适配器实现几乎是相同的，它们都实现了 SocketAdapter 接口。 适配器接口也可以是抽象类。**

下面是使用适配器设计模式实现的测试程序。
```java
package com.xym.design.adapter;

public class AdapterPatternTest {

    public static void main(String[] args) {

        testClassAdapter();
        testObjectAdapter();
    }

    private static void testObjectAdapter() {
        SocketAdapter sockAdapter = new SocketObjectAdapterImpl();
        Volt v3 = getVolt(sockAdapter,3);
        Volt v12 = getVolt(sockAdapter,12);
        Volt v120 = getVolt(sockAdapter,120);
        System.out.println("v3 volts using Object Adapter="+v3.getVolts());
        System.out.println("v12 volts using Object Adapter="+v12.getVolts());
        System.out.println("v120 volts using Object Adapter="+v120.getVolts());
    }

    private static void testClassAdapter() {
        SocketAdapter sockAdapter = new SocketClassAdapterImpl();
        Volt v3 = getVolt(sockAdapter,3);
        Volt v12 = getVolt(sockAdapter,12);
        Volt v120 = getVolt(sockAdapter,120);
        System.out.println("v3 volts using Class Adapter="+v3.getVolts());
        System.out.println("v12 volts using Class Adapter="+v12.getVolts());
        System.out.println("v120 volts using Class Adapter="+v120.getVolts());
    }

    private static Volt getVolt(SocketAdapter sockAdapter, int i) {
        switch (i){
            case 3: return sockAdapter.get3Volt();
            case 12: return sockAdapter.get12Volt();
            case 120: return sockAdapter.get120Volt();
            default: return sockAdapter.get120Volt();
        }
    }
}
```
当我们运行上面的测试程序时，我们得到下面的输出:
```bash
v3 volts using Class Adapter=3
v12 volts using Class Adapter=12
v120 volts using Class Adapter=120
v3 volts using Object Adapter=3
v12 volts using Object Adapter=12
v120 volts using Object Adapter=120
```

JDK中使用适配器模式示例如下:
- java.util.Arrays#asList()
- java.io.InputStreamReader(InputStream) (returns a Reader)
- java.io.OutputStreamWriter(OutputStream) (returns a Writer)