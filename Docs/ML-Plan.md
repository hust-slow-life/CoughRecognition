# CoughRecognition ML 方案文档

> 版本：1.0
> 创建时间：2026-01-22
> 最后更新：2026-01-22
> 状态：草案

---

## 一、概述

本文档详细描述 CoughRecognition 项目的机器学习方案，包括数据集选择、模型架构、训练流程和部署策略。

### 1.1 ML 任务定义

| 任务 | 输入 | 输出 | 用途 |
|------|------|------|------|
| 咳嗽检测 | 音频片段 | 是否包含咳嗽 | 后台监控触发录制 |
| 哭泣检测 | 音频片段 | 是否包含哭泣 | 后台监控触发录制 |
| 咳嗽分类 | 咳嗽音频 | 咳嗽类型 | 辅助诊断 |
| 咳嗽定位 | 音频片段 | 咳嗽时间点 | 多次咳嗽分离 |

---

## 二、数据集调研

### 2.1 咳嗽相关数据集

#### COUGHVID

| 项目 | 内容 |
|------|------|
| **规模** | 20,000+ 条咳嗽录音 |
| **来源** | 众包收集，全球用户 |
| **标注** | 健康状态（healthy/COVID/symptomatic）、咳嗽类型 |
| **格式** | WAV/WebM，采样率不统一（需预处理） |
| **链接** | https://github.com/virufy/coughvid |

**适用性分析**：
- 优势：数据量大，标注丰富，包含 COVID 相关标签
- 局限：以成人为主，缺少儿童数据；录音环境差异大
- 用途：咳嗽检测预训练、成人咳嗽分类参考

#### Coswara

| 项目 | 内容 |
|------|------|
| **规模** | 2,000+ 用户，每用户多种音频 |
| **来源** | 印度理工学院（IISc Bangalore） |
| **内容** | 咳嗽（浅咳/深咳）、呼吸（浅/深）、语音、数数 |
| **标注** | 症状、COVID 状态、年龄、性别 |
| **链接** | https://github.com/iiscleap/Coswara-Data |

**适用性分析**：
- 优势：包含年龄信息，多种呼吸道声音
- 局限：数据量较小，儿童样本少
- 用途：补充 COUGHVID，呼吸音分析参考

#### AudioSet

| 项目 | 内容 |
|------|------|
| **规模** | 200 万+ 音频片段（YouTube） |
| **来源** | Google Research |
| **咳嗽相关** | 标签包含 Cough、Sneeze、Throat clearing 等 |
| **链接** | https://research.google.com/audioset/ |

**适用性分析**：
- 优势：规模大，场景丰富，是 YAMNet 的训练数据
- 局限：YouTube 音频质量参差不齐，无法直接下载
- 用途：理解预训练模型能力，不直接用于训练

### 2.2 哭泣相关数据集

#### ESC-50

| 项目 | 内容 |
|------|------|
| **规模** | 2,000 条环境声音（50 类 × 40 条） |
| **来源** | 学术数据集（Karol Piczak） |
| **相关类别** | crying_baby（婴儿哭声） |
| **格式** | WAV，5 秒，44.1kHz |
| **链接** | https://github.com/karolpiczak/ESC-50 |

**适用性分析**：
- 优势：音频质量好，标注准确，数据干净
- 局限：每类仅 40 条，数据量小
- 用途：哭泣检测初始训练，配合数据增强

#### Donate a Cry

| 项目 | 内容 |
|------|------|
| **规模** | 1,000+ 婴儿哭声 |
| **来源** | 众包收集 |
| **标注** | 哭声原因（饥饿、疼痛、疲倦等） |
| **链接** | https://github.com/gveres/donateacry-corpus |

**适用性分析**：
- 优势：专注婴儿哭声，包含原因标注
- 局限：数据量中等，标注原因可能不准确
- 用途：婴儿哭泣检测训练

### 2.3 数据集局限性总结

| 问题 | 影响 | 缓解策略 |
|------|------|----------|
| 缺少儿童咳嗽数据 | 儿童场景检测准确率可能较低 | 收集真实用户数据迭代优化 |
| 录音环境差异大 | 模型泛化能力受影响 | 数据增强（加噪、混响） |
| 咳嗽类型标注不统一 | 分类标准难以对齐 | 重新定义统一分类体系 |
| 缺少中文/亚洲人群数据 | 可能存在人群差异 | 关注用户反馈，持续优化 |

---

## 三、模型方案

### 3.1 整体架构

```
┌────────────────────────────────────────────────────────┐
│                    iOS 应用                            │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │ 咳嗽检测模型 │   │ 哭泣检测模型 │   │ 咳嗽分类模型 │  │
│  │ (Core ML)   │   │ (Core ML)   │   │ (云端API)  │   │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘  │
│         │                 │                 │         │
│         └─────────────────┴─────────────────┘         │
│                          │                            │
│                    YAMNet 特征提取                     │
│                          │                            │
│                    Mel 频谱图生成                      │
│                          │                            │
│                    音频预处理                          │
└────────────────────────────────────────────────────────┘
```

### 3.2 咳嗽检测模型

#### 方案：YAMNet 迁移学习

```
输入音频 → 重采样(16kHz) → 分帧(0.96s) → Mel频谱图 → YAMNet → 二分类头 → 咳嗽/非咳嗽
```

**YAMNet 简介**：
- Google 开源的音频分类模型
- 基于 MobileNet v1 架构，针对移动端优化
- 在 AudioSet（521 类）上预训练
- 原生支持咳嗽检测（类别 #48: Cough）

**迁移学习策略**：
```
┌─────────────────────────────────────┐
│          YAMNet 特征提取器           │  ← 冻结参数
│  (MobileNet v1 + Mel Spectrogram)  │
└─────────────────────────────────────┘
                  │
                  │ 1024 维特征向量
                  ↓
┌─────────────────────────────────────┐
│           自定义分类头               │  ← 训练参数
│  Dense(256) → Dropout → Dense(2)   │
└─────────────────────────────────────┘
```

**模型规格**：
| 项目 | 规格 |
|------|------|
| 输入 | 0.96 秒音频（15600 采样点 @ 16kHz） |
| 输出 | 二分类概率（咳嗽/非咳嗽） |
| 模型大小 | ~13MB（YAMNet） + ~1MB（分类头） |
| 推理速度 | <50ms（iPhone 12+） |

### 3.3 哭泣检测模型

#### 方案：YAMNet 迁移学习（与咳嗽检测类似）

```
输入音频 → 重采样(16kHz) → 分帧(0.96s) → Mel频谱图 → YAMNet → 二分类头 → 哭泣/非哭泣
```

**训练数据**：
- ESC-50 crying_baby 类别（40 条 × 数据增强）
- Donate a Cry 数据集（1000+ 条）
- 负样本：ESC-50 其他类别、静音、白噪声

**模型规格**：
| 项目 | 规格 |
|------|------|
| 输入 | 0.96 秒音频 |
| 输出 | 二分类概率（哭泣/非哭泣） |
| 模型大小 | ~14MB |
| 推理速度 | <50ms |

### 3.4 咳嗽分类模型

#### 方案：自定义分类器（云端部署）

```
咳嗽音频 → 特征提取 → 分类器 → 咳嗽类型
```

**分类体系**：
| 类别 | 声学特征 | 临床意义 |
|------|----------|----------|
| 干咳 (dry) | 短促、高频为主 | 感冒初期、过敏、咽炎 |
| 湿咳 (wet) | 伴有痰音、低频成分多 | 支气管炎、肺炎 |
| 犬吠样咳 (barking) | 类似狗叫、喉部振动 | 急性喉炎（假性哮吼） |
| 喘息性咳嗽 (wheezing) | 伴有喘息音 | 哮喘、毛细支气管炎 |
| 痉挛性咳嗽 (paroxysmal) | 阵发性、连续多声 | 百日咳 |

**模型架构**：
```
┌─────────────────────────────────────┐
│          YAMNet 特征提取器           │
└─────────────────────────────────────┘
                  │
                  │ 1024 维特征向量
                  ↓
┌─────────────────────────────────────┐
│            分类网络                  │
│  Dense(512) → BatchNorm → ReLU     │
│  Dropout(0.3)                       │
│  Dense(256) → BatchNorm → ReLU     │
│  Dropout(0.3)                       │
│  Dense(5) → Softmax                │
└─────────────────────────────────────┘
```

**为什么云端部署**：
- 分类需要更复杂的模型，准确率优先
- 可持续迭代更新，无需发布新版本
- 处理时间要求宽松（<1 分钟可接受）

### 3.5 咳嗽定位（时间点检测）

#### 方案：滑动窗口 + 咳嗽检测

```
长音频 → 滑动窗口(0.96s, 步长0.48s) → 咳嗽检测 → 概率序列 → 峰值检测 → 咳嗽时间点列表
```

**实现细节**：
1. 对整段录音使用滑动窗口
2. 每个窗口计算咳嗽概率
3. 使用阈值（如 0.7）和非极大值抑制（NMS）定位咳嗽事件
4. 输出每次咳嗽的起始时间

---

## 四、训练流程

### 4.1 数据预处理

#### 采样率标准化
```python
# 统一转换为 16kHz 单声道
target_sample_rate = 16000
audio = librosa.resample(audio, orig_sr=orig_sr, target_sr=target_sample_rate)
audio = librosa.to_mono(audio)
```

#### 时长标准化
```python
# YAMNet 输入: 0.96 秒
target_length = 15600  # 0.975s @ 16kHz

# 短音频: zero-padding
if len(audio) < target_length:
    audio = np.pad(audio, (0, target_length - len(audio)))

# 长音频: 分帧处理
else:
    frames = librosa.util.frame(audio, frame_length=target_length, hop_length=target_length//2)
```

#### Mel 频谱图生成
```python
# YAMNet 使用的参数
mel_spec = librosa.feature.melspectrogram(
    y=audio,
    sr=16000,
    n_mels=64,
    n_fft=400,      # 25ms 窗口
    hop_length=160, # 10ms 步长
    fmin=125,
    fmax=7500
)
log_mel = np.log(mel_spec + 1e-6)
```

### 4.2 数据增强

| 增强方法 | 参数 | 目的 |
|----------|------|------|
| 时间拉伸 | rate: 0.9-1.1 | 适应说话速度变化 |
| 音高变换 | semitones: -2 to +2 | 适应不同年龄/性别 |
| 添加噪声 | SNR: 10-30dB | 提高噪声环境鲁棒性 |
| 混响 | RT60: 0.1-0.5s | 模拟室内环境 |
| 随机裁剪 | - | 增加位置不变性 |
| SpecAugment | freq_mask, time_mask | 防止过拟合 |

```python
# 示例：audiomentations 库
from audiomentations import Compose, AddGaussianNoise, TimeStretch, PitchShift

augment = Compose([
    AddGaussianNoise(min_amplitude=0.001, max_amplitude=0.015, p=0.5),
    TimeStretch(min_rate=0.9, max_rate=1.1, p=0.5),
    PitchShift(min_semitones=-2, max_semitones=2, p=0.5),
])

augmented_audio = augment(samples=audio, sample_rate=16000)
```

### 4.3 训练配置

#### 咳嗽检测模型
```python
# 数据划分
train_ratio = 0.8
val_ratio = 0.1
test_ratio = 0.1

# 训练参数
batch_size = 32
epochs = 50
learning_rate = 1e-4
optimizer = Adam(lr=learning_rate)
loss = BinaryCrossentropy()

# 早停策略
early_stopping = EarlyStopping(
    monitor='val_loss',
    patience=5,
    restore_best_weights=True
)

# 学习率调度
lr_scheduler = ReduceLROnPlateau(
    monitor='val_loss',
    factor=0.5,
    patience=3
)
```

#### 类别不平衡处理
```python
# 咳嗽样本通常少于非咳嗽样本
# 方案1：类别权重
class_weight = {
    0: 1.0,  # 非咳嗽
    1: 3.0   # 咳嗽（提高权重）
}

# 方案2：过采样
from imblearn.over_sampling import SMOTE
X_resampled, y_resampled = SMOTE().fit_resample(X, y)
```

### 4.4 评估指标

| 指标 | 公式 | 目标值 | 说明 |
|------|------|--------|------|
| Accuracy | (TP+TN)/(TP+TN+FP+FN) | >90% | 整体准确率 |
| Precision | TP/(TP+FP) | >85% | 检测为咳嗽的准确性 |
| Recall | TP/(TP+FN) | >90% | 咳嗽检出率（重要） |
| F1 Score | 2×P×R/(P+R) | >87% | 综合指标 |
| AUC-ROC | - | >0.95 | 分类能力 |

**召回率优先**：
- 漏检咳嗽比误报更严重
- 调整分类阈值，提高召回率
- 可接受适度降低精确率

### 4.5 评估流程

```python
# 混淆矩阵
from sklearn.metrics import confusion_matrix, classification_report

y_pred = model.predict(X_test)
y_pred_classes = (y_pred > threshold).astype(int)

print(classification_report(y_test, y_pred_classes))
print(confusion_matrix(y_test, y_pred_classes))

# ROC 曲线
from sklearn.metrics import roc_curve, auc

fpr, tpr, thresholds = roc_curve(y_test, y_pred)
roc_auc = auc(fpr, tpr)
```

---

## 五、部署方案

### 5.1 模型转换流程

```
TensorFlow/PyTorch 模型
         │
         ↓
    TensorFlow SavedModel / ONNX
         │
         ↓
    Core ML Tools 转换
         │
         ↓
    .mlmodel / .mlpackage
         │
         ↓
    Xcode 集成
```

#### TensorFlow → Core ML
```python
import coremltools as ct

# 加载 TensorFlow 模型
tf_model = tf.keras.models.load_model('cough_detector.h5')

# 转换为 Core ML
mlmodel = ct.convert(
    tf_model,
    inputs=[ct.TensorType(shape=(1, 96, 64, 1), name='mel_spectrogram')],
    minimum_deployment_target=ct.target.iOS17
)

# 添加元数据
mlmodel.author = 'CoughRecognition'
mlmodel.short_description = '咳嗽检测模型'
mlmodel.version = '1.0.0'

# 保存
mlmodel.save('CoughDetector.mlpackage')
```

#### PyTorch → Core ML
```python
import coremltools as ct
import torch

# 加载 PyTorch 模型
torch_model = torch.load('cough_detector.pt')
torch_model.eval()

# 跟踪模型
example_input = torch.rand(1, 96, 64)
traced_model = torch.jit.trace(torch_model, example_input)

# 转换为 Core ML
mlmodel = ct.convert(
    traced_model,
    inputs=[ct.TensorType(shape=(1, 96, 64), name='mel_spectrogram')],
    minimum_deployment_target=ct.target.iOS17
)

mlmodel.save('CoughDetector.mlpackage')
```

### 5.2 iOS 集成

#### 模型加载
```swift
import CoreML
import AVFoundation

class CoughDetector {
    private let model: CoughDetector

    init() throws {
        let config = MLModelConfiguration()
        config.computeUnits = .all  // 使用 GPU/Neural Engine
        self.model = try CoughDetector(configuration: config)
    }
}
```

#### 音频处理
```swift
// 音频预处理
func preprocessAudio(_ audioBuffer: AVAudioPCMBuffer) -> MLMultiArray? {
    // 1. 重采样到 16kHz
    let resampled = resample(audioBuffer, to: 16000)

    // 2. 计算 Mel 频谱图
    let melSpec = computeMelSpectrogram(resampled)

    // 3. 转换为 MLMultiArray
    return melSpec.toMLMultiArray()
}

// 推理
func detect(audioBuffer: AVAudioPCMBuffer) throws -> Float {
    guard let input = preprocessAudio(audioBuffer) else {
        throw DetectionError.preprocessingFailed
    }

    let output = try model.prediction(mel_spectrogram: input)
    return output.probability[1]  // 咳嗽概率
}
```

### 5.3 实时推理优化

| 优化策略 | 说明 | 效果 |
|----------|------|------|
| 使用 Neural Engine | MLModelConfiguration.computeUnits = .all | 速度提升 2-3x |
| 批量推理 | 多帧一起处理 | 减少开销 |
| 音频缓冲复用 | 避免频繁内存分配 | 减少 GC |
| 后台线程处理 | 不阻塞 UI | 流畅体验 |
| 模型预热 | 启动时预加载 | 首次推理更快 |

### 5.4 后台监控实现

```swift
class AudioMonitor {
    private let audioEngine = AVAudioEngine()
    private let coughDetector: CoughDetector
    private let cryDetector: CryDetector

    // 循环缓冲区（保存最近 5 秒音频）
    private var audioBuffer: CircularBuffer<Float>

    func startMonitoring() {
        let inputNode = audioEngine.inputNode
        let format = inputNode.inputFormat(forBus: 0)

        inputNode.installTap(onBus: 0, bufferSize: 4096, format: format) { [weak self] buffer, time in
            self?.processAudio(buffer)
        }

        try? audioEngine.start()
    }

    private func processAudio(_ buffer: AVAudioPCMBuffer) {
        // 1. 添加到循环缓冲区
        audioBuffer.append(buffer)

        // 2. 每 0.96 秒检测一次
        if shouldRunDetection() {
            let segment = audioBuffer.getLastSegment(duration: 0.96)

            let coughProb = try? coughDetector.detect(segment)
            let cryProb = try? cryDetector.detect(segment)

            // 3. 如果检测到事件，保存录音
            if coughProb > 0.7 || cryProb > 0.7 {
                saveRecording(audioBuffer.getLastSegment(duration: 10.0))
            }
        }
    }
}
```

---

## 六、年龄段适配策略

### 6.1 年龄段差异分析

| 年龄段 | 咳嗽特征 | 模型挑战 |
|--------|----------|----------|
| 婴儿(0-1岁) | 咳嗽声弱、易与哭声混淆 | 需区分咳嗽和哭泣 |
| 幼儿(1-3岁) | 声音高亢、咳嗽模式不规律 | 频率特征不同于成人 |
| 学龄前(3-6岁) | 咳嗽力度增强、模式更规律 | 逐渐接近成人特征 |
| 学龄(6-18岁) | 接近成人咳嗽特征 | 可使用通用模型 |

### 6.2 方案对比

#### 方案 A：单一通用模型 + 年龄段后处理

```
音频 → 通用咳嗽检测模型 → 检测结果 → 年龄段后处理 → 最终结果
                                           ↑
                                      年龄段参数
```

**优点**：
- 模型体积小，只需维护一个模型
- 实现简单

**缺点**：
- 对各年龄段准确率可能不均衡
- 后处理能力有限

#### 方案 B：多模型方案

```
音频 → 年龄段选择 → 对应年龄段模型 → 检测结果
         ↓
    婴儿模型 / 幼儿模型 / 通用模型
```

**优点**：
- 各年龄段可针对性优化
- 准确率更高

**缺点**：
- 模型体积大（需要多个模型）
- 维护成本高

### 6.3 推荐方案：渐进式优化

**MVP 阶段**：使用方案 A（单一通用模型）
- 先使用通用模型快速上线
- 收集用户反馈和真实数据

**迭代阶段**：逐步演进为方案 B
- 根据用户数据分析各年龄段表现
- 针对准确率较低的年龄段训练专用模型
- 优先考虑婴儿年龄段（差异最大）

### 6.4 年龄段特征调整

```python
# 根据年龄段调整检测阈值
AGE_THRESHOLDS = {
    'infant': {      # 0-1岁
        'cough_threshold': 0.6,   # 降低阈值，提高召回
        'cry_threshold': 0.7,
    },
    'toddler': {     # 1-3岁
        'cough_threshold': 0.65,
        'cry_threshold': 0.75,
    },
    'preschool': {   # 3-6岁
        'cough_threshold': 0.7,
        'cry_threshold': 0.8,
    },
    'school_age': {  # 6-18岁
        'cough_threshold': 0.7,
        'cry_threshold': 0.85,    # 提高阈值，减少误报
    }
}
```

---

## 七、模型迭代策略

### 7.1 用户反馈收集

| 反馈类型 | 收集方式 | 用途 |
|----------|----------|------|
| 误报标记 | 用户标记"这不是咳嗽" | 负样本收集 |
| 漏检反馈 | 用户补充录音 | 正样本收集 |
| 分类纠正 | 用户修改咳嗽类型 | 分类标签校正 |

### 7.2 数据收集（需用户同意）

```swift
// 匿名数据收集（用户同意后）
struct AnonymousDataCollection {
    let audioHash: String        // 音频指纹，非原始音频
    let modelPrediction: Float   // 模型预测值
    let userFeedback: Bool       // 用户反馈
    let ageGroup: String         // 年龄段
    let timestamp: Date          // 时间戳
}
```

### 7.3 模型更新流程

```
用户反馈数据收集
       │
       ↓
数据清洗和标注
       │
       ↓
模型重新训练
       │
       ↓
A/B 测试验证
       │
       ↓
模型发布（App 更新 / 云端更新）
```

---

## 八、技术风险和缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 儿童数据不足 | 儿童场景准确率低 | 优先收集用户数据，数据增强 |
| 环境噪音干扰 | 误报率高 | 噪声增强训练，VAD 预过滤 |
| 模型体积过大 | App 安装包大 | 模型压缩，按需下载 |
| 推理耗电 | 后台监控耗电快 | 优化推理频率，低电量策略 |
| Core ML 兼容性 | 部分设备不支持 | 设置最低 iOS 版本，降级方案 |

---

## 九、里程碑规划

### Phase 1：基础模型（2 周）
- [ ] 数据集下载和预处理
- [ ] YAMNet 迁移学习环境搭建
- [ ] 咳嗽检测模型训练
- [ ] Core ML 模型转换

### Phase 2：检测模型完善（1 周）
- [ ] 哭泣检测模型训练
- [ ] iOS 集成测试
- [ ] 性能优化

### Phase 3：分类模型（2 周）
- [ ] 咳嗽分类数据准备
- [ ] 分类模型训练
- [ ] 云端 API 部署

### Phase 4：迭代优化（持续）
- [ ] 用户反馈收集
- [ ] 模型持续优化
- [ ] 年龄段专用模型（如需要）

---

## 十、参考资料

### 论文
1. Hershey, S. et al. "CNN Architectures for Large-Scale Audio Classification" (YAMNet)
2. Piczak, K. "Environmental Sound Classification with Convolutional Neural Networks" (ESC-50)
3. Orlandic, L. et al. "The COUGHVID crowdsourcing dataset" (COUGHVID)

### 开源项目
1. TensorFlow Models - YAMNet: https://github.com/tensorflow/models/tree/master/research/audioset/yamnet
2. Core ML Tools: https://github.com/apple/coremltools
3. audiomentations: https://github.com/iver56/audiomentations

### 工具文档
1. Core ML 文档: https://developer.apple.com/documentation/coreml
2. AVFoundation 音频处理: https://developer.apple.com/documentation/avfoundation

---

## 附录

### A. 数据集获取方式

```bash
# COUGHVID
git clone https://github.com/virufy/coughvid.git

# ESC-50
git clone https://github.com/karolpiczak/ESC-50.git

# Coswara
git clone https://github.com/iiscleap/Coswara-Data.git

# Donate a Cry
git clone https://github.com/gveres/donateacry-corpus.git
```

### B. 开发环境

| 工具 | 版本 | 用途 |
|------|------|------|
| Python | 3.10+ | 模型训练 |
| TensorFlow | 2.15+ | 深度学习框架 |
| coremltools | 7.0+ | Core ML 转换 |
| librosa | 0.10+ | 音频处理 |
| audiomentations | 0.34+ | 数据增强 |
| Xcode | 15+ | iOS 开发 |

---

*文档结束*
