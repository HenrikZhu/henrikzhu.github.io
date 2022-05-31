---
title: "Davinci"
date: 2020-01-05T08:24:45+08:00
draft: true
---


## 打包

#### 前端
前端源代码在 webapp/ 目录中；davinci-ui/ 目录为编译后的前端文件

```
npm run build
```

#### 后端
用户配置在项目根目录 /config/ 下，项目启动脚本和升级补丁脚本在项目根目录 /bin/ 下， 后端代码及核心配置在 server/ 目录下, 日志在项目根目录 /log/ 下。注意：此处所指项目根目录都指环境变量 DAVINCI3_HOME 所配置的目录，在使用 IDE 开发过程中也需要配置环境变量，如 Idea 关于环境变量加载的优先级：`Run/Debug Configurations` 中配置的 `Environment variables` —>  IDE缓存的系统环境变量。

1. 打完整 release 包需要修改根目录下 /assembly/src/main/assembly/assembly.xml 中相关版本信息，然后在根目录下执行: `mvn clean package` 即可；
2. 打 server 包可直接在 server/ 目录下执行 `mvn clean package`。

## 1 环境准备

- JDK 1.8（或更高版本）
- MySql5.5（或更高版本）

## 2 配置部署

### 2.1 初始化目录

将下载好的 Davinci 包（Release 包，不是 Source 包）解压到某个系统目录，如：~/app/davinci

```
cd ~/app/davinci
unzip davinci-assembly_3.0.1-0.3.0-SNAPSHOT-dist.zip
```

解压后目录结构如下图所示：

![初始化目录](https://edp963.github.io/davinci/assets/images/deployment/2.1.1.png)

### 2.2 配置环境变量

将上述解压后的目录配置到环境变量 DAVINCI3_HOME

```
export DAVINCI3_HOME=~/app/davinci/davinci-assembly_3.0.1-0.3.0-SNAPSHOT-dist
```

### 2.3 初始化数据库

修改 bin 目录下 initdb.sh 中要的数据库信息为要初始化的数据库，如 davinci0.3

```
mysql -P 3306 -h localhost -u root -proot davinci0.3 < $DAVINCI3_HOME/bin/davinci.sql
```

运行脚本初始化数据库（注：由于 Davinci 系统数据库中包含存储过程，请务必在创建数据库时赋予执行权限）

```
sh bin/initdb.sh
```

### 2.4 初始化配置

Davinci 的配置主要包括：server、datasource、mail、chrome、cache 等配置

进入`config`目录，将`application.yml.example`重命名为`application.yml` 后开始配置

```
cd config
mv application.yml.example application.yml
```

**注意：由于 Davinci 使用 ymal 作为应用配置文件格式，请务必确保每个配置项键后的冒号和值之间至少有一个空格**

### 2.5 datasource 配置 （检查 `application.yml`中的 url，driver-class-name 是否一致，以及username，password是否正确）

这里的 datasource 配置指 Davinci 系统的数据源，配置如下：

```
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/davinci0.3?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true&serverTimezone=UTC
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
```

将上一步初始化的数据库地址和驱动类配置到`url`与 `driver-class-name`中，url 与 driver-class-name 中的参数不要做任何修改，然后修改正确的数据库访问用户和密码即`username`和`password`

## 3 自定义数据源 （需要将下列配置添加到 datasource_driver.yml）

### 3.1 打开自定义数据源配置文件

```
  mv datasource_driver.yml.example datasource_driver.yml
```

### 3.2 如下配置你的数据源，这里以 postgresql 为例

```
  postgresql:
    name: postgresql
    desc: postgresql
    version:
    driver: org.postgresql.Driver
    keyword_prefix:
    keyword_suffix:
    alias_prefix: \"
    alias_suffix: \"
```
## 4 新增数据源 （需要先执行第三步）

在数据源列表页，点击右上角“+”按钮弹出新增数据源表单 ![新增数据源1](https://edp963.github.io/davinci/assets/images/source/1.1.png) ![新增数据源2](https://edp963.github.io/davinci/assets/images/source/1.2.png)


用户名和密码为数据源连接账户及密码，连接 URL 请填写完整 JDBC 连接地址，通常格式为

```
jdbc:<数据源名称>://<数据源域名或IP>:(<端口>)/<数据源实例>(?<连接参数>)
```

其中端口和连接参数为可选项。部分数据源在写法上有少许差异，比如名称和域名/IP之间不需要 `//`、参数连接符使用 `;` 等等

配置信息栏填写连接参数，以键值对的形式，可根据不同数据源情况自行填写

在输入完上述内容之后，可以点击“点击测试”按钮检查数据源是否可以正常连接；测试成功并填写完毕之后，点击下方的保存按钮保存数据源配置

列如：

类型: `JDBC`

数据库: `postgres`

用户名: `username`

密码: `password`

连接url: `jdbc:postgres://hostname:5432/postgres`