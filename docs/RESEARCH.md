# 技术调研文档

> 日期: 2026-06-24
> 版本: v0.2
> 状态: 调研中

---

## 1. 类似产品调研

### 1.1 国内产品

#### 小日常 (Daily)
- 平台: iOS / Android
- 特点: 简洁习惯打卡，每日记录
- 优点: UI 清新，操作简单
- 不足: 没有积分和兑换机制，无 AI

#### 宝宝生活记录
- 平台: 微信小程序
- 特点: 记录宝宝日常作息
- 优点: 微信生态，无需安装
- 不足: 偏记录，缺少激励和智能分析

### 1.2 国外产品

#### Homey
- 平台: iOS / Android
- 特点: 家庭任务管理 + 零花钱挂钩
- 优点: 任务-奖励闭环，多孩支持
- 不足: 英文界面，付费，无 AI

#### Habitica
- 平台: Web / iOS / Android
- 特点: RPG 游戏化习惯追踪
- 优点: 游戏元素丰富，成就系统完善
- 不足: 对小孩来说太复杂，无 IM 集成

#### iRewardChart
- 平台: iOS
- 特点: 可视化奖励图表
- 优点: 直观，适合低龄儿童
- 不足: 功能单一，无 AI

#### Greenlight
- 平台: iOS / Android
- 特点: 儿童理财 + 任务管理
- 优点: 积分与真实金钱挂钩
- 不足: 仅美国市场

### 1.3 市场空白
目前市面上 **没有** 同时具备以下特点的产品：
1. AI 智能分析和建议
2. 飞书/微信自然语言交互
3. 积分永不过期 + 扣分机制
4. 多渠道统一接入

---

## 2. 技术方案（已确定）

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────┐
│                    用户接入层                          │
│  飞书 Bot │ 微信 Bot │ Flutter App │ Web 管理台        │
└────┬─────────┴────┬──────┴─────┬──────────┴────┬─────┘
     │              │            │               │
     ▼              ▼            ▼               ▼
┌──────────────────────────────────────────────────────┐
│                   API Gateway                         │
│            (认证 / 限流 / 路由)                        │
└───────────────────────┬──────────────────────────────┘
                        │
     ┌──────────────────┼──────────────────┐
     ▼                  ▼                  ▼
┌──────────┐   ┌──────────────┐   ┌──────────────┐
│ 业务服务  │   │  AI Agent    │   │  消息推送     │
│ 积分/行为 │   │  分析/建议    │   │  提醒/报告    │
│ 礼品/孩子 │   │  对话/意图    │   │              │
└────┬─────┘   └──────┬───────┘   └──────────────┘
     │                │
     ▼                ▼
┌──────────────────────────────────────────────────────┐
│                    数据层                              │
│  PostgreSQL │ Redis │ 向量数据库(可选)                  │
└──────────────────────────────────────────────────────┘
```

### 2.2 前端：Flutter

| 项目 | 说明 |
|------|------|
| 框架 | Flutter 3.x + Dart |
| 状态管理 | Riverpod / Provider |
| 网络请求 | Dio |
| 本地存储 | Hive（离线缓存） |
| 图表 | fl_chart |
| 目标平台 | Android + iOS |

### 2.3 后端：Node.js 或 Python

#### 方案 A：Node.js + NestJS（推荐）
| 项目 | 说明 |
|------|------|
| 框架 | NestJS (TypeScript) |
| 数据库 | PostgreSQL + Prisma ORM |
| 缓存 | Redis |
| 消息队列 | Bull (Redis-based) |
| 优势 | 与灵犀伴学技术栈一致，类型安全 |

#### 方案 B：Python + FastAPI
| 项目 | 说明 |
|------|------|
| 框架 | FastAPI |
| 数据库 | PostgreSQL + SQLAlchemy |
| 缓存 | Redis |
| 消息队列 | Celery |
| 优势 | AI/ML 生态更丰富 |

### 2.4 AI Agent 方案

#### 2.4.1 模型选择

| 方案 | 模型 | 优势 | 劣势 |
|------|------|------|------|
| A | OpenAI GPT-4o | 能力强，稳定 | 需要翻墙，费用 |
| B | Claude API | 长上下文，安全 | 需要翻墙 |
| C | DeepSeek | 国内直连，便宜 | 能力稍弱 |
| D | 本地部署 (Qwen) | 完全可控 | 需要 GPU |
| **推荐** | **DeepSeek + 备用 OpenAI** | 国内直连为主，高质量备用 | |

#### 2.4.2 Agent 架构

```
用户消息
    │
    ▼
┌─────────────┐
│  意图识别    │  ← LLM 分类：打卡/查询/兑换/咨询
└──────┬──────┘
       │
  ┌────┼────┬────────┐
  ▼    ▼    ▼        ▼
打卡  查询  兑换    咨询
  │    │    │        │
  ▼    ▼    ▼        ▼
执行  查DB  执行   AI对话
操作  返回  操作   生成建议
  │    │    │        │
  └────┼────┴────────┘
       ▼
   生成回复
```

#### 2.4.3 核心 Prompt 设计

**行为分析 Prompt：**
```
你是儿童教育专家 AI 助手。根据以下孩子的行为数据，生成分析报告和建议。

孩子：{child_name}，年龄：{age}
时间范围：{date_range}

行为数据：
{behavior_data}

请分析：
1. 整体表现（正面肯定）
2. 亮点行为（具体表扬）
3. 需要改进的方面（温和指出）
4. 给家长的具体建议（可操作）
5. 给孩子的鼓励话语
```

**意图识别 Prompt：**
```
分析用户消息的意图，分类为以下之一：
- RECORD: 记录行为（打卡/加分/扣分）
- QUERY: 查询积分/表现
- REDEEM: 兑换礼品
- ADVICE: 咨询建议
- CHAT: 闲聊

用户消息：{message}
当前孩子：{child_name}

返回 JSON：
{
  "intent": "RECORD",
  "child": "小明",
  "action": "add",
  "behavior": "课外阅读",
  "points": 3
}
```

### 2.5 飞书对接

#### 2.5.1 接入方式
- 飞书开放平台创建企业自建应用
- 开启机器人能力
- 配置消息回调 URL（Webhook）
- 接收/发送消息通过飞书 API

#### 2.5.2 消息格式
```json
// 接收到的消息
{
  "event": {
    "message": {
      "message_id": "...",
      "chat_id": "...",
      "message_type": "text",
      "content": "{\"text\":\"小明今天按时起床了\"}"
    }
  }
}

// 发送回复
{
  "msg_type": "interactive",
  "card": {
    "elements": [
      {"tag": "markdown", "content": "✅ 已记录：小明 起床洗漱 +2\n当前积分：**85**"}
    ]
  }
}
```

### 2.6 微信对接

#### 2.6.1 接入方式
- **方案 A**: 微信公众号（服务号）
  - 需要认证
  - 支持模板消息推送
  - 用户需关注
- **方案 B**: 企业微信
  - 更灵活
  - 支持群聊
  - 推荐用于家庭场景
- **方案 C**: 个人微信（非官方）
  - 风险高，不推荐

#### 2.6.2 推荐方案
**企业微信 + 微信公众号双通道**
- 企业微信：家长日常使用，支持群聊
- 公众号：轻量查询，推送报告

### 2.7 数据库设计（初步）

```sql
-- 家庭
CREATE TABLE families (
  id UUID PRIMARY KEY,
  name VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 孩子
CREATE TABLE children (
  id UUID PRIMARY KEY,
  family_id UUID REFERENCES families(id),
  name VARCHAR(50),
  avatar VARCHAR(255),
  birth_date DATE,
  total_points INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 行为模板
CREATE TABLE behavior_templates (
  id UUID PRIMARY KEY,
  family_id UUID REFERENCES families(id),
  name VARCHAR(100),        -- "早上起床洗漱"
  icon VARCHAR(50),         -- "🌅"
  category VARCHAR(50),     -- "生活" / "学习" / "品德"
  points INTEGER,           -- 正数加分，负数扣分
  is_daily BOOLEAN,         -- 是否每日重复
  time_start TIME,          -- 可选时间段
  time_end TIME,
  is_system BOOLEAN DEFAULT FALSE
);

-- 积分记录（流水）
CREATE TABLE point_records (
  id UUID PRIMARY KEY,
  child_id UUID REFERENCES children(id),
  behavior_id UUID REFERENCES behavior_templates(id),
  points INTEGER,           -- 本次积分变动
  balance_after INTEGER,    -- 变动后余额
  note TEXT,                -- 备注
  recorded_by VARCHAR(50),  -- "parent" / "ai" / "child"
  source VARCHAR(20),       -- "app" / "feishu" / "wechat"
  created_at TIMESTAMP DEFAULT NOW()
);

-- 礼品
CREATE TABLE gifts (
  id UUID PRIMARY KEY,
  family_id UUID REFERENCES families(id),
  name VARCHAR(100),
  icon VARCHAR(50),
  category VARCHAR(50),
  points_cost INTEGER,
  is_active BOOLEAN DEFAULT TRUE
);

-- 兑换记录
CREATE TABLE redemption_records (
  id UUID PRIMARY KEY,
  child_id UUID REFERENCES children(id),
  gift_id UUID REFERENCES gifts(id),
  points_spent INTEGER,
  status VARCHAR(20) DEFAULT 'pending', -- pending / approved / completed
  approved_by VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 成就
CREATE TABLE achievements (
  id UUID PRIMARY KEY,
  child_id UUID REFERENCES children(id),
  achievement_type VARCHAR(50),
  unlocked_at TIMESTAMP DEFAULT NOW()
);

-- AI 对话历史
CREATE TABLE ai_conversations (
  id UUID PRIMARY KEY,
  child_id UUID REFERENCES children(id),
  role VARCHAR(20),         -- "user" / "assistant"
  content TEXT,
  source VARCHAR(20),       -- "feishu" / "wechat" / "app"
  created_at TIMESTAMP DEFAULT NOW()
);

-- 用户渠道绑定
CREATE TABLE user_channels (
  id UUID PRIMARY KEY,
  family_id UUID REFERENCES families(id),
  channel_type VARCHAR(20), -- "feishu" / "wechat"
  channel_user_id VARCHAR(100),
  display_name VARCHAR(100),
  is_parent BOOLEAN DEFAULT TRUE
);
```

---

## 3. 开发计划

### 3.1 第一阶段：后端 + 飞书 Bot（2-3 周）
1. 搭建后端项目骨架
2. 数据库设计和建表
3. 实现核心业务 API（行为/积分/礼品）
4. 飞书 Bot 对接
5. 基础 AI 对话（意图识别 + 积分操作）

### 3.2 第二阶段：AI 增强 + 微信（2-3 周）
1. AI 行为分析引擎
2. AI 建议生成
3. 微信 Bot 对接
4. 定时报告推送

### 3.3 第三阶段：Flutter App（3-4 周）
1. App UI 设计和开发
2. 对接后端 API
3. 数据统计图表
4. 成就系统
5. 礼品商城

---

## 4. 待调研

- [ ] 飞书企业应用审核流程和时间
- [ ] 微信服务号认证要求
- [ ] DeepSeek API 的速率限制和费用
- [ ] AI Prompt 效果测试
- [ ] 数据库性能预估（多孩并发）

---

## 变更记录

| 日期 | 版本 | 变更内容 |
|------|------|----------|
| 2026-06-24 | v0.1 | 初始调研文档 |
| 2026-06-24 | v0.2 | 确定技术方案：Flutter + 后端 + AI Agent + 飞书/微信 |
