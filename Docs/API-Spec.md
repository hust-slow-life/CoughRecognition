# CoughRecognition API 接口规格文档

> 版本：1.0
> 创建时间：2026-01-22
> 最后更新：2026-01-22
> 状态：草案

---

## 一、概述

本文档定义 CoughRecognition 云端服务的 API 接口规格，包括咳嗽分析 API 和 AI 诊疗建议 API。

### 1.1 基础信息

| 项目 | 内容 |
|------|------|
| Base URL | `https://api.coughrecognition.com/v1` |
| 协议 | HTTPS |
| 认证方式 | Bearer Token |
| 数据格式 | JSON（请求/响应）、multipart/form-data（文件上传） |
| 字符编码 | UTF-8 |

### 1.2 通用请求头

```http
Authorization: Bearer <access_token>
Content-Type: application/json
Accept: application/json
X-Client-Version: 1.0.0
X-Device-ID: <device_uuid>
```

### 1.3 通用响应格式

#### 成功响应
```json
{
  "success": true,
  "data": { ... },
  "request_id": "req_abc123"
}
```

#### 错误响应
```json
{
  "success": false,
  "error": {
    "code": "INVALID_AUDIO_FORMAT",
    "message": "不支持的音频格式，请使用 WAV 或 M4A 格式",
    "details": { ... }
  },
  "request_id": "req_abc123"
}
```

### 1.4 错误码定义

| 错误码 | HTTP 状态码 | 说明 |
|--------|-------------|------|
| UNAUTHORIZED | 401 | 未授权或 Token 过期 |
| INVALID_REQUEST | 400 | 请求参数错误 |
| INVALID_AUDIO_FORMAT | 400 | 音频格式不支持 |
| AUDIO_TOO_SHORT | 400 | 音频时长过短（<1秒） |
| AUDIO_TOO_LONG | 400 | 音频时长过长（>120秒） |
| PROCESSING_FAILED | 500 | 服务器处理失败 |
| RATE_LIMITED | 429 | 请求频率超限 |
| SERVICE_UNAVAILABLE | 503 | 服务暂时不可用 |

---

## 二、咳嗽分析 API

### 2.1 接口定义

**功能**：接收音频文件，返回咳嗽检测和分类结果

| 项目 | 内容 |
|------|------|
| 路径 | `POST /cough/analyze` |
| Content-Type | `multipart/form-data` |
| 响应时间 | <30 秒（建议） |

### 2.2 请求参数

#### 请求体（multipart/form-data）

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| audio | file | 是 | 音频文件 |
| age_group | string | 是 | 孩子年龄段 |
| recorded_at | string | 否 | 录制时间（ISO 8601） |
| duration | number | 否 | 音频时长（秒） |
| source | string | 否 | 录音来源 |
| device_info | object | 否 | 设备信息 |

#### 参数详细说明

**audio（音频文件）**
| 属性 | 要求 |
|------|------|
| 格式 | WAV, M4A, CAF |
| 采样率 | 16kHz - 48kHz |
| 声道 | 单声道或立体声 |
| 时长 | 1 秒 - 120 秒 |
| 大小 | 最大 10MB |

**age_group（年龄段）**
| 值 | 说明 |
|------|------|
| `infant` | 婴儿（0-1岁） |
| `toddler` | 幼儿（1-3岁） |
| `preschool` | 学龄前（3-6岁） |
| `school_age` | 学龄（6-18岁） |

**source（录音来源）**
| 值 | 说明 |
|------|------|
| `manual` | 手动录制 |
| `monitor` | 后台监控自动录制 |

**device_info（设备信息）**
```json
{
  "model": "iPhone 15 Pro",
  "os_version": "iOS 17.2",
  "app_version": "1.0.0"
}
```

### 2.3 请求示例

```bash
curl -X POST "https://api.coughrecognition.com/v1/cough/analyze" \
  -H "Authorization: Bearer <access_token>" \
  -F "audio=@recording.m4a" \
  -F "age_group=toddler" \
  -F "recorded_at=2026-01-22T10:30:00+08:00" \
  -F "duration=8.5" \
  -F "source=manual"
```

### 2.4 响应格式

#### 成功响应

```json
{
  "success": true,
  "data": {
    "analysis_id": "ana_20260122_abc123",
    "audio_duration": 8.5,
    "coughs": [
      {
        "index": 1,
        "timestamp": 1.2,
        "duration": 0.8,
        "type": "dry",
        "type_label": "干咳",
        "confidence": 0.92,
        "features": {
          "intensity": "moderate",
          "frequency_peak": 1200
        }
      },
      {
        "index": 2,
        "timestamp": 3.5,
        "duration": 1.2,
        "type": "wet",
        "type_label": "湿咳",
        "confidence": 0.87,
        "features": {
          "intensity": "strong",
          "frequency_peak": 800,
          "has_phlegm_sound": true
        }
      },
      {
        "index": 3,
        "timestamp": 6.8,
        "duration": 0.6,
        "type": "dry",
        "type_label": "干咳",
        "confidence": 0.89,
        "features": {
          "intensity": "mild",
          "frequency_peak": 1100
        }
      }
    ],
    "summary": {
      "total_coughs": 3,
      "cough_rate": 21.2,
      "dominant_type": "dry",
      "dominant_type_label": "干咳",
      "type_distribution": {
        "dry": 2,
        "wet": 1
      },
      "average_confidence": 0.89,
      "alerts": []
    },
    "metadata": {
      "model_version": "1.0.0",
      "processed_at": "2026-01-22T10:30:05+08:00",
      "processing_time_ms": 2350
    }
  },
  "request_id": "req_20260122_xyz789"
}
```

### 2.5 响应字段说明

#### coughs（咳嗽列表）

| 字段 | 类型 | 说明 |
|------|------|------|
| index | number | 咳嗽序号（从1开始） |
| timestamp | number | 咳嗽开始时间（秒） |
| duration | number | 咳嗽持续时间（秒） |
| type | string | 咳嗽类型（枚举） |
| type_label | string | 咳嗽类型中文名称 |
| confidence | number | 置信度（0-1） |
| features | object | 声学特征（可选） |

**咳嗽类型（type）枚举**

| 值 | 中文名 | 说明 |
|------|--------|------|
| `dry` | 干咳 | 无痰或少痰的咳嗽 |
| `wet` | 湿咳 | 有痰的咳嗽 |
| `barking` | 犬吠样咳 | 类似狗叫的咳嗽声 |
| `wheezing` | 喘息性咳嗽 | 伴有喘息音 |
| `paroxysmal` | 痉挛性咳嗽 | 阵发性连续咳嗽 |
| `unknown` | 未知 | 无法明确分类 |

**声学特征（features）**

| 字段 | 类型 | 说明 |
|------|------|------|
| intensity | string | 强度：mild/moderate/strong |
| frequency_peak | number | 主频率峰值（Hz） |
| has_phlegm_sound | boolean | 是否有痰音 |

#### summary（整体统计）

| 字段 | 类型 | 说明 |
|------|------|------|
| total_coughs | number | 咳嗽总次数 |
| cough_rate | number | 咳嗽频率（次/分钟） |
| dominant_type | string | 主要咳嗽类型 |
| dominant_type_label | string | 主要咳嗽类型中文名 |
| type_distribution | object | 各类型分布统计 |
| average_confidence | number | 平均置信度 |
| alerts | array | 特殊警报列表 |

**alerts（警报类型）**

| 值 | 说明 | 建议操作 |
|------|------|----------|
| `high_frequency` | 咳嗽频率过高（>30次/分钟） | 建议就医 |
| `barking_detected` | 检测到犬吠样咳 | 警惕急性喉炎 |
| `paroxysmal_detected` | 检测到痉挛性咳嗽 | 警惕百日咳 |
| `wheezing_detected` | 检测到喘息音 | 警惕哮喘 |

### 2.6 错误响应示例

```json
{
  "success": false,
  "error": {
    "code": "AUDIO_TOO_SHORT",
    "message": "音频时长过短，请提供至少 1 秒的录音",
    "details": {
      "provided_duration": 0.5,
      "minimum_duration": 1.0
    }
  },
  "request_id": "req_20260122_err456"
}
```

---

## 三、AI 诊疗建议 API

### 3.1 接口定义

**功能**：基于咳嗽分析结果和病程信息，生成 AI 诊疗建议

| 项目 | 内容 |
|------|------|
| 路径 | `POST /diagnosis/suggest` |
| Content-Type | `application/json` |
| 响应时间 | <60 秒 |

### 3.2 请求参数

```json
{
  "analysis_id": "ana_20260122_abc123",
  "cough_analysis": { ... },
  "age_group": "toddler",
  "age_months": 24,
  "medical_history": {
    "symptoms": ["fever", "runny_nose", "sore_throat"],
    "temperature": 38.2,
    "temperature_unit": "celsius",
    "medications": [
      {
        "name": "布洛芬",
        "dosage": "5ml",
        "frequency": "每6小时"
      }
    ],
    "illness_days": 3,
    "cough_pattern": {
      "worse_at_night": true,
      "worse_after_activity": false,
      "triggered_by": ["cold_air"]
    },
    "additional_notes": "昨晚咳嗽明显加重，凌晨咳醒两次"
  },
  "history_analyses": [
    {
      "analysis_id": "ana_20260121_def456",
      "recorded_at": "2026-01-21T22:00:00+08:00",
      "total_coughs": 15,
      "dominant_type": "dry"
    }
  ]
}
```

### 3.3 请求参数说明

#### 顶层参数

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| analysis_id | string | 否 | 关联的咳嗽分析 ID |
| cough_analysis | object | 是 | 咳嗽分析结果（来自分析 API） |
| age_group | string | 是 | 年龄段 |
| age_months | number | 否 | 精确月龄 |
| medical_history | object | 是 | 病程信息 |
| history_analyses | array | 否 | 历史分析记录（用于趋势分析） |

#### medical_history（病程信息）

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| symptoms | array | 是 | 症状列表 |
| temperature | number | 否 | 体温数值 |
| temperature_unit | string | 否 | 温度单位：celsius/fahrenheit |
| medications | array | 否 | 用药列表 |
| illness_days | number | 是 | 病程天数 |
| cough_pattern | object | 否 | 咳嗽模式 |
| additional_notes | string | 否 | 其他备注 |

**症状（symptoms）枚举**

| 值 | 中文名 |
|------|--------|
| `fever` | 发烧 |
| `runny_nose` | 流鼻涕 |
| `stuffy_nose` | 鼻塞 |
| `sore_throat` | 咽喉痛 |
| `headache` | 头痛 |
| `fatigue` | 疲劳 |
| `loss_of_appetite` | 食欲不振 |
| `vomiting` | 呕吐 |
| `diarrhea` | 腹泻 |
| `rash` | 皮疹 |
| `difficulty_breathing` | 呼吸困难 |
| `chest_pain` | 胸痛 |
| `ear_pain` | 耳痛 |

#### cough_pattern（咳嗽模式）

| 参数名 | 类型 | 说明 |
|--------|------|------|
| worse_at_night | boolean | 夜间加重 |
| worse_after_activity | boolean | 活动后加重 |
| worse_in_morning | boolean | 晨起加重 |
| triggered_by | array | 诱因列表 |

**咳嗽诱因（triggered_by）**

| 值 | 说明 |
|------|------|
| `cold_air` | 冷空气 |
| `exercise` | 运动 |
| `eating` | 进食 |
| `lying_down` | 躺下 |
| `crying` | 哭闹 |
| `dust` | 灰尘 |
| `smoke` | 烟雾 |

### 3.4 响应格式

```json
{
  "success": true,
  "data": {
    "suggestion_id": "sug_20260122_ghi789",
    "advice": {
      "summary": "根据咳嗽分析和症状，孩子可能是普通感冒引起的上呼吸道感染，目前以干咳为主，病程第3天属于正常范围。",
      "detailed_analysis": "从录音分析来看，孩子的咳嗽以干咳为主（占67%），咳嗽频率约21次/分钟，属于中等程度。结合发烧（38.2°C）、流鼻涕、咽喉痛等症状，以及病程3天的时间线，符合普通病毒性感冒的典型表现。\n\n夜间咳嗽加重是常见现象，可能与平躺时鼻涕倒流刺激咽喉有关。",
      "care_suggestions": [
        "保持室内空气湿润，可使用加湿器",
        "让孩子多喝温水，保持咽喉湿润",
        "睡觉时可适当垫高枕头，缓解鼻涕倒流",
        "继续按医嘱使用布洛芬控制发烧",
        "注意观察咳嗽是否有痰音变化"
      ],
      "warning_signs": [
        "体温持续超过39°C或发烧超过3天",
        "出现呼吸急促或呼吸困难",
        "咳嗽性质突然改变（如出现犬吠样咳）",
        "精神状态明显变差、拒食",
        "嘴唇发紫"
      ]
    },
    "recommendation": {
      "should_see_doctor": false,
      "urgency": "low",
      "urgency_label": "暂可居家观察",
      "reason": "症状在普通感冒正常范围内，暂无需立即就医",
      "follow_up_days": 2
    },
    "possible_conditions": [
      {
        "name": "普通感冒（上呼吸道感染）",
        "probability": "high",
        "probability_label": "可能性高",
        "description": "病毒感染引起，通常7-10天自愈"
      },
      {
        "name": "急性支气管炎",
        "probability": "low",
        "probability_label": "可能性低",
        "description": "如咳嗽持续加重或出现痰咳需考虑"
      }
    ],
    "disclaimer": {
      "text": "以上建议仅供参考，不能替代医生的专业诊断。如有疑虑或症状加重，请及时就医。",
      "version": "1.0",
      "generated_at": "2026-01-22T10:30:30+08:00"
    },
    "metadata": {
      "model_version": "gpt-4-medical-1.0",
      "processed_at": "2026-01-22T10:30:30+08:00",
      "processing_time_ms": 3500
    }
  },
  "request_id": "req_20260122_jkl012"
}
```

### 3.5 响应字段说明

#### advice（诊疗建议）

| 字段 | 类型 | 说明 |
|------|------|------|
| summary | string | 简短总结（1-2句） |
| detailed_analysis | string | 详细分析（可包含换行） |
| care_suggestions | array | 护理建议列表 |
| warning_signs | array | 警示症状列表 |

#### recommendation（就医建议）

| 字段 | 类型 | 说明 |
|------|------|------|
| should_see_doctor | boolean | 是否建议就医 |
| urgency | string | 紧急程度 |
| urgency_label | string | 紧急程度中文说明 |
| reason | string | 建议原因 |
| follow_up_days | number | 建议复查天数（如适用） |

**urgency（紧急程度）枚举**

| 值 | 中文名 | 说明 |
|------|--------|------|
| `emergency` | 立即就医 | 需要紧急处理 |
| `high` | 尽快就医 | 当天或次日就医 |
| `medium` | 建议就医 | 2-3天内就医 |
| `low` | 可观察 | 暂可居家观察 |

#### possible_conditions（可能疾病）

| 字段 | 类型 | 说明 |
|------|------|------|
| name | string | 疾病名称 |
| probability | string | 可能性：high/medium/low |
| probability_label | string | 可能性中文说明 |
| description | string | 简要说明 |

### 3.6 特殊场景响应

#### 需要紧急就医的情况

```json
{
  "success": true,
  "data": {
    "suggestion_id": "sug_20260122_urgent",
    "advice": {
      "summary": "检测到犬吠样咳嗽，需要警惕急性喉炎（假性哮吼），建议尽快就医。",
      "detailed_analysis": "从录音中检测到明显的犬吠样咳嗽特征，这种咳嗽声音类似狗叫，通常提示喉部存在肿胀或炎症。结合孩子2岁的年龄段，这是急性喉炎的典型表现。\n\n急性喉炎可能导致呼吸道狭窄，在夜间尤其需要警惕。",
      "care_suggestions": [
        "保持镇定，安抚孩子情绪",
        "让孩子呼吸湿润空气（可在浴室开热水制造蒸汽）",
        "让孩子保持直立姿势",
        "尽快前往医院就诊"
      ],
      "warning_signs": [
        "呼吸时发出尖锐声音（喉喘鸣）",
        "呼吸困难、胸部凹陷",
        "嘴唇或指甲发紫",
        "无法说话或吞咽",
        "意识模糊"
      ]
    },
    "recommendation": {
      "should_see_doctor": true,
      "urgency": "high",
      "urgency_label": "尽快就医",
      "reason": "检测到犬吠样咳嗽，需排除急性喉炎",
      "follow_up_days": null
    },
    "possible_conditions": [
      {
        "name": "急性喉炎（假性哮吼）",
        "probability": "high",
        "probability_label": "可能性高",
        "description": "喉部病毒感染导致肿胀，多发于6个月-3岁幼儿"
      }
    ],
    "disclaimer": { ... }
  }
}
```

---

## 四、Prompt 设计

### 4.1 系统提示词（System Prompt）

```
你是一位专业的儿童呼吸道健康顾问AI助手，帮助家长理解孩子的咳嗽症状并提供护理建议。

## 你的角色
- 你是辅助健康顾问，不是医生
- 你的建议仅供参考，不能替代专业医疗诊断
- 你需要帮助家长了解孩子的症状，并指导他们何时应该就医

## 你拥有的能力
- 分析咳嗽类型和频率的临床意义
- 结合症状评估可能的疾病方向
- 提供家庭护理建议
- 判断就医紧迫性

## 你的限制
- 不能做出明确的医学诊断
- 不能开具处方或推荐具体药物（除非用户已在使用）
- 不能替代医生的面诊

## 回答原则
1. **安全第一**：宁可建议就医，不要遗漏危险信号
2. **年龄适配**：充分考虑不同年龄段的特殊性
   - 婴儿（0-1岁）：任何异常都需要更谨慎对待
   - 幼儿（1-3岁）：急性喉炎、毛细支气管炎高发期
   - 学龄前（3-6岁）：哮喘、过敏需要关注
   - 学龄儿童：症状描述更可靠
3. **通俗易懂**：使用家长能理解的语言
4. **实用建议**：给出可操作的护理指导
5. **情绪支持**：理解家长的焦虑，给予适当安慰

## 危险信号（必须建议立即就医）
- 呼吸困难或呼吸急促
- 嘴唇或指甲发紫（紫绀）
- 意识改变、极度嗜睡
- 拒绝进食/饮水超过24小时
- 高烧不退（婴儿>38°C，幼儿>39°C持续）
- 犬吠样咳嗽（提示急性喉炎）
- 痉挛性咳嗽后出现"鸡鸣样"吸气声（提示百日咳）

## 输出格式要求
请按照以下JSON结构输出，确保内容专业且易于理解：

{
  "advice": {
    "summary": "一句话总结（<50字）",
    "detailed_analysis": "详细分析（200-400字）",
    "care_suggestions": ["建议1", "建议2", ...],
    "warning_signs": ["警示1", "警示2", ...]
  },
  "recommendation": {
    "should_see_doctor": true/false,
    "urgency": "emergency/high/medium/low",
    "urgency_label": "中文说明",
    "reason": "建议原因"
  },
  "possible_conditions": [
    {
      "name": "疾病名称",
      "probability": "high/medium/low",
      "probability_label": "中文说明",
      "description": "简要说明"
    }
  ]
}
```

### 4.2 用户提示词模板（User Prompt Template）

```
请根据以下信息，为家长提供咳嗽分析和护理建议：

## 孩子信息
- 年龄段：{{age_group_label}}（{{age_months}}个月）

## 咳嗽分析结果
- 分析时间：{{analysis_time}}
- 录音时长：{{audio_duration}}秒
- 检测到咳嗽：{{total_coughs}}次
- 咳嗽频率：{{cough_rate}}次/分钟
- 主要类型：{{dominant_type_label}}
- 类型分布：{{type_distribution_text}}
- 平均置信度：{{average_confidence}}
{{#if alerts}}
- 特殊警报：{{alerts_text}}
{{/if}}

## 病程信息
- 病程天数：第{{illness_days}}天
- 当前症状：{{symptoms_text}}
{{#if temperature}}
- 体温：{{temperature}}°C
{{/if}}
{{#if medications}}
- 目前用药：{{medications_text}}
{{/if}}
{{#if cough_pattern}}
- 咳嗽特点：
  {{#if cough_pattern.worse_at_night}}- 夜间加重{{/if}}
  {{#if cough_pattern.worse_in_morning}}- 晨起加重{{/if}}
  {{#if cough_pattern.worse_after_activity}}- 活动后加重{{/if}}
  {{#if cough_pattern.triggered_by}}- 诱因：{{triggered_by_text}}{{/if}}
{{/if}}
{{#if additional_notes}}
- 其他说明：{{additional_notes}}
{{/if}}

{{#if history_analyses}}
## 历史咳嗽记录
{{#each history_analyses}}
- {{recorded_at}}: {{total_coughs}}次咳嗽，主要为{{dominant_type_label}}
{{/each}}
{{/if}}

请基于以上信息，提供：
1. 咳嗽情况分析
2. 可能的疾病方向（仅供参考）
3. 家庭护理建议
4. 是否需要就医及紧急程度
5. 需要警惕的危险信号

请以JSON格式输出。
```

### 4.3 Prompt 变量说明

| 变量名 | 类型 | 说明 | 示例值 |
|--------|------|------|--------|
| age_group_label | string | 年龄段中文名 | "幼儿" |
| age_months | number | 月龄 | 24 |
| analysis_time | string | 分析时间 | "2026-01-22 10:30" |
| audio_duration | number | 录音时长 | 8.5 |
| total_coughs | number | 咳嗽次数 | 3 |
| cough_rate | number | 咳嗽频率 | 21.2 |
| dominant_type_label | string | 主要咳嗽类型 | "干咳" |
| type_distribution_text | string | 类型分布 | "干咳2次，湿咳1次" |
| average_confidence | number | 平均置信度 | 0.89 |
| alerts_text | string | 警报文本 | "检测到喘息音" |
| illness_days | number | 病程天数 | 3 |
| symptoms_text | string | 症状列表 | "发烧、流鼻涕、咽喉痛" |
| temperature | number | 体温 | 38.2 |
| medications_text | string | 用药情况 | "布洛芬 5ml 每6小时" |
| additional_notes | string | 补充说明 | "昨晚咳嗽加重" |

### 4.4 模板渲染示例

**输入数据：**
```json
{
  "age_group_label": "幼儿",
  "age_months": 24,
  "analysis_time": "2026-01-22 10:30",
  "audio_duration": 8.5,
  "total_coughs": 3,
  "cough_rate": 21.2,
  "dominant_type_label": "干咳",
  "type_distribution_text": "干咳2次，湿咳1次",
  "average_confidence": 0.89,
  "illness_days": 3,
  "symptoms_text": "发烧、流鼻涕、咽喉痛",
  "temperature": 38.2,
  "medications_text": "布洛芬 5ml 每6小时",
  "cough_pattern": {
    "worse_at_night": true
  },
  "additional_notes": "昨晚咳嗽明显加重，凌晨咳醒两次"
}
```

**渲染后的 Prompt：**
```
请根据以下信息，为家长提供咳嗽分析和护理建议：

## 孩子信息
- 年龄段：幼儿（24个月）

## 咳嗽分析结果
- 分析时间：2026-01-22 10:30
- 录音时长：8.5秒
- 检测到咳嗽：3次
- 咳嗽频率：21.2次/分钟
- 主要类型：干咳
- 类型分布：干咳2次，湿咳1次
- 平均置信度：0.89

## 病程信息
- 病程天数：第3天
- 当前症状：发烧、流鼻涕、咽喉痛
- 体温：38.2°C
- 目前用药：布洛芬 5ml 每6小时
- 咳嗽特点：
  - 夜间加重
- 其他说明：昨晚咳嗽明显加重，凌晨咳醒两次

请基于以上信息，提供：
1. 咳嗽情况分析
2. 可能的疾病方向（仅供参考）
3. 家庭护理建议
4. 是否需要就医及紧急程度
5. 需要警惕的危险信号

请以JSON格式输出。
```

---

## 五、接口调用流程

### 5.1 典型使用流程

```
┌─────────────┐
│   iOS App   │
└──────┬──────┘
       │
       │ 1. 用户录制咳嗽音频
       │
       ▼
┌─────────────────────────────────────────┐
│ POST /cough/analyze                      │
│ - 上传音频文件                           │
│ - 年龄段                                 │
└───────────────────┬─────────────────────┘
                    │
                    │ 返回咳嗽分析结果
                    │
                    ▼
┌─────────────────────────────────────────┐
│ 用户补充病程信息                         │
│ - 症状选择                               │
│ - 体温记录                               │
│ - 用药情况                               │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ POST /diagnosis/suggest                  │
│ - 咳嗽分析结果                           │
│ - 病程信息                               │
└───────────────────┬─────────────────────┘
                    │
                    │ 返回 AI 诊疗建议
                    │
                    ▼
┌─────────────────────────────────────────┐
│ App 展示结果                             │
│ - 咳嗽分析                               │
│ - 护理建议                               │
│ - 就医建议                               │
│ - 免责声明                               │
└─────────────────────────────────────────┘
```

### 5.2 iOS 调用示例（Swift）

```swift
import Foundation

// MARK: - API Client

class CoughRecognitionAPI {
    static let shared = CoughRecognitionAPI()
    private let baseURL = "https://api.coughrecognition.com/v1"
    private var accessToken: String?

    // MARK: - 咳嗽分析

    func analyzeCough(
        audioURL: URL,
        ageGroup: AgeGroup,
        recordedAt: Date,
        source: RecordingSource
    ) async throws -> CoughAnalysisResult {

        let url = URL(string: "\(baseURL)/cough/analyze")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(accessToken ?? "")", forHTTPHeaderField: "Authorization")

        // 构建 multipart form data
        let boundary = UUID().uuidString
        request.setValue("multipart/form-data; boundary=\(boundary)", forHTTPHeaderField: "Content-Type")

        var body = Data()

        // 添加音频文件
        let audioData = try Data(contentsOf: audioURL)
        body.append("--\(boundary)\r\n")
        body.append("Content-Disposition: form-data; name=\"audio\"; filename=\"recording.m4a\"\r\n")
        body.append("Content-Type: audio/m4a\r\n\r\n")
        body.append(audioData)
        body.append("\r\n")

        // 添加其他参数
        body.append("--\(boundary)\r\n")
        body.append("Content-Disposition: form-data; name=\"age_group\"\r\n\r\n")
        body.append("\(ageGroup.rawValue)\r\n")

        body.append("--\(boundary)\r\n")
        body.append("Content-Disposition: form-data; name=\"recorded_at\"\r\n\r\n")
        body.append("\(ISO8601DateFormatter().string(from: recordedAt))\r\n")

        body.append("--\(boundary)\r\n")
        body.append("Content-Disposition: form-data; name=\"source\"\r\n\r\n")
        body.append("\(source.rawValue)\r\n")

        body.append("--\(boundary)--\r\n")

        request.httpBody = body

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.requestFailed
        }

        return try JSONDecoder().decode(CoughAnalysisResponse.self, from: data).data
    }

    // MARK: - AI 诊疗建议

    func getDiagnosisSuggestion(
        analysisResult: CoughAnalysisResult,
        ageGroup: AgeGroup,
        ageMonths: Int?,
        medicalHistory: MedicalHistory
    ) async throws -> DiagnosisSuggestion {

        let url = URL(string: "\(baseURL)/diagnosis/suggest")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(accessToken ?? "")", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let requestBody = DiagnosisRequest(
            analysisId: analysisResult.analysisId,
            coughAnalysis: analysisResult,
            ageGroup: ageGroup.rawValue,
            ageMonths: ageMonths,
            medicalHistory: medicalHistory
        )

        request.httpBody = try JSONEncoder().encode(requestBody)

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.requestFailed
        }

        return try JSONDecoder().decode(DiagnosisResponse.self, from: data).data
    }
}

// MARK: - Data Models

enum AgeGroup: String, Codable {
    case infant = "infant"
    case toddler = "toddler"
    case preschool = "preschool"
    case schoolAge = "school_age"
}

enum RecordingSource: String, Codable {
    case manual = "manual"
    case monitor = "monitor"
}

struct MedicalHistory: Codable {
    let symptoms: [String]
    let temperature: Double?
    let temperatureUnit: String?
    let medications: [Medication]?
    let illnessDays: Int
    let coughPattern: CoughPattern?
    let additionalNotes: String?
}

struct Medication: Codable {
    let name: String
    let dosage: String
    let frequency: String
}

struct CoughPattern: Codable {
    let worseAtNight: Bool?
    let worseAfterActivity: Bool?
    let triggeredBy: [String]?
}
```

---

## 六、安全与隐私

### 6.1 数据传输安全

| 项目 | 措施 |
|------|------|
| 传输加密 | TLS 1.3 |
| 证书验证 | Certificate Pinning |
| API 认证 | JWT Token + Refresh Token |

### 6.2 数据存储

| 数据类型 | 存储策略 | 保留期限 |
|----------|----------|----------|
| 音频文件 | 分析完成后不保留原始音频 | 即时删除 |
| 分析结果 | 加密存储 | 用户可请求删除 |
| 用户信息 | 最小化收集 | 遵守隐私政策 |

### 6.3 免责声明

每次 API 响应都包含免责声明，App 端必须展示：

```
以上建议仅供参考，不能替代医生的专业诊断。
如有疑虑或症状加重，请及时就医。
```

---

## 七、附录

### A. 完整错误码列表

| 错误码 | HTTP 状态码 | 说明 |
|--------|-------------|------|
| UNAUTHORIZED | 401 | 未授权或 Token 过期 |
| FORBIDDEN | 403 | 无权限访问 |
| INVALID_REQUEST | 400 | 请求参数错误 |
| INVALID_AUDIO_FORMAT | 400 | 音频格式不支持 |
| AUDIO_TOO_SHORT | 400 | 音频时长过短 |
| AUDIO_TOO_LONG | 400 | 音频时长过长 |
| AUDIO_CORRUPT | 400 | 音频文件损坏 |
| MISSING_REQUIRED_FIELD | 400 | 缺少必填字段 |
| INVALID_AGE_GROUP | 400 | 无效的年龄段 |
| ANALYSIS_NOT_FOUND | 404 | 分析记录不存在 |
| PROCESSING_FAILED | 500 | 服务器处理失败 |
| MODEL_ERROR | 500 | ML 模型错误 |
| RATE_LIMITED | 429 | 请求频率超限 |
| SERVICE_UNAVAILABLE | 503 | 服务暂时不可用 |

### B. 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0 | 2026-01-22 | 初始版本 |

### C. 相关文档

- [PRD.md](../PRD.md) - 产品需求文档
- [ML-Plan.md](ML-Plan.md) - ML 方案文档
- [设计方案.md](../设计方案.md) - 技术设计方案

---

*文档结束*
