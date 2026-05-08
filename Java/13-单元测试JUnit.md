# Java 单元测试 JUnit

**单元测试** 是对代码中最小可测试单元（通常是一个方法）进行验证的过程。**JUnit** 是 Java 最流行的单元测试框架，几乎所有 Java 项目都在用。

## 一、为什么要写单元测试？

```java
// 你写了一个方法
public int add(int a, int b) {
    return a + b;
}

// 没测试：肉眼检查 → 改了代码 → 再肉眼检查 → 重复 N 遍
// 有测试：写一次断言 → 自动跑 → 永远帮你检查
```

**好处：**
- ✅ **防止回归**：改代码不怕弄坏旧功能
- ✅ **快速反馈**：秒级验证逻辑正确
- ✅ **倒逼好设计**：可测试的代码通常结构更清晰
- ✅ **活文档**：测试就是最直观的使用示例
- ✅ **重构信心**：有测试兜底敢重构

> 💡 **专业 Java 项目的测试代码量通常和业务代码 1:1 甚至更高。**

## 二、JUnit 5 引入

JUnit 5 = **JUnit Platform + JUnit Jupiter + JUnit Vintage**

我们日常用的就是 **Jupiter**。

### Maven 依赖

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

### 项目结构约定

```
src/
├── main/java/        ← 业务代码
│   └── com/example/Calculator.java
└── test/java/        ← 测试代码（同样的包结构）
    └── com/example/CalculatorTest.java
```

## 三、第一个 JUnit 测试

```java
// 业务类
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
}
```

```java
// 测试类
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class CalculatorTest {

    @Test
    void testAdd() {
        Calculator calc = new Calculator();
        int result = calc.add(2, 3);
        assertEquals(5, result);   // 期望值, 实际值
    }
}
```

**运行方式：** IDE 里右键测试类 → "Run"，绿色 ✅ 表示通过，红色 ❌ 表示失败。

## 四、JUnit 5 核心注解

| 注解            | 作用                       |
| ------------- | ------------------------ |
| `@Test`       | 标记测试方法                   |
| `@BeforeEach` | 每个测试方法**之前**执行（初始化）       |
| `@AfterEach`  | 每个测试方法**之后**执行（清理）        |
| `@BeforeAll`  | 所有测试**之前**执行**一次**（必须 static）|
| `@AfterAll`   | 所有测试**之后**执行**一次**（必须 static）|
| `@DisplayName`| 给测试取个好看的名字（支持中文）         |
| `@Disabled`   | 暂时跳过这个测试                |
| `@Nested`     | 嵌套测试类（按场景分组）            |
| `@Tag`        | 给测试打标签，可按标签运行            |

### 完整示例

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("计算器测试")
public class CalculatorTest {
    private Calculator calc;

    @BeforeAll
    static void setUpAll() {
        System.out.println("整个测试类开始");
    }

    @BeforeEach
    void setUp() {
        calc = new Calculator();   // 每个测试前都创建新实例
        System.out.println("一个测试开始");
    }

    @Test
    @DisplayName("加法应该正常工作")
    void testAdd() {
        assertEquals(5, calc.add(2, 3));
    }

    @Test
    @DisplayName("减法测试")
    void testSubtract() {
        assertEquals(1, calc.subtract(3, 2));
    }

    @Test
    @Disabled("功能还没实现，先跳过")
    void testDivide() {
        // ...
    }

    @AfterEach
    void tearDown() {
        System.out.println("一个测试结束");
    }

    @AfterAll
    static void tearDownAll() {
        System.out.println("整个测试类结束");
    }
}
```

## 五、断言（Assertions）—— 测试的核心

所有断言都来自 `org.junit.jupiter.api.Assertions`，建议**静态导入**：

```java
import static org.junit.jupiter.api.Assertions.*;
```

### 常用断言

```java
// 相等判断
assertEquals(5, calc.add(2, 3));
assertEquals(3.14, result, 0.001);   // 浮点数：第三个参数是允许误差

// 不相等
assertNotEquals(0, result);

// 真假判断
assertTrue(user.isActive());
assertFalse(list.isEmpty());

// 空值判断
assertNull(value);
assertNotNull(user);

// 引用判断
assertSame(obj1, obj2);     // 是同一个对象（==）
assertNotSame(obj1, obj2);

// 数组/集合
assertArrayEquals(new int[]{1, 2, 3}, result);
assertIterableEquals(List.of("a", "b"), result);

// 带消息的断言（失败时会显示）
assertEquals(5, result, "加法计算错了！");

// 直接让测试失败
fail("不应该走到这里");
```

### 异常断言（重点）

测试方法是否**正确抛出异常**：

```java
@Test
void testDivideByZero() {
    Calculator calc = new Calculator();

    // 期望抛出 ArithmeticException
    ArithmeticException ex = assertThrows(
        ArithmeticException.class,
        () -> calc.divide(10, 0)
    );

    // 还可以验证异常消息
    assertEquals("/ by zero", ex.getMessage());
}

// 断言不抛异常
@Test
void testNoException() {
    assertDoesNotThrow(() -> calc.add(1, 2));
}
```

### 多重断言（assertAll）

希望多个断言**全部执行**，即使中间某个失败：

```java
@Test
void testUser() {
    User u = new User("小林", 18);

    assertAll("用户属性",
        () -> assertEquals("小林", u.getName()),
        () -> assertEquals(18, u.getAge()),
        () -> assertNotNull(u.getId())
    );
    // 三个断言都会执行，失败的会一起报告
}
```

### 超时断言

```java
@Test
void testTimeout() {
    // 必须在 1 秒内完成
    assertTimeout(Duration.ofSeconds(1), () -> {
        Thread.sleep(500);
    });
}
```

## 六、参数化测试（强大！）

**用不同参数跑同一个测试方法。** 避免写一堆重复的 `@Test`。

### 引入依赖

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

### @ValueSource

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 4, 5})
void testIsPositive(int n) {
    assertTrue(n > 0);
}
// 自动跑 5 次，每次 n 不同
```

### @CsvSource

```java
@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "10, -5, 5"
})
void testAdd(int a, int b, int expected) {
    assertEquals(expected, calc.add(a, b));
}
```

### @CsvFileSource（从文件读）

```java
@ParameterizedTest
@CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
void testFromFile(int a, int b, int expected) {
    assertEquals(expected, calc.add(a, b));
}
```

### @MethodSource

```java
@ParameterizedTest
@MethodSource("provideTestData")
void testAdd(int a, int b, int expected) {
    assertEquals(expected, calc.add(a, b));
}

static Stream<Arguments> provideTestData() {
    return Stream.of(
        Arguments.of(1, 1, 2),
        Arguments.of(2, 3, 5),
        Arguments.of(-1, 1, 0)
    );
}
```

## 七、嵌套测试

**按场景把测试组织成"分组"**，可读性更好：

```java
@DisplayName("用户服务测试")
class UserServiceTest {
    UserService service = new UserService();

    @Nested
    @DisplayName("当用户已注册时")
    class WhenUserExists {

        @Test
        @DisplayName("登录应成功")
        void loginSuccess() { /* ... */ }

        @Test
        @DisplayName("不能重复注册")
        void registerFail() { /* ... */ }
    }

    @Nested
    @DisplayName("当用户未注册时")
    class WhenUserNotExists {

        @Test
        @DisplayName("登录应失败")
        void loginFail() { /* ... */ }
    }
}
```

## 八、Mock 测试 —— Mockito

实际业务里，一个类经常**依赖**别的类（比如 Service 依赖 DAO）。测试时不想真的连数据库，就需要"假对象"——这就是 **Mock**。

### 引入依赖

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

### 基本用法

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserDao userDao;     // 假的 DAO

    @InjectMocks
    private UserService service;  // 自动把上面的 mock 注入

    @Test
    void testFindById() {
        // ① 设置 mock 的行为：调用 findById(1) 时返回这个假用户
        User fakeUser = new User(1L, "假小林");
        when(userDao.findById(1L)).thenReturn(fakeUser);

        // ② 调用真实 service 方法
        User result = service.getUserById(1L);

        // ③ 验证结果
        assertEquals("假小林", result.getName());

        // ④ 验证 mock 被调用了
        verify(userDao).findById(1L);
        verify(userDao, times(1)).findById(anyLong());
    }

    @Test
    void testFindByIdNotFound() {
        when(userDao.findById(999L)).thenReturn(null);

        assertThrows(UserNotFoundException.class,
            () -> service.getUserById(999L));
    }

    @Test
    void testException() {
        // 让 mock 抛异常
        when(userDao.findById(anyLong()))
            .thenThrow(new RuntimeException("数据库炸了"));

        assertThrows(RuntimeException.class,
            () -> service.getUserById(1L));
    }
}
```

### Mockito 的核心套路

```
1. when(假对象.方法(参数)).thenReturn(假数据)    ← 设定行为
2. 调用真实代码
3. 断言结果
4. verify(假对象).方法(参数)                    ← 验证调用
```

## 九、覆盖率工具：JaCoCo

**测试覆盖率** 衡量你的测试覆盖了多少代码。

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

运行 `mvn test`，会在 `target/site/jacoco/index.html` 生成可视化报告。

> 💡 **覆盖率不是越高越好。** 70%~80% 是合理目标，盲目追求 100% 会写一堆没价值的测试。

## 十、好测试的特征：FIRST 原则

| 原则                | 含义                  |
| ----------------- | ------------------- |
| **F** Fast        | 快——单元测试要在毫秒级        |
| **I** Independent | 独立——测试之间不能相互依赖      |
| **R** Repeatable  | 可重复——任何环境跑结果一样      |
| **S** Self-validating | 自验证——明确通过/失败，不靠肉眼  |
| **T** Timely      | 及时——和业务代码同步写        |

## 十一、TDD 简介（测试驱动开发）

**TDD（Test-Driven Development）** 的工作流是"**红 → 绿 → 重构**"：

```
1. 红：先写一个失败的测试（功能还没实现）
2. 绿：写最简单的代码让测试通过
3. 重构：在测试保护下优化代码
   ↓
   循环
```

> 💡 不一定每个项目都要严格 TDD，但**写代码之前先想"怎么测"** 是非常好的习惯。

## 十二、最佳实践

✅ **该做的：**
- 测试方法名要**清晰描述场景**，比如 `should_returnNull_when_userNotFound`
- 一个测试只验证**一件事**，失败原因清晰
- 用 `@DisplayName` 写中文说明，让报告好看
- **测试要先于或同步于业务代码**编写
- 边界情况和异常场景要测

❌ **不该做的：**
- 不要测试 getter/setter（没意义）
- 不要在测试里写复杂逻辑（测试本身要简单）
- 不要让测试依赖具体的执行顺序
- 不要在测试里依赖**真实**的数据库、网络、时间——用 Mock
- 不要忽略失败的测试（要么修，要么删）

## 十三、命名风格推荐

```java
// 风格 1：should_xxx_when_xxx
void should_returnUser_when_idExists() { }
void should_throwException_when_idIsNegative() { }

// 风格 2：methodName_条件_期望
void findById_validId_returnsUser() { }
void findById_invalidId_throwsException() { }

// 风格 3：中文（小项目/学习项目可以用）
@DisplayName("根据 ID 查询用户：ID 存在时应返回用户")
void test1() { }
```

---

📝 *持续更新中……*
