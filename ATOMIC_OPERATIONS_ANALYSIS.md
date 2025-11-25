# 视频世界原子操作分析（Atomic Operations Analysis）

## 文档目标

本文档旨在系统性地分析和设计一套**最小完备**的原子操作集合，使LLM能够通过这些操作有效地访问和理解视频内容。这些原子操作将在Phase 1被LLM组合成高层工具，在Phase 2被LLM循环调用以完成复杂的视频推理任务。

---

## 一、原子操作的设计原则

### 1.1 什么是"原子操作"

原子操作是视频世界API的最底层接口，具有以下特征：

1. **不可再分性**：操作本身是一个完整的、不可进一步拆解的功能单元
2. **通用性**：不针对特定任务，而是提供通用的视频访问能力
3. **可组合性**：可以被组合成更复杂的操作流程
4. **LLM友好**：接口设计使LLM能够理解其语义并正确调用

### 1.2 完备性要求

原子操作集合应覆盖视频理解的所有基础维度：

- **时间维度**：视频的时间结构、导航、检索
- **空间维度**：帧内的视觉内容、对象、场景
- **语义维度**：内容的语义理解、描述、关联
- **实体维度**：对象的跟踪、状态、关系
- **记忆维度**：中间结果的存储、检索、整合

### 1.3 粒度平衡

原子操作的粒度需要平衡：

- **太细**（如"读取第N帧的像素"）→ LLM难以有效组合，调用次数过多
- **太粗**（如"找到视频中的主角并总结其行为"）→ 失去组合灵活性，本质上是预定义pipeline
- **适中**（如"在时间段T内检测类别C的对象"）→ 既是独立功能单元，又可灵活组合

---

## 二、视频理解能力分解

为了设计完备的原子操作集，我们首先分解视频理解任务所需的基础能力。

### 2.1 时间结构理解能力

**问题**：视频是时间序列数据，如何让LLM理解和导航视频的时间结构？

**需要的能力**：
1. **宏观结构感知**：了解视频的整体时间划分（场景、章节）
2. **时间点定位**：根据描述定位到特定时间点或时间段
3. **时间范围操作**：在指定时间范围内进行操作
4. **时序关系推理**：理解事件的先后、因果关系（由LLM推理，API提供时间信息支持）

**对应原子操作**：
- `get_video_info` - 获取视频基本信息（时长、帧率等）
- `get_temporal_structure` - 获取视频的时间结构（场景分割）
- `temporal_search` - 根据描述在全视频中检索时间段

### 2.2 视觉内容感知能力

**问题**：LLM本身无法"看"视频，需要通过API获取视觉信息的文本描述。

**需要的能力**：
1. **整体描述**：对一段视频内容的自然语言描述
2. **对象检测**：识别画面中的对象及其位置
3. **动作识别**：识别正在发生的动作/事件
4. **场景属性**：识别场景类型、环境属性

**对应原子操作**：
- `describe_visual` - 生成视频片段的视觉描述
- `detect_objects` - 检测指定时间点的对象
- `recognize_activity` - 识别时间段内的活动/动作
- `get_scene_attributes` - 获取场景属性（室内/室外、光照、天气等）

### 2.3 语义检索能力

**问题**：长视频中如何快速定位到与查询语义相关的内容？

**需要的能力**：
1. **跨模态检索**：用文本描述检索视频内容
2. **相似性检索**：找到视觉或语义相似的片段
3. **细粒度匹配**：不仅是关键词匹配，而是语义理解

**对应原子操作**：
- `semantic_search` - 语义检索视频片段
- `find_similar_segments` - 找到与给定片段相似的其他片段

### 2.4 实体跟踪能力

**问题**：视频中的对象（人、物体）会跨时间出现，如何建立跨时间的实体引用？

**需要的能力**：
1. **实体注册**：建立对感兴趣对象的持久引用
2. **轨迹查询**：查询实体在视频中何时何地出现
3. **状态查询**：查询实体在特定时刻的状态
4. **关系推理**：实体间的空间/语义关系（部分由LLM推理）

**对应原子操作**：
- `register_entity` - 注册一个实体并分配ID
- `track_entity` - 获取实体的时空轨迹
- `query_entity_state` - 查询实体在特定时刻的状态
- `spatial_relation` - 查询两个实体在某时刻的空间关系

### 2.5 音频信息获取能力

**问题**：视频包含音频轨道，音频信息对理解至关重要。

**需要的能力**：
1. **语音识别**：获取对话/旁白文本
2. **说话人识别**：识别谁在说话
3. **音频事件**：识别非语音声音（音乐、环境音等）

**对应原子操作**：
- `get_transcript` - 获取语音转文本结果
- `detect_audio_events` - 检测音频事件（掌声、音乐等）

### 2.6 记忆管理能力

**问题**：LLM的上下文窗口有限，长视频推理需要外部记忆支持。

**需要的能力**：
1. **信息存储**：将中间推理结果存储为记忆
2. **信息检索**：根据需要检索相关记忆
3. **层次组织**：支持不同粒度的记忆（帧、段、事件）
4. **记忆整合**：将多条记忆整合为更高层的摘要

**对应原子操作**：
- `write_memory` - 写入记忆
- `read_memory` - 读取记忆
- `list_memories` - 列出所有记忆（用于回顾）

### 2.7 元信息查询能力

**问题**：LLM需要了解当前状态、已有资源等元信息以规划下一步。

**需要的能力**：
1. **视频元数据**：视频的基本属性
2. **已注册实体**：当前会话中注册了哪些实体
3. **操作历史**：已经执行了哪些操作（可选）

**对应原子操作**：
- `get_video_info` - 获取视频元数据
- `list_entities` - 列出已注册的实体

---

## 三、原子操作完整集合设计

基于以上能力分解，我们设计以下原子操作集合。

### 分类体系

```
原子操作（18个）
├── 时间导航类（3个）
│   ├── get_video_info
│   ├── get_temporal_structure
│   └── temporal_search
├── 视觉感知类（4个）
│   ├── describe_visual
│   ├── detect_objects
│   ├── recognize_activity
│   └── get_scene_attributes
├── 语义检索类（2个）
│   ├── semantic_search
│   └── find_similar_segments
├── 实体跟踪类（4个）
│   ├── register_entity
│   ├── track_entity
│   ├── query_entity_state
│   └── spatial_relation
├── 音频感知类（2个）
│   ├── get_transcript
│   └── detect_audio_events
└── 记忆管理类（3个）
    ├── write_memory
    ├── read_memory
    └── list_memories
```

---

## 四、原子操作详细规范

### 4.1 时间导航类

#### OP-1: `get_video_info`

**功能**：获取视频的基本元信息

**输入参数**：
- `video_id` (string): 视频标识符

**输出**：
```json
{
  "duration": 180.5,           // 总时长（秒）
  "fps": 30.0,                 // 帧率
  "resolution": "1920x1080",   // 分辨率
  "has_audio": true,           // 是否有音频
  "num_frames": 5415           // 总帧数
}
```

**设计理由**：
- LLM需要知道视频总长度以规划探索策略
- 帧率信息用于时间戳与帧号的转换
- 音频信息决定是否调用音频相关操作

---

#### OP-2: `get_temporal_structure`

**功能**：获取视频的时间结构（场景切分）

**输入参数**：
- `video_id` (string): 视频标识符
- `granularity` (enum): 粒度级别
  - `"coarse"`: 粗粒度（大场景，如电影的不同地点）
  - `"fine"`: 细粒度（镜头级别）

**输出**：
```json
{
  "segments": [
    {
      "segment_id": "seg_001",
      "start_time": 0.0,
      "end_time": 15.4,
      "type": "scene_change"    // 类型：scene_change, shot_change
    },
    {
      "segment_id": "seg_002",
      "start_time": 15.4,
      "end_time": 32.1,
      "type": "scene_change"
    }
  ]
}
```

**设计理由**：
- 提供视频的"章节感"，避免LLM盲目搜索
- 粗粒度用于快速浏览，细粒度用于精确分析
- segment_id可用于后续操作的引用

**与高层工具的关系**：
- 高层工具可能组合此操作与`describe_visual`，生成"场景摘要"工具

---

#### OP-3: `temporal_search`

**功能**：根据文本描述在全视频中检索相关时间段

**输入参数**：
- `video_id` (string): 视频标识符
- `query` (string): 文本查询，如"person enters a room"
- `top_k` (int): 返回最相关的k个结果
- `modality` (enum, optional): 检索模态
  - `"visual"`: 仅视觉
  - `"audio"`: 仅音频（基于字幕）
  - `"both"`: 多模态融合

**输出**：
```json
{
  "results": [
    {
      "segment_id": "seg_042",
      "start_time": 45.2,
      "end_time": 48.6,
      "confidence": 0.89         // 匹配置信度
    },
    {
      "segment_id": "seg_087",
      "start_time": 102.3,
      "end_time": 105.1,
      "confidence": 0.76
    }
  ]
}
```

**设计理由**：
- 这是最核心的"定位"操作，几乎所有任务都需要先定位再分析
- 返回多个候选而非单个，让LLM可以进一步筛选
- confidence评分帮助LLM判断是否需要更多探索

**与高层工具的关系**：
- 可组合`temporal_search` + `describe_visual` + `detect_objects`形成"精确定位"工具

---

### 4.2 视觉感知类

#### OP-4: `describe_visual`

**功能**：生成视频片段的自然语言描述

**输入参数**：
- `video_id` (string): 视频标识符
- `start_time` (float): 起始时间（秒）
- `end_time` (float): 结束时间（秒）
- `detail_level` (enum): 描述详细程度
  - `"brief"`: 一句话概括（<20词）
  - `"standard"`: 标准描述（2-3句话）
  - `"detailed"`: 详细描述（>5句话，包含细节）
- `focus` (string, optional): 聚焦方面，如"people", "actions", "objects", "scene"

**输出**：
```json
{
  "description": "A woman in a red apron is cooking at a stove in a kitchen. She stirs a pot with a wooden spoon while looking at her phone.",
  "confidence": 0.91
}
```

**设计理由**：
- 这是LLM获取视觉信息的主要途径，将视觉转为文本
- 不同的detail_level适应不同的任务需求：
  - brief用于快速浏览大量片段
  - detailed用于回答需要细节的问题
- focus参数让LLM能引导描述的侧重点

**与其他操作的互补**：
- `describe_visual`提供整体理解
- `detect_objects`提供结构化的对象信息
- `recognize_activity`提供动作标签
- 三者结合可获得多角度的视觉信息

---

#### OP-5: `detect_objects`

**功能**：检测指定时间点画面中的对象

**输入参数**：
- `video_id` (string): 视频标识符
- `timestamp` (float): 时间点（秒）
- `category` (string, optional): 目标类别，如"person", "vehicle", "animal"
  - 如不指定，返回所有检测到的对象
- `query` (string, optional): 自然语言查询，如"person wearing red"
  - 支持开放词汇检测

**输出**：
```json
{
  "objects": [
    {
      "object_id": "obj_tmp_001",    // 临时ID（仅在此帧有效）
      "category": "person",
      "bbox": [0.32, 0.15, 0.25, 0.70],  // 归一化坐标 [x, y, w, h]
      "confidence": 0.94,
      "attributes": {                // 可选属性
        "clothing_color": "red",
        "pose": "standing"
      }
    },
    {
      "object_id": "obj_tmp_002",
      "category": "phone",
      "bbox": [0.45, 0.38, 0.08, 0.12],
      "confidence": 0.87
    }
  ]
}
```

**设计理由**：
- 提供结构化的对象信息（相比`describe_visual`的自由文本）
- bbox信息支持空间推理（"A在B的左边"）
- object_id可用于后续的`register_entity`
- category和query的灵活性支持不同的查询需求

**注意**：
- 此操作返回的object_id是临时的，仅在当前帧有效
- 如需跨帧跟踪，需使用`register_entity`建立持久实体

---

#### OP-6: `recognize_activity`

**功能**：识别视频片段中的活动/动作

**输入参数**：
- `video_id` (string): 视频标识符
- `start_time` (float): 起始时间
- `end_time` (float): 结束时间
- `granularity` (enum): 识别粒度
  - `"atomic"`: 原子动作（如"walking", "sitting"）
  - `"event"`: 事件级别（如"cooking a meal", "having a conversation"）

**输出**：
```json
{
  "activities": [
    {
      "label": "cooking",
      "confidence": 0.89,
      "start_time": 45.2,        // 可选：动作的精确起止时间
      "end_time": 48.1
    },
    {
      "label": "using_phone",
      "confidence": 0.76,
      "start_time": 46.5,
      "end_time": 48.6
    }
  ]
}
```

**设计理由**：
- 动作是视频理解的关键语义信息
- 返回多个动作支持并发动作场景
- 时间定位信息帮助LLM理解动作的时序关系

**与`describe_visual`的区别**：
- `recognize_activity`返回结构化的动作标签（易于程序处理）
- `describe_visual`返回自然语言描述（更灵活但需LLM解析）

---

#### OP-7: `get_scene_attributes`

**功能**：获取场景的环境属性

**输入参数**：
- `video_id` (string): 视频标识符
- `timestamp` (float): 时间点
- `attributes` (list[string], optional): 指定查询的属性类型
  - 可选值：`["location", "lighting", "weather", "time_of_day", "camera_angle"]`
  - 如不指定，返回所有可识别的属性

**输出**：
```json
{
  "location": "indoor_kitchen",
  "lighting": "artificial_bright",
  "time_of_day": "daytime",
  "camera_angle": "medium_shot"
}
```

**设计理由**：
- 场景属性对某些问题至关重要（"视频是白天还是晚上拍的？"）
- 与对象、动作信息互补，形成完整的场景理解
- 结构化输出便于LLM进行逻辑推理

---

### 4.3 语义检索类

#### OP-8: `semantic_search`

**功能**：基于语义相似度检索视频片段

**输入参数**：
- `video_id` (string): 视频标识符
- `query` (string): 语义查询
- `search_scope` (object, optional): 检索范围限制
  ```json
  {
    "start_time": 0.0,
    "end_time": 180.0
  }
  ```
- `top_k` (int): 返回结果数

**输出**：
```json
{
  "results": [
    {
      "segment_id": "seg_042",
      "start_time": 45.2,
      "end_time": 48.6,
      "relevance": 0.89
    }
  ]
}
```

**与`temporal_search`的区别**：
- `temporal_search`：快速、基于预计算的CLIP特征，全局检索
- `semantic_search`：更深层的语义理解，可在局部范围精确检索
- 实际上两者可以合并为一个操作，根据参数自动选择策略

**设计理由**：
- 语义检索是视频理解的核心能力
- 支持范围限制允许"先粗后精"的检索策略

---

#### OP-9: `find_similar_segments`

**功能**：找到与给定片段视觉或语义相似的其他片段

**输入参数**：
- `video_id` (string): 视频标识符
- `reference_segment` (object): 参考片段
  ```json
  {
    "start_time": 45.2,
    "end_time": 48.6
  }
  ```
- `similarity_type` (enum): 相似性类型
  - `"visual"`: 视觉相似（颜色、纹理、构图）
  - `"semantic"`: 语义相似（内容、场景类型）
  - `"motion"`: 运动模式相似
- `top_k` (int): 返回结果数

**输出**：
```json
{
  "similar_segments": [
    {
      "segment_id": "seg_087",
      "start_time": 102.3,
      "end_time": 105.8,
      "similarity": 0.85
    }
  ]
}
```

**设计理由**：
- 支持"找到类似事件"类的问题（"这个动作重复了几次？"）
- 不同similarity_type适应不同的任务需求
- 基于示例检索比文本查询更精确

**应用场景**：
- 计数任务："找到所有类似的投篮动作"
- 模式发现："找到所有类似的对话场景"

---

### 4.4 实体跟踪类

#### OP-10: `register_entity`

**功能**：为感兴趣的对象注册一个持久的实体ID

**输入参数**：
- `video_id` (string): 视频标识符
- `initial_timestamp` (float): 实体首次明确出现的时间点
- `description` (string): 实体描述，如"woman in red apron"
- `object_id` (string, optional): 来自`detect_objects`的临时对象ID
  - 如提供，直接使用该检测结果
  - 如不提供，系统根据description自动定位

**输出**：
```json
{
  "entity_id": "ent_001",
  "matched_bbox": [0.32, 0.15, 0.25, 0.70],
  "confidence": 0.91
}
```

**设计理由**：
- 视频中的对象会跨时间出现，需要持久的引用机制
- 实体ID是后续`track_entity`、`query_entity_state`的前提
- 描述式注册(description)比位置式注册(bbox)更符合LLM的使用习惯

**注意**：
- 一次对话中，entity_id在整个视频范围内唯一且持久
- 注册操作是有状态的（会修改系统状态）

---

#### OP-11: `track_entity`

**功能**：获取实体在视频中的时空轨迹

**输入参数**：
- `video_id` (string): 视频标识符
- `entity_id` (string): 实体ID
- `time_range` (object, optional): 限制查询的时间范围

**输出**：
```json
{
  "appearances": [
    {
      "start_time": 45.2,
      "end_time": 52.8,
      "segment_id": "seg_042",
      "trajectory_summary": "stationary in center"  // 运动模式概述
    },
    {
      "start_time": 78.5,
      "end_time": 85.2,
      "segment_id": "seg_067",
      "trajectory_summary": "moving left to right"
    }
  ],
  "total_duration": 14.7,
  "first_appearance": 45.2,
  "last_appearance": 85.2
}
```

**设计理由**：
- 回答"X何时出现"、"X出现了几次"等问题的基础
- trajectory_summary提供运动模式的抽象，避免返回大量bbox数据
- 时间范围限制支持局部查询

**输出简化**：
- 不返回每一帧的bbox（数据量太大）
- 只返回出现的时间段和运动概述
- 如需精确bbox，可再调用`detect_objects`

---

#### OP-12: `query_entity_state`

**功能**：查询实体在特定时刻的状态

**输入参数**：
- `video_id` (string): 视频标识符
- `entity_id` (string): 实体ID
- `timestamp` (float): 查询时间点
- `query` (string): 状态查询，如"What is she holding?", "Is he sitting or standing?"

**输出**：
```json
{
  "answer": "She is holding a wooden spoon in her right hand.",
  "confidence": 0.87,
  "bbox": [0.32, 0.15, 0.25, 0.70]
}
```

**设计理由**：
- 实体的状态是动态变化的，需要时间点 + 实体ID的双重定位
- 开放式查询支持灵活的问题类型
- 相比通用的`describe_visual`，聚焦于特定实体的状态更精确

**与`describe_visual`的区别**：
- `describe_visual`：描述整个画面
- `query_entity_state`：仅关注特定实体，忽略其他信息

---

#### OP-13: `spatial_relation`

**功能**：查询两个实体在某时刻的空间关系

**输入参数**：
- `video_id` (string): 视频标识符
- `entity_id_1` (string): 实体1的ID
- `entity_id_2` (string): 实体2的ID
- `timestamp` (float): 查询时间点
- `relation_type` (enum, optional): 关系类型
  - `"direction"`: 方向关系（left/right/above/below）
  - `"distance"`: 距离关系（near/far）
  - `"interaction"`: 交互关系（touching/holding/etc）

**输出**：
```json
{
  "relation": "entity_1 is to the left of entity_2",
  "distance": "close",              // "close" | "medium" | "far"
  "interaction": "no_interaction",   // 或 "holding", "touching" 等
  "confidence": 0.85
}
```

**设计理由**：
- 空间关系对某些问题至关重要（"A把东西递给了B"）
- 系统计算空间关系比让LLM从bbox推理更可靠
- 支持不同类型的关系查询

---

### 4.5 音频感知类

#### OP-14: `get_transcript`

**功能**：获取语音转文本结果

**输入参数**：
- `video_id` (string): 视频标识符
- `time_range` (object, optional): 时间范围
  ```json
  {
    "start_time": 45.0,
    "end_time": 60.0
  }
  ```
  - 如不指定，返回全视频字幕
- `include_speakers` (bool): 是否识别说话人（如果支持）

**输出**：
```json
{
  "transcript": [
    {
      "start_time": 46.2,
      "end_time": 49.5,
      "text": "Can you pass me the salt?",
      "speaker": "speaker_1",     // 可选
      "confidence": 0.92
    },
    {
      "start_time": 50.1,
      "end_time": 52.3,
      "text": "Sure, here you go.",
      "speaker": "speaker_2",
      "confidence": 0.89
    }
  ]
}
```

**设计理由**：
- 对话内容是视频理解的重要线索
- 时间对齐的字幕支持"谁在何时说了什么"的推理
- 说话人识别帮助建立对话结构

**应用场景**：
- 回答对话内容相关的问题
- 与视觉信息结合理解视听一致性

---

#### OP-15: `detect_audio_events`

**功能**：检测非语音音频事件

**输入参数**：
- `video_id` (string): 视频标识符
- `time_range` (object, optional): 时间范围
- `event_types` (list[string], optional): 指定检测的事件类型
  - 可选值：`["music", "applause", "laughter", "door_knock", "glass_breaking", ...]`

**输出**：
```json
{
  "audio_events": [
    {
      "event_type": "music",
      "start_time": 45.0,
      "end_time": 78.5,
      "confidence": 0.91
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

**设计理由**：
- 环境音对场景理解很重要（"有人敲门"、"背景音乐"）
- 补充视觉信息，形成多模态理解
- 事件类型枚举便于LLM处理

---

### 4.6 记忆管理类

#### OP-16: `write_memory`

**功能**：将信息写入外部记忆

**输入参数**：
- `video_id` (string): 视频标识符
- `memory_type` (enum): 记忆类型
  - `"observation"`: 观察到的事实（来自API返回）
  - `"inference"`: LLM的推理结论
  - `"hypothesis"`: 待验证的假设
- `content` (string): 记忆内容（自然语言）
- `time_range` (object, optional): 相关的时间范围
- `related_entities` (list[string], optional): 相关的实体ID
- `importance` (float, optional): 重要性评分 [0, 1]

**输出**：
```json
{
  "memory_id": "mem_001",
  "success": true
}
```

**设计理由**：
- LLM需要显式的记忆机制来处理长视频
- memory_type帮助区分事实与推理
- 元数据（时间、实体、重要性）支持后续检索

**使用场景**：
```
# LLM调用示例
1. temporal_search("woman enters room") -> 找到seg_042
2. describe_visual(seg_042) -> "A woman in red enters the living room"
3. write_memory(
     type="observation",
     content="Target woman first appeared at 45.2s, entering living room",
     time_range={45.2, 48.6},
     importance=0.9
   )
```

---

#### OP-17: `read_memory`

**功能**：从记忆中检索相关信息

**输入参数**：
- `video_id` (string): 视频标识符
- `query` (string): 检索查询
- `memory_type` (enum, optional): 限制记忆类型
- `top_k` (int): 返回结果数

**输出**：
```json
{
  "memories": [
    {
      "memory_id": "mem_001",
      "content": "Target woman first appeared at 45.2s, entering living room",
      "memory_type": "observation",
      "time_range": {"start_time": 45.2, "end_time": 48.6},
      "related_entities": ["ent_001"],
      "importance": 0.9,
      "relevance": 0.95       // 与查询的相关性
    }
  ]
}
```

**设计理由**：
- 基于语义相似度检索，而非精确匹配
- 返回完整的元数据帮助LLM判断记忆的可用性
- relevance评分指示匹配质量

**使用场景**：
```
# LLM需要回忆之前的发现
read_memory(query="when did the woman first appear")
-> 获取mem_001
-> 继续后续推理
```

---

#### OP-18: `list_memories`

**功能**：列出所有或特定类型的记忆（用于全局回顾）

**输入参数**：
- `video_id` (string): 视频标识符
- `memory_type` (enum, optional): 筛选记忆类型
- `sort_by` (enum): 排序方式
  - `"time"`: 按相关时间排序
  - `"importance"`: 按重要性排序
  - `"recency"`: 按写入时间排序

**输出**：
```json
{
  "memories": [
    {
      "memory_id": "mem_001",
      "content": "...",
      "memory_type": "observation",
      "importance": 0.9
    },
    {
      "memory_id": "mem_002",
      "content": "...",
      "memory_type": "inference",
      "importance": 0.8
    }
  ],
  "total_count": 2
}
```

**设计理由**：
- 允许LLM"回顾"已收集的所有信息
- 在生成最终答案前，检查是否遗漏关键信息
- 不同排序方式支持不同的回顾策略

---

## 五、完备性与最小性分析

### 5.1 完备性检验

我们通过典型的视频理解任务验证原子操作集的完备性：

**任务类型1：定位+描述**
- 问题："视频中第一次出现猫是什么时候，它在做什么？"
- 需要的操作：
  1. `temporal_search("cat")` - 定位
  2. `describe_visual(...)` - 描述
  3. `write_memory(...)` - 记录答案
- ✅ 覆盖

**任务类型2：计数**
- 问题："视频中人物A和B拥抱了几次？"
- 需要的操作：
  1. `temporal_search("person A")` - 粗定位
  2. `register_entity(...)` 两次 - 注册A和B
  3. `track_entity(...)` 两次 - 获取A和B的轨迹
  4. `spatial_relation(..., "interaction")` 在多个时间点查询 - 检测拥抱
  5. `write_memory(...)` - 记录每次拥抱
  6. `list_memories()` - 汇总计数
- ✅ 覆盖

**任务类型3：时序推理**
- 问题："在主角离开房间之前，他拿了什么东西？"
- 需要的操作：
  1. `temporal_search("person leaves room")` - 定位离开时间T
  2. `register_entity(...)` - 注册主角
  3. `track_entity(...)` - 确认主角何时在房间内
  4. `query_entity_state(..., "what is he holding", time=T-5)` - 查询离开前的状态
- ✅ 覆盖

**任务类型4：因果推理**
- 问题："为什么角色A突然跑出房间？"
- 需要的操作：
  1. `temporal_search("person runs out")` - 定位事件
  2. `describe_visual(before_time, event_time)` - 查看之前发生了什么
  3. `get_transcript(before_time, event_time)` - 查看对话
  4. `detect_audio_events(...)` - 查看是否有特殊声音（如警报）
  5. LLM推理因果关系
  6. `write_memory(type="inference", ...)` - 记录推理结论
- ✅ 覆盖

**任务类型5：多模态整合**
- 问题："视频中谁说了'I love you'，当时他们在哪里？"
- 需要的操作：
  1. `get_transcript(...)` - 找到该文本及时间
  2. `detect_objects(timestamp, category="person")` - 识别说话人
  3. `get_scene_attributes(timestamp, ["location"])` - 识别地点
- ✅ 覆盖

**结论**：18个原子操作能够支持主流视频理解任务的分解与执行。

### 5.2 最小性检验

是否有冗余操作？

**候选冗余1：`temporal_search` vs `semantic_search`**
- 分析：功能高度重合，但`temporal_search`侧重全局快速检索，`semantic_search`侧重局部精确检索
- 结论：可以合并为一个操作，通过参数区分模式
- **简化后：保留`temporal_search`，移除`semantic_search`**

**候选冗余2：`describe_visual` vs `query_entity_state`**
- 分析：`describe_visual`描述全局，`query_entity_state`描述特定实体
- 能否用`describe_visual` + LLM提取代替？理论可以，但效率低且容易出错
- 结论：保留两者

**候选冗余3：多个感知类操作**
- `describe_visual`、`detect_objects`、`recognize_activity`、`get_scene_attributes`是否可以合并？
- 分析：四者提供不同粒度和结构的信息，合并会导致：
  - 输出过于复杂（难以解析）
  - 计算浪费（不需要时也返回）
- 结论：保留分离设计

**候选冗余4：`list_memories` vs `read_memory`**
- `list_memories`可以通过`read_memory(query="*")`实现
- 但`list_memories`提供排序等功能
- 结论：可以合并，`read_memory`增加`list_all`参数
- **简化后：保留`read_memory`，移除`list_memories`**

**最终精简至16个原子操作**。

---

## 六、设计权衡讨论

### 6.1 粒度选择

**过细的反例**：
- ❌ `get_frame_at_timestamp(timestamp)` → 返回原始图像
  - 问题：LLM无法直接处理图像，必须再调用视觉理解API
  - 应该直接提供`describe_visual`等高层操作

**过粗的反例**：
- ❌ `find_and_describe_person(description)` → 找到人物并描述其所有行为
  - 问题：这已经是一个pipeline，失去了组合灵活性
  - 应该拆分为`temporal_search` + `register_entity` + `track_entity` + `describe_visual`等

### 6.2 返回内容的选择

**原则**：返回LLM可直接利用的信息，避免需要二次处理

**好的设计**：
- ✅ `recognize_activity` 返回动作标签列表
  - LLM可以直接使用："activities include cooking and talking"

**差的设计**：
- ❌ `detect_objects` 返回原始检测框图像
  - LLM无法处理图像，需要再调用描述API

### 6.3 有状态 vs 无状态

**有状态操作**（会修改系统状态）：
- `register_entity` - 创建实体
- `write_memory` - 写入记忆

**无状态操作**（纯查询）：
- 其他所有操作

**设计考虑**：
- 最小化有状态操作，降低系统复杂度
- 实体和记忆是必要的状态（无法避免）

### 6.4 可选参数的使用

**原则**：
- 必需参数：操作语义所必需的
- 可选参数：提供额外控制，但有合理默认值

**示例**：
```python
describe_visual(video_id, start_time, end_time,
                detail_level="standard",  # 可选，有默认值
                focus=None)               # 可选，默认全面描述
```

---

## 七、与Phase 1工具诱导的关系

### 7.1 原子操作是工具诱导的基础

Phase 1中，LLM将原子操作组合成高层工具。例如：

**高层工具示例：`locate_first_appearance`**
```json
{
  "tool_name": "locate_first_appearance",
  "description": "定位某个对象/人物在视频中的首次出现",
  "inputs": {
    "description": "string"  // 如"woman in red"
  },
  "atomic_operations": [
    {
      "op": "temporal_search",
      "params": {"query": "{{description}}", "top_k": 10}
    },
    {
      "op": "describe_visual",
      "params": {
        "start_time": "{{results[0].start_time}}",
        "end_time": "{{results[0].end_time}}",
        "focus": "{{description}}"
      }
    },
    {
      "op": "write_memory",
      "params": {
        "type": "observation",
        "content": "First appearance at {{results[0].start_time}}",
        "importance": 0.9
      }
    }
  ]
}
```

### 7.2 原子操作设计对工具诱导的影响

**原子操作的可组合性直接决定了高层工具的表达能力**：

- 如果原子操作粒度太粗 → 高层工具缺乏灵活性
- 如果原子操作粒度太细 → 高层工具需要组合大量操作，效率低
- 如果原子操作缺乏正交性（有功能重叠）→ LLM难以选择合适的操作

**我们的设计保证**：
1. **正交性**：每个原子操作专注于一个独立维度的能力
2. **可组合性**：原子操作之间可以通过数据流（一个操作的输出作为另一个的输入）自然连接
3. **完备性**：覆盖视频理解的所有基础维度

---

## 八、潜在扩展方向

虽然当前设计已经较为完备，但未来可根据需要扩展：

### 8.1 短期可能的扩展（保持原子性）

1. **OCR文本提取**
   - `extract_text(video_id, timestamp)` → 提取画面中的文字
   - 适用于包含大量文字信息的视频（新闻、演示文稿）

2. **人脸识别与情感**
   - `detect_faces(video_id, timestamp)` → 检测人脸及表情
   - `recognize_emotion(video_id, entity_id, timestamp)` → 识别情绪

3. **相机运动分析**
   - `get_camera_motion(video_id, start_time, end_time)` → 检测镜头运动（平移、缩放、旋转）
   - 适用于电影分析等任务

### 8.2 更高层的扩展（谨慎）

4. **时序关系判断**
   - `compare_temporal_order(event1, event2)` → 判断事件先后
   - 考虑：这可能由LLM基于`write_memory`的时间戳推理，不一定需要专门操作

5. **因果关系检测**
   - `detect_causality(event1, event2)` → 判断因果关系
   - 考虑：因果关系更适合由LLM推理，而非底层API

---

## 九、总结

### 9.1 最终原子操作集（17个）

合并冗余后，最终推荐的原子操作集：

| ID | 操作名 | 类别 | 功能 |
|----|--------|------|------|
| 1  | get_video_info | 元信息 | 获取视频基本信息 |
| 2  | get_temporal_structure | 时间导航 | 获取场景切分 |
| 3  | temporal_search | 时间导航 | 时间段检索 |
| 4  | describe_visual | 视觉感知 | 生成视觉描述 |
| 5  | detect_objects | 视觉感知 | 对象检测 |
| 6  | recognize_activity | 视觉感知 | 动作识别 |
| 7  | get_scene_attributes | 视觉感知 | 场景属性 |
| 8  | find_similar_segments | 语义检索 | 相似片段查找 |
| 9  | register_entity | 实体跟踪 | 注册实体 |
| 10 | track_entity | 实体跟踪 | 获取实体轨迹 |
| 11 | query_entity_state | 实体跟踪 | 查询实体状态 |
| 12 | spatial_relation | 实体跟踪 | 空间关系 |
| 13 | get_transcript | 音频感知 | 语音转文本 |
| 14 | detect_audio_events | 音频感知 | 音频事件检测 |
| 15 | write_memory | 记忆管理 | 写入记忆 |
| 16 | read_memory | 记忆管理 | 读取记忆 |
| 17 | list_entities | 元信息 | 列出已注册实体 |

### 9.2 关键设计特性

1. **完备性**：覆盖时间、空间、语义、实体、音频、记忆六大维度
2. **最小性**：每个操作都是必要的，无冗余
3. **正交性**：操作之间功能独立，组合灵活
4. **LLM友好**：接口语义清晰，参数合理，输出可直接使用
5. **可扩展性**：保留扩展空间，但核心集合稳定

### 9.3 与整体框架的配合

```
原子操作（17个）
    ↓ Phase 1: Tool Induction
高层工具库（~10-20个，由LLM自动生成）
    ↓ Phase 2: Tool-based Reasoning
视频理解任务（QA, Reasoning, Summarization）
```

这17个原子操作是整个框架的基石，它们的设计质量直接决定了：
- Phase 1能生成多有用的高层工具
- Phase 2能多有效地完成复杂推理任务

---

**文档版本**：v1.0
**最后更新**：2025-11-25
