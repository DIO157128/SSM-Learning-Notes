# SSM学习笔记

## Day1

### xml中bean的配置

```xml
<bean id="bookDao" class="com.itheima.dao.impl.BookDaoImpl"/>
<bean id="bookService" class="com.itheima.service.impl.BookServiceImpl">
        <!--7.配置server与dao的关系-->
        <!--property标签表示配置当前bean的属性
        name属性表示配置哪一个具体的属性
        ref属性表示参照哪一个bean-->
        <property name="bookDao" ref="bookDao"/>
    </bean>
```

name是service中定义的那个bookDao属性，ref是上面定义的bookDao的bean

```xml
<!--scope：为bean设置作用范围，可选值为单例singleton，非单例prototype-->
    <bean id="bookDao" name="dao" class="com.itheima.dao.impl.BookDaoImpl" scope="prototype"/>
```

singleton是单例，创建出来的Dao对象都是同一个地址

### 为什么bean默认是单例？

spring容器适合管理那些可以复用的对象，比如Dao, ServiceImpl。耳封装实体数据的域对象不适合交给容器管理。

### 创建bean的三种方法

#### 实例化

```xml
<bean id="bookDao" class="com.itheima.dao.impl.BookDaoImpl"/>
```

spring创建bean调用的是无参的构造函数，是通过反射机制来调用的。

#### 静态工厂

```xml
<bean id="orderDao" class="com.itheima.factory.OrderDaoFactory" factory-method="getOrderDao"/>
```

要指定工厂方法

#### 实例工厂

```xml
<bean id="userFactory" class="com.itheima.factory.UserDaoFactory"/>

 <bean id="userDao" factory-method="getUserDao" factory-bean="userFactory"/>
```

先写实例工厂的bean，然后再写用哪个工厂的哪个工厂方法创建bean

但是这样很麻烦，所以可以直接创建FactoryBean

```java
public class UserDaoFactoryBean implements FactoryBean<UserDao> {
    //代替原始实例工厂中创建对象的方法
    public UserDao getObject() throws Exception {
        return new UserDaoImpl();
    }

    public Class<?> getObjectType() {
        return UserDao.class;
    }

}
```

```xml
<bean id="userDao" class="com.itheima.factory.UserDaoFactoryBean"/>
```

相当于原本由Dao自己创建变成工厂创建了

这样默认创建单例对象，如果创建非单例，需要覆盖方法

```java
public boolean isSingleton() {
    return false;
}
```

alt+insert生成override方法就行

### bean的生命周期

```xml
<bean id="bookDao" class="com.itheima.dao.impl.BookDaoImpl" init-method="init" destroy-method="destory"/>
```

在配置文件中指明初始方法和销毁方法。

或者直接让bean实现InitializingBean, DisposableBean这俩接口，对应方法如下

```java
public class BookServiceImpl implements BookService, InitializingBean, DisposableBean {
    private BookDao bookDao;

    public void setBookDao(BookDao bookDao) {
        System.out.println("set .....");
        this.bookDao = bookDao;
    }

    public void save() {
        System.out.println("book service save ...");
        bookDao.save();
    }

    public void destroy() throws Exception {
        System.out.println("service destroy");
    }

    public void afterPropertiesSet() throws Exception {
        System.out.println("service init");
    }
}
```

```java
public class AppForLifeCycle {
    public static void main( String[] args ) {
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
        BookDao bookDao = (BookDao) ctx.getBean("bookDao");
        bookDao.save();
    }
}
```

需要注意的是，在这样的main中，如果直接运行，是无法触发销毁函数的，因为IoC容器运行在java虚拟机中，执行到最后就退出了。如果需要调用销毁函数，需要显式地关闭容器。

```java
public class AppForLifeCycle {
    public static void main( String[] args ) {
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");

        BookDao bookDao = (BookDao) ctx.getBean("bookDao");
        bookDao.save();
        //注册关闭钩子函数，在虚拟机退出之前回调此函数，关闭容器
        //ctx.registerShutdownHook();
        //关闭容器
        ctx.close();
    }
}
```

close()这个方法在ApplicationContext接口中并没有定义（Ctrl+H查看）

![image-20241113153502581](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20241113153502581.png)

在AbstractApplicationContext这个抽象类中才有定义，因此，如果需要使用close()方法，ctx对象类型需要指定为ClassPathXmlApplicationContext

值得注意的是，close()方法比较暴力，可以使用钩子函数去关闭

```java
ctx.registerShutdownHook();
```

钩子函数地位置比较随意，目的是告诉虚拟机在退出之前关闭容器，耳close之后容器就不生效了

上述程序地运行结果为

```
init... //Dao对象初始化
set ..... //Service对象进行赋值
service init //Service对象进行初始化
book dao save ... //Dao对象调用方法
service destroy //关闭容器，销毁service对象
destory... //销毁Dao对象
```

事实上，在配置中指定了容器中有哪些bean之后，无论是否显式的获取bean，bean都会被初始化，容器关闭前bean会被销毁

### DI的四种方式

#### setter注入

分为引用类型（对象）和简单类型（int String等）

```xml
<bean id="bookDao" class="com.itheima.dao.impl.BookDaoImpl">
    <!--property标签：设置注入属性-->
    <!--name属性：设置注入的属性名，实际是set方法对应的名称-->
    <!--value属性：设置注入简单类型数据值-->
    <property name="connectionNum" value="100"/>
    <property name="databaseName" value="mysql"/>
</bean>

<bean id="userDao" class="com.itheima.dao.impl.UserDaoImpl"/>

<!--注入引用类型-->
<bean id="bookService" class="com.itheima.service.impl.BookServiceImpl">
    <!--property标签：设置注入属性-->
    <!--name属性：设置注入的属性名，实际是set方法对应的名称-->
    <!--ref属性：设置注入引用类型bean的id或name-->
    <property name="bookDao" ref="bookDao"/>
    <property name="userDao" ref="userDao"/>
</bean>
```

都需要在bean中定义类型属性并提供可访问的setter方法

#### 构造器注入

```Java
public class BookDaoImpl implements BookDao {
    private String databaseName;
    private int connectionNum;

    public BookDaoImpl(String databaseName, int connectionNum) {
        this.databaseName = databaseName;
        this.connectionNum = connectionNum;
    }

    public void save() {
        System.out.println("book dao save ..."+databaseName+","+connectionNum);
    }
}
```

```xml
<bean id="bookDao" class="com.itheima.dao.impl.BookDaoImpl">
    <!--根据构造方法参数位置注入-->
    <constructor-arg index="0" value="mysql"/>
    <constructor-arg index="1" value="100"/>
</bean>

```

```java
public class BookServiceImpl implements BookService{
    private BookDao bookDao;
    private UserDao userDao;

    public BookServiceImpl(BookDao bookDao, UserDao userDao) {
        this.bookDao = bookDao;
        this.userDao = userDao;
    }

    public void save() {
        System.out.println("book service save ...");
        bookDao.save();
        userDao.save();
    }
}
```

```xml
<bean id="bookService" class="com.itheima.service.impl.BookServiceImpl">
    <constructor-arg name="userDao" ref="userDao"/>
    <constructor-arg name="bookDao" ref="bookDao"/>
</bean>
```

把原本setter干的事交给constructor，同时在xml中注明

name是构造方法中形参的名字，ref是xml中上下文中bean的名字

#### 两种注入方式的选择

对于强制依赖，使用构造器注入，使用setter注入有概率不进行注入导致报空指针

可选注入使用setter注入，较为灵活

第三方框架大部分使用构造器注入

#### 自动装配

```xml
<bean id="bookService" class="com.itheima.service.impl.BookServiceImpl" autowire="byType"/>
```

按类型自动装配要求被装配的属性在上下文中必须是唯一的，比如上下文中不能有bookDao1,bookDao2

第二种是byName，要求带装配的属性名与上下文中的bean名称一样

自动装配只能用于引用类型DI，简单类型不可以。

自动装配优先级低于setter和构造器注入

#### 集合装配

```java
public class BookDaoImpl implements BookDao {

    private int[] array;

    private List<String> list;

    private Set<String> set;

    private Map<String,String> map;

    private Properties properties;

}
```

```xml
<bean id="bookDao" class="com.itheima.dao.impl.BookDaoImpl">
    <!--数组注入-->
    <property name="array">
        <array>
            <value>100</value>
            <value>200</value>
            <value>300</value>
        </array>
    </property>
    <!--list集合注入-->
    <property name="list">
        <list>
            <value>itcast</value>
            <value>itheima</value>
            <value>boxuegu</value>
            <value>chuanzhihui</value>
        </list>
    </property>
    <!--set集合注入-->
    <property name="set">
        <set>
            <value>itcast</value>
            <value>itheima</value>
            <value>boxuegu</value>
            <value>boxuegu</value>
        </set>
    </property>
    <!--map集合注入-->
    <property name="map">
        <map>
            <entry key="country" value="china"/>
            <entry key="province" value="henan"/>
            <entry key="city" value="kaifeng"/>
        </map>
    </property>
    <!--Properties注入-->
    <property name="properties">
        <props>
            <prop key="country">china</prop>
            <prop key="province">henan</prop>
            <prop key="city">kaifeng</prop>
        </props>
    </property>
</bean>
```