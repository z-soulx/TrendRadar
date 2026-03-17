# TrendRadar GitHub Actions 部署指南

## 一、项目概述

TrendRadar 通过 GitHub Actions 定时运行，自动爬取热搜平台 + RSS 新闻，匹配 OTA 酒店行业关键词，将命中结果推送到企业微信群。

**运行流程：**

```
定时触发(cron) → 爬取 9 个平台热搜 + RSS → 关键词匹配 → 推送企业微信
```

## 二、部署步骤

### Step 1: Fork 仓库

在 GitHub 上 Fork 本仓库到你的账号下，或者将本地代码推送到你的 GitHub 仓库。

```bash
# 如果是新仓库
cd /path/to/TrendRadar
git remote set-url origin https://github.com/YOUR_USERNAME/TrendRadar.git
git push -u origin main
```

### Step 2: 配置 Secrets

进入 GitHub 仓库页面 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

#### 必填 Secret

| Secret 名称 | 值 | 说明 |
|---|---|---|
| `WEWORK_WEBHOOK_URL` | `https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=你的key` | 企业微信群机器人 Webhook |

#### 可选 Secret（按需配置）

| Secret 名称 | 说明 |
|---|---|
| `S3_BUCKET_NAME` | S3 桶名（增量模式必须，见 Step 4） |
| `S3_ACCESS_KEY_ID` | S3 访问密钥 ID |
| `S3_SECRET_ACCESS_KEY` | S3 访问密钥 |
| `S3_ENDPOINT_URL` | S3 端点（如 Cloudflare R2、阿里 OSS） |
| `S3_REGION` | S3 区域 |
| `AI_API_KEY` | AI 分析 API Key（如需 AI 总结功能） |
| `AI_MODEL` | AI 模型名（如 `gpt-4o-mini`） |
| `AI_API_BASE` | AI API 地址 |
| `AI_ANALYSIS_ENABLED` | 设为 `true` 开启 AI 分析 |

### Step 3: 选择运行模式

编辑 `config/config.yaml` 中的 `report.mode`：

#### 模式 A：`current`（推荐新手，无需 S3）

```yaml
report:
  mode: "current"
```

- 每次运行独立，不依赖历史数据
- 推送当前热搜中匹配关键词的内容
- **无需配置 S3**，最简单

#### 模式 B：`incremental`（推荐生产使用，需要 S3）

```yaml
report:
  mode: "incremental"
```

- 只推送**新增**的匹配内容，避免重复推送
- 需要 S3 远程存储来跨 Actions 运行保存数据
- 配置方法见 Step 4

### Step 4: 配置远程存储（incremental 模式必须）

GitHub Actions 每次运行后文件系统会清空。如果使用 `incremental` 模式，必须配置 S3 兼容的远程存储来持久化数据库。

推荐免费方案：**Cloudflare R2**（每月 10GB 免费）

1. 注册 [Cloudflare](https://dash.cloudflare.com/) 账号
2. 进入 R2 → 创建存储桶（如 `trendradar-data`）
3. 创建 API Token（R2 读写权限）
4. 在 GitHub Secrets 中添加：

```
S3_BUCKET_NAME=trendradar-data
S3_ACCESS_KEY_ID=你的access_key
S3_SECRET_ACCESS_KEY=你的secret_key
S3_ENDPOINT_URL=https://你的account_id.r2.cloudflarestorage.com
S3_REGION=auto
```

其他兼容 S3 的存储也可以：阿里云 OSS、腾讯云 COS、MinIO 等。

### Step 5: 调整运行频率

编辑 `.github/workflows/crawler.yml` 中的 cron 表达式：

```yaml
schedule:
  - cron: "33 * * * *"  # 每小时第33分钟运行
```

常用调度方案：

| cron 表达式 | 说明 |
|---|---|
| `"33 * * * *"` | 每小时运行（默认） |
| `"0 0-14 * * *"` | 北京时间 8:00-22:00 每小时 |
| `"0 1,5,9,13 * * *"` | 北京时间 9/13/17/21 点（每天4次） |
| `"0 */2 * * *"` | 每 2 小时运行 |

> 注意：GitHub cron 使用 **UTC 时区**，北京时间 = UTC + 8

### Step 6: 启用 Workflow

1. 进入 GitHub 仓库 → **Actions** 页签
2. 如果看到 "Workflows aren't being run" 提示，点击 **I understand my workflows, go ahead and enable them**
3. 左侧找到 **Get Hot News** → 点击 **Enable workflow**
4. 点击 **Run workflow** 手动触发一次测试

### Step 7: 验证运行

1. 在 **Actions** 页签查看运行日志
2. 检查企业微信群是否收到推送
3. 如果日志显示 `0 条频率词匹配` 且没有推送，说明当前热搜没有酒店行业相关内容，**这是正常的**

## 三、签到续期机制

本项目有 **7 天试用期** 设计：

- 每 7 天需要手动运行一次 **Check In** workflow 来续期
- 如果 7 天没有签到，`Get Hot News` 会自动停止
- 签到方法：Actions → **Check In** → **Run workflow**

> 如需长期无人值守运行，建议部署 Docker 版本或去掉 `crawler.yml` 中的 Check Expiration 步骤。

## 四、文件说明

| 文件 | 作用 | 是否需要修改 |
|---|---|---|
| `config/config.yaml` | 主配置（平台、报告模式、存储等） | 根据需要 |
| `config/frequency_words.txt` | 关键词规则 | 根据监控需求调整 |
| `.github/workflows/crawler.yml` | Actions 定时任务 | 调整运行频率 |
| `.github/workflows/clean-crawler.yml` | 签到续期 | 一般不改 |

## 五、关键词说明

当前配置的监控关键词组（`config/frequency_words.txt`）：

### 单词触发（出现即命中）

| 组名 | 关键词 | 用途 |
|---|---|---|
| 酒店 | 酒店 | 任何酒店相关新闻 |
| 民宿 | 民宿 | 民宿行业动态 |
| OTA平台 | 携程、飞猪、去哪儿、同程旅行 | OTA 平台新闻 |
| 酒店品牌 | 华住、锦江、亚朵、万豪、希尔顿等 | 品牌动态 |
| 准考证/考试 | 准考证、省考、国考、事业编、联考 | 考试订房需求 |
| 机票/航空 | 机票、航司 | 出行交通 |
| 文旅/签证 | 文旅、免签 | 旅游政策 |

### AND 组合（必须同时出现）

公务员+考试、事业单位+考试、教师+招聘、五一+旅游、国庆+旅游、酒店+促销、演唱会+住宿、消费+增长 等 26 个组合。

### 添加新关键词

编辑 `config/frequency_words.txt`，语法：

```text
# 单词匹配（出现即命中）
[组名]
关键词A
关键词B
@最大显示条数

# AND 组合（必须同时出现，每个+词单独一行）
[组名]
+词1
+词2
@最大显示条数

# 正则匹配
[组名]
/模式A|模式B|模式C/
@最大显示条数
```

## 六、故障排查

| 问题 | 原因 | 解决 |
|---|---|---|
| Actions 运行失败 | Secrets 未配置 | 检查 `WEWORK_WEBHOOK_URL` 是否设置 |
| 0 条匹配无推送 | 当前热搜无相关内容 | 正常，等待相关热点出现 |
| incremental 模式每次都推全量 | 未配置 S3 | 配置远程存储或改用 `current` 模式 |
| 企业微信推送失败 | Webhook URL 过期 | 重新创建群机器人获取新 URL |
| workflow 被禁用 | 7 天未签到 | 运行 Check In，然后重新启用 |
| RSS 抓取失败 | 源站不可用 | 检查 RSS URL 是否可访问 |

## 七、本地测试

```bash
# 激活 conda 环境
conda activate trendradar

# 设置环境变量（避免 webhook 写入代码）
export WEWORK_WEBHOOK_URL="https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=你的key"

# 运行
python -m trendradar
```

## 八、监控平台列表

当前配置的 9 个数据源：

| 平台 | ID | 类型 |
|---|---|---|
| 微博热搜 | weibo | 热榜 |
| 百度热搜 | baidu | 热榜 |
| 今日头条 | toutiao | 热榜 |
| 抖音热榜 | douyin | 热榜 |
| 知乎热榜 | zhihu | 热榜 |
| 澎湃新闻 | thepaper | 热榜 |
| 华尔街见闻 | wallstreetcn | 热榜 |
| 财联社 | cls | 热榜 |
| 凤凰网 | ifeng | 热榜 |
| 36氪 | 36kr | RSS |
