---
title: Tomcat类加载机制与热部署原理详解
date: 2025-10-13 1:07:00
categories:
  - 后端技术
  - 中间件
  - Tomcat
tags:
  - Tomcat
  - 类加载机制
  - 热部署
---

# 1.Tomcat类加载机制详解
## 1.1 JVM类加载器
Java中有3种类加载器，当然你也可以自定义类加载器
- 引导类加载器（启动类加载器）：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如rt.jar、charsets.jar
- 扩展类加载器：负责加载支撑JVM运行的JRE的lib目录下ext扩展目录中的核心jar包
- 应用程序类加载器（系统类加载器）：负责ClassPath路径下的类包，主要就是加载你自己写的类
- 自定义类加载器：自己实现，负责加载自定义路径下的类包
``` Java
public class ClassLoaderDemo{
  public static void main(String[] args){
    // BootStrapClassLoader c/c++实现，java层面是获取不到的，会输出null
    System.out.println(ReentrantLock.class.getClassLoader());
    // ExtClassLoader
    System.out.println(ZipInfo.class.getClassLoader());
    // AppClassLoader
    System.out.println(ClassLoaderDemo.class.getClassLoader());
    System.out.println("===========JVM类加载器父子关系==============");
    // AppClassLoader 
    System.out.println(ClassLoader.getSystemClassLoader());
    // ExtClassLoader
    System.out.println(ClassLoader.getSystemClassLoader().getParent());
    // BootStrapClassLoader
    System.out.println(ClassLoader.getSystemClassLoader().getParent().getParent());
  }
}
```
## 1.2 双亲委派机制
Java中的类加载依赖双亲委派机制，加载某个类时会先委托父加载器寻找目标类，找不到再委托上层父加载器加载，如果所有父加载器在自己的加载类路径下都找不到目标类，则在自己的类加载目录中查找并载入目标类，双亲委派机制说简单点就是，先找父亲加载，不行再由儿子自己加载
<img width="1404" height="1188" alt="image" src="https://github.com/user-attachments/assets/dcd36232-ea06-418a-a266-178af135f9b0" />
### 为什么设计双亲委派机制？
- 沙箱安全机制:防止核心类库API被随意篡改
- 避免类的重复加载：父加载器已经加载了该类时，就没必要子类加载器再加载一次，保证被加载类的唯一性
### ClassLocader#loadClass源码分析
```Java
public abstract class ClassLoader {
    // The parent class loader for delegation
    // Note: VM hardcoded the offset of this field, thus all new fields
    // must be added *after* it.
    private final ClassLoader parent;
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
}
```
## 1.3 Tomcat为什么（如何）打破双亲委派机制？
Tomcat的主要目的是充当应用服务器并处理用户的业务请求，使用场景可能会单机部署多应用。所以Tomcat打破双亲委派机制，核心是为了解决多Web应用独立运行的类隔离需求：双亲委派要求类加载器优先委托父加载器加载类，而Tomcat中多个Web应用可能依赖同一类的不同版本（如不同Spring版本），若按双亲委派，父加载器（如CommonClassLoader）加载一个版本后，所有应用都只能使用该版本，会引发类版本冲突；因此Tomcat自定义了WebAppClassLoader等加载器，让每个Web应用的类加载器优先加载自身WEB-INF/classes和WEB-INF/lib下的类，仅在自身未找到时才委托父加载器，从而实现不同应用类的独立隔离，保证多应用在同一容器中互不干扰地运行。
Tomcat打破双亲委派具体实现就是重写ClassLoader的两个方法：findClass和loadClass，来改变双亲委派的类加载顺序，但是直接重写这两个实现依然需要保证核心类库被篡改，所以为了保证核心类库的安全性，并没有一开始就使用系统类型加载器加载，而是先查询本地目录缓存，为了避免本地目录下的类覆盖JRE的核心类，会先尝试用JVM扩展类加载器ExtClassLoader去加载
### findClass方法
```Java
    @Override
    public Class<?> findClass(String name) throws ClassNotFoundException {

        if (log.isDebugEnabled())
            log.debug("    findClass(" + name + ")");

        checkStateForClassLoading(name);

        // (1) Permission to define this class when using a SecurityManager
        if (securityManager != null) {
            int i = name.lastIndexOf('.');
            if (i >= 0) {
                try {
                    if (log.isTraceEnabled())
                        log.trace("      securityManager.checkPackageDefinition");
                    securityManager.checkPackageDefinition(name.substring(0,i));
                } catch (Exception se) {
                    if (log.isTraceEnabled())
                        log.trace("      -->Exception-->ClassNotFoundException", se);
                    throw new ClassNotFoundException(name, se);
                }
            }
        }

        // Ask our superclass to locate this class, if possible
        // (throws ClassNotFoundException if it is not found)
        Class<?> clazz = null;
        try {
            if (log.isTraceEnabled())
                log.trace("      findClassInternal(" + name + ")");
            try {
                if (securityManager != null) {
                    PrivilegedAction<Class<?>> dp =
                        new PrivilegedFindClassByName(name);
                    clazz = AccessController.doPrivileged(dp);
                } else {
                     // 1、先在应用本地目录下查找类 
                    clazz = findClassInternal(name);
                }
            } catch(AccessControlException ace) {
                log.warn("WebappClassLoader.findClassInternal(" + name
                        + ") security exception: " + ace.getMessage(), ace);
                throw new ClassNotFoundException(name, ace);
            } catch (RuntimeException e) {
                if (log.isTraceEnabled())
                    log.trace("      -->RuntimeException Rethrown", e);
                throw e;
            }
            if ((clazz == null) && hasExternalRepositories) {
                try {
                     // 2、如果在本地目录没有找到，委派父加载器去查找
                    clazz = super.findClass(name);
                } catch(AccessControlException ace) {
                    log.warn("WebappClassLoader.findClassInternal(" + name
                            + ") security exception: " + ace.getMessage(), ace);
                    throw new ClassNotFoundException(name, ace);
                } catch (RuntimeException e) {
                    if (log.isTraceEnabled())
                        log.trace("      -->RuntimeException Rethrown", e);
                    throw e;
                }
            }
            // 3、如果父加载器也没找到，抛出异常
            if (clazz == null) {
                if (log.isDebugEnabled())
                    log.debug("    --> Returning ClassNotFoundException");
                throw new ClassNotFoundException(name);
            }
        } catch (ClassNotFoundException e) {
            if (log.isTraceEnabled())
                log.trace("    --> Passing on ClassNotFoundException");
            throw e;
        }

        // Return the class we have located
        if (log.isTraceEnabled())
            log.debug("      Returning class " + clazz);

        if (log.isTraceEnabled()) {
            ClassLoader cl;
            if (Globals.IS_SECURITY_ENABLED){
                cl = AccessController.doPrivileged(
                    new PrivilegedGetClassLoader(clazz));
            } else {
                cl = clazz.getClassLoader();
            }
            log.debug("      Loaded by " + cl.toString());
        }
        return (clazz);

    }
```
### loadClass方法
```Java
@Override
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

    synchronized (getClassLoadingLock(name)) {
        Class<?> clazz = null;
        // 1、从本地缓存中查找是否加载过此类
        clazz = findLoadedClass0(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return clazz;
        }

        // 2、从AppClassLoader中查找是否加载过此类
        clazz = findLoadedClass(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return clazz;
        }

        String resourceName = binaryNameToPath(name, false);
        // 3、尝试用ExtClassLoader 类加载器加载类,防止应用覆盖JRE的核心类
        ClassLoader javaseLoader = getJavaseClassLoader();
        boolean tryLoadingFromJavaseLoader;
        try {
            URL url;
            if (securityManager != null) {
                PrivilegedAction<URL> dp = new PrivilegedJavaseGetResource(resourceName);
                url = AccessController.doPrivileged(dp);
            } else {
                url = javaseLoader.getResource(resourceName);
            }
            tryLoadingFromJavaseLoader = (url != null);
        } catch (Throwable t) {
            tryLoadingFromJavaseLoader = true;
        }

        boolean delegateLoad = delegate || filter(name, true);

        // 4、判断是否设置了delegate属性,如果设置为true那么就按照双亲委派机制加载类
        if (delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader1 " + parent);
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }

        // 5、默认是设置delegate是false的,那么就会先用WebAppClassLoader进行加载
        if (log.isDebugEnabled())
            log.debug("  Searching local repositories");
        try {
            clazz = findClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from local repository");
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // 6、如果在WebAppClassLoader没找到类，那么就委托给AppClassLoader去加载
        if (!delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader at end: " + parent);
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }
    }
    throw new ClassNotFoundException(name);
}

```

## 1.4 Tomcat如何隔离Web应用
Tomcat作为Servlet容器，它负责加载我们的Servlet类，此外它还负责加载Servlet所依赖的JAR包。并且Tomcat本身也是一个Java程序，因此它需要加载自己的类和依赖的JAR包。Tomcat此时需要解决下面这三个问题：

1. Tomcat运行了两个Web应用程序，两个Web应用中有同名的Servlet,但是功能不同，Tomcat需要同时加载和管理这两个同名的Servlet类，保证它们不会冲突。Tomcat是如何实现Web应用之间的所有类都完成隔离的？
2. 两个Web应用都依赖同一个第三方的JAR包，比如Spring,Spring的JAR包被加载到内存后，Tomcat要保证这两个Web应用能够共享，也就是说Spring的JAR包只被加载一次，否则随着依赖第三方JAR包的增多，JVM的内存会膨胀，Tomcat是如何解决的？
3. 跟JVM一样，我们需要隔离Tomcat本身的类和Web应用的类。Tomcat是怎么实现的？
也就是如何实现内部类隔离，三方JAR包资源共享，以及自身的隔离性
### Tomcat类加载器的层次结构
为了解决这些问题，Tomcat设计了类加载器的层次结构，它们的关系如下图所示：
<img width="1668" height="2034" alt="image" src="https://github.com/user-attachments/assets/ffc48507-ed9f-47a8-9b39-76ff73eaa2be" />
- commonLoader:Tomcat最基本的类加载器，加载路径中的class可以被Tomcat容器以及本身各个Webapp访问；
- catalinaLoader:Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见；
- sharedLoader:各个Webapp共享的类加载器，加载路径中的class对于所有的Webapp可见，但是对于Tomcat容器不可见；
- WebappClassLoader：各个Webapp私有的类加载器，加载路径中的class，只对当前webapp可见，加载war包里相关的类，每个war包应用都有自己的WebAppClassLoader，实现相互隔离，加载对应各自的依赖版本
### 全盘负责委托机制
当一个ClassLoader装载一个类的时候，除非显式的使用另一个ClassLoader，该类所依赖及引用的类也由这个ClassLoader载入。
比如Spring作为Bean工厂，它需要创建业务类的实例，并且创建业务类实例之前需要加载这些类。Spring是通过Class.forName来加载业务类的，下面是forName的源码：
``` Java
public static Class<?> forName(String className){
    Class<?> caller = Reflection.getCallerClass();
    return forName(className,true,CLassLoader.getClassLoader(caller),caller);
}
```
forName的函数中，会默认使用调用者的类加载器去加载业务类。
Web应用之间共享的JAR包可以交给SharedClassLoader来加载，从而避免重复加载。Spring作为共享的第三方JAR包，它本身是由SharedClassLoader来加载的，Spring又要去加载业务类，按照前面的那条规则，加载Spring的类加载器也会用来加载业务类，但是业务类再Web应用目录下，不在SharedClassLoader的加载路径下，Tomcat是如何解决这个问题的？
### 线程上下文加载器
于是线程上下文加载器登场了，它其实是一种类加载器传递机制。为什么叫做“线程上下文加载器”？因为这个类加载器保存在线程私有数据里，只要是同一个线程，一旦设置了线程上下文加载器，在线程后续执行过程中就能把这个线程类加载器取出来使用。因此Tomcat为每个Web应用创建一个WebAppClassLoader类加载器，并在启动Web应用的线程里设置线程上下文加载器，这样Spring在启动时就将线程上下文类加载器取出来，用来加载Bean。源代码如下：
``` Java
c1 = Thread.currentThread().getContextClassLoader();
```
线程上下文加载器不仅仅可以用在Tomcat和Spring类加载器的场景里，核心框架类需要加载具体实现类时都可以用到它，比如我们熟悉的JDBC就是通过上下文类加载器来加载不同的数据库驱动的
# 2. Tomcat热加载和热部署
项目开发过程中，经常要改动Java/JSP文件，但是又不想重新启动Tomcat,有两种方式：热加载和热部署。热部署表示重新部署应用，它执行的主体是HOST。热加载表示重新加载class，它的执行主体是Context。
- 热加载：在server.xml -> context标签中 设置 reloadable = "true"
```XML
<Context docBase="C:\project" path="/project" reloadable="true"/>
```
- 热部署：在server.xml -> Host标签中 设置 autoDeploy="true"
``` XML
<Host name="localhost" appBase="webapps" unpackWARS="true" autoDeploy="true" />
```
它们的区别是：
- 热加载的实现方式是Web容器启动一个后台线程，定期检测类文件的变化，如果有变化，就重新加载类，这个过程中不会清空Session，一般用在开发环境
- 热部署原理类似，也是由后台线程定时检测Web应用的变化，但它会重新加载整个Web应用。这种方式会清空Session，比热加载更加彻底，一般用于生产环境

## 2.1 Tomcat开启后台线程执行周期性任务
Tomcat 通过ScheduledThreadPoolExecutor（定时线程池） 管理后台周期性任务，核心是在容器初始化时（如 Service 启动阶段）创建固定数量的后台线程，绑定到 Catalina 生命周期中统一管理。典型场景包括：会话过期清理（定期扫描 HttpSession，销毁超时会话）、日志滚动（按时间 / 大小切割访问日志）、连接池维护（检测并回收空闲数据库连接）、集群节点心跳检测等。任务调度支持 “固定延迟”（如每隔 30 秒执行）或 “固定速率”（如每分钟执行一次），且线程池会自动处理任务异常，避免单个任务失败导致整个线程池崩溃，确保周期性任务稳定执行且不阻塞主线程。

## 2.2 Tomcat热加载实现原理
Tomcat 热加载的核心是基于类加载器的 “重新加载” 机制，仅针对 Web 应用内部的类和资源（如 WEB-INF/classes、WEB-INF/lib），无需重启整个 Tomcat 容器。具体逻辑：
1. 后台线程（如 WebAppLoader 的监控线程）定期扫描 WEB-INF/classes 和 lib 目录下文件的 “最后修改时间”，对比上次扫描记录；
2. 若检测到文件变化（如.class 文件更新、JAR 包替换），Tomcat 会销毁当前 Web 应用的WebAppClassLoader（应用级类加载器）（同时销毁其加载的所有类、Servlet 实例、线程资源）；
3. 创建新的 WebAppClassLoader，重新加载更新后的类和资源，初始化 Servlet、Spring 容器等组件，完成 “热加载”。该机制仅适用于开发环境（如调试时修改代码），且无法处理服务器级配置（如 server.xml）的变化，因这类配置依赖 Tomcat 核心类加载器，无法单独重新加载。
``` Java
// 1. 启动资源监控线程（位于WebAppLoader中）
@Override
public void start() {
    // 启动定时监控线程（每2秒扫描一次资源变化）
    monitorThread = new Thread(new ResourceMonitor(), "WebappResourceMonitor");
    monitorThread.setDaemon(true);
    monitorThread.start();
}

// 2. 资源监控逻辑（检测类或JAR包变化）
class ResourceMonitor implements Runnable {
    private long lastModified = 0;  // 上次修改时间

    @Override
    public void run() {
        while (running) {
            // 扫描WEB-INF/classes和lib目录的最后修改时间
            long currentModified = getResourcesLastModified();
            if (currentModified > lastModified) {
                // 检测到变化，触发热加载
                context.reload();  // 调用Context的重新加载方法
                lastModified = currentModified;
            }
            Thread.sleep(2000);  // 间隔2秒扫描
        }
    }
}

// 3. Context的reload方法（销毁旧类加载器，创建新的）
@Override
public void reload() {
    // 销毁当前Web应用的类加载器和资源
    WebappClassLoader oldLoader = this.loader;
    oldLoader.stop();  // 释放类、Servlet实例等资源
    
    // 创建新的类加载器，重新加载资源
    this.loader = new WebappClassLoader(oldLoader.getParent());
    this.loader.start();  // 加载更新后的类和配置
}
```
## 2.3 Tomcat热部署实现原理
Tomcat 热部署是整个 Web 应用的 “卸载 - 重新部署” 过程，支持替换应用的完整资源（包括 WAR 包、配置文件、静态资源等），可用于生产环境的版本更新（通常需短暂停止应用）。具体逻辑：
1. 触发方式（手动复制新 WAR 包、通过 Manager 应用 / API 发送部署指令）；
2. 卸载旧应用：停止应用所有 Servlet 和 Filter，关闭数据库连接池、线程池等资源，销毁对应的 WebAppClassLoader 和应用上下文（Context），删除旧应用的临时目录；
3. 部署新应用：解压新 WAR 包（若为 WAR 格式），创建新的 Context 和 WebAppClassLoader，加载新的类、资源和配置文件，初始化 Servlet、Listener 等组件，绑定到 Connector（连接器）接收请求。

与热加载的核心区别是：热部署针对 “整个应用” 的替换，可处理配置文件（如 web.xml）的变化；热加载仅针对 “类和资源” 的重新加载，依赖应用级类加载器，不涉及 Context 销毁。

``` Java
// 1. 热部署触发（检测webapps目录下的WAR包变化）
public class HostConfig implements LifecycleListener {
    @Override
    public void lifecycleEvent(LifecycleEvent event) {
        if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
            checkResources();  // 定期检查应用资源变化
        }
    }

    private void checkResources() {
        // 扫描webapps目录下的WAR包或目录
        for (File app : webappsDir.listFiles()) {
            if (isModified(app)) {  // 检测到应用更新（如WAR包替换）
                redeploy(app);  // 执行热部署
            }
        }
    }
}

// 2. 重新部署逻辑（卸载旧应用+部署新应用）
private void redeploy(File app) {
    String appName = app.getName();
    // 第一步：卸载旧应用
    Context oldContext = host.findContext(appName);
    if (oldContext != null) {
        host.removeChild(oldContext);  // 从Host中移除旧Context
        oldContext.stop();  // 停止旧应用（释放连接池、线程等）
        oldContext.destroy();  // 销毁旧应用资源
    }
    // 第二步：部署新应用
    Context newContext = createContext(app);  // 创建新Context
    host.addChild(newContext);  // 添加到Host
    newContext.start();  // 启动新应用（加载新类、初始化Servlet等）
}
```


