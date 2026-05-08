# Maven 构建工具

**Maven** 是 Java 生态最主流的项目管理和构建工具。它解决了三大问题：**依赖管理、项目构建、项目结构标准化**。掌握 Maven 是 Java 工程师的必备技能。

## 一、为什么需要 Maven？

### 没有 Maven 的痛苦

想用一个第三方库（比如 MySQL 驱动），你要：
1. 去官网下载 jar 包
2. 放到项目的 lib 文件夹
3. 在 IDE 里手动引入
4. 这个 jar 又依赖别的 jar？继续手动下载……
5. 多个项目都用同一个 jar？每个项目都拷一份
6. 想升级版本？所有项目都要手动换

### 有 Maven 之后

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
</dependency>
```

写完这几行，Maven **自动**：
- 下载这个 jar 包到本地
- 下载它依赖的所有 jar（**传递依赖**）
- 多个项目共享同一份缓存
- 升级版本只改一个数字

## 二、安装 Maven

### Windows / Mac / Linux

1. 下载：https://maven.apache.org/download.cgi
2. 解压到任意目录
3. 配置环境变量 `MAVEN_HOME` 和 `PATH`
4. 验证：

```bash
mvn -v

# 输出类似：
# Apache Maven 3.9.6
# Java version: 17.0.9
```

> 💡 IDE（IntelliJ IDEA、Eclipse）**自带 Maven**，新手不装也能用，但学习推荐自己装一份。

## 三、Maven 项目标准结构

Maven 强制约定的目录结构（**约定优于配置**）：

```
my-project/
├── pom.xml                        ← 项目核心配置文件
├── src/
│   ├── main/
│   │   ├── java/                  ← 主代码
│   │   │   └── com/example/App.java
│   │   └── resources/             ← 配置文件、静态资源
│   │       └── application.yml
│   └── test/
│       ├── java/                  ← 测试代码
│       │   └── com/example/AppTest.java
│       └── resources/             ← 测试资源
└── target/                        ← 构建产物（自动生成）
    ├── classes/                   ← 编译后的 class
    └── my-project-1.0.jar         ← 打好的 jar 包
```

> 💡 **不要随便改这个结构**，Maven 默认就按这个找文件。

## 四、pom.xml 核心元素

`pom.xml`（**P**roject **O**bject **M**odel）是 Maven 项目的"身份证"。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <!-- ① 项目坐标（GAV）—— 唯一标识 -->
    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>           <!-- jar / war / pom -->

    <name>My Project</name>
    <description>我的第一个 Maven 项目</description>

    <!-- ② 全局属性 -->
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <!-- ③ 依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <!-- ④ 构建配置 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

### GAV 坐标 —— 找包的"门牌号"

| 字段             | 含义                  | 示例                        |
| -------------- | ------------------- | ------------------------- |
| **groupId**    | 组织/团队/公司域名（倒写）      | `org.springframework.boot` |
| **artifactId** | 项目/模块名              | `spring-boot-starter-web`  |
| **version**    | 版本号                 | `3.2.5`                    |

**任何 jar 包都靠这三个唯一确定。** 想找包？去 **[Maven 中央仓库](https://mvnrepository.com)** 搜，复制坐标就行。

## 五、依赖管理

### 添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.3.0</version>
    </dependency>
</dependencies>
```

保存 `pom.xml`，Maven 自动下载。

### 依赖范围（scope）

| scope          | 编译 | 测试 | 运行 | 打包 | 说明                 |
| -------------- | -- | -- | -- | -- | ------------------ |
| **compile**（默认）| ✅  | ✅  | ✅  | ✅  | 全程可用               |
| **test**       | ❌  | ✅  | ❌  | ❌  | 仅测试用（如 JUnit）      |
| **provided**   | ✅  | ✅  | ❌  | ❌  | 容器/JDK 已提供（如 servlet-api）|
| **runtime**    | ❌  | ✅  | ✅  | ✅  | 运行时才用（如 JDBC 驱动）   |
| **system**     | -  | -  | -  | -  | 本地路径（不推荐）          |

```xml
<!-- 测试库 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>

<!-- 容器提供，不打包进 war -->
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.0.0</version>
    <scope>provided</scope>
</dependency>
```

### 传递依赖

A 依赖 B，B 依赖 C → **你只要写 A，B 和 C 自动来。**

```
你 → spring-boot-starter-web
            ↓ 自动带来
        spring-web、spring-mvc、jackson、tomcat-embed...
```

### 排除依赖（exclusions）

如果传递依赖里有你**不想要的包**，可以排除：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.2.5</version>
    <exclusions>
        <!-- 不要默认的 logback，我想用 log4j2 -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 依赖冲突 & 版本仲裁

A 依赖 `xxx:1.0`，B 依赖 `xxx:2.0`，怎么办？

Maven 的规则：
1. **路径最短优先**（依赖层级越浅的优先）
2. **声明顺序优先**（路径相同时，pom 里先声明的赢）

查看冲突：
```bash
mvn dependency:tree
```

输出类似：
```
[INFO] com.example:my-project:jar:1.0
[INFO] +- org.springframework:spring-core:jar:6.1.5:compile
[INFO] |  \- org.springframework:spring-jcl:jar:6.1.5:compile
[INFO] +- com.mysql:mysql-connector-j:jar:8.3.0:runtime
```

### 强制锁定版本（dependencyManagement）

在父 pom 里**只声明版本**，子模块用时**不用写版本号**：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.3.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <!-- 不写 version，自动用上面声明的 -->
    </dependency>
</dependencies>
```

> 💡 这就是为什么 Spring Boot 项目里很多依赖**都不写版本号**——版本由 `spring-boot-starter-parent` 统一管。

## 六、Maven 生命周期

Maven 把构建过程标准化为**三套生命周期**：

| 生命周期    | 作用     |
| ------- | ------ |
| clean   | 清理构建产物 |
| **default** | 编译、测试、打包、部署（**最常用**） |
| site    | 生成项目站点文档 |

### default 生命周期的核心阶段

```
validate → compile → test → package → verify → install → deploy
```

| 阶段          | 作用                   |
| ----------- | -------------------- |
| **compile** | 编译主代码                |
| **test**    | 运行单元测试               |
| **package** | 打包成 jar/war          |
| **install** | 安装到**本地仓库**（其他本地项目可用） |
| **deploy**  | 部署到**远程仓库**（团队共享）    |

> ⚠️ **重要规则：** 执行某个阶段时，**之前的阶段会自动执行**。比如 `mvn package` 会先 compile + test 再打包。

## 七、常用 Maven 命令

```bash
# 清理 target 目录
mvn clean

# 编译
mvn compile

# 运行测试
mvn test

# 打包（会先 compile + test）
mvn package

# 安装到本地仓库（~/.m2/repository）
mvn install

# 跳过测试打包
mvn package -DskipTests

# 跳过测试且不编译测试代码
mvn package -Dmaven.test.skip=true

# 组合命令（最常用！）
mvn clean install      # 清理 + 完整构建 + 安装
mvn clean package      # 清理 + 打包

# 查看依赖树
mvn dependency:tree

# 查看 effective pom（合并所有继承后的最终配置）
mvn help:effective-pom

# 强制更新依赖
mvn clean install -U
```

## 八、Maven 仓库

```
本地仓库（~/.m2/repository）        ← 你电脑上的缓存
        ↓ 找不到时去
远程仓库（中央仓库 / 私服）
```

### 三种仓库

| 仓库    | 说明                           |
| ----- | ---------------------------- |
| 本地仓库  | 默认在用户目录下 `.m2/repository`    |
| 中央仓库  | Maven 官方提供，包最全               |
| 私服    | 公司搭建的内部仓库（Nexus、Artifactory）|

### 配置阿里云镜像（国内必备）

中央仓库在国外，国内访问慢。配置阿里云镜像加速：

打开（或创建）`~/.m2/settings.xml`：

```xml
<settings>
    <mirrors>
        <mirror>
            <id>aliyun</id>
            <name>阿里云公共仓库</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
    </mirrors>
</settings>
```

> 💡 配置后，国内下载速度提升 10 倍以上。

### 修改本地仓库位置

默认 `~/.m2/repository` 会越来越大，可以改到大盘：

```xml
<settings>
    <localRepository>D:/maven-repo</localRepository>
</settings>
```

## 九、多模块项目

实际项目通常拆成**多个模块**，比如：

```
my-shop/                    ← 父项目（packaging=pom）
├── pom.xml
├── shop-common/            ← 公共工具
├── shop-user/              ← 用户模块
├── shop-order/             ← 订单模块
└── shop-web/               ← Web 入口
```

### 父 pom

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-shop</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>     <!-- 父项目用 pom -->

    <modules>
        <module>shop-common</module>
        <module>shop-user</module>
        <module>shop-order</module>
        <module>shop-web</module>
    </modules>

    <!-- 统一管理依赖版本 -->
    <dependencyManagement>
        <dependencies>
            <!-- ... -->
        </dependencies>
    </dependencyManagement>
</project>
```

### 子模块 pom

```xml
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-shop</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>shop-user</artifactId>
    <!-- groupId 和 version 继承自父 pom，不用写 -->

    <dependencies>
        <!-- 引用同项目的另一个模块 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>shop-common</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
</project>
```

## 十、常用插件

Maven 的功能都靠**插件**实现。

### 编译插件

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <source>17</source>
        <target>17</target>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
```

### 打可执行 jar（包含所有依赖）

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.2</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.example.App</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

打包后可以直接：
```bash
java -jar my-project-1.0.jar
```

### Spring Boot 打包插件

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

## 十一、Profile —— 多环境配置

不同环境（开发/测试/生产）用不同配置：

```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <env>dev</env>
            <db.url>jdbc:mysql://localhost:3306/devdb</db.url>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>

    <profile>
        <id>prod</id>
        <properties>
            <env>prod</env>
            <db.url>jdbc:mysql://prod-server:3306/proddb</db.url>
        </properties>
    </profile>
</profiles>
```

激活某个 profile：
```bash
mvn package -Pprod
```

## 十二、Maven vs Gradle

| 对比项     | Maven         | Gradle           |
| ------- | ------------- | ---------------- |
| 配置文件    | XML（pom.xml） | DSL（Groovy/Kotlin）|
| 学习曲线    | 简单            | 略陡               |
| 灵活性     | 标准、规范         | 灵活、可定制           |
| 性能      | 一般            | 快（增量构建）          |
| Java 生态 | 主流            | Android 主流，后端兴起 |

> 💡 **Java 后端项目目前 Maven 仍是主流**，Android 和部分大型项目用 Gradle。先学 Maven 完全够用。

## 十三、常见问题

### 1. 依赖下载失败 / 慢

```bash
# 用阿里云镜像（前面讲过）
# 或强制重新下载
mvn clean install -U
```

### 2. 找不到主类 / 打的 jar 跑不起来

普通 `mvn package` 出来的 jar 不包含依赖，要用 **shade** 或 **spring-boot-maven-plugin**。

### 3. 编译版本错误

确保设置了 Java 版本：
```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
</properties>
```

### 4. 中文乱码

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>
```

### 5. IDE 同步问题

IntelliJ 右键 pom.xml → Maven → Reimport，几乎能解决 90% 的玄学问题。

## 十四、最佳实践

✅ **该做的：**
- 配置**阿里云镜像**加速
- 用 `dependencyManagement` 统一版本
- 多模块项目用**父子结构**
- 公司级代码用**私服（Nexus）**
- 依赖冲突时用 `mvn dependency:tree` 排查

❌ **不该做的：**
- 不要在代码里硬编码 jar 路径
- 不要把 jar 包放在项目里（让 Maven 管）
- 不要随便改 `<version>`，多人协作会冲突
- 不要把 `target/` 提交到 Git（要写进 `.gitignore`）

## 十五、典型 .gitignore 配置

Java + Maven 项目的标配 `.gitignore`：

```gitignore
# Maven
target/

# IDE
.idea/
*.iml
.vscode/
.settings/
.project
.classpath

# 日志
logs/
*.log

# 系统文件
.DS_Store
Thumbs.db
```

---

📝 *持续更新中……*
