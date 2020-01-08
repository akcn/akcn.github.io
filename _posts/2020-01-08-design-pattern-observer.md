---
title: 观察者模式与事件监听机制
categories: 
  - design pattern
tags: 
  - observer
---

## 观察者设计模式
定义: 在对象之间定义了一对多的依赖，这样一来，当一个对象改变状态，依赖它的对象会收到通知并自动更新。
其实就是发布订阅模式，发布者发布信息，订阅者获取信息，订阅了就能收到信息，没订阅就收不到信息。
该模式包含四个角色:
- **抽象被观察者角色:** 也就是一个抽象主题，它把所有对观察者对象的引用保存在一个集合中，每个主题都可以有任意数量的观察者。抽象主题提供一个接口，可以增加和删除观察者角色。一般用一个抽象类和接口来实现。
- **抽象观察者角色:** 为所有的具体观察者定义一个接口，在得到主题通知时更新自己。
- **具体被观察者角色:** 也就是一个具体的主题，在集合主题的内部状态改变时，所有登记过的观察者发出通知。
- **具体观察者角色:** 实现抽象观察者角色所需要的更新接口，一边使本身的状态与制图的状态相协调。
下面是一个例子，有一个微信公众号服务，不定时发布一些消息，关注公众号就可以收到推送消息，取消关注就收不到推送消息。
```java
// 抽象被观察者接口 声明了添加、删除、通知观察者方法
public interface Observerable {

    void registerObserver(Observer o);

    void removeObserver(Observer o);

    void notifyObserver();
}
```
```java
// 抽象观察者 定义了一个update()方法，当被观察者调用notifyObservers()方法时，观察者的update()方法会被回调。
public interface Observer {

    void update(String message);
}
```
```java
import java.util.ArrayList;
import java.util.List;

// 被观察者，也就是微信公众号服务
public class WechatServer implements Observerable {

	// 注意到这个List集合的泛型参数为Observer接口，设计原则：面向接口编程而不是面向实现编程
    private List<Observer> observers;
    private String message;

    public WechatServer(){
        observers = new ArrayList<>();
    }

    @Override
    public void registerObserver(Observer o) {
        observers.add(o);
    }

    @Override
    public void removeObserver(Observer o) {
        if (!observers.isEmpty()) {
            observers.remove(o);
        }
    }

    @Override
    public void notifyObserver() {
        observers.forEach(o -> o.update(message));
    }

    public void setInfomation(String s) {
        this.message = s;
        System.out.println("微信服务更新消息： " + s);
        notifyObserver();
    }
}
```
```java
// 观察者 实现了update方法
public class User implements Observer {

    private String name;
    private String message;

    public User(String name) {
        this.name = name;
    }

    @Override
    public void update(String message) {
        this.message = message;
        read();
    }

    public void read(){
        System.out.println(name + " 收到推送消息： " + message);
    }
}
```
```java
// 测试类 用户ZhangSan看到消息后颇为震惊，果断取消订阅，这时公众号又推送了一条消息，此时用户ZhangSan已经收不到消息，其他用户还是正常能收到推送消息。
public class TestObserver {
    public static void main(String[] args) {
        WechatServer wechatServer = new WechatServer();
        Observer userZhang = new User("ZhangSan");
        Observer userLi = new User("LiSi");
        Observer userWang = new User("WangWu");
        wechatServer.registerObserver(userZhang);
        wechatServer.registerObserver(userLi);
        wechatServer.registerObserver(userWang);
        wechatServer.setInfomation("PHP是世界上最好用的语言！");
        System.out.println("===================");
        wechatServer.removeObserver(userZhang);
        wechatServer.setInfomation("JAVA是世界上最好用的语言！");
    }
}
```
```text
微信服务更新消息： PHP是世界上最好用的语言！
ZhangSan 收到推送消息： PHP是世界上最好用的语言！
LiSi 收到推送消息： PHP是世界上最好用的语言！
WangWu 收到推送消息： PHP是世界上最好用的语言！
===================
微信服务更新消息： JAVA是世界上最好用的语言！
LiSi 收到推送消息： JAVA是世界上最好用的语言！
WangWu 收到推送消息： JAVA是世界上最好用的语言！
```
### 小结
- 这个模式是松偶合的。改变主题或观察者中的一方，另一方不会受到影像。
- JDK中也有自带的观察者模式。但是被观察者是一个类而不是接口，限制了它的复用能力。
- 在JavaBean和Swing中也可以看到观察者模式的影子。

## 事件监听机制

首先设计一个可以被别的对象监听的对象
事件处理模型涉及到三个组件：事件源、事件对象、事件监听器。其中事件对象用来包装事件源
下面是一个监听 对象方法的例子
```java
// 被监听的对象
public class Person {

    // 事件监听器 若多个用list替换
    private PersonListener listener;

    // 注册监听器
    void register(PersonListener listener){
        this.listener = listener;
    }

	// 被监听的方法
    void eat(Event e){
        if (listener != null) {
            listener.doEat(e);
        }
    }

    // 被监听的方法
    void run(Event e){
        if (listener != null) {
            listener.doRun(e);
        }
    }
}
```
```java
// Person对象的事件监听器
public interface PersonListener {

    void doEat(Event e);

    void doRun(Event e);
}
```
```java
// 事件对象 用来封装事件源
public class Event {

    private Object source;

    public Event(Object source) {
        this.source = source;
    }

    public Object getSource() {
        return source;
    }

    public void setSource(Object source) {
        this.source = source;
    }
}
```
```java
// 测试程序
public class Test {
    public static void main(String[] args) {
        Person person = new Person();
        // 被监听对象上注册监听器
        person.register(new PersonListener() {
            @Override
            public void doEat(Event e) {
                Person person = (Person) e.getSource();
                System.out.println(person + " is eating");
            }

            @Override
            public void doRun(Event e) {
                Person person = (Person) e.getSource();
                System.out.println(person + " is running");
            }
        });
        // 触发eat事件时 将当前上下文环境中的person对象作为事件源包装成事件对象传入方法
        person.eat(new Event(person));
        person.run(new Event(person));
    }
}
```
### 什么是事件监听机制
个人理解: 当程序执行到某个阶段(时间点)，需要额外增加一些其它的操作(代码)，此时可以提前定义一个事件发布器，并持有(注册)监听器，在执行到此阶段(时间点)将当前的某个上下文变量(封装成事件对象)传入发布器，发布器内部检索出事件对应的监听器，遍历监听器并将事件对象作为参数传入执行，发布器，事件，事件监听器三者解耦
在讲解事件监听机制前,我们先回顾下设计模式中的观察者模式,因为事件监听机制可以说是在典型观察者模式基础上的进一步抽象和改进。我们可以在JDK或者各种开源框架比如Spring中看到它的身影,从这个意义上说,事件监听机制也可以看做一种对传统观察者模式的具体实现,不同的框架对其实现方式会有些许差别。

典型的观察者模式将有依赖关系的对象抽象为了观察者和主题两个不同的角色,多个观察者同时观察一个主题,两者只通过抽象接口保持松耦合状态,这样双方可以相对独立的进行扩展和变化:比如可以很方便的增删观察者,修改观察者中的更新逻辑而不用修改主题中的代码。但是这种解耦进行的并不彻底,这具体体现在以下几个方面:
1. 抽象主题需要依赖抽象观察者,而这种依赖关系完全可以去除。
2. 主题需要维护观察者列表,并对外提供动态增删观察者的接口。
3. 主题状态改变时需要由自己去通知观察者进行更新。

我们可以把主题(Subject)替换成事件(event),把对特定主题进行观察的观察者(Observer)替换成对特定事件进行监听的监听器(EventListener),而把原有主题中负责维护主题与观察者映射关系以及在自身状态改变时通知观察者的职责从中抽出,放入一个新的角色事件发布器(EventPublisher)中,事件监听模式的轮廓就展现在了我们眼前,
常见事件监听机制的主要角色如下
- 事件及事件源:对应于观察者模式中的主题。事件源发生某事件是特定事件监听器被触发的原因。
- 事件监听器:对应于观察者模式中的观察者。监听器监听特定事件,并在内部定义了事件发生后的响应逻辑。
- 事件发布器:事件监听器的容器,对外提供发布事件和增删事件监听器的接口,维护事件和事件监听器之间的映射关系,并在事件发生时负责通知相关监听器。
Spring框架对事件的发布与监听提供了相对完整的支持,它扩展了JDK中对自定义事件监听提供的基础框架,并与Spring的IOC特性作了整合,使得用户可以根据自己的业务特点进行相关的自定义,并依托Spring容器方便的实现监听器的注册和事件的发布。因为Spring的事件监听依托于JDK提供的底层支持,为了更好的理解,先来看下JDK中为用户实现自定义事件监听提供的基础框架。

### JDK中对事件监听机制的支持
JDK为用户实现自定义事件监听提供了两个基础的类。一个是代表所有可被监听事件的事件基类java.util.EventObject,所有自定义事件类型都必须继承该类,
该类内部有一个Object类型的source变量,逻辑上表示发生该事件的事件源,实际中可以用来存储包含该事件的一些相关信息。类结构如下所示
```java
public class EventObject implements java.io.Serializable {

    private static final long serialVersionUID = 5516075349620653480L;

    /**
     * The object on which the Event initially occurred.
     */
    protected transient Object  source;

    /**
     * Constructs a prototypical Event.
     *
     * @param    source    The object on which the Event initially occurred.
     * @exception  IllegalArgumentException  if source is null.
     */
    public EventObject(Object source) {
        if (source == null)
            throw new IllegalArgumentException("null source");

        this.source = source;
    }

    /**
     * The object on which the Event initially occurred.
     *
     * @return   The object on which the Event initially occurred.
     */
    public Object getSource() {
        return source;
    }

    /**
     * Returns a String representation of this EventObject.
     *
     * @return  A a String representation of this EventObject.
     */
    public String toString() {
        return getClass().getName() + "[source=" + source + "]";
    }
}
```
另一个则是对所有事件监听器进行抽象的接口java.util.EventListener,这是一个标记接口,内部没有任何抽象方法,所有自定义事件监听器都必须实现该标记接口
```java
/**
 * A tagging interface that all event listener interfaces must extend.
 * @since JDK1.1
 */
public interface EventListener {
}
```
以上就是JDK为我们实现自定义事件监听提供的底层支持。针对具体业务场景,我们通过扩展java.util.EventObject来自定义事件类型,同时通过扩展java.util.EventListener来定义在特定事件发生时被触发的事件监听器。当然,不要忘了还要定义一个事件发布器来管理事件监听器并提供发布事件的功能。

基于JDK实现对任务执行结果的监听

想象我们正在做一个关于Spark的任务调度系统,我们需要把任务提交到集群中并监控任务的执行状态,当任务执行完毕(成功或者失败),除了必须对数据库进行更新外,还可以执行一些额外的工作:比如将任务执行结果以邮件的形式发送给用户。这些额外的工作后期还有较大的变动可能:比如还需要以短信的形式通知用户,对于特定的失败任务需要通知相关运维人员进行排查等等,所以不宜直接写死在主流程代码中。最好的方式自然是以事件监听的方式动态的增删对于任务执行结果的处理逻辑。为此我们可以基于JDK提供的事件框架,打造一个能够对任务执行结果进行监听的弹性系统。


```java
// 任务结束事件的事件源，任务包含任务名和任务状态
@Data
public class Task {

    private String name;

    private TaskFinishStatus status;

}
```
```java
// 任务状态是个枚举常量,只有成功和失败两种取值。
public enum  TaskFinishStatus {
    SUCCEDD,
    FAIL;
}
```
```java
// 任务结束事件 自定义事件类型TaskFinishEvent继承自JDK中的EventObject,构造时会传入Task作为事件源。
public class TaskFinishEvent extends EventObject {
    /**
     * Constructs a prototypical Event.
     *
     * @param source The object on which the Event initially occurred.
     * @throws IllegalArgumentException if source is null.
     */
    public TaskFinishEvent(Object source) {
        super(source);
    }
}
```
```java
// 该事件的监听器抽象 继承标记接口EventListner表示该接口的实现类是一个监听器,同时在内部定义了事件发生时的响应方法onTaskFinish(event),接收一个TaskFinishEvent作为参数。
public interface TaskFinishEventListner extends EventListener {
    
    void onTaskFinish(TaskFinishEvent event);
}
```
```java
// 邮件服务监听器 该邮件服务监听器将在监听到任务结束事件时将任务的执行结果发送给用户。
@Data
public class MailTaskFinishListener implements TaskFinishEventListner {

    private String emial;

    @Override
    public void onTaskFinish(TaskFinishEvent event) {
        System.out.println("Send Emial to "+emial+" Task:"+event.getSource());
    }
}
```
```java
// 自定义事件发布器
public class TaskFinishEventPublisher {
    
    private List<TaskFinishEventListner> listners=new ArrayList<>();
    
    //注册监听器
    public synchronized void register(TaskFinishEventListner listner){
        if(!listners.contains(listner)){
            listners.add(listner);
        }
    }
    
    //移除监听器
    public synchronized boolean remove(TaskFinishEventListner listner){
        return listners.remove(listner);
    }
    
    
    //发布任务结束事件
    public void publishEvent(TaskFinishEvent event){
        
        for(TaskFinishEventListner listner:listners){
            listner.onTaskFinish(event);
        }
    }
}
```
```java
// 测试代码
public class TestTaskFinishListener {


    public static void main(String[] args) {

        //事件源
        Task source = new Task("用户统计", TaskFinishStatus.SUCCEDD);

        //任务结束事件
        TaskFinishEvent event = new TaskFinishEvent(source);

        //邮件服务监听器
        MailTaskFinishListener mailListener = new MailTaskFinishListener("test@163.com");

        //事件发布器
        TaskFinishEventPublisher publisher = new TaskFinishEventPublisher();

        //注册邮件服务监听器
        publisher.register(mailListener);

        //发布事件
        publisher.publishEvent(event);

    }
}
```
如果后期因为需求变动需要在任务结束时将结果以短信的方式发送给用户,则可以再添加一个短信服务监听器
```java
@Data
@AllArgsConstructor
public class SmsTaskFinishListener implements TaskFinishEventListner {

    private String address;

    @Override
    public void onTaskFinish(TaskFinishEvent event) {
        System.out.println("Send Message to "+address+" Task:"+event.getSource());
    }
}
```
在测试代码中添加如下代码向事件发布器注册该监听器
```java
SmsTaskFinishListener smsListener = new SmsTaskFinishListener("123456789");

//注册短信服务监听器
publisher.register(smsListener);
```

基于JDK的支持要实现对自定义事件的监听还是比较麻烦的,要做的工作比较多。而且自定义的事件发布器也不能提供对所有事件的统一发布支持。基于Spring框架实现自定义事件监听则要简单很多,功能也更加强大。

### Spring容器对事件监听机制的支持
Spring容器,具体而言是ApplicationContext接口定义的容器提供了一套相对完善的事件发布和监听框架,其遵循了JDK中的事件监听标准,并使用容器来管理相关组件,使得用户不用关心事件发布和监听的具体细节,降低了开发难度也简化了开发流程。下面看看对于事件监听机制中的各主要角色,Spring框架中是如何定义的,以及相关的类体系结构
- 事件 Spring为容器内事件定义了一个抽象类ApplicationEvent,该类继承了JDK中的事件基类EventObject。因而自定义容器内事件除了需要继承ApplicationEvent之外,还要传入事件源作为构造参数。
- 事件监听器 Spring定义了一个ApplicationListener接口作为为事件监听器的抽象,接口定义如下
```java
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

   /**
    * Handle an application event.
    * @param event the event to respond to
    */
   void onApplicationEvent(E event);

}
```
1. 该接口继承了JDK中表示事件监听器的标记接口EventListener,内部只定义了一个抽象方法onApplicationEvent(event),当监听的事件在容器中被发布,该方法将被调用。
2. 同时,该接口是一个泛型接口,其实现类可以通过传入泛型参数指定该事件监听器要对哪些事件进行监听。这样有什么好处？这样所有的事件监听器就可以由一个事件发布器进行管理,并对所有事件进行统一发布,而具体的事件和事件监听器之间的映射关系,则可以通过反射读取泛型参数类型的方式进行匹配,稍后我们会对原理进行讲解。
3. 最后,所有的事件监听器都必须向容器注册,容器能够对其进行识别并委托容器内真正的事件发布器进行管理。

- 事件发布器 ApplicationContext接口继承了ApplicationEventPublisher接口,从而提供了对外发布事件的能力
那么是否可以说ApplicationContext,即容器本身就担当了事件发布器的角色呢？其实这是不准确的,容器本身仅仅是对外提供了事件发布的接口,真正的工作其实是委托给了具体容器内部一个`ApplicationEventMulticaster`对象,其定义在AbstractApplicationContext抽象类内部,如下所示
```java
/** Helper class used in event publishing */
private ApplicationEventMulticaster applicationEventMulticaster;
```
所以,真正的事件发布器是ApplicationEventMulticaster,这是一个接口,定义了事件发布器需要具备的基本功能:管理事件监听器以及发布事件。其默认实现类是
SimpleApplicationEventMulticaster,该组件会在容器启动时被自动创建,并以单例的形式存在,管理了所有的事件监听器,并提供针对所有容器内事件的发布功能。

### 基于Spring实现对任务执行结果的监听
基于Spring框架来实现对自定义事件的监听流程十分简单,只需要三部:1.自定义事件类 2.自定义事件监听器并向容器注册 3.发布事件
1. 自定任务结束事件
```java
// 定义一个任务结束事件TaskFinishEvent2,该类继承抽象类ApplicationEvent来遵循容器事件规范。
public class TaskFinishEvent2 extends ApplicationEvent {
    /**
     * Create a new ApplicationEvent.
     *
     * @param source the object on which the event initially occurred (never {@code null})
     */
    public TaskFinishEvent2(Object source) {
        super(source);
    }
}
```
2. 自定义邮件服务监听器并向容器注册
```java
// 该类实现了容器事件规范定义的监听器接口,通过泛型参数指定对上面定义的任务结束事件进行监听,通过@Component注解向容器进行注册
@Component
public class MailTaskFinishListener2 implements ApplicationListener<TaskFinishEvent2> {

    private String emial="test@163.com";
    
    @Override
    public void onApplicationEvent(TaskFinishEvent2 event) {
        
        System.out.println("Send Emial to "+emial+" Task:"+event.getSource());
        
    }
}
```
3. 发布事件
从上面对Spring事件监听机制的类结构分析可知,发布事件的功能定义在ApplicationEventPublisher接口中,而ApplicationContext继承了该接口,所以最好的方法是通过实现ApplicationContextAware接口获取ApplicationContext实例,然后调用其发布事件方法。如下所示定义了一个发布容器事件的代理类
```java
@Component
public class EventPublisher implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    //发布事件
    public void publishEvent(ApplicationEvent event){

        applicationContext.publishEvent(event);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext=applicationContext;
    }
}
```
在此基础上,还可以自定义一个短信服务监听器,在任务执行结束时发送短信通知用户。过程和上面自定义邮件服务监听器类似:实现ApplicationListner接口并重写抽象方法,然后通过注解或者xml的方式向容器注册。
下一篇分析其实现源码