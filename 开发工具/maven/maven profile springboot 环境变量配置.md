


# Maven Profile 详解

Maven Profile 是 Maven 提供的一种**环境隔离和构建配置定制**机制，允许你为不同的环境（如开发、测试、生产）、不同的操作系统或不同的构建需求定义不同的配置集，在构建时通过指定 Profile 来激活对应的配置，从而实现**一次定义、多环境复用**。

## 一、Profile 的核心作用

- 隔离不同环境的配置：如数据库连接、服务器地址、端口等。
- 定制构建行为：如不同环境的依赖、插件配置、打包方式、资源过滤等。
- 适配不同的运行环境：如 Windows/Linux 系统、JDK 版本差异等。

## 二、Profile 的定义位置

Profile 可以定义在**三个不同的位置**，作用范围不同：

### 1. 项目内的 `pom.xml`

仅对当前项目有效，是最常用的方式。

xml

```xml
<project>
  ...
  <profiles>
    <!-- 开发环境 Profile -->
    <profile>
      <id>dev</id> <!-- Profile 唯一标识，必须指定 -->
      <properties>
        <!-- 定义环境变量 -->
        <env>dev</env>
        <db.url>jdbc:mysql://localhost:3306/dev_db</db.url>
        <db.username>dev_user</db.username>
      </properties>
    </profile>

    <!-- 生产环境 Profile -->
    <profile>
      <id>prod</id>
      <properties>
        <env>prod</env>
        <db.url>jdbc:mysql://192.168.1.100:3306/prod_db</db.url>
        <db.username>prod_user</db.username>
      </properties>
    </profile>
  </profiles>
  ...
</project>
```

### 2. 用户级配置：`~/.m2/settings.xml`

对当前用户的所有 Maven 项目有效，适合定义个人常用的 Profile（如私服镜像、代理配置）。

xml

```xml
<settings>
  ...
  <profiles>
    <profile>
      <id>my-private-repo</id>
      <repositories>
        <repository>
          <id>private-repo</id>
          <url>https://maven.example.com/repo</url>
        </repository>
      </repositories>
    </profile>
  </profiles>
  ...
</settings>
```

### 3. 全局配置：`{MAVEN_HOME}/conf/settings.xml`

对所有使用该 Maven 安装的用户和项目有效，一般由运维人员配置（如公司全局私服）。

## 三、Profile 的激活方式

激活 Profile 是使用它的核心，主要有**手动激活**和**自动激活**两种方式：

### 1. 手动激活（最常用）

#### （1）命令行参数激活

通过 `-P` 参数指定 Profile ID（多个 Profile 用逗号分隔）：

bash

运行

```bash
# 激活 dev 环境
mvn clean package -Pdev

# 同时激活 dev 和 test 环境
mvn clean package -Pdev,test
```

#### （2）在 `settings.xml` 中设置默认激活

xml

```xml
<settings>
  ...
  <activeProfiles>
    <!-- 默认激活 dev Profile -->
    <activeProfile>dev</activeProfile>
  </activeProfiles>
  ...
</settings>
```

### 2. 自动激活

Maven 支持根据**环境条件**自动激活 Profile，无需手动指定：

#### （1）根据 JDK 版本激活

xml

```xml
<profile>
  <id>jdk17</id>
  <activation>
    <jdk>17</jdk> <!-- 匹配 JDK 17，支持范围如 1.8+, [11,17) -->
  </activation>
</profile>
```

#### （2）根据操作系统激活

xml

```xml
<profile>
  <id>windows</id>
  <activation>
    <os>
      <name>Windows</name>
      <arch>amd64</arch>
    </os>
  </activation>
</profile>

<profile>
  <id>linux</id>
  <activation>
    <os>
      <name>Linux</name>
    </os>
  </activation>
</profile>
```

#### （3）根据系统属性激活

xml

```xml
<profile>
  <id>debug</id>
  <activation>
    <property>
      <name>debug</name> <!-- 当命令行传入 -Ddebug 时激活 -->
      <value>true</value> <!-- 可选，指定属性值 -->
    </property>
  </activation>
</profile>
```

#### （4）根据文件存在 / 缺失激活

xml

```xml
<profile>
  <id>local-dev</id>
  <activation>
    <file>
      <exists>src/main/resources/local.properties</exists> <!-- 文件存在时激活 -->
      <!-- <missing>src/main/resources/prod.properties</missing> 文件缺失时激活 -->
    </file>
  </activation>
</profile>
```

## 四、Profile 的常用配置项

Profile 内部可以覆盖或新增 Maven 构建的几乎所有配置，常见的有：

### 1. 自定义属性（`<properties>`）

如环境变量、数据库配置、服务地址等，可在 POM 或资源文件中通过 `${属性名}` 引用。

### 2. 依赖配置（`<dependencies>`/`<dependencyManagement>`）

为不同环境引入不同的依赖（如开发环境引入测试依赖，生产环境排除）：

xml

```xml
<profile>
  <id>test</id>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</profile>
```

### 3. 插件配置（`<build>/<plugins>`）

定制插件的行为（如打包时的资源过滤、编译参数）：

xml

```xml
<profile>
  <id>prod</id>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>17</source>
          <target>17</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</profile>
```

### 4. 资源过滤（`<build>/<resources>`）

实现不同环境的资源文件动态替换（如 `application-${env}.properties`）：

xml

```xml
<project>
  ...
  <build>
    <resources>
      <resource>
        <directory>src/main/resources</directory>
        <filtering>true</filtering> <!-- 开启资源过滤 -->
        <includes>
          <include>application*.properties</include>
        </includes>
      </resource>
    </resources>
  </build>

  <profiles>
    <profile>
      <id>dev</id>
      <properties>
        <env>dev</env>
      </properties>
    </profile>
    <profile>
      <id>prod</id>
      <properties>
        <env>prod</env>
      </properties>
    </profile>
  </profiles>
</project>
```

此时资源文件中可以写 `${db.url}`，构建时会替换为 Profile 中定义的值。

### 5. 仓库配置（`<repositories>`/`<pluginRepositories>`）

为不同环境配置不同的依赖仓库（如开发环境用私服，生产环境用中央仓库）。

## 五、查看已激活的 Profile

可以通过以下命令查看当前构建中激活的 Profile：

bash

运行

```bash
mvn help:active-profiles
```

也可以通过 `mvn help:all-profiles` 查看所有可用的 Profile。

## 六、最佳实践

1. **命名规范**：Profile ID 采用有意义的名称（如 `dev`/`test`/`prod`/`windows`/`linux`），避免无意义的数字或字母。
2. **最小化配置**：Profile 中只定义与环境相关的差异配置，公共配置放在 POM 主配置中，减少冗余。
3. **资源分离**：将不同环境的配置文件放在不同目录（如 `src/main/resources/dev`、`src/main/resources/prod`），通过 Profile 控制资源加载。
4. **避免硬编码**：尽量使用属性引用，而非硬编码配置值，提高可维护性。
5. **默认 Profile**：在 `settings.xml` 中设置默认激活的开发环境 Profile，减少日常构建的命令行参数。

## 七、示例：完整的多环境 Profile 配置

xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>my-project</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <!-- 公共属性 -->
  <properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <!-- 资源过滤配置 -->
  <build>
    <resources>
      <resource>
        <directory>src/main/resources</directory>
        <filtering>true</filtering>
        <includes>
          <include>**/*.properties</include>
          <include>**/*.yml</include>
        </includes>
      </resource>
    </resources>
  </build>

  <!-- 多环境 Profile -->
  <profiles>
    <!-- 开发环境 -->
    <profile>
      <id>dev</id>
      <properties>
        <env>dev</env>
        <db.url>jdbc:mysql://localhost:3306/dev_db?useSSL=false</db.url>
        <db.username>root</db.username>
        <db.password>123456</db.password>
        <server.port>8080</server.port>
      </properties>
    </profile>

    <!-- 测试环境 -->
    <profile>
      <id>test</id>
      <properties>
        <env>test</env>
        <db.url>jdbc:mysql://192.168.1.50:3306/test_db?useSSL=false</db.url>
        <db.username>test_user</db.username>
        <db.password>test123</db.password>
        <server.port>8081</server.port>
      </properties>
    </profile>

    <!-- 生产环境 -->
    <profile>
      <id>prod</id>
      <properties>
        <env>prod</env>
        <db.url>jdbc:mysql://10.0.0.10:3306/prod_db?useSSL=true</db.url>
        <db.username>prod_user</db.username>
        <db.password>${prod.db.password}</db.password> <!-- 建议通过 settings.xml 的 server 配置或环境变量传递 -->
        <server.port>80</server.port>
      </properties>
      <build>
        <plugins>
          <!-- 生产环境打包时跳过测试 -->
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
              <skipTests>true</skipTests>
            </configuration>
          </plugin>
        </plugins>
      </build>
    </profile>
  </profiles>
</project>
```

通过以上配置，你可以轻松实现：

- 开发环境：`mvn clean package -Pdev`
- 测试环境：`mvn clean package -Ptest`
- 生产环境：`mvn clean package -Pprod`

Maven Profile 是实现多环境构建的核心工具，合理使用可以极大提升项目的可维护性和构建效率。

编辑分享

springboot 配置文件的环境变量 和 maven profile 中的环境变量的区别

# SpringBoot 配置文件环境变量与 Maven Profile 环境变量的区别

SpringBoot 配置文件中的环境变量（如 `application-dev.properties` 中的配置）和 Maven Profile 中的环境变量，是**两个不同阶段、不同作用域**的配置机制，核心区别体现在**执行时机、作用范围、加载方式、使用场景**等方面。下面从多个维度详细对比，并说明如何协同使用。

## 一、核心定位与执行时机

这是两者最根本的区别，决定了它们的作用阶段。

|特性|Maven Profile 环境变量|SpringBoot 配置文件环境变量|
|---|---|---|
|**执行阶段**|**构建时（Compile/Package 阶段）**<br><br>Maven 作为构建工具，在编译、打包项目时解析和使用这些变量。|**运行时（Runtime 阶段）**<br><br>SpringBoot 应用启动时，加载配置文件并解析其中的变量，供应用运行时使用。|
|**本质**|Maven 构建过程的**构建参数**，用于定制构建行为。|SpringBoot 应用的**运行参数**，用于配置应用的运行时行为。|
|**生命周期**|随 Maven 构建过程结束而失效（打包后仅保留最终替换结果）。|随 SpringBoot 应用的启动而加载，应用运行期间一直有效。|

### 举例理解

- 用 Maven Profile 定义 `${jdk.version=17}`：作用是让 Maven 在**编译时**使用 JDK 17 编译代码，构建完成后这个变量就没用了。
- 用 SpringBoot 配置定义 `server.port=8080`：作用是让 SpringBoot 应用**启动时**监听 8080 端口，运行期间一直生效。

## 二、作用范围与生效范围

两者的作用域完全不同，一个针对**构建过程**，一个针对**应用运行**。

### 1. Maven Profile 环境变量的作用范围

- **构建层面**：影响 Maven 构建的所有环节，包括：
    - 依赖的引入 / 排除（如开发环境引入 `spring-boot-starter-test`，生产环境排除）；
    - 插件的配置（如编译插件的 JDK 版本、打包插件的资源过滤）；
    - 资源文件的**静态替换**（如打包时将 `${db.url}` 替换为具体值）；
    - 仓库的选择（如开发环境用私服，生产环境用中央仓库）。
- **作用域**：仅对当前 Maven 项目的构建过程有效，无法直接被 SpringBoot 应用运行时感知（除非通过资源过滤将其写入配置文件）。

### 2. SpringBoot 配置文件环境变量的作用范围

- **运行层面**：影响 SpringBoot 应用的运行时行为，包括：
    - 数据库连接、Redis 地址、服务端口等运行时配置；
    - 日志级别、缓存配置、第三方服务地址等；
    - Spring 容器的 Bean 初始化、自动配置等。
- **作用域**：仅对 SpringBoot 应用的运行时有效，与构建过程无关（即使构建时没有 Maven，直接运行 JAR 包也能加载）。

## 三、定义与引用方式

两者的定义格式、引用语法和加载优先级都不同。

### 1. 定义方式

|类型|定义位置与格式|
|---|---|
|Maven Profile 变量|在 `pom.xml` 的 `<profile>` 中通过 `<properties>` 定义：<br><br>`xml<br><profile><br> <id>dev</id><br> <properties><br> <env.dev.db.url>jdbc:mysql://localhost:3306/dev_db</env.dev.db.url><br> </properties><br></profile><br>`|
|SpringBoot 配置变量|在 `src/main/resources` 下的配置文件中定义，支持多种格式：<br><br>**properties 格式**：`application-dev.properties`<br><br>`spring.datasource.url=jdbc:mysql://localhost:3306/dev_db`<br><br>**yml 格式**：`application-dev.yml`<br><br>`yaml<br>spring:<br> datasource:<br> url: jdbc:mysql://localhost:3306/dev_db<br>`|

### 2. 引用方式

|类型|引用语法与场景|
|---|---|
|Maven Profile 变量|- 在 `pom.xml` 中通过 `${变量名}` 引用（如插件配置、依赖版本）；<br><br>- 在资源文件中通过 `${变量名}` 引用（需开启 Maven 资源过滤）；<br><br>- 命令行通过 `-D变量名=值` 覆盖。|
|SpringBoot 配置变量|- 在代码中通过 `@Value("${变量名}")` 或 `@ConfigurationProperties` 注入；<br><br>- 在配置文件中通过 `${变量名}` 跨配置引用；<br><br>- 命令行通过 `--变量名=值`（或 `-D变量名=值`）覆盖；<br><br>- 支持通过环境变量、系统属性、配置中心等多种方式加载。|

### 3. 加载优先级

- **Maven Profile 变量**：优先级为「命令行 `-D` 参数」>「用户 settings.xml 中的 Profile」>「项目 pom.xml 中的 Profile」>「全局 settings.xml 中的 Profile」。
- **SpringBoot 配置变量**：遵循 SpringBoot 固有的配置加载优先级（从高到低）：命令行参数 > 系统属性 > 环境变量 > 应用配置文件（`application-{profile}.yml`）> 默认配置文件（`application.yml`）> 内置默认值。

## 四、动态性与可修改性

两者的灵活度差异较大，核心在于**构建时**和**运行时**的区别。

|特性|Maven Profile 环境变量|SpringBoot 配置文件环境变量|
|---|---|---|
|**动态修改**|构建后即固化（如打包成 JAR 后，变量替换结果已写入文件），无法在运行时修改。|运行时可灵活修改：<br><br>- 通过命令行参数覆盖；<br><br>- 通过配置中心（Nacos/Apollo）动态刷新；<br><br>- 更换激活的 Profile 即可加载不同配置。|
|**多环境切换**|切换环境需要**重新构建**项目（如 `mvn package -Pprod` 重新打包）。|切换环境无需重新构建，仅需在启动时指定 Profile（如 `java -jar app.jar --spring.profiles.active=prod`）。|

## 五、使用场景的核心差异

根据两者的特性，适用场景有明确的划分：

### 1. Maven Profile 环境变量的适用场景

**所有需要在构建阶段决定的行为**，典型场景：

- 不同环境的**资源过滤与静态替换**（如打包时将配置文件中的占位符替换为具体值）；
- 不同环境的**依赖管理**（如开发环境引入测试依赖，生产环境排除）；
- 构建参数的定制（如编译的 JDK 版本、打包的文件名、是否跳过测试）；
- 不同环境的**插件配置**（如生产环境打包时跳过单元测试，开发环境执行测试）。

### 2. SpringBoot 配置文件环境变量的适用场景

**所有需要在运行时决定的应用配置**，典型场景：

- 应用的运行时参数（端口、上下文路径、日志级别）；
- 第三方服务的连接配置（数据库、Redis、MQ、微服务注册中心地址）；
- 业务相关的配置（如接口超时时间、限流阈值）；
- 需要动态调整的配置（如通过配置中心刷新的参数）。

## 六、如何协同使用（最佳实践）

实际项目中，两者并非互斥，而是**互补使用**，核心是**通过 Maven Profile 控制 SpringBoot 配置文件的激活和静态替换**，具体步骤：

### 步骤 1：在 Maven Profile 中定义 SpringBoot 的激活环境

在 `pom.xml` 中定义不同环境的 Profile，并通过属性指定 SpringBoot 要激活的 Profile：

xml

```xml
<profiles>
  <!-- 开发环境 -->
  <profile>
    <id>dev</id>
    <properties>
      <!-- 定义 SpringBoot 要激活的 Profile 名称 -->
      <spring.profiles.active>dev</spring.profiles.active>
      <!-- 可选：定义需要静态替换的变量 -->
      <db.url>jdbc:mysql://localhost:3306/dev_db</db.url>
    </properties>
    <!-- 设为默认激活 -->
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
  </profile>

  <!-- 生产环境 -->
  <profile>
    <id>prod</id>
    <properties>
      <spring.profiles.active>prod</spring.profiles.active>
      <db.url>jdbc:mysql://10.0.0.10:3306/prod_db</db.url>
    </properties>
  </profile>
</profiles>
```

### 步骤 2：开启 Maven 资源过滤，将变量写入 SpringBoot 配置文件

在 `pom.xml` 的构建配置中开启资源过滤，让 Maven 打包时替换 SpringBoot 配置文件中的占位符：

xml

```xml
<build>
  <resources>
    <resource>
      <directory>src/main/resources</directory>
      <!-- 开启资源过滤，允许替换配置文件中的 ${变量名} -->
      <filtering>true</filtering>
      <includes>
        <include>application*.properties</include>
        <include>application*.yml</include>
      </includes>
    </resource>
  </resources>
</build>
```

### 步骤 3：在 SpringBoot 配置文件中引用 Maven 变量

在 `application.properties` 中通过 `${变量名}` 引用 Maven Profile 中定义的变量，同时指定 SpringBoot 的激活 Profile：

properties

```properties
# 引用 Maven Profile 中的 spring.profiles.active，激活对应环境的配置
spring.profiles.active=${spring.profiles.active}

# 引用 Maven Profile 中的数据库地址（静态替换，打包后固化）
spring.datasource.url=${db.url}
spring.datasource.username=${db.username}
```

### 步骤 4：运行时切换 SpringBoot 环境（无需重新构建）

如果需要在运行时切换环境，可直接通过命令行参数覆盖，无需重新打包：

bash

运行

```bash
# 构建时用 dev Profile 打包，运行时指定 prod 环境
java -jar app.jar --spring.profiles.active=prod
```

## 七、总结表格

|对比维度|Maven Profile 环境变量|SpringBoot 配置文件环境变量|
|---|---|---|
|**执行阶段**|构建时（Compile/Package）|运行时（Runtime）|
|**作用**|定制构建行为，如编译、打包、依赖管理|配置应用运行时行为，如端口、数据库连接|
|**作用域**|Maven 构建过程|SpringBoot 应用运行期|
|**动态性**|构建后固化，无法运行时修改|运行时可灵活修改、动态刷新|
|**环境切换**|需重新构建项目|无需重新构建，启动时指定 Profile 即可|
|**引用方式**|`${变量名}`（POM / 资源文件），需开启 Maven 过滤|`@Value`/`@ConfigurationProperties`（代码），`${变量名}`（配置文件）|
|**核心场景**|构建参数、资源静态替换、依赖管理|运行时配置、动态参数、第三方服务连接|

## 核心结论

- **Maven Profile 管 “构建”**：决定项目怎么打包、编译，是**构建期的静态配置**；
- **SpringBoot 配置管 “运行”**：决定应用怎么运行，是**运行期的动态配置**；
- 实际项目中，应让 Maven Profile 负责**构建阶段的环境隔离**，SpringBoot 配置负责**运行阶段的环境配置**，通过资源过滤实现两者的协同，既保证构建的灵活性，又兼顾运行的动态性。

如何在 Spring Boot 中使用 Maven Profile 环境变量？

# 在 Spring Boot 中使用 Maven Profile 环境变量的完整指南

在 Spring Boot 中使用 Maven Profile 环境变量，核心是**利用 Maven 构建阶段的资源过滤和属性传递**，将 Maven Profile 中定义的变量传递到 Spring Boot 运行时的配置中。主要有两种核心方式：**静态替换（构建时固化）** 和**动态关联（运行时匹配）**，下面结合具体示例详细讲解实现步骤、最佳实践和常见问题。

## 一、核心原理

Maven Profile 是**构建时**的配置机制，Spring Boot 是**运行时**的应用框架，两者的桥接依赖：

1. **Maven 资源过滤**：开启后，Maven 在打包时会将资源文件（如 `application.yml`）中的 `${变量名}` 替换为 Maven Profile 中定义的实际值。
2. **Profile 激活关联**：通过 Maven Profile 定义 `spring.profiles.active` 变量，让 Spring Boot 启动时自动加载对应环境的配置文件（如 `application-dev.yml`）。

## 二、具体实现步骤

### 步骤 1：在 `pom.xml` 中定义 Maven Profile

首先在项目的 `pom.xml` 中定义不同环境的 Maven Profile，包含自定义变量（如数据库配置、Spring Boot 激活的 Profile 名称）。

xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>springboot-maven-profile</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <!-- 1. 定义 Maven Profile -->
    <profiles>
        <!-- 开发环境 Profile -->
        <profile>
            <id>dev</id>
            <!-- 设置为默认激活，日常开发无需手动指定 -->
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <!-- 关键：指定 Spring Boot 要激活的 Profile 名称 -->
                <spring.profiles.active>dev</spring.profiles.active>
                <!-- 自定义环境变量：数据库配置 -->
                <app.db.url>jdbc:mysql://localhost:3306/dev_db?useSSL=false&amp;serverTimezone=UTC</app.db.url>
                <app.db.username>root</app.db.username>
                <app.db.password>123456</app.db.username>
                <!-- 自定义环境变量：服务端口 -->
                <app.server.port>8080</app.server.port>
            </properties>
        </profile>

        <!-- 测试环境 Profile -->
        <profile>
            <id>test</id>
            <properties>
                <spring.profiles.active>test</spring.profiles.active>
                <app.db.url>jdbc:mysql://192.168.1.50:3306/test_db?useSSL=false&amp;serverTimezone=UTC</app.db.url>
                <app.db.username>test_user</app.db.username>
                <app.db.password>test123</app.db.password>
                <app.server.port>8081</app.server.port>
            </properties>
        </profile>

        <!-- 生产环境 Profile -->
        <profile>
            <id>prod</id>
            <properties>
                <spring.profiles.active>prod</spring.profiles.active>
                <app.db.url>jdbc:mysql://10.0.0.10:3306/prod_db?useSSL=true&amp;serverTimezone=UTC</app.db.url>
                <app.db.username>prod_user</app.db.username>
                <app.db.password>${prod.db.password}</app.db.password> <!-- 建议通过环境变量/settings.xml 传递 -->
                <app.server.port>80</app.server.port>
            </properties>
        </profile>
    </profiles>

    <!-- 2. 开启 Maven 资源过滤（关键：让 Maven 替换配置文件中的变量） -->
    <build>
        <resources>
            <resource>
                <!-- 指定资源目录 -->
                <directory>src/main/resources</directory>
                <!-- 开启资源过滤：允许替换 ${变量名} 占位符 -->
                <filtering>true</filtering>
                <!-- 只对配置文件进行过滤，提高构建效率 -->
                <includes>
                    <include>application*.yml</include>
                    <include>application*.properties</include>
                    <include>**/*.xml</include>
                </includes>
            </resource>
            <!-- 排除无需过滤的资源（如静态文件），可选 -->
            <resource>
                <directory>src/main/resources</directory>
                <filtering>false</filtering>
                <excludes>
                    <exclude>application*.yml</exclude>
                    <exclude>application*.properties</exclude>
                </excludes>
            </resource>
        </resources>

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 步骤 2：创建 Spring Boot 多环境配置文件

在 `src/main/resources` 目录下创建对应环境的配置文件，Maven Profile 会通过 `spring.profiles.active` 变量关联这些文件。

#### 1. 主配置文件：`application.yml`

主配置文件中引用 Maven Profile 中的变量，作为全局配置入口：

yaml

```yaml
# 引用 Maven Profile 中的 spring.profiles.active，激活对应环境的配置
spring:
  profiles:
    active: ${spring.profiles.active}

# 引用 Maven Profile 中的自定义变量（构建时静态替换）
server:
  port: ${app.server.port}

spring:
  datasource:
    url: ${app.db.url}
    username: ${app.db.username}
    password: ${app.db.password}
    driver-class-name: com.mysql.cj.jdbc.Driver
```

#### 2. 环境专属配置文件（可选）

如果需要更细粒度的环境配置，可以创建环境专属文件，例如：

- `application-dev.yml`（开发环境）：
    
    yaml
    
    ```yaml
    # 开发环境专属配置
    logging:
      level:
        com.example: DEBUG
    spring:
      devtools:
        restart:
          enabled: true
    ```
    
- `application-prod.yml`（生产环境）：
    
    yaml
    
    ```yaml
    # 生产环境专属配置
    logging:
      level:
        com.example: INFO
      file:
        name: /var/log/app.log
    spring:
      devtools:
        restart:
          enabled: false
    ```
    

### 步骤 3：在 Spring Boot 代码中使用变量

通过 Spring Boot 的注解将配置变量注入到代码中，使用方式与普通配置一致。

java

运行

```java
package com.example.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class EnvController {

    // 注入 Maven Profile 传递的数据库 URL 变量
    @Value("${spring.datasource.url}")
    private String dbUrl;

    // 注入激活的环境名称
    @Value("${spring.profiles.active}")
    private String activeEnv;

    @GetMapping("/env")
    public String getEnv() {
        return "当前环境：" + activeEnv + "<br>数据库地址：" + dbUrl;
    }
}
```

### 步骤 4：构建和运行项目

通过 Maven 命令激活不同的 Profile，构建并运行项目。

#### 1. 构建项目（激活指定 Maven Profile）

bash

运行

```bash
# 激活开发环境（默认，可省略 -Pdev）
mvn clean package -Pdev

# 激活测试环境
mvn clean package -Ptest

# 激活生产环境（可通过 -D 传递敏感参数）
mvn clean package -Pprod -Dprod.db.password=Prod@123456
```

#### 2. 运行项目

bash

运行

```bash
# 运行打包后的 JAR 包
java -jar target/springboot-maven-profile-1.0.0.jar
```

#### 3. 运行时覆盖配置（无需重新构建）

如果需要在运行时修改配置，可通过命令行参数覆盖，优先级高于 Maven 构建时的变量：

bash

运行

```bash
# 构建时用 dev Profile，运行时指定 prod 环境和端口
java -jar target/springboot-maven-profile-1.0.0.jar --spring.profiles.active=prod --server.port=8088
```

## 三、两种核心使用方式的对比

|方式|实现方式|特点|适用场景|
|---|---|---|---|
|**静态替换**|Maven 打包时直接替换配置文件中的 `${变量名}`|构建后变量值固化到 JAR 包中，运行时无法修改|固定的环境配置（如数据库地址、服务端口）|
|**动态关联**|Maven Profile 仅指定 `spring.profiles.active`，具体配置在 Spring Boot 环境文件中|构建后可通过命令行 / 配置中心动态切换环境|需要灵活切换的配置（如日志级别、功能开关）|

**最佳实践**：结合两种方式，**静态替换核心固定配置**（如数据库地址），**动态关联环境专属配置**（如日志级别），兼顾构建效率和运行灵活性。

## 四、常见问题与解决方案

### 1. 变量替换不生效

- **原因**：未开启 Maven 资源过滤，或资源目录配置错误。
- **解决方案**：确保 `pom.xml` 中 `<resource>` 节点开启 `<filtering>true</filtering>`，并指定正确的 `directory`。

### 2. 特殊字符（如 &、空格）被转义

- **原因**：Maven 资源过滤时会对 XML/Properties 中的特殊字符进行转义。
- **解决方案**：在 Maven 变量中使用 XML 实体（如 `&amp;` 代替 `&`），或在 YAML 中用引号包裹值。

### 3. 生产环境敏感信息（如密码）泄露

- **原因**：将敏感信息直接写在 `pom.xml` 中，存在安全风险。
- **解决方案**：
    1. 通过命令行 `-D` 参数传递：`mvn package -Pprod -Dprod.db.password=xxx`；
    2. 在 `~/.m2/settings.xml` 中配置服务器凭证，通过 `${settings.server.password}` 引用；
    3. 使用 Spring Cloud Config/Nacos 等配置中心管理敏感配置。

### 4. 多 Profile 同时激活

- **需求**：同时激活多个 Maven Profile（如 `dev` + `debug`）。
- **解决方案**：命令行用逗号分隔 Profile ID：`mvn package -Pdev,debug`。

## 五、最佳实践总结

1. **命名规范**：Maven Profile ID 与 Spring Boot Profile 名称保持一致（如 `dev`/`test`/`prod`），避免混淆。
2. **最小化静态替换**：仅对核心固定配置（如数据库地址）进行静态替换，动态配置通过 Spring Boot 环境文件管理。
3. **敏感信息隔离**：不将密码、密钥等敏感信息写在 `pom.xml` 中，通过命令行、配置中心或环境变量传递。
4. **默认激活开发环境**：将 `dev` Profile 设置为默认激活，减少日常开发的命令行参数。
5. **资源过滤优化**：只对配置文件开启过滤，排除静态资源（如 JS/CSS/ 图片），提高构建效率。

通过以上方式，即可完美实现 Maven Profile 环境变量在 Spring Boot 中的使用，既利用了 Maven 构建时的环境隔离能力，又保留了 Spring Boot 运行时的配置灵活性。

maven delimiters 配合

# Maven Delimiters 与 Spring Boot 结合使用的完整指南

Maven 的**delimiters（分隔符）** 是控制资源过滤时**占位符解析规则**的核心配置，默认情况下 Maven 使用 `${变量名}` 作为占位符，但通过自定义 delimiters，你可以修改占位符的格式（如改为 `@变量名@`），避免与 Spring Boot 配置文件的占位符语法冲突，同时更灵活地控制资源过滤的行为。本文将详细讲解 Maven delimiters 的作用、配置方式，以及如何与 Spring Boot 协同使用。

## 一、Maven Delimiters 的核心作用

在 Maven 资源过滤（`filtering=true`）过程中，delimiters 用于**定义占位符的起始和结束标记**，Maven 会根据这些标记识别资源文件中的变量，并替换为 Maven Profile（或 POM）中定义的实际值。

### 1. 默认分隔符

Maven 资源过滤的默认分隔符是：

- **起始分隔符**：`${`
- **结束分隔符**：`}`

这意味着默认情况下，资源文件中的 `${变量名}` 会被 Maven 解析并替换。

### 2. 为什么需要自定义 Delimiters？

在与 Spring Boot 结合时，默认分隔符可能会引发**冲突**：

- Spring Boot 配置文件（如 `application.yml`）本身也支持 `${变量名}` 作为占位符（用于运行时解析）；
- 如果同时开启 Maven 资源过滤，Maven 会在构建时提前替换这些占位符，导致 Spring Boot 运行时无法动态解析自己的占位符。

例如：

yaml

```yaml
# 期望 Spring Boot 运行时解析 ${JAVA_HOME}，但 Maven 构建时会尝试替换（若 POM 中无此变量，会替换为空）
app.java.home: ${JAVA_HOME}
```

此时，自定义 Maven 的 delimiters（如改为 `@变量名@`），可以让 Maven 只解析特定格式的占位符，避免与 Spring Boot 的占位符冲突。

## 二、Maven Delimiters 的配置方式

Maven 支持两种方式配置 delimiters：**全局配置**（针对所有资源）和**局部配置**（针对单个资源目录），核心是通过 `<build>/<filters>` 或 `<resource>/<filtering>` 配合 `<delimiters>` 节点实现。

### 1. 基础配置：修改资源过滤的分隔符

在 `pom.xml` 的 `<resources>` 节点中，为指定的资源目录配置自定义 delimiters：

xml

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <!-- 开启资源过滤 -->
            <filtering>true</filtering>
            <includes>
                <include>application*.yml</include>
                <include>application*.properties</include>
            </includes>
            <!-- 配置自定义分隔符（关键） -->
            <delimiters>
                <!-- 优先级：先解析 @变量名@，再解析默认的 ${变量名}（可选） -->
                <delimiter>@</delimiter>
                <!-- 保留默认分隔符（可选，若需要同时支持两种格式） -->
                <!-- <delimiter>${*}</delimiter> -->
            </delimiters>
        </resource>
        <!-- 静态资源不开启过滤，提高构建效率 -->
        <resource>
            <directory>src/main/resources</directory>
            <filtering>false</filtering>
            <excludes>
                <exclude>application*.yml</exclude>
                <exclude>application*.properties</exclude>
            </excludes>
        </resource>
    </resources>
</build>
```

**配置说明**：

- `<delimiter>@</delimiter>`：表示使用**单字符分隔符**，占位符格式为 `@变量名@`（Maven 会自动将单个字符作为起始和结束标记）。
- 若需要自定义**多字符分隔符**（如 `{%` 和 `%}`），需使用 `${*}` 格式：
    
    xml
    
    ```xml
    <delimiters>
        <delimiter>${*}</delimiter> <!-- 默认分隔符 ${变量名} -->
        <delimiter>{%*%}</delimiter> <!-- 自定义分隔符 {%变量名%} -->
    </delimiters>
    ```
    
    其中 `*` 是变量名的占位符，`{%*%}` 表示起始为 `{%`，结束为 `%}`。

### 2. 结合 Profile 配置分隔符（多环境适配）

可以在 Maven Profile 中为不同环境配置不同的 delimiters（虽然很少用，但可实现灵活控制）：

xml

```xml
<profiles>
    <profile>
        <id>dev</id>
        <build>
            <resources>
                <resource>
                    <directory>src/main/resources</directory>
                    <filtering>true</filtering>
                    <delimiters>
                        <delimiter>@</delimiter>
                    </delimiters>
                </resource>
            </resources>
        </build>
    </profile>
</profiles>
```

### 3. 全局过滤配置（通过 filters 文件）

除了直接在 POM 中定义变量，还可以通过**过滤文件（.properties）** 定义变量，并配合 delimiters 解析：

#### 步骤 1：创建过滤文件

在 `src/main/filters` 目录下创建环境专属的过滤文件：

- `dev.properties`：
    
    properties
    
    ```properties
    app.db.url=jdbc:mysql://localhost:3306/dev_db
    app.server.port=8080
    ```
    
- `prod.properties`：
    
    properties
    
    ```properties
    app.db.url=jdbc:mysql://10.0.0.10:3306/prod_db
    app.server.port=80
    ```
    

#### 步骤 2：在 POM 中配置 filters 和 delimiters

xml

```xml
<build>
    <!-- 配置过滤文件的位置 -->
    <filters>
        <filter>src/main/filters/@env@.properties</filter> <!-- 使用 @env@ 作为占位符 -->
    </filters>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <delimiters>
                <delimiter>@</delimiter>
            </delimiters>
        </resource>
    </resources>
</build>

<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <env>dev</env> <!-- 替换过滤文件中的 @env@ -->
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <env>prod</env>
        </properties>
    </profile>
</profiles>
```

## 三、与 Spring Boot 结合的实战示例

下面通过一个完整的示例，展示如何使用自定义 delimiters 实现 Maven Profile 变量与 Spring Boot 配置的协同，同时避免占位符冲突。

### 步骤 1：配置 POM.xml（自定义 Delimiters + Profile）

xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>springboot-maven-delimiters</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <!-- 1. 定义 Maven Profile -->
    <profiles>
        <profile>
            <id>dev</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <spring.profiles.active>dev</spring.profiles.active>
                <app.db.url>jdbc:mysql://localhost:3306/dev_db?useSSL=false</app.db.url>
                <app.server.port>8080</app.server.port>
            </properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <spring.profiles.active>prod</spring.profiles.active>
                <app.db.url>jdbc:mysql://10.0.0.10:3306/prod_db?useSSL=true</app.db.url>
                <app.server.port>80</app.server.port>
            </properties>
        </profile>
    </profiles>

    <!-- 2. 配置资源过滤和自定义 Delimiters -->
    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>application*.yml</include>
                    <include>application*.properties</include>
                </includes>
                <!-- 自定义分隔符：使用 @变量名@ 格式 -->
                <delimiters>
                    <delimiter>@</delimiter>
                </delimiters>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>false</filtering>
                <excludes>
                    <exclude>application*.yml</exclude>
                    <exclude>application*.properties</exclude>
                </excludes>
            </resource>
        </resources>

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 步骤 2：编写 Spring Boot 配置文件（使用 @变量名 @）

在 `application.yml` 中使用 Maven 自定义的 `@变量名@` 占位符，同时保留 Spring Boot 自身的 `${变量名}` 占位符：

yaml

```yaml
# Maven 解析 @spring.profiles.active@，Spring Boot 运行时无需解析
spring:
  profiles:
    active: @spring.profiles.active@

# Maven 解析 @app.server.port@，构建时替换为具体值
server:
  port: @app.server.port@

# Maven 解析 @app.db.url@，构建时替换
spring:
  datasource:
    url: @app.db.url@
    username: @app.db.username@
    password: @app.db.password@
    driver-class-name: com.mysql.cj.jdbc.Driver

# Spring Boot 运行时解析 ${JAVA_HOME}（Maven 不会处理，因为分隔符不同）
app:
  java:
    home: ${JAVA_HOME}
```

### 步骤 3：构建并运行项目

bash

运行

```bash
# 激活 dev Profile 构建
mvn clean package -Pdev

# 运行项目
java -jar target/springboot-maven-delimiters-1.0.0.jar
```

**效果**：

- Maven 会将 `@spring.profiles.active@` 替换为 `dev`，`@app.server.port@` 替换为 `8080`；
- Spring Boot 运行时会解析 `${JAVA_HOME}` 为系统环境变量的值，不会被 Maven 干扰。

## 四、常见 Delimiters 配置场景

### 场景 1：仅使用自定义分隔符（@变量名 @）

xml

```xml
<delimiters>
    <delimiter>@</delimiter>
</delimiters>
```

**占位符格式**：`@变量名@`，适用于避免与 Spring Boot 占位符冲突。

### 场景 2：同时支持默认分隔符和自定义分隔符

xml

```xml
<delimiters>
    <delimiter>@</delimiter>
    <delimiter>${*}</delimiter>
</delimiters>
```

**占位符格式**：既支持 `@变量名@`，也支持 `${变量名}`，适用于需要兼容旧配置的场景。

### 场景 3：使用多字符分隔符（{% 变量名 %}）

xml

```xml
<delimiters>
    <delimiter>{%*%}</delimiter>
</delimiters>
```

**占位符格式**：`{%变量名%}`，适用于需要更独特的占位符格式，避免与其他工具冲突。

## 五、最佳实践与注意事项

### 1. 最佳实践

- **隔离分隔符**：将 Maven 的占位符格式改为 `@变量名@`，Spring Boot 保留 `${变量名}`，避免冲突。
- **按需过滤**：仅对配置文件开启资源过滤，静态资源（JS/CSS/ 图片）关闭过滤，提高构建效率。
- **敏感信息处理**：生产环境的密码等敏感信息，通过命令行 `-D` 参数或 `settings.xml` 传递，不写在 POM 或过滤文件中。
- **统一命名规范**：Maven Profile ID 与 Spring Boot Profile 名称保持一致（如 `dev`/`prod`），便于维护。

### 2. 注意事项

- **分隔符优先级**：Maven 会按照 `<delimiters>` 中的顺序解析占位符，先匹配的先替换。
- **特殊字符处理**：若占位符中包含特殊字符（如空格、&），需在 POM 中使用 XML 实体（如 `&amp;` 代替 `&`）。
- **Spring Boot 配置优先级**：运行时通过命令行参数（如 `--server.port=8088`）覆盖的配置，优先级高于 Maven 构建时的替换值。
- **多模块项目**：在父 POM 中配置 delimiters，子模块可继承该配置，保持统一。

## 六、总结

Maven delimiters 是控制资源过滤占位符的关键配置，通过自定义分隔符，可有效解决与 Spring Boot 配置文件的占位符冲突问题。核心要点：

1. **配置方式**：在 `<resources>` 节点中通过 `<delimiters>` 定义自定义分隔符；
2. **格式选择**：推荐使用 `@变量名@` 作为 Maven 占位符，避免与 Spring Boot 的 `${变量名}` 冲突；
3. **协同使用**：Maven 负责构建时的静态替换，Spring Boot 负责运行时的动态解析，各司其职。

通过合理配置 delimiters，可让 Maven 和 Spring Boot 的环境变量机制完美协同，既保证构建的灵活性，又保留运行时的动态性。

如何在多模块 Maven 项目中统一管理和使用 delimiters？

# 多模块 Maven 项目中统一管理 Delimiters 的完整指南

在多模块 Maven 项目中，**统一管理 delimiters（分隔符）** 的核心是利用 Maven 的**继承机制**（父 POM 定义全局配置，子模块继承）和**聚合机制**（父模块统一构建子模块），避免在每个子模块中重复配置 delimiters，同时保证所有模块的资源过滤规则一致。本文将详细讲解实现方案、实战示例、最佳实践和常见问题。

## 一、核心思路：父 POM 统一配置，子模块继承

Maven 的多模块项目中，**父模块（Parent Module）** 负责定义全局的构建配置（包括 delimiters、资源过滤、Profile 等），**子模块（Child Module）** 直接继承这些配置，无需重复编写。这种方式的优势：

1. **配置集中化**：所有模块的 delimiters 规则统一维护，修改时只需改父 POM。
2. **一致性**：避免不同子模块使用不同的分隔符格式，导致构建行为不一致。
3. **易维护**：减少重复代码，降低后期维护成本。

## 二、实现步骤：从父 POM 配置到子模块使用

### 步骤 1：搭建多模块项目结构

首先创建标准的 Maven 多模块项目结构，典型结构如下：

plaintext

```plaintext
springboot-multi-module/  # 父模块（聚合+父POM）
├── pom.xml               # 父POM：定义全局delimiters、Profile、资源过滤等
├── module-common/        # 子模块：通用工具类
│   └── pom.xml
├── module-web/           # 子模块：Web应用（Spring Boot）
│   └── pom.xml
└── module-service/       # 子模块：业务服务
    └── pom.xml
```

### 步骤 2：在父 POM 中统一配置 Delimiters

父 POM 作为**所有子模块的父工程**，需要在 `<build>` 节点中定义**全局的资源过滤和 delimiters 配置**，并通过 `<modules>` 聚合子模块。

#### 父 POM（`springboot-multi-module/pom.xml`）完整配置：

xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 1. 父POM基本信息（作为子模块的父工程） -->
    <groupId>com.example</groupId>
    <artifactId>springboot-multi-module</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging> <!-- 聚合工程必须为pom打包方式 -->

    <!-- 2. 聚合子模块 -->
    <modules>
        <module>module-common</module>
        <module>module-service</module>
        <module>module-web</module>
    </modules>

    <!-- 3. 定义全局属性（所有子模块可引用） -->
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring.boot.version>3.2.0</spring.boot.version>
    </properties>

    <!-- 4. 统一配置全局构建规则（包括delimiters） -->
    <build>
        <!-- 4.1 全局资源配置：所有子模块继承此配置 -->
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <!-- 开启资源过滤 -->
                <filtering>true</filtering>
                <!-- 只对配置文件进行过滤，提高效率 -->
                <includes>
                    <include>application*.yml</include>
                    <include>application*.properties</include>
                    <include>**/*.xml</include>
                </includes>
                <!-- 核心：统一配置delimiters（使用@变量名@格式） -->
                <delimiters>
                    <delimiter>@</delimiter> <!-- 自定义分隔符：@变量名@ -->
                    <!-- 可选：保留默认分隔符（如需兼容${变量名}） -->
                    <!-- <delimiter>${*}</delimiter> -->
                </delimiters>
            </resource>
            <!-- 排除无需过滤的静态资源 -->
            <resource>
                <directory>src/main/resources</directory>
                <filtering>false</filtering>
                <excludes>
                    <exclude>application*.yml</exclude>
                    <exclude>application*.properties</exclude>
                </excludes>
            </resource>
        </resources>

        <!-- 4.2 全局插件管理（统一插件版本，子模块可直接使用） -->
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring.boot.version}</version>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>3.3.1</version> <!-- 建议指定版本，避免依赖冲突 -->
                    <!-- 可选：通过插件配置delimiters（与上面的配置等效） -->
                    <configuration>
                        <delimiters>
                            <delimiter>@</delimiter>
                        </delimiters>
                    </configuration>
                </plugin>
            </plugins>
        </plugins>
    </build>

    <!-- 5. 统一定义Maven Profile（所有子模块共享） -->
    <profiles>
        <profile>
            <id>dev</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <spring.profiles.active>dev</spring.profiles.active>
                <app.db.url>jdbc:mysql://localhost:3306/dev_db</app.db.url>
                <app.server.port>8080</app.server.port>
            </properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <spring.profiles.active>prod</spring.profiles.active>
                <app.db.url>jdbc:mysql://10.0.0.10:3306/prod_db</app.db.url>
                <app.server.port>80</app.server.port>
            </properties>
        </profile>
    </profiles>
</project>
```

**关键配置说明**：

1. **`<packaging>pom</packaging>`**：父模块必须使用 `pom` 打包方式，作为聚合和父工程。
2. **`<resources>` 全局配置**：子模块会继承此配置，无需重复定义 delimiters。
3. **`<pluginManagement>`**：统一管理插件版本和配置，子模块可直接引用插件，无需指定版本。
4. **全局 Profile**：所有子模块共享相同的环境变量，保证多模块环境一致。

### 步骤 3：子模块继承父 POM 并按需调整

子模块的 POM 只需继承父 POM，即可复用所有全局配置（包括 delimiters、资源过滤、Profile 等），仅需根据模块特性做少量调整。

#### 示例 1：通用工具模块（`module-common/pom.xml`）

xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <!-- 继承父POM，自动获取delimiters、Profile等配置 -->
    <parent>
        <groupId>com.example</groupId>
        <artifactId>springboot-multi-module</artifactId>
        <version>1.0.0</version>
        <relativePath>../pom.xml</relativePath> <!-- 指向父POM的路径 -->
    </parent>

    <artifactId>module-common</artifactId>
    <packaging>jar</packaging>

    <!-- 子模块专属依赖（无需重复配置构建和delimiters） -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
    </dependencies>
</project>
```

#### 示例 2：Web 模块（`module-web/pom.xml`，Spring Boot 应用）

xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <groupId>com.example</groupId>
        <artifactId>springboot-multi-module</artifactId>
        <version>1.0.0</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>module-web</artifactId>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>module-service</artifactId>
            <version>1.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <!-- 仅需配置Spring Boot插件（版本由父POM的pluginManagement管理） -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

**核心优势**：子模块无需配置 delimiters 和资源过滤，直接继承父 POM 的全局配置，保证所有模块的规则一致。

### 步骤 4：子模块中使用统一的 Delimiters

所有子模块的资源文件（如 `application.yml`）都可以使用父 POM 定义的 `@变量名@` 占位符，Maven 会在构建时统一替换。

#### 示例：Web 模块的配置文件（`module-web/src/main/resources/application.yml`）

yaml

```yaml
# 使用父POM定义的delimiters：@spring.profiles.active@
spring:
  profiles:
    active: @spring.profiles.active@

# 引用父POM Profile中的变量
server:
  port: @app.server.port@

spring:
  datasource:
    url: @app.db.url@
    username: @app.db.username@
    password: @app.db.password@

# Spring Boot运行时解析自身的${变量名}（与Maven的delimiters不冲突）
app:
  java:
    home: ${JAVA_HOME}
```

### 步骤 5：统一构建多模块项目

在父模块根目录执行构建命令，Maven 会自动将全局配置（包括 delimiters）应用到所有子模块：

bash

运行

```bash
# 激活prod Profile，构建所有子模块
mvn clean package -Pprod

# 仅构建web模块（仍会继承父POM的配置）
mvn clean package -Pprod -pl module-web -am
```

- `-pl`：指定要构建的模块（`module-web`）。
- `-am`：自动构建依赖的模块（如 `module-service`、`module-common`）。

## 三、进阶配置：局部覆盖与特殊模块处理

在某些场景下，个别子模块可能需要**自定义 delimiters**（如特殊的资源过滤规则），此时可以在子模块中覆盖父 POM 的配置，实现**全局统一 + 局部定制**。

### 示例：子模块覆盖 Delimiters（`module-service/pom.xml`）

xml

```xml
<build>
    <!-- 覆盖父POM的资源配置，使用自定义delimiters -->
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <delimiters>
                <delimiter>{%*%}</delimiter> <!-- 自定义分隔符：{%变量名%} -->
            </delimiters>
        </resource>
    </resources>
</build>
```

**注意**：局部覆盖仅适用于特殊模块，尽量避免大量子模块自定义 delimiters，否则会破坏配置的统一性。

## 四、通过 Maven 插件统一配置 Delimiters

除了在 `<resources>` 中配置 delimiters，还可以通过 **`maven-resources-plugin`** 插件在父 POM 中统一配置，效果相同，且更便于集中管理。

### 父 POM 中的插件配置：

xml

```xml
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>3.3.1</version>
                <configuration>
                    <!-- 统一配置delimiters -->
                    <delimiters>
                        <delimiter>@</delimiter>
                    </delimiters>
                    <!-- 统一配置资源过滤的编码 -->
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </plugins>

    <!-- 所有子模块执行资源插件时，会使用上述配置 -->
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <executions>
                <execution>
                    <id>default-resources</id>
                    <phase>process-resources</phase>
                    <goals>
                        <goal>resources</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

这种方式的优势是：将 delimiters 配置与插件绑定，更符合 Maven 的插件化设计思想，且便于通过插件参数调整过滤行为。

## 五、最佳实践与注意事项

### 1. 最佳实践

#### （1）严格遵循 “父 POM 统一配置，子模块继承”

- 所有全局规则（delimiters、资源过滤、Profile、插件版本）都在父 POM 中定义，子模块仅负责业务逻辑和专属依赖。
- 避免子模块重复配置构建规则，减少维护成本。

#### （2）统一占位符格式

- 推荐使用 `@变量名@` 作为 Maven 的占位符，与 Spring Boot 的 `${变量名}` 隔离，避免冲突。
- 所有子模块使用相同的占位符格式，保证团队协作的一致性。

#### （3）按需过滤资源

- 仅对配置文件（`application*.yml`、`*.properties`）开启过滤，静态资源（JS/CSS/ 图片）关闭过滤，提高构建效率。

#### （4）使用 `pluginManagement` 统一插件版本

- 在父 POM 中通过 `pluginManagement` 管理所有插件的版本和配置，子模块直接引用插件，避免版本冲突。

### 2. 注意事项

#### （1）子模块的 `<relativePath>` 配置

- 子模块的 `<parent>` 节点中，`relativePath` 需正确指向父 POM 的路径（如 `../pom.xml`），否则 Maven 会从本地仓库或远程仓库查找父 POM，导致配置失效。

#### （2）多模块构建的顺序

- Maven 会根据子模块的依赖关系自动确定构建顺序，无需手动指定，但需保证依赖关系的正确性。

#### （3）资源目录的一致性

- 所有子模块的资源目录建议统一为 `src/main/resources`，避免自定义资源目录导致父 POM 的配置无法生效。

#### （4）敏感信息的处理

- 生产环境的密码等敏感信息，通过命令行 `-D` 参数或 `~/.m2/settings.xml` 传递，不写在父 POM 的 Profile 中。

## 六、总结

在多模块 Maven 项目中统一管理 delimiters 的核心是**利用父 POM 的继承机制**，将 delimiters、资源过滤、Profile 等配置集中定义，子模块直接继承使用。关键要点：

1. **父 POM 角色**：作为聚合工程和父工程，定义全局的 delimiters、资源过滤、插件配置和 Profile。
2. **子模块角色**：继承父 POM 配置，仅关注业务依赖和模块专属配置。
3. **冲突避免**：使用 `@变量名@` 作为 Maven 占位符，与 Spring Boot 的 `${变量名}` 隔离。
4. **灵活性**：特殊模块可局部覆盖配置，实现全局统一与局部定制的平衡。

这种方式不仅保证了多模块项目中 delimiters 的统一管理，还能提升项目的可维护性和构建效率，是企业级多模块项目的最佳实践。
