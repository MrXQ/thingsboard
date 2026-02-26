# ThingsBoard 开发环境搭建手册

> 适用版本：ThingsBoard 4.3.x（分支 `release-4.3-sae-main`）
>
> 操作系统：Windows 11
>
> 最后更新：2026-02-26

---

## 目录

1. [项目概述](#1-项目概述)
2. [前置条件](#2-前置条件)
3. [问题一：Gradle 下载失败导致 packaging 模块构建失败](#3-问题一gradle-下载失败导致-packaging-模块构建失败)
4. [问题二：Yarn 继承 Maven 代理参数导致 ui-ngx 构建失败](#4-问题二yarn-继承-maven-代理参数导致-ui-ngx-构建失败)
5. [问题三：IDE 环境选择——从 VSCode 迁移到 IntelliJ IDEA](#5-问题三ide-环境选择从-vscode-迁移到-intellij-idea)
6. [完整构建验证](#6-完整构建验证)
7. [常用命令速查](#7-常用命令速查)

---

## 1. 项目概述

ThingsBoard 是一个开源物联网平台，采用 **Maven + Gradle + Node.js** 混合构建体系：

- **Maven**：主构建工具，管理 Java 模块的编译、打包和依赖
- **Gradle**：由 `gradle-maven-plugin`（版本 1.0.12）在 Maven 打包阶段调用，负责生成 Linux RPM/DEB 安装包（`packaging/java/` 和 `packaging/js/`）
- **Node.js + Yarn**：由 `frontend-maven-plugin`（版本 1.12.0）管理，负责前端 Angular UI（`ui-ngx/`）和 JS Executor（`msa/js-executor/`）的构建

### 关键版本

| 组件 | 版本 |
|------|------|
| Java | 17+ |
| Maven | 3.x |
| Gradle（自动下载） | 7.3.3 |
| Node.js（自动下载） | v22.18.0 |
| Yarn（自动下载） | v1.22.22 |
| Angular | 18.2.13 |

---

## 2. 前置条件

### 2.1 网络代理

本项目在公司内网开发，需要通过代理（`localhost:10808`）访问外部资源。涉及三处代理配置：

- **Maven 代理**：`~/.m2/settings.xml`
- **Gradle 代理**：`~/.gradle/gradle.properties`
- **系统代理**：v2ray / clash 等工具监听 `localhost:10808`

### 2.2 必备软件

- JDK 17 或更高版本
- Maven 3.x
- IntelliJ IDEA（推荐 Ultimate 版）
- Git
- 代理工具（确保 `localhost:10808` 可用）

> **注意**：Node.js 和 Yarn 不需要手动安装，`frontend-maven-plugin` 会自动下载到各模块的 `target/` 目录中。

---

## 3. 问题一：Gradle 下载失败导致 packaging 模块构建失败

### 3.1 问题现象

执行 `mvn package` 时，到达 packaging 阶段（如 `application/src/main/...` 打 DEB/RPM 包），`gradle-maven-plugin` 会尝试下载 Gradle 发行版。在公司代理环境下，自动下载会失败，报错类似：

```
Exception in thread "main" java.net.ConnectException:
  Connection timed out: connect
```

或 Gradle Wrapper 下载超时/卡住。

### 3.2 根因分析

`gradle-maven-plugin`（org.thingsboard:gradle-maven-plugin:1.0.12）内部使用 Gradle Wrapper 机制。Wrapper 会从 `https://services.gradle.org/distributions/` 下载 Gradle 发行版 zip 文件。在代理环境下，如果 Gradle 没有正确配置代理属性，下载会失败。

### 3.3 解决方案

#### 步骤 1：手动下载 Gradle 发行版

从以下地址手动下载 Gradle 7.3.3 的 bin 发行版 zip 文件（可通过浏览器使用代理下载）：

```
https://services.gradle.org/distributions/gradle-7.3.3-bin.zip
```

#### 步骤 2：放置到 Gradle Wrapper 缓存目录

将下载好的 zip 文件放置到 Gradle Wrapper 的缓存目录中：

```
C:\Users\<你的用户名>\.gradle\wrapper\dists\gradle-7.3.3-bin\<hash>\
```

其中 `<hash>` 是一串随机字符（如 `6a41zxkdtcxs8rphpq6y0069z`）。如果该目录不存在，先运行一次构建让 Wrapper 创建目录结构，然后把 zip 文件放入。

最终目录结构应类似：

```
C:\Users\<用户名>\.gradle\wrapper\dists\gradle-7.3.3-bin\6a41zxkdtcxs8rphpq6y0069z\
├── gradle-7.3.3\          ← 解压后的 Gradle 目录
├── gradle-7.3.3-bin.zip   ← 下载的 zip 文件
├── gradle-7.3.3-bin.zip.lck
└── gradle-7.3.3-bin.zip.ok  ← 此文件标志下载完成
```

> **关键**：确保 `.zip.ok` 文件存在，Wrapper 据此判断 zip 已下载完成。如果不存在，可手动创建一个空的 `.ok` 文件。

#### 步骤 3：配置 Gradle 全局代理

在 `C:\Users\<你的用户名>\.gradle\gradle.properties` 文件中添加代理配置（如文件不存在则新建）：

```properties
systemProp.http.proxyHost=localhost
systemProp.http.proxyPort=10808
systemProp.https.proxyHost=localhost
systemProp.https.proxyPort=10808
```

此配置确保 Gradle 在后续运行中（如下载插件依赖 `nebula.ospackage:8.6.3` 等）能正确使用代理。

#### 步骤 4：验证

运行以下命令验证 packaging 模块是否能正常构建：

```bash
cd packaging/java
../../gradlew build --info
```

如果输出中没有下载超时错误，说明 Gradle 配置成功。

---

## 4. 问题二：Yarn 继承 Maven 代理参数导致 ui-ngx 构建失败

### 4.1 问题现象

执行 `mvn package` 构建 `ui-ngx` 模块时，Yarn 命令以退出码 1（exit code 1）失败。错误日志中可能出现类似以下内容：

```
error An unexpected error occurred: "https://registry.yarnpkg.com/...:
  self-signed certificate in certificate chain"
```

或者 Yarn 命令行参数中出现了不应该存在的代理参数。

### 4.2 根因分析

`frontend-maven-plugin` 在调用 Yarn 时，默认会将 Maven `settings.xml` 中配置的代理信息（`localhost:10808`）作为命令行参数传递给 Yarn。然而 Yarn 并不能正确处理这种方式传入的代理参数，导致命令执行失败。

具体流程：

1. Maven 读取 `~/.m2/settings.xml` 中的 `<proxies>` 配置
2. `frontend-maven-plugin` 检测到 Maven 代理设置
3. 插件将代理以参数形式附加到 Yarn 命令中
4. Yarn 不能正确解析这些参数 → **exit code 1**

### 4.3 解决方案

在 `ui-ngx/pom.xml` 的 `frontend-maven-plugin` 配置中，添加一行禁止 Yarn 继承 Maven 代理配置：

```xml
<plugin>
    <groupId>com.github.eirslett</groupId>
    <artifactId>frontend-maven-plugin</artifactId>
    <configuration>
        <installDirectory>target</installDirectory>
        <workingDirectory>${basedir}</workingDirectory>
        <!-- 关键修复：阻止 Maven 代理参数传递给 Yarn -->
        <yarnInheritsProxyConfigFromMaven>false</yarnInheritsProxyConfigFromMaven>
    </configuration>
    ...
</plugin>
```

**文件位置**：`ui-ngx/pom.xml` 第 51 行

此修改已提交至仓库：

```
commit c017164 - Fix npm initiated yarn command included proxy param
                 result in exit code 1 and failing ui-ngx build
```

### 4.4 补充说明

- Yarn 自身的网络配置在 `ui-ngx/.yarnrc` 中，目前只设置了 `network-timeout 1000000`
- 如果 Yarn 自身需要代理，应通过环境变量 `HTTP_PROXY` / `HTTPS_PROXY` 或 `.yarnrc` 配置，而不是由 Maven 传递
- `msa/js-executor/pom.xml` 中也使用了 `frontend-maven-plugin`，如遇到相同问题，需做相同修改

---

## 5. 问题三：IDE 环境选择——从 VSCode 迁移到 IntelliJ IDEA

### 5.1 问题现象

使用 VSCode 进行 ThingsBoard 开发时遇到多种问题：

- Java 语言服务（Language Server）对大型 Maven 多模块项目支持不够稳定
- Maven 构建集成不如原生 IDE 顺畅
- 调试、重构等高级功能体验较差
- 代码索引速度慢，频繁出现卡顿

### 5.2 解决方案：使用 IntelliJ IDEA

ThingsBoard 作为大型 Java + Maven 多模块项目，IntelliJ IDEA 是更合适的 IDE 选择。

#### 步骤 1：克隆仓库

```bash
git clone -b release-4.3-sae-main https://github.com/MrXQ/thingsboard.git --depth 1
```

> `--depth 1` 表示浅克隆，仅拉取最新一次提交，可大幅减少下载体积和时间。如需完整历史记录，去掉该参数即可。

#### 步骤 2：首次命令行构建

在用 IDEA 打开项目**之前**，先在命令行完成一次完整构建，确保所有依赖下载完毕且构建正常：

```bash
cd thingsboard
mvn clean install -DskipTests
```

> **为什么先命令行构建？**
> - 确保 Maven 依赖、Node.js、Yarn、Gradle 发行版等全部下载到位
> - 如遇到问题一（Gradle）或问题二（Yarn 代理）中描述的错误，可先按前文方案修复
> - IDEA 导入时会直接复用已下载的依赖，索引速度更快、更稳定

#### 步骤 3：用 IDEA 导入项目

1. 打开 IntelliJ IDEA
2. 选择 **File → Open**
3. 选择项目根目录（如 `D:\M088516\Projects\thingsboard`）
4. IDEA 会自动检测到 `pom.xml` 并提示作为 Maven 项目导入
5. 等待索引完成（首次可能需要较长时间）

#### 步骤 4：配置 JDK

1. **File → Project Structure → Project**
2. 设置 Project SDK 为 JDK 17 或更高版本
3. Project language level 设为 17

#### 步骤 5：配置 Maven（可选）

1. **File → Settings → Build, Execution, Deployment → Build Tools → Maven**
2. 确认 Maven home path 指向本地 Maven 安装路径
3. User settings file 指向 `C:\Users\<用户名>\.m2\settings.xml`（含代理配置）

#### 步骤 6：配置代理（可选）

如果 IDEA 自身需要访问外部资源（如插件市场）：

1. **File → Settings → Appearance & Behavior → System Settings → HTTP Proxy**
2. 选择 **Manual proxy configuration**
3. 填入 `localhost` / `10808`

### 5.3 IDEA 项目结构

导入后，IDEA 会识别以下关键模块：

```
thingsboard (root)
├── application          ← 主应用入口
├── common/              ← 公共模块
├── dao/                 ← 数据访问层
├── msa/                 ← 微服务架构
│   ├── js-executor      ← JavaScript 执行器
│   ├── tb-node          ← TB 节点
│   ├── transport/       ← 传输层
│   └── web-ui           ← Web UI 服务
├── rule-engine/         ← 规则引擎
├── transport/           ← 传输协议
├── ui-ngx/              ← Angular 前端（可独立开发）
└── packaging/           ← Linux 打包（java/ 和 js/）
```

---

## 6. 完整构建验证

### 6.1 环境检查清单

在开始构建前，确认以下配置全部就位：

| 检查项 | 配置文件 | 关键内容 |
|--------|----------|----------|
| Maven 代理 | `~/.m2/settings.xml` | `<proxy>` 指向 `localhost:10808` |
| Gradle 代理 | `~/.gradle/gradle.properties` | `systemProp.http(s).proxyHost/Port` |
| Gradle 发行版 | `~/.gradle/wrapper/dists/gradle-7.3.3-bin/` | zip 已下载，`.ok` 文件存在 |
| Yarn 代理隔离 | `ui-ngx/pom.xml` | `<yarnInheritsProxyConfigFromMaven>false</yarnInheritsProxyConfigFromMaven>` |
| JDK | 系统环境变量 | JDK 17+ 已安装且 `JAVA_HOME` 已设置 |

### 6.2 构建命令

```bash
# 完整构建（跳过测试，加快速度）
mvn clean install -DskipTests

# 仅构建前端
cd ui-ngx
mvn clean install -DskipTests

# 仅构建后端（跳过前端和打包）
mvn clean install -DskipTests -pl !ui-ngx -pl !packaging
```

### 6.3 常见构建失败排查

| 错误现象 | 可能原因 | 解决方法 |
|----------|----------|----------|
| Gradle 下载超时 | 代理未配置或 zip 未手动放置 | 参见 [问题一](#3-问题一gradle-下载失败导致-packaging-模块构建失败) |
| Yarn exit code 1 | Maven 代理参数被传递给 Yarn | 参见 [问题二](#4-问题二yarn-继承-maven-代理参数导致-ui-ngx-构建失败) |
| `nebula.ospackage` 插件下载失败 | Gradle 代理未配置 | 检查 `~/.gradle/gradle.properties` |
| `node` / `yarn` 命令未找到 | `frontend-maven-plugin` 未正确安装 | 删除 `ui-ngx/target/` 后重试 |
| OutOfMemoryError | 前端构建内存不足 | 设置 `MAVEN_OPTS=-Xmx4g` |

---

## 7. 常用命令速查

```bash
# 检查 Maven 代理配置
cat ~/.m2/settings.xml

# 检查 Gradle 代理配置
cat ~/.gradle/gradle.properties

# 检查 Gradle 缓存的发行版
ls ~/.gradle/wrapper/dists/

# 查看 Node.js / Yarn 版本（由 frontend-maven-plugin 安装）
ui-ngx/target/node/node --version
ui-ngx/target/node/yarn/dist/bin/yarn --version

# 清理并重新构建
mvn clean install -DskipTests

# 仅启动前端开发服务器
cd ui-ngx && mvn compile -P yarn-start
```

---

## 附录：配置文件模板

### A. Maven 代理配置（`~/.m2/settings.xml`）

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <proxies>
    <proxy>
      <id>http-proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>localhost</host>
      <port>10808</port>
    </proxy>
    <proxy>
      <id>https-proxy</id>
      <active>true</active>
      <protocol>https</protocol>
      <host>localhost</host>
      <port>10808</port>
    </proxy>
  </proxies>
</settings>
```

### B. Gradle 代理配置（`~/.gradle/gradle.properties`）

```properties
systemProp.http.proxyHost=localhost
systemProp.http.proxyPort=10808
systemProp.https.proxyHost=localhost
systemProp.https.proxyPort=10808
```
