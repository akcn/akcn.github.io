---
title: 策略模式
categories: 
  - design pattern
tags: 
  - strategy
---
我们定义了多个算法，并让客户端应用程序传递算法作为参数使用
策略模式的最佳示例之一是使用Collections.sort ()方法接收Comparator 参数。 根据 Comparator 接口的不同实现，对象以不同的方式进行排序。
举个例子，我们将尝试实现一个简单的购物车，我们有两个支付策略-使用信用卡或使用支付宝。
首先，我们将为策略模式示例创建一个接口，在我们的示例中支付作为参数传递的金额。
```java
package com.xym.design.strategy;

public interface PaymentStrategy {
    void pay(int amount);
}
````
现在，我们必须创建具体的实现算法，使用信用卡或通过支付宝支付。
```java
// 信用卡策略
@AllArgsConstructor
public class CreditCardStrategy implements PaymentStrategy {

    private String name;
    private String cardNumber;
    private String cvv;
    private String dateOfExpiry;

    @Override
    public void pay(int amount) {
        System.out.println(amount +" paid with credit/debit card");
    }
}
```
```java
package com.xym.design.strategy;
import lombok.Data;

// 支付宝策略
@Data
public class AlipayStrategy implements PaymentStrategy {

    private String emailId;
    private String password;

    @Override
    public void pay(int amount) {
        System.out.println(amount + " paid using Alipay.");
    }
}
```
现在我们的策略模式示例算法已经准备就绪。 我们可以实施购物车和支付方式将需要输入作为支付策略。
```java
package com.xym.design.strategy;
import lombok.AllArgsConstructor;

@AllArgsConstructor
public class Item {

    private String upcCode;
    private int price;

    public String getUpcCode() {
        return upcCode;
    }

    public int getPrice() {
        return price;
    }

}
```
```java
package com.xym.design.strategy;

import java.util.ArrayList;
import java.util.List;

public class ShoppingCart {

    //List of items
    List<Item> items;

    public ShoppingCart(){
        this.items=new ArrayList<Item>();
    }

    public void addItem(Item item){
        this.items.add(item);
    }

    public void removeItem(Item item){
        this.items.remove(item);
    }

    public int calculateTotal(){
        int sum = 0;
        for(Item item : items){
            sum += item.getPrice();
        }
        return sum;
    }

    public void pay(PaymentStrategy paymentMethod){
        int amount = calculateTotal();
        paymentMethod.pay(amount);
    }
}
```

注意，购物车的支付方式需要支付算法作为参数，并不存储它作为实例变量。
让我们用一个简单的程序来测试我们的策略模式示例。
```java
package com.xym.design.strategy;

public class ShoppingCartTest {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();

        Item item1 = new Item("1234",10);
        Item item2 = new Item("5678",40);

        cart.addItem(item1);
        cart.addItem(item2);

        //pay by paypal
        cart.pay(new AlipayStrategy("myemail@example.com", "mypwd"));

        //pay by credit card
        cart.pay(new CreditCardStrategy("Pankaj Kumar", "1234567890123456", "786", "12/15"));
    }
}
```
运行程序输出:
```bash
50 paid using Alipay.
50 paid with credit/debit card
```
