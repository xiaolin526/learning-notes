# MyBatis-Plus

**MyBatis-Plus（简称 MP）** 是 MyBatis 的**增强工具**——在保留 MyBatis 所有功能的基础上，提供了大量开箱即用的 CRUD 操作，让你**只需定义接口，不用写 SQL**。

国内中小公司后端项目基本上**首选 MyBatis-Plus**。

## 一、先简单认识 MyBatis

### MyBatis 是什么？

MyBatis 是一个**半自动 ORM 框架**——你写 SQL，它负责把 SQL 结果**自动映射成 Java 对象**。

```java
// 你写一个 Mapper 接口
public interface UserMapper {
    User findById(Long id);
}

// 配合 SQL（XML 或注解）
@Select("SELECT * FROM user WHERE id = #{id}")
User findById(Long id);

// 直接调用，自动执行 SQL + 映射结果
User u = userMapper.findById(1L);
```

### MyBatis vs JDBC

| 对比     | JDBC                  | MyBatis             |
| ------ | --------------------- | ------------------- |
| 代码量    | 多（手写连接、结果映射）          | 少                   |
| SQL 控制 | 完全自己写                 | 完全自己写（灵活）          |
| 结果映射   | 手动 `rs.getXxx()`     | 自动映射到对象             |
| 学习成本   | 低                     | 中                   |

### MyBatis vs MyBatis-Plus

| 对比          | MyBatis           | MyBatis-Plus            |
| ----------- | ----------------- | ----------------------- |
| 简单 CRUD     | 要写 SQL            | **零 SQL**（继承 BaseMapper）|
| 复杂 SQL      | 灵活                | 也支持，并提供 Wrapper 条件构造器 |
| 分页          | 自己写或装插件           | **内置分页插件**             |
| 代码生成        | 无                 | **官方代码生成器**            |
| 学习成本        | 中                 | 略高（要懂 MyBatis 基础）       |

> 💡 **MyBatis-Plus 完全兼容 MyBatis**，你随时可以混用。

## 二、引入 MyBatis-Plus

### Spring Boot 项目（最常见）

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.7</version>
</dependency>

<!-- 数据库驱动 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
</dependency>
```

> ⚠️ **Spring Boot 2.x** 用 `mybatis-plus-boot-starter`；**Spring Boot 3.x** 用 `mybatis-plus-spring-boot3-starter`。版本一定要对应。

### 配置数据库连接

`application.yml`：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb?serverTimezone=UTC&useSSL=false
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl   # 打印 SQL（开发用）
    map-underscore-to-camel-case: true                       # 下划线转驼峰（默认开启）
  global-config:
    db-config:
      id-type: auto          # 主键策略：自增
      logic-delete-field: deleted   # 逻辑删除字段名
      logic-delete-value: 1
      logic-not-delete-value: 0
```

## 三、第一个 MP 程序

### 1. 数据库准备

```sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    age INT,
    email VARCHAR(100),
    create_time DATETIME,
    deleted TINYINT DEFAULT 0
);
```

### 2. 实体类

```java
import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@TableName("user")    // 表名（如果类名和表名一致可省略）
public class User {

    @TableId(type = IdType.AUTO)    // 主键自增
    private Long id;

    private String name;
    private Integer age;
    private String email;

    @TableField(fill = FieldFill.INSERT)   // 插入时自动填充
    private LocalDateTime createTime;

    @TableLogic    // 逻辑删除标记
    private Integer deleted;
}
```

### 3. Mapper 接口（核心！）

```java
import com.baomidou.mybatisplus.core.mapper.BaseMapper;

public interface UserMapper extends BaseMapper<User> {
    // 不用写任何方法，BaseMapper 自带 CRUD
}
```

### 4. 启动类加扫描

```java
@SpringBootApplication
@MapperScan("com.example.mapper")   // 扫描 Mapper 接口
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 5. 直接用！

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public void demo() {
        // 插入
        User u = new User();
        u.setName("小林");
        u.setAge(18);
        userMapper.insert(u);
        System.out.println("生成的 ID = " + u.getId());   // 自动回填

        // 查询单条
        User user = userMapper.selectById(1L);

        // 查询全部
        List<User> all = userMapper.selectList(null);

        // 更新
        user.setAge(20);
        userMapper.updateById(user);

        // 删除
        userMapper.deleteById(1L);
    }
}
```

> 🎉 **没写一行 SQL，CRUD 全搞定。** 这就是 MyBatis-Plus 的魅力。

## 四、BaseMapper 内置方法清单

| 方法                         | 作用            |
| -------------------------- | ------------- |
| `insert(T)`                | 插入一条          |
| `deleteById(id)`           | 按 ID 删除       |
| `deleteByMap(map)`         | 按 Map 条件删除    |
| `delete(wrapper)`          | 按 Wrapper 删除  |
| `deleteBatchIds(ids)`      | 批量删除          |
| `updateById(T)`            | 按 ID 更新       |
| `update(T, wrapper)`       | 按 Wrapper 更新  |
| `selectById(id)`           | 按 ID 查        |
| `selectBatchIds(ids)`      | 批量查           |
| `selectByMap(map)`         | 按 Map 条件查     |
| `selectOne(wrapper)`       | 查一条           |
| `selectCount(wrapper)`     | 查总数           |
| `selectList(wrapper)`      | 查列表           |
| `selectPage(page, wrapper)`| 分页查           |

## 五、条件构造器 Wrapper（核心！）

复杂查询用 **QueryWrapper / LambdaQueryWrapper**：

### QueryWrapper（字符串字段名）

```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("name", "小林")
       .gt("age", 18)
       .like("email", "@example.com")
       .orderByDesc("create_time");

List<User> list = userMapper.selectList(wrapper);
// SELECT * FROM user
// WHERE name = '小林' AND age > 18 AND email LIKE '%@example.com%'
// ORDER BY create_time DESC
```

### LambdaQueryWrapper（推荐！类型安全）

字段名通过方法引用，**编译期检查**，重命名字段也不会出错：

```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getName, "小林")
       .gt(User::getAge, 18)
       .like(User::getEmail, "@example.com")
       .orderByDesc(User::getCreateTime);

List<User> list = userMapper.selectList(wrapper);
```

### Wrapper 常用方法

| 方法                     | SQL                       |
| ---------------------- | ------------------------- |
| `eq(col, val)`         | `col = val`               |
| `ne(col, val)`         | `col != val`              |
| `gt / ge / lt / le`    | `>, >=, <, <=`            |
| `between(col, a, b)`   | `col BETWEEN a AND b`     |
| `like(col, val)`       | `col LIKE '%val%'`        |
| `likeLeft / likeRight` | `LIKE '%val'` / `'val%'`  |
| `in(col, list)`        | `col IN (...)`            |
| `notIn`                | `col NOT IN`              |
| `isNull / isNotNull`   | `IS NULL` / `IS NOT NULL` |
| `and / or`             | 条件组合                      |
| `groupBy / having`     | 分组                        |
| `orderByAsc / orderByDesc` | 排序                  |
| `select("col1","col2")`| 指定查询列                     |
| `last("LIMIT 5")`      | 拼接末尾 SQL                  |

### 条件 or 组合

```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getName, "小林")
       .and(w -> w.gt(User::getAge, 18).or().eq(User::getEmail, "vip@x.com"));
// WHERE name = '小林' AND (age > 18 OR email = 'vip@x.com')
```

### 动态条件（实战常用）

```java
public List<User> search(String name, Integer minAge) {
    return userMapper.selectList(
        new LambdaQueryWrapper<User>()
            .like(StringUtils.hasText(name), User::getName, name)
            .ge(minAge != null, User::getAge, minAge)
    );
    // StringUtils.hasText(name) 为 true 才加这个条件
}
```

> 💡 **第一个参数是 boolean 条件**——为 false 就跳过这个查询条件。**这是动态拼 SQL 最优雅的写法。**

## 六、分页插件

### 1. 配置分页拦截器

```java
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import com.baomidou.mybatisplus.annotation.DbType;
import org.springframework.context.annotation.*;

@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

### 2. 使用分页查询

```java
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;

// 第 1 页，每页 10 条
Page<User> page = new Page<>(1, 10);

// 加查询条件
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.gt(User::getAge, 18);

Page<User> result = userMapper.selectPage(page, wrapper);

System.out.println("总记录数：" + result.getTotal());
System.out.println("总页数：" + result.getPages());
System.out.println("当前页数据：" + result.getRecords());
```

## 七、Service 层封装（IService）

MyBatis-Plus 还提供了 **Service 层的封装**，进一步减少代码。

### 1. 定义 Service 接口

```java
import com.baomidou.mybatisplus.extension.service.IService;

public interface UserService extends IService<User> {
    // 自定义业务方法
}
```

### 2. 实现类

```java
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;

@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    // 不用写代码，CRUD 全自动
}
```

### 3. 使用

```java
@Autowired
private UserService userService;

userService.save(user);                // 插入
userService.removeById(1L);            // 删除
userService.updateById(user);          // 更新
userService.getById(1L);               // 查询
userService.list();                    // 查列表
userService.page(new Page<>(1, 10));   // 分页
userService.count();                   // 计数

// 批量操作（自动开启事务）
userService.saveBatch(userList);
userService.saveOrUpdateBatch(userList);

// 链式调用（很强大！）
List<User> users = userService.lambdaQuery()
    .gt(User::getAge, 18)
    .like(User::getName, "林")
    .orderByDesc(User::getCreateTime)
    .list();

userService.lambdaUpdate()
    .set(User::getAge, 20)
    .eq(User::getName, "小林")
    .update();
```

## 八、自动填充（@TableField fill）

**插入/更新时自动填充字段**，比如创建时间、更新时间、操作人。

### 1. 实体加注解

```java
@Data
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;

    @TableField(fill = FieldFill.INSERT)        // 仅插入时填充
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE) // 插入和更新都填充
    private LocalDateTime updateTime;
}
```

### 2. 实现填充器

```java
import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;

@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

之后任何 `insert` / `update` 都会**自动设置时间**，业务代码不用管。

## 九、逻辑删除

**不真删数据**，只把 `deleted` 字段从 0 改为 1。安全 + 可恢复。

### 1. 实体加注解

```java
@TableLogic
private Integer deleted;
```

### 2. 配置默认值

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-not-delete-value: 0
      logic-delete-value: 1
```

### 3. 效果

```java
userMapper.deleteById(1L);
// 实际执行：UPDATE user SET deleted = 1 WHERE id = 1
// 而不是：DELETE FROM user WHERE id = 1

userMapper.selectList(null);
// 自动加条件：WHERE deleted = 0
```

> 💡 **所有查询都会自动过滤已删除数据**，业务代码完全无感。

## 十、乐观锁

并发更新场景下，通过 **版本号** 防止数据被覆盖。

### 1. 表加 version 字段

```sql
ALTER TABLE user ADD version INT DEFAULT 0;
```

### 2. 实体加注解

```java
@Version
private Integer version;
```

### 3. 配置插件

```java
interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
```

### 4. 效果

```java
User u = userMapper.selectById(1L);   // version = 0
u.setAge(20);
userMapper.updateById(u);
// 实际执行：UPDATE user SET age = 20, version = 1 WHERE id = 1 AND version = 0
// 如果中间被别人改过（version 变成 1），就更新失败
```

## 十一、自定义 SQL

简单 CRUD 用 BaseMapper，**复杂 SQL** 还是要自己写。

### 方式 1：注解

```java
public interface UserMapper extends BaseMapper<User> {

    @Select("SELECT * FROM user WHERE age > #{age} AND name LIKE CONCAT('%', #{name}, '%')")
    List<User> searchByAgeAndName(@Param("age") int age, @Param("name") String name);
}
```

### 方式 2：XML（推荐复杂 SQL）

`resources/mapper/UserMapper.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.UserMapper">

    <select id="searchByAgeAndName" resultType="com.example.entity.User">
        SELECT * FROM user
        <where>
            <if test="age != null">
                AND age &gt; #{age}
            </if>
            <if test="name != null and name != ''">
                AND name LIKE CONCAT('%', #{name}, '%')
            </if>
        </where>
    </select>

</mapper>
```

接口：
```java
List<User> searchByAgeAndName(@Param("age") Integer age, @Param("name") String name);
```

配置 mapper 位置：
```yaml
mybatis-plus:
  mapper-locations: classpath*:mapper/**/*.xml
```

### 动态 SQL 常用标签

| 标签              | 作用     |
| --------------- | ------ |
| `<if>`          | 条件判断   |
| `<where>`       | 自动加 WHERE，去掉前置 AND/OR |
| `<set>`         | UPDATE 时自动加 SET，去掉尾逗号 |
| `<foreach>`     | 循环（IN 查询常用） |
| `<choose/when/otherwise>` | switch-case |

```xml
<select id="findByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</select>
```

## 十二、代码生成器（神器！）

**根据数据库表自动生成 Entity / Mapper / Service / Controller**，省下大量样板代码。

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.7</version>
</dependency>
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity-engine-core</artifactId>
    <version>2.3</version>
</dependency>
```

```java
public class CodeGenerator {
    public static void main(String[] args) {
        FastAutoGenerator.create(
                "jdbc:mysql://localhost:3306/testdb",
                "root",
                "123456")
            .globalConfig(builder -> builder
                .author("xiaolin")
                .outputDir("./src/main/java"))
            .packageConfig(builder -> builder
                .parent("com.example")
                .entity("entity")
                .mapper("mapper")
                .service("service")
                .controller("controller"))
            .strategyConfig(builder -> builder
                .addInclude("user", "order"))   // 要生成的表
            .execute();
    }
}
```

跑一次，整套 CRUD 代码全部生成好。

## 十三、最佳实践

### ✅ 该做的

- **优先用 LambdaQueryWrapper**，字段重命名也不怕
- **复杂查询写 XML**，简单 CRUD 用 BaseMapper
- **配置阶段就开 SQL 日志**，看实际执行的 SQL
- **逻辑删除替代物理删除**，保留数据
- **批量操作用 saveBatch / updateBatchById**，性能更好
- **分页插件必装**，否则手写分页 SQL 容易错

### ❌ 不该做的

- 不要在生产环境开**全表查**（`selectList(null)` + 上千万数据 = 灾难）
- 不要忽略 **Wrapper 的动态条件参数**（容易少加判断导致全表查）
- 不要 N+1 查询（循环里调 `selectById`）
- 不要忘记 **N+1 用 join 或 IN 优化**
- 不要把业务逻辑写到 SQL 里（难维护）

## 十四、和 Spring Boot 整合的典型分层

```
Controller    （接收请求）
    ↓
Service       （业务逻辑，调用多个 Mapper）
    ↓
Mapper        （数据访问，继承 BaseMapper）
    ↓
MySQL
```

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public User getById(@PathVariable Long id) {
        return userService.getById(id);
    }

    @GetMapping
    public Page<User> page(@RequestParam(defaultValue = "1") int pageNum,
                           @RequestParam(defaultValue = "10") int pageSize) {
        return userService.page(new Page<>(pageNum, pageSize));
    }

    @PostMapping
    public boolean create(@RequestBody User user) {
        return userService.save(user);
    }

    @PutMapping
    public boolean update(@RequestBody User user) {
        return userService.updateById(user);
    }

    @DeleteMapping("/{id}")
    public boolean delete(@PathVariable Long id) {
        return userService.removeById(id);
    }
}
```

> 🎉 **一个完整的 CRUD 接口就这么 30 行代码搞定。**

## 十五、学习资源

- 官方文档（中文，超详细）：https://baomidou.com/
- GitHub：https://github.com/baomidou/mybatis-plus
- 国内项目里 95% 的 MP 用法都在官方文档里能找到

---

📝 *持续更新中……*
