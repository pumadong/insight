# JDK动态代理+职责链设计模式

## JDK动态代理演示

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class MyProxy {
    /**
     * 一个接口
     */
    public interface HelloService{
        void sayHello();
    }
    /**
     * 目标类实现接口
     */
    static class HelloServiceImpl implements HelloService{

        @Override
        public void sayHello() {
            System.out.println("sayHello......");
        }
    }
    /**
     * 自定义代理类需要实现InvocationHandler接口
     */
    static  class TargetProxy implements InvocationHandler {
        /**
         * 目标对象
         */
        private Object target;

        public TargetProxy(Object target){
            this.target = target;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("------插入前置通知代码-------------");
            //执行相应的目标方法
            Object rs = method.invoke(target, args);
            System.out.println("------插入后置处理代码-------------");
            return rs;
        }

        public static Object wrap(Object target) {
            return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                    target.getClass().getInterfaces(),
                    new TargetProxy(target));
        }
    }
    public static void main(String[] args)  {
        HelloService proxyService = (HelloService) TargetProxy.wrap(new HelloServiceImpl());
        proxyService.sayHello();
    }
}
```

运行结果：

```
------插入前置通知代码-------------
sayHello......
------插入后置处理代码-------------
```

## 优化-引入Interceptor和Invocation

* 虽然，以上代码实现了代理的功能，但是在设计上有个明显缺陷，HWInvocationHandler是动态代理类，是个工具类，业务代码不应该写在invoke方法里，这不符合面向对象的思想。

  可以设计一个Interceptor接口，需要做什么拦截，实现这个接口即可。

* 同时，为了实现这个接口可以做前后拦截，我们设计一个Invocation类，来封装拦截目标对象的信息，作为拦截器的参数，把拦截目标对象真正执行的方法放到Interceptor中完成。

  这样，就可以实现前后拦截，同时可以根据需要，对拦截对象的参数做修改。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class MyProxy {
    /**
     * 一个接口
     */
    public interface HelloService {
        void sayHello();
    }

    /**
     * 目标类实现接口
     */
    static class HelloServiceImpl implements HelloService {
        @Override
        public void sayHello() {
            System.out.println("sayHello......");
        }
    }

    /**
     * 拦截目标对象
     */
    static class Invocation {
        /**
         * 目标对象
         */
        private Object target;
        /**
         * 执行的方法
         */
        private Method method;
        /**
         * 方法的参数
         */
        private Object[] args;

        public Invocation(Object target, Method method, Object[] args) {
            this.target = target;
            this.method = method;
            this.args = args;
        }

        /**
         * 执行目标对象的方法
         */
        public Object process() throws Exception {
            return method.invoke(target, args);
        }
    }

    public interface Interceptor {
        /**
         * 具体拦截处理
         */
        Object intercept(Invocation invocation) throws Exception;
    }

    static class TransactionInterceptor implements Interceptor {
        @Override
        public Object intercept(Invocation invocation) throws Exception {
            System.out.println("------插入前置通知代码-------------");
            Object result = invocation.process();
            System.out.println("------插入后置处理代码-------------");
            return result;
        }
    }

    static class TargetProxy implements InvocationHandler {
        private Object target;

        private Interceptor interceptor;

        public TargetProxy(Object target, Interceptor interceptor) {
            this.target = target;
            this.interceptor = interceptor;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Invocation invocation = new Invocation(target, method, args);
            return interceptor.intercept(invocation);
        }

        public static Object wrap(Object target, Interceptor interceptor) {
            TargetProxy targetProxy = new TargetProxy(target, interceptor);
            return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                    target.getClass().getInterfaces(),
                    targetProxy);
        }
    }

    public static void main(String[] args) {
        HelloService target = new HelloServiceImpl();
        Interceptor transactionInterceptor = new TransactionInterceptor();
        HelloService targetProxy = (HelloService) TargetProxy.wrap(target, transactionInterceptor);
        targetProxy.sayHello();
    }
}
```

运行结果：

```
------插入前置通知代码-------------
sayHello......
------插入后置处理代码-------------
```

## 优化-隐藏TargetProxy

对于目标类来说，只需要了解插入了什么拦截；不需要知道TargetProxy的存在。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class MyProxy {
    /**
     * 一个接口
     */
    public interface HelloService {
        void sayHello();
    }

    /**
     * 目标类实现接口
     */
    static class HelloServiceImpl implements HelloService {
        @Override
        public void sayHello() {
            System.out.println("sayHello......");
        }
    }

    /**
     * 拦截目标对象
     */
    static class Invocation {
        /**
         * 目标对象
         */
        private Object target;
        /**
         * 执行的方法
         */
        private Method method;
        /**
         * 方法的参数
         */
        private Object[] args;

        public Invocation(Object target, Method method, Object[] args) {
            this.target = target;
            this.method = method;
            this.args = args;
        }

        /**
         * 执行目标对象的方法
         */
        public Object process() throws Exception {
            return method.invoke(target, args);
        }
    }

    public interface Interceptor {
        /**
         * 具体拦截处理
         */
        Object intercept(Invocation invocation) throws Exception;

        /**
         *  插入目标类
         */
        Object plugin(Object target);
    }

    static class TransactionInterceptor implements Interceptor {
        @Override
        public Object intercept(Invocation invocation) throws Exception {
            System.out.println("------插入前置通知代码-------------");
            Object result = invocation.process();
            System.out.println("------插入后置处理代码-------------");
            return result;
        }

        @Override
        public Object plugin(Object target) {
            return TargetProxy.wrap(target, this);
        }
    }

    static class TargetProxy implements InvocationHandler {
        private Object target;

        private Interceptor interceptor;

        public TargetProxy(Object target, Interceptor interceptor) {
            this.target = target;
            this.interceptor = interceptor;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Invocation invocation = new Invocation(target, method, args);
            return interceptor.intercept(invocation);
        }

        public static Object wrap(Object target, Interceptor interceptor) {
            TargetProxy targetProxy = new TargetProxy(target, interceptor);
            return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                    target.getClass().getInterfaces(),
                    targetProxy);
        }
    }

    public static void main(String[] args) {
        HelloService target = new HelloServiceImpl();
        Interceptor transactionInterceptor = new TransactionInterceptor();
        // 把事务拦截器插入到目标类中
        HelloService targetProxy = (HelloService) transactionInterceptor.plugin(target);
        targetProxy.sayHello();
    }
}
```

运行结果：

```
------插入前置通知代码-------------
sayHello......
------插入后置处理代码-------------
```

## 优化-引入责任链设计模式

对于多个拦截器的场景，利用面向对象的设计思想再封装一下，引入职责链设计模式。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class MyProxy {
    /**
     * 一个接口
     */
    public interface HelloService {
        void sayHello();
    }

    /**
     * 目标类实现接口
     */
    static class HelloServiceImpl implements HelloService {
        @Override
        public void sayHello() {
            System.out.println("sayHello......");
        }
    }

    /**
     * 拦截目标对象
     */
    static class Invocation {
        /**
         * 目标对象
         */
        private Object target;
        /**
         * 执行的方法
         */
        private Method method;
        /**
         * 方法的参数
         */
        private Object[] args;

        public Invocation(Object target, Method method, Object[] args) {
            this.target = target;
            this.method = method;
            this.args = args;
        }

        /**
         * 执行目标对象的方法
         */
        public Object process() throws Exception {
            return method.invoke(target, args);
        }
    }

    public interface Interceptor {
        /**
         * 具体拦截处理
         */
        Object intercept(Invocation invocation) throws Exception;

        /**
         *  插入目标类
         */
        Object plugin(Object target);
    }

    static class TransactionInterceptor implements Interceptor {
        @Override
        public Object intercept(Invocation invocation) throws Exception {
            System.out.println("------插入前置通知代码-------------");
            Object result = invocation.process();
            System.out.println("------插入后置处理代码-------------");
            return result;
        }

        @Override
        public Object plugin(Object target) {
            return TargetProxy.wrap(target, this);
        }
    }

    static class TargetProxy implements InvocationHandler {
        private Object target;

        private Interceptor interceptor;

        public TargetProxy(Object target, Interceptor interceptor) {
            this.target = target;
            this.interceptor = interceptor;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Invocation invocation = new Invocation(target, method, args);
            return interceptor.intercept(invocation);
        }

        public static Object wrap(Object target, Interceptor interceptor) {
            TargetProxy targetProxy = new TargetProxy(target, interceptor);
            return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                    target.getClass().getInterfaces(),
                    targetProxy);
        }
    }

    static class InterceptorChain {

        private List<Interceptor> interceptorList = new ArrayList<>();

        /**
         * 插入所有拦截器
         */
        public Object pluginAll(Object target) {
            for (Interceptor interceptor : interceptorList) {
                target = interceptor.plugin(target);
            }
            return target;
        }

        public void addInterceptor(Interceptor interceptor) {
            interceptorList.add(interceptor);
        }
        /**
         * 返回一个不可修改集合，只能通过addInterceptor方法添加
         * 这样控制权就在自己手里
         */
        public List<Interceptor> getInterceptorList() {
            return Collections.unmodifiableList(interceptorList);
        }
    }

    public static void main(String[] args) {
        HelloService target = new HelloServiceImpl();
        Interceptor transactionInterceptor = new TransactionInterceptor();
        InterceptorChain interceptorChain = new InterceptorChain();
        interceptorChain.addInterceptor(transactionInterceptor);
        HelloService targetProxy = (HelloService) interceptorChain.pluginAll(target);
        targetProxy.sayHello();
    }
}
```

以上，展示的是“JDK动态代理+责任链设计模式”，Mybatis拦截器就是基于此开发的。

# Mybatis Plugin 插件概念

## 原理

Mybatis的拦截器实现机制，和上面的最后优化代码非常相似。

它也有个代理类Plugin(类似上面的TargetProxy)，同样实现InvocationHandler接口。

当我们调用四大接口（ParameterHandler、ResultSetHandler、StatementHandler、Executor）的对象时，会执行Plugin的invoke方法：

```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
```

以上，达到了拦截目标方法的结果。

以Executor为例，执行的流程：

* Executor.方法 -> Plugin.invoke -> Interceptor.intercept -> Invocation.proceed -> Executor.方法.invoke

## 如何自定义拦截器

### Interceptor接口

```java
package org.apache.ibatis.plugin;

import java.util.Properties;

/**
 * @author Clinton Begin
 */
public interface Interceptor {
	// Executor.方法 -> Plugin.invoke -> Interceptor.intercept -> Invocation.proceed -> Executor.方法.invoke
  // 执行拦截
	Object intercept(Invocation invocation) throws Throwable;
	// 输入目标对象，返回代理对象
  Object plugin(Object target);
	// 在Mybatis配置文件中指定一些属性
  void setProperties(Properties properties);

}
```

### 自定义拦截器

ExamplePlugIn和上面的TransactionInterceptor性质是一样的。

```java
@Intercepts({@Signature( type= Executor.class,  method = "update", args ={MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
	public Object intercept(Invocation invocation) throws Throwable {
    System.out.println("------插入前置通知代码-------------");
    Object result = invocation.process();
    System.out.println("------插入后置处理代码-------------");
    return result;
  }
  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }
  public void setProperties(Properties properties) {
  }
}
```

### mybatis-config.xml

```xml
<plugins>
	<plugin interceptor="xxx">
	<!-- * 表示对全部mapper做隔离 -->
	<property name="mappers" value="XXMapper"/>
	<property name="mode" value="auto"/>
</plugin>
```

至此，自定义MyBatis插件流程大致就是这样了。

## Mybatis四大接口

* Executor：Mybatis的执行器，用于执行CRUD操作。
* ParameterHandler：处理SQL的参数对象。
* ResultSetHandler：对象SQL的返回结果。
* StatementHandler：负责处理Mybatis与JDBC之间Statement的交互。

**下图描述Mybatis框架的执行过程**：

  ```mermaid
graph TD
session[SqlSession 对外接口]==>executor[Executor 内部执行器]==>statement[StatementHandler JDBC封装层] ==>db((数据库))
param["ParameterHandler 参数处理层"]==>statement
db ==> result[ResultSetHandler 返回结果集]
  ```

# Mybatis Plugin 插件源码

## Plugin类清单

* Interceptor：拦截器接口。
* InterceptorChain：拦截器职责链。
* Intercepts：注解，和Signature一起，根据目标对象的type/method/args判断是否需要拦截。
* Invocation：封装拦截目标对象。
* Plugin：代理类，实现了InvocationHandler，就是我们上面实现的TargetProxy。
* PluginException：自定义异常类。
* Signature：注解，type/method/args

这些代码和我们上面实现的非常相似，有兴趣直接找源码看就可以了。

## Configuration

Mybatis是通过初始化配置文件把所有的拦截器添加到拦截器链中。

```java
  public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }

  public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
      ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }

  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }

  public Executor newExecutor(Transaction transaction) {
    return newExecutor(transaction, defaultExecutorType);
  }

  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

# 原文地址

https://www.cnblogs.com/qdhxhz/p/11390778.html

# 致谢

**作者有这样一句话，蛮奋斗的：**

 ```
我相信，无论今后的道路多么坎坷，只要抓住今天，迟早会在奋斗中尝到人生的甘甜。抓住人生中的一分一秒，胜过虚度中的一月一年！
 ```

这篇博客，增强对JDK动态代理和责任链模式的理解，在代码设计上也很有启发。:thumbsup:

