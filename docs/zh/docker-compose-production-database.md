# Xboard 单机 Docker Compose + MySQL 部署文档

本文只说明一种部署方式：

- 单机部署。
- 使用 Docker Compose。
- 使用 MySQL。
- 不使用 Nginx。
- 不需要 clone 整个项目，只需要写 `compose.yaml` 拉取镜像。
- Xboard 宿主机端口使用 `10088`。
- Cloudflare Origin Rule 把请求转发到源站 `10088`。
- Ubuntu 使用 `ufw` 放行必要端口。
- MySQL 数据必须持久化。
- 必须定期备份数据库。

## 一、部署结构

```text
用户访问 https://你的域名
    |
    v
Cloudflare
    |
    v
服务器公网 IP:10088
    |
    v
Xboard 容器内部端口:7001
    |
    v
MySQL 容器
```

Xboard 和 MySQL 在同一个 Docker Compose 项目里。Xboard 通过服务名 `mysql` 连接数据库。

不需要下载整个 Xboard 项目。Docker 会直接拉取镜像：

```text
ghcr.io/cedar2025/xboard:latest
```

## 二、首次安装

本章节只用于第一次安装。安装完成后，日常启动和更新请看后面的“生产环境日常操作”。

### 1. 创建目录

使用 root 用户登录服务器后，执行：

```bash
mkdir -p /root/xboard
cd /root/xboard

mkdir -p backup storage/logs storage/theme plugins .docker/.data
touch .env
```

目录结构：

```text
/root/xboard
├── compose.yaml
├── .env
├── .docker/.data/
├── storage/logs/
├── storage/theme/
├── plugins/
└── backup/
```

`touch .env` 很重要。Xboard 容器会把宿主机的 `.env` 挂载到容器里的 `/www/.env`。

这里不需要手动填写 `.env`。先创建空文件即可：

```bash
touch .env
```

首次执行 `xboard:install` 时，安装程序会根据你的选择自动写入 `.env`，包括：

- `APP_KEY`
- `DB_CONNECTION=mysql`
- `DB_HOST=mysql`
- `DB_PORT=3306`
- `DB_DATABASE=xboard`
- `DB_USERNAME=xboard`
- `DB_PASSWORD=你输入的数据库密码`
- Redis 配置
- `INSTALLED=true`

安装完成后，不要删除 `.env`。以后启动、更新、迁移机器都要保留这个文件。

### 2. 创建 compose.yaml

在 `/root/xboard/compose.yaml` 写入：

```yaml
services:
  xboard:
    image: ghcr.io/cedar2025/xboard:latest
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    ports:
      - "10088:7001"
    volumes:
      - ./.env:/www/.env
      - ./.docker/.data/:/www/.docker/.data
      - ./storage/logs:/www/storage/logs
      - ./storage/theme:/www/storage/theme
      - ./plugins:/www/plugins
      - redis-data:/data
    environment:
      - RESOURCE_PROFILE=balanced
      - ENABLE_HORIZON=true
      - ENABLE_REDIS=true
      - docker=true

  mysql:
    image: mysql:8.4
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: xboard
      MYSQL_USER: xboard
      MYSQL_PASSWORD: change_this_xboard_password
      MYSQL_ROOT_PASSWORD: change_this_root_password
      TZ: Asia/Hong_Kong
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-time-zone=+08:00
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-pchange_this_root_password"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  redis-data:
  mysql-data:
```

上线前必须修改：

```yaml
MYSQL_PASSWORD: change_this_xboard_password
MYSQL_ROOT_PASSWORD: change_this_root_password
```

同时把 healthcheck 里的 root 密码改成同一个值：

```yaml
-pchange_this_root_password
```

### 3. 首次启动 MySQL

首次安装时，建议先单独启动 MySQL：

```bash
cd /root/xboard
docker compose up -d mysql
```

查看状态：

```bash
docker compose ps
docker compose logs -f mysql
```

等 MySQL 显示 `healthy` 后，再安装 Xboard。

### 4. 执行 Xboard 安装

不要使用 `ENABLE_SQLITE=true`，因为这里使用 MySQL。

安装前 `.env` 应该是一个空文件。安装程序会自动把数据库配置、应用密钥和安装状态写进去。

```bash
docker compose run -it --rm xboard php artisan xboard:install
```

安装时选择：

```text
数据库类型：MySQL
数据库地址：mysql
数据库端口：3306
数据库名：xboard
数据库用户名：xboard
数据库密码：你在 compose.yaml 里设置的 MYSQL_PASSWORD
Redis：启用 Docker 内置 Redis
管理员邮箱：填写你的邮箱
```

安装完成后，保存终端显示的：

- 管理员邮箱
- 管理员密码
- 管理后台地址

### 5. 首次启动全部服务

安装完成后启动全部服务：

```bash
docker compose up -d
```

查看状态：

```bash
docker compose ps
```

查看 Xboard 日志：

```bash
docker compose logs -f xboard
```

如果 Cloudflare 还没配置好，可以先测试：

```text
http://服务器IP:10088
```

## 三、Cloudflare 设置

本方案不使用 Nginx。

使用 Cloudflare Origin Rule，把访问域名的请求转发到源站 `10088` 端口。

请求路径：

```text
用户访问 https://你的域名
    -> Cloudflare
    -> Origin Rule 改写源站目标端口为 10088
    -> http://服务器公网IP:10088
```

Cloudflare Origin Rule 里设置：

```text
Destination Port / Rewrite to: 10088
```

注意：用户不需要访问 `https://你的域名:10088`。用户仍然访问正常 HTTPS 域名，由 Cloudflare 在回源时连接服务器的 `10088` 端口。

服务器防火墙或安全组需要允许 Cloudflare 访问 `10088`。如果可以限制来源 IP，建议只允许 Cloudflare IP 段访问 `10088`，不要对全网开放。

## 四、Ubuntu 防火墙端口

本方案部署在 Ubuntu 上，建议使用 `ufw` 管理宿主机防火墙。

### 1. 需要开放的端口

宿主机需要开放：

| 端口 | 协议 | 用途 | 是否必须 |
| --- | --- | --- | --- |
| `22` | TCP | SSH 登录服务器 | 必须，除非你改了 SSH 端口 |
| `10088` | TCP | Xboard 面板入口，Cloudflare 回源访问，也用于节点对接面板 | 必须 |

不需要开放：

| 端口 | 原因 |
| --- | --- |
| `3306` | MySQL 只给 Docker 内部的 Xboard 使用，不应该暴露公网 |
| `6379` | Redis 使用容器内部 Unix socket，不应该暴露公网 |
| `7001` | 容器内部端口已经映射到宿主机 `10088`，宿主机不需要开放 `7001` |
| `8076` | 这是容器内部 WebSocket 服务端口，Caddy 已经通过 `/ws` 转发，不需要暴露 |

节点对接面板时使用的也是 Xboard 入口：

```text
https://你的域名
```

Cloudflare 回源到：

```text
http://服务器公网IP:10088
```

节点 WebSocket 地址会由面板返回，默认是：

```text
wss://你的域名/ws
```

在 Docker 一体化部署里，`/ws` 会通过容器内 Caddy 转发到内部 `8076`，所以宿主机不需要开放 `8076`。

### 2. ufw 开放端口

先确保 SSH 不会被断开：

```bash
ufw allow 22/tcp
```

开放 Xboard 对外端口：

```bash
ufw allow 10088/tcp
```

启用 ufw：

```bash
ufw enable
```

查看状态：

```bash
ufw status verbose
```

### 3. 更安全的做法：只允许 Cloudflare 访问 10088

如果你只通过 Cloudflare 访问 Xboard，建议不要对全网开放 `10088`，而是只允许 Cloudflare IP 段访问。

做法是：

1. 先允许 SSH：

```bash
ufw allow 22/tcp
```

2. 只允许 Cloudflare IP 访问 `10088`。

Cloudflare IP 段会变化，建议以 Cloudflare 官方页面为准：

```text
https://www.cloudflare.com/ips/
```

示例命令格式：

```bash
ufw allow from CLOUDFLARE_IP段 to any port 10088 proto tcp
```

例如：

```bash
ufw allow from 173.245.48.0/20 to any port 10088 proto tcp
```

需要把 Cloudflare 官方列出的 IPv4 和 IPv6 段都加进去。

3. 不要再执行这个全开放命令：

```bash
ufw allow 10088/tcp
```

如果之前已经全开放，可以删除规则：

```bash
ufw status numbered
ufw delete 规则编号
```

### 4. 节点服务器自身端口

这里说的是 Xboard 面板服务器需要开放的端口。

如果你还有独立节点服务器，节点服务器还需要开放节点协议自己的入站端口，例如你在后台配置的 Shadowsocks、Trojan、VLESS 等端口。这些端口属于节点服务器，不属于 Xboard 面板服务器。

## 五、生产环境日常操作

本章节用于安装完成后的正式运行。不要再执行 `xboard:install`。

### 1. 日常启动

日常启动不需要分开启动 MySQL 和 Xboard，直接执行：

```bash
cd /root/xboard
docker compose up -d
```

Compose 会按 `depends_on` 先启动 MySQL，再启动 Xboard。

### 2. 日常停止

```bash
cd /root/xboard
docker compose down
```

注意：不要加 `-v`。

不要执行：

```bash
docker compose down -v
```

`-v` 会删除 Docker volume，可能导致 MySQL 数据丢失。

### 3. 查看状态

```bash
cd /root/xboard
docker compose ps
```

### 4. 查看日志

查看 Xboard：

```bash
docker compose logs -f xboard
```

查看 MySQL：

```bash
docker compose logs -f mysql
```

### 5. 重启服务

重启全部服务：

```bash
docker compose restart
```

只重启 Xboard：

```bash
docker compose restart xboard
```

只重启 MySQL：

```bash
docker compose restart mysql
```

## 六、生产环境更新

更新前先备份数据库。备份方法见“数据库备份”章节。

更新命令：

```bash
cd /root/xboard

docker compose pull
docker compose up -d
```

当前 Xboard 镜像启动时会自动执行：

```bash
php artisan xboard:update
```

通常不需要手动执行迁移命令。

更新完成后查看日志：

```bash
docker compose logs -f xboard
```

## 七、数据持久化

MySQL 数据保存在 Docker volume：

```yaml
volumes:
  mysql-data:
```

这个 volume 会挂载到 MySQL 容器内部：

```yaml
- mysql-data:/var/lib/mysql
```

所以正常操作不会删除数据库数据，例如：

```bash
docker compose down
docker compose up -d
docker compose pull
docker compose up -d
```

危险操作：

```bash
docker compose down -v
docker volume rm xboard_mysql-data
```

查看 MySQL volume：

```bash
docker volume ls | grep mysql-data
docker volume inspect xboard_mysql-data
```

需要长期保存的数据：

- `/root/xboard/.env`
- `/root/xboard/compose.yaml`
- `/root/xboard/plugins/`
- `/root/xboard/storage/theme/`
- `/root/xboard/.docker/.data/`
- MySQL 的 `mysql-data` volume
- `backup/` 里的数据库备份

## 八、数据库备份

### 1. 手动备份

```bash
cd /root/xboard
mkdir -p backup

docker compose exec mysql sh -c \
  'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction --routines --triggers xboard' \
  | gzip > backup/xboard_$(date +%F_%H-%M-%S).sql.gz
```

查看备份文件：

```bash
ls -lh backup/
```

### 2. 同时备份 .env

数据库备份之外，也要备份 `.env`：

```bash
cd /root/xboard
cp .env backup/env_$(date +%F_%H-%M-%S)
```

`.env` 里有数据库连接信息、应用密钥和安装状态。丢失它会增加恢复难度。

### 3. 自动备份

编辑 crontab：

```bash
crontab -e
```

加入：

```cron
30 3 * * * cd /root/xboard && mkdir -p backup && docker compose exec mysql sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction --routines --triggers xboard' | gzip > backup/xboard_$(date +\%F_\%H-\%M-\%S).sql.gz
40 3 * * * cd /root/xboard && find backup -name 'xboard_*.sql.gz' -mtime +14 -delete
```

含义：

- 每天 03:30 备份数据库。
- 每天 03:40 删除 14 天以前的数据库备份。

建议定期把 `backup/` 同步到服务器外部，例如对象存储、另一台服务器或本地电脑。只放在同一台服务器上，硬盘损坏时仍然会丢失。

### 4. 检查备份是否可用

检查压缩文件：

```bash
gunzip -t backup/xboard_你的备份文件.sql.gz
```

查看 SQL 内容开头：

```bash
gunzip -c backup/xboard_你的备份文件.sql.gz | head
```

更严格的方法是在另一台测试服务器上恢复一次，确认 Xboard 可以正常启动。

## 九、数据库恢复和迁移机器

迁移机器不需要重新安装 Xboard。不要再执行：

```bash
docker compose run -it --rm xboard php artisan xboard:install
```

迁移时需要带走：

- `/root/xboard/compose.yaml`
- `/root/xboard/.env`
- 数据库备份文件，例如 `backup/xboard_xxx.sql.gz`
- `/root/xboard/plugins/`
- `/root/xboard/storage/theme/`
- `/root/xboard/.docker/.data/`

在新机器上恢复：

```bash
cd /root/xboard
docker compose up -d mysql
```

恢复数据库：

```bash
gunzip -c backup/xboard_你的备份文件.sql.gz | docker compose exec -T mysql sh -c \
  'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" xboard'
```

恢复完成后启动全部服务：

```bash
docker compose up -d
```

查看日志：

```bash
docker compose logs -f xboard
```

## 十、常见问题

### MySQL 连接失败

如果 MySQL 和 Xboard 在同一个 `compose.yaml` 里，数据库地址必须填：

```text
mysql
```

不要填：

```text
127.0.0.1
```

在容器里，`127.0.0.1` 指的是 Xboard 容器自己，不是 MySQL 容器。

### .env 变成了目录

如果安装失败，并提示 `.env` 有问题，可能是 Docker 把 `.env` 创建成了目录。

处理：

```bash
docker compose down
rm -rf .env
touch .env
docker compose run -it --rm xboard php artisan xboard:install
```

### 如何确认 Xboard 使用的是 MySQL

查看 `.env`：

```bash
grep DB_CONNECTION .env
```

应该看到：

```text
DB_CONNECTION=mysql
```

也可以进入容器检查：

```bash
docker compose exec xboard php artisan tinker
```

然后执行：

```php
config('database.default')
```

应该返回：

```text
mysql
```

## 十一、最终建议

1. 首次安装时，先启动 MySQL，再执行 `xboard:install`，最后启动全部服务。
2. 安装完成后，日常只使用 `docker compose up -d` 启动。
3. 生产更新前必须先备份数据库。
4. 不要执行 `docker compose down -v`。
5. 每天自动备份数据库，并保留最近 14 天。
6. 定期把备份复制到服务器外部。
7. 迁移机器时恢复 `.env` 和数据库，不要重新安装。
8. Cloudflare Origin Rule 负责把请求转发到源站 `10088`，不需要 Nginx。
9. 面板服务器只需要开放 SSH 和 `10088`；不要开放 MySQL、Redis 或内部 WebSocket 端口。
