# CoughRecognition 服务端设计文档

> 版本：1.0
> 创建时间：2026-01-22
> 最后更新：2026-01-22
> 状态：草案

---

## 一、概述

### 1.1 服务端职责

CoughRecognition 服务端承担以下核心职责：

| 职责 | 说明 |
|------|------|
| 用户管理 | 注册、登录、设备绑定、孩子档案管理 |
| 音频处理 | 接收、验证、预处理音频文件 |
| ML 推理 | 调用 ML 服务进行咳嗽分类 |
| AI 诊疗 | 调用 LLM 生成诊疗建议 |
| 数据存储 | 用户数据、分析记录、音频文件管理 |
| 隐私保护 | 数据加密、访问控制、合规性保障 |

### 1.2 系统架构图

```
                           ┌─────────────────────────────────────────────────────┐
                           │                    客户端层                          │
                           │                  iOS App (Swift)                    │
                           └──────────────────────────┬──────────────────────────┘
                                                      │
                                                      │ HTTPS
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      反向代理层                                              │
│                                   Nginx / Traefik                                           │
│                          (SSL 终止, 负载均衡, 速率限制, CORS)                                │
└─────────────────────────────────────────────────────┬───────────────────────────────────────┘
                                                      │
                           ┌──────────────────────────┴──────────────────────────┐
                           │                                                      │
                           ▼                                                      ▼
┌─────────────────────────────────────────────┐    ┌─────────────────────────────────────────────┐
│             API 网关 (Node.js)               │    │            ML 服务 (Python)                 │
│                                              │    │                                             │
│  ┌────────────────────────────────────────┐ │    │  ┌─────────────────────────────────────────┐│
│  │         Fastify + TypeScript           │ │    │  │           FastAPI + Uvicorn             ││
│  │                                        │ │    │  │                                         ││
│  │  ┌────────────┐  ┌────────────────┐   │ │    │  │  ┌─────────────────────────────────────┐││
│  │  │ 认证中间件  │  │  请求验证       │   │ │    │  │  │        咳嗽分类模型                 │││
│  │  └────────────┘  └────────────────┘   │ │◄───┼──┼─►│        (YAMNet + 自定义分类头)        │││
│  │  ┌────────────┐  ┌────────────────┐   │ │    │  │  └─────────────────────────────────────┘││
│  │  │ 业务逻辑层  │  │  路由控制       │   │ │    │  │                                         ││
│  │  └────────────┘  └────────────────┘   │ │    │  │  ┌─────────────────────────────────────┐││
│  │  ┌────────────┐  ┌────────────────┐   │ │    │  │  │        音频预处理                   │││
│  │  │ 数据访问层  │  │  LLM 客户端     │   │ │    │  │  │        (Mel 频谱图生成)              │││
│  │  └────────────┘  └────────────────┘   │ │    │  │  └─────────────────────────────────────┘││
│  └────────────────────────────────────────┘ │    │  └─────────────────────────────────────────┘│
│                                              │    │                                             │
└───────────────────┬──────────────────────────┘    └─────────────────────────────────────────────┘
                    │
    ┌───────────────┼───────────────┬───────────────────────┐
    │               │               │                       │
    ▼               ▼               ▼                       ▼
┌────────┐    ┌──────────┐    ┌──────────┐           ┌─────────────┐
│PostgreSQL│   │  Redis   │    │ S3/MinIO │           │  LLM API    │
│          │   │          │    │          │           │             │
│ 用户数据  │   │ 会话缓存  │    │ 音频存储  │           │ OpenAI /    │
│ 分析记录  │   │ 速率限制  │    │ (可选)   │           │ Claude API  │
│ 孩子档案  │   │ Token黑名单│   │          │           │             │
└────────┘    └──────────┘    └──────────┘           └─────────────┘
```

### 1.3 服务通信方式

| 服务对 | 协议 | 说明 |
|--------|------|------|
| 客户端 → API 网关 | HTTPS (REST) | JSON / multipart |
| API 网关 → ML 服务 | HTTP (REST) | 内网通信 |
| API 网关 → LLM API | HTTPS | 第三方 API |
| API 网关 → 数据库 | TCP | PostgreSQL 协议 |
| API 网关 → Redis | TCP | Redis 协议 |
| API 网关 → S3 | HTTPS | S3 协议 |

---

## 二、技术栈选型

### 2.1 运行环境

| 组件 | 选型 | 版本 | 说明 |
|------|------|------|------|
| 运行时 | Node.js | 20 LTS | 长期支持版本，性能稳定 |
| 包管理 | pnpm | 8.x | 磁盘空间效率高，依赖管理严格 |
| 语言 | TypeScript | 5.x | 类型安全，开发体验好 |

### 2.2 Web 框架

**选型：Fastify + TypeScript**

| 考量 | Fastify | Express | NestJS |
|------|---------|---------|--------|
| 性能 | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| 生态 | ★★★★☆ | ★★★★★ | ★★★★☆ |
| 类型支持 | ★★★★★ | ★★★☆☆ | ★★★★★ |
| 学习成本 | ★★★★☆ | ★★★★★ | ★★★☆☆ |
| JSON Schema | 内置 | 需插件 | 内置 |
| 启动速度 | 快 | 快 | 较慢 |

**Fastify 核心特性**：
- 高性能（比 Express 快约 2x）
- 内置 JSON Schema 验证
- 插件化架构，按需加载
- 原生 TypeScript 支持
- 优秀的日志集成（Pino）

### 2.3 数据库

#### PostgreSQL（主数据库）

| 用途 | 数据类型 |
|------|----------|
| 用户账户 | 注册信息、认证凭证 |
| 设备管理 | 设备信息、绑定关系 |
| 孩子档案 | 基本信息、健康记录 |
| 分析记录 | 咳嗽分析结果、诊疗建议 |

**版本**：PostgreSQL 16

**扩展**：
- `uuid-ossp`：UUID 生成
- `pgcrypto`：数据加密

#### Redis（缓存/会话）

| 用途 | 数据类型 | TTL |
|------|----------|-----|
| 会话管理 | Refresh Token | 7 天 |
| Token 黑名单 | Access Token ID | 与 Token 有效期一致 |
| 速率限制 | 请求计数 | 1 分钟 |
| 临时数据 | 验证码等 | 5 分钟 |

**版本**：Redis 7.x

### 2.4 对象存储

**选型：S3 兼容存储（MinIO / AWS S3）**

| 场景 | 选择 |
|------|------|
| 自托管 | MinIO |
| 云部署 | AWS S3 / 阿里云 OSS / 腾讯云 COS |

**存储内容**：
- 原始音频文件（可选，根据隐私策略）
- 处理后的音频特征（如需保留）
- 用户上传的其他文件

### 2.5 ML 推理服务

**选型：Python 微服务（FastAPI）**

| 项目 | 规格 |
|------|------|
| 框架 | FastAPI |
| 服务器 | Uvicorn |
| ML 框架 | TensorFlow / PyTorch |
| 音频处理 | librosa |

**架构优势**：
- Node.js API 网关与 Python ML 服务分离
- ML 服务可独立扩展和部署
- 与 ML-Plan.md 中的训练环境一致
- 便于模型热更新

**通信接口**：

```
POST /ml/classify
Content-Type: multipart/form-data

Request:
  - audio: 音频文件
  - age_group: 年龄段

Response:
{
  "coughs": [...],
  "summary": {...},
  "metadata": {...}
}
```

### 2.6 依赖项清单

#### API 网关（Node.js）

```json
{
  "dependencies": {
    "fastify": "^4.26.0",
    "@fastify/cors": "^9.0.0",
    "@fastify/helmet": "^11.1.0",
    "@fastify/multipart": "^8.1.0",
    "@fastify/jwt": "^8.0.0",
    "@fastify/rate-limit": "^9.1.0",
    "@fastify/swagger": "^8.14.0",
    "@fastify/swagger-ui": "^3.0.0",
    "drizzle-orm": "^0.29.0",
    "postgres": "^3.4.0",
    "ioredis": "^5.3.0",
    "@aws-sdk/client-s3": "^3.500.0",
    "zod": "^3.22.0",
    "bcryptjs": "^2.4.3",
    "nanoid": "^5.0.0",
    "pino": "^8.18.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/node": "^20.11.0",
    "vitest": "^1.2.0",
    "drizzle-kit": "^0.20.0",
    "tsx": "^4.7.0",
    "eslint": "^8.56.0",
    "@typescript-eslint/eslint-plugin": "^6.20.0",
    "prettier": "^3.2.0"
  }
}
```

#### ML 服务（Python）

```txt
# requirements.txt
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
python-multipart>=0.0.6
tensorflow>=2.15.0
librosa>=0.10.0
numpy>=1.26.0
pydantic>=2.6.0
```

---

## 三、用户系统设计

### 3.1 数据模型

#### 用户模型 (User)

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone           VARCHAR(20) UNIQUE,          -- 手机号（可选）
    email           VARCHAR(255) UNIQUE,         -- 邮箱（可选）
    password_hash   VARCHAR(255),                -- 密码哈希
    nickname        VARCHAR(50),                 -- 昵称
    avatar_url      VARCHAR(500),                -- 头像 URL
    status          VARCHAR(20) DEFAULT 'active', -- active / suspended / deleted
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    last_login_at   TIMESTAMPTZ,

    CONSTRAINT check_login_method CHECK (phone IS NOT NULL OR email IS NOT NULL)
);

CREATE INDEX idx_users_phone ON users(phone) WHERE phone IS NOT NULL;
CREATE INDEX idx_users_email ON users(email) WHERE email IS NOT NULL;
```

#### 设备模型 (Device)

```sql
CREATE TABLE devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_uuid     VARCHAR(100) NOT NULL,       -- 设备唯一标识
    device_name     VARCHAR(100),                -- 设备名称（如 "iPhone 15 Pro"）
    device_model    VARCHAR(50),                 -- 设备型号
    os_version      VARCHAR(20),                 -- 系统版本
    app_version     VARCHAR(20),                 -- App 版本
    push_token      VARCHAR(500),                -- 推送 Token
    is_active       BOOLEAN DEFAULT true,        -- 是否活跃
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ,

    UNIQUE(user_id, device_uuid)
);

CREATE INDEX idx_devices_user_id ON devices(user_id);
CREATE INDEX idx_devices_device_uuid ON devices(device_uuid);
```

#### 孩子档案模型 (Child)

```sql
CREATE TABLE children (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name            VARCHAR(50) NOT NULL,        -- 孩子姓名
    birth_date      DATE,                        -- 出生日期
    gender          VARCHAR(10),                 -- male / female / other
    avatar_url      VARCHAR(500),                -- 头像 URL
    notes           TEXT,                        -- 备注
    is_archived     BOOLEAN DEFAULT false,       -- 是否归档
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_children_user_id ON children(user_id);
```

#### 分析记录模型 (Analysis)

```sql
CREATE TABLE analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    child_id        UUID REFERENCES children(id) ON DELETE SET NULL,
    device_id       UUID REFERENCES devices(id) ON DELETE SET NULL,

    -- 音频信息
    audio_duration  DECIMAL(10, 3),              -- 音频时长（秒）
    audio_hash      VARCHAR(64),                 -- 音频哈希（去重/完整性）
    recorded_at     TIMESTAMPTZ,                 -- 录制时间
    source          VARCHAR(20),                 -- manual / monitor

    -- 分析结果
    coughs_data     JSONB,                       -- 咳嗽详情 JSON
    summary_data    JSONB,                       -- 汇总数据 JSON
    model_version   VARCHAR(20),                 -- 模型版本

    -- 诊疗建议（可选）
    diagnosis_data  JSONB,                       -- 诊疗建议 JSON

    -- 元数据
    processing_time_ms INTEGER,                  -- 处理耗时
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_analyses_user_id ON analyses(user_id);
CREATE INDEX idx_analyses_child_id ON analyses(child_id);
CREATE INDEX idx_analyses_created_at ON analyses(created_at DESC);
```

### 3.2 认证方案

#### JWT Token 设计

**Access Token**：
- 有效期：15 分钟
- 存储位置：客户端内存
- 载荷：

```json
{
  "sub": "user_uuid",
  "jti": "token_uuid",
  "iat": 1705900000,
  "exp": 1705900900,
  "device_id": "device_uuid"
}
```

**Refresh Token**：
- 有效期：7 天
- 存储位置：客户端安全存储（Keychain）+ Redis
- 载荷：

```json
{
  "sub": "user_uuid",
  "jti": "refresh_token_uuid",
  "iat": 1705900000,
  "exp": 1706504900,
  "device_id": "device_uuid",
  "type": "refresh"
}
```

#### Token 刷新流程

```
┌─────────┐                    ┌────────────┐                    ┌───────┐
│  Client │                    │ API Server │                    │ Redis │
└────┬────┘                    └──────┬─────┘                    └───┬───┘
     │                                │                              │
     │ 1. Access Token 过期            │                              │
     │ POST /auth/refresh             │                              │
     │ { refresh_token }              │                              │
     │ ──────────────────────────────►│                              │
     │                                │                              │
     │                                │ 2. 验证 Refresh Token        │
     │                                │ GET refresh:{jti}            │
     │                                │ ─────────────────────────────►│
     │                                │                              │
     │                                │◄──────────────────────────────│
     │                                │ 3. Token 存在且有效            │
     │                                │                              │
     │                                │ 4. 生成新 Token 对            │
     │                                │ 5. 旧 Refresh Token 加入黑名单│
     │                                │ DEL refresh:{old_jti}        │
     │                                │ SET refresh:{new_jti}        │
     │                                │ ─────────────────────────────►│
     │                                │                              │
     │ 6. 返回新 Token 对              │                              │
     │◄───────────────────────────────│                              │
     │                                │                              │
```

#### Token 撤销机制

```typescript
// Token 黑名单（用于注销、强制下线等场景）
interface TokenBlacklist {
  // Access Token 黑名单：存储 jti，TTL = Token 剩余有效期
  addToBlacklist(jti: string, expiresIn: number): Promise<void>;
  isBlacklisted(jti: string): Promise<boolean>;
}
```

### 3.3 设备管理

#### 设备绑定流程

```
1. 用户登录时：
   - 检查设备是否已绑定
   - 已绑定：更新设备信息（app_version 等）
   - 未绑定：创建新设备记录

2. 设备数量限制：
   - 默认每用户最多 5 个活跃设备
   - 超出时提示用户移除旧设备

3. 设备信息更新：
   - 每次 API 请求更新 last_seen_at
   - App 版本更新时更新 app_version
```

#### 多设备同步

```typescript
// 设备同步事件
enum DeviceSyncEvent {
  CHILD_CREATED = 'child_created',
  CHILD_UPDATED = 'child_updated',
  ANALYSIS_COMPLETED = 'analysis_completed',
  SETTINGS_CHANGED = 'settings_changed'
}

// 通过推送通知触发同步
// 客户端收到通知后主动拉取最新数据
```

### 3.4 用户 API 概要

> 详细 API 规格将在后续补充至 API-Spec.md

#### 认证相关

| 路径 | 方法 | 说明 |
|------|------|------|
| `/auth/register` | POST | 用户注册（手机号/邮箱） |
| `/auth/login` | POST | 用户登录 |
| `/auth/logout` | POST | 用户登出 |
| `/auth/refresh` | POST | 刷新 Token |
| `/auth/password/reset` | POST | 重置密码 |

#### 用户管理

| 路径 | 方法 | 说明 |
|------|------|------|
| `/user/profile` | GET | 获取用户信息 |
| `/user/profile` | PATCH | 更新用户信息 |
| `/user/devices` | GET | 获取设备列表 |
| `/user/devices/:id` | DELETE | 移除设备 |

#### 孩子档案

| 路径 | 方法 | 说明 |
|------|------|------|
| `/children` | GET | 获取孩子列表 |
| `/children` | POST | 添加孩子 |
| `/children/:id` | GET | 获取孩子详情 |
| `/children/:id` | PATCH | 更新孩子信息 |
| `/children/:id` | DELETE | 删除/归档孩子 |

#### 分析记录

| 路径 | 方法 | 说明 |
|------|------|------|
| `/analyses` | GET | 获取分析历史 |
| `/analyses/:id` | GET | 获取分析详情 |

---

## 四、隐私与存储方案

### 4.1 隐私保护原则

| 原则 | 实施措施 |
|------|----------|
| 数据最小化 | 只收集必要数据，音频分析后可不保留原始文件 |
| 目的限定 | 数据仅用于咳嗽分析，不用于其他目的 |
| 存储限制 | 设置数据保留期限，定期清理过期数据 |
| 安全保障 | 传输加密 + 存储加密 |
| 用户控制 | 用户可随时查看、导出、删除自己的数据 |

### 4.2 音频文件处理流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 客户端上传   │ ──► │ 服务端接收   │ ──► │  ML 分析    │ ──► │ 返回结果    │
│ (HTTPS 加密) │     │ (临时存储)   │     │ (内存处理)   │     │ (删除音频)   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

#### 处理策略选项

**方案 A：即时处理，不存储原始音频（推荐）**

```
1. 接收音频文件到内存/临时目录
2. 验证文件格式和大小
3. 调用 ML 服务进行分析
4. 返回分析结果
5. 立即删除临时文件
6. 仅存储分析结果（不含原始音频）
```

**方案 B：保留音频用于模型优化（需用户同意）**

```
1. 接收音频文件
2. 用户明确同意数据用于改进服务
3. 音频加密后存储到 S3
4. 存储时间：最长 90 天
5. 用户可随时撤回同意并删除
```

### 4.3 加密策略

#### 传输加密

| 层级 | 措施 |
|------|------|
| 网络层 | TLS 1.3 |
| 应用层 | HTTPS（强制） |
| 证书 | Let's Encrypt / 商业证书 |
| HSTS | 启用，max-age=31536000 |

#### 存储加密

| 数据类型 | 加密方式 |
|----------|----------|
| 数据库敏感字段 | AES-256-GCM（应用层加密） |
| S3 对象 | SSE-S3 / SSE-KMS |
| Redis 数据 | TLS 连接（传输加密） |

#### 敏感字段加密示例

```typescript
// 需要加密的字段
interface EncryptedFields {
  // 用户表
  phone: string;        // 手机号
  email: string;        // 邮箱

  // 孩子表
  name: string;         // 姓名
  birthDate: Date;      // 出生日期
}

// 加密工具
class FieldEncryption {
  private readonly key: Buffer;
  private readonly algorithm = 'aes-256-gcm';

  encrypt(plaintext: string): EncryptedValue {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
    // ... 加密逻辑
  }

  decrypt(encrypted: EncryptedValue): string {
    // ... 解密逻辑
  }
}
```

### 4.4 存储架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                           存储层架构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│  │   PostgreSQL    │  │     Redis       │  │    S3/MinIO     │      │
│  │                 │  │                 │  │                 │      │
│  │  • 用户数据      │  │  • 会话缓存     │  │  • 音频文件      │      │
│  │  • 孩子档案      │  │  • Token 黑名单 │  │    (可选)       │      │
│  │  • 分析记录      │  │  • 速率限制     │  │  • 用户头像      │      │
│  │  • 诊疗建议      │  │  • 临时数据     │  │                 │      │
│  │                 │  │                 │  │                 │      │
│  │  加密：字段级    │  │  加密：传输层   │  │  加密：对象级    │      │
│  │  备份：每日增量  │  │  持久化：RDB    │  │  备份：跨区域    │      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.5 数据保留策略

| 数据类型 | 保留期限 | 删除方式 |
|----------|----------|----------|
| 用户账户 | 永久（直到用户删除） | 用户请求删除 |
| 设备信息 | 永久（直到用户删除） | 用户移除设备 |
| 孩子档案 | 永久（直到用户删除） | 用户删除 |
| 分析记录 | 2 年 | 自动清理 |
| 原始音频 | 不保留 / 90 天（如用户同意） | 即时删除 / 定时清理 |
| 日志数据 | 30 天 | 自动轮转 |

#### 数据清理作业

```typescript
// 定时任务：每日凌晨 3:00 执行
@Cron('0 3 * * *')
async cleanExpiredData() {
  // 1. 清理过期分析记录
  await this.analysisRepo.deleteOlderThan(2, 'years');

  // 2. 清理过期音频文件
  await this.storageService.deleteExpiredAudio(90, 'days');

  // 3. 清理过期日志
  await this.logService.rotateOldLogs(30, 'days');
}
```

### 4.6 GDPR / 隐私合规

#### 用户权利实现

| 权利 | 实现方式 |
|------|----------|
| 知情权 | 隐私政策、首次使用说明 |
| 访问权 | 数据导出功能 |
| 更正权 | 资料编辑功能 |
| 删除权 | 账户删除功能（级联删除所有数据） |
| 可携权 | 数据导出为标准格式（JSON） |
| 反对权 | 可选择不参与数据改进计划 |

#### 数据导出格式

```json
{
  "export_date": "2026-01-22T10:00:00Z",
  "user": {
    "id": "...",
    "created_at": "...",
    "profile": {...}
  },
  "children": [...],
  "analyses": [...],
  "devices": [...]
}
```

#### 账户删除流程

```
1. 用户请求删除账户
2. 验证用户身份（密码/验证码）
3. 标记账户为"待删除"状态
4. 30 天宽限期（可撤销）
5. 30 天后执行物理删除：
   - 删除所有用户数据
   - 删除所有关联音频文件
   - 删除所有分析记录
   - 删除 Redis 中的会话数据
6. 删除完成，保留最小审计日志
```

---

## 五、项目结构

### 5.1 目录结构

```
server/
├── src/
│   ├── app.ts                    # 应用入口
│   ├── config/                   # 配置管理
│   │   ├── index.ts              # 配置导出
│   │   ├── database.ts           # 数据库配置
│   │   ├── redis.ts              # Redis 配置
│   │   └── env.ts                # 环境变量验证
│   │
│   ├── modules/                  # 业务模块
│   │   ├── auth/                 # 认证模块
│   │   │   ├── auth.routes.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.schema.ts    # Zod 验证
│   │   │   └── auth.types.ts
│   │   │
│   │   ├── user/                 # 用户模块
│   │   │   ├── user.routes.ts
│   │   │   ├── user.controller.ts
│   │   │   ├── user.service.ts
│   │   │   └── user.schema.ts
│   │   │
│   │   ├── child/                # 孩子档案模块
│   │   │   └── ...
│   │   │
│   │   ├── analysis/             # 分析模块
│   │   │   ├── analysis.routes.ts
│   │   │   ├── analysis.controller.ts
│   │   │   ├── analysis.service.ts
│   │   │   └── analysis.schema.ts
│   │   │
│   │   └── diagnosis/            # 诊疗建议模块
│   │       └── ...
│   │
│   ├── db/                       # 数据库层
│   │   ├── schema/               # Drizzle Schema
│   │   │   ├── users.ts
│   │   │   ├── devices.ts
│   │   │   ├── children.ts
│   │   │   └── analyses.ts
│   │   ├── migrations/           # 数据库迁移
│   │   └── index.ts              # 数据库客户端
│   │
│   ├── services/                 # 外部服务客户端
│   │   ├── ml-client.ts          # ML 服务客户端
│   │   ├── llm-client.ts         # LLM API 客户端
│   │   ├── storage.ts            # S3 存储客户端
│   │   └── redis.ts              # Redis 客户端
│   │
│   ├── plugins/                  # Fastify 插件
│   │   ├── auth.ts               # 认证插件
│   │   ├── error-handler.ts      # 错误处理
│   │   └── request-logger.ts     # 请求日志
│   │
│   ├── utils/                    # 工具函数
│   │   ├── crypto.ts             # 加密工具
│   │   ├── validation.ts         # 验证工具
│   │   └── errors.ts             # 错误定义
│   │
│   └── types/                    # 类型定义
│       ├── fastify.d.ts          # Fastify 类型扩展
│       └── common.ts             # 通用类型
│
├── tests/                        # 测试文件
│   ├── unit/                     # 单元测试
│   ├── integration/              # 集成测试
│   └── fixtures/                 # 测试数据
│
├── scripts/                      # 脚本
│   ├── migrate.ts                # 数据库迁移
│   └── seed.ts                   # 测试数据填充
│
├── docker/                       # Docker 相关
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── docker-compose.dev.yml
│
├── k8s/                          # Kubernetes 配置
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── secrets.yaml
│
├── drizzle.config.ts             # Drizzle 配置
├── tsconfig.json                 # TypeScript 配置
├── vitest.config.ts              # Vitest 配置
├── .eslintrc.js                  # ESLint 配置
├── .prettierrc                   # Prettier 配置
├── .env.example                  # 环境变量示例
└── package.json
```

### 5.2 分层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Routes (路由层)                             │
│                    定义路由、请求验证、响应格式                        │
├─────────────────────────────────────────────────────────────────────┤
│                       Controllers (控制层)                           │
│                    处理 HTTP 请求、调用服务层                         │
├─────────────────────────────────────────────────────────────────────┤
│                        Services (服务层)                             │
│                    业务逻辑、事务管理、服务编排                        │
├─────────────────────────────────────────────────────────────────────┤
│                      Repositories (数据层)                           │
│                    数据访问、ORM 操作、缓存管理                        │
├─────────────────────────────────────────────────────────────────────┤
│                       Database / Cache                               │
│                    PostgreSQL / Redis / S3                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 模块职责

| 模块 | 职责 | 依赖服务 |
|------|------|----------|
| auth | 认证、授权、Token 管理 | Redis |
| user | 用户管理、设备管理 | PostgreSQL |
| child | 孩子档案 CRUD | PostgreSQL |
| analysis | 咳嗽分析、结果存储 | PostgreSQL, ML 服务 |
| diagnosis | AI 诊疗建议生成 | PostgreSQL, LLM API |

### 5.4 配置管理

```typescript
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  // 应用配置
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
  HOST: z.string().default('0.0.0.0'),

  // 数据库
  DATABASE_URL: z.string().url(),

  // Redis
  REDIS_URL: z.string().url(),

  // JWT
  JWT_SECRET: z.string().min(32),
  JWT_ACCESS_EXPIRES_IN: z.string().default('15m'),
  JWT_REFRESH_EXPIRES_IN: z.string().default('7d'),

  // S3
  S3_ENDPOINT: z.string().url().optional(),
  S3_ACCESS_KEY: z.string().optional(),
  S3_SECRET_KEY: z.string().optional(),
  S3_BUCKET: z.string().optional(),

  // ML 服务
  ML_SERVICE_URL: z.string().url(),

  // LLM API
  LLM_API_KEY: z.string(),
  LLM_API_URL: z.string().url().optional(),

  // 加密
  ENCRYPTION_KEY: z.string().min(32),
});

export const env = envSchema.parse(process.env);
```

---

## 六、部署方案

### 6.1 直接启动（开发 / 小规模部署）

#### 系统要求

| 组件 | 最低配置 | 推荐配置 |
|------|----------|----------|
| CPU | 2 核 | 4 核 |
| 内存 | 4 GB | 8 GB |
| 存储 | 20 GB SSD | 50 GB SSD |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

#### 安装步骤

```bash
# 1. 安装 Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. 安装 pnpm
npm install -g pnpm

# 3. 安装 PostgreSQL
sudo apt-get install -y postgresql postgresql-contrib

# 4. 安装 Redis
sudo apt-get install -y redis-server

# 5. 克隆代码并安装依赖
git clone <repo-url> cough-recognition-server
cd cough-recognition-server
pnpm install

# 6. 配置环境变量
cp .env.example .env
# 编辑 .env 填写配置

# 7. 运行数据库迁移
pnpm db:migrate

# 8. 启动服务
pnpm start
```

#### 使用 PM2 管理进程

```bash
# 安装 PM2
npm install -g pm2

# 启动服务
pm2 start ecosystem.config.js

# ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'cough-api',
      script: './dist/app.js',
      instances: 'max',        // 根据 CPU 核心数
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
      },
    },
  ],
};

# 设置开机自启
pm2 startup
pm2 save
```

### 6.2 Docker 部署

#### Dockerfile

```dockerfile
# docker/Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# 安装 pnpm
RUN npm install -g pnpm

# 复制依赖文件
COPY package.json pnpm-lock.yaml ./

# 安装依赖
RUN pnpm install --frozen-lockfile

# 复制源码并构建
COPY . .
RUN pnpm build

# 生产镜像
FROM node:20-alpine AS runner

WORKDIR /app

# 安装 pnpm
RUN npm install -g pnpm

# 只复制生产依赖
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod

# 复制构建产物
COPY --from=builder /app/dist ./dist

# 非 root 用户运行
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 app
USER app

EXPOSE 3000

CMD ["node", "dist/app.js"]
```

#### docker-compose.yml

```yaml
# docker/docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:password@postgres:5432/cough_recognition
      - REDIS_URL=redis://redis:6379
      - ML_SERVICE_URL=http://ml-service:8000
    depends_on:
      - postgres
      - redis
      - ml-service
    restart: unless-stopped

  ml-service:
    build:
      context: ../ml-service
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - MODEL_PATH=/models
    volumes:
      - ./models:/models:ro
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=cough_recognition
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - api
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

#### 启动命令

```bash
# 构建并启动
docker-compose -f docker/docker-compose.yml up -d --build

# 查看日志
docker-compose -f docker/docker-compose.yml logs -f api

# 停止服务
docker-compose -f docker/docker-compose.yml down
```

### 6.3 Kubernetes 部署

#### Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cough-recognition-api
  labels:
    app: cough-recognition-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cough-recognition-api
  template:
    metadata:
      labels:
        app: cough-recognition-api
    spec:
      containers:
        - name: api
          image: your-registry/cough-recognition-api:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: cough-recognition-config
            - secretRef:
                name: cough-recognition-secrets
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cough-recognition-ml
  labels:
    app: cough-recognition-ml
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cough-recognition-ml
  template:
    metadata:
      labels:
        app: cough-recognition-ml
    spec:
      containers:
        - name: ml
          image: your-registry/cough-recognition-ml:latest
          ports:
            - containerPort: 8000
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
```

#### Service

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: cough-recognition-api
spec:
  selector:
    app: cough-recognition-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: cough-recognition-ml
spec:
  selector:
    app: cough-recognition-ml
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP
```

#### Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cough-recognition-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
    - hosts:
        - api.coughrecognition.com
      secretName: cough-recognition-tls
  rules:
    - host: api.coughrecognition.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: cough-recognition-api
                port:
                  number: 80
```

#### ConfigMap 和 Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cough-recognition-config
data:
  NODE_ENV: "production"
  ML_SERVICE_URL: "http://cough-recognition-ml:8000"
  JWT_ACCESS_EXPIRES_IN: "15m"
  JWT_REFRESH_EXPIRES_IN: "7d"

---
# k8s/secrets.yaml (实际部署时使用 kubectl create secret)
apiVersion: v1
kind: Secret
metadata:
  name: cough-recognition-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://..."
  REDIS_URL: "redis://..."
  JWT_SECRET: "..."
  ENCRYPTION_KEY: "..."
  LLM_API_KEY: "..."
```

### 6.4 反向代理配置（Nginx）

```nginx
# nginx.conf

# 上游服务器组
upstream api_servers {
    least_conn;
    server api:3000 weight=1;
    keepalive 32;
}

upstream ml_servers {
    least_conn;
    server ml-service:8000 weight=1;
}

# 速率限制区域
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=upload_limit:10m rate=2r/s;

server {
    listen 80;
    server_name api.coughrecognition.com;

    # 强制 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.coughrecognition.com;

    # SSL 配置
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # 安全头
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";

    # 文件上传大小限制
    client_max_body_size 15M;

    # 日志
    access_log /var/log/nginx/api_access.log;
    error_log /var/log/nginx/api_error.log;

    # 健康检查
    location /health {
        proxy_pass http://api_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # 文件上传接口（更严格的速率限制）
    location ~ ^/v1/cough/analyze {
        limit_req zone=upload_limit burst=5 nodelay;

        proxy_pass http://api_servers;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";

        # 上传超时设置
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }

    # API 请求
    location /v1/ {
        limit_req zone=api_limit burst=20 nodelay;

        proxy_pass http://api_servers;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    # 默认返回 404
    location / {
        return 404;
    }
}
```

### 6.5 环境变量配置

```bash
# .env.example

# 应用配置
NODE_ENV=production
PORT=3000
HOST=0.0.0.0

# 数据库
DATABASE_URL=postgresql://user:password@localhost:5432/cough_recognition

# Redis
REDIS_URL=redis://localhost:6379

# JWT 配置
JWT_SECRET=your-very-secure-jwt-secret-at-least-32-chars
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# S3 / MinIO（可选）
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=cough-recognition

# ML 服务
ML_SERVICE_URL=http://localhost:8000

# LLM API
LLM_API_KEY=your-llm-api-key
LLM_API_URL=https://api.openai.com/v1

# 加密密钥（32+ 字符）
ENCRYPTION_KEY=your-encryption-key-at-least-32-characters

# 速率限制
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW_MS=60000

# 日志级别
LOG_LEVEL=info
```

---

## 七、开发工具链

### 7.1 包管理

**选型：pnpm**

| 特性 | pnpm | npm | yarn |
|------|------|-----|------|
| 磁盘空间 | ★★★★★ | ★★☆☆☆ | ★★★☆☆ |
| 安装速度 | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| 依赖隔离 | ★★★★★ | ★★★☆☆ | ★★★☆☆ |
| Monorepo | ★★★★★ | ★★☆☆☆ | ★★★★☆ |

### 7.2 代码规范

#### ESLint 配置

```javascript
// .eslintrc.js
module.exports = {
  root: true,
  parser: '@typescript-eslint/parser',
  parserOptions: {
    project: './tsconfig.json',
    ecmaVersion: 2022,
    sourceType: 'module',
  },
  plugins: ['@typescript-eslint'],
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:@typescript-eslint/recommended-requiring-type-checking',
    'prettier',
  ],
  rules: {
    '@typescript-eslint/explicit-function-return-type': 'error',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    '@typescript-eslint/no-floating-promises': 'error',
    'no-console': 'warn',
  },
};
```

#### Prettier 配置

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

### 7.3 测试框架

**选型：Vitest**

| 特性 | Vitest | Jest |
|------|--------|------|
| 速度 | ★★★★★ | ★★★☆☆ |
| ESM 支持 | ★★★★★ | ★★★☆☆ |
| TypeScript | ★★★★★ | ★★★★☆ |
| Watch 模式 | ★★★★★ | ★★★★☆ |

#### 配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['tests/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules', 'tests'],
    },
    setupFiles: ['tests/setup.ts'],
  },
});
```

#### 测试示例

```typescript
// tests/unit/auth.service.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { AuthService } from '../../src/modules/auth/auth.service';

describe('AuthService', () => {
  let authService: AuthService;

  beforeEach(() => {
    authService = new AuthService(/* mock dependencies */);
  });

  describe('login', () => {
    it('should return tokens for valid credentials', async () => {
      const result = await authService.login({
        email: 'test@example.com',
        password: 'password123',
      });

      expect(result).toHaveProperty('accessToken');
      expect(result).toHaveProperty('refreshToken');
    });

    it('should throw error for invalid credentials', async () => {
      await expect(
        authService.login({
          email: 'test@example.com',
          password: 'wrong-password',
        })
      ).rejects.toThrow('Invalid credentials');
    });
  });
});
```

### 7.4 API 文档

**选型：OpenAPI/Swagger（Fastify 插件）**

#### 配置

```typescript
// src/plugins/swagger.ts
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUi from '@fastify/swagger-ui';
import { FastifyInstance } from 'fastify';

export async function registerSwagger(app: FastifyInstance): Promise<void> {
  await app.register(fastifySwagger, {
    openapi: {
      info: {
        title: 'CoughRecognition API',
        description: 'API for cough analysis and diagnosis suggestions',
        version: '1.0.0',
      },
      servers: [
        { url: 'https://api.coughrecognition.com/v1' },
      ],
      components: {
        securitySchemes: {
          bearerAuth: {
            type: 'http',
            scheme: 'bearer',
            bearerFormat: 'JWT',
          },
        },
      },
    },
  });

  await app.register(fastifySwaggerUi, {
    routePrefix: '/docs',
    uiConfig: {
      docExpansion: 'list',
      deepLinking: true,
    },
  });
}
```

#### 路由文档示例

```typescript
// src/modules/analysis/analysis.routes.ts
export const analysisRoutes: FastifyPluginAsync = async (app) => {
  app.post<{ Body: AnalysisRequest }>(
    '/cough/analyze',
    {
      schema: {
        description: 'Analyze cough audio and return classification results',
        tags: ['analysis'],
        security: [{ bearerAuth: [] }],
        consumes: ['multipart/form-data'],
        body: {
          type: 'object',
          required: ['audio', 'age_group'],
          properties: {
            audio: { type: 'string', format: 'binary' },
            age_group: { type: 'string', enum: ['infant', 'toddler', 'preschool', 'school_age'] },
            recorded_at: { type: 'string', format: 'date-time' },
          },
        },
        response: {
          200: AnalysisResponseSchema,
          400: ErrorResponseSchema,
          401: ErrorResponseSchema,
        },
      },
    },
    analysisController.analyze
  );
};
```

### 7.5 开发脚本

```json
// package.json scripts
{
  "scripts": {
    "dev": "tsx watch src/app.ts",
    "build": "tsc",
    "start": "node dist/app.js",
    "lint": "eslint src --ext .ts",
    "lint:fix": "eslint src --ext .ts --fix",
    "format": "prettier --write \"src/**/*.ts\"",
    "test": "vitest",
    "test:coverage": "vitest --coverage",
    "db:generate": "drizzle-kit generate:pg",
    "db:migrate": "drizzle-kit push:pg",
    "db:studio": "drizzle-kit studio"
  }
}
```

---

## 八、附录

### A. 与 API-Spec.md 对照

| API-Spec.md 规格 | 服务端实现 |
|------------------|------------|
| Base URL: `/v1` | Fastify 路由前缀 |
| Bearer Token 认证 | @fastify/jwt 插件 |
| multipart/form-data | @fastify/multipart 插件 |
| JSON 响应格式 | Fastify 内置 JSON 序列化 |
| 错误码定义 | 自定义错误类 + 错误处理插件 |
| 咳嗽分析 API | analysis 模块 |
| AI 诊疗建议 API | diagnosis 模块 |

### B. 与 ML-Plan.md 对照

| ML-Plan.md 规格 | 服务端集成方式 |
|-----------------|---------------|
| 咳嗽分类模型（云端） | Python ML 微服务 |
| 模型输入（Mel 频谱图） | ML 服务内部处理 |
| 分类输出（5 类咳嗽） | API 返回格式与 API-Spec 一致 |
| 模型版本管理 | metadata.model_version 字段 |

### C. 相关文档

- [PRD.md](../PRD.md) - 产品需求文档
- [API-Spec.md](API-Spec.md) - API 接口规格
- [ML-Plan.md](ML-Plan.md) - ML 方案文档
- [设计方案.md](../设计方案.md) - 技术设计方案

---

*文档结束*
