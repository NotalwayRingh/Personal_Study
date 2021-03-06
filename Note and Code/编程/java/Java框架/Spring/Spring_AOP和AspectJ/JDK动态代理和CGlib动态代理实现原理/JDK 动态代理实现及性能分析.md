JDK 动态代理实现是必须基于接口的,意思就是
假设你要织入的函数为 hello(), hello在类 Realsubject 中定义,
你要将 Realsubject织入, 那么必须先让 Realsubject 继承一个自定义的接口,比如自定义一个接口Subject,
接口中再声明 hello() 方法 

为什么 JDK动态代理 一定要基于接口呢?
JDK 动态代理使用的 是 java.lang.reflect.Proxy; 中的 newProxyInstance 方法
查看 Spring AOP 的 JDK动态代理 源码部分 ,可以发现它调用了 Proxy.newProxyInstance 方法

首先看看 Proxy.newProxyInstance 即 JDK 动态代理是如何使用的:
```java
//定义一个接口
package com.sun.proxy;

public interface Subject {
    void sayHello();
    void sayGoodBye();
}

//实现这个接口
package com.sun.proxy;

public class RealSubject implements Subject{
    public void sayHello() {
        System.out.println("Hello");
    }

    public void sayGoodBye() {
        System.out.println("GoodBye");
    }
}
```
要使用 newProxyInstance, 就必须再另外实现一个 Invocation 接口 的代理类,不妨称为 ProxySubject 类  
并且必须重写 InvocationHandler 的 invoke 方法
```java
package com.sun.proxy;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class ProxySubject implements InvocationHandler{
    private Object target;

    public ProxySubject(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("调用前");
        Object object = method.invoke(target, args);
        System.out.println("调用后");
        return object;
    }
}
```
现在我们可以使用 newProxyInstance 了,创建一个 Test类
```java
package com.sun.proxy;
import java.lang.reflect.Proxy;

public class Test {
    public static void main(String[] args) {
        //System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        //如果要获得生成的动态代理类的 .class文件,则要通过上面一句改变系统环境变量
        
        Subject subject = (Subject) Proxy.newProxyInstance(RealSubject.class.getClassLoader(), RealSubject.class.getInterfaces(), new ProxySubject(new RealSubject()));
        subject.sayHello();
        subject.sayGoodBye();

        //查看subject对象的类型
        System.out.println(subject.getClass().getName());
    }
}
```
结果如下:
调用前
Hello
调用后
调用前
GoodBye
调用后
com.sun.proxy.$Proxy0


我们使用Spring AOP ,假设我们指定了被拦截的函数,被拦截的函数之前要调用的函数(@Before修饰),以及被拦截
的函数执行后要调用的函数(@After),那么最后他们一定会被整合一个动态代理类的 invoke 方法中,上面的
    System.out.println("调用前");
    System.out.println("调用后");
就相当于 Spring AOP 中用 @Before 和 @After 修饰的方法,我们可以用 反射获取 @Before 和 @After 的方法
然后放入 invoke 方法中



下面回到 JDK 动态代理类, newProxyInstance 到底干了什么? 
它最终创建了一个动态代理类(生成一个一个新的.class文件),这个动态代理类的名称有自己的一套命名规则
我们要拦截的函数和织入的函数最终都交给动态代理类执行
动态代理类的内部并没有要拦截的函数的具体内容(即本例中 RealSubject中 的sayHello 和 sayGoodBye函数)
它需要通过反射来获取这些函数的 Method 对象,最终调用他们,这也是 JDK 动态代理的性能瓶颈所在
在 Test类的 main 方法的第一句加上:
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
我们就可以获取代理类的.class文件(本例中将获得 $Proxy0.class 文件),反编译查看代理类的代码:
```java
$Proxy0.class 文件:

package com.sun.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements Subject {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m4;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void sayGoodBye() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("com.sun.proxy.Subject").getMethod("sayHello");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m4 = Class.forName("com.sun.proxy.Subject").getMethod("sayGoodBye");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
查看源码 m1 m2 m3 m0 均使用了反射,其中 m3 m4 就是我们的  sayHello 和 sayGoodBye 方法 

现在再来看 newProxyInstance 内部的大致过程,里面还有大量的安全验证(有些类不允许被设置代理)等细节我们将
其省略

先看这个 newProxyInstance 函数:  它需要的三个参数:

1.类加载器 
2.RealSubject.class.getInterfaces(), 就是我们自定义的 Subject 接口的Class 对象
3.我们自定义的实现了 Invocation 的代理类的Class 对象 
)
注意到第二个参数了吗, 它就是为什么我们最终一定会用到 自定义的 Subject 接口原因

它会获取这个 Subject 接口内部的所有函数信息,然后经过各种操作,动态生成字节码(生成.class文件),
自己再另外构建一个动态代理类,这个动态代理类的类型名字是它根据一套规则定义的
(比如上例中 $Proxy0 就是最后生成的代理类的名称)

在新构建的类(上例中 $Proxy0 )中,它继承了我们自定义的 Subject 接口(因此我们可以用
Subject 类型变量来 获取 newProxyInstance 的返回值,上例中 newProxyInstance 返回的是
$Proxy0 ), $Proxy0 实现了 Subject 接口内的所有方法(注意是所有),比如本例中的Subject内有 sayHello()
方法和 sayGoodBye()方法 那  $Proxy0 也会实现 sayHello()方法和 sayGoodBye()方法
$Proxy0 中实现的所有方法内部都会执行 invoke 方法

上例中 $Proxy0 类的 sayHello() 和 sayGoodBye() 内部都会调用 invoke,参数为
1.$Proxy0 类的对象
2.RealSubject 中的 sayHello或 sayGoodBye 方法的 Method 类对象
3.sayHello 或 sayGoodBye 方法的参数,本例中为空

再总结:
这里回顾一下 newProxyInstance 是如何把我们 RealSubject 里的 hello() 与invoke 结合起来的:
Proxy 的 newProxyInstance() 会另外创建一个代理类,即最终会创建一个代理类的.class文件(这个代理类的名称
有自己的一套命名规则),在创建动态代理类的过程中,我们需要将接口 Class对象 作为参数传给 newProxyInstance
()函数 (例子中的Subject的 Class对象) ,借助反射就能知道接口中所有的函数名称,最后动态代理类继承了这个接
口并重新实现接口所有的函数,此外,新生成的动态代理类还会在构造函数中把我们自定义的实现了 
InvocationHandler 的代理类(本例了中的 RealSubject类)注入到自己内部(这里我们自定义的动态代理类就进来
了! ),重新实现的每个函数都会调用 invoke() 方法,这个 invoke()方法就是 Realsubject 的 invoke 方法这
样两者就联系起来了.观察 invoke 方法,它的第二个参数是 Method 类型的, 也就是说,动态代理类必须用反射获取
要拦截的函数(例子中的 sayHello和 sayGoodBye),使用反射会导致性能的下降
 
而在 Spring AOP 中,只不过再在 invoke 方法中做手脚,把我们要拦截的函数 ,要织入的函数都集成到
invoke 方法中去了(Spring 通过反射读取@Before 等其它要织入的函数,把它们组织到 invoke()中)
