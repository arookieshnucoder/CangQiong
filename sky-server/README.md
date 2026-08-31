# Sky Server 本地启动指南

本文只说明 Java 后端的本地启动方式。前端、Nginx 和微信小程序均不是启动后端的必要条件。

## 1. 环境要求

- JDK 21
- Maven 3.9+
- MySQL 8.0+
- Redis（仅验证数据库迁移时可以暂不启动；完整使用所有接口时需要）

在 PowerShell 中确认环境：

```powershell
java -version
mvn -version
mysql --version
```

`mvn -version` 输出中的 Java version 应为 21。

## 2. 准备 MySQL

先确认 MySQL 服务已启动，并且本地账号能够登录：

```powershell
mysql -u root -p
```

该命令会直接在终端中安全地询问密码。登录成功后可以退出：

```sql
exit;
```

后端使用 Liquibase 自动创建 `sky_take_out` 数据库和业务表，不需要手动执行 `demo/mysql.sql`。MySQL 用户需要具有创建数据库和数据表的权限。

## 3. 配置数据库连接

打开一个新的 PowerShell，在当前终端会话中设置连接信息：

```powershell
$env:DB_USERNAME = "root"
$env:DB_PASSWORD = "填写本机 MySQL 密码"
```

默认连接地址是：

```text
jdbc:mysql://localhost:3306/sky_take_out?createDatabaseIfNotExist=true
```

MySQL 不在本机或端口不是 3306 时，再设置完整地址：

```powershell
$env:DB_URL = "jdbc:mysql://localhost:3306/sky_take_out?createDatabaseIfNotExist=true"
```

这些环境变量只对当前 PowerShell 窗口生效。关闭终端后不会保存，也不会把密码写入 Git。

## 4. 启动后端

### 方式一：开发模式

项目由 `sky-common`、`sky-pojo` 和 `sky-server` 三个模块组成。首次启动先在项目根目录安装所有模块：

```powershell
Set-Location D:\MS\sky-take-out
mvn -pl sky-server -am install -DskipTests
```

构建成功后启动服务：

```powershell
mvn -f sky-server/pom.xml spring-boot:run
```

以后只修改 `sky-server` 时，直接执行第二条命令即可。修改了 `sky-common` 或 `sky-pojo` 后，需要重新执行第一条命令。

按 `Ctrl+C` 停止后端。

### 方式二：打包后运行

在项目根目录执行：

```powershell
mvn -pl sky-server -am clean package -DskipTests
java -jar sky-server/target/sky-server-1.0-SNAPSHOT.jar
```

这种方式最接近部署环境。每次修改 Java 代码后都需要重新打包。

## 5. Liquibase 自动建表

后端启动时会自动读取：

```text
sky-server/src/main/resources/db/changelog/db.changelog-master.xml
```

首次连接空数据库时，Liquibase 会创建：

- 11 张业务表
- `DATABASECHANGELOG`：记录已经执行的迁移
- `DATABASECHANGELOGLOCK`：防止多个实例同时迁移

启动日志出现类似以下内容表示迁移已执行：

```text
Running Changeset
Liquibase command 'update' was executed successfully
```

可以登录 MySQL 验证：

```sql
USE sky_take_out;
SHOW TABLES;
SELECT ID, AUTHOR, FILENAME, DATEEXECUTED
FROM DATABASECHANGELOG
ORDER BY ORDEREXECUTED;
```

以后修改表结构时，应新增 `002-xxx.xml`、`003-xxx.xml`，并在主 changelog 中引入。不要修改已经在数据库执行过的 `001-create-initial-schema.xml`。

## 6. 验证服务和接口

启动成功后，控制台会出现：

```text
Tomcat started on port(s): 8080
server started
```

可以访问：

- Knife4j：<http://localhost:8080/doc.html>
- Swagger UI：<http://localhost:8080/swagger-ui.html>
- OpenAPI JSON：<http://localhost:8080/v3/api-docs>

也可以在 PowerShell 中检查：

```powershell
Invoke-WebRequest http://localhost:8080/v3/api-docs -UseBasicParsing
```

Liquibase 会在不存在 `admin` 用户时创建一个本地开发管理员：

```text
用户名：admin
密码：123456
```

该账号仅用于本地开发，首次登录后应修改默认密码。其他业务数据不会自动插入。

## 7. Redis

只验证 Spring Boot、MySQL 和 Liquibase 时，可以暂时不启动 Redis。但是店铺状态、缓存等接口会连接 Redis；要验证所有接口，需先启动 Redis，并让连接信息与 `application.yml` 一致：

```yaml
spring:
  data:
    redis:
      host: 127.0.0.1
      port: 6379
      password: 123456
```

若本地 Redis 没有密码，应删除或覆盖该密码配置。

## 8. 常见错误

### MySQL 拒绝登录

```text
Access denied for user 'root'@'localhost'
```

原因是 `DB_USERNAME` 或 `DB_PASSWORD` 不正确。先用 `mysql -u root -p` 验证账号，再在同一个 PowerShell 中重新设置环境变量。

### 无权自动创建数据库

```text
Unknown database 'sky_take_out'
```

给当前 MySQL 用户授予建库权限，或者先手动创建空数据库：

```sql
CREATE DATABASE sky_take_out;
```

不要手动创建业务表，后续表结构由 Liquibase 管理。

### Redis 连接失败

```text
Unable to connect to Redis
Connection refused: localhost/127.0.0.1:6379
```

启动 Redis，或暂时不要调用依赖 Redis 的接口。

### 8080 端口被占用

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen
```

停止占用端口的程序，或者临时使用其他端口：

```powershell
mvn -f sky-server/pom.xml spring-boot:run "-Dspring-boot.run.arguments=--server.port=8081"
```

### Lombok 或 javac 编译错误

确认 Maven 使用 JDK 21：

```powershell
mvn -version
```

如果不是 21，检查 `JAVA_HOME` 后重新打开 PowerShell。