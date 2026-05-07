# Lambda 与 Stream API

Java 8 引入的两大利器，让代码更简洁、更优雅。

## 一、Lambda 表达式

### 什么是 Lambda？

**Lambda 是一个匿名函数**，可以作为参数传递。本质是简化"函数式接口"的实现。

### 基本语法

```
(参数列表) -> { 方法体 }
```

### 三步演化：从匿名内部类到 Lambda

```java
// ① 传统方式：匿名内部类
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};

// ② Lambda 写法
Runnable r2 = () -> {
    System.out.println("Hello");
};

// ③ 单行方法体可以省略大括号
Runnable r3 = () -> System.out.println("Hello");
```

### 常见写法

```java
// 无参数
() -> System.out.println("Hi")

// 一个参数（参数类型可省略，括号也可省略）
x -> x * 2

// 多个参数
(a, b) -> a + b

// 多行方法体（必须加大括号和 return）
(a, b) -> {
    int sum = a + b;
    return sum;
}
```

## 二、函数式接口

**只有一个抽象方法的接口**叫函数式接口，可以用 Lambda 实现。

```java
@FunctionalInterface          // 注解，表示这是函数式接口（可选但推荐）
public interface Calculator {
    int calc(int a, int b);   // 唯一的抽象方法
}

// 使用 Lambda 实现
Calculator add = (a, b) -> a + b;
Calculator sub = (a, b) -> a - b;

System.out.println(add.calc(3, 5));   // 8
System.out.println(sub.calc(10, 4));  // 6
```

### Java 内置的常用函数式接口

| 接口            | 抽象方法           | 用途              |
| ------------- | -------------- | --------------- |
| `Runnable`    | `void run()`   | 无参无返回值          |
| `Supplier<T>` | `T get()`      | 无参，返回 T（"提供者"）  |
| `Consumer<T>` | `void accept(T)` | 接受 T，无返回（"消费者"）|
| `Function<T,R>` | `R apply(T)` | T → R（"转换器"）   |
| `Predicate<T>`| `boolean test(T)` | 返回布尔值（"判断"）  |
| `BiFunction<T,U,R>` | `R apply(T,U)` | 两个参数转换为 R   |

```java
import java.util.function.*;

Supplier<String> supplier = () -> "Hello";
System.out.println(supplier.get());          // Hello

Consumer<String> consumer = s -> System.out.println("收到：" + s);
consumer.accept("快递");                       // 收到：快递

Function<Integer, String> func = n -> "数字是 " + n;
System.out.println(func.apply(42));          // 数字是 42

Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4));          // true
```

## 三、方法引用

如果 Lambda 只是**调用一个已有的方法**，可以用 `::` 进一步简化。

```java
// Lambda
Consumer<String> c1 = s -> System.out.println(s);

// 方法引用（等价）
Consumer<String> c2 = System.out::println;
```

### 四种方法引用

```java
// ① 静态方法引用
Function<Integer, String> f1 = String::valueOf;
// 等价于 n -> String.valueOf(n)

// ② 实例方法引用（特定对象）
String prefix = "Mr.";
Function<String, String> f2 = prefix::concat;
// 等价于 s -> prefix.concat(s)

// ③ 实例方法引用（任意对象）
Function<String, Integer> f3 = String::length;
// 等价于 s -> s.length()

// ④ 构造方法引用
Supplier<ArrayList<String>> s1 = ArrayList::new;
// 等价于 () -> new ArrayList<>()
```

## 四、Stream API

**Stream 是对集合的"流式操作"**，可以像流水线一样对数据进行过滤、转换、聚合。

### 核心理念

```
集合 → Stream → 中间操作（filter/map/...）→ 终止操作（collect/count/...）→ 结果
```

### 创建 Stream

```java
import java.util.stream.Stream;

// 从集合创建
List<String> list = List.of("a", "b", "c");
Stream<String> s1 = list.stream();

// 直接创建
Stream<String> s2 = Stream.of("a", "b", "c");

// 从数组
int[] arr = {1, 2, 3};
IntStream s3 = Arrays.stream(arr);
```

## 五、中间操作（返回新 Stream）

### filter —— 过滤

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

List<Integer> evens = nums.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
// [2, 4]
```

### map —— 转换

```java
List<String> names = List.of("alice", "bob", "charlie");

List<String> upper = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// [ALICE, BOB, CHARLIE]

// 转换类型
List<Integer> lengths = names.stream()
    .map(String::length)
    .collect(Collectors.toList());
// [5, 3, 7]
```

### sorted —— 排序

```java
List<Integer> nums = List.of(3, 1, 4, 1, 5, 9, 2, 6);

List<Integer> sorted = nums.stream()
    .sorted()
    .collect(Collectors.toList());
// [1, 1, 2, 3, 4, 5, 6, 9]

// 自定义排序（倒序）
List<Integer> desc = nums.stream()
    .sorted((a, b) -> b - a)
    .collect(Collectors.toList());
```

### distinct —— 去重

```java
List<Integer> nums = List.of(1, 2, 2, 3, 3, 3);

List<Integer> unique = nums.stream()
    .distinct()
    .collect(Collectors.toList());
// [1, 2, 3]
```

### limit / skip —— 截取

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

List<Integer> first3 = nums.stream()
    .limit(3)
    .collect(Collectors.toList());
// [1, 2, 3]

List<Integer> after2 = nums.stream()
    .skip(2)
    .collect(Collectors.toList());
// [3, 4, 5]
```

## 六、终止操作（产生结果）

### collect —— 收集到集合

```java
// 收集为 List
List<Integer> list = nums.stream().collect(Collectors.toList());

// 收集为 Set
Set<Integer> set = nums.stream().collect(Collectors.toSet());

// 收集为 Map
Map<String, Integer> map = list.stream()
    .collect(Collectors.toMap(
        s -> s,           // key
        String::length    // value
    ));

// 拼接成字符串
String joined = List.of("a", "b", "c").stream()
    .collect(Collectors.joining(", "));
// "a, b, c"
```

### count / sum / max / min

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

long count = nums.stream().count();                          // 5
int sum = nums.stream().mapToInt(Integer::intValue).sum();   // 15
OptionalInt max = nums.stream().mapToInt(Integer::intValue).max();  // 5
double avg = nums.stream().mapToInt(Integer::intValue).average().orElse(0); // 3.0
```

### forEach —— 遍历

```java
List.of("a", "b", "c").stream()
    .forEach(System.out::println);
```

### anyMatch / allMatch / noneMatch

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

boolean hasEven = nums.stream().anyMatch(n -> n % 2 == 0);   // true
boolean allPositive = nums.stream().allMatch(n -> n > 0);    // true
boolean noneNegative = nums.stream().noneMatch(n -> n < 0);  // true
```

### findFirst / findAny

```java
Optional<Integer> first = nums.stream()
    .filter(n -> n > 3)
    .findFirst();

if (first.isPresent()) {
    System.out.println(first.get());   // 4
}
```

## 七、分组与分区（重点）

### groupingBy —— 分组

```java
class Person {
    String name;
    String city;
    int age;
    // 构造方法、getter 省略
}

List<Person> people = List.of(
    new Person("Alice", "北京", 25),
    new Person("Bob", "上海", 30),
    new Person("Charlie", "北京", 28)
);

// 按城市分组
Map<String, List<Person>> byCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity));
// { "北京": [Alice, Charlie], "上海": [Bob] }

// 按城市统计人数
Map<String, Long> countByCity = people.stream()
    .collect(Collectors.groupingBy(
        Person::getCity,
        Collectors.counting()
    ));
// { "北京": 2, "上海": 1 }
```

### partitioningBy —— 分区（按布尔值分两组）

```java
Map<Boolean, List<Integer>> parts = nums.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// { true: [2, 4], false: [1, 3, 5] }
```

## 八、综合实战

```java
class Order {
    String customer;
    String product;
    double amount;
    // ...
}

List<Order> orders = ...;   // 一堆订单

// 需求：统计每个客户的总消费金额，并按金额从高到低排序，取前 3
Map<String, Double> top3 = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCustomer,
        Collectors.summingDouble(Order::getAmount)
    ))
    .entrySet().stream()
    .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
    .limit(3)
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        Map.Entry::getValue,
        (a, b) -> a,
        LinkedHashMap::new   // 保持顺序
    ));
```

> 这种业务场景如果用传统 for 循环写，至少 30 行；用 Stream 一气呵成。

## 九、注意事项

### 1. Stream 只能消费一次

```java
Stream<Integer> s = Stream.of(1, 2, 3);
s.forEach(System.out::println);   // ✅ 第一次 OK
s.forEach(System.out::println);   // ❌ IllegalStateException
```

### 2. Stream 操作是惰性的

```java
List.of(1, 2, 3).stream()
    .filter(n -> {
        System.out.println("过滤 " + n);
        return n > 1;
    });
// 没有任何输出！因为没有终止操作，中间操作不会真正执行。
```

### 3. 并行流（parallelStream）

```java
// 大数据量时可以用并行流加速
long count = bigList.parallelStream()
    .filter(x -> x > 100)
    .count();
```

> ⚠️ 但小数据量用并行流反而更慢（线程开销）；处理共享状态时还要注意线程安全。

### 4. Optional 处理可能为空的结果

```java
Optional<Integer> first = nums.stream()
    .filter(n -> n > 100)
    .findFirst();

// 用法：
first.ifPresent(System.out::println);              // 存在才执行
int val = first.orElse(0);                          // 不存在则返回默认值
int val2 = first.orElseThrow(() -> new RuntimeException("没找到"));
```

## 十、Stream vs 传统 for

| 对比项     | for 循环         | Stream             |
| ------- | -------------- | ------------------ |
| 代码量     | 多              | 少                  |
| 可读性     | 关注"怎么做"        | 关注"做什么"            |
| 性能      | 略快（小数据量）       | 略慢（小数据量），并行流大数据量更快 |
| 调试      | 容易             | 较难（链式调用）           |
| 适用场景    | 简单遍历、需要中途 break | 数据转换、过滤、聚合、分组      |

> 💡 **简单遍历用 for，数据处理流水线用 Stream**，按需选择。

---

📝 *持续更新中……*
