
# JDBC 与数据库操作

**JDBC（Java Database Connectivity）** 是 Java 操作数据库的标准 API。虽然实际开发中我们用 MyBatis、JPA 等更高层的框架，但**它们的底层都是 JDBC**，所以必须掌握。

## 一、JDBC 核心概念

```
Java 程序
    ↓
JDBC API（java.sql 包）
    ↓
JDBC 驱动（数据库厂商提供，如 mysql-connector）
    ↓
数据库（MySQL / PostgreSQL / Oracle ...）
```

**JDBC 的好处：** Java 代码不用关心是哪个数据库，换数据库只需要换驱动。

## 二、JDBC 的六大核心对象

| 对象                 | 作用                  |
| ------------------ | ------------------- |
| `DriverManager`    | 注册驱动、获取数据库连接        |
| `Connection`       | 代表一次数据库连接           |
| `Statement`        | 执行 SQL（容易 SQL 注入，少用）|
| `PreparedStatement`| 预编译 SQL（**推荐**）     |
| `ResultSet`        | 查询结果集               |
| `CallableStatement`| 调用存储过程              |

## 三、JDBC 操作六步走

```
① 加载驱动 → ② 获取连接 → ③ 创建 Statement
   → ④ 执行 SQL → ⑤ 处理结果 → ⑥ 关闭资源
```

## 四、准备工作

### 1. 引入驱动

Maven 项目在 `pom.xml` 里加：

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
</dependency>
```

> ⚠️ MySQL 8.x 用 `mysql-connector-j`，5.x 用 `mysql-connector-java`。

### 2. 准备数据库

```sql
CREATE DATABASE testdb;
USE testdb;

CREATE TABLE user (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    age INT,
    email VARCHAR(100)
);
```

## 五、第一个 JDBC 程序：查询

```java
import java.sql.*;

public class JdbcDemo {
    public static void main(String[] args) {
        // ① 数据库连接信息
        String url = "jdbc:mysql://localhost:3306/testdb?serverTimezone=UTC&useSSL=false";
        String user = "root";
        String password = "123456";

        // ② 用 try-with-resources 自动关闭资源
        try (Connection conn = DriverManager.getConnection(url, user, password);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT * FROM user")) {

            // ③ 遍历结果集
            while (rs.next()) {
                int id = rs.getInt("id");
                String name = rs.getString("name");
                int age = rs.getInt("age");
                System.out.println(id + " | " + name + " | " + age);
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

> 💡 JDBC 4.0 之后**不需要**再手动 `Class.forName("com.mysql.cj.jdbc.Driver")`，驱动会自动注册。

## 六、PreparedStatement（强烈推荐！）

`Statement` 拼接 SQL 字符串容易导致 **SQL 注入**，绝大多数情况都应该用 `PreparedStatement`。

### SQL 注入的危险

```java
// ❌ 危险写法
String name = "' OR '1'='1";   // 来自用户输入
String sql = "SELECT * FROM user WHERE name = '" + name + "'";
// 拼出来变成：SELECT * FROM user WHERE name = '' OR '1'='1'
// → 返回所有用户！
```

### PreparedStatement 的安全写法

```java
String sql = "SELECT * FROM user WHERE name = ? AND age > ?";

try (Connection conn = DriverManager.getConnection(url, user, password);
     PreparedStatement ps = conn.prepareStatement(sql)) {

    ps.setString(1, "小林");   // 第 1 个 ?
    ps.setInt(2, 18);          // 第 2 个 ?

    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            System.out.println(rs.getString("name"));
        }
    }
} catch (SQLException e) {
    e.printStackTrace();
}
```

> ⚠️ **注意：参数索引从 1 开始，不是 0！**

### PreparedStatement 的优势

✅ 防 SQL 注入
✅ 可复用（编译一次，多次执行）
✅ 性能更好

## 七、增删改：executeUpdate

`executeUpdate()` 用于 INSERT / UPDATE / DELETE，返回**受影响的行数**。

```java
String sql = "INSERT INTO user (name, age, email) VALUES (?, ?, ?)";

try (Connection conn = DriverManager.getConnection(url, user, password);
     PreparedStatement ps = conn.prepareStatement(sql)) {

    ps.setString(1, "小林");
    ps.setInt(2, 18);
    ps.setString(3, "xiaolin@example.com");

    int rows = ps.executeUpdate();
    System.out.println("插入了 " + rows + " 行");
}
```

### 获取自增主键

```java
String sql = "INSERT INTO user (name, age) VALUES (?, ?)";

try (PreparedStatement ps = conn.prepareStatement(sql,
        Statement.RETURN_GENERATED_KEYS)) {   // 关键：声明要返回自增主键

    ps.setString(1, "小林");
    ps.setInt(2, 18);
    ps.executeUpdate();

    try (ResultSet keys = ps.getGeneratedKeys()) {
        if (keys.next()) {
            long id = keys.getLong(1);
            System.out.println("新插入的 ID = " + id);
        }
    }
}
```

## 八、ResultSet 的常用方法

```java
ResultSet rs = ps.executeQuery();

while (rs.next()) {                        // 移动到下一行
    // 按列名取值（推荐）
    int id = rs.getInt("id");
    String name = rs.getString("name");

    // 按列索引取值（从 1 开始）
    int age = rs.getInt(3);

    // 处理 null
    String email = rs.getString("email");
    if (rs.wasNull()) {
        email = "无";
    }
}
```

### 常用 getXxx 方法

| 方法               | 用途       |
| ---------------- | -------- |
| `getInt`         | int      |
| `getLong`        | long     |
| `getString`      | String   |
| `getDouble`      | double   |
| `getBoolean`     | boolean  |
| `getDate`        | java.sql.Date |
| `getTimestamp`   | java.sql.Timestamp |
| `getObject`      | 任意类型     |

## 九、事务（Transaction）

**事务保证一组 SQL 要么全部成功，要么全部回滚。** 经典例子：转账。

```java
Connection conn = null;
try {
    conn = DriverManager.getConnection(url, user, password);
    conn.setAutoCommit(false);   // ① 关闭自动提交，开启手动事务

    PreparedStatement ps1 = conn.prepareStatement(
        "UPDATE account SET balance = balance - ? WHERE id = ?");
    ps1.setDouble(1, 100);
    ps1.setLong(2, 1);            // 张三扣 100
    ps1.executeUpdate();

    // 模拟出错
    if (Math.random() < 0.5) throw new RuntimeException("转账失败！");

    PreparedStatement ps2 = conn.prepareStatement(
        "UPDATE account SET balance = balance + ? WHERE id = ?");
    ps2.setDouble(1, 100);
    ps2.setLong(2, 2);            // 李四加 100
    ps2.executeUpdate();

    conn.commit();   // ② 提交事务
    System.out.println("转账成功");

} catch (Exception e) {
    if (conn != null) {
        try {
            conn.rollback();   // ③ 出错则回滚
            System.out.println("已回滚");
        } catch (SQLException ex) {
            ex.printStackTrace();
        }
    }
} finally {
    if (conn != null) {
        try { conn.close(); } catch (SQLException e) {}
    }
}
```

### 事务的 ACID 特性

| 特性           | 说明                |
| ------------ | ----------------- |
| **A** 原子性    | 要么全做，要么全不做        |
| **C** 一致性    | 转账前后总金额不变         |
| **I** 隔离性    | 多个事务互不干扰          |
| **D** 持久性    | 提交后数据永久保存         |

### 事务隔离级别

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

| 级别                | 脏读  | 不可重复读 | 幻读  |
| ----------------- | --- | ----- | --- |
| READ_UNCOMMITTED  | ✅   | ✅     | ✅   |
| READ_COMMITTED    | ❌   | ✅     | ✅   |
| REPEATABLE_READ   | ❌   | ❌     | ✅   |
| SERIALIZABLE      | ❌   | ❌     | ❌   |

> 💡 MySQL 默认是 `REPEATABLE_READ`，PostgreSQL/Oracle 默认是 `READ_COMMITTED`。

## 十、批量操作（性能优化）

一次插 1000 条数据，挨个 execute 慢得要命，用批量：

```java
String sql = "INSERT INTO user (name, age) VALUES (?, ?)";

try (Connection conn = DriverManager.getConnection(url, user, password);
     PreparedStatement ps = conn.prepareStatement(sql)) {

    conn.setAutoCommit(false);

    for (int i = 0; i < 1000; i++) {
        ps.setString(1, "用户" + i);
        ps.setInt(2, 18 + i % 50);
        ps.addBatch();              // 加入批次

        if (i % 100 == 0) {
            ps.executeBatch();       // 每 100 条执行一次
            ps.clearBatch();
        }
    }
    ps.executeBatch();               // 执行剩余的
    conn.commit();
}
```

> 💡 **配合连接 URL 加 `rewriteBatchedStatements=true`**，MySQL 会把批量 INSERT 合并成一条，性能再提升 10 倍。

## 十一、连接池（实战必用）

每次新建连接代价很大（TCP 握手、认证），实际项目都用**连接池**复用连接。

### 主流连接池

| 连接池        | 特点              |
| ---------- | --------------- |
| **HikariCP** | 性能最快，Spring Boot 默认 ⭐ |
| Druid      | 阿里出品，监控功能强       |
| DBCP       | 老牌，性能一般         |
| C3P0       | 老牌，性能一般         |

### HikariCP 使用示例

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

```java
import com.zaxxer.hikari.*;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/testdb");
config.setUsername("root");
config.setPassword("123456");
config.setMaximumPoolSize(10);

HikariDataSource ds = new HikariDataSource(config);

// 用法和原生 JDBC 几乎一样
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT * FROM user");
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
        System.out.println(rs.getString("name"));
    }
}
```

> 💡 `conn.close()` 在连接池中**不是真的关闭**，而是**归还连接**到池子里。

## 十二、封装一个简单的 DAO

实战中我们会把 JDBC 操作封装成 **DAO（Data Access Object）**：

```java
public class UserDao {
    private DataSource dataSource;

    public UserDao(DataSource ds) { this.dataSource = ds; }

    // 查询
    public User findById(long id) throws SQLException {
        String sql = "SELECT * FROM user WHERE id = ?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setLong(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    User u = new User();
                    u.setId(rs.getLong("id"));
                    u.setName(rs.getString("name"));
                    u.setAge(rs.getInt("age"));
                    return u;
                }
                return null;
            }
        }
    }

    // 插入
    public int insert(User u) throws SQLException {
        String sql = "INSERT INTO user (name, age) VALUES (?, ?)";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, u.getName());
            ps.setInt(2, u.getAge());
            return ps.executeUpdate();
        }
    }
}
```

## 十三、JDBC 之上的进化

实际开发中很少直接写 JDBC，因为代码太啰嗦，常用更高层的框架：

| 框架                | 特点                      |
| ----------------- | ----------------------- |
| **MyBatis**       | SQL 映射框架，灵活，国内最流行 ⭐    |
| **JPA / Hibernate** | ORM 框架，对象-表自动映射         |
| **Spring JDBC**   | Spring 提供，简化 JDBC，比原生省事 |
| **Spring Data JPA** | JPA 的 Spring 封装，"零 SQL" |

### 同样查询的代码量对比

```java
// 原生 JDBC：~20 行
PreparedStatement ps = conn.prepareStatement("SELECT * FROM user WHERE id = ?");
ps.setLong(1, id);
ResultSet rs = ps.executeQuery();
// ... 手动映射

// MyBatis：1 行（XML/注解里写 SQL）
User u = userMapper.findById(id);

// Spring Data JPA：连 SQL 都不用写
User u = userRepository.findById(id);
```

> 💡 但你**必须先懂 JDBC**，因为这些框架底层都是它，出问题才能排查。

## 十四、最佳实践

✅ **该做的：**
- **永远用 `PreparedStatement`**，不用 `Statement`
- **永远用 try-with-resources** 自动关闭资源
- 实际项目用**连接池**（HikariCP）
- 显式管理**事务**，别依赖默认的自动提交
- 数据库连接信息放配置文件，不要硬编码

❌ **不该做的：**
- 不要拼接 SQL 字符串（SQL 注入）
- 不要忘记关闭连接（即使是连接池也要 close 归还）
- 不要在循环里 `executeUpdate`，用批量操作
- 大查询不要 `SELECT *`，按需取列

---

📝 *持续更新中……*
