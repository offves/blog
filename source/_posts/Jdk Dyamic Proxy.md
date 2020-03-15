---
title: Jdk Dynamic Proxy
date: 2018-03-11 11:07:29
tags:
    - java
categories:
    - java
---
> 代理是一种常用的设计模式，其目的就是为其他对象提供一个代理以控制对某个对象的访问。代理类负责为委托类预处理消息，过滤消息并转发消息，以及进行消息被委托类执行后的后续处理。

代理模式UML图：
![代理模式UML图](/Users/offves/Projects/blog/source/_posts/images/代理模式UML图.png)

# 动态代理步骤:

 1. 创建一个实现接口 InvocationHandler 的类，实现 invoke 方法；

 2. 创建被代理的类以及接口；

 3. 通过Proxy的静态方法 newProxyInstance(ClassLoaderloader, Class[] interfaces, InvocationHandler h) 创建一个代理；

 4. 通过代理调用方法。

    

## 代理处理器:

```java
package com.offves.jdkproxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/
 * 代理处理器
 * @author offves
 * @since 2018-3-11 15:28
 */
public class JdkDynamicProxy implements InvocationHandler {
    private Object target;

    public JdkDynamicProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = null;
        try {
            System.out.println("invoke method before.");
            System.out.println("invoke method.");
            
            // 执行被代理类方法
            result = method.invoke(target, args);
    
            System.out.println("invoke method after.");
        } catch (Exception e) {
            System.out.println("invoke method exception");
            e.printStackTrace();
        }
        return result;
    }
}
```

## 被代理接口:

```java
package com.offves.jdkproxy;

/
 * 被代理接口
 * @author offves
 * @since 2018-3-11 15:24
 */
public interface Subject {
    void Hello(String str);
}
```

## 被代理类:

```java
package com.offves.jdkproxy;

/
 * 被代理类
 * @author offves
 * @since 2018-3-11 15:27
 */
public class HelloSubject implements Subject {
    @Override
    public void Hello(String str) {
        System.out.println(str);
    }
}
```

## 测试:

```java
package com.offves.jdkproxy;

import sun.misc.ProxyGenerator;

import java.io.FileOutputStream;
import java.io.IOException;
import java.lang.reflect.Proxy;

/
 * 测试 Jdk Dynamic Proxy
 * @author offves
 * @since 2018-3-11 15:35
 */
public class DynamicProxyTest {
    public static void main(String[] args) {
        JdkDynamicProxy jdkDynamicProxy = new JdkDynamicProxy(new HelloSubject());

        Subject proxyInstance = (Subject) Proxy.newProxyInstance(HelloSubject.class.getClassLoader(), HelloSubject.class.getInterfaces(), jdkDynamicProxy);

        System.out.println(proxyInstance.getClass().getName());

        proxyInstance.Hello("Hello Proxy");
    }
}
```

## console结果:

```
com.sun.proxy.$Proxy0
invoke method before.
invoke method.
Hello Proxy
invoke method after.
```



# 以newProxyInstance方法为切入点来剖析代理类的生成及代理方法的调用:

```java
public class Proxy implements java.io.Serializable {
    /
     * @param loader 类加载器
     * @param interfaces 被代理类实现的所有接口
     * @param h InvocationHandler
     */
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException {
        
        // InvocationHandler 为空, 抛出空指针异常
        if (h == null) {
            throw new NullPointerException();
        }
    
        // 克隆被代理类接口(不知道有啥子用)
        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }
    
        /*
         * Look up or generate the designated proxy class.
         */
        // 产生代理类
        Class<?# cl = getProxyClass0(loader, intfs);
    
        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }
            // 获取代理类构造器
            final Constructor<?# cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                // 修改代理类的访问权限
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            // 生成代理类的实例, 并把 InvocationHandler 的实例传给它的构造方法(以便于调用 InvocationHandler.invoke())
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
    
    /
     * 产生代理类
     */
    private static Class<?# getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
        // 接口数量不能大于 65535
        if (interfaces.length # 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        
        // 从缓存中去获取代理类
        return proxyClassCache.get(loader, interfaces);
    }
}
```

# 代理类缓存: 

```java
final class WeakCache<K, P, V# {
    /
     * Look-up the value through the cache. This always evaluates the
     * {@code subKeyFactory} function and optionally evaluates
     * {@code valueFactory} function if there is no entry in the cache for given
     * pair of (key, subKey) or the entry has already been cleared.
     *
     * @param key       possibly null key
     * @param parameter parameter used together with key to create sub-key and
     *                  value (should not be null)
     * @return the cached value (never null)
     * @throws NullPointerException if {@code parameter} passed in or
     *                              {@code sub-key} calculated by
     *                              {@code subKeyFactory} or {@code value}
     *                              calculated by {@code valueFactory} is null.
     */
    public V get(K key, P parameter) {
        Objects.requireNonNull(parameter);

        expungeStaleEntries();

        Object cacheKey = CacheKey.valueOf(key, refQueue);

        // lazily install the 2nd level valuesMap for the particular cacheKey
        ConcurrentMap<Object, Supplier<V># valuesMap = map.get(cacheKey);
        if (valuesMap == null) {
            ConcurrentMap<Object, Supplier<V># oldValuesMap
                = map.putIfAbsent(cacheKey,
                                  valuesMap = new ConcurrentHashMap<>());
            if (oldValuesMap != null) {
                valuesMap = oldValuesMap;
            }
        }

        // create subKey and retrieve the possible Supplier<V# stored by that
        // subKey from valuesMap
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
        Supplier<V# supplier = valuesMap.get(subKey);
        Factory factory = null;

        while (true) {
            if (supplier != null) {
                // supplier might be a Factory or a CacheValue<V# instance
                // 获取动态代理类，其中 supplier 是 Factory,这个类定义在 WeakCach 的内部。
                V value = supplier.get();
                if (value != null) {
                    return value;
                }
            }
            // else no supplier in cache
            // or a supplier that returned null (could be a cleared CacheValue
            // or a Factory that wasn't successful in installing the CacheValue)

            // lazily construct a Factory
            if (factory == null) {
                factory = new Factory(key, parameter, subKey, valuesMap);
            }

            if (supplier == null) {
                supplier = valuesMap.putIfAbsent(subKey, factory);
                if (supplier == null) {
                    // successfully installed Factory
                    supplier = factory;
                }
                // else retry with winning supplier
            } else {
                if (valuesMap.replace(subKey, supplier, factory)) {
                    // successfully replaced
                    // cleared CacheEntry / unsuccessful Factory
                    // with our Factory
                    supplier = factory;
                } else {
                    // retry with current supplier
                    supplier = valuesMap.get(subKey);
                }
            }
        }
    }
    
    /
     * A factory {@link Supplier} that implements the lazy synchronized
     * construction of the value and installment of it into the cache.
     */
    private final class Factory implements Supplier<V# {
        @Override
        public synchronized V get() { // serialize access
            // re-check
            Supplier<V# supplier = valuesMap.get(subKey);
            if (supplier != this) {
                // something changed while we were waiting:
                // might be that we were replaced by a CacheValue
                // or were removed because of failure ->
                // return null to signal WeakCache.get() to retry
                // the loop
                return null;
            }
            // else still us (supplier == this)

            // create new value
            V value = null;
            try {
                // 这才是重点
                value = Objects.requireNonNull(valueFactory.apply(key, parameter));
            } finally {
                if (value == null) { // remove us on failure
                    valuesMap.remove(subKey, this);
                }
            }
            // the only path to reach here is with non-null value
            assert value != null;

            // wrap value with CacheValue (WeakReference)
            CacheValue<V# cacheValue = new CacheValue<>(value);

            // put into reverseMap
            reverseMap.put(cacheValue, Boolean.TRUE);

            // try replacing us with CacheValue (this should always succeed)
            if (!valuesMap.replace(subKey, this, cacheValue)) {
                throw new AssertionError("Should not reach here");
            }

            // successfully replaced us with new CacheValue -# return the value
            // wrapped by it
            return value;
        }
    }
}
```

# valueFactory 的实现是 ProxyClassFactory (Proxy 类的内部类)

```java
private static final class ProxyClassFactory implements BiFunction<ClassLoader, Class<?>[], Class<?># {
    // 代理类前缀
    // prefix for all proxy class names
    private static final String proxyClassNamePrefix = "$Proxy";
    
    // 代理类唯一编号
    // next number to use for generation of unique proxy class names
    private static final AtomicLong nextUniqueNumber = new AtomicLong();
    
    @Override
    public Class<?# apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean# interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?# intf : interfaces) {
            /*
             * Verify that the class loader resolves the name of this
             * interface to the same Class object.
             */
            // 验证指定的类加载器(loader)加载接口所得到的Class对象(interfaceClass)是否与intf对象相同
            Class<?# interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            /*
             * Verify that the Class object actually represents an
             * interface.
             */
            // 验证该Class对象是不是接口
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            /*
             * Verify that this interface is not a duplicate.
             */
            // 验证该接口是否重复了
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }

        String proxyPkg = null;     // package to define proxy class in
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        /*
         * Record the package of a non-public proxy interface so that the
         * proxy class will be defined in the same package.  Verify that
         * all non-public proxy interfaces are in the same package.
         */
        for (Class<?# intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {
            // if no non-public proxy interfaces, use com.sun.proxy package
            // 指定代理类包名为 com.sun.proxy
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        /*
         * Choose a name for the proxy class to generate.
         */
        // 将当前nextUniqueNumber的值以原子的方式的加1，所以第一次生成代理类的名字为$Proxy0.class
        long num = nextUniqueNumber.getAndIncrement();
        // 拼接代理类全限定名
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        /*
         * Generate the specified proxy class.
         */
        // 生成代理类字节码(重点)
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            /*
             * A ClassFormatError here means that (barring bugs in the
             * proxy class generation code) there was some other
             * invalid aspect of the arguments supplied to the proxy
             * class creation (such as virtual machine limitations
             * exceeded).
             */
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

# 代理类字节码文件生成: 

```java
public class ProxyGenerator {
    public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        // 生成字节码文件的真正方法
        final byte[] var4 = var3.generateClassFile();
        // 保存字节码文件
        if (saveGeneratedFiles) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    try {
                        int var1 = var0.lastIndexOf(46);
                        Path var2;
                        if (var1 # 0) {
                            Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar));
                            Files.createDirectories(var3);
                            var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                        } else {
                            var2 = Paths.get(var0 + ".class");
                        }
    
                        Files.write(var2, var4, new OpenOption[0]);
                        return null;
                    } catch (IOException var4x) {
                        throw new InternalError("I/O exception saving generated file: " + var4x);
                    }
                }
            });
        }
    
        return var4;
    }
    
    private byte[] generateClassFile() {
        // 把 Object 的 hashCode(), equals(), toString() 加入代理类方法
        this.addProxyMethod(hashCodeMethod, Object.class);
        this.addProxyMethod(equalsMethod, Object.class);
        this.addProxyMethod(toStringMethod, Object.class);
        Class[] var1 = this.interfaces;
        int var2 = var1.length;

        int var3;
        Class var4;
        // 把所有接口的方法加入代理类方法
        for(var3 = 0; var3 < var2; ++var3) {
            var4 = var1[var3];
            Method[] var5 = var4.getMethods();
            int var6 = var5.length;

            for(int var7 = 0; var7 < var6; ++var7) {
                Method var8 = var5[var7];
                this.addProxyMethod(var8, var4);
            }
        }

        Iterator var11 = this.proxyMethods.values().iterator();

        List var12;
        while(var11.hasNext()) {
            var12 = (List)var11.next();
            checkReturnTypes(var12);// 检查代理类方法中是否有方法签名且返回值一样的
        }

        Iterator var15;
        try {
            // 加入构造方法
            this.methods.add(this.generateConstructor());
            var11 = this.proxyMethods.values().iterator();

            while(var11.hasNext()) {
                var12 = (List)var11.next();
                var15 = var12.iterator();

                while(var15.hasNext()) {
                    ProxyGenerator.ProxyMethod var16 = (ProxyGenerator.ProxyMethod)var15.next();
                    this.fields.add(new ProxyGenerator.FieldInfo(var16.methodFieldName, "Ljava/lang/reflect/Method;", 10));
                    this.methods.add(var16.generateMethod());
                }
            }

            this.methods.add(this.generateStaticInitializer());
        } catch (IOException var10) {
            throw new InternalError("unexpected I/O Exception", var10);
        }

        if (this.methods.size() # 65535) {//代理类方法超过65535将抛出异常
            throw new IllegalArgumentException("method limit exceeded");
        } else if (this.fields.size() # 65535) {//代理类字段超过65535将抛出异常
            throw new IllegalArgumentException("field limit exceeded");
        } else {
            ......
        }
    }
    
    private void addProxyMethod(Method var1, Class<?# var2) {
        String var3 = var1.getName();// 方法名
        Class[] var4 = var1.getParameterTypes();// 方法参数类型
        Class var5 = var1.getReturnType();// 方法返回值类型
        Class[] var6 = var1.getExceptionTypes();// 方法异常类型
        String var7 = var3 + getParameterDescriptors(var4);
        Object var8 = (List)this.proxyMethods.get(var7);
        if (var8 != null) {
            Iterator var9 = ((List)var8).iterator();

            while(var9.hasNext()) {
                ProxyGenerator.ProxyMethod var10 = (ProxyGenerator.ProxyMethod)var9.next();
                if (var5 == var10.returnType) {
                    ArrayList var11 = new ArrayList();
                    collectCompatibleTypes(var6, var10.exceptionTypes, var11);
                    collectCompatibleTypes(var10.exceptionTypes, var6, var11);
                    var10.exceptionTypes = new Class[var11.size()];
                    var10.exceptionTypes = (Class[])var11.toArray(var10.exceptionTypes);
                    return;
                }
            }
        } else {
            var8 = new ArrayList(3);
            this.proxyMethods.put(var7, var8);
        }

        ((List)var8).add(new ProxyGenerator.ProxyMethod(var3, var4, var5, var6, var2, null));
    }
}
```

# 生成 subjectProxy.class: 
将测试类修改为以下内容

```java
package com.offves.jdkproxy;

import sun.misc.ProxyGenerator;

import java.io.FileOutputStream;
import java.io.IOException;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

/
 * 测试 Jdk Dynamic Proxy
 * @author offves
 * @since 2018-3-11 15:35
 */
public class DynamicProxyTest {
    public static void main(String[] args) {
        InvocationHandler invocationHandler = new JdkDynamicProxy(new HelloSubject());

        Subject proxyInstance = (Subject) Proxy.newProxyInstance(invocationHandler.getClass().getClassLoader(), HelloSubject.class.getInterfaces(), invocationHandler);

        System.out.println(proxyInstance.getClass().getName());

        proxyInstance.Hello("Hello Proxy");

        createProxyClass();
    }
    
    
    private static void createProxyClass() {
        String name = "HelloSubjectProxy";
        byte[] bytes = ProxyGenerator.generateProxyClass(name, new Class[]{HelloSubject.class});
        FileOutputStream outputStream = null;
        try {
            outputStream = new FileOutputStream(name+".class");
            outputStream.write(bytes);
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            if (outputStream != null) {
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

# HelloSubjectProxy.class

```java
import com.offves.jdkproxy.HelloSubject;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class HelloSubjectProxy extends Proxy implements HelloSubject {
    private static Method m1;
    private static Method m8;
    private static Method m2;
    private static Method m6;
    private static Method m5;
    private static Method m7;
    private static Method m3;
    private static Method m9;
    private static Method m0;
    private static Method m4;

    public HelloSubjectProxy(InvocationHandler var1) throws  {
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

    public final void notify() throws  {
        try {
            super.h.invoke(this, m8, (Object[])null);
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

    public final void wait(long var1) throws InterruptedException {
        try {
            super.h.invoke(this, m6, new Object[]{var1});
        } catch (RuntimeException | InterruptedException | Error var4) {
            throw var4;
        } catch (Throwable var5) {
            throw new UndeclaredThrowableException(var5);
        }
    }

    public final void wait(long var1, int var3) throws InterruptedException {
        try {
            super.h.invoke(this, m5, new Object[]{var1, var3});
        } catch (RuntimeException | InterruptedException | Error var5) {
            throw var5;
        } catch (Throwable var6) {
            throw new UndeclaredThrowableException(var6);
        }
    }

    public final Class getClass() throws  {
        try {
            return (Class)super.h.invoke(this, m7, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void Hello(String var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void notifyAll() throws  {
        try {
            super.h.invoke(this, m9, (Object[])null);
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

    public final void wait() throws InterruptedException {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | InterruptedException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m8 = Class.forName("com.offves.jdkproxy.HelloSubject").getMethod("notify");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m6 = Class.forName("com.offves.jdkproxy.HelloSubject").getMethod("wait", Long.TYPE);
            m5 = Class.forName("com.offves.jdkproxy.HelloSubject").getMethod("wait", Long.TYPE, Integer.TYPE);
            m7 = Class.forName("com.offves.jdkproxy.HelloSubject").getMethod("getClass");
            m3 = Class.forName("com.offves.jdkproxy.HelloSubject").getMethod("Hello", Class.forName("java.lang.String"));
            m9 = Class.forName("com.offves.jdkproxy.HelloSubject").getMethod("notifyAll");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m4 = Class.forName("com.offves.jdkproxy.HelloSubject").getMethod("wait");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

以上就是对代理类如何生成, 代理类方法如何被调用的分析.

