# MyBatis-Plus 条件构造器(Wrapper)完全指南

## 一、继承体系概览

```
Wrapper(抽象类,顶层)
 └─ AbstractWrapper(封装所有通用条件方法:eq/ne/like/gt...)
     ├─ QueryWrapper              ── 查询,额外有 select()
     ├─ UpdateWrapper             ── 更新,额外有 set()
     ├─ AbstractLambdaWrapper
     │   ├─ LambdaQueryWrapper    ── QueryWrapper 的 Lambda 版
     │   └─ LambdaUpdateWrapper   ── UpdateWrapper 的 Lambda 版
     └─ ...

链式构造器(继承自 AbstractChainWrapper,内部组合上面的 Wrapper):
  QueryChainWrapper / LambdaQueryChainWrapper
  UpdateChainWrapper / LambdaUpdateChainWrapper
```

核心要点:**绝大部分条件方法(eq、like、gt 等)都定义在 `AbstractWrapper` 上,所以四种构造器都能用。** 区别只在于:Query 系有 `select()`,Update 系有 `set()`,Lambda 系用方法引用代替字符串字段名。

---

## 二、AbstractWrapper 的全部条件方法

以下方法所有构造器通用。下面用 `QueryWrapper<User>` 举例,Lambda 版把 `"字段名"` 换成 `User::getXxx` 即可。

### 1. 比较运算

| 方法 | 含义 | 示例 | 生成 SQL |
|------|------|------|----------|
| `eq` | 等于 = | `eq("name", "张三")` | `name = '张三'` |
| `ne` | 不等于 <> | `ne("age", 18)` | `age <> 18` |
| `gt` | 大于 > | `gt("age", 18)` | `age > 18` |
| `ge` | 大于等于 >= | `ge("age", 18)` | `age >= 18` |
| `lt` | 小于 < | `lt("age", 60)` | `age < 60` |
| `le` | 小于等于 <= | `le("age", 60)` | `age <= 60` |

```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("name", "张三")
       .ge("age", 18)
       .lt("age", 60);
// WHERE name = '张三' AND age >= 18 AND age < 60
```

### 2. 范围:between / in

| 方法 | 含义 | 示例 |
|------|------|------|
| `between` | BETWEEN a AND b | `between("age", 18, 30)` |
| `notBetween` | NOT BETWEEN | `notBetween("age", 18, 30)` |
| `in` | IN (...) | `in("id", 1, 2, 3)` 或 `in("id", idList)` |
| `notIn` | NOT IN (...) | `notIn("id", idList)` |
| `inSql` | IN (子查询) | `inSql("id", "select id from t where ...")` |
| `notInSql` | NOT IN (子查询) | `notInSql("id", "select ...")` |

```java
wrapper.between("age", 18, 30)
       .in("dept_id", Arrays.asList(1, 2, 3))
       .inSql("id", "select user_id from t_order where amount > 100");
// WHERE age BETWEEN 18 AND 30
//   AND dept_id IN (1,2,3)
//   AND id IN (select user_id from t_order where amount > 100)
```

### 3. 模糊匹配:like 系列

| 方法 | 含义 | 生成 |
|------|------|------|
| `like` | 两侧模糊 | `LIKE '%值%'` |
| `notLike` | 非模糊 | `NOT LIKE '%值%'` |
| `likeLeft` | 左模糊 | `LIKE '%值'` |
| `likeRight` | 右模糊(可走索引) | `LIKE '值%'` |

```java
wrapper.like("name", "张")        // name LIKE '%张%'
       .likeRight("phone", "138"); // phone LIKE '138%'
```

### 4. 空值判断

| 方法 | 生成 |
|------|------|
| `isNull("email")` | `email IS NULL` |
| `isNotNull("email")` | `email IS NOT NULL` |

### 5. 分组、排序、having

```java
wrapper.groupBy("dept_id")                  // GROUP BY dept_id
       .having("count(id) > {0}", 5)        // HAVING count(id) > 5
       .orderByDesc("create_time")          // ORDER BY create_time DESC
       .orderByAsc("age");                  // , age ASC

// orderBy 通用形式:第一个参数 isAsc 控制升降序
wrapper.orderBy(true, false, "create_time"); // ORDER BY create_time DESC
```

`orderByDesc`/`orderByAsc` 可一次传多个字段:`orderByDesc("a", "b")`。

### 6. 逻辑拼接:and / or / 嵌套

默认多个条件之间是 `AND` 连接。要用 `OR`,在两个条件中间插入 `.or()`:

```java
// name = '张三' OR age = 20
wrapper.eq("name", "张三").or().eq("age", 20);
```

**嵌套**(给一段条件加括号),and/or 接收 lambda:

```java
// status = 1 AND (name = '张三' OR name = '李四')
wrapper.eq("status", 1)
       .and(w -> w.eq("name", "张三").or().eq("name", "李四"));
// 等价于 .or(w -> ...) 产生 OR (...)
```

`nested` 也能产生括号,但不带前置 and/or 连接词:

```java
wrapper.nested(w -> w.eq("a", 1).or().eq("b", 2)).eq("c", 3);
// (a = 1 OR b = 2) AND c = 3
```

### 7. 其它高级方法

| 方法 | 用途 | 示例 |
|------|------|------|
| `apply` | 拼接自定义 SQL 片段(支持占位符防注入) | `apply("date_format(create_time,'%Y-%m-%d') = {0}", "2024-01-01")` |
| `last` | 在 SQL 最末尾拼接(慎用,无法防注入) | `last("limit 1")` |
| `exists` | EXISTS 子查询 | `exists("select 1 from t_order o where o.uid = id")` |
| `notExists` | NOT EXISTS | `notExists("select 1 from ...")` |
| `func` | 在链式中按条件分支调用 | 见下 |

```java
// apply 一定要用 {0} 占位符而不是字符串拼接,否则有 SQL 注入风险
wrapper.apply("date_format(create_time,'%Y-%m-%d') = {0}", "2024-06-01");

// func:根据条件决定走哪个分支,保持链式不断
wrapper.func(w -> {
    if (someFlag) {
        w.eq("status", 1);
    } else {
        w.ne("status", 1);
    }
});
```

---

## 三、condition 参数:动态条件(非常重要)

几乎所有条件方法都有一个**重载版本,第一个参数是 `boolean condition`**。为 `true` 时该条件才生效,为 `false` 时直接跳过。这是替代大量 `if` 判断的利器。

```java
String name = request.getName();   // 可能为空
Integer age = request.getAge();    // 可能为 null

// 传统写法要写一堆 if
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.like(StringUtils.isNotBlank(name), "name", name)
       .eq(age != null, "age", age)
       .ge(age != null, "create_time", someTime);
// name 为空时不拼 like,age 为 null 时不拼 eq —— 一行搞定
```

这是实际项目中处理「多条件可选查询」的标准做法。

---

## 四、QueryWrapper —— 查询专用

继承 AbstractWrapper,额外提供 `select()` 指定查询列。

```java
QueryWrapper<User> wrapper = new QueryWrapper<>();

// 只查指定字段
wrapper.select("id", "name", "age")
       .eq("status", 1);
// SELECT id, name, age FROM user WHERE status = 1

// 排除某些字段(谓词写法):查所有列但排除 password、salt
QueryWrapper<User> w2 = new QueryWrapper<>();
w2.select(User.class, info ->
        !info.getColumn().equals("password")
     && !info.getColumn().equals("salt"));
```

配合 mapper 使用:

```java
List<User> list = userMapper.selectList(wrapper);
User one = userMapper.selectOne(wrapper);
Long count = userMapper.selectCount(wrapper);
```

---

## 五、UpdateWrapper —— 更新专用

额外提供 `set()` 和 `setSql()`,可以在**不构造实体对象**的情况下更新字段。

```java
UpdateWrapper<User> wrapper = new UpdateWrapper<>();
wrapper.set("name", "新名字")
       .set("age", 30)
       .set(someFlag, "email", "x@y.com")  // 也支持 condition
       .setSql("version = version + 1")      // 原生 SQL 片段
       .eq("id", 1);                         // WHERE 条件
// UPDATE user SET name='新名字', age=30, version=version+1 WHERE id=1

userMapper.update(null, wrapper);  // 第一个参数实体传 null,因为字段都在 set 里了
```

对比:如果用实体对象更新,则是 `userMapper.update(userEntity, wrapper)`,此时 wrapper 只负责 where 条件,set 的内容来自实体非空字段。

---

## 六、LambdaQueryWrapper —— 推荐首选

用方法引用代替字符串字段名,**编译期校验、重构友好、杜绝列名拼写错误**。

```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.select(User::getId, User::getName)
       .eq(User::getStatus, 1)
       .like(User::getName, "张")
       .ge(User::getAge, 18)
       .orderByDesc(User::getCreateTime);

List<User> list = userMapper.selectList(wrapper);
```

获取方式有三种,效果相同:

```java
// 1. 直接 new
new LambdaQueryWrapper<User>()

// 2. Wrappers 工具类(推荐)
Wrappers.<User>lambdaQuery()

// 3. 从 QueryWrapper 转
new QueryWrapper<User>().lambda()
```

---

## 七、LambdaUpdateWrapper

UpdateWrapper 的 Lambda 版:

```java
LambdaUpdateWrapper<User> wrapper = Wrappers.<User>lambdaUpdate()
        .set(User::getName, "新名字")
        .set(User::getAge, 30)
        .eq(User::getId, 1);

userMapper.update(null, wrapper);
```

---

## 八、链式构造器(Chain)—— 条件 + 执行一气呵成

链式构造器把「构造条件」和「执行方法」连起来,末尾直接 `.list()` / `.one()` / `.update()` 等。需要 `Service` 层支持(继承 `IService`/`ServiceImpl`)。

```java
// 通过 ChainWrappers 或 Service 的链式方法获取

// 查询:LambdaQueryChainWrapper
List<User> list = userService.lambdaQuery()
        .eq(User::getStatus, 1)
        .like(User::getName, "张")
        .ge(User::getAge, 18)
        .list();          // .list() / .one() / .page(page) / .count()

// 更新:LambdaUpdateChainWrapper
boolean ok = userService.lambdaUpdate()
        .set(User::getStatus, 0)
        .eq(User::getId, 1)
        .update();        // .update() / .remove()

// 字符串版
userService.query().eq("status", 1).list();
userService.update().set("status", 0).eq("id", 1).update();
```

链式构造器收尾方法一览:

- 查询:`.list()`、`.one()`、`.oneOpt()`、`.page(IPage)`、`.count()`、`.exists()`
- 更新/删除:`.update()`、`.remove()`

---

## 九、Wrappers 工具类(快捷创建)

`com.baomidou.mybatisplus.core.toolkit.Wrappers` 提供静态工厂方法,比 new 更简洁:

```java
Wrappers.<User>query()         // QueryWrapper<User>
Wrappers.<User>update()        // UpdateWrapper<User>
Wrappers.<User>lambdaQuery()   // LambdaQueryWrapper<User>
Wrappers.<User>lambdaUpdate()  // LambdaUpdateWrapper<User>
Wrappers.emptyWrapper()        // 空条件(查全表时占位)
```

---

## 十、完整实战示例

一个典型的「分页 + 多条件可选查询」:

```java
public IPage<User> queryUsers(UserQueryDTO dto) {
    LambdaQueryWrapper<User> wrapper = Wrappers.<User>lambdaQuery()
        // 仅当参数非空时拼接条件
        .like(StringUtils.isNotBlank(dto.getName()),
              User::getName, dto.getName())
        .eq(dto.getStatus() != null,
              User::getStatus, dto.getStatus())
        .between(dto.getStartTime() != null && dto.getEndTime() != null,
              User::getCreateTime, dto.getStartTime(), dto.getEndTime())
        .in(CollectionUtils.isNotEmpty(dto.getDeptIds()),
              User::getDeptId, dto.getDeptIds())
        // 嵌套 OR
        .and(dto.getKeyword() != null, w -> w
              .like(User::getName, dto.getKeyword())
              .or()
              .like(User::getEmail, dto.getKeyword()))
        .orderByDesc(User::getCreateTime);

    Page<User> page = new Page<>(dto.getPageNum(), dto.getPageSize());
    return userMapper.selectPage(page, wrapper);
}
```

---

## 十一、注意事项与最佳实践

1. **优先用 Lambda 版**(`LambdaQueryWrapper` / `LambdaUpdateWrapper`):类型安全,字段改名时编译器会报错,避免字符串硬编码列名。

2. **善用 `condition` 参数**代替 `if`:`eq(参数 != null, 字段, 参数)`,让动态查询代码扁平清晰。

3. **`apply` 用占位符,不要字符串拼接**:`apply("col = {0}", value)` 能防 SQL 注入;`last()` 无法防注入,只在确实安全(如固定的 `limit 1`)时使用。

4. **`UpdateWrapper.update` 实体传 null**:字段值放在 `set()` 里时,`userMapper.update(null, wrapper)` 实体参数传 null。

5. **`select` 排除字段**用谓词写法:适合「查所有列但排除大字段 / 敏感字段」的场景。

6. **链式构造器需要 Service 支持**:`lambdaQuery()`/`lambdaUpdate()` 来自 `IService`,Mapper 层用普通 Wrapper + `selectList` 等方法。

7. **OR 的作用域**:`.or()` 只影响它前后两个条件;要把多个条件括起来再 OR,用 `and(w -> ...)` / `or(w -> ...)` 嵌套。

---

## 速查表

| 需求 | 用什么 |
|------|--------|
| 普通查询 | `LambdaQueryWrapper` + `selectList/selectOne/selectPage` |
| 指定查询列 | `QueryWrapper.select(...)` |
| 不带实体更新字段 | `LambdaUpdateWrapper.set(...)` + `update(null, wrapper)` |
| 条件 + 直接执行 | `service.lambdaQuery()...list()` |
| 快捷创建 | `Wrappers.lambdaQuery()` 等 |
| 动态可选条件 | 各方法的 `condition` 重载 |
| 加括号 / 嵌套 OR | `and(w -> ...)` / `or(w -> ...)` / `nested(...)` |
| 自定义 SQL 片段 | `apply("col = {0}", v)` |
