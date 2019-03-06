#### JDK动态代理和CGLIB字节码生成的区别？

1）JDK动态代理只能对实现了接口的类生成代理，而不能针对类。

2）CGLIB是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法，

并覆盖其中方法实现增强，但是因为采用的是继承，所以该类或方法最好不要声明成final，

对于final类或方法，是无法继承的。

##### JDK



``` java
package com.bsx.test.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

	/**

    - 接口动态代理需要2步
    - 第一步：定义额外的操作
    - 通过实现 InvocationHandler 接口，来定义在执行代理对象方法前后自己的动作。
    - 第二步：获取代理对象
    - 通过 Proxy.newProxyInstance 获取代理对象，这一步的作用是根据指定的1.classLoader，
        2.要代理的接口，以及3.传递进来的处理者。来生成真正的代理对象。
    */
    public class InterfaceProxy implements InvocationHandler {
    // 获取代理对象
    public <T> T getProxy() {
        return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }
    private Object target;
      public InterfaceProxy(Object target) {
        this.target = target;
    }
      @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object object = method.invoke(target, args);
        after();
        return object;
    }
  
}
```



##### CGLIB

---------------------
```java
package com.lanhuigu.spring.proxy.compare;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**

- CGLibProxy动态代理类
*/
public class CGLibProxy implements MethodInterceptor {
/** CGLib需要代理的目标对象 */
  private Object targetObject;
  public Object createProxyObject(Object obj) {
    this.targetObject = obj;
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(obj.getClass());
    enhancer.setCallback(this);
    Object proxyObj = enhancer.create();
    // 返回代理对象
    return proxyObj;
}
  @Override
  public Object intercept(Object proxy, Method method, Object[] args,
                        MethodProxy methodProxy) throws Throwable {
    Object obj = null;
    before();
    obj = method.invoke(targetObject, args);
    after();
    return obj;
}
  private void checkPopedom() {
    System.out.println("======检查权限checkPopedom()======");
}
```



#### JDK和CGLIB动态代理总结

JDK代理是不需要第三方库支持，只需要JDK环境就可以进行代理，使用条件:

1）实现InvocationHandler 

2）使用Proxy.newProxyInstance产生代理对象

3）被代理的对象必须要实现接口

CGLib必须依赖于CGLib的类库，但是它需要类来实现任何接口代理的是指定的类生成一个子类，

覆盖其中的方法，是一种继承但是针对接口编程的环境下推荐使用JDK的代理；

#### CGlib比JDK快？

1）使用CGLib实现动态代理，CGLib底层采用ASM字节码生成框架，使用字节码技术生成代理类，

在jdk6之前比使用Java反射效率要高。唯一需要注意的是，CGLib不能对声明为final的方法进行代理，

因为CGLib原理是动态生成被代理类的子类。

2）每一次jdk版本升级，jdk代理效率都得到提升，而CGLIB代理消息确有点跟不上步伐。

#### Spring如何选择用JDK还是CGLiB？

1）当Bean实现接口时，Spring就会用JDK的动态代理。

2）当Bean没有实现接口时，Spring使用CGlib是实现。

3）可以强制使用CGlib（在spring配置中加入<aop:aspectj-autoproxy proxy-target-class="true"/>）。