# 视频世界原子操作设计（Atomic Operations Design）

## 文档目标

本文档定义了一套**最小完备**的原子操作集合，使LLM能够通过这些操作访问和理解视频内容。这些原子操作将在Phase 1被LLM组合成高层工具，在Phase 2被LLM循环调用以完成复杂的视频推理任务。

---

## 一、设计原则

### 1.1 原子操作的核心特征

1. **不可再分性**：每个操作是一个完整的、不可进一步拆解的功能单元
2. **通用性**：不针对特定任务，提供通用的视频访问能力
3. **可组合性**：可以被组合成更复杂的操作流程
4. **LLM友好**：接口语义清晰，参数合理，输出可直接使用
5. **正交性**：操作之间功能独立，最小化重叠

### 1.2 粒度平衡

- **太细**（如"读取第N帧的像素"）→ LLM难以有效组合，调用次数过多
- **太粗**（如"找到视频中的主角并总结其行为"）→ 失去组合灵活性，本质上是预定义pipeline
- **适中**（如"在时间段T内检测类别C的对象"）→ 既是独立功能单元，又可灵活组合

---

## 二、原子操作分类体系

我们将原子操作分为5大类，共16个操作：

```
视频世界原子操作（16个）
│
├── 1. Video API（视频数据类）- 2个
│   ├── get_video_info         // 获取视频元信息
│   └── get_temporal_structure // 获取时间结构
│
├── 2. Perception API（感知类）- 6个
│   ├── describe_visual        // 生成视觉描述
│   ├── detect_objects         // 对象检测
│   ├── recognize_activity     // 动作识别
│   ├── get_scene_attributes   // 场景属性
│   ├── get_transcript         // 语音转文本
│   └── detect_audio_events    // 音频事件检测
│
├── 3. Entity Trace API（实体追踪类）- 4个
│   ├── register_entity        // 注册实体
│   ├── track_entity           // 获取实体轨迹
│   ├── query_entity_state     // 查询实体状态
│   └── spatial_relation       // 查询空间关系
│
├── 4. Retrieval API（检索类）- 2个
│   ├── temporal_search        // 时间段检索
│   └── find_similar_segments  // 相似片段查找
│
└── 5. Memory API（记忆类）- 2个
    ├── write_memory           // 写入记忆
    └── read_memory            // 读取记忆
```

---

## 三、原子操作详细规范

### 3.1 Video API（视频数据类）

视频数据类API提供视频的原始数据和结构信息，包括元数据和时间结构，是LLM访问视频的基础。

---

#### API-1: `get_video_info`

**功能**：获取视频的基本元信息

**输入参数**：
```json
{
  "video_id": "string"  // 视频唯一标识符
}
```

**输出**：
```json
{
  "duration": 180.5,           // 总时长（秒）
  "fps": 30.0,                 // 帧率
  "resolution": {
    "width": 1920,
    "height": 1080
  },
  "has_audio": true,           // 是否有音频
  "num_frames": 5415,          // 总帧数
  "file_size_mb": 245.7        // 文件大小（MB）
}
```

**设计理由**：
- LLM需要知道视频总长度以规划探索策略
- 帧率信息用于时间戳与帧号的转换
- 音频信息决定是否调用音频相关API

**典型使用场景**：
```
LLM: 首先调用 get_video_info() 了解视频是3分钟长，有音频
     → 决定探索策略：可以使用 get_transcript() 获取对话
```

---

#### API-2: `get_temporal_structure`

**功能**：获取视频的时间结构（场景切分）

**输入参数**：
```json
{
  "video_id": "string",
  "granularity": "coarse"  // 粒度: "coarse"(粗) | "fine"(细)
}
```

**输出**：
```json
{
  "segments": [
    {
      "segment_id": "seg_001",
      "start_time": 0.0,
      "end_time": 15.4,
      "duration": 15.4,
      "type": "scene"          // "scene"(场景变化) | "shot"(镜头切换)
    },
    {
      "segment_id": "seg_002",
      "start_time": 15.4,
      "end_time": 32.1,
      "duration": 16.7,
      "type": "scene"
    }
  ],
  "total_segments": 12
}
```

**设计理由**：
- 提供视频的"章节感"，避免LLM盲目搜索
- 粗粒度（coarse）：大场景切换，适合快速浏览（如电影中不同地点）
- 细粒度（fine）：镜头级别，适合精确分析（如同一场景内的镜头切换）
- segment_id可用于后续操作的引用

**典型使用场景**：
```
LLM: 调用 get_temporal_structure(granularity="coarse")
     → 发现视频有12个大场景
     → 决定先用 temporal_search() 在全局检索关键段
     → 而不是逐秒扫描
```

---

### 3.2 Perception API（感知类）

感知类API是LLM获取视频内容信息的核心手段，涵盖视觉和音频两个模态。

---

#### API-3: `describe_visual`

**功能**：生成视频片段的自然语言描述

**输入参数**：
```json
{
  "video_id": "string",
  "start_time": 45.2,          // 起始时间（秒）
  "end_time": 48.6,            // 结束时间（秒）
  "detail_level": "standard",  // 描述详细程度: "brief" | "standard" | "detailed"
  "focus": null                // 可选：聚焦方面 "people" | "actions" | "objects" | "scene"
}
```

**输出**：
```json
{
  "description": "A woman in a red apron is cooking at a stove in a modern kitchen. She stirs a pot with a wooden spoon while occasionally glancing at her phone on the counter.",
  "confidence": 0.91,
  "num_frames_analyzed": 8     // 分析的帧数
}
```

**detail_level说明**：
- `"brief"`: 一句话概括（<20词），用于快速浏览大量片段
- `"standard"`: 标准描述（2-3句话），平衡详细度与效率
- `"detailed"`: 详细描述（>5句话），包含更多细节，用于关键片段

**设计理由**：
- 这是LLM获取视觉信息的主要途径，将视觉转换为文本
- 不同detail_level适应不同任务需求
- focus参数让LLM能引导描述的侧重点
- 相比结构化输出，自然语言描述更灵活，能表达复杂场景

**与其他感知API的互补**：
- `describe_visual`：整体自然语言描述，灵活但需LLM解析
- `detect_objects`：结构化的对象信息，精确但有限
- `recognize_activity`：结构化的动作标签，便于程序处理
- `get_scene_attributes`：结构化的场景属性，便于逻辑推理

---

#### API-4: `detect_objects`

**功能**：检测指定时间点画面中的对象

**输入参数**：
```json
{
  "video_id": "string",
  "timestamp": 45.2,           // 时间点（秒）
  "category": null,            // 可选：目标类别 "person" | "vehicle" | "animal" | ...
  "query": null,               // 可选：自然语言查询，如"person wearing red"（开放词汇检测）
  "confidence_threshold": 0.3  // 置信度阈值
}
```

**输出**：
```json
{
  "timestamp": 45.2,           // 实际使用的时间戳
  "objects": [
    {
      "object_id": "obj_tmp_001",    // 临时ID（仅在此帧有效）
      "category": "person",
      "bbox": [0.32, 0.15, 0.25, 0.70],  // 归一化坐标 [x, y, w, h]
      "confidence": 0.94,
      "attributes": {                    // 可选：可识别的属性
        "clothing_color": "red",
        "age_group": "adult",
        "gender": "female"
      }
    },
    {
      "object_id": "obj_tmp_002",
      "category": "phone",
      "bbox": [0.52, 0.35, 0.06, 0.10],
      "confidence": 0.87
    }
  ]
}
```

**参数说明**：
- `category` 和 `query` 二选一或都不指定：
  - 都不指定：返回所有检测到的对象
  - 指定category：只返回该类别的对象（基于预定义类别）
  - 指定query：开放词汇检测，可以检测任意文本描述的对象

**设计理由**：
- 提供结构化的对象信息，便于程序处理
- bbox信息支持空间推理（"A在B的左边"）
- object_id可用于`register_entity`建立持久引用
- 开放词汇检测（query参数）提供灵活性

**重要**：
- 此操作返回的object_id是临时的，仅在当前帧有效
- 如需跨帧跟踪，必须使用`register_entity`建立持久实体

---

#### API-5: `recognize_activity`

**功能**：识别视频片段中的活动/动作

**输入参数**：
```json
{
  "video_id": "string",
  "start_time": 45.2,
  "end_time": 48.6,
  "granularity": "event"       // "atomic"(原子动作) | "event"(事件)
}
```

**输出**：
```json
{
  "activities": [
    {
      "label": "cooking",
      "confidence": 0.89,
      "start_time": 45.2,      // 可选：动作的精确起止时间
      "end_time": 48.1
    },
    {
      "label": "using_phone",
      "confidence": 0.76,
      "start_time": 46.5,
      "end_time": 48.6
    }
  ],
  "primary_activity": "cooking"  // 主要动作
}
```

**granularity说明**：
- `"atomic"`: 原子动作级别 - "walking", "sitting", "reaching", "grasping"
- `"event"`: 事件级别 - "cooking a meal", "having a conversation", "playing basketball"

**设计理由**：
- 动作是视频理解的关键语义信息
- 返回多个动作支持并发动作场景（如边做饭边打电话）
- 时间定位信息帮助LLM理解动作的时序关系
- 结构化标签相比自然语言描述更易于程序处理

---

#### API-6: `get_scene_attributes`

**功能**：获取场景的环境属性

**输入参数**：
```json
{
  "video_id": "string",
  "timestamp": 45.2,
  "attributes": ["location", "lighting", "time_of_day"]  // 可选：指定查询的属性
}
```

**输出**：
```json
{
  "location": "indoor_kitchen",      // 室内厨房
  "lighting": "artificial_bright",   // 人工光源，明亮
  "time_of_day": "daytime",          // 白天
  "weather": null,                   // 室内场景无天气信息
  "camera_angle": "medium_shot",     // 中景
  "camera_motion": "static"          // 静止镜头
}
```

**可查询的属性类型**：
- `location`: 地点类型（indoor_kitchen, outdoor_street, indoor_office, ...）
- `lighting`: 光照条件（artificial_bright, natural_dim, dark, ...）
- `time_of_day`: 时间段（daytime, night, dawn, dusk）
- `weather`: 天气（sunny, rainy, cloudy, ...，仅适用于户外场景）
- `camera_angle`: 镜头角度（close_up, medium_shot, long_shot, ...）
- `camera_motion`: 镜头运动（static, pan, tilt, zoom, tracking, ...）

**设计理由**：
- 场景属性对某些问题至关重要（"视频是白天还是晚上拍的？"）
- 结构化输出便于LLM进行逻辑推理
- 与对象、动作信息互补，形成完整的场景理解
- 如不指定attributes参数，返回所有可识别的属性

---

#### API-7: `get_transcript`

**功能**：获取语音转文本结果（ASR）

**输入参数**：
```json
{
  "video_id": "string",
  "time_range": {              // 可选：时间范围
    "start_time": 45.0,
    "end_time": 60.0
  },
  "include_speaker_info": false  // 是否包含说话人信息（如果支持）
}
```

**输出**：
```json
{
  "transcript": [
    {
      "start_time": 46.2,
      "end_time": 49.5,
      "text": "Can you pass me the salt?",
      "speaker_id": "speaker_1",   // 可选：说话人ID
      "confidence": 0.92
    },
    {
      "start_time": 50.1,
      "end_time": 52.3,
      "text": "Sure, here you go.",
      "speaker_id": "speaker_2",
      "confidence": 0.89
    }
  ]
}
```

**设计理由**：
- 对话内容是视频理解的重要线索
- 时间对齐的字幕支持"谁在何时说了什么"的推理
- 说话人识别帮助建立对话结构
- 如不指定time_range，返回全视频字幕

**应用场景**：
- 回答对话内容相关的问题
- 与视觉信息结合理解视听一致性
- 通过字幕进行基于文本的检索

---

#### API-8: `detect_audio_events`

**功能**：检测非语音音频事件

**输入参数**：
```json
{
  "video_id": "string",
  "time_range": {              // 可选：时间范围
    "start_time": 45.0,
    "end_time": 60.0
  },
  "event_types": null          // 可选：指定检测的事件类型列表
}
```

**输出**：
```json
{
  "audio_events": [
    {
      "event_type": "music",
      "start_time": 45.0,
      "end_time": 78.5,
      "confidence": 0.91,
      "description": "background_music"  // 可选：更详细的描述
    },
    {
      "event_type": "door_knock",
      "start_time": 62.3,
      "end_time": 63.1,
      "confidence": 0.87
    }
  ]
}
```

**常见音频事件类型**：
- `music`: 音乐
- `applause`: 掌声
- `laughter`: 笑声
- `door_knock`: 敲门声
- `glass_breaking`: 玻璃破碎
- `car_horn`: 汽车喇叭
- `footsteps`: 脚步声
- `alarm`: 警报声
- `phone_ringing`: 电话铃声
- ...

**设计理由**：
- 环境音对场景理解很重要（"有人敲门"、"背景音乐"）
- 补充视觉信息，形成多模态理解
- 事件类型枚举便于LLM处理
- 如不指定event_types，返回所有检测到的音频事件

---

### 3.3 Entity Trace API（实体追踪类）

实体追踪类API让LLM能够建立对视频中对象的持久引用，并跨时间跟踪对象的状态和关系。

---

#### API-9: `register_entity`

**功能**：为感兴趣的对象注册一个持久的实体ID

**输入参数**：
```json
{
  "video_id": "string",
  "initial_timestamp": 45.2,   // 实体首次明确出现的时间点
  "description": "woman in red apron",  // 实体描述
  "object_id": null            // 可选：来自detect_objects的临时对象ID
}
```

**输出**：
```json
{
  "entity_id": "ent_001",      // 全局唯一实体ID
  "matched_bbox": [0.32, 0.15, 0.25, 0.70],
  "confidence": 0.91,
  "visual_signature": "..."    // 内部使用的视觉特征签名
}
```

**参数说明**：
- `object_id`: 如提供，直接使用该检测结果；如不提供，系统根据description自动定位
- `description`: 自然语言描述，比bbox更符合LLM的使用习惯

**设计理由**：
- 视频中的对象会跨时间出现，需要持久的引用机制
- 实体ID是后续`track_entity`、`query_entity_state`、`spatial_relation`的前提
- 描述式注册比位置式注册更自然

**重要特性**：
- 一次对话会话中，entity_id在整个视频范围内唯一且持久
- 注册操作是有状态的（会修改系统状态）
- 系统会建立视觉特征用于后续跟踪

**典型使用场景**：
```
1. LLM调用 detect_objects(timestamp=45.2, query="woman in red")
2. 发现 obj_tmp_001
3. LLM调用 register_entity(initial_timestamp=45.2, object_id="obj_tmp_001")
4. 获得 entity_id="ent_001"
5. 后续可用 ent_001 引用该女性，即使她移动或暂时离开画面
```

---

#### API-10: `track_entity`

**功能**：获取实体在视频中的时空轨迹

**输入参数**：
```json
{
  "video_id": "string",
  "entity_id": "ent_001",
  "time_range": {              // 可选：限制查询的时间范围
    "start_time": 0.0,
    "end_time": 180.0
  }
}
```

**输出**：
```json
{
  "entity_id": "ent_001",
  "appearances": [
    {
      "appearance_id": "app_001",
      "start_time": 45.2,
      "end_time": 52.8,
      "segment_id": "seg_042",
      "duration": 7.6,
      "trajectory_summary": "stationary in center",  // 运动模式概述
      "avg_bbox": [0.33, 0.18, 0.24, 0.68]          // 平均位置
    },
    {
      "appearance_id": "app_002",
      "start_time": 78.5,
      "end_time": 85.2,
      "segment_id": "seg_067",
      "duration": 6.7,
      "trajectory_summary": "moving from left to right",
      "avg_bbox": [0.65, 0.22, 0.23, 0.65]
    }
  ],
  "summary": {
    "total_appearances": 2,
    "total_duration": 14.3,
    "first_appearance": 45.2,
    "last_appearance": 85.2,
    "disappearance_periods": [     // 消失的时间段
      {
        "start_time": 52.8,
        "end_time": 78.5,
        "duration": 25.7
      }
    ]
  }
}
```

**trajectory_summary说明**：
- `"stationary"`: 静止不动
- `"moving left to right"`: 从左向右移动
- `"moving right to left"`: 从右向左移动
- `"approaching camera"`: 靠近镜头
- `"moving away from camera"`: 远离镜头
- `"circular motion"`: 圆周运动
- `"erratic motion"`: 不规则运动

**设计理由**：
- 回答"X何时出现"、"X出现了几次"、"X在哪里"等问题的基础
- trajectory_summary提供运动模式的抽象，避免返回大量逐帧bbox数据
- 区分多次"出现"（appearance），每次出现是一个连续的时间段
- 时间范围限制支持局部查询

**输出简化说明**：
- 不返回每一帧的bbox（数据量太大，LLM难以处理）
- 只返回出现的时间段和运动概述
- 如需某个时刻的精确bbox，可调用`detect_objects`

---

#### API-11: `query_entity_state`

**功能**：查询实体在特定时刻的状态

**输入参数**：
```json
{
  "video_id": "string",
  "entity_id": "ent_001",
  "timestamp": 46.5,
  "query": "What is she holding?"  // 状态查询（自然语言）
}
```

**输出**：
```json
{
  "answer": "She is holding a wooden spoon in her right hand.",
  "confidence": 0.87,
  "bbox": [0.32, 0.15, 0.25, 0.70],
  "timestamp": 46.5
}
```

**query示例**：
- "What is he holding?"
- "Is she sitting or standing?"
- "What color is his shirt?"
- "What is he looking at?"
- "What expression does she have?"

**设计理由**：
- 实体的状态是动态变化的，需要时间点 + 实体ID的双重定位
- 开放式查询支持灵活的问题类型
- 相比通用的`describe_visual`，聚焦于特定实体更精确

**与`describe_visual`的区别**：
- `describe_visual(start, end)`: 描述整个画面的所有内容
- `query_entity_state(entity_id, timestamp, query)`: 只关注特定实体，忽略其他内容

---

#### API-12: `spatial_relation`

**功能**：查询两个实体在某时刻的空间关系

**输入参数**：
```json
{
  "video_id": "string",
  "entity_id_1": "ent_001",
  "entity_id_2": "ent_002",
  "timestamp": 46.5,
  "relation_type": null        // 可选：指定关系类型
}
```

**输出**：
```json
{
  "timestamp": 46.5,
  "entities": {
    "entity_1": {
      "entity_id": "ent_001",
      "bbox": [0.32, 0.15, 0.25, 0.70]
    },
    "entity_2": {
      "entity_id": "ent_002",
      "bbox": [0.65, 0.42, 0.15, 0.35]
    }
  },
  "relations": {
    "direction": "entity_1 is to the left of entity_2",
    "distance": "far",                   // "close" | "medium" | "far"
    "distance_normalized": 0.72,         // 归一化距离 [0, 1]
    "interaction": "no_interaction",     // "no_interaction" | "touching" | "holding" | ...
    "relative_size": "entity_1 is larger than entity_2"
  },
  "confidence": 0.85
}
```

**relation_type说明**：
- `"direction"`: 方向关系（left/right/above/below）
- `"distance"`: 距离关系（close/medium/far）
- `"interaction"`: 交互关系（touching/holding/looking_at/...）
- `null`: 返回所有关系类型

**设计理由**：
- 空间关系对某些问题至关重要（"A把东西递给了B"、"谁离门更近"）
- 系统计算空间关系比让LLM从bbox推理更可靠
- 支持不同类型的关系查询
- 可用于验证事件（如"A和B拥抱了"需要close距离 + touching交互）

---

### 3.4 Retrieval API（检索类）

检索类API使LLM能够在长视频中快速定位相关内容，是高效视频探索的关键。

---

#### API-13: `temporal_search`

**功能**：根据文本描述在视频中检索相关时间段

**输入参数**：
```json
{
  "video_id": "string",
  "query": "person enters a room",     // 文本查询
  "top_k": 10,                         // 返回最相关的k个结果
  "time_range": null,                  // 可选：限制检索范围
  "modality": "visual"                 // 检索模态
}
```

**输出**：
```json
{
  "query": "person enters a room",
  "results": [
    {
      "segment_id": "seg_042",
      "start_time": 45.2,
      "end_time": 48.6,
      "duration": 3.4,
      "confidence": 0.89,
      "matched_modality": "visual"     // 匹配的模态
    },
    {
      "segment_id": "seg_087",
      "start_time": 102.3,
      "end_time": 105.1,
      "duration": 2.8,
      "confidence": 0.76,
      "matched_modality": "visual"
    }
  ],
  "search_time_ms": 45.2               // 检索耗时
}
```

**modality参数说明**：
- `"visual"`: 仅基于视觉内容检索（使用CLIP等视觉-语言模型）
- `"audio"`: 仅基于音频内容检索（基于字幕文本匹配）
- `"multimodal"`: 多模态融合检索（综合视觉和音频）

**time_range说明**：
- 如不指定，在全视频范围内检索
- 可指定范围以实现"先粗后精"的检索策略

**设计理由**：
- 这是最核心的"定位"操作，几乎所有任务都需要先定位再分析
- 返回多个候选而非单个，让LLM可以进一步筛选验证
- confidence评分帮助LLM判断是否需要更多探索
- 支持局部检索，配合分层策略提高效率

**典型使用场景**：
```
问题："视频中第一次出现猫是什么时候？"

LLM推理：
1. temporal_search(query="cat", top_k=10)
   → 获得10个候选片段，置信度0.65-0.92
2. 对置信度最高的3个候选：
   describe_visual(start, end, focus="animals")
   → 验证是否真的有猫
3. 找到第一个确认有猫的片段
4. write_memory("First cat appearance at 45.2s")
```

---

#### API-14: `find_similar_segments`

**功能**：找到与给定片段视觉或语义相似的其他片段

**输入参数**：
```json
{
  "video_id": "string",
  "reference_segment": {
    "start_time": 45.2,
    "end_time": 48.6
  },
  "similarity_type": "semantic",       // 相似性类型
  "top_k": 10,
  "time_range": null                   // 可选：限制检索范围
}
```

**输出**：
```json
{
  "reference_segment": {
    "start_time": 45.2,
    "end_time": 48.6,
    "duration": 3.4
  },
  "similar_segments": [
    {
      "segment_id": "seg_087",
      "start_time": 102.3,
      "end_time": 105.8,
      "duration": 3.5,
      "similarity_score": 0.85,
      "similarity_explanation": "Both show cooking activities in kitchen setting"
    },
    {
      "segment_id": "seg_124",
      "start_time": 145.1,
      "end_time": 148.2,
      "duration": 3.1,
      "similarity_score": 0.78,
      "similarity_explanation": "Similar person and setting, different activity"
    }
  ]
}
```

**similarity_type说明**：
- `"visual"`: 视觉相似（颜色分布、纹理、构图相似）
  - 适用于：找到视觉风格相似的片段
- `"semantic"`: 语义相似（内容、场景类型、动作类型相似）
  - 适用于：找到语义上相关的片段（如"所有做饭的片段"）
- `"motion"`: 运动模式相似（相机运动、对象运动相似）
  - 适用于：找到动态特征相似的片段

**设计理由**：
- 支持"找到类似事件"类的问题（"这个动作重复了几次？"）
- 基于示例检索比文本查询更精确（"找到所有和这段类似的片段"）
- 不同similarity_type适应不同的任务需求
- similarity_explanation帮助LLM理解匹配原因

**典型使用场景**：
```
问题："视频中人物投篮了几次？"

LLM推理：
1. temporal_search(query="person shooting basketball")
   → 找到第一个投篮片段 seg_042
2. find_similar_segments(
     reference=seg_042,
     similarity_type="semantic",
     top_k=20
   )
   → 找到所有语义相似的片段（可能都是投篮动作）
3. 对每个候选片段验证：
   recognize_activity(start, end)
   → 确认是否是投篮动作
4. 统计确认的投篮次数
```

---

### 3.5 Memory API（记忆类）

记忆类API为LLM提供外部记忆机制，解决长视频推理中的上下文窗口限制问题。

---

#### API-15: `write_memory`

**功能**：将信息写入外部记忆

**输入参数**：
```json
{
  "video_id": "string",
  "memory_type": "observation",        // 记忆类型
  "content": "Target woman (ent_001) first appeared at 45.2s, entering kitchen",
  "time_range": {                      // 可选：相关的时间范围
    "start_time": 45.2,
    "end_time": 48.6
  },
  "related_entities": ["ent_001"],     // 可选：相关的实体ID
  "importance": 0.9                    // 可选：重要性评分 [0, 1]
}
```

**输出**：
```json
{
  "memory_id": "mem_001",
  "success": true,
  "created_at": 1637654321.5           // 创建时间戳
}
```

**memory_type说明**：
- `"observation"`: 观察到的事实（直接来自API返回）
  - 例："seg_042包含一个女性在做饭"
- `"inference"`: LLM的推理结论（基于多个观察的推理）
  - 例："女性角色是视频的主角"
- `"hypothesis"`: 待验证的假设（需要后续验证）
  - 例："女性可能在准备晚餐"
- `"answer"`: 对问题的答案或结论
  - 例："第一次出现猫是在45.2秒"

**importance说明**：
- 0.0-0.3: 低重要性（细节信息）
- 0.4-0.7: 中等重要性（有用信息）
- 0.8-1.0: 高重要性（关键信息）

**设计理由**：
- LLM需要显式的记忆机制来处理长视频（上下文窗口有限）
- memory_type帮助区分事实与推理，便于后续检索和推理
- 元数据（时间、实体、重要性）支持智能检索
- importance评分可用于记忆管理（如容量限制时优先保留重要记忆）

**典型使用场景**：
```
LLM执行长视频任务时：

1. 每次发现关键信息，立即写入记忆：
   write_memory(
     type="observation",
     content="ent_001 appeared in kitchen at 45.2s",
     importance=0.8
   )

2. 基于多个观察做出推理后，记录推理结果：
   write_memory(
     type="inference",
     content="ent_001 is preparing a meal for guests",
     importance=0.9
   )

3. 最终答案也可存入记忆：
   write_memory(
     type="answer",
     content="The protagonist prepared dinner between 45-68s",
     importance=1.0
   )
```

---

#### API-16: `read_memory`

**功能**：从记忆中检索相关信息

**输入参数**：
```json
{
  "video_id": "string",
  "query": "when did the woman first appear",  // 检索查询
  "memory_type": null,                         // 可选：限制记忆类型
  "top_k": 5,                                  // 返回结果数
  "time_range": null,                          // 可选：限制时间范围
  "min_importance": 0.0                        // 可选：最低重要性阈值
}
```

**输出**：
```json
{
  "query": "when did the woman first appear",
  "memories": [
    {
      "memory_id": "mem_001",
      "content": "Target woman (ent_001) first appeared at 45.2s, entering kitchen",
      "memory_type": "observation",
      "time_range": {
        "start_time": 45.2,
        "end_time": 48.6
      },
      "related_entities": ["ent_001"],
      "importance": 0.9,
      "relevance": 0.95,                       // 与查询的相关性 [0, 1]
      "created_at": 1637654321.5
    },
    {
      "memory_id": "mem_003",
      "content": "ent_001 is the main character, appears in 8 scenes",
      "memory_type": "inference",
      "importance": 0.85,
      "relevance": 0.78,
      "created_at": 1637654456.2
    }
  ],
  "total_matches": 2
}
```

**检索机制**：
- 基于语义相似度检索，而非精确文本匹配
- 综合考虑：语义相关性、重要性、时间相关性
- 返回完整的元数据帮助LLM判断记忆的可用性

**过滤参数说明**：
- `memory_type`: 只返回指定类型的记忆
- `time_range`: 只返回时间范围内相关的记忆
- `min_importance`: 只返回重要性≥阈值的记忆

**特殊用法**：
```json
// 列出所有记忆（用于全局回顾）
{
  "query": "*",
  "top_k": 100
}

// 只检索推理结论
{
  "query": "main character",
  "memory_type": "inference"
}

// 只检索高重要性记忆
{
  "query": "*",
  "min_importance": 0.8,
  "top_k": 20
}
```

**设计理由**：
- 语义检索比精确匹配更灵活（"第一次出现" vs "首次登场"都能匹配）
- relevance评分指示匹配质量，帮助LLM选择最相关的记忆
- 支持多种过滤方式，适应不同的检索需求
- 可用于"回顾"所有已收集的信息

**典型使用场景**：
```
LLM在推理过程中：

1. 需要回忆之前的发现：
   read_memory(query="when did the woman first appear")
   → 获取相关记忆，继续推理

2. 生成最终答案前，检查是否遗漏关键信息：
   read_memory(query="*", min_importance=0.8)
   → 回顾所有高重要性记忆

3. 验证一致性：
   read_memory(query="protagonist identity", memory_type="inference")
   → 检查之前的推理结论
```

---

## 四、完备性与最小性分析

### 4.1 完备性验证

通过5类典型视频理解任务验证原子操作集的完备性：

#### 任务1：定位+描述

**问题**："视频中第一次出现猫是什么时候，它在做什么？"

**操作序列**：
```
1. get_video_info(video_id)
   → 了解视频长度，规划策略

2. temporal_search(query="cat", top_k=10)
   → 获得10个候选片段

3. 对置信度最高的候选：
   describe_visual(start, end, focus="animals")
   → 验证是否真的有猫

4. 找到第一个确认有猫的片段：
   describe_visual(start, end, detail_level="detailed")
   → 获取详细描述

5. write_memory(
     type="answer",
     content="First cat appearance at 45.2s, playing with yarn"
   )
```

✅ **完全覆盖**

---

#### 任务2：计数

**问题**："视频中人物A和B拥抱了几次？"

**操作序列**：
```
1. temporal_search(query="person A and person B", top_k=20)
   → 粗定位A和B同时出现的片段

2. 对第一个候选片段：
   detect_objects(timestamp, category="person")
   → 识别两个人物

3. register_entity(timestamp, "person A") → ent_001
   register_entity(timestamp, "person B") → ent_002
   → 注册两个实体

4. track_entity(ent_001) 和 track_entity(ent_002)
   → 获取两人的出现时间段

5. 找到两人同时出现的时间段：
   对每个重叠时间段：
     spatial_relation(ent_001, ent_002, timestamp, "interaction")
     → 检测是否有拥抱交互

6. 每次检测到拥抱：
   write_memory(
     type="observation",
     content="Hug detected at timestamp X",
     related_entities=[ent_001, ent_002]
   )

7. read_memory(query="hug", memory_type="observation")
   → 统计拥抱次数
```

✅ **完全覆盖**

---

#### 任务3：时序推理

**问题**："在主角离开房间之前，他拿了什么东西？"

**操作序列**：
```
1. temporal_search(query="person leaves room", top_k=5)
   → 定位离开房间的时刻T

2. describe_visual(T-10, T)
   → 确认是主角离开房间

3. register_entity(T-5, "main character") → ent_001

4. query_entity_state(
     ent_001,
     timestamp=T-3,
     query="What is he holding?"
   )
   → 查询离开前持有的物品

5. write_memory(
     type="answer",
     content="Before leaving, he picked up his phone"
   )
```

✅ **完全覆盖**

---

#### 任务4：因果推理

**问题**："为什么角色A突然跑出房间？"

**操作序列**：
```
1. temporal_search(query="person runs out of room", top_k=5)
   → 定位事件时刻T

2. register_entity(T, "person A") → ent_001

3. 查看之前发生了什么：
   describe_visual(T-10, T, detail_level="detailed")
   → 获取视觉线索

4. get_transcript(time_range={T-10, T})
   → 获取对话内容

5. detect_audio_events(time_range={T-10, T})
   → 检测是否有特殊声音（如火警）

6. write_memory(
     type="observation",
     content="At T-5, fire alarm sound detected"
   )

7. write_memory(
     type="inference",
     content="Person A ran out because of fire alarm"
   )
```

✅ **完全覆盖**（LLM进行因果推理，API提供证据）

---

#### 任务5：多模态整合

**问题**："视频中谁说了'I love you'，当时他们在哪里？"

**操作序列**：
```
1. get_transcript()
   → 找到"I love you"出现的时间戳T

2. detect_objects(timestamp=T, category="person")
   → 识别画面中的人物

3. 如果有多人，结合说话人信息：
   get_transcript(time_range={T-1, T+1}, include_speaker_info=true)
   → 确定说话人

4. get_scene_attributes(timestamp=T, attributes=["location"])
   → 识别地点

5. write_memory(
     type="answer",
     content="Speaker_1 said 'I love you' at 45.2s in living_room"
   )
```

✅ **完全覆盖**

---

### 4.2 最小性分析

**是否有冗余操作？**

我们已经在设计过程中进行了精简：

1. ✅ 合并了`temporal_search`和`semantic_search`（功能重叠）
2. ✅ 合并了`read_memory`和`list_memories`（后者可通过前者实现）
3. ✅ 移除了`list_entities`，改为在必要时通过`track_entity`按需查询

**是否所有操作都必要？**

- **Video API (2个)**：必要，提供视频的基础数据和结构
- **Perception API (6个)**：必要，提供不同粒度和模态的感知
  - `describe_visual`：自然语言描述，灵活
  - `detect_objects`：结构化对象信息
  - `recognize_activity`：结构化动作信息
  - `get_scene_attributes`：结构化场景属性
  - `get_transcript`：音频-文本模态
  - `detect_audio_events`：音频事件
  - 这6个操作互补而非重叠
- **Entity Trace API (4个)**：必要，实体跟踪是核心能力
- **Retrieval API (2个)**：必要，长视频检索的两种主要方式
- **Memory API (2个)**：必要，外部记忆的读写

**结论**：当前16个原子操作是最小完备集合。

---

## 五、设计权衡讨论

### 5.1 为什么这样分类？

**按功能维度而非技术实现分类**：

- ❌ 不按技术分类（视觉模型API、语言模型API、检索API）
- ✅ 按LLM视角的功能分类（我需要做什么）

**5个类别的逻辑**：

1. **Video**: "我需要获取视频的原始数据和结构"
2. **Perception**: "我需要观察视频的内容"
3. **Entity Trace**: "我需要跟踪特定对象"
4. **Retrieval**: "我需要在视频中找到相关内容"
5. **Memory**: "我需要记住和回忆信息"

这种分类符合LLM的推理过程。

### 5.2 粒度权衡

**Video API为何只有2个？**

- `get_video_info`：最基础的元数据（raw metadata）
- `get_temporal_structure`：时间结构数据（raw temporal structure）
- 这两个API专注于提供视频的原始数据，不涉及语义理解
- 更高层的语义检索通过`temporal_search`（Retrieval）实现

**Perception API为何有6个？**

- 需要覆盖视觉（4个）和音频（2个）两个模态
- 视觉API提供不同结构的信息：
  - 自然语言（`describe_visual`）
  - 对象结构（`detect_objects`）
  - 动作结构（`recognize_activity`）
  - 场景属性（`get_scene_attributes`）
- 无法进一步合并而不损失功能或效率

**Retrieval API为何只有2个？**

- `temporal_search`：文本→视频检索（最常用）
- `find_similar_segments`：视频→视频检索（用于计数、模式发现）
- 这是检索的两种基本范式

### 5.3 输出设计原则

**返回LLM可直接使用的信息**：

✅ 好的设计：
```json
{
  "activities": ["cooking", "using_phone"],
  "primary_activity": "cooking"
}
```
→ LLM可以直接处理："The main activity is cooking"

❌ 差的设计：
```json
{
  "activity_vector": [0.89, 0.12, 0.03, ...]  // 400维向量
}
```
→ LLM无法处理向量

**提供置信度/相关性评分**：

帮助LLM判断结果的可靠性和进一步行动：
- confidence < 0.5 → 可能需要更多验证
- confidence > 0.9 → 可以直接采纳

**提供必要但不过量的结构化信息**：

平衡：
- 太少结构 → LLM需要大量解析
- 太多结构 → 输出冗长，LLM难以处理

### 5.4 有状态 vs 无状态

**有状态操作（2个）**：
- `register_entity` - 创建实体（修改系统状态）
- `write_memory` - 写入记忆（修改系统状态）

**无状态操作（14个）**：
- 所有其他操作都是纯查询

**为什么最小化有状态操作？**

- 有状态操作增加系统复杂度
- 需要管理状态的生命周期
- 需要处理并发和一致性
- 但实体和记忆是必要的状态（无法避免）

---

## 六、与Phase 1工具诱导的关系

### 6.1 原子操作是工具诱导的基础

在Phase 1中，LLM将这16个原子操作组合成高层工具。

**高层工具示例1**：`locate_first_appearance`

```json
{
  "tool_name": "locate_first_appearance",
  "description": "定位某个对象在视频中的首次出现",
  "inputs": {
    "description": "string"  // 如 "woman in red"
  },
  "outputs": {
    "start_time": "float",
    "end_time": "float",
    "confidence": "float"
  },
  "steps": [
    {
      "api": "temporal_search",
      "params": {
        "query": "{{description}}",
        "top_k": 10
      },
      "save_as": "candidates"
    },
    {
      "api": "describe_visual",
      "params": {
        "start_time": "{{candidates[0].start_time}}",
        "end_time": "{{candidates[0].end_time}}",
        "focus": "{{description}}"
      },
      "save_as": "verification"
    },
    {
      "condition": "{{description}} in {{verification.description}}",
      "then": {
        "api": "write_memory",
        "params": {
          "memory_type": "observation",
          "content": "First appearance of {{description}} at {{candidates[0].start_time}}s",
          "importance": 0.9
        }
      }
    }
  ]
}
```

**高层工具示例2**：`count_occurrences`

```json
{
  "tool_name": "count_occurrences",
  "description": "统计某个事件在视频中出现的次数",
  "inputs": {
    "event_description": "string",
    "example_segment": "object"  // 可选
  },
  "outputs": {
    "count": "int",
    "occurrences": "list"
  },
  "steps": [
    {
      "api": "find_similar_segments",
      "params": {
        "reference_segment": "{{example_segment}}",
        "similarity_type": "semantic",
        "top_k": 50
      },
      "save_as": "candidates"
    },
    {
      "for_each": "candidates",
      "do": {
        "api": "recognize_activity",
        "params": {
          "start_time": "{{item.start_time}}",
          "end_time": "{{item.end_time}}"
        },
        "save_as": "activity_{{index}}"
      }
    },
    {
      "aggregate": "filter and count matching activities",
      "save_as": "count"
    }
  ]
}
```

### 6.2 原子操作设计影响工具诱导质量

**如果原子操作设计不好，会导致**：

1. **粒度太粗** → 高层工具缺乏灵活性
   - 例：如果只有`find_and_count_events(description)`
   - 无法组合出`locate_first_appearance`

2. **粒度太细** → 高层工具需要组合大量操作
   - 例：如果只有`get_frame_pixels(timestamp)`
   - 需要LLM自己调用视觉理解模型

3. **缺乏正交性** → LLM难以选择合适的操作
   - 例：如果有5个相似的检索API
   - LLM不知道该用哪个

**我们的设计保证**：

1. ✅ **正交性**：16个操作在5个维度上功能独立
2. ✅ **可组合性**：操作输出可作为其他操作的输入
3. ✅ **完备性**：覆盖视频理解的所有基础能力
4. ✅ **适中粒度**：既不太细也不太粗，适合组合

---

## 七、实现建议

### 7.1 预计算 vs 实时计算

**预计算（离线处理）**：
- `get_temporal_structure`：场景分割（TransNetV2）
- `temporal_search`的特征：CLIP嵌入、字幕索引
- `track_entity`的轨迹：视频跟踪（ByteTrack）
- `get_transcript`：ASR（Whisper）

**实时计算（按需）**：
- `describe_visual`：Video LLaVA（每次调用~1-2秒）
- `detect_objects` with query：开放词汇检测（Grounding DINO）
- `query_entity_state`：VQA模型
- `spatial_relation`：基于bbox计算（快）

**混合策略**：
- `detect_objects`无query：可预计算通用对象检测
- `recognize_activity`：可预计算常见动作分类

### 7.2 技术栈推荐

| API | 推荐模型/技术 |
|-----|-------------|
| get_temporal_structure | TransNetV2, PySceneDetect |
| describe_visual | Video-LLaVA, PLLaVA, VideoChat |
| detect_objects | YOLO-World, Grounding DINO |
| recognize_activity | VideoMAE, TimeSformer, Kinetics预训练 |
| get_scene_attributes | CLIP, 场景分类器 |
| get_transcript | Whisper (OpenAI) |
| detect_audio_events | PANNs, VGGSound |
| temporal_search | CLIP特征 + FAISS索引 |
| find_similar_segments | 视频特征 + 相似度计算 |
| track_entity | ByteTrack, DeepSORT |
| spatial_relation | bbox几何计算 + 交互检测 |
| Memory存储 | PostgreSQL + pgvector, Redis |

### 7.3 性能优化

**缓存策略**：
```python
# describe_visual结果缓存
cache_key = f"{video_id}:{start_time}:{end_time}:{detail_level}"
if redis.exists(cache_key):
    return redis.get(cache_key)
```

**批处理**：
```python
# 批量detect_objects
timestamps = [45.2, 46.3, 47.1, ...]
results = batch_detect_objects(video_id, timestamps)
```

**并行处理**：
```python
# 并行处理多个候选片段
with ThreadPoolExecutor() as executor:
    futures = [
        executor.submit(describe_visual, seg.start, seg.end)
        for seg in candidates
    ]
    results = [f.result() for f in futures]
```

---

## 八、总结

### 8.1 核心特性

1. **完备性**：16个原子操作覆盖视频理解的5个核心维度
2. **最小性**：每个操作都是必要的，无冗余
3. **正交性**：操作之间功能独立，组合灵活
4. **LLM友好**：接口清晰，参数合理，输出可直接使用
5. **可扩展性**：保留扩展空间，但核心集合稳定

### 8.2 API速查表

| 分类 | API名称 | 核心功能 |
|-----|---------|---------|
| **Video** | get_video_info | 获取视频元信息 |
| | get_temporal_structure | 获取场景切分 |
| **Perception** | describe_visual | 生成视觉描述 |
| | detect_objects | 对象检测 |
| | recognize_activity | 动作识别 |
| | get_scene_attributes | 场景属性 |
| | get_transcript | 语音转文本 |
| | detect_audio_events | 音频事件检测 |
| **Entity Trace** | register_entity | 注册实体 |
| | track_entity | 获取实体轨迹 |
| | query_entity_state | 查询实体状态 |
| | spatial_relation | 空间关系 |
| **Retrieval** | temporal_search | 时间段检索 |
| | find_similar_segments | 相似片段查找 |
| **Memory** | write_memory | 写入记忆 |
| | read_memory | 读取记忆 |

### 8.3 与整体框架的配合

```
原子操作（16个）
    ↓
    Phase 1: Tool Induction
    （LLM自动组合原子操作）
    ↓
高层工具库（~10-20个）
    ↓
    Phase 2: Tool-based Reasoning
    （LLM循环调用工具）
    ↓
视频理解任务完成
（QA, Reasoning, Summarization）
```

这16个原子操作是整个Video-Agent框架的基石。

---

**文档版本**：v2.0
**最后更新**：2025-11-25
**变更说明**：按照5类分类重新组织，优化API设计
