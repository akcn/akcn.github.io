---
title: Spring 类型转换
categories:
  - Spring
tags:
  - converter
---

Spring为内置类型提供了开箱即用的各种转换器；这意味着可以转换为基本类型，如String、Integer、Boolean和许多其他类型。

### 内置Converter转换器
Maven依赖:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>20.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```
我们将从 Spring 中可用的现成转换器开始; 让我们看一下 String 到 Integer 的转换:
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = SpringbootdemoApplication.class)
class ConverterDemoTest {

    @Autowired
    ConversionService conversionService;

    @Test
    public void whenConvertStringToIntegerUsingDefaultConverter_thenSuccess() {
        assertThat(conversionService.convert("25", Integer.class)).isEqualTo(25);
    }
}
```
我们唯一需要做的就是自动装载Spring提供的ConversionService并调用convert()方法，第一个参数是我们要转换的值，第二个参数是我们要转换的目标目标类型
除了String转Integer这个例子外，还有很多种其它的组合可供我们使用

### 创建自定义Converter转换器
我们来看一个String转换为Employee的示例
```java
@Data
@AllArgsConstructor
public class Employee {
    private long id;
    private double salary;
}
```
字符串用逗号分隔，分别表示id和salary，如1,5000
为了创建我们的自定义转换器，我们需要实现Converter接口并实现convert ()方法:
```java
public class StringToEmployeeConverter implements Converter<String, Employee> {
    @Override
    public Employee convert(String s) {
        String[] data = s.split(",");
        return new Employee(
                Long.parseLong(data[0]),
                Double.parseDouble(data[1]));
    }
}
```
我们还需要通过将 StringToEmployeeConverter 添加到 FormatterRegistry 来告诉 Spring 这个新转换器。 这可以通过实现 WebMvcConfigurer 重写 addFormatters ()方法来实现:
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToEmployeeConverter());
    }
}
```
只需要这两步 我们的新转换器现在可用于 ConversionService，我们可以像使用其他内置转换器一样使用它:
```java
@Test
public void whenConvertStringToEmployee_thenSuccess() {
    Employee employee = conversionService.convert("1,50000.00", Employee.class);
    Employee actualEmployee = new Employee(1, 50000.00);
    assertThat(conversionService.convert("1,50000.00",
            Employee.class))
            .isEqualToComparingFieldByField(actualEmployee);
}
```
除了使用 ConversionService 进行显式转换之外，Spring 还可以隐式转换 Controller 方法中所有已注册转换器的值:
```java
@RestController
public class StringToEmployeeConverterController {

    @GetMapping("/string-to-employee")
    public ResponseEntity<Object> getStringToEmployee(@RequestParam("employee") Employee employee) {
        return ResponseEntity.ok(employee);
    }
}
```

这是使用转换器更自然的方法。 让我们添加一个测试来看看它是如何运行的:
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = SpringbootdemoApplication.class, webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
class StringToEmployeeConverterControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void getStringToEmployeeTest() throws Exception {
        mockMvc.perform(get("/string-to-employee?employee=1,2000"))
                .andDo(print())
                .andExpect(jsonPath("$.id", is(1)))
                .andExpect(jsonPath("$.salary", is(2000.0))).andExpect(status().isOk());
    }

}
```
正如你所看到的，测试将打印请求以及响应的所有细节。 下面是 JSON 格式的 Employee 对象，作为响应的一部分返回:
`{"id":1,"salary":2000.0}`

### 创建一个ConverterFactory
也可以创建一个 ConverterFactory，根据需要创建转换器。 这对于为 Enums 创建转换器特别有帮助。
让我们来看一个非常简单的 Enum:
```java
public enum Modes {
    ALPHA, BETA;
}
```
接下来，让我们创建一个 StringToEnumConverterFactory，它可以生成用于将 String 转换为任何 Enum 的转换器:
```java
@Component
public class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

    private static class StringToEnumConverter<T extends Enum> implements Converter<String, T> {

        private Class<T> enumType;

        public StringToEnumConverter(Class<T> enumType) {
            this.enumType = enumType;
        }

        @Override
        @SuppressWarnings("unchecked")
        public T convert(String s) {
            return (T) Enum.valueOf(this.enumType, s.trim());
        }
    }

    @Override
    public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
        return new StringToEnumConverter<>(targetType);
    }
}
```
正如我们所看到的，工厂类内部使用了一个转换器接口的实现类。
这里需要注意的一点是，虽然我们将使用 Modes Enum 来演示使用方法，但是在 StringToEnumConverterFactory 中没有提到 Enum。 我们的工厂类足够通用，可以根据需要为任何 Enum 类型生成转换器。
下一步是注册这个工厂类，就像我们在前面的例子中注册转换器一样:
```java
@Override
public void addFormatters(FormatterRegistry registry) {
    registry.addConverter(new StringToEmployeeConverter());
    registry.addConverterFactory(new StringToEnumConverterFactory());
}
```

现在，ConversionService 已经准备好将 string 转换为 Enums:
```java
@Test
public void whenConvertStringToEnum_thenSuccess() {
    assertThat(conversionService.convert("ALPHA", Modes.class))
            .isEqualTo(Modes.ALPHA);
}
```
### 创建一个GenericConverter
Genericconverter 为我们提供了更大的灵活性，可以以牺牲某些类型安全为代价来创建一个更为通用的 Converter。
让我们考虑一个将 Integer、 Double 或 String 转换为 BigDecimal 值的示例。 一个简单的 GenericConverter 可以达到这个目的。
第一步是告诉 Spring 支持哪些类型的转换。 我们通过创建一组 ConvertiblePair 来实现:
```java
@Override
public Set<ConvertiblePair> getConvertibleTypes() {
    ConvertiblePair[] pairs = new ConvertiblePair[] {
            new ConvertiblePair(Number.class, BigDecimal.class),
            new ConvertiblePair(String.class, BigDecimal.class)};
    return ImmutableSet.copyOf(pairs);
}
```
下一步是在同一个类中重写 convert ()方法:
```java
@Override
public Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {
    if (sourceType.getType() == BigDecimal.class) {
        return source;
    }
    if(sourceType.getType() == String.class) {
        String number = (String) source;
        return new BigDecimal(number);
    } else {
        Number number = (Number) source;
        BigDecimal converted = new BigDecimal(number.doubleValue());
        return converted.setScale(2, BigDecimal.ROUND_HALF_EVEN);
    }
}
```
Convert ()方法尽可能简单。 然而，TypeDescriptor 在获取有关源和目标类型的详细信息方面提供了很大的灵活性。
正如你可能已经猜到的，下一步是注册这个转换器:
```java
@Override
public void addFormatters(FormatterRegistry registry) {
    registry.addConverter(new StringToEmployeeConverter());
    registry.addConverterFactory(new StringToEnumConverterFactory());
    registry.addConverter(new GenericBigDecimalConverter());
}
```
使用这个转换器类似于我们已经看到的其他例子:
```java
@Test
public void whenConvertingToBigDecimalUsingGenericConverter_thenSuccess() {
    assertThat(conversionService
            .convert(Integer.valueOf(11), BigDecimal.class))
            .isEqualTo(BigDecimal.valueOf(11.00)
                    .setScale(2, BigDecimal.ROUND_HALF_EVEN));
    assertThat(conversionService
            .convert(Double.valueOf(25.23), BigDecimal.class))
            .isEqualByComparingTo(BigDecimal.valueOf(Double.valueOf(25.23)));
    assertThat(conversionService.convert("2.32", BigDecimal.class))
            .isEqualTo(BigDecimal.valueOf(2.32));
}
```

> 翻译自 [https://www.baeldung.com/spring-type-conversions](https://www.baeldung.com/spring-type-conversions)