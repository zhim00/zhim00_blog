---
date: 2026-03-03T17:10:50+08:00
draft: false
title: 使用 Docker 部署 PostgreSQL + pgvector
tags: [""]
description: 保姆级教学，从零开始部署带 pgvector 的 PostgreSQL，含踩坑指南
image: pg.jpg
categories: ["杂谈"]
---
## 前言

随着大模型（LLM）和 RAG（检索增强生成）应用的兴起，向量数据库成为了基础设施中的标配。PostgreSQL 凭借强大的 `pgvector` 插件，成为了目前最流行的向量数据库解决方案之一。

本文将详细介绍如何在 CentOS 7 环境下，使用 Docker 和 Docker Compose 快速部署带有 `pgvector` 的 PostgreSQL

---

## 1. 环境准备

本教程基于以下环境进行验证，建议版本接近以避免兼容性问题：

*   **OS:** CentOS Linux release 7.9.2009 (Core)
*   **Docker:** 26.1.4 (Docker Engine - Community)
*   **Docker Compose:** v2.27.1

**前置检查：**
确保 Docker 服务已启动。

```bash
systemctl start docker
docker --version
docker compose version
```

---

## 2. 部署步骤

### 2.1 创建工作目录

为了方便管理和数据持久化，我们在服务器上创建一个专门的目录：

```bash
mkdir -p /opt/docker/pgvector
cd /opt/docker/pgvector
```

### 2.2 编写 docker-compose.yml

我们将使用官方推荐的 `pgvector/pgvector` 镜像，该镜像基于官方 Postgres 构建并预装了 vector 扩展。

创建并编辑文件：

```bash
vi docker-compose.yml
```

**配置内容如下：**

```yaml
services:
  postgres:
    # 推荐使用 pg16 版本，性能更佳
    # 如果客户端工具较旧，请阅读文末的“坑点排查”部分，可能需要降级到 pg14
    image: pgvector/pgvector:pg16
    container_name: postgres_vector
    restart: always
    environment:
      POSTGRES_USER: postgres         # 数据库用户名
      POSTGRES_PASSWORD: postgres # 数据库密码
      # 不设置 POSTGRES_DB 时，默认数据库名会与用户名一致（即 postgres）
    ports:
      - "5432:5432"                 # 主机端口:容器端口
    volumes:
      # 数据持久化映射，确保容器删除后数据不丢失
      # ./pgdata 代表当前目录下的 pgdata 文件夹
      - ./pgdata:/var/lib/postgresql/data
    # 共享内存配置，PostgreSQL 建议适当调大
    shm_size: 1g
```

### 2.3 启动服务

使用 Docker Compose V2 命令启动容器（如果镜像不存在会先去拉去镜像）：

```bash
# -d 表示后台运行
docker compose up -d
```

查看容器状态：

```bash
docker compose ps
```

状态显示 `Up` 即表示启动成功。

---

## 3. 初始化 pgvector 插件

容器启动后，PostgreSQL 服务已经运行，但需要在具体的数据库中启用 `vector` 扩展才能存储向量数据。

### 3.1 进入容器数据库

这一步有客户端工具的建议用**客户端工具**连接，会方便好看一点，比如navicat、datagrip等等，就不用执行下方的代码了

```bash
# 注意：如果不指定数据库名(-d)，默认会尝试连接与用户同名的数据库
docker compose exec postgres psql -U postgres -d postgres
```

### 3.2 启用扩展

在 SQL 交互界面（`console`）执行：

```sql
CREATE EXTENSION vector;
```

提示 `CREATE EXTENSION` 即成功。

> **注意**：扩展是“==数据库级别== ”的。如果你以后创建了新数据库 `testdb`，需要在 `testdb` 里再次执行此命令。

---

## 4. 坑点排查

### 坑点一：连接报错 `column "datlastsysoid" does not exist`

如果你使用 旧版Navicat、旧版 pgAdmin 或 DBeaver 连接数据库时，可能会遇到如下报错：

```text
ERROR: column "datlastsysoid" does not exist
LINE 1: SELECT DISTINCT datlastsysoid FROM pg_database
```

**原因分析：**
PostgreSQL 15 及以上版本删除了系统表中的 `datlastsysoid` 字段。如果你使用的 Docker 镜像是 `pg16` 或 `pg15`，而客户端工具（如 Navicat 12/15）版本较老，就会因为查询不存在的字段而报错。

**解决方案（二选一）：**

1.  **方案 A（推荐）：升级客户端**

	*   将 Navicat 升级到 **16.2+** 版本。
	*   或者使用最新版的 DBeaver（免费且兼容性好）。
	*   IDEA内置的也可以凑活着用

2.  **方案 B：降级数据库版本**

	*   如果你必须使用旧版 Navicat，需将 `docker-compose.yml` 中的镜像改为 `pgvector/pgvector:pg14`。
	*   **警告**：降级版本需要删除现有的数据卷，否则无法启动！

	```bash
	docker compose down
	rm -rf ./pgdata  # 删除旧版数据
	# 修改 yaml 后重新启动
	docker compose up -d
	```

### 坑点二：防火墙设置

如果无法从外部连接数据库，注意**检查 CentOS 防火墙：**

```bash
# 检查状态
systemctl status firewalld

# 如果开启，需放行 5432 端口
firewall-cmd --zone=public --add-port=5432/tcp --permanent
firewall-cmd --reload

# 或者像我一样一劳永逸，直接关掉
# 停止 firewalld 服务
systemctl stop firewalld

# 禁用 firewalld 服务，防止开机自启
systemctl disable firewalld

# 确认状态
systemctl status firewalld
```