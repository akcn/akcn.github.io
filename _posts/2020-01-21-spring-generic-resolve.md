---
title: Spring4 解析泛型参数
categories:
  - Spring
tags:
  - generic resolve
---
泛型参数在运行时会被擦除，在类信息Class中依然有关于泛型的一些信息，可以通过反射得到
在API文档中，Type接口的说明如下：
Type is the common superinterface for all types in the Java programming language. These include raw types, parameterized types, array types, type variables and primitive types.
Type 是 Java 编程语言中所有类型的公共高级接口。它们包括原始类型、参数化类型、数组类型、类型变量和基本类型。
从JDK1.5开始使用。

Class实现了Type，Type的其他子接口还有：
- TypeVariable：类型参数，可以有上界，比如：T extends Number
- ParameterizedType：参数化的类型，有原始类型和具体的类型参数，比如：List<String> 
- WildcardType：通配符类型，比如：?, ? extends Number, ? super Integer


Class有如下方法，可以获取类的泛型参数信息：
`public TypeVariable<Class<T>>[] getTypeParameters()`

Field有如下方法：
`public Type getGenericType()`

Method有如下方法：
`
public Type getGenericReturnType()
public Type[] getGenericParameterTypes()
public Type[] getGenericExceptionTypes()
`
Constructor有如下方法：
`public Type[] getGenericParameterTypes() `

示例:

```java
public class GenericDemo {
  static class GenericTest<U extends Comparable<U>, V> {
    U u;
    V v;
    List<String> list;

    public U test(List<? extends Number> numbers) {
      return null;
    }
  }

  public static void main(String[] args) throws Exception {
    Class<?> cls = GenericTest.class;
    // 类的类型参数
    for (TypeVariable t : cls.getTypeParameters()) {
      System.out.println(t.getName() + " extends " + Arrays.toString(t.getBounds()));
    }

    // 字段 - 泛型类型
    Field fu = cls.getDeclaredField("u");
    System.out.println(fu.getGenericType());

    // 字段 - 参数化的类型
    Field flist = cls.getDeclaredField("list");
    Type listType = flist.getGenericType();
    if (listType instanceof ParameterizedType) {
      ParameterizedType pType = (ParameterizedType) listType;
      System.out.println("raw type: " + pType.getRawType() + ",type arguments:"
              + Arrays.toString(pType.getActualTypeArguments()));
    }

    // 方法的泛型参数
    Method m = cls.getMethod("test", new Class[] { List.class });
    for (Type t : m.getGenericParameterTypes()) {
      System.out.println(t);
    }
  }
}
```
输出:
```bash
U extends [java.lang.Comparable<U>]
V extends [class java.lang.Object]
U
raw type: interface java.util.List,type arguments:[class java.lang.String]
java.util.List<? extends java.lang.Number>
```
使用Java原生的Api操作起来相对比较麻烦，Spring提供的ResolvableType API，提供了更加简单易用的泛型操作支持，对于获取更复杂的泛型操作ResolvableType更加简单。
演示代码:
```java
public class ResolveGenericDemo {
  interface IService<M,N>{}

  static class A{}
  static class B{}
  static class C{}
  static class D{}

  static class ABService implements IService<A, B> {}
  static class CDService implements IService<C, D> {}

  private IService<A, B> abService;
  private IService<C, D> cdService;
  private List<List<String>> list;
  private Map<String, Map<String, Integer>> map;
  private List<String>[] array;

  public ResolveGenericDemo(List<List<String>> list, Map<String, Map<String, Integer>> map) {}

  private HashMap<String, List<String>> method() {
    return null;
  }

  public static void main(String[] args) {
    // 得到类型的ResolvableType
    ResolvableType resolvableType = ResolvableType.forClass(ABService.class);
    ResolvableType[] interfaces = resolvableType.getInterfaces();
    System.out.println(Arrays.toString(interfaces));
    // 得到该类实现的第1个接口的泛型参数的第1个位置（从0开始）的类型信息
    // 通过getGeneric（泛型参数索引）得到某个位置的泛型；
    // resolve()把实际泛型参数解析出来
    Class<?> resolve = interfaces[0].getGeneric(0).resolve();
    System.out.println(resolve);

    // 得到字段级别的泛型信息
    ResolvableType resolvableType2 = ResolvableType.forField(/* 通过Spring反射工具类获取Field */ReflectionUtils.findField(ResolveGenericDemo.class, "cdService"));
    Class<?> resolve2 = resolvableType2.getGeneric(0).resolve();
    System.out.println(resolve2);

    ResolvableType resolvableType3 = ResolvableType.forField(ReflectionUtils.findField(ResolveGenericDemo.class, "list"));
    //  List<List<String>> list;是一种嵌套的泛型用例，我们可以通过如下操作获取String类型：
    Class<?> resolve3 = resolvableType3.getGeneric(0).getGeneric(0).resolve();
    // 更简单的写法:
    // Class<?> resolve3 = resolvableType3.getGeneric(0, 0).resolve();
    System.out.println(resolve3);


    ResolvableType resolvableType4 = ResolvableType.forField(ReflectionUtils.findField(ResolveGenericDemo.class, "map"));
    // 比如Map<String, Map<String, Integer>> map;我们想得到Integer，可以使用：
    Class<?> resolve4 = resolvableType4.getGeneric(1).getGeneric(1).resolve();
    // 更简单的写法:
    // Class<?> resolve4 = resolvableType4.getGeneric(1, 1).resolve();
    System.out.println(resolve4);

    // 得到方法返回值的泛型信息
    ResolvableType resolvableType5 = ResolvableType.forMethodReturnType(ReflectionUtils.findMethod(ResolveGenericDemo.class, "method"));
    // 得到Map中的List中的String泛型实参：
    Class<?> resolve5 = resolvableType5.getGeneric(1, 0).resolve();
    System.out.println(resolve5);

    // 得到构造器参数的泛型信息
    ResolvableType resolvableType6 =
            ResolvableType.forConstructorParameter(ClassUtils.getConstructorIfAvailable(ResolveGenericDemo.class, List.class, Map.class), 1);
    // 我们可以通过如下方式得到第1个参数（ Map<String, Map<String, Integer>>）中的Integer：
    Class<?> resolve6 = resolvableType6.getGeneric(1, 1).resolve();
    System.out.println(resolve6);

    // 如对于private List<String>[] array; 可以通过如下方式获取List的泛型实参String：
    ResolvableType resolvableType7 = ResolvableType.forField(ReflectionUtils.findField(ResolveGenericDemo.class, "array"));
    resolvableType7.isArray();// 判断是否是数组
    Class<?> resolve7 = resolvableType7.getComponentType().getGeneric(0).resolve();
    System.out.println(resolve7);

    // 自定义泛型类型
    // 相当于创建一个List<String>类型:
    ResolvableType resolvableType8 = ResolvableType.forClassWithGenerics(List.class, String.class);
    // 相当于创建一个List<String>[]数组:
    ResolvableType resolvableType9 = ResolvableType.forArrayComponent(resolvableType8);
    // 得到相应的泛型信息:
    Class<?> resolve8 = resolvableType9.getComponentType().getGeneric(0).resolve();
    System.out.println(resolve8);

    // 泛型等价比较
    System.out.println(resolvableType7.isAssignableFrom(resolvableType9));

    // 创建一个List<Integer>[]数组，与之前的List<String>[]数组比较，将返回false。
    ResolvableType resolvableType10 = ResolvableType.forClassWithGenerics(List.class, Integer.class);
    ResolvableType resolvableType11= ResolvableType.forArrayComponent(resolvableType10);
    resolvableType11.getComponentType().getGeneric(0).resolve();
    System.out.println(resolvableType7.isAssignableFrom(resolvableType11));
  }
}
```
输出:
```bash
[com.xym.treedemo.type.ResolveGenericDemo$IService<com.xym.treedemo.type.ResolveGenericDemo$A, com.xym.treedemo.type.ResolveGenericDemo$B>]
class com.xym.treedemo.type.ResolveGenericDemo$A
class com.xym.treedemo.type.ResolveGenericDemo$C
class java.lang.String
class java.lang.Integer
class java.lang.String
class java.lang.Integer
class java.lang.String
class java.lang.String
true
false
```

从如上操作可以看出其泛型操作功能十分完善，尤其在嵌套的泛型信息获取上相当简洁。目前整个Spring4环境都使用这个API来操作泛型信息。

> 参考: [https://www.iteye.com/blog/jinnianshilongnian-1993608](https://www.iteye.com/blog/jinnianshilongnian-1993608)
> [https://www.cnblogs.com/swiftma/p/6804342.html](https://www.cnblogs.com/swiftma/p/6804342.html)