# 🔌 API 集成技术文档

## 1. AI OpenAI-Compatible 集成

后端 AI 层统一通过 OpenAI 兼容的 `POST /chat/completions` 协议调用模型，支持三种提供商：

| Provider | `AI_PROVIDER` | Base URL | 默认模型 |
|----------|---------------|----------|----------|
| OpenRouter | `openrouter` | `https://openrouter.ai/api/v1` | `deepseek/deepseek-v3.2` |
| MiniMax Token Plan | `minimax` | `https://api.minimaxi.com/v1` | `MiniMax-M2.7` |
| 通用 OpenAI 兼容服务 | `openai-compatible` | 用户自定义 | 用户自定义 |

### 1.1 环境变量

**OpenRouter：**

```env
AI_PROVIDER=openrouter
OPENROUTER_API_KEY=sk-or-v1-xxx
OPENROUTER_MODEL=deepseek/deepseek-v3.2
```

**MiniMax Token Plan：**

```env
AI_PROVIDER=minimax
MINIMAX_API_KEY=your_minimax_token_plan_api_key
MINIMAX_BASE_URL=https://api.minimaxi.com/v1
MINIMAX_MODEL=MiniMax-M2.7
```

MiniMax Token Plan 注意事项：
- Token Plan API Key 是专属 key，与按量计费 API Key 不可互换。
- M2.7 / M2.7-highspeed 按请求额度计算，每 5 小时滚动重置。

**通用 OpenAI 兼容接口：**

```env
AI_PROVIDER=openai-compatible
OPENAI_COMPATIBLE_BASE_URL=https://your-provider.example.com/v1
OPENAI_COMPATIBLE_API_KEY=your_api_key
OPENAI_COMPATIBLE_MODEL=your-model-name
```

### 1.2 Chat Completion 调用

```bash
curl "$AI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $AI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "messages": [
      { "role": "system", "content": "你是一个热点分析专家。" },
      { "role": "user", "content": "请分析这条热点内容..." }
    ],
    "temperature": 0.2,
    "max_tokens": 500,
    "stream": false
  }'
```

### 1.3 响应格式

```json
{
  "id": "chatcmpl-xxxxxxxxxxxxxxxxx",
  "object": "chat.completion",
  "created": 1677652288,
  "model": "openai/gpt-4",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "{\"isReal\": true, \"relevance\": 85, \"importance\": \"high\", \"summary\": \"...\"}"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 15,
    "total_tokens": 25
  }
}
```

---

## 2. Twitter API (twitterapi.io) 集成

### 2.1 认证

```typescript
const TWITTER_API_BASE = 'https://api.twitterapi.io';
const TWITTER_API_KEY = process.env.TWITTER_API_KEY;

const headers = {
  'X-API-Key': TWITTER_API_KEY,
  'Content-Type': 'application/json'
};
```

### 2.2 高级搜索 API

**Endpoint:** `GET /twitter/tweet/advanced_search`

**参数:**
- `query` (string, required): 搜索查询，支持高级语法
- `queryType` (enum, required): `Latest` 或 `Top`
- `cursor` (string, optional): 分页游标

**查询语法示例:**
```
"AI" OR "GPT" lang:en since:2024-01-01
from:OpenAI OR from:Anthropic
#AINews min_faves:100
```

**请求示例:**

```typescript
async function searchTwitter(query: string, cursor?: string) {
  const params = new URLSearchParams({
    query: query,
    queryType: 'Latest'
  });
  
  if (cursor) {
    params.append('cursor', cursor);
  }

  const response = await fetch(
    `${TWITTER_API_BASE}/twitter/tweet/advanced_search?${params}`,
    { headers }
  );

  return response.json();
}
```

**响应格式:**

```json
{
  "tweets": [
    {
      "type": "tweet",
      "id": "1234567890",
      "url": "https://twitter.com/user/status/1234567890",
      "text": "Breaking: OpenAI announces GPT-5...",
      "source": "Twitter Web App",
      "retweetCount": 1500,
      "replyCount": 300,
      "likeCount": 5000,
      "quoteCount": 200,
      "viewCount": 150000,
      "createdAt": "2024-01-15T10:30:00Z",
      "lang": "en",
      "author": {
        "userName": "techreporter",
        "name": "Tech Reporter",
        "isBlueVerified": true,
        "followers": 50000,
        "profilePicture": "https://..."
      },
      "entities": {
        "hashtags": [{ "text": "AI" }],
        "urls": [{ "expanded_url": "https://..." }]
      }
    }
  ],
  "has_next_page": true,
  "next_cursor": "xxxx"
}
```

### 2.3 获取热门趋势

**Endpoint:** `GET /twitter/trends`

```typescript
async function getTrends(woeid: number = 1) { // 1 = Worldwide
  const response = await fetch(
    `${TWITTER_API_BASE}/twitter/trends?woeid=${woeid}`,
    { headers }
  );
  return response.json();
}
```

---

## 3. 网页搜索爬虫

### 3.1 Bing 搜索爬虫

```typescript
import axios from 'axios';
import * as cheerio from 'cheerio';

const USER_AGENTS = [
  'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...',
  'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36...',
];

async function searchBing(query: string): Promise<SearchResult[]> {
  const userAgent = USER_AGENTS[Math.floor(Math.random() * USER_AGENTS.length)];
  
  const response = await axios.get('https://www.bing.com/search', {
    params: { q: query },
    headers: { 'User-Agent': userAgent }
  });

  const $ = cheerio.load(response.data);
  const results: SearchResult[] = [];

  $('li.b_algo').each((_, element) => {
    const title = $(element).find('h2 a').text();
    const url = $(element).find('h2 a').attr('href');
    const snippet = $(element).find('.b_caption p').text();
    
    if (title && url) {
      results.push({ title, url, snippet, source: 'bing' });
    }
  });

  return results;
}
```

### 3.2 频率控制

```typescript
class RateLimiter {
  private queue: (() => Promise<void>)[] = [];
  private processing = false;
  private lastRequestTime = 0;
  private minInterval = 5000; // 5 秒间隔

  async add<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try {
          const result = await fn();
          resolve(result);
        } catch (error) {
          reject(error);
        }
      });
      this.process();
    });
  }

  private async process() {
    if (this.processing || this.queue.length === 0) return;
    
    this.processing = true;
    
    while (this.queue.length > 0) {
      const elapsed = Date.now() - this.lastRequestTime;
      if (elapsed < this.minInterval) {
        await new Promise(r => setTimeout(r, this.minInterval - elapsed));
      }
      
      const task = this.queue.shift();
      if (task) {
        this.lastRequestTime = Date.now();
        await task();
      }
    }
    
    this.processing = false;
  }
}
```

---

## 4. Prisma + SQLite 配置

### 4.1 Schema 定义

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model Keyword {
  id        String    @id @default(uuid())
  text      String    @unique
  category  String?
  isActive  Boolean   @default(true)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  hotspots  Hotspot[]
}

model Hotspot {
  id          String   @id @default(uuid())
  title       String
  content     String
  url         String
  source      String   // twitter, bing, google
  sourceId    String?  // 原始推文ID等
  isReal      Boolean  @default(true)
  relevance   Int      @default(0)
  importance  String   @default("low")
  summary     String?
  viewCount   Int?
  likeCount   Int?
  retweetCount Int?
  publishedAt DateTime?
  createdAt   DateTime @default(now())
  keywordId   String?
  keyword     Keyword? @relation(fields: [keywordId], references: [id])
  
  @@unique([url, source])
}

model Notification {
  id        String   @id @default(uuid())
  type      String   // hotspot, alert
  title     String
  content   String
  isRead    Boolean  @default(false)
  hotspotId String?
  createdAt DateTime @default(now())
}

model Setting {
  id    String @id @default(uuid())
  key   String @unique
  value String
}
```

### 4.2 迁移命令

```bash
# 初始化数据库
npx prisma migrate dev --name init

# 生成 Prisma Client
npx prisma generate
```

### 4.3 环境变量

```env
DATABASE_URL="file:./dev.db"
```

---

## 5. Express + WebSocket 配置

### 5.1 服务器配置

```typescript
import express from 'express';
import { createServer } from 'http';
import { Server } from 'socket.io';
import cors from 'cors';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: {
    origin: process.env.CLIENT_URL || 'http://localhost:5173',
    methods: ['GET', 'POST']
  }
});

app.use(cors());
app.use(express.json());

// WebSocket 连接
io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);
  
  socket.on('subscribe', (keywords: string[]) => {
    keywords.forEach(kw => socket.join(`keyword:${kw}`));
  });
  
  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
  });
});

// 发送热点通知
function notifyNewHotspot(hotspot: Hotspot) {
  io.to(`keyword:${hotspot.keyword?.text}`).emit('hotspot:new', hotspot);
  io.emit('notification', {
    type: 'hotspot',
    title: '发现新热点',
    content: hotspot.title
  });
}

export { app, httpServer, io, notifyNewHotspot };
```

### 5.2 路由结构

```typescript
// routes/keywords.ts
import { Router } from 'express';
import { prisma } from '../db';

const router = Router();

router.get('/', async (req, res) => {
  const keywords = await prisma.keyword.findMany({
    orderBy: { createdAt: 'desc' }
  });
  res.json(keywords);
});

router.post('/', async (req, res) => {
  const { text, category } = req.body;
  const keyword = await prisma.keyword.create({
    data: { text, category }
  });
  res.status(201).json(keyword);
});

router.delete('/:id', async (req, res) => {
  await prisma.keyword.delete({
    where: { id: req.params.id }
  });
  res.status(204).send();
});

export default router;
```

---

## 6. 定时任务配置

```typescript
import cron from 'node-cron';

// 每 30 分钟执行一次热点检查
cron.schedule('*/30 * * * *', async () => {
  console.log('Running hotspot check...');
  await checkHotspots();
});

async function checkHotspots() {
  const keywords = await prisma.keyword.findMany({
    where: { isActive: true }
  });

  for (const keyword of keywords) {
    // 1. 从 Twitter 搜索
    const tweets = await searchTwitter(keyword.text);
    
    // 2. 从 Bing 搜索
    const webResults = await searchBing(keyword.text);
    
    // 3. AI 分析
    for (const item of [...tweets, ...webResults]) {
      const analysis = await analyzeHotspot(item.content);
      
      if (analysis.isReal && analysis.relevance > 60) {
        // 4. 保存并通知
        const hotspot = await saveHotspot(item, analysis, keyword);
        notifyNewHotspot(hotspot);
      }
    }
  }
}
```

---

## 7. 邮件通知配置

```typescript
import nodemailer from 'nodemailer';

const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,
  port: parseInt(process.env.SMTP_PORT || '587'),
  secure: false,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS
  }
});

async function sendEmailNotification(hotspot: Hotspot) {
  await transporter.sendMail({
    from: process.env.SMTP_USER,
    to: process.env.NOTIFY_EMAIL,
    subject: `🔥 新热点: ${hotspot.title}`,
    html: `
      <h2>${hotspot.title}</h2>
      <p>${hotspot.summary}</p>
      <p><strong>重要程度:</strong> ${hotspot.importance}</p>
      <p><strong>相关性:</strong> ${hotspot.relevance}%</p>
      <p><a href="${hotspot.url}">查看原文</a></p>
    `
  });
}
```
