# blog-lts

本仓库是个人博客 **Kingcanfish 的小星球** 的 Hugo 源码与构建产物（`public/`）。

- 在线地址：`https://blog.gxy.plus`
- 技术栈：Hugo（建议 extended）+ `hugo-theme-stack`（`themes/stack` 子模块）
- 内容方向：Go/后端/网络/数据结构等技术笔记 + 年度回顾/随笔

## 文章索引（按主题）

### 年度回顾 / 随笔

- [2019年度总结](content/posts/2019-year/index.md)
- [2020 杂事随记](content/posts/2020/index.md)
- [2024 ⏩⏩](content/posts/2024/index.md)
- [2025 年度回顾：缝隙中的光与尘](content/posts/2025-year-blog/index.md)
- [2019暑假目标](content/posts/2019-summer-goals/index.md)
- [2020寒假清单](content/posts/2020-winter-todo-list/index.md)
- [说点什么](content/posts/say-something/index.md)
- [好想回饶中啊](content/posts/miss-raopzhong/index.md)
- [送给银河系之外的你](content/posts/to-you-beyond-galaxy/index.md)

### Go / 并发 / 底层

- [2020 Go 开发者学习路线](content/posts/Go_developer/index.md)
- [Golang 调度器 GMP 原理与调度全分析](content/posts/goroutine/index.md)
- [Go的channel 的小学习](content/posts/GoChannel/index.md)
- [Go 语言的 sync.Map 浅析](content/posts/syncMap/index.md)
- [【转】GO：sync.Mutex 的实现与演进](content/posts/go_mutex/index.md)
- [Go和Redis中的Hash](content/posts/HashInGoAndRedis/index.md)
- [值类型, 引用类型, 值传递, 引用传递一锅炖](content/posts/PassedByValue/index.md)

### 后端 / 数据库 / 运维

- [浅谈 MySQL 的连表查询](content/posts/mysql-join-query/index.md)
- [nginx配置的一些坑](content/posts/nginx-configuration-tips/index.md)
- [使用acme.sh配置https的证书](content/posts/wildcard-ssl-certificate/index.md)
- [scoop 安装软件命令](content/posts/scoop_cmd/index.md)

### 计算机基础 / 面试

- [家园后端学习书单](content/posts/backend-learning-books/index.md)
- [一些学习知识点](content/posts/learnlist/index.md)
- [计算机网络要点](content/posts/computer-network/index.md)
- [计算机网络知识脑图](content/posts/networkmind/index.md)
- [数据结构与算法](content/posts/data-structure/index.md)
- [生产者消费者模型与信号量](content/posts/producer-consumer-semaphore/index.md)

### 建站

- [搭建自己的博客（一）](content/posts/build-my-blog-1/index.md)

## 本地运行

准备：安装 Hugo（建议 **extended**，主题最低要求 `>= 0.87.0`）。

1) 拉取源码（包含主题子模块）

```bash
git clone --recurse-submodules <repo-url>
```

如果已 clone：

```bash
git submodule update --init --recursive
```

2) 启动本地预览

```bash
hugo server -D
```

## 写作约定（与现有 posts 保持一致）

- 推荐使用 Hugo Page Bundle：每篇文章一个目录，入口为 `content/posts/<slug>/index.md`
- 图片/附件放在同目录（或子目录）并用相对路径引用，例如：`photos/xxx.jpg`
- 文章摘要分隔符：`<!--more-->`
- 常用 Front Matter 字段：`title` / `date` / `description` / `tags` / `categories` / `featured_image` / `comment` / `draft`

创建新文章（bundle）：

```bash
hugo new posts/<slug>/index.md
```

## 构建与发布

```bash
hugo --minify
```

- 默认输出目录：`public/`
- 站点配置：`hugo.yaml`（`baseurl`、评论 `giscus`、主题配置等）
