# Maven 详解 - 从零到精通

> 本文档系统讲解 Apache Maven 的核心概念、构建生命周期、插件体系、依赖管理和企业级最佳实践，适合 Java 开发者深入掌握构建工具。

---

## 目录

- [Part 1: Maven 概述与安装](#part-1-maven-概述与安装)
- [Part 2: POM 文件深度解析](#part-2-pom-文件深度解析)
- [Part 3: Maven 生命周期](#part-3-maven-生命周期)
- [Part 4: Maven 插件深度解析](#part-4-maven-插件深度解析)
- [Part 5: 依赖管理进阶](#part-5-依赖管理进阶)
- [Part 6: 多模块项目](#part-6-多模块项目)
- [Part 7: Profile 环境配置](#part-7-profile-环境配置)
- [Part 8: 私服 Nexus 配置](#part-8-私服-nexus-配置)
- [Part 9: Maven Wrapper](#part-9-maven-wrapper)
- [Part 10: 构建优化](#part-10-构建优化)
- [Part 11: 常见面试题 FAQ](#part-11-常见面试题-faq)

---

## Part 1: Maven 概述与安装

### 1.1 什么是 Maven

Apache Maven 是一个基于项目对象模型（POM）的**项目管理和构建自动化工具**。它提供了：

- **依赖管理**：自动下载、管理项目依赖及传递性依赖
- **标准构建生命周期**：compile → test → package → install → deploy
- **插件体系**：通过插件扩展构建能力（编译、测试、打包、部署等）
- **项目结构规范**：约定优于配置（Convention over Configuration）
- **多模块支持**：统一管理大型项目的多个模块

### 1.2 Maven vs Gradle vs Ant

| 特性 | Maven | Gradle | Ant |
|------|-------|--------|-----|
| 配置格式 | XML（pom.xml） | Groovy/Kotlin DSL | XML（build.xml） |
| 构建范式 | 声明式 | 命令式+声明式 | 命令式 |
| 依赖管理 | 内置（强大） | 内置（灵活） | 需 Ivy 插件 |
| 生命周期 | 固定（3套） | 灵活自定义 | 无内置 |
| 构建速度 | 较慢 | 快（增量/缓存） | 较快 |
| 学习曲线 | 中等 | 较陡 | 平缓 |
| 生态 | 极丰富 | 丰富（尤其Android） | 较少 |
| 适用场景 | Java 企业项目 | Android/大型项目 | 遗留项目 |

### 1.3 安装与配置

#### 安装 Maven

```bash
# macOS（使用 Homebrew）
brew install maven

# Ubuntu/Debian
sudo apt update && sudo apt install maven

# Windows（使用 Chocolatey）
choco install maven

# 手动安装
# 1. 下载 https://maven.apache.org/download.cgi
# 2. 解压到 /opt/maven（Linux）或 C:\maven（Windows）
# 3. 配置环境变量：
export MAVEN_HOME=/opt/maven/apache-maven-3.9.6
export PATH=$PATH:$MAVEN_HOME/bin

# 验证安装
mvn -version
# Apache Maven 3.9.6 (bc0240f3c744dd6b6ec2920b3cd08dcc624952a9)
# Maven home: /opt/homebrew/Cellar/maven/3.9.6/libexec
# Java version: 17.0.9, vendor: Eclipse Adoptium
```

#### Maven 目录结构

```
$MAVEN_HOME/
├── bin/
│   ├── mvn          # 主命令（Linux/Mac）
│   └── mvn.cmd      # 主命令（Windows）
├── boot/            # 类加载器
├── conf/
│   └── settings.xml # Maven 全局配置
├── lib/             # Maven 核心 JAR
└── LICENSE
```

#### settings.xml 配置

Maven 有两个 settings.xml：
- 全局：`$MAVEN_HOME/conf/settings.xml`
- 用户：`~/.m2/settings.xml`（优先级更高）

```xml
<!-- ~/.m2/settings.xml -->
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">

  <!-- 本地仓库路径（默认 ~/.m2/repository） -->
  <localRepository>/data/maven/repository</localRepository>

  <!-- 镜像配置（使用阿里云加速） -->
  <mirrors>
    <mirror>
      <id>aliyun</id>
      <name>Aliyun Maven Mirror</name>
      <url>https://maven.aliyun.com/repository/public</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>

  <!-- JDK 版本配置 -->
  <profiles>
    <profile>
      <id>jdk-17</id>
      <activation>
        <activeByDefault>true</activeByDefault>
        <jdk>17</jdk>
      </activation>
      <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <maven.compiler.compilerVersion>17</maven.compiler.compilerVersion>
      </properties>
    </profile>
  </profiles>

</settings>
```

### 1.4 标准目录结构

```
my-project/
├── pom.xml                          # 项目对象模型
├── src/
│   ├── main/
│   │   ├── java/                    # 主代码
│   │   │   └── com/example/App.java
│   │   └── resources/               # 主资源文件
│   │       ├── application.yml
│   │       └── logback.xml
│   └── test/
│       ├── java/                    # 测试代码
│       │   └── com/example/AppTest.java
│       └── resources/               # 测试资源
│           └── test-application.yml
└── target/                          # 构建输出（不提交到 Git）
    ├── classes/                     # 编译后的 .class 文件
    ├── test-classes/
    ├── my-project-1.0.0.jar         # 打包产物
    └── surefire-reports/            # 测试报告
```

---

## Part 2: POM 文件深度解析

### 2.1 POM 文件结构总览

POM（Project Object Model）是 Maven 的核心概念，每个 Maven 项目都有一个 pom.xml 文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

  <!-- POM 模型版本，固定为 4.0.0 -->
  <modelVersion>4.0.0</modelVersion>

  <!-- ============ 项目坐标 ============ -->
  <groupId>com.example</groupId>       <!-- 组织/公司 -->
  <artifactId>my-project</artifactId>  <!-- 项目名 -->
  <version>1.0.0-SNAPSHOT</version>   <!-- 版本号 -->
  <packaging>jar</packaging>           <!-- 打包方式 -->

  <!-- ============ 项目信息 ============ -->
  <name>My Project</name>
  <description>A sample Maven project</description>
  <url>https://github.com/example/my-project</url>

  <!-- ============ 父项目（继承） ============ -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
  </parent>

  <!-- ============ 属性（变量） ============ -->
  <properties>
    <java.version>17</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <lombok.version>1.18.30</lombok.version>
  </properties>

  <!-- ============ 依赖管理（子模块继承） ============ -->
  <dependencyManagement>
    <dependencies>
      <!-- 版本锁定，子模块使用时无需指定版本 -->
    </dependencies>
  </dependencyManagement>

  <!-- ============ 依赖 ============ -->
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>

  <!-- ============ 构建配置 ============ -->
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>

</project>
```

### 2.2 GAV 坐标详解

Maven 用三个坐标唯一标识一个构件（artifact）：

| 元素 | 说明 | 示例 |
|------|------|------|
| `groupId` | 组织/公司标识，通常为反向域名 | `com.example`, `org.springframework` |
| `artifactId` | 项目名，同一 groupId 下唯一 | `spring-boot-starter-web` |
| `version` | 版本号，遵循语义化版本 | `3.2.0`, `1.0.0-SNAPSHOT` |

**版本号规范**：
- `SNAPSHOT`：开发中的快照版本，可被覆盖（如 `1.0.0-SNAPSHOT`）
- `RELEASE`：正式发布版本，不可修改
- 语义化版本：`MAJOR.MINOR.PATCH`（如 `2.3.1`）

```bash
# 本地仓库路径规则
# ~/.m2/repository/{groupId替换.为/}/{artifactId}/{version}/{artifactId}-{version}.jar
# 例如：
# ~/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.2.0/spring-boot-starter-web-3.2.0.jar
```

### 2.3 依赖配置详解

```xml
<dependencies>

  <!-- 基础依赖（compile scope，默认） -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 无需指定 version（Spring Boot Parent 管理） -->
  </dependency>

  <!-- 只在测试时使用 -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>

  <!-- 只在编译时使用，不打包进最终产物 -->
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <scope>provided</scope>
    <optional>true</optional>
  </dependency>

  <!-- 运行时需要，编译时不需要 -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
  </dependency>

  <!-- 系统范围（不推荐，指向本地 JAR） -->
  <dependency>
    <groupId>com.oracle</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>19.3</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/libs/ojdbc8.jar</systemPath>
  </dependency>

  <!-- 排除传递依赖 -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
    <exclusions>
      <exclusion>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
      </exclusion>
    </exclusions>
  </dependency>

</dependencies>
```

#### 依赖 Scope 总结

| Scope | 编译 | 测试 | 运行 | 打包 | 说明 |
|-------|------|------|------|------|------|
| `compile` | ✓ | ✓ | ✓ | ✓ | 默认值，最常用 |
| `test` | ✗ | ✓ | ✗ | ✗ | 仅测试阶段 |
| `provided` | ✓ | ✓ | ✗ | ✗ | 运行环境已提供（如 Servlet API） |
| `runtime` | ✗ | ✓ | ✓ | ✓ | 仅运行时（如 JDBC 驱动） |
| `system` | ✓ | ✓ | ✗ | ✗ | 本地 JAR，不推荐 |
| `import` | — | — | — | — | 仅用于 `<dependencyManagement>` 中的 BOM |

### 2.4 属性（Properties）

```xml
<properties>
  <!-- 版本属性（统一管理） -->
  <java.version>17</java.version>
  <spring-cloud.version>2023.0.0</spring-cloud.version>
  <mybatis-plus.version>3.5.5</mybatis-plus.version>

  <!-- 编码设置 -->
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

  <!-- Maven 编译配置 -->
  <maven.compiler.source>${java.version}</maven.compiler.source>
  <maven.compiler.target>${java.version}</maven.compiler.target>
  <maven.test.skip>false</maven.test.skip>
</properties>
```

**内置属性**：
- `${project.version}`：当前项目版本
- `${project.groupId}`：当前项目 groupId
- `${project.basedir}`：项目根目录
- `${project.build.directory}`：构建输出目录（target）
- `${settings.localRepository}`：本地仓库路径
- `${env.JAVA_HOME}`：环境变量

---

## Part 3: Maven 生命周期

### 3.1 三套生命周期

Maven 有三套相互独立的生命周期：

| 生命周期 | 说明 | 常用阶段 |
|---------|------|----------|
| **default** | 主要构建生命周期 | compile, test, package, install, deploy |
| **clean** | 清理构建产物 | pre-clean, clean, post-clean |
| **site** | 生成项目文档站点 | pre-site, site, post-site, site-deploy |

### 3.2 Default 生命周期阶段

Default 生命周期包含23个阶段，按序执行（执行某阶段会先执行之前所有阶段）：

```
1.  validate          验证项目配置是否正确
2.  initialize        初始化构建状态
3.  generate-sources  生成源代码（如 JAXB、protobuf）
4.  process-sources   处理源代码
5.  generate-resources 生成资源文件
6.  process-resources 复制/过滤资源文件到 target/classes
7.  compile           ★ 编译主代码
8.  process-classes   处理编译后的类文件
9.  generate-test-sources  生成测试源代码
10. process-test-sources   处理测试源代码
11. generate-test-resources 生成测试资源
12. process-test-resources  复制/过滤测试资源
13. test-compile      编译测试代码
14. process-test-classes    处理测试类文件
15. test              ★ 运行单元测试
16. prepare-package   打包前准备
17. package           ★ 打包（生成 jar/war）
18. pre-integration-test    集成测试前
19. integration-test  运行集成测试
20. post-integration-test   集成测试后
21. verify            验证包的有效性
22. install           ★ 安装到本地仓库（~/.m2）
23. deploy            ★ 部署到远程仓库
```

### 3.3 常用 Maven 命令

```bash
# 清理
mvn clean

# 编译
mvn compile

# 编译 + 测试
mvn test

# 打包（跳过测试）
mvn package -DskipTests

# 安装到本地仓库
mvn install

# 部署到远程仓库
mvn deploy

# 清理 + 打包（最常用）
mvn clean package

# 清理 + 安装（多模块项目常用）
mvn clean install

# 跳过所有测试
mvn clean package -DskipTests

# 跳过测试编译和执行
mvn clean package -Dmaven.test.skip=true

# 生成项目站点
mvn site

# 查看依赖树
mvn dependency:tree

# 分析依赖（未使用/缺失）
mvn dependency:analyze

# 显示有更新的依赖
mvn versions:display-dependency-updates

# 显示插件更新
mvn versions:display-plugin-updates

# 查看有效 POM（合并父 POM 后的完整 POM）
mvn help:effective-pom

# 查看所有激活的 profile
mvn help:active-profiles

# 调试模式（显示详细日志）
mvn clean package -X

# 静默模式（只显示错误）
mvn clean package -q

# 不从远程仓库更新（离线模式）
mvn clean package -o

# 更新 SNAPSHOT 依赖
mvn clean package -U
```

### 3.4 阶段绑定与插件

生命周期阶段本身不做任何事情，具体工作由绑定到该阶段的**插件目标（goal）**完成：

```
生命周期阶段         绑定的默认插件 goal
compile          → maven-compiler-plugin:compile
test             → maven-surefire-plugin:test
package(jar)     → maven-jar-plugin:jar
package(war)     → maven-war-plugin:war
install          → maven-install-plugin:install
deploy           → maven-deploy-plugin:deploy
clean            → maven-clean-plugin:clean
```

```bash
# 直接执行插件 goal（不触发生命周期）
mvn compiler:compile        # 只编译
mvn surefire:test           # 只运行测试
mvn dependency:tree         # 显示依赖树
mvn spring-boot:run         # 运行 Spring Boot 应用
```

---

## Part 4: Maven 插件深度解析

### 4.1 maven-compiler-plugin

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.12.1</version>
  <configuration>
    <source>17</source>
    <target>17</target>
    <!-- 或使用更现代的方式 -->
    <release>17</release>
    <encoding>UTF-8</encoding>
    <!-- 启用增量编译 -->
    <useIncrementalCompilation>true</useIncrementalCompilation>
    <!-- 编译参数 -->
    <compilerArgs>
      <arg>-parameters</arg>  <!-- 保留方法参数名 -->
      <arg>-Xlint:all</arg>   <!-- 启用所有警告 -->
    </compilerArgs>
    <!-- 注解处理器路径（如 Lombok、MapStruct） -->
    <annotationProcessorPaths>
      <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
      </path>
      <path>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${mapstruct.version}</version>
      </path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

### 4.2 maven-surefire-plugin（单元测试）

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.2.3</version>
  <configuration>
    <!-- 并行运行测试 -->
    <parallel>methods</parallel>
    <threadCount>4</threadCount>

    <!-- 包含/排除测试类 -->
    <includes>
      <include>**/*Test.java</include>
      <include>**/*Tests.java</include>
    </includes>
    <excludes>
      <exclude>**/*IntegrationTest.java</exclude>
    </excludes>

    <!-- JVM 参数 -->
    <argLine>-Xmx1024m -XX:MaxPermSize=256m</argLine>

    <!-- 失败时继续（不中止构建） -->
    <testFailureIgnore>false</testFailureIgnore>

    <!-- 测试环境变量 -->
    <environmentVariables>
      <SPRING_PROFILES_ACTIVE>test</SPRING_PROFILES_ACTIVE>
    </environmentVariables>
  </configuration>
</plugin>
```

### 4.3 maven-failsafe-plugin（集成测试）

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-failsafe-plugin</artifactId>
  <version>3.2.3</version>
  <executions>
    <execution>
      <goals>
        <goal>integration-test</goal>
        <goal>verify</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <includes>
      <include>**/*IT.java</include>
      <include>**/*IntegrationTest.java</include>
    </includes>
  </configuration>
</plugin>
```

### 4.4 maven-jar-plugin

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <configuration>
    <archive>
      <manifest>
        <!-- 可执行 JAR -->
        <addClasspath>true</addClasspath>
        <classpathPrefix>lib/</classpathPrefix>
        <mainClass>com.example.Application</mainClass>
      </manifest>
      <manifestEntries>
        <Implementation-Version>${project.version}</Implementation-Version>
        <Build-Time>${maven.build.timestamp}</Build-Time>
        <Git-Commit>${git.commit.id.abbrev}</Git-Commit>
      </manifestEntries>
    </archive>
    <!-- 排除不需要的文件 -->
    <excludes>
      <exclude>**/*.properties</exclude>
    </excludes>
  </configuration>
</plugin>
```

### 4.5 spring-boot-maven-plugin

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <!-- 主类（通常自动检测） -->
    <mainClass>com.example.Application</mainClass>
    <!-- 排除 Lombok 等仅编译时依赖 -->
    <excludes>
      <exclude>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
      </exclude>
    </excludes>
    <!-- 构建分层 JAR（Docker 优化） -->
    <layers>
      <enabled>true</enabled>
    </layers>
  </configuration>
  <executions>
    <execution>
      <goals>
        <!-- 构建可执行 JAR -->
        <goal>repackage</goal>
        <!-- 在 package 阶段注入构建信息 -->
        <goal>build-info</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

```bash
# 运行 Spring Boot 应用
mvn spring-boot:run

# 指定 profile
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# 构建 Docker 镜像（Buildpacks）
mvn spring-boot:build-image
```

### 4.6 maven-resources-plugin

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-resources-plugin</artifactId>
  <configuration>
    <encoding>UTF-8</encoding>
    <!-- 资源过滤（替换 ${} 变量） -->
    <useDefaultDelimiters>true</useDefaultDelimiters>
  </configuration>
</plugin>

<!-- 在 resources 中开启过滤 -->
<build>
  <resources>
    <resource>
      <directory>src/main/resources</directory>
      <filtering>true</filtering>  <!-- 开启变量替换 -->
      <includes>
        <include>**/*.yml</include>
        <include>**/*.properties</include>
      </includes>
    </resource>
    <resource>
      <directory>src/main/resources</directory>
      <filtering>false</filtering>  <!-- 二进制文件不过滤 -->
      <includes>
        <include>**/*.png</include>
        <include>**/*.pdf</include>
      </includes>
    </resource>
  </resources>
</build>
```

### 4.7 jacoco-maven-plugin（代码覆盖率）

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <executions>
    <execution>
      <id>prepare-agent</id>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals><goal>report</goal></goals>
    </execution>
    <!-- 覆盖率门控（低于阈值则构建失败） -->
    <execution>
      <id>jacoco-check</id>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>  <!-- 行覆盖率 >= 80% -->
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

---

## Part 5: 依赖管理进阶

### 5.1 传递性依赖

Maven 自动引入依赖的依赖（传递性依赖），最深可传递多层。

```bash
# 查看完整依赖树
mvn dependency:tree

# 只显示特定依赖的传递路径
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-core

# 输出到文件
mvn dependency:tree -Doutput=dependency-tree.txt
```

**传递性依赖 Scope 规则**：

| 直接依赖 Scope \ 传递依赖 Scope | compile | runtime | provided | test |
|--------------------------------|---------|---------|----------|------|
| **compile** | compile | runtime | — | — |
| **runtime** | runtime | runtime | — | — |
| **provided** | provided | provided | — | — |
| **test** | test | test | — | — |

（— 表示该传递依赖不会被引入）

### 5.2 依赖冲突与仲裁

当同一依赖有多个版本时，Maven 使用以下规则仲裁：

**规则1：最近声明原则（Nearest Definition）**
依赖树中距离项目最近的版本获胜。

```
例：项目 → A 1.0 → C 2.0
    项目 → B 1.0 → D 1.0 → C 3.0

C 2.0 距离更近（2步），所以 C 2.0 获胜。
```

**规则2：先声明原则（First Declaration）**
如果距离相同，POM 中先声明的依赖获胜。

```bash
# 诊断依赖冲突
mvn dependency:tree -Dverbose

# 找出某依赖被谁引入
mvn dependency:tree -Dincludes=log4j:log4j
```

**解决冲突的方式**：
```xml
<!-- 方式1：直接声明版本（最优先） -->
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>2.0.9</version>  <!-- 明确指定版本 -->
</dependency>

<!-- 方式2：在 dependencyManagement 中锁定版本 -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>2.0.9</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<!-- 方式3：排除冲突的传递依赖 -->
<dependency>
  <groupId>some.lib</groupId>
  <artifactId>some-artifact</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

### 5.3 BOM（Bill of Materials）

BOM 是一种特殊的 POM，只包含 `dependencyManagement`，用于统一管理一组相关依赖的版本：

```xml
<!-- 引入 BOM（使用 import scope） -->
<dependencyManagement>
  <dependencies>
    <!-- Spring Boot BOM -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <!-- Spring Cloud BOM -->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<!-- 使用时无需指定版本 -->
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
</dependencies>
```

### 5.4 自定义 BOM

为团队内部库创建 BOM，统一管理版本：

```xml
<!-- company-bom/pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.company</groupId>
  <artifactId>company-bom</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>
  <name>Company BOM</name>

  <dependencyManagement>
    <dependencies>
      <!-- 统一管理公司内部库版本 -->
      <dependency>
        <groupId>com.company</groupId>
        <artifactId>common-utils</artifactId>
        <version>2.5.0</version>
      </dependency>
      <dependency>
        <groupId>com.company</groupId>
        <artifactId>security-starter</artifactId>
        <version>1.3.0</version>
      </dependency>
      <!-- 第三方库版本锁定 -->
      <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson2</artifactId>
        <version>2.0.43</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

### 5.5 依赖分析工具

```bash
# 分析未使用和缺失的依赖
mvn dependency:analyze
# Used undeclared: 使用了但未声明的依赖（依赖了传递依赖，不稳定）
# Unused declared: 声明了但未使用的依赖（可以移除）

# 复制依赖到 target/dependency
mvn dependency:copy-dependencies

# 查看某依赖的完整依赖路径
mvn dependency:tree -Dverbose -Dincludes=com.example:some-lib

# 使用 OWASP 安全扫描
mvn org.owasp:dependency-check-maven:check
```

---

## Part 6: 多模块项目

### 6.1 多模块项目结构

```
company-platform/              ← 父模块（聚合+继承）
├── pom.xml                    ← 父 POM
├── platform-common/           ← 公共工具模块
│   └── pom.xml
├── platform-security/         ← 安全模块
│   └── pom.xml
├── platform-api/              ← API 接口定义
│   └── pom.xml
├── platform-service/          ← 业务服务层
│   └── pom.xml
└── platform-web/              ← Web 启动模块
    └── pom.xml
```

### 6.2 父 POM 配置

```xml
<!-- company-platform/pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.company</groupId>
  <artifactId>company-platform</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>  <!-- 必须是 pom -->

  <!-- 继承 Spring Boot Parent -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
  </parent>

  <!-- 子模块列表（聚合） -->
  <modules>
    <module>platform-common</module>
    <module>platform-security</module>
    <module>platform-api</module>
    <module>platform-service</module>
    <module>platform-web</module>
  </modules>

  <properties>
    <java.version>17</java.version>
    <platform.version>1.0.0-SNAPSHOT</platform.version>
    <mybatis-plus.version>3.5.5</mybatis-plus.version>
  </properties>

  <!-- 统一依赖版本（子模块继承，不自动引入） -->
  <dependencyManagement>
    <dependencies>
      <!-- 内部模块互相引用 -->
      <dependency>
        <groupId>com.company</groupId>
        <artifactId>platform-common</artifactId>
        <version>${platform.version}</version>
      </dependency>
      <dependency>
        <groupId>com.company</groupId>
        <artifactId>platform-security</artifactId>
        <version>${platform.version}</version>
      </dependency>
      <!-- 第三方依赖版本 -->
      <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>${mybatis-plus.version}</version>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <!-- 所有子模块共享的依赖（直接引入） -->
  <dependencies>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>
  </dependencies>

  <!-- 所有子模块共享的插件配置 -->
  <build>
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-compiler-plugin</artifactId>
          <configuration>
            <source>${java.version}</source>
            <target>${java.version}</target>
          </configuration>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>

</project>
```

### 6.3 子模块 POM 配置

```xml
<!-- platform-web/pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>

  <!-- 声明父模块 -->
  <parent>
    <groupId>com.company</groupId>
    <artifactId>company-platform</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <!-- 相对路径（默认 ../pom.xml，可省略） -->
    <relativePath>../pom.xml</relativePath>
  </parent>

  <!-- 子模块 artifactId（groupId 和 version 从父模块继承） -->
  <artifactId>platform-web</artifactId>
  <packaging>jar</packaging>

  <dependencies>
    <!-- 引用内部模块 -->
    <dependency>
      <groupId>com.company</groupId>
      <artifactId>platform-service</artifactId>
    </dependency>
    <!-- Spring Web（版本由父 POM 管理） -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

### 6.4 多模块构建命令

```bash
# 构建整个多模块项目（在根目录执行）
mvn clean install

# 只构建指定模块及其依赖
mvn clean install -pl platform-web -am
# -pl: 指定模块（platform-web）
# -am: also-make，同时构建依赖的模块

# 只构建指定模块及依赖它的模块
mvn clean install -pl platform-common -amd
# -amd: also-make-dependents

# 排除某模块
mvn clean install -pl !platform-web

# 并行构建（4线程）
mvn clean install -T 4

# 自动并行（使用 CPU 核数）
mvn clean install -T 1C
```

### 6.5 版本管理

```bash
# 统一修改所有模块版本
mvn versions:set -DnewVersion=1.1.0-SNAPSHOT

# 确认版本变更
mvn versions:commit

# 回滚版本变更
mvn versions:revert

# 修改父模块版本
mvn versions:update-parent -DparentVersion=3.2.1

# 查看可更新的依赖
mvn versions:display-dependency-updates
```

---

## Part 7: Profile 环境配置

### 7.1 Profile 基础

Maven Profile 用于根据不同环境激活不同配置，支持：自定义 properties、激活不同依赖、使用不同的 plugin 配置、过滤不同的资源文件。

```xml
<profiles>

  <!-- 开发环境 -->
  <profile>
    <id>dev</id>
    <activation>
      <activeByDefault>true</activeByDefault>  <!-- 默认激活 -->
    </activation>
    <properties>
      <spring.profiles.active>dev</spring.profiles.active>
      <db.url>jdbc:mysql://localhost:3306/devdb</db.url>
      <log.level>DEBUG</log.level>
    </properties>
    <dependencies>
      <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
      </dependency>
    </dependencies>
  </profile>

  <!-- 测试环境 -->
  <profile>
    <id>test</id>
    <properties>
      <spring.profiles.active>test</spring.profiles.active>
      <db.url>jdbc:mysql://test-db:3306/testdb</db.url>
      <log.level>INFO</log.level>
    </properties>
  </profile>

  <!-- 生产环境 -->
  <profile>
    <id>prod</id>
    <properties>
      <spring.profiles.active>prod</spring.profiles.active>
      <db.url>jdbc:mysql://prod-db:3306/proddb</db.url>
      <log.level>WARN</log.level>
    </properties>
    <build>
      <plugins>
        <!-- 生产环境开启代码混淆 -->
        <plugin>
          <groupId>com.github.wvengen</groupId>
          <artifactId>proguard-maven-plugin</artifactId>
          <executions>
            <execution>
              <phase>package</phase>
              <goals><goal>proguard</goal></goals>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </build>
  </profile>

</profiles>
```

### 7.2 激活 Profile

```bash
# 命令行激活
mvn clean package -P prod
mvn clean package -P prod,security  # 同时激活多个
mvn clean package -P !dev           # 禁用指定 profile

# 查看激活的 profile
mvn help:active-profiles
```

### 7.3 自动激活条件

```xml
<profiles>

  <!-- 根据 JDK 版本激活 -->
  <profile>
    <id>jdk17-features</id>
    <activation>
      <jdk>[17,)</jdk>  <!-- JDK 17 及以上 -->
    </activation>
  </profile>

  <!-- 根据操作系统激活 -->
  <profile>
    <id>windows-build</id>
    <activation>
      <os>
        <name>Windows 10</name>
        <family>windows</family>
      </os>
    </activation>
  </profile>

  <!-- 根据属性激活 -->
  <profile>
    <id>integration-tests</id>
    <activation>
      <property>
        <name>run.integration.tests</name>
        <value>true</value>
      </property>
    </activation>
  </profile>

  <!-- 根据文件存在激活 -->
  <profile>
    <id>local-config</id>
    <activation>
      <file>
        <exists>${basedir}/local.properties</exists>
      </file>
    </activation>
  </profile>

</profiles>
```

### 7.4 Profile 与资源过滤结合

```xml
<build>
  <resources>
    <resource>
      <directory>src/main/resources</directory>
      <filtering>true</filtering>
    </resource>
  </resources>
</build>

<profiles>
  <profile>
    <id>prod</id>
    <build>
      <resources>
        <!-- 使用生产环境专用配置目录覆盖 -->
        <resource>
          <directory>src/main/resources-prod</directory>
          <filtering>true</filtering>
        </resource>
      </resources>
    </build>
  </profile>
</profiles>
```

```
src/main/
├── resources/
│   └── application.yml           # 通用配置
├── resources-dev/
│   └── application-dev.yml       # dev 专用
└── resources-prod/
    └── application-prod.yml      # prod 专用
```

---

## Part 8: 私服 Nexus 配置

### 8.1 为什么需要私服

| 需求 | 私服解决方案 |
|------|-------------|
| 加速依赖下载 | 代理中央仓库，缓存到内网 |
| 内部库共享 | 发布内部 artifacts 供团队使用 |
| 安全控制 | 审批依赖，防止引入问题组件 |
| 离线构建 | 内网不访问公网时可从私服拉取 |
| 版本控制 | 锁定可用版本，避免随意升级 |

### 8.2 Nexus Repository Manager 安装

```bash
# Docker 安装
docker run -d \
  --name nexus \
  -p 8081:8081 \
  -v nexus-data:/nexus-data \
  sonatype/nexus3:latest

# 获取初始密码
docker exec nexus cat /nexus-data/admin.password

# 访问 http://localhost:8081
```

**Nexus 仓库类型**：
- **hosted**：存储本地发布的构件
- **proxy**：代理远程仓库（如 Maven Central）
- **group**：组合多个仓库，统一入口

### 8.3 配置 Maven 使用私服

```xml
<!-- ~/.m2/settings.xml -->
<settings>

  <!-- 私服认证信息 -->
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
  </servers>

  <!-- 镜像配置：将所有请求转向私服 -->
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>  <!-- 代理所有仓库 -->
      <url>http://nexus.company.com:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>

  <!-- profile 中配置仓库 -->
  <profiles>
    <profile>
      <id>nexus</id>
      <repositories>
        <repository>
          <id>nexus-releases</id>
          <url>http://nexus.company.com:8081/repository/maven-releases/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
        <repository>
          <id>nexus-snapshots</id>
          <url>http://nexus.company.com:8081/repository/maven-snapshots/</url>
          <releases><enabled>false</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>

</settings>
```

### 8.4 发布到私服

```xml
<!-- pom.xml 中配置发布目标 -->
<distributionManagement>
  <repository>
    <id>nexus-releases</id>  <!-- 与 settings.xml 中 server id 一致 -->
    <name>Nexus Release Repository</name>
    <url>http://nexus.company.com:8081/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <name>Nexus Snapshot Repository</name>
    <url>http://nexus.company.com:8081/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

```bash
# 发布 SNAPSHOT 版本
mvn clean deploy

# 发布 RELEASE 版本
mvn versions:set -DnewVersion=1.0.0
mvn clean deploy

# 使用 maven-release-plugin 自动化发布流程
mvn release:prepare  # 设置版本、打 tag、更新到下一 SNAPSHOT
mvn release:perform  # 构建 tag 版本并 deploy
```

### 8.5 使用阿里云镜像（国内推荐）

```xml
<!-- ~/.m2/settings.xml -->
<mirrors>
  <mirror>
    <id>aliyun</id>
    <mirrorOf>central</mirrorOf>
    <name>阿里云 Maven 镜像</name>
    <url>https://maven.aliyun.com/repository/public</url>
  </mirror>
</mirrors>
```

---

## Part 9: Maven Wrapper

### 9.1 什么是 Maven Wrapper

Maven Wrapper（mvnw）允许项目自带 Maven，无需预先安装 Maven，团队成员使用统一版本。

```bash
# 初始化 Maven Wrapper
mvn wrapper:wrapper

# 指定特定版本
mvn wrapper:wrapper -Dmaven=3.9.6
```

生成的文件：
```
project/
├── mvnw           # Linux/Mac 可执行脚本
├── mvnw.cmd       # Windows 可执行脚本
└── .mvn/
    └── wrapper/
        └── maven-wrapper.properties  # 指定 Maven 版本和下载 URL
```

### 9.2 使用 Maven Wrapper

```bash
# Linux/Mac
./mvnw clean package
./mvnw spring-boot:run

# Windows
.\mvnw.cmd clean package

# 其余命令与 mvn 完全相同
./mvnw -P prod clean deploy
```

### 9.3 .mvn/wrapper/maven-wrapper.properties

```properties
# 指定 Maven 版本
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip

# 可选：指定下载存储路径
distributionPath=wrapper/dists

# 可选：使用内网镜像
distributionUrl=http://nexus.company.com:8081/repository/maven-public/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
```

---

## Part 10: 构建优化

### 10.1 并行构建

```bash
# 使用4个线程并行构建
mvn clean install -T 4

# 使用 CPU 核数的线程
mvn clean install -T 1C

# 注意：并行构建要求模块间依赖正确声明，且无共享状态
```

### 10.2 跳过不必要的步骤

```bash
# 跳过测试执行（编译测试代码）
mvn clean package -DskipTests

# 完全跳过测试（不编译不执行）
mvn clean package -Dmaven.test.skip=true

# 跳过 Javadoc 生成
mvn clean package -Dmaven.javadoc.skip=true

# 跳过代码检查
mvn clean package -Dcheckstyle.skip=true -Dfindbugs.skip=true

# CI 环境常用组合
mvn clean package -DskipTests -Dmaven.javadoc.skip=true -q
```

### 10.3 本地仓库缓存优化

```xml
<!-- 减少不必要的远程仓库检查 -->
<repository>
  <id>central</id>
  <snapshots>
    <enabled>false</enabled>
  </snapshots>
  <releases>
    <!-- updatePolicy: always / daily / never / interval:N（分钟） -->
    <updatePolicy>never</updatePolicy>  <!-- release 版本永不更新 -->
  </releases>
</repository>
```

```bash
# 使用离线模式（所有依赖已在本地）
mvn clean package -o

# 强制更新 SNAPSHOT 依赖
mvn clean package -U
```

### 10.4 增量编译

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <!-- 启用增量编译（默认已开启） -->
    <useIncrementalCompilation>true</useIncrementalCompilation>
  </configuration>
</plugin>
```

### 10.5 Maven Daemon（mvnd）

Maven Daemon 是 Maven 的守护进程模式，显著提升构建速度（类似 Gradle Daemon）：

```bash
# 安装 mvnd
# 下载：https://github.com/apache/maven-mvnd/releases

# 使用（命令与 mvn 兼容）
mvnd clean package
mvnd clean install -T 1C

# 停止 daemon
mvnd --stop

# 查看 daemon 状态
mvnd --status
```

性能提升原因：
- JVM 热身后不需要重启
- 并行模块构建
- 更好的进度输出

### 10.6 Docker 分层构建优化

```dockerfile
# 利用 Spring Boot 分层 JAR 优化 Docker 镜像构建
FROM eclipse-temurin:17-jre AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

分层顺序（不经常变更的层靠前）：
1. `dependencies`：第三方依赖（最少变更）
2. `spring-boot-loader`：Spring Boot 加载器
3. `snapshot-dependencies`：SNAPSHOT 依赖
4. `application`：业务代码（最频繁变更）

---

## Part 11: 常见面试题 FAQ

### Q1. Maven 的依赖调解（Dependency Mediation）规则是什么？

**答**：Maven 使用两条规则调解版本冲突：

1. **最近定义原则（Nearest Definition）**：在依赖树中，距离当前项目最近（路径最短）的版本获胜。
   - 项目 → A → C:2.0（距离2）
   - 项目 → B → D → C:3.0（距离3）
   - C:2.0 获胜

2. **先声明原则**：如果路径长度相同，POM 中先声明的获胜。

**最佳实践**：在项目 POM 的 `<dependencyManagement>` 中明确锁定版本，优先级最高。

### Q2. compile、provided、runtime、test scope 的区别？

**答**：
- `compile`（默认）：编译、测试、运行时都有效，打包进最终产物。
- `provided`：编译和测试时有效，运行时由容器提供（如 Servlet API 由 Tomcat 提供），不打包。
- `runtime`：测试和运行时有效，编译时不需要（如 JDBC 驱动），打包进产物。
- `test`：仅测试编译和执行时有效（如 JUnit），不打包。

### Q3. dependencyManagement 和 dependencies 的区别？

**答**：
- `<dependencyManagement>`：只声明版本和 scope，**不实际引入依赖**。子模块在 `<dependencies>` 中声明时无需指定版本，会自动继承。用于统一管理版本，避免版本散落。
- `<dependencies>`：直接引入依赖，**会实际添加到类路径**。

在父 POM 中，`dependencyManagement` 是版本锁定，`dependencies` 中的依赖所有子模块都会继承。

### Q4. Maven 如何处理 SNAPSHOT 版本？

**答**：SNAPSHOT 版本（如 `1.0.0-SNAPSHOT`）是开发中的不稳定版本：
- 每次构建都会检查远程仓库是否有更新的 SNAPSHOT（默认每天检查一次）
- 在 Nexus 中，SNAPSHOT 版本实际存储为带时间戳的版本（如 `1.0.0-20240315.123456-1`）
- 可以强制更新：`mvn -U`
- SNAPSHOT 版本只能发布到 snapshot 仓库，RELEASE 版本只能发布到 release 仓库

### Q5. mvn install 和 mvn deploy 的区别？

**答**：
- `mvn install`：将构建产物安装到**本地仓库**（`~/.m2/repository`），供本地其他项目使用。
- `mvn deploy`：将构建产物部署到**远程仓库**（Nexus、Artifactory 等），供团队共享。

### Q6. 什么是 Maven 的生命周期？有几个？

**答**：Maven 有三套独立的生命周期：
1. **default**：项目构建的核心生命周期，包含 compile、test、package、install、deploy 等23个阶段。
2. **clean**：清理构建产物，包含 pre-clean、clean、post-clean 3个阶段。
3. **site**：生成项目文档站点，包含 pre-site、site、post-site、site-deploy 4个阶段。

执行某个阶段会自动执行该生命周期中之前的所有阶段（如 `mvn package` 会先执行 compile、test）。

### Q7. 如何排除传递依赖？

**答**：使用 `<exclusions>` 标签：
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-logging</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

常见场景：将默认的 Logback 换成 Log4j2 时，需要排除 `spring-boot-starter-logging`。

### Q8. Maven 插件的 goal 和生命周期阶段是什么关系？

**答**：
- **Goal（目标）**：插件的具体执行单元，如 `maven-compiler-plugin:compile`。
- **Phase（阶段）**：生命周期中的一个步骤，本身不做任何事情。
- **绑定（Binding）**：将 Goal 绑定到 Phase，当执行该 Phase 时自动调用 Goal。

可以直接执行 goal（`mvn compiler:compile`），也可以执行 phase（`mvn compile`，触发绑定到该 phase 的所有 goals）。

### Q9. 什么是 BOM？如何使用？

**答**：BOM（Bill of Materials，物料清单）是一种只包含 `<dependencyManagement>` 的特殊 POM，用于统一管理一组相关依赖的版本。

使用方式：在 `<dependencyManagement>` 中以 `import` scope 引入：
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### Q10. Maven 多模块项目中，-pl 和 -am 参数的作用？

**答**：
- `-pl`（projects list）：指定要构建的模块
- `-am`（also-make）：同时构建指定模块依赖的所有模块
- `-amd`（also-make-dependents）：同时构建依赖指定模块的所有模块

```bash
# 只构建 web 模块及其所有依赖模块
mvn install -pl web -am

# 构建 common 模块及所有依赖它的模块
mvn install -pl common -amd
```

### Q11. 如何查看一个依赖是怎么被引入的？

**答**：
```bash
# 显示完整依赖树（包含 omitted 的版本）
mvn dependency:tree -Dverbose

# 只显示特定依赖的传递路径
mvn dependency:tree -Dincludes=groupId:artifactId

# 例如查找 jackson-core 的引入路径
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-core
```

### Q12. Maven 的 Repository 和 Plugin Repository 是什么？

**答**：
- `<repositories>`：声明下载**项目依赖**（jar、pom）的仓库。
- `<pluginRepositories>`：声明下载 **Maven 插件**的仓库。

两者是分开配置的，因为插件本身也是 Maven 的依赖，可能托管在不同仓库。

### Q13. 如何跳过 Maven 测试？

**答**：有两种方式：
- `-DskipTests`：跳过测试执行，但**仍然编译**测试代码。
- `-Dmaven.test.skip=true`：完全跳过，**不编译也不执行**测试。

推荐使用 `-DskipTests`，因为它能检测测试代码的编译错误，只是不执行测试。

### Q14. Maven 项目中如何管理版本号？

**答**：最佳实践：
1. 在父 POM 的 `<properties>` 中定义版本变量（如 `<spring.version>3.2.0</spring.version>`）
2. 在 `<dependencyManagement>` 中使用 `${spring.version}` 引用
3. 子模块在 `<dependencies>` 中声明时无需指定版本
4. 使用 `mvn versions:set -DnewVersion=x.x.x` 统一修改所有模块版本
5. 使用 BOM 管理一组相关依赖

### Q15. 什么是 Maven 的 effective POM？

**答**：Effective POM 是当前项目实际生效的 POM，是项目 POM + 所有父 POM（包括 Super POM）合并的结果。

```bash
# 查看 effective POM
mvn help:effective-pom
mvn help:effective-pom > effective-pom.xml  # 保存到文件
```

Super POM 是 Maven 内置的顶层 POM，定义了默认仓库（Maven Central）、默认目录结构、默认插件等。所有项目都隐式继承 Super POM。

---

## Part 12: Maven 常用插件大全

### 12.1 代码质量插件

#### Checkstyle

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <version>3.3.1</version>
  <configuration>
    <configLocation>google_checks.xml</configLocation>
    <!-- 或自定义：<configLocation>checkstyle.xml</configLocation> -->
    <encoding>UTF-8</encoding>
    <consoleOutput>true</consoleOutput>
    <failsOnError>true</failsOnError>
    <excludes>**/generated/**</excludes>
  </configuration>
  <executions>
    <execution>
      <id>validate</id>
      <phase>validate</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

#### SpotBugs（PMD 替代）

```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.8.3.1</version>
  <configuration>
    <effort>Max</effort>
    <threshold>Medium</threshold>
    <failOnError>true</failOnError>
    <excludeFilterFile>spotbugs-exclude.xml</excludeFilterFile>
  </configuration>
  <executions>
    <execution>
      <phase>verify</phase>
      <goals><goal>check</goal></goals>
    </execution>
  </executions>
</plugin>
```

```bash
# 运行 SpotBugs 检查
mvn spotbugs:check

# 生成报告并在浏览器中查看
mvn spotbugs:gui
```

#### PMD

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-pmd-plugin</artifactId>
  <version>3.21.2</version>
  <configuration>
    <rulesets>
      <ruleset>/rulesets/java/maven-pmd-plugin-default.xml</ruleset>
    </rulesets>
    <failOnViolation>true</failOnViolation>
    <printFailingErrors>true</printFailingErrors>
  </configuration>
  <executions>
    <execution>
      <goals>
        <goal>check</goal>
        <goal>cpd-check</goal>  <!-- 重复代码检测 -->
      </goals>
    </execution>
  </executions>
</plugin>
```

### 12.2 文档生成插件

#### maven-javadoc-plugin

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-javadoc-plugin</artifactId>
  <version>3.6.3</version>
  <configuration>
    <encoding>UTF-8</encoding>
    <charset>UTF-8</charset>
    <docencoding>UTF-8</docencoding>
    <!-- 跳过 Javadoc 错误（不推荐在严格模式下使用） -->
    <!-- <failOnError>false</failOnError> -->
    <doclint>none</doclint>  <!-- 关闭 lint 检查 -->
  </configuration>
  <executions>
    <execution>
      <id>attach-javadocs</id>
      <goals>
        <goal>jar</goal>  <!-- 生成 javadoc jar -->
      </goals>
    </execution>
  </executions>
</plugin>
```

```bash
# 生成 Javadoc
mvn javadoc:javadoc

# 生成聚合文档（多模块）
mvn javadoc:aggregate
```

### 12.3 源码打包插件

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-source-plugin</artifactId>
  <version>3.3.0</version>
  <executions>
    <execution>
      <id>attach-sources</id>
      <goals>
        <goal>jar-no-fork</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

### 12.4 Assembly 插件（自定义打包）

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-assembly-plugin</artifactId>
  <version>3.6.0</version>
  <configuration>
    <descriptorRefs>
      <!-- 打包为包含所有依赖的 fat jar -->
      <descriptorRef>jar-with-dependencies</descriptorRef>
    </descriptorRefs>
    <archive>
      <manifest>
        <mainClass>com.example.Application</mainClass>
      </manifest>
    </archive>
  </configuration>
  <executions>
    <execution>
      <id>make-assembly</id>
      <phase>package</phase>
      <goals><goal>single</goal></goals>
    </execution>
  </executions>
</plugin>
```

### 12.5 Shade 插件（uber jar）

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>3.5.1</version>
  <executions>
    <execution>
      <phase>package</phase>
      <goals><goal>shade</goal></goals>
      <configuration>
        <!-- 重定位依赖包名（避免冲突） -->
        <relocations>
          <relocation>
            <pattern>com.google.guava</pattern>
            <shadedPattern>com.example.shaded.guava</shadedPattern>
          </relocation>
        </relocations>
        <!-- 排除不需要的文件 -->
        <filters>
          <filter>
            <artifact>*:*</artifact>
            <excludes>
              <exclude>META-INF/*.SF</exclude>
              <exclude>META-INF/*.DSA</exclude>
              <exclude>META-INF/*.RSA</exclude>
            </excludes>
          </filter>
        </filters>
        <transformers>
          <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
            <mainClass>com.example.Application</mainClass>
          </transformer>
          <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
        </transformers>
      </configuration>
    </execution>
  </executions>
</plugin>
```

### 12.6 git-commit-id-maven-plugin

```xml
<!-- 将 Git 信息注入到构建中，方便追踪生产版本 -->
<plugin>
  <groupId>io.github.git-commit-id</groupId>
  <artifactId>git-commit-id-maven-plugin</artifactId>
  <version>7.0.0</version>
  <executions>
    <execution>
      <id>get-the-git-infos</id>
      <goals>
        <goal>revision</goal>
      </goals>
      <phase>initialize</phase>
    </execution>
  </executions>
  <configuration>
    <generateGitPropertiesFile>true</generateGitPropertiesFile>
    <generateGitPropertiesFilename>${project.build.outputDirectory}/git.properties</generateGitPropertiesFilename>
    <includeOnlyProperties>
      <includeOnlyProperty>git.commit.id.abbrev</includeOnlyProperty>
      <includeOnlyProperty>git.branch</includeOnlyProperty>
      <includeOnlyProperty>git.build.time</includeOnlyProperty>
    </includeOnlyProperties>
  </configuration>
</plugin>
```

---

## Part 13: Maven 实战案例

### 13.1 Spring Boot 项目完整 POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
    <relativePath/>
  </parent>

  <groupId>com.company</groupId>
  <artifactId>order-service</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <name>Order Service</name>

  <properties>
    <java.version>17</java.version>
    <spring-cloud.version>2023.0.0</spring-cloud.version>
    <mybatis-plus.version>3.5.5</mybatis-plus.version>
    <mapstruct.version>1.5.5.Final</mapstruct.version>
    <lombok.version>1.18.30</lombok.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- Web -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- 换 log4j2（排除 logback） -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <exclusions>
        <exclusion>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-log4j2</artifactId>
    </dependency>
    <!-- 数据库 -->
    <dependency>
      <groupId>com.baomidou</groupId>
      <artifactId>mybatis-plus-boot-starter</artifactId>
      <version>${mybatis-plus.version}</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <scope>runtime</scope>
    </dependency>
    <!-- 缓存 -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <!-- 工具库 -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct</artifactId>
      <version>${mapstruct.version}</version>
    </dependency>
    <!-- 测试 -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
          <excludes>
            <exclude>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
            </exclude>
          </excludes>
          <layers><enabled>true</enabled></layers>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <annotationProcessorPaths>
            <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <version>${lombok.version}</version>
            </path>
            <path>
              <groupId>org.mapstruct</groupId>
              <artifactId>mapstruct-processor</artifactId>
              <version>${mapstruct.version}</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.jacoco</groupId>
        <artifactId>jacoco-maven-plugin</artifactId>
        <executions>
          <execution><id>prepare-agent</id><goals><goal>prepare-agent</goal></goals></execution>
          <execution><id>report</id><phase>test</phase><goals><goal>report</goal></goals></execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

### 13.2 发布到 Maven Central

开源项目发布到 Maven Central 的完整流程：

```xml
<!-- 发布到 Maven Central 必需的配置 -->
<build>
  <plugins>
    <!-- GPG 签名 -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-gpg-plugin</artifactId>
      <version>3.1.0</version>
      <executions>
        <execution>
          <id>sign-artifacts</id>
          <phase>verify</phase>
          <goals><goal>sign</goal></goals>
        </execution>
      </executions>
    </plugin>
    <!-- Nexus Staging（发布到 OSSRH） -->
    <plugin>
      <groupId>org.sonatype.plugins</groupId>
      <artifactId>nexus-staging-maven-plugin</artifactId>
      <version>1.6.13</version>
      <extensions>true</extensions>
      <configuration>
        <serverId>ossrh</serverId>
        <nexusUrl>https://s01.oss.sonatype.org/</nexusUrl>
        <autoReleaseAfterClose>true</autoReleaseAfterClose>
      </configuration>
    </plugin>
    <!-- 源码和 Javadoc（Central 要求） -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-source-plugin</artifactId>
      <executions>
        <execution>
          <id>attach-sources</id>
          <goals><goal>jar-no-fork</goal></goals>
        </execution>
      </executions>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-javadoc-plugin</artifactId>
      <executions>
        <execution>
          <id>attach-javadocs</id>
          <goals><goal>jar</goal></goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>

<distributionManagement>
  <snapshotRepository>
    <id>ossrh</id>
    <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
  </snapshotRepository>
  <repository>
    <id>ossrh</id>
    <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
  </repository>
</distributionManagement>
```

```bash
# 发布命令
mvn clean deploy -P release
```

---

## Part 14: Maven 与 Spring Boot 集成最佳实践

### 14.1 Spring Boot 版本升级策略

```bash
# 查看当前 Spring Boot 依赖版本
mvn dependency:tree -Dincludes=org.springframework.boot

# 查看可用的更新版本
mvn versions:display-dependency-updates -Dincludes=org.springframework.boot

# 升级步骤：
# 1. 修改 parent 中的 spring-boot 版本
# 2. mvn clean verify 运行所有测试
# 3. 查看 migration guide 处理 breaking changes
```

### 14.2 多环境配置文件管理

```xml
<!-- application.properties 中使用 Maven 变量 -->
<!-- 资源过滤（filtering=true）后，${} 变量会被替换 -->
```
```properties
# src/main/resources/application.properties
spring.profiles.active=@spring.profiles.active@
app.version=@project.version@
app.build-time=@maven.build.timestamp@
```

```xml
<!-- pom.xml -->
<build>
  <resources>
    <resource>
      <directory>src/main/resources</directory>
      <filtering>true</filtering>
    </resource>
  </resources>
</build>

<profiles>
  <profile>
    <id>dev</id>
    <activation><activeByDefault>true</activeByDefault></activation>
    <properties>
      <spring.profiles.active>dev</spring.profiles.active>
    </properties>
  </profile>
  <profile>
    <id>prod</id>
    <properties>
      <spring.profiles.active>prod</spring.profiles.active>
    </properties>
  </profile>
</profiles>
```

注意：Spring Boot 使用 `@变量名@` 而非 `${变量名}`，是为了避免与 Spring 的 `${}` 占位符冲突。

### 14.3 测试配置

```xml
<!-- 单元测试和集成测试分离 -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <!-- 单元测试：排除集成测试 -->
    <excludes>
      <exclude>**/*IT.java</exclude>
      <exclude>**/*IntegrationTest.java</exclude>
    </excludes>
  </configuration>
</plugin>

<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-failsafe-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>integration-test</goal>
        <goal>verify</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <!-- 集成测试：使用 IT 后缀 -->
    <includes>
      <include>**/*IT.java</include>
      <include>**/*IntegrationTest.java</include>
    </includes>
  </configuration>
</plugin>
```

```bash
# 只运行单元测试
mvn test

# 运行单元测试 + 集成测试
mvn verify

# 只运行集成测试
mvn failsafe:integration-test failsafe:verify
```

---

## Part 15: Maven 常见问题与解决

### 15.1 依赖下载失败

```bash
# 问题：依赖下载失败，本地仓库有损坏的 .lastUpdated 文件

# 清理损坏的缓存文件
find ~/.m2/repository -name "*.lastUpdated" -delete  # Linux/Mac
# Windows PowerShell:
Get-ChildItem -Recurse -Filter "*.lastUpdated" ~/.m2/repository | Remove-Item

# 强制重新下载
mvn clean install -U

# 删除损坏的特定依赖
# 手动删除 ~/.m2/repository/com/example/xxx 目录，再重新构建
```

### 15.2 依赖冲突解决

```bash
# 查看完整依赖树
mvn dependency:tree -Dverbose 2>&1 | grep -A5 "jackson"

# 解决方案：在 dependencyManagement 中锁定版本
```

```xml
<dependencyManagement>
  <dependencies>
    <!-- 强制使用指定版本，优先级最高 -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.16.1</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### 15.3 构建速度慢

```bash
# 诊断慢的插件
mvn clean install -Dprofile.activation.property.detection=true

# 解决方案
# 1. 并行构建
mvn clean install -T 1C

# 2. 跳过不必要的步骤
mvn clean install -DskipTests -Dmaven.javadoc.skip=true

# 3. 使用 mvnd（Maven Daemon）
mvnd clean install

# 4. 使用离线模式（依赖已存在本地）
mvnd clean install -o

# 5. 只构建有变化的模块
mvn clean install -pl changed-module -am
```

### 15.4 OutOfMemory 错误

```bash
# 增大 Maven JVM 内存
export MAVEN_OPTS="-Xmx2g -XX:MaxMetaspaceSize=512m"

# Windows
set MAVEN_OPTS=-Xmx2g -XX:MaxMetaspaceSize=512m

# 或在 .mvn/jvm.config 文件中配置（Maven 3.3+）
# .mvn/jvm.config:
# -Xmx2g
# -XX:MaxMetaspaceSize=512m
```

### 15.5 多模块 SNAPSHOT 依赖找不到

```bash
# 问题：模块A依赖模块B，但找不到模块B的SNAPSHOT版本

# 解决：先安装模块B到本地仓库
mvn clean install -pl module-b

# 或者从根模块构建，带上 -am
mvn clean install -pl module-a -am

# 如果在 CI 中，需要先 install 再 verify
mvn install -DskipTests  # 先安装所有模块
mvn verify               # 再运行测试
```

---

## 附录 A：Maven 命令速查表

```bash
# 构建相关
mvn clean              # 清理 target 目录
mvn compile            # 编译主代码
mvn test               # 运行单元测试
mvn package            # 打包
mvn install            # 安装到本地仓库
mvn deploy             # 部署到远程仓库
mvn verify             # 验证（含集成测试）
mvn clean package -DskipTests  # 跳过测试打包

# 依赖相关
mvn dependency:tree            # 显示依赖树
mvn dependency:tree -Dverbose  # 详细依赖树
mvn dependency:analyze         # 分析依赖使用
mvn dependency:copy-dependencies  # 复制依赖

# 信息相关
mvn help:effective-pom         # 查看有效 POM
mvn help:active-profiles       # 查看激活的 profile
mvn help:describe -Dplugin=surefire  # 查看插件说明
mvn versions:display-dependency-updates  # 查看依赖更新

# 版本管理
mvn versions:set -DnewVersion=1.1.0  # 设置版本
mvn versions:commit                   # 确认版本变更
mvn versions:revert                   # 回滚版本变更

# 多模块相关
mvn install -pl module-a -am          # 构建模块及依赖
mvn install -pl module-a -amd         # 构建模块及其下游
mvn install -T 1C                     # 并行构建

# 调试
mvn clean package -X         # 调试模式
mvn clean package -e         # 显示错误堆栈
mvn clean package -q         # 静默模式
mvn clean package -o         # 离线模式
mvn clean package -U         # 强制更新 SNAPSHOT
```

## 附录 B：常用 settings.xml 模板

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
          https://maven.apache.org/xsd/settings-1.0.0.xsd">

  <localRepository>${user.home}/.m2/repository</localRepository>

  <servers>
    <server>
      <id>nexus-releases</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://nexus.company.com:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>

  <profiles>
    <profile>
      <id>nexus</id>
      <repositories>
        <repository>
          <id>nexus</id>
          <url>http://nexus.company.com:8081/repository/maven-public/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>nexus</id>
          <url>http://nexus.company.com:8081/repository/maven-public/</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>

</settings>
```

---

*Maven 详解 - 从零到精通 | 版本 2.0 | 适用于 Maven 3.6+*

## Part 16: Maven 高级特性

### 16.1 自定义插件开发

```xml
<!-- 自定义 Maven 插件的 POM -->
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.company</groupId>
  <artifactId>my-maven-plugin</artifactId>
  <version>1.0.0</version>
  <packaging>maven-plugin</packaging>

  <dependencies>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-plugin-api</artifactId>
      <version>3.9.6</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.maven.plugin-tools</groupId>
      <artifactId>maven-plugin-annotations</artifactId>
      <version>3.10.2</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>
</project>
```

```java
// 自定义 Mojo 实现
import org.apache.maven.plugin.AbstractMojo;
import org.apache.maven.plugin.MojoExecutionException;
import org.apache.maven.plugins.annotations.Mojo;
import org.apache.maven.plugins.annotations.Parameter;

@Mojo(name = "greet", defaultPhase = LifecyclePhase.PACKAGE)
public class GreetMojo extends AbstractMojo {

    @Parameter(property = "greeting", defaultValue = "Hello, Maven!")
    private String greeting;

    @Parameter(defaultValue = "${project.version}", readonly = true)
    private String projectVersion;

    @Override
    public void execute() throws MojoExecutionException {
        getLog().info(greeting + " (version: " + projectVersion + ")");
    }
}
```

```bash
# 使用自定义插件
mvn com.company:my-maven-plugin:greet
mvn com.company:my-maven-plugin:greet -Dgreeting="Hi!"
```

### 16.2 Maven Extensions

Maven Extensions 在构建过程的更早阶段加载，可以修改 Maven 核心行为。

```xml
<!-- .mvn/extensions.xml -->
<extensions xmlns="http://maven.apache.org/EXTENSIONS/1.0.0">
  <!-- Polyglot Maven：支持 YAML/Groovy/Kotlin 格式的 POM -->
  <extension>
    <groupId>io.takari.polyglot</groupId>
    <artifactId>polyglot-yaml</artifactId>
    <version>0.4.6</version>
  </extension>
</extensions>
```

### 16.3 Maven Enforcer Plugin

强制执行构建规范，如最低 JDK 版本、Maven 版本、禁止某些依赖：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-enforcer-plugin</artifactId>
  <version>3.4.1</version>
  <executions>
    <execution>
      <id>enforce-rules</id>
      <goals><goal>enforce</goal></goals>
      <configuration>
        <rules>
          <!-- 要求最低 JDK 版本 -->
          <requireJavaVersion>
            <version>[17,)</version>
            <message>JDK 17+ is required!</message>
          </requireJavaVersion>
          <!-- 要求最低 Maven 版本 -->
          <requireMavenVersion>
            <version>[3.8,)</version>
          </requireMavenVersion>
          <!-- 禁止使用某依赖 -->
          <bannedDependencies>
            <excludes>
              <exclude>log4j:log4j</exclude>  <!-- 禁用 Log4j 1.x -->
              <exclude>commons-logging:commons-logging</exclude>
            </excludes>
          </bannedDependencies>
          <!-- 禁止重复依赖 -->
          <banDuplicatePomDependencyVersions/>
          <!-- 要求唯一依赖版本 -->
          <requireUpperBoundDeps/>
        </rules>
        <fail>true</fail>
      </configuration>
    </execution>
  </executions>
</plugin>
```

### 16.4 Build Helper Plugin

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>build-helper-maven-plugin</artifactId>
  <version>3.5.0</version>
  <executions>
    <!-- 添加额外的源码目录 -->
    <execution>
      <id>add-source</id>
      <phase>generate-sources</phase>
      <goals><goal>add-source</goal></goals>
      <configuration>
        <sources>
          <source>src/main/generated</source>
          <source>target/generated-sources/jaxb</source>
        </sources>
      </configuration>
    </execution>
    <!-- 添加额外的测试源码目录 -->
    <execution>
      <id>add-test-source</id>
      <phase>generate-test-sources</phase>
      <goals><goal>add-test-source</goal></goals>
      <configuration>
        <sources>
          <source>src/integration-test/java</source>
        </sources>
      </configuration>
    </execution>
    <!-- 解析版本号组件（如 1.2.3 → major=1, minor=2, patch=3） -->
    <execution>
      <id>parse-version</id>
      <goals><goal>parse-version</goal></goals>
    </execution>
  </executions>
</plugin>
```

---

## Part 17: Maven 安全与最佳实践

### 17.1 依赖安全扫描

```xml
<!-- OWASP Dependency Check -->
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>9.0.9</version>
  <configuration>
    <failBuildOnCVSS>7</failBuildOnCVSS>   <!-- CVSS >= 7 时失败 -->
    <suppressionFile>owasp-suppressions.xml</suppressionFile>
    <nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>
  </configuration>
  <executions>
    <execution>
      <goals><goal>check</goal></goals>
    </execution>
  </executions>
</plugin>
```

```bash
# 运行安全扫描
mvn dependency-check:check

# 生成报告
mvn dependency-check:aggregate  # 多模块聚合报告
```

### 17.2 敏感信息安全

```xml
<!-- 避免在 pom.xml 中硬编码密码 -->
<!-- 使用 settings.xml 中的 server 配置 -->
<servers>
  <server>
    <id>nexus</id>
    <!-- 从环境变量读取 -->
    <username>${env.NEXUS_USER}</username>
    <password>${env.NEXUS_PASSWORD}</password>
  </server>
</servers>

<!-- 或使用 Maven Password Encryption -->
```
```bash
# Maven 密码加密
mvn --encrypt-master-password myMasterPassword
# 输出：{encrypted_master_password}
# 保存到 ~/.m2/settings-security.xml

mvn --encrypt-password myNexusPassword
# 输出：{encrypted_nexus_password}
# 保存到 settings.xml 的 <password> 中
```

### 17.3 可重现构建（Reproducible Builds）

```xml
<properties>
  <!-- 固定时间戳，使构建结果可重现 -->
  <project.build.outputTimestamp>2024-01-01T00:00:00Z</project.build.outputTimestamp>
</properties>

<plugin>
  <groupId>io.github.zlika</groupId>
  <artifactId>reproducible-build-maven-plugin</artifactId>
  <version>0.16</version>
  <executions>
    <execution>
      <goals><goal>strip-jar</goal></goals>
    </execution>
  </executions>
</plugin>
```

---

## Part 18: Maven 与 CI/CD 深度集成

### 18.1 GitHub Actions 完整流水线

```yaml
# .github/workflows/maven.yml
name: Java CI with Maven

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '17'
  MAVEN_OPTS: '-Xmx2g'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK
      uses: actions/setup-java@v4
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: temurin
        cache: maven

    - name: Build with Maven
      run: mvn -B clean verify --no-transfer-progress

    - name: Publish Test Results
      uses: EnricoMi/publish-unit-test-result-action@v2
      if: always()
      with:
        files: 'target/**/TEST-*.xml'

    - name: Upload Coverage
      uses: codecov/codecov-action@v3
      with:
        file: target/site/jacoco/jacoco.xml

  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: temurin
        cache: maven
    - name: OWASP Security Scan
      run: mvn -B dependency-check:check --no-transfer-progress
      continue-on-error: true
    - name: Upload OWASP Report
      uses: actions/upload-artifact@v3
      with:
        name: owasp-report
        path: target/dependency-check-report.html

  release:
    needs: [build, security]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: temurin
        server-id: nexus-releases
        server-username: NEXUS_USER
        server-password: NEXUS_PASSWORD
    - name: Deploy to Nexus
      run: mvn -B deploy -DskipTests --no-transfer-progress
      env:
        NEXUS_USER: ${{ secrets.NEXUS_USER }}
        NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
```

### 18.2 GitLab CI 流水线

```yaml
# .gitlab-ci.yml
image: maven:3.9.6-eclipse-temurin-17

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -Xmx2g"
  MAVEN_CLI_OPTS: "--batch-mode --no-transfer-progress"

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .m2/repository/

stages: [build, test, quality, deploy]

build:
  stage: build
  script: mvn $MAVEN_CLI_OPTS compile -DskipTests

unit-test:
  stage: test
  script: mvn $MAVEN_CLI_OPTS test
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
      coverage_report:
        coverage_format: jacoco
        path: target/site/jacoco/jacoco.xml

integration-test:
  stage: test
  script: mvn $MAVEN_CLI_OPTS verify -P integration-test
  only: [main, develop]

code-quality:
  stage: quality
  script:
    - mvn $MAVEN_CLI_OPTS spotbugs:check
    - mvn $MAVEN_CLI_OPTS pmd:check
  allow_failure: true

deploy-nexus:
  stage: deploy
  script:
    - mvn $MAVEN_CLI_OPTS deploy -DskipTests
  environment:
    name: nexus
  only: [main]
```

---

*Maven 详解 - 完整企业版文档 | 版本 3.0*

## Part 19: Maven 生态系统

### 19.1 Maven Central 使用技巧

```bash
# 在 Maven Central 搜索依赖
# 网址：https://central.sonatype.com
# 旧版：https://search.maven.org

# 使用 mvn 命令搜索
# （需要安装 mvn-search 插件或使用在线搜索）

# 查看某 groupId 下的所有 artifactId
# https://central.sonatype.com/namespace/org.springframework.boot

# 查看某依赖的所有版本
mvn versions:display-dependency-updates -Dincludes=org.springframework.boot:spring-boot-starter-web
```

### 19.2 常用 BOM 汇总

| BOM | 说明 | groupId:artifactId |
|-----|------|--------------------|
| Spring Boot BOM | Spring Boot 全家桶 | org.springframework.boot:spring-boot-dependencies |
| Spring Cloud BOM | Spring Cloud 组件 | org.springframework.cloud:spring-cloud-dependencies |
| Spring Cloud Alibaba BOM | 阿里云组件 | com.alibaba.cloud:spring-cloud-alibaba-dependencies |
| AWS SDK BOM | AWS Java SDK | software.amazon.awssdk:bom |
| Testcontainers BOM | 测试容器 | org.testcontainers:testcontainers-bom |
| Micrometer BOM | 监控指标 | io.micrometer:micrometer-bom |

### 19.3 Maven 与其他构建工具互操作

#### Maven 与 Gradle 项目共存

```bash
# 将 Gradle 项目转为 Maven
# 在 Gradle 项目中安装 maven-publish 插件，生成 pom.xml

# 将 Maven 项目迁移到 Gradle
gradle init --type pom  # 从现有 pom.xml 生成 build.gradle
```

### 19.4 常用第三方插件

| 插件 | groupId | 说明 |
|------|---------|------|
| versions-maven-plugin | org.codehaus.mojo | 管理版本号 |
| exec-maven-plugin | org.codehaus.mojo | 执行 Java 程序和系统命令 |
| build-helper-maven-plugin | org.codehaus.mojo | 额外源码目录、版本解析 |
| flatten-maven-plugin | org.codehaus.mojo | 展平 POM（发布时去除 parent 引用） |
| git-commit-id-maven-plugin | io.github.git-commit-id | 注入 Git 信息 |
| dockerfile-maven-plugin | com.spotify | 构建 Docker 镜像 |
| jib-maven-plugin | com.google.cloud.tools | 无 Docker 守护进程构建镜像 |
| sonar-maven-plugin | org.sonarsource.scanner.maven | SonarQube 代码分析 |

### 19.5 Jib：无 Dockerfile 构建 Docker 镜像

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.4.0</version>
  <configuration>
    <from>
      <image>eclipse-temurin:17-jre</image>
    </from>
    <to>
      <image>registry.company.com/order-service</image>
      <tags>
        <tag>${project.version}</tag>
        <tag>latest</tag>
      </tags>
    </to>
    <container>
      <ports>
        <port>8080</port>
      </ports>
      <jvmFlags>
        <jvmFlag>-Xmx512m</jvmFlag>
        <jvmFlag>-XX:+UseContainerSupport</jvmFlag>
      </jvmFlags>
      <environment>
        <SPRING_PROFILES_ACTIVE>prod</SPRING_PROFILES_ACTIVE>
      </environment>
    </container>
  </configuration>
</plugin>
```

```bash
# 构建并推送镜像（不需要本地安装 Docker）
mvn jib:build

# 构建到本地 Docker daemon
mvn jib:dockerBuild

# 构建为 tar 文件
mvn jib:buildTar
```

---

## Part 20: Maven 最佳实践总结

### 20.1 POM 设计原则

1. **单一职责**：每个模块只做一件事（API 定义、业务逻辑、Web 层分离）
2. **版本集中管理**：所有版本号在父 POM 的 `<properties>` 或 `<dependencyManagement>` 中集中声明
3. **避免版本散落**：子模块依赖不应指定版本号（由父 POM 管理）
4. **最小依赖原则**：只引入真正需要的依赖，避免引入过多无用依赖
5. **依赖 scope 精准**：compile/test/provided/runtime 要用对，减少最终包体积

### 20.2 多模块项目设计

```
推荐的模块划分：

platform-parent          （父模块：版本管理、公共配置）
  ├── platform-api       （接口定义：实体、DTO、接口声明）
  ├── platform-common    （公共工具：工具类、常量、异常定义）
  ├── platform-service   （业务逻辑：Service 实现、Repository）
  └── platform-web       （Web 层：Controller、启动类）

依赖关系：
  web → service → api
  所有模块 → common

避免循环依赖！
```

### 20.3 持续集成建议

```bash
# CI 环境的标准构建命令
mvn --batch-mode          # 非交互模式
    --no-transfer-progress # 不显示传输进度（减少日志）
    clean verify           # 清理 + 全量验证
    -Dmaven.test.failure.ignore=false  # 测试失败则终止

# 完整命令
mvn --batch-mode --no-transfer-progress clean verify
```

### 20.4 团队规范建议

| 规范 | 建议 |
|------|------|
| POM 格式 | 统一使用 IDE 格式化，保持 POM 可读性 |
| 版本命名 | 开发阶段 SNAPSHOT，发布 RELEASE，遵循语义化 |
| 依赖审查 | PR 时 Review 新增依赖的必要性和安全性 |
| 定期更新 | 每月检查并更新一次依赖版本（安全补丁优先） |
| 测试覆盖 | 核心模块代码覆盖率 >= 80%，用 JaCoCo 门控 |
| 本地仓库 | CI 中使用缓存加速，减少重复下载 |
| 私服管理 | 所有依赖经过私服，禁止 CI 直连 Maven Central |

---

## 附录 C：Maven 版本历史与新特性

| 版本 | 重要特性 |
|------|----------|
| Maven 3.3 | .mvn 目录、JVM 配置文件、Core Extensions |
| Maven 3.5 | 彩色输出、多线程改进 |
| Maven 3.6 | 更好的错误信息、性能改进 |
| Maven 3.8 | 默认 HTTPS、Wrapper 支持 |
| Maven 3.9 | 性能提升、改进的 CI 友好版本 |
| Maven 4.0 | 新 POM 模型（model version 4.1.0）、Build/Consumer POM 分离 |

---

*Maven 详解 - 从零到精通完整版 | 最后更新：2024年 | 适用于 Apache Maven 3.6+*
