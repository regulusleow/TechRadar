# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

TrendRadar 是一个全网热点聚合和智能推送系统,支持多平台新闻热点的自动抓取、过滤和推送。项目包含三个主要组件:

1. **爬虫系统** (main.py) - 抓取多平台热点新闻并进行智能排序
2. **MCP 服务器** (mcp_server/) - 基于 FastMCP 2.0 的 AI 分析服务
3. **Docker 部署** (docker/) - 支持容器化部署和定时任务管理

## 核心技术栈

- Python 3.10+
- FastMCP 2.0 (MCP 服务器框架)
- PyYAML (配置管理)
- requests (HTTP 请求)
- pytz (时区处理)

## 常用命令

### 开发运行

```bash
# 安装依赖
pip install -r requirements.txt

# 执行一次爬虫
python main.py

# 启动 MCP 服务器 (stdio 模式,用于 AI 客户端集成)
python -m mcp_server.server

# 启动 MCP 服务器 (HTTP 模式,用于 Web 访问)
python -m mcp_server.server --http
```

### Docker 部署

```bash
# 使用 docker-compose 启动
cd docker
docker-compose up -d

# 查看日志
docker-compose logs -f

# 停止容器
docker-compose down

# 重启服务
docker-compose restart

# 进入容器管理
docker exec -it trendradar-news python manage.py
```

### GitHub Actions

项目使用 GitHub Actions 自动执行爬虫:

- 工作流文件: `.github/workflows/crawler.yml`
- 执行频率: 每 2 小时的第 17 分钟执行一次 (cron: "17 */2 * * *")
- 手动触发: 在 Actions 页面点击 "Run workflow"

## 项目架构

### 目录结构

```
TrendRadar/
├── main.py                    # 爬虫主程序 (约 4000+ 行)
├── config/
│   ├── config.yaml           # 主配置文件
│   └── frequency_words.txt   # 关键词配置
├── mcp_server/               # MCP 服务器
│   ├── server.py            # FastMCP 服务器入口
│   ├── tools/               # 13 个分析工具
│   │   ├── data_query.py    # 基础数据查询
│   │   ├── analytics.py     # 趋势分析
│   │   ├── search_tools.py  # 智能检索
│   │   ├── config_mgmt.py   # 配置管理
│   │   └── system.py        # 系统管理
│   ├── services/            # 业务服务层
│   └── utils/               # 工具函数
├── docker/                   # Docker 部署
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── manage.py            # 容器管理工具
│   └── entrypoint.sh
└── output/                   # 输出目录
    ├── history/             # 历史数据 (JSON)
    └── html/                # HTML 报告
```

### 核心模块说明

#### 1. 爬虫系统 (main.py)

单文件实现,主要功能:

- **数据抓取**: 从 newsnow API 获取多平台热点数据
- **关键词过滤**: 支持必须词(+)、过滤词(!)、数量限制(@)等高级语法
- **智能排序**: 基于排名权重(60%)、频次权重(30%)、热度权重(10%)的算法
- **推送模式**: 支持 daily(当日汇总)、current(当前榜单)、incremental(增量监控)
- **多渠道推送**: 企业微信、飞书、钉钉、Telegram、邮件、ntfy、Bark、Slack
- **数据持久化**: 保存 JSON 历史数据和生成 HTML 报告

重要函数:
- `load_config()` - 加载并合并环境变量与 YAML 配置
- `filter_news_by_keywords()` - 关键词过滤核心逻辑
- `calculate_comprehensive_score()` - 热点综合评分算法
- `send_notification()` - 统一推送接口

#### 2. MCP 服务器 (mcp_server/)

基于 FastMCP 2.0 框架,提供 13 个分析工具:

**日期解析** (优先调用):
- `resolve_date_range` - 自然语言日期解析

**基础查询**:
- `query_news_by_date` - 按日期查询新闻
- `query_news_by_platform` - 按平台查询新闻
- `query_news_by_topic` - 按主题查询新闻

**智能检索**:
- `smart_search` - 多维度智能搜索
- `find_similar_news` - 相似新闻查找
- `search_related_history` - 历史关联检索

**趋势分析**:
- `analyze_topic_trend` - 话题趋势分析
- `detect_hot_spike` - 爆火检测
- `predict_trend` - 趋势预测
- `analyze_sentiment` - 情感倾向分析

**数据洞察**:
- `compare_platforms` - 跨平台对比
- `generate_summary` - 智能摘要生成

#### 3. Docker 管理 (docker/manage.py)

容器内管理工具,提供交互式菜单:

- 查看定时任务状态
- 修改执行频率
- 手动执行爬虫
- 查看执行日志
- 查看配置信息

## 配置说明

### 主配置文件 (config/config.yaml)

关键配置项:

```yaml
crawler:
  enable_crawler: true        # 是否启用爬虫
  request_interval: 1000      # 请求间隔(毫秒)

report:
  mode: "daily"              # 推送模式: daily|current|incremental
  rank_threshold: 5          # 排名高亮阈值
  sort_by_position_first: false  # 排序优先级
  max_news_per_keyword: 0    # 每个关键词最大显示数量

notification:
  enable_notification: true   # 是否启用通知
  push_window:
    enabled: false           # 推送时间窗口控制
    time_range:
      start: "20:00"
      end: "22:00"
    once_per_day: true       # 每天只推送一次

weight:
  rank_weight: 0.6          # 排名权重
  frequency_weight: 0.3     # 频次权重
  hotness_weight: 0.1       # 热度权重

platforms:                   # 监控平台列表
  - id: "toutiao"
    name: "今日头条"
  # ... 其他平台
```

### 关键词配置 (config/frequency_words.txt)

支持高级语法:

```
# 基础关键词
AI 人工智能 机器学习

# 必须词 (+): 限定范围
AI +苹果 +谷歌

# 过滤词 (!): 排除干扰
特斯拉 !股价

# 数量限制 (@): 控制显示
比亚迪 @5

# 空行分隔: 独立统计
比特币 以太坊

美国大选 特朗普
```

### 环境变量配置

优先级高于 YAML 配置的环境变量:

- `REPORT_MODE` - 推送模式
- `ENABLE_CRAWLER` - 是否启用爬虫
- `ENABLE_NOTIFICATION` - 是否启用通知
- `SORT_BY_POSITION_FIRST` - 排序优先级
- `MAX_NEWS_PER_KEYWORD` - 显示数量限制
- `FEISHU_WEBHOOK_URL`, `TELEGRAM_BOT_TOKEN` 等 - 推送渠道配置

## 数据流程

1. **数据抓取**: main.py 从 newsnow API 获取各平台热点
2. **关键词过滤**: 根据 frequency_words.txt 筛选匹配新闻
3. **智能排序**: 使用综合评分算法重新排序
4. **推送决策**: 根据 mode 和 push_window 决定是否推送
5. **多渠道推送**: 并发推送到配置的各个渠道
6. **数据持久化**: 保存到 output/history/ 和生成 HTML 报告
7. **MCP 分析**: MCP 服务器读取历史数据提供 AI 分析能力

## 部署方式

### 1. GitHub Actions (默认)

- Fork 项目后自动启用
- 配置 GitHub Secrets 添加推送渠道
- 每 2 小时自动执行
- 结果推送到 GitHub Pages

### 2. Docker 部署

- 支持自定义执行频率
- 使用 supercronic 管理定时任务
- 提供交互式管理工具 (manage.py)
- 适合个人服务器或 NAS

### 3. MCP 客户端集成

- Cherry Studio (GUI 配置)
- Claude Desktop
- Cursor
- Cline

## 注意事项

### Git 提交规范

- 不要在 commit 消息中添加署名 (Co-Authored-By)
- GitHub Actions 自动提交格式: "🔄 Auto update at YYYY-MM-DD HH:MM:SS"

### 安全配置

- **切勿**在 config.yaml 中填写 webhook 地址
- 应使用 GitHub Secrets 或环境变量配置敏感信息
- Docker 部署时使用 .env 文件 (已在 .gitignore 中)

### API 使用规范

- 项目使用 newsnow 项目的 API
- GitHub Actions 默认间隔 2 小时,避免对源服务器造成压力
- Docker 部署请合理控制推送频率

### 推送模式选择

- `daily`: 适合企业管理者/普通用户,按时推送当日汇总
- `current`: 适合自媒体人/内容创作者,推送当前榜单
- `incremental`: 适合投资者/交易员,仅推送新增内容

## 版本信息

当前版本: 3.4.1

版本检查机制:
- 从 GitHub master 分支的 version 文件获取最新版本
- 启动时自动检查并提示更新 (可通过 show_version_update 关闭)
