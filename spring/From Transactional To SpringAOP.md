# Transactional注解

## Transactional不生效场景

### 非public方法

@Transactional是基于动态代理实现的，在bean初始化过程中，对含有@Transactional标注的bean实例创建代理对象，这里就存在一个spring扫描@Transactional注解信息的过程。

标注@Transactional的方法如果修饰符不是public，那么就默认方法的@Transactional信息为空，那么将不会对bean进行代理对象创建或者不会对方法进行代理调用。

### 内部方法互相调用时事务不生效

```java
@Service
public class Demo {
	@Transactional
  public void A(){
    //doSomething
  }
  
  public void B(){
    //doSomething
    A();
  }
}
```

### try-catch的坑: 当doSomething抛异常时事务没生效

```java
@Transactional
public void A(){
  try{
    //doSomething
  }catch(Exception e){
    log.error("",e);
  }
}
```

**解决办法：**

不要把异常自己吃掉了，必须要抛到方法外面去，就是把try-catch去掉好了。

### 异常正常抛出来了，但是还是事务没生效

```java
@Service
public class Demo {
	@Transactional
  public void A() throws FileNotFoundException{
    //doSomething
    //下面文件不存在抛出异常 FileNotFoundException
    FileInputStream fis = new FileInputStream("D://a.txt");
  }
}
```

**解决办法：**

Spring事务默认只回滚RuntimeException和Error，所以写明你想要回滚的异常即可，如解决办法：@Transactional(rollbackFor = XXException.class)，都回滚的话可以写@Transactional(rollbackFor = Throwable.class)。

### 多个数据源的事务管理

```java
@Service
public class Demo {
  
  @Autowired
  private UserDao userDao; //操作数据库A
  
  @Autowired
  private OrderDao orderDao;//操作数据库B
  
	@Transactional
  public void A(){
    //doSomething
    userDao.insert(user);
  }
  
  @Transactional
  public void A(){
    //doSomething
    orderDao.insert(order);
  }
}
```

**解决办法：**

多个数据源需要事务管理需要声明多个事务管理器，并且在@Transactional注明使用哪个事务管理器 @Transactional(transactionManager = "transaction_Manager")。

# Spring多数据源多事务

## 数据源配置

### 第一步：配置两个数据源

```xml
<bean id="productlibDataSource" class="com.dianping.zebra.group.jdbc.GroupDataSource" init-method="init" destroy-method="close">
 ...
</bean>

<bean id="productmartDataSource" class="com.dianping.zebra.group.jdbc.GroupDataSource" init-method="init" destroy-method="close">
 ...
</bean>
```

### 第二步：配置两个工厂指向不同的数据源

```xml
<bean id="productlibSqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!--dataource-->
        <property name="dataSource" ref="productlibDataSource" />
  			...
  </bean>
  

<bean id="productmartSqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!--dataource-->
        <property name="dataSource" ref="productmartDataSource"/>
        ...
 </bean>
```

### 第三步：配置对应的mapper扫描

```xml
<bean class="com.dianping.zebra.dao.mybatis.ZebraMapperScannerConfigurer">
		<!--这里改成实际dao目录,如果有多个，可以用,;\t\n进行分割-->
		<property name="sqlSessionFactoryBeanName" value="productlibSqlSessionFactory"/>
    <property name="basePackage" value="com.sankuai.meituan.waimai.product.platformmanage.dao.mapper.productlib" />
</bean>


<bean class="com.dianping.zebra.dao.mybatis.ZebraMapperScannerConfigurer">
		<!--这里改成实际dao目录,如果有多个，可以用,;\t\n进行分割-->
		<property name="sqlSessionFactoryBeanName" value="productmartSqlSessionFactory"/>
		<property name="basePackage" value="com.sankuai.meituan.waimai.product.platformmanage.dao.mapper.statistical" />
</bean>
```

## 事务配置

### 第一步：添加两个事务，分别指向不同的数据源

```xml
<!-- 开启事务控制的注解支持 -->
<tx:annotation-driven/>

<bean id="mybatisTransactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager" primary="true">
		<property name="dataSource" ref="productlibDataSource" />
</bean>

<bean id="productmartTransactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="productmartDataSource"/>
</bean>
```

注意：要选择其中一个TransactionManager的**primary设置为true**，否则会报错：

> Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'org.springframework.jdbc.datasource.DataSourceTransactionManager' available: expected single matching bean but found 2

### 第二步：在方法上添加注解就可以了

```java
@Override
@Transactional("productmartTransactionManager")
public void tranUpdateData(String dt, Long infoId, List<StatisticalDataDto> dataDtos) {
			....
}
```

注意：@Transactional里面的value是DataSourceTransactionManager这个bean的名字。

# Spring的AOP原理

事务自调用失效和public字段不能直接访问：https://blog.csdn.net/leisurelen/article/details/108102586

