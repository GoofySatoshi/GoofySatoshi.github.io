---
title: JVM核心知识点八股文
date: 2025-12-11 15:30:00
categories: 
  - 后端面试
tags: 
  - JVM
  - 垃圾回收
  - 内存模型
  - 八股文
permalink: /jvm-interview-2025/
---

# JVM 核心知识点八股文（高频面试版）
本文整理 JVM 面试中最常考察的核心知识点，覆盖内存结构、垃圾回收、类加载、性能调优等高频考点，适合面试复习和知识点梳理。

## 一、JVM 内存结构（运行时数据区）
JDK 8 及以上版本的内存结构移除了永久代（PermGen），替换为元空间（Metaspace），核心分为以下区域：

### 1. 线程私有区域
- **程序计数器**：记录当前线程执行的字节码行号，唯一不会抛出 OOM 的区域；
- **虚拟机栈**：存储方法调用的栈帧（局部变量表、操作数栈、动态链接、方法出口），栈深度溢出抛 `StackOverflowError`，内存不足抛 OOM；
- **本地方法栈**：为 Native 方法服务，与虚拟机栈逻辑一致，同样会抛 `StackOverflowError`/OOM。

### 2. 线程共享区域
- **堆**：JVM 最大的内存区域，存储对象实例和数组，是垃圾回收（GC）的核心区域；
  - 分代：新生代（Eden + Survivor0 + Survivor1，比例 8:1:1）、老年代；
  - 溢出：对象无法分配内存时抛 `OutOfMemoryError: Java heap space`。
- **元空间（Metaspace）**：替代永久代，存储类元信息、常量池、方法字节码等，直接使用本地内存，默认无上限（可通过 `-XX:MetaspaceSize`/`-XX:MaxMetaspaceSize` 限制），溢出抛 `OutOfMemoryError: Metaspace`。

### 3. 直接内存（非运行时数据区）
- 基于 NIO 的堆外内存，不受 JVM 堆大小限制，但受物理内存限制，溢出抛 `OutOfMemoryError: Direct buffer memory`。

## 二、垃圾回收（GC）核心
### 1. 如何判断对象可回收？
- **引用计数法**：对象引用数为 0 则标记为可回收，缺点：无法解决循环引用问题；
- **可达性分析算法**：以 GC Roots 为起点，遍历对象引用链，无引用链的对象标记为可回收；
  - GC Roots 包括：虚拟机栈中引用的对象、本地方法栈中引用的对象、方法区中类静态属性/常量引用的对象。

### 2. 常见 GC 算法
- **标记-清除（Mark-Sweep）**：标记可回收对象 → 清除，缺点：内存碎片、效率低；
- **标记-复制（Mark-Copy）**：将内存分为两块，只使用一块，回收时将存活对象复制到另一块，优点：无内存碎片，缺点：内存利用率低（仅 50%），新生代（Eden/Survivor）采用此算法；
- **标记-整理（Mark-Compact）**：标记可回收对象 → 存活对象向一端移动 → 清除端外内存，无碎片、利用率 100%，老年代采用此算法。

### 3. 经典垃圾收集器
| 收集器       | 适用区域 | 核心特点                     | 回收线程 |
|--------------|----------|------------------------------|----------|
| Serial       | 新生代   | 单线程、STW 时间长、简单高效 | 单线程   |
| ParNew       | 新生代   | Serial 多线程版本            | 多线程   |
| Parallel Scavenge | 新生代 | 关注吞吐量（运行代码时间/总时间） | 多线程 |
| Serial Old   | 老年代   | Serial 老年代版本            | 单线程   |
| Parallel Old | 老年代   | Parallel Scavenge 老年代版本 | 多线程   |
| CMS          | 老年代   | 低延迟、并发回收、三步 STW   | 多线程   |
| G1           | 整堆     | 分区回收、兼顾吞吐量和延迟   | 多线程   |
| ZGC/Shenandoah | 整堆   | 超低延迟（毫秒级）、大内存友好 | 多线程 |

### 4. GC 触发时机
- **Minor GC（新生代 GC）**：Eden 区满时触发，频率高、STW 时间短；
- **Major GC（老年代 GC）**：老年代空间不足时触发，常伴随 Minor GC，STW 时间长；
- **Full GC**：整堆回收（新生代+老年代+元空间），触发场景：老年代满、元空间满、System.gc() 显式调用、Minor GC 后老年代无法容纳晋升对象。

## 三、类加载机制
### 1. 类加载的完整流程
- **加载**：将类的字节码文件（.class）加载到内存，生成 Class 对象；
- **验证**：校验字节码合法性（文件格式、元数据、字节码指令、符号引用验证）；
- **准备**：为类静态变量分配内存并赋默认值（如 int → 0，引用 → null）；
- **解析**：将符号引用替换为直接引用（内存地址）；
- **初始化**：执行静态代码块、为静态变量赋初始值（程序员定义的值）。

### 2. 类加载器分类
- **启动类加载器（Bootstrap ClassLoader）**：C++ 实现，加载 JRE/lib 核心类（如 rt.jar）；
- **扩展类加载器（Extension ClassLoader）**：Java 实现，加载 JRE/lib/ext 扩展类；
- **应用类加载器（Application ClassLoader）**：Java 实现，加载用户自定义类（classpath 下）；
- **自定义类加载器**：继承 ClassLoader，实现自定义加载逻辑（如热部署、加密类加载）。

### 3. 双亲委派机制
- **核心规则**：类加载时先委托父加载器加载，父加载器无法加载时再自己加载；
- **优点**：避免类重复加载、保证核心类（如 java.lang.String）不被篡改；
- **打破场景**：Tomcat 类加载（为不同 Web 应用隔离类）、热部署、SPI 加载（如 JDBC）。

## 四、JVM 性能调优
### 1. 核心调优参数
```bash
# 堆内存设置
-Xms2g        # 初始堆内存（建议与-Xmx一致，避免频繁扩容）
-Xmx2g        # 最大堆内存
-Xmn512m      # 新生代内存（建议占堆的 1/4~1/3）
-XX:SurvivorRatio=8  # Eden:S0:S1 = 8:1:1
-XX:NewRatio=2       # 老年代:新生代 = 2:1（JDK8 前）

# 元空间设置
-XX:MetaspaceSize=128m  # 元空间初始大小
-XX:MaxMetaspaceSize=256m # 元空间最大大小

# GC 收集器设置
-XX:+UseParallelGC       # 新生代使用 Parallel Scavenge
-XX:+UseParallelOldGC    # 老年代使用 Parallel Old
-XX:+UseG1GC             # 使用 G1 收集器
-XX:MaxGCPauseMillis=200 # G1 最大停顿时间

# 日志配置
-XX:+PrintGCDetails      # 打印 GC 详细日志
-XX:+PrintGCTimeStamps   # 打印 GC 时间戳
-Xloggc:/tmp/gc.log      # GC 日志输出路径
