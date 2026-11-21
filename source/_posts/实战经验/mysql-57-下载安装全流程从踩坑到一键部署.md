---
title: "MySQL 5.7 下载安装全流程：从踩坑到一键部署"
date: 2026-11-21 10:37:18
categories:
  - "实战经验"
tags:
  - "mysql"
  - "数据库"
  - "安装"
  - "实战经验"
---
## 写在前面

MySQL 5.7 现场安装看着简单，实际上每年都有人栽在同一个地方：解压完就去启动服务，结果服务死活起不来，翻开日志全是 `failed to set datadir`、`Table 'mysql.user' doesn't exist`。我自己也踩过一整轮，从排查到做出一套"拷进去右键就能装"的脚本，把坑全填了。这篇把下载、安装、初始化、排障一次讲清楚，照着做基本不会卡。

## 一、先搞懂为什么会启动失败

动手之前先记住三个高频坑，后面所有步骤都是围绕它们设计的：

| 坑 | 现象 | 根本原因 |
|---|---|---|
| 没有 my.ini | 报 `Can't change dir to ...\data` 或 `failed to set datadir` | 解压版不带配置文件，mysqld 用内置默认 datadir，路径根本不存在 |
| datadir 指错 | 服务找不到数据目录 | 服务注册时没带 `--defaults-file`，mysqld 不知道配置文件在哪 |
| 没初始化 | 报 `Table 'mysql.user' doesn't exist` | 光解压不初始化，mysql、sys 这些系统库压根没建出来 |

一句话：**解压 ≠ 安装，必须先有配置、再初始化、最后注册服务**。

## 二、方案选择：zip 版还是 MSI 版

| 对比 | MSI 安装器 | zip 免安装版（本文方案） |
|---|---|---|
| 网络依赖 | 部分步骤要联网拉组件 | 完全离线，拷进去就装 |
| 可复现性 | 图形向导一步步点，容易漏 | 脚本固定，每台机器结果一致 |
| 配置文件 | 分散在 ProgramData | my.ini 就放 MySQL 根目录 |
| 服务参数 | 向导隐式处理 | 显式带 `--defaults-file`，datadir 不会跑偏 |
| 排障 | 难定位 | 步骤、日志、验证都在脚本里 |

现场、内网、批量部署，优先 zip 版。

## 三、下载

MySQL 5.7 已经停止官方维护（EOL），官网首页默认推 8.x，得去归档页下：

1. 打开官方归档：<https://downloads.mysql.com/archives/community/>
2. Product Version 选 **5.7.36**（5.7 最后一个版本）
3. Operating System 选 **Microsoft Windows**
4. 下载 **mysql-5.7.36-winx64.zip**（64 位免安装版）
5. 现场通常没外网，务必在办公室先下好，用 U 盘拷过去

> 挑 5.7.36 而不是 8.x：老业务系统大量依赖 5.7 的认证插件和 SQL 语法，升级成本远高于用最后一个稳定版。

## 四、安装前准备

| 项 | 要求 | 说明 |
|---|---|---|
| 管理员账号 | 必须 | 装服务、写目录都要管理员权限 |
| VC++ 运行库 | 建议先装 | Visual C++ Redistributable 2013 x64，缺了初始化会报 `MSVCR120.dll` 找不到 |
| 端口 3306 | 确认空闲 | 被占用就换端口，或先停掉占用程序 |
| 路径 | 纯英文、无空格 | 推荐 `C:\mysql-5.7.36-winx64`，空格路径在 MySQL 生态里坑多 |

## 五、安装步骤

### 1. 解压

解压到纯英文路径，例如 `C:\mysql-5.7.36-winx64`，里面应有 `bin\`、`docs\`、`share\` 等目录。

注意：解压出来**默认没有 data 目录、没有 my.ini**，这是正常的，接下来生成。

### 2. 写 my.ini

在 MySQL 根目录建一个 `my.ini`（注意是 ANSI/UTF-8 无 BOM 编码）：

```ini
[mysqld]
basedir=C:/Program Files/MySQL/MySQL Server 5.7
datadir=C:/ProgramData/MySQL/MySQL Server 5.7/data
port=3306
character-set-server=utf8mb4
collation-server=utf8mb4_general_ci
default-storage-engine=INNODB
explicit_defaults_for_timestamp=true
max_connections=200

[client]
port=3306
default-character-set=utf8mb4
```

两个关键点：

- `basedir` / `datadir` 必须是你要放的真实路径，路径用**正斜杠 `/`**（反斜杠在 ini 里容易被当转义）
- `datadir` 指向的目录可以先不存在，初始化时会自动建；但**必须是空目录或全新目录**

### 3. 初始化（最关键的一步）

用管理员打开命令行，切到 MySQL 根目录执行：

```bat
bin\mysqld.exe --defaults-file=my.ini --initialize-insecure --console
```

- `--defaults-file` **必须是 mysqld 后面的第一个参数**，否则不生效
- `--initialize-insecure` 表示 root 用空密码，省去去日志里翻随机密码的麻烦（装完立刻改密码即可）
- 看到程序正常退出、data 目录下出现 `ibdata1`、`ib_logfile*` 和 `mysql\` 目录，就说明初始化成功

如果这里报 `Table 'mysql.user' doesn't exist` 之外的错，先看屏幕日志，多半是路径写错或目录非空。

### 4. 注册服务并启动

```bat
bin\mysqld.exe --defaults-file=my.ini --install MySql57
net start MySql57
```

同样，`--defaults-file` 放在最前面。服务名 `MySql57` 可以自己改。

### 5. 验证

```bat
bin\mysql.exe -uroot -e "SELECT VERSION(), @@datadir;"
```

能打印出版本号和正确的数据目录，就装好了。再看系统库是否齐全：

```bat
bin\mysql.exe -uroot -e "SHOW DATABASES;"
```

应能看到 `information_schema`、`mysql`、`performance_schema`、`sys` 四个库。

## 六、装完立刻做的两件事

**1）改 root 空密码**

```bat
bin\mysql.exe -uroot -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '你的密码';"
```

**2）建业务库和应用账号**（别让应用直接用 root）

```sql
CREATE DATABASE IF NOT EXISTS myapp_db
  DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_general_ci;

CREATE USER IF NOT EXISTS 'app'@'localhost' IDENTIFIED BY '强密码';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp_db.* TO 'app'@'localhost';
FLUSH PRIVILEGES;
```

应用和数据库不在同一台机器时，把账号的 `localhost` 改成应用服务器 IP 或 `%`，并放行防火墙 3306。

## 七、常见故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| `Can't change dir to ...\data` | 没有 my.ini 或 datadir 写错 | 检查 my.ini 是否存在、`basedir`/`datadir` 是否正确 |
| `Table 'mysql.user' doesn't exist` | data 是半初始化残目录 | 删掉 data 重新走第 3 步初始化 |
| 缺 `MSVCR120.dll` | 没装 VC++ 2013 运行库 | 装 Visual C++ Redistributable 2013 x64 |
| 服务启动就停 | 端口占用 / 目录无权限 / my.ini 错 | 看 `data\*.err` 最后 30 行 |
| 服务正常但远程连不上 | 防火墙挡了 3306 | 防火墙放行 TCP 3306 |

看日志的快捷命令：

```bat
type "C:\mysql-5.7.36-winx64\data\*.err" | more
```

## 八、日常运维与卸载

| 操作 | 命令 |
|---|---|
| 启动 / 停止 | `net start MySql57` / `net stop MySql57` |
| 改配置 | 改 my.ini 后重启服务 |
| 卸载服务（保留数据） | `bin\mysqld.exe --remove MySql57` |

卸载只移除服务，data 目录不会删，数据还在；确认不要了再手动删 data。

## 结语

MySQL 5.7 现场安装其实就三件事：**写对 my.ini、执行初始化、注册服务时带 `--defaults-file`**。把这三点记住，无论内网离线还是批量部署，都能一遍过。我把这套流程固化成了一键脚本（生成配置 → 初始化 → 装服务 → 启动 → 验证），现场拷进去右键管理员运行即可，需要的话可以按本文逻辑自行改写。
