# VPS 自建 RSS 聚合与内容推送系统实战

> 算法推荐让人焦虑，RSS 让人重新掌控信息源。本文教你在一台 VPS 上用 FreshRSS / Miniflux 搭建私有阅读器，用 RSSHub 把没有 RSS 的网站变成 RSS，再通过 Webhook、Telegram、Kindle、邮件把内容精准推送到你手上——一套属于你自己的「信息管道」。

## 目录

- [一、为什么要自建 RSS](#一为什么要自建-rss)
- [二、选型：FreshRSS vs Miniflux vs Tiny Tiny RSS](#二选型freshrss-vs-miniflux-vs-tiny-tiny-rss)
- [三、Docker 部署 FreshRSS](#三docker-部署-freshrss)
- [四、部署 Miniflux（极简派）](#四部署-miniflux极简派)
- [五、RSSHub：万物皆可 RSS](#五rsshub万物皆可-rss)
- [六、RSS-Bridge 与自建全文输出](#六rss-bridge-与自建全文输出)
- [七、自动抓取全文与去广告](#七自动抓取全文与去广告)
- [八、内容推送：Webhook / Telegram / Kindle / 邮件](#八内容推送webhook--telegram--kindle--邮件)
- [九、规则过滤：只看你想要的](#九规则过滤只看你想要的)
- [十、抓取调度与性能优化](#十抓取调度与性能优化)
- [十一、备份、迁移与安全](#十一备份迁移与安全)
- [十二、常见故障与排查](#十二常见故障与排查)
- [十三、FAQ](#十三faq)
- [十四、相关资源](#十四相关资源)

---

## 一、为什么要自建 RSS

- **无算法**：只看你订阅的，不被平台推荐牵着走。
- **无广告**：阅读器里清清爽爽。
- **不丢历史**：平台封号/删帖，你的存档还在。
- **跨平台**：手机、平板、电脑、Kindle 都能读。
- **可自动化**：新内容自动进你的工作流（推送、归档、翻译、摘要）。

代价是需要一台常年在线的机器来抓取与存储。这台机器最好是**线路稳定、长期在线**的，否则抓取任务经常中断——这也是很多人选 [VPSVIP](https://vpsvip.net) 这类优化线路 VPS 做自托管中枢的原因。

---

## 二、选型：FreshRSS vs Miniflux vs Tiny Tiny RSS

| 维度 | FreshRSS | Miniflux | Tiny Tiny RSS |
|------|----------|----------|----------------|
| 语言 | PHP | Go | PHP |
| 资源占用 | 中 | 极低 | 中 |
| 界面美观 | 好 | 简洁实用 | 一般 |
| 移动端 | 官方 App + 第三方 | 官方 App + 第三方 | 第三方 |
| 扩展性 | 强（插件/API） | 中（规则过滤） | 强（插件） |
| 全文抓取 | 内置选项 | 内置 | 插件 |
| 适合 | 大多数人 | 极简/低配 | 折腾党 |

**建议**：求省心就 **FreshRSS**，求极简省资源就 **Miniflux**。两者都提供 **Google Reader API**，可被众多客户端（Reeder、NetNewsWire、Fluent Reader）直接连接。

---

## 三、Docker 部署 FreshRSS

### 3.1 目录与 Compose

```yaml
# docker-compose.yml
services:
  freshrss:
    image: freshrss/freshrss:latest
    container_name: freshrss
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      TZ: Asia/Shanghai
      CRON_MIN: "*/20"          # 每 20 分钟抓取一次
    volumes:
      - ./data:/var/www/FreshRSS/data
      - ./extensions:/var/www/FreshRSS/extensions
```

```bash
mkdir -p data extensions && docker compose up -d
```

访问 `http://服务器IP:8081` 完成初始化（选 SQLite 即可，小规模无需 MySQL）。

### 3.2 反向代理 + HTTPS

```nginx
server {
    listen 443 ssl http2;
    server_name rss.example.com;

    ssl_certificate     /etc/letsencrypt/live/rss.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rss.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> FreshRSS 的 Google Reader API 需要正确的 `X-Forwarded-Proto`，否则客户端连不上。

### 3.3 导入订阅源

- 从旧阅读器导出 **OPML**，在「订阅管理 → 导入/导出」上传即可。
- 批量加种子源：内置了一系列推荐源，可按分类勾选。

---

## 四、部署 Miniflux（极简派）

```yaml
services:
  miniflux:
    image: miniflux/miniflux:latest
    restart: unless-stopped
    ports:
      - "8082:8080"
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://miniflux:secret@db/miniflux?sslmode=disable
      RUN_MIGRATIONS: 1
      CREATE_ADMIN: 1
      ADMIN_USERNAME: admin
      ADMIN_PASSWORD: change-me-please
      POLLING_FREQUENCY: 20
      BASE_URL: https://rss.example.com
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: miniflux
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: miniflux
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "miniflux"]
      interval: 10s
```

Miniflux 的优势：**内存占用极低（几十 MB）**、内置全文抓取与规则过滤、界面无冗余。

---

## 五、RSSHub：万物皆可 RSS

很多网站（社交、视频、论坛）没有 RSS，RSSHub 用「路由」把它们变成 Feed。

### 5.1 Docker 部署

```yaml
services:
  rsshub:
    image: diygod/rsshub:latest
    restart: unless-stopped
    ports:
      - "1200:1200"
    environment:
      NODE_ENV: production
      CACHE_TYPE: memory
      CACHE_EXPIRE: 3600
    depends_on:
      - browserless
  browserless:
    image: browserless/chrome:latest
    restart: unless-stopped
```

（部分路由需要 `browserless` 渲染 JS 页面。）

### 5.2 路由示例

| 需求 | 路由 |
|------|------|
| 某站点的公开账号 | `/twitter/user/:id`（需配置，注意平台变化） |
| 视频频道 | `/youtube/channel/:id` |
| 论坛板块 | `/discourse/:host/:category` |
| 播客 | `/podcast/:id` |
| GitHub 仓库动态 | `/github/repos/:user/:repo/releases` |
| 通用网页监控 | `/web/follow/:url` |

把生成的 `https://rss.example.com/xxx` 地址填进 FreshRSS 即可。

### 5.3 自建 vs 公共实例

- 公共实例（rsshub.app）**限流、常被墙**，不稳定。
- 自建实例**只有你自己用**，稳定、可加自定义路由。

**强烈建议自建。**

---

## 六、RSS-Bridge 与自建全文输出

- **RSS-Bridge**（PHP）：提供一批「桥」，把无 RSS 网站转成 Feed，与 RSSHub 互补。适合 RSSHub 没有覆盖的站点。
- **全文输出**：很多 Feed 只给摘要，用 FreshRSS 的全文抓取或第三方服务补全。

```yaml
services:
  rss-bridge:
    image: rssbridge/rss-bridge:latest
    restart: unless-stopped
    ports:
      - "3000:80"
    volumes:
      - ./config.ini.php:/config/config.ini.php
```

---

## 七、自动抓取全文与去广告

### 7.1 FreshRSS 内置

「设置 → 归档 → 全文」里选「抓取全文」，配合 `fivefilters full-text`（可自建）获得接近原站的完整阅读体验。

### 7.2 自建 Full-Text RSS

```yaml
services:
  fulltextrss:
    image: heussd/fivefilters-full-text-rss:latest
    restart: unless-stopped
    ports:
      - "3001:80"
    environment:
      - TZ=Asia/Shanghai
```

在 FreshRSS 里把全文抓取地址指向它即可。

### 7.3 用 CSS 选择器补全

对特定站点，可直接用「自定义全文选择器」（XPath/CSS）精确抓取正文，避免夹杂导航与广告。

---

## 八、内容推送：Webhook / Telegram / Kindle / 邮件

自建 RSS 的真正威力在于**把新内容推到你常在的地方**。

### 8.1 Webhook 自动触发

FreshRSS 支持在「扩展 → Webhook」里配置：有新条目就 POST 到你的服务。

```json
{
  "url": "https://your-hook.example.com/freshrss",
  "event": "new_articles",
  "method": "POST"
}
```

接收端可做：翻译、摘要、入库、转发。

### 8.2 推到 Telegram

```bash
#!/bin/bash
# 用 RSSHub + rss-to-telegram 或自写脚本
BOT_TOKEN="123:ABC"
CHAT_ID="你的chatid"
FEED_URL="https://rss.example.com/feed/all"

curl -s "$FEED_URL" | grep -oP '(?<=<link>).*?(?=</link>)' | head -5 | while read url; do
  curl -s -X POST "https://api.telegram.org/bot$BOT_TOKEN/sendMessage" \
    -d chat_id="$CHAT_ID" -d text="$url"
done
```

更省心的做法：用 **RSS-to-Telegram-Bot（RSStT）** 的 Docker 镜像，它专门干这件事，支持去重、媒体、翻译。

### 8.3 推到 Kindle

把长文打包成电子书，推到 Kindle 邮箱：

```bash
# 用 calibre 的 ebook-convert + Kindle 邮箱
# 1) 抓取文章 → 2) 合成 epub → 3) 邮件发送
ebook-convert article.html article.epub
msmtp -a default kindle-xxxx@kindle.com < mail.txt
```

配合 cron 每天定时打包「今日精选」，Kindle 上离线阅读，体验极好。

### 8.4 邮件摘要

用 `mailparse` 或简单脚本，把每日未读生成 HTML 邮件发给自己：

```bash
curl -s "https://rss.example.com/api/greader.php/reader/api/0/stream/contents/user/-/state/com.google/unread" \
  -H "Authorization: GoogleLogin auth=$AUTH" | \
  python3 daily_digest.py | msmtp you@example.com
```

### 8.5 推送方式对比

| 通道 | 实时性 | 适合 |
|------|--------|------|
| Webhook | 实时 | 接自动化/入库/翻译 |
| Telegram | 实时 | 手机即时提醒 |
| Kindle | 定时 | 深度长文、离线 |
| 邮件 | 定时 | 每日摘要、归档 |

---

## 九、规则过滤：只看你想要的

信息过载的解法不是「少订阅」，而是**精准过滤**。

### 9.1 FreshRSS 过滤

- 为每个 Feed 设置「仅保留包含关键词的条目」。
- 走**正则**去重、屏蔽标题党（如标题含「震惊」「点击查看」）。

### 9.2 Miniflux 规则

Miniflux 支持按 Feed 设置过滤规则（标题/内容/URL 匹配正则），命中则隐藏或标记。

```
title=~"招聘|广告"  →  标记已读
url!~"example.com" →  不抓取
```

### 9.3 分层阅读法

1. **必读**：给最重要的源打星，推送即时提醒。
2. **扫读**：一般源进「未读」批量扫。
3. **归档**：低价值的只存不推，需要时搜。

**关键**：让推送只承载「必读」，其余靠主动进入阅读器，才是可持续的信息习惯。

### 9.4 关键词高亮与自动化动作

除过滤外，还可以用关键词做「高亮」或「自动打标签」，例如把含 `Release`、`CVE`、`优惠` 的条目单独标记，再从高亮流里生成一个专用 Feed 推送。这比笼统的「全部推送」精准得多。

```bash
# 思路示意：抽取标题含关键词的条目，生成独立 feed
curl -s "$FRESHRSS/api/greader.php/reader/api/0/stream/contents/feed/all" \
  -H "Authorization: GoogleLogin auth=$AUTH" \
  | jq -r '.items[] | select(.title | test("Release|CVE|优惠")) | .title + "  " + (.canonical[0].href)'
```

---

## 十、抓取调度与性能优化

几百个源同时抓取，会瞬间打满 CPU 与带宽，也容易触发对端限流。合理的调度很重要。

### 10.1 抓取频率

| 源类型 | 建议频率 |
|--------|----------|
| 新闻/快讯 | 15~20 分钟 |
| 博客/周刊 | 1~2 小时 |
| 低更新频率 | 每天 1 次 |

FreshRSS 的 `CRON_MIN`、Miniflux 的 `POLLING_FREQUENCY` 都是**分钟**单位；不要图快设成 `*`，弊大于利。

### 10.2 错峰与并发

- 给抓取任务加**随机抖动**，避免整点一起冲。
- 限制并发连接数（FreshRSS 有 `MAX_CONCURRENT` 相关配置，RSSHub 用 `CACHE_EXPIRE` 降低回源）。
- 对同一域名的多个源，尽量合并或降低频率。

### 10.3 缓存策略

RSSHub 默认内存缓存，重启即丢。生产建议换 Redis：

```yaml
  rsshub:
    environment:
      CACHE_TYPE: redis
      REDIS_URL: redis://redis:6379/
  redis:
    image: redis:7-alpine
    restart: unless-stopped
```

### 10.4 资源占用参考

| 组件 | 空载内存 | 说明 |
|------|----------|------|
| FreshRSS | ~120MB | PHP-FPM |
| Miniflux | ~40MB | Go，极省 |
| RSSHub | ~150MB | Node，视路由 |
| browserless | ~300MB+ | 仅需 JS 路由时启用 |
| Postgres | ~60MB | Miniflux 依赖 |

**结论**：单机 2 核 2G 可轻松承载个人级 RSS 系统；若只求省资源，Miniflux + 按需 RSSHub 是最佳组合。

---

## 十一、备份、迁移与安全

### 10.1 备份

```bash
# FreshRSS（SQLite）
docker compose exec freshrss tar czf - /var/www/FreshRSS/data \
  > freshrss-$(date +%F).tar.gz

# Miniflux（Postgres）
docker compose exec -T db pg_dump -U miniflux miniflux | gzip > miniflux-$(date +%F).sql.gz
```

订阅列表可另导出 OPML，便于跨阅读器迁移。

### 10.2 安全

- 全站 HTTPS；反向代理加 Basic Auth 或只允许内网/VPN 访问。
- Miniflux 用强 admin 密码，及时升级镜像。
- RSSHub 自建实例不要暴露敏感的账号态路由。
- 启用容器非 root 运行，限制挂载目录。

### 10.3 迁移

换机器时：恢复数据 + 恢复 OPML + 改 DNS 即可。因为是 Docker，重建环境只需 5 分钟。

---

## 十二、常见故障与排查

| 现象 | 原因 | 处理 |
|------|------|------|
| 抓取不到新内容 | 目标站点改版 / 被限流 | 检查 RSSHub 路由、看日志 |
| 客户端连不上 API | 缺 `X-Forwarded-Proto` | 反代补上该头 |
| 全文抓取失败 | 站点反爬 / 需 JS | 用 browserless，或换选择器 |
| 推送重复 | 无去重 | 记录已推送 ID，或改用 RSStT |
| 内存占用高 | 抓取并发过大 | 降低并发/频率，Miniflux 更省 |
| 磁盘涨得快 | 缓存无上限 | 设 `CACHE_EXPIRE`，定期清缓存 |

排查入口：

```bash
docker compose logs -f freshrss
docker compose logs -f rsshub
curl -s http://127.0.0.1:1200/github/repos/vpsvip-net/vps-tools/releases | head
```

---

## 十三、FAQ

**Q1：自建 RSS 需要多强的服务器？**
A：FreshRSS + RSSHub 组合，1 核 1G 可跑，2 核 2G 更稳。Miniflux 极省，512M 也够。

**Q2：RSSHub 一定要自建吗？**
A：强烈建议。公共实例限流且不稳定；自建只服务你一人，速度和可用性都更好。

**Q3：怎么把没有 RSS 的网站接进来？**
A：先查 RSSHub 有没有现成路由；没有就用 RSS-Bridge 或 `/web/follow` 通用监控路由。

**Q4：手机上怎么读？**
A：用 Reeder / NetNewsWire / Fluent Reader 等客户端连接 FreshRSS 或 Miniflux 的 API 即可，数据在服务器，多端同步。

**Q5：怎么防止信息过载？**
A：给源分层 + 只把「必读」推送。阅读器负责「有得读」，推送负责「提醒你读」。

**Q6：能自动翻译外文源吗？**
A：可以。在 Webhook 接收端接翻译 API，或推送到自托管模型做摘要+翻译，再转发。

**Q7：数据会被第三方看到吗？**
A：自建实例数据全在你自己的服务器与数据库里。注意关闭不必要的公网暴露。

**Q8：换服务器会丢订阅吗？**
A：不会，只要备份了数据库/卷并导出 OPML；Docker 方案重建极快。

---

## 十四、相关资源

- [VPSVIP 官网](https://vpsvip.net) —— 稳定优化线路 VPS，适合长期在线的 RSS 抓取与推送中枢
- [ClashVIP](https://clashvip.net) —— 网络与节点资源
- [nav.clashvip.net](https://nav.clashvip.net) —— 导航与工具集合
- [clashhub.net](https://clashhub.net) —— 教程与文档
- [bbs.clashhub.net](https://bbs.clashhub.net) —— 社区讨论
- [clash-for-windows.net](https://clash-for-windows.net) —— 客户端下载
- [FreshRSS 官网](https://freshrss.org/)
- [Miniflux 官网](https://miniflux.app/)
- [RSSHub 文档](https://docs.rsshub.app/)
- [RSS-Bridge](https://rss-bridge.org/)
- [RSStT (RSS to Telegram)](https://github.com/Rongronggg9/RSS-to-Telegram-Bot)

---

## 免责声明

1. 本仓库内容仅供技术学习与参考；
2. 请遵守所在国家/地区法律法规以及各网站的使用条款与 robots 规则；
3. 自动化抓取请控制频率，尊重目标站点，避免造成负担；
4. 请妥善保管账号凭据与备份，做好服务器安全加固。

## 许可证

MIT License

---
更新时间：2026-09-22
