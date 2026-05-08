# JVM 基础

**JVM（Java Virtual Machine）** 是 Java 跨平台的核心——一次编译，到处运行。理解 JVM 是从"会写 Java"进阶到"懂 Java"的分水岭。

## 一、Java 程序的执行流程

```
源代码 .java
   ↓ 编译（javac）
字节码 .class  ←—— 跨平台的关键，只要有 JVM 就能跑
   ↓ JVM 加载
类加载器 ClassLoader
   ↓
运行时数据区（内存）
   ↓
执行引擎（解释器 + JIT 编译器）
   ↓
机器码 → CPU 执行
```

## 二、JVM 内存结构（运行时数据区）

```
┌─────────────────────────────────────────┐
│              方法区（元空间）              │  ← 线程共享
│   类信息、常量、静态变量、JIT 编译后的代码      │
├─────────────────────────────────────────┤
│                  堆（Heap）              │  ← 线程共享
│           几乎所有对象都在这里分配             │
├──────────────┬──────────────────────────┤
│   虚拟机栈    │   本地方法栈   │  程序计数器  │  ← 线程私有
│ (每个方法栈帧) │  (native 方法) │ (字节码地址) │
└──────────────┴──────────────────────────┘
```

### 1. 程序计数器（PC Register）

- **线程私有**，每个线程一份
- 记录当前线程**正在执行的字节码地址**
- 唯一**不会 OOM** 的区域

### 2. 虚拟机栈（JVM Stack）

- **线程私有**，每个线程一份
- 每个**方法**调用对应一个**栈帧**（Stack Frame）
- 栈帧里存：**局部变量表、操作数栈、动态链接、返回地址**
- 异常：`StackOverflowError`（栈深度过大）、`OutOfMemoryError`

```java
public void a() {
    a();   // 无限递归 → StackOverflowError
}
```

### 3. 本地方法栈

和虚拟机栈类似，但用于执行 **native 方法**（如 `Object.hashCode()`）。

### 4. 堆（Heap）—— 最重要

- **线程共享**，JVM 启动时创建
- **几乎所有对象都在堆上分配**
- 是 GC 的主战场
- 异常：`OutOfMemoryError: Java heap space`

```java
Object o = new Object();   // o 在栈，new Object() 在堆
```

### 5. 方法区（Method Area）

- **线程共享**
- 存储**类信息**、**常量**、**静态变量**、JIT 编译后的代码
- JDK 7 之前叫"永久代（PermGen）"，JDK 8 之后改为"**元空间（Metaspace）**"，使用本地内存
- 异常：`OutOfMemoryError: Metaspace`

## 三、堆的内部结构

```
┌────────────────────── 堆 ──────────────────────┐
│                                                │
│  ┌─────── 新生代（Young Gen）─────┐              │
│  │  Eden  │  Survivor 0 │ Survivor 1 │          │
│  │ (8/10) │   (1/10)    │  (1/10)    │          │
│  └─────────────────────────────────┘            │
│                                                │
│  ┌────────── 老年代（Old Gen）────────┐         │
│  │                                    │         │
│  └────────────────────────────────────┘         │
└────────────────────────────────────────────────┘
```

**对象的一生：**

1. **新对象** → 分配到 **Eden 区**
2. Eden 满了 → 触发 **Minor GC**，存活的对象 → Survivor 0
3. 再次 GC → Survivor 0 的存活对象 → Survivor 1（每次年龄 +1）
4. 年龄达到阈值（默认 15）→ 晋升到**老年代**
5. 老年代满了 → 触发 **Full GC**

## 四、对象的创建过程

```java
Person p = new Person("小林");
```

JVM 干了这些事：
1. **类加载检查**：Person 类是否已加载？没有就先加载
2. **分配内存**：在堆中划一块空间
3. **初始化零值**：字段设为默认值（int = 0，String = null）
4. **设置对象头**：哈希码、GC 分代年龄等元信息
5. **执行构造方法**：把 "小林" 赋给 name

## 五、垃圾回收（GC）

### 怎么判断对象是"垃圾"？

#### 方式 1：引用计数（Java 不用这个）

每个对象记录被引用次数，归 0 就是垃圾。**问题：处理不了循环引用**。

```java
A a = new A();
B b = new B();
a.ref = b;
b.ref = a;
a = null; b = null;
// 引用计数法认为还活着，其实已经无法访问 → 内存泄漏
```

#### 方式 2：可达性分析（Java 实际使用）

从一组 **GC Root** 出发，能到达的对象就是"活的"，到不了的就是"垃圾"。

**GC Root 包括：**
- 栈中的局部变量
- 静态变量
- 常量
- JNI 引用

```
GC Root → A → B → C   ← 活的
         ↓
         D → E         ← 活的

   X → Y → Z           ← 全是垃圾（GC Root 到不了）
```

### 引用的四种类型

| 类型     | 关键字                | 何时回收            | 用途              |
| ------ | ------------------ | --------------- | --------------- |
| 强引用    | 直接 `=` 赋值          | 永不回收（OOM 也不回收） | 最常用             |
| 软引用    | `SoftReference`    | 内存不足时回收         | 缓存              |
| 弱引用    | `WeakReference`    | 下次 GC 就回收       | `ThreadLocal`、缓存|
| 虚引用    | `PhantomReference` | 任何时候都可能回收       | 跟踪对象回收          |

## 六、垃圾回收算法

### 1. 标记-清除（Mark-Sweep）
- 标记垃圾，再统一清除
- 缺点：**产生内存碎片**

### 2. 复制（Copying）
- 把内存分两半，存活对象复制到另一半，然后清空当前半
- 优点：无碎片
- 缺点：浪费一半空间
- 应用：**新生代**（Eden + Survivor）

### 3. 标记-整理（Mark-Compact）
- 标记后，存活对象向一端移动，然后清理边界外的内存
- 优点：无碎片
- 应用：**老年代**

### 4. 分代收集（实际使用的策略）
- 新生代用**复制算法**（对象朝生夕死，存活少，复制开销小）
- 老年代用**标记-整理**（对象存活率高）

## 七、常见垃圾收集器

| 收集器      | 区域   | 特点                       |
| -------- | ---- | ------------------------ |
| Serial   | 新生代  | 单线程，简单                   |
| ParNew   | 新生代  | Serial 的多线程版             |
| Parallel | 新老都有 | 关注吞吐量（JDK 8 默认）          |
| CMS      | 老年代  | 关注低延迟，已废弃                |
| **G1**   | 整个堆  | 平衡吞吐与延迟（JDK 9+ 默认）       |
| ZGC/Shenandoah | 整个堆 | 超低延迟，亚毫秒级停顿（JDK 11+）|

## 八、Minor GC vs Full GC

| 类型        | 触发时机              | 影响        |
| --------- | ----------------- | --------- |
| Minor GC  | 新生代满了             | 停顿短，频繁    |
| Major GC  | 老年代回收             | 停顿较长      |
| **Full GC** | 老年代满 / 元空间满 / 手动 | **停顿最长**，要尽量避免 |

> 💡 **STW（Stop The World）**：GC 时所有用户线程暂停，这就是为什么 Java 服务器有时候会"卡一下"的原因。

## 九、类加载机制

### 类加载的 5 个阶段

```
加载（Loading）→ 验证（Verification）→ 准备（Preparation）
   → 解析（Resolution）→ 初始化（Initialization）
```

- **加载**：把字节码读入内存，生成 Class 对象
- **验证**：检查字节码合法性
- **准备**：给静态变量分配内存并赋默认值
- **解析**：把符号引用转为直接引用
- **初始化**：执行 `static` 代码块和静态变量赋值

### 类加载器

```
Bootstrap ClassLoader（启动类加载器，C++ 实现）
    ↓ 加载 jre/lib 下的核心类
Extension ClassLoader（扩展类加载器）
    ↓ 加载 jre/lib/ext 下的类
Application ClassLoader（应用类加载器）
    ↓ 加载 classpath 下的类
自定义 ClassLoader
```

### 双亲委派模型

**子加载器先委托父加载器加载，父加载器找不到才自己加载。**

**好处：**
- 避免类重复加载
- 保护核心类（你写的 `java.lang.String` 永远不会替换 JDK 自带的）

```java
// 你自己写一个 java.lang.String，是没用的
package java.lang;
public class String {
    public static void main(String[] args) {
        System.out.println("我替换 JDK 的 String！");
    }
}
// 运行报错：找不到 main 方法（因为加载的是 JDK 的 String）
```

## 十、常见 JVM 参数

```bash
# 堆大小
-Xms512m              # 初始堆大小
-Xmx2g                # 最大堆大小（一般两者设成一样，避免动态调整）

# 新生代/老年代比例
-Xmn256m              # 新生代大小
-XX:NewRatio=2        # 老:新 = 2:1

# Survivor 比例
-XX:SurvivorRatio=8   # Eden:S0:S1 = 8:1:1（默认）

# 元空间
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=256m

# 选择 GC 收集器
-XX:+UseG1GC          # 使用 G1
-XX:+UseZGC           # 使用 ZGC

# GC 日志
-Xlog:gc*             # JDK 9+
-XX:+PrintGCDetails   # JDK 8

# OOM 时导出堆
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dump.hprof
```

## 十一、常见 OOM 类型

| 异常                                            | 原因         | 解决                   |
| --------------------------------------------- | ---------- | -------------------- |
| `java.lang.OutOfMemoryError: Java heap space` | 堆内存不够      | 增大 `-Xmx`，排查内存泄漏     |
| `OutOfMemoryError: Metaspace`                 | 元空间不够      | 增大 `-XX:MaxMetaspaceSize`，检查动态生成类 |
| `OutOfMemoryError: GC overhead limit exceeded`| GC 占用太多 CPU | 一般是堆太小，需要扩容          |
| `StackOverflowError`                          | 栈溢出（递归过深）  | 检查递归终止条件             |

## 十二、JVM 调优工具

| 工具         | 用途                      |
| ---------- | ----------------------- |
| `jps`      | 查看 Java 进程              |
| `jstat`    | 查看 GC 统计信息              |
| `jmap`     | 导出堆 dump 文件             |
| `jstack`   | 导出线程栈（排查死锁、卡顿）          |
| `jconsole` | 图形化监控                   |
| `JVisualVM`| 综合分析工具                  |
| `Arthas`   | 阿里开源的诊断工具（实战神器）         |
| `MAT`      | 分析堆 dump，排查内存泄漏         |

```bash
# 实战示例
jps -l                          # 列出所有 Java 进程
jstat -gc <pid> 1000            # 每秒打印 GC 信息
jmap -dump:format=b,file=heap.hprof <pid>   # 导出堆
jstack <pid>                    # 打印线程栈
```

## 十三、面试高频问题

1. **JVM 内存结构？** → 五大区域（堆、栈、方法区、本地方法栈、PC）
2. **堆怎么分代？** → 新生代（Eden + 2 Survivor）+ 老年代
3. **怎么判断对象是垃圾？** → 可达性分析 + GC Root
4. **GC 算法？** → 标记清除、复制、标记整理、分代收集
5. **Minor GC 和 Full GC 区别？**
6. **类加载过程？** → 加载、验证、准备、解析、初始化
7. **双亲委派？为什么要双亲委派？**
8. **常见 OOM 场景？**

---

📝 *持续更新中……*
