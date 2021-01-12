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
public class MyProxy2 {
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

public class MyProxy3 {
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

public class MyProxy4 {
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

## 如何自定义拦截器

## Mybatis四大接口



# Mybatis Plugin 插件源码

## 拦截器链InterceptorChain

## Configuration

## Plugin

## Interceptor接口

# 原文地址

https://www.cnblogs.com/qdhxhz/p/11390778.html