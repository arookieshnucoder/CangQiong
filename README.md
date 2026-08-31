# 苍穹外卖

苍穹外卖是一个基于 Spring Boot、MyBatis、MySQL、Redis 和 Vue 的外卖管理系统。本仓库已经包含 Java 后端、Liquibase 数据库迁移、构建后的网页管理端和 Windows 版 Nginx。

本文件主要说明 Windows 本地环境下如何启动网页管理端和 Java 后端。微信小程序不在本次启动范围内。

主要更新了数据库的自动Migration，不需要额外去运行sql文件就可以自动建表，并且建立admin管理员账户，可以理解为一个简易启动版本。

## 项目结构

```text
sky-take-out/
├── sky-common/                  # 后端公共模块
├── sky-pojo/                    # DTO、Entity、VO
├── sky-server/                  # Spring Boot 后端
├── project-rjwm-admin-vue-ts/   # Vue 管理端源码
├── nginx-1.20.2/                # Nginx 和已构建的管理端页面
│   └── html/sky/                # 管理端静态文件
├── mp-weixin/                   # 微信小程序，本指南暂不使用
└── demo/mysql.sql               # 原始 SQL 参考文件，不需要手动执行
```

## 请求链路

```text
浏览器 http://localhost
          │
          ▼
Nginx（80 端口）
  ├── /        → nginx-1.20.2/html/sky 静态页面
  ├── /api/*   → http://localhost:8080/admin/*
  └── /ws/*    → http://localhost:8080/ws/*
                         │
                         ▼
                Java 后端（8080 端口）
                         │
                         ▼
                    MySQL / Redis
```

浏览器只需要访问 Nginx 的 80 端口。管理端发送的 `/api` 请求由 Nginx 自动转发到 Java 后端，不需要在浏览器中直接填写 8080。

## 环境要求

- Windows 10/11
- JDK 21
- Maven 3.9+
- MySQL 8.0+
- Redis：只验证启动和数据库迁移时可暂不运行；完整使用管理端时需要

在 PowerShell 中检查：

```powershell
java -version
mvn -version
mysql --version
```

`mvn -version` 中显示的 Java version 应为 21。

## 首次启动

### 1. 启动 MySQL

确认 MySQL 服务已经启动，并测试本地账号：

```powershell
mysql -u root -p
```

命令会在终端内询问密码。登录成功后执行 `exit;` 退出。

不需要手动创建 `sky_take_out` 数据库，也不需要执行 `demo/mysql.sql`。后端连接 MySQL 后，Liquibase 会自动创建数据库、数据表和本地管理员。MySQL 账号必须具有创建数据库和数据表的权限。

### 2. 设置数据库连接

打开 PowerShell，在当前终端会话中设置本机 MySQL 账号：

```powershell
$env:DB_USERNAME = "root"
$env:DB_PASSWORD = "填写本机 MySQL 密码"
```

默认连接地址是：

```text
jdbc:mysql://localhost:3306/sky_take_out?createDatabaseIfNotExist=true
```

如果 MySQL 不在本机或端口不是 3306，可以覆盖完整地址：

```powershell
$env:DB_URL = "jdbc:mysql://localhost:3306/sky_take_out?createDatabaseIfNotExist=true"
```

这些环境变量只在当前 PowerShell 窗口中有效。请在同一个窗口中启动后端，不要把真实数据库密码提交到 Git。

### 3. 构建 Java 后端

首次启动时，在项目根目录执行：

```powershell
mvn -pl sky-server -am install -DskipTests
```

该命令会按顺序构建 `sky-common`、`sky-pojo` 和 `sky-server`，并将它们安装到本机 Maven 仓库。

### 4. 启动 Java 后端

开发模式推荐使用：

```powershell
mvn -f sky-server/pom.xml spring-boot:run
```

后端监听：

```text
http://localhost:8080
```

看到以下日志说明后端启动完成：

```text
Tomcat started on port(s): 8080
server started
```

保持该 PowerShell 窗口运行。使用 `Ctrl+C` 停止后端。

### 5. 启动 Nginx 和网页管理端

仓库已经包含构建后的网页管理端，不需要安装 Node.js、Vue 或 npm 依赖即可查看和使用当前页面。

首次运行先确保 Nginx 临时目录存在：

```powershell
New-Item -ItemType Directory -Force nginx-1.20.2/temp | Out-Null
```

然后启动 Nginx：

```powershell
Set-Location nginx-1.20.2
.\nginx.exe
Set-Location ..
```

也可以在资源管理器中双击 `nginx-1.20.2/nginx.exe`。Nginx 在后台运行，不会自动打开浏览器。

手动访问：

```text
http://localhost
```

停止 Nginx：

```powershell
Set-Location nginx-1.20.2
.\nginx.exe -s stop
Set-Location ..
```

## 数据库 Migration

项目使用 Liquibase 管理数据库结构，入口文件是：

```text
sky-server/src/main/resources/db/changelog/db.changelog-master.xml
```

当前迁移包括：

```text
001-create-initial-schema.xml   # 创建 11 张业务表
002-insert-default-admin.xml    # 创建本地管理员
```

后端每次启动时都会检查迁移状态。Liquibase 只执行尚未执行的 changeSet，不会每次删除或重建表。执行记录保存在：

- `DATABASECHANGELOG`
- `DATABASECHANGELOGLOCK`

默认本地管理员：

```text
用户名：admin
密码：123456
```

该账号仅用于本地开发，首次登录后应修改默认密码。正式环境不要使用默认密码或 MD5 密码方案。

以后变更数据库结构时，需要新增 `003-xxx.xml`、`004-xxx.xml` 等迁移文件，并在主 changelog 中引入。不要修改已经执行过的迁移文件。

## Redis

管理员账号和登录数据保存在 MySQL，不依赖 Redis。没有 Redis 时后端仍可能启动和登录，但店铺状态、菜品缓存等部分接口无法正常使用。

完整使用网页管理端前，请启动 Redis，并确保连接与以下默认配置一致：

```text
host: 127.0.0.1
port: 6379
password: 123456
```

Redis 没有密码时，需要同步修改 `sky-server/src/main/resources/application.yml` 中的 Redis 密码配置。

## 后续开发与重新构建

只修改 `sky-server` 中的 Java 代码时，可以停止后端并重新运行：

```powershell
mvn -f sky-server/pom.xml spring-boot:run
```

修改了 `sky-common` 或 `sky-pojo` 后，先重新安装模块：

```powershell
mvn -pl sky-server -am install -DskipTests
mvn -f sky-server/pom.xml spring-boot:run
```

也可以打包为可执行 jar：

```powershell
mvn -pl sky-server -am clean package -DskipTests
java -jar sky-server/target/sky-server-1.0-SNAPSHOT.jar
```

重新执行 `package` 会覆盖上一次生成的 jar。修改代码或配置后，运行 jar 的方式必须重新打包并重启。

Vue 管理端源码位于 `project-rjwm-admin-vue-ts`。只有修改前端源码并重新构建页面时，才需要处理 Node.js 和 Vue 依赖；使用仓库现有的 Nginx 静态页面不需要构建前端。

## 接口文档

后端启动后可访问：

- Knife4j：<http://localhost:8080/doc.html>
- Swagger UI：<http://localhost:8080/swagger-ui.html>
- OpenAPI JSON：<http://localhost:8080/v3/api-docs>

## 常见问题

### MySQL 拒绝登录

```text
Access denied for user 'root'@'localhost'
```

使用 `mysql -u root -p` 验证账号密码，并确认 `DB_USERNAME`、`DB_PASSWORD` 已在启动后端的同一个 PowerShell 中设置。

### 无权自动创建数据库

如果 MySQL 用户没有建库权限，可以先手动创建空数据库：

```sql
CREATE DATABASE sky_take_out;
```

不要手动创建业务表，表结构继续交给 Liquibase 管理。

### 网页能打开但登录或请求失败

依次检查：

```powershell
Test-NetConnection localhost -Port 80
Test-NetConnection localhost -Port 8080
Test-NetConnection localhost -Port 3306
Test-NetConnection localhost -Port 6379
```

- 80：Nginx 和网页管理端
- 8080：Java 后端
- 3306：MySQL
- 6379：Redis

网页能打开只说明 Nginx 正常。如果 8080 未监听，所有 `/api` 请求都会失败。

### Nginx 双击后没有反应

Nginx 默认在后台运行，不会显示窗口或自动打开浏览器。先执行配置检查：

```powershell
New-Item -ItemType Directory -Force nginx-1.20.2/temp | Out-Null
Set-Location nginx-1.20.2
.\nginx.exe -t
Set-Location ..
```

配置通过后启动，再手动访问 <http://localhost>。

更详细的后端说明见 [sky-server/README.md](./sky-server/README.md)。
