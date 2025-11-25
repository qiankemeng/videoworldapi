# 视频世界原子操作设计 v2.0（Atomic Operations Design v2.0）

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
6. **分层设计**：底层提供原始数据，上层提供语义理解

### 1.2 粒度平衡

- **太细**（如"读取第N帧的像素矩阵"）→ LLM难以有效组合，调用次数过多
- **太粗**（如"找到视频中的主角并总结其行为"）→ 失去组合灵活性，本质上是预定义pipeline
- **适中**（如"采样指定区间的关键帧"）→ 既是独立功能单元，又可灵活组合

---

## 二、原子操作分类体系

我们将原子操作分为5大类，共**18个**操作：

```
视频世界原子操作（18个）
│
├── 1. Video API（视频数据类）- 4个
│   ├── get_video_info         // 获取视频元信息
│   ├── get_temporal_structure // 获取时间结构
│   ├── sample_frames          // 采样视频帧
│   └── crop_region            // 裁切图像区域
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

视频数据类API提供视频的**原始数据**访问能力，包括元数据、时间结构、帧数据和图像处理。这是最底层的API，其他API可能在内部调用这些操作。

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
  "video_id": "string",
  "duration": 180.5,           // 总时长（秒）
  "fps": 30.0,                 // 帧率
  "resolution": {
    "width": 1920,
    "height": 1080
  },
  "aspect_ratio": "16:9",
  "has_audio": true,           // 是否有音频
  "audio_channels": 2,
  "audio_sample_rate": 48000,
  "num_frames": 5415,          // 总帧数
  "file_size_mb": 245.7,
  "codec": "h264",
  "bitrate_kbps": 5000
}
```

**设计理由**：
- 提供视频的完整元数据，帮助LLM规划后续操作
- 帧率和总帧数用于时间戳与帧号的转换
- 音频信息决定是否调用音频相关API
- 分辨率信息用于空间计算和采样策略

**典型使用场景**：
```
LLM: 调用 get_video_info()
     → 发现视频30fps, 180秒, 1920x1080
     → 决定采样策略：每2秒采样1帧，共90帧
```

---

#### API-2: `get_temporal_structure`

**功能**：获取视频的时间结构（场景切分）

**输入参数**：
```json
{
  "video_id": "string",
  "granularity": "coarse"      // 粒度: "coarse"(粗) | "fine"(细)
}
```

**输出**：
```json
{
  "video_id": "string",
  "granularity": "coarse",
  "segments": [
    {
      "segment_id": "seg_001",
      "start_time": 0.0,
      "end_time": 15.4,
      "start_frame": 0,
      "end_frame": 462,
      "duration": 15.4,
      "num_frames": 462,
      "type": "scene",         // "scene"(场景变化) | "shot"(镜头切换)
      "transition_type": "cut" // 可选：转场类型 "cut", "fade", "dissolve"
    },
    {
      "segment_id": "seg_002",
      "start_time": 15.4,
      "end_time": 32.1,
      "start_frame": 462,
      "end_frame": 963,
      "duration": 16.7,
      "num_frames": 501,
      "type": "scene",
      "transition_type": "fade"
    }
  ],
  "total_segments": 12
}
```

**设计理由**：
- 提供视频的时间结构划分，是探索视频的基础
- 同时返回时间戳和帧号，方便不同场景使用
- transition_type帮助理解场景转换方式
- segment_id可用于后续操作的引用

**典型使用场景**：
```
LLM: 调用 get_temporal_structure(granularity="coarse")
     → 发现12个大场景
     → 可以逐场景分析，而不是盲目扫描全视频
```

---

#### API-3: `sample_frames` ⭐ 新增

**功能**：从视频中采样帧，获取原始图像数据

**输入参数**：
```json
{
  "video_id": "string",
  "sample_method": "uniform",  // 采样方法
  "time_range": {              // 采样的时间范围
    "start_time": 0.0,
    "end_time": 180.5
  },
  "num_frames": 10,            // 采样帧数（与sample_interval二选一）
  "sample_interval": null,     // 可选：采样间隔（秒），如2.0表示每2秒采样1帧
  "resolution": {              // 可选：输出分辨率
    "width": 640,              // 如不指定，使用原始分辨率
    "height": 360
  },
  "format": "base64"           // 输出格式: "base64" | "url" | "frame_id"
}
```

**sample_method说明**：
- `"uniform"`: 均匀采样（等间隔）
- `"keyframe"`: 采样关键帧（I-frames）
- `"adaptive"`: 自适应采样（场景变化处密集采样）
- `"specific"`: 指定时间戳（需提供timestamps参数）

**输出**：
```json
{
  "video_id": "string",
  "frames": [
    {
      "frame_id": "frame_001",     // 帧的唯一标识
      "timestamp": 0.0,            // 时间戳（秒）
      "frame_number": 0,           // 帧号
      "resolution": {
        "width": 640,
        "height": 360
      },
      "image_data": "data:image/jpeg;base64,/9j/4AAQ...",  // 图像数据（根据format）
      "image_url": null,           // 如果format="url"
      "file_size_kb": 45.2
    },
    {
      "frame_id": "frame_002",
      "timestamp": 18.0,
      "frame_number": 540,
      "resolution": {
        "width": 640,
        "height": 360
      },
      "image_data": "data:image/jpeg;base64,/9j/4AAQ...",
      "file_size_kb": 48.1
    }
  ],
  "total_frames": 10,
  "sample_method": "uniform",
  "actual_interval": 18.0          // 实际采样间隔（秒）
}
```

**设计理由**：
- **提供原始帧数据访问**：这是Video API的核心能力，其他API可能在内部调用
- **灵活的采样策略**：支持多种采样方法，适应不同任务需求
- **分辨率可调**：降低分辨率减少数据传输和处理成本
- **多种输出格式**：
  - `base64`: 直接内嵌图像，适合多模态LLM（如GPT-4V, Claude）直接"看"
  - `url`: 图像URL引用，减少数据传输
  - `frame_id`: 仅返回帧ID，延迟加载

**与其他API的关系**：
- Perception API（如`describe_visual`, `detect_objects`）可以：
  - 选项1：接受时间戳，内部调用`sample_frames`
  - 选项2：接受`frame_id`，处理已采样的帧
  - 我们推荐选项1，因为更符合LLM使用习惯

**典型使用场景**：

**场景1：多模态LLM直接观察视频**
```
LLM（GPT-4V）推理：
1. sample_frames(num_frames=10, format="base64")
   → 获得10张base64图像
2. LLM直接"看"这10张图像
   → "我看到图像1-3是厨房场景，有一个穿红衣服的女性..."
3. 基于观察决定下一步行动
```

**场景2：高分辨率分析**
```
LLM推理：
1. temporal_search(query="car accident")
   → 找到候选时间段45.2-48.6s
2. sample_frames(
     time_range={45.2, 48.6},
     num_frames=30,
     resolution={1920, 1080},  # 原始高分辨率
     format="url"
   )
   → 获得该段的30帧高清图像URL
3. 调用外部视觉专家模型进行精细分析
```

**场景3：关键帧提取**
```
LLM推理：
1. get_temporal_structure(granularity="fine")
   → 获得所有镜头切换点
2. sample_frames(
     sample_method="keyframe",
     format="frame_id"
   )
   → 获得所有关键帧ID
3. 对每个关键帧调用 describe_visual()
   → 生成视频摘要
```

**性能考虑**：
- 采样10帧@640x360，base64格式：约500KB数据
- 如果num_frames过大（>50），建议使用format="frame_id"或"url"
- resolution降低4倍（1920→960），数据量降低约16倍

---

#### API-4: `crop_region` ⭐ 新增

**功能**：从图像/帧中裁切指定区域

**输入参数**：
```json
{
  "video_id": "string",
  "source": {                  // 源图像
    "type": "frame",           // "frame" | "timestamp" | "image_data"
    "frame_id": "frame_001",   // 如果type="frame"
    "timestamp": null,         // 如果type="timestamp"
    "image_data": null         // 如果type="image_data"（base64）
  },
  "regions": [                 // 裁切区域列表（支持批量）
    {
      "bbox": [0.32, 0.15, 0.25, 0.70],  // 归一化坐标 [x, y, width, height]
      "label": "person_1"      // 可选：区域标签
    },
    {
      "bbox": [0.65, 0.42, 0.15, 0.35],
      "label": "phone"
    }
  ],
  "output_resolution": {       // 可选：输出分辨率
    "width": 224,              // 如不指定，保持裁切区域的原始大小
    "height": 224
  },
  "padding": 0.1,              // 可选：边界扩展比例（0.1表示扩展10%）
  "format": "base64"           // 输出格式: "base64" | "url"
}
```

**输出**：
```json
{
  "video_id": "string",
  "source_info": {
    "frame_id": "frame_001",
    "timestamp": 45.2,
    "original_resolution": {
      "width": 1920,
      "height": 1080
    }
  },
  "cropped_regions": [
    {
      "region_id": "crop_001",
      "label": "person_1",
      "bbox": [0.32, 0.15, 0.25, 0.70],
      "bbox_pixels": [614, 162, 480, 756],  // 像素坐标
      "resolution": {
        "width": 224,
        "height": 224
      },
      "image_data": "data:image/jpeg;base64,/9j/4AAQ...",
      "image_url": null,
      "file_size_kb": 12.3
    },
    {
      "region_id": "crop_002",
      "label": "phone",
      "bbox": [0.65, 0.42, 0.15, 0.35],
      "bbox_pixels": [1248, 454, 288, 378],
      "resolution": {
        "width": 224,
        "height": 224
      },
      "image_data": "data:image/jpeg;base64,/9j/4AAQ...",
      "file_size_kb": 8.7
    }
  ],
  "total_regions": 2
}
```

**设计理由**：
- **支持细粒度区域分析**：聚焦于感兴趣的局部区域
- **支持批量裁切**：一次调用可裁切多个区域，提高效率
- **分辨率归一化**：统一输出大小，便于后续处理（如分类器输入）
- **padding参数**：避免裁切过紧，包含上下文信息
- **多种源类型**：可以从已采样的帧、指定时间戳或直接的图像数据裁切

**与其他API的关系**：
- `detect_objects` → 返回bbox → `crop_region` → 提取对象ROI
- `register_entity` → 定位实体 → `crop_region` → 提取实体外观
- `query_entity_state` → 可能内部使用`crop_region`聚焦实体

**典型使用场景**：

**场景1：提取实体外观特征**
```
LLM推理：
1. detect_objects(timestamp=45.2, query="person in red")
   → 获得bbox [0.32, 0.15, 0.25, 0.70]
2. crop_region(
     source={type: "timestamp", timestamp: 45.2},
     regions=[{bbox: [0.32, 0.15, 0.25, 0.70], label: "target"}],
     output_resolution={224, 224}
   )
   → 获得该人物的裁切图像
3. 保存到记忆或用于后续识别
```

**场景2：细粒度物体识别**
```
LLM推理：
问题："女性手里拿的是什么？"

1. register_entity(timestamp=45.2, description="woman") → ent_001
2. detect_objects(timestamp=45.2, category="person")
   → 获得女性的bbox
3. 手部区域估计：bbox的右下角小区域
   hand_bbox = [0.45, 0.38, 0.08, 0.12]
4. crop_region(
     source={type: "timestamp", timestamp: 45.2},
     regions=[{bbox: hand_bbox, label: "hand_region"}],
     output_resolution={512, 512},
     padding=0.2  # 扩展20%包含上下文
   )
   → 获得手部区域高清图像
5. 将裁切图像送入多模态LLM或专业识别模型
   → "A wooden spoon"
```

**场景3：批量对象分析**
```
LLM推理：
1. detect_objects(timestamp=45.2)
   → 检测到5个对象，获得5个bbox
2. crop_region(
     source={type: "timestamp", timestamp: 45.2},
     regions=[5个bbox],
     output_resolution={224, 224}
   )
   → 一次性获得5个对象的裁切图像
3. 对每个裁切图像进行分类或识别
```

**性能考虑**：
- 批量裁切比多次单独调用更高效
- 输出分辨率设为224x224（ImageNet标准），便于使用预训练模型
- padding可以包含上下文，提高识别准确率

---

### 3.2 Perception API（感知类）

感知类API对视频内容进行语义理解，将视觉和音频信息转换为LLM可理解的文本描述或结构化数据。

**设计说明**：
- Perception API接受**时间戳**或**时间范围**作为输入（而非frame_id）
- 内部会调用Video API的`sample_frames`进行采样（对LLM透明）
- 这样的设计更符合LLM的使用习惯（LLM思考的是"45秒时发生了什么"，而非"frame_1234是什么"）

---

#### API-5: `describe_visual`

**功能**：生成视频片段的自然语言描述

**输入参数**：
```json
{
  "video_id": "string",
  "start_time": 45.2,          // 起始时间（秒）
  "end_time": 48.6,            // 结束时间（秒）
  "detail_level": "standard",  // 描述详细程度: "brief" | "standard" | "detailed"
  "focus": null,               // 可选：聚焦方面 "people" | "actions" | "objects" | "scene"
  "frame_source": null         // 可选：指定使用的frame_id列表（高级用法）
}
```

**输出**：
```json
{
  "description": "A woman in a red apron is cooking at a stove in a modern kitchen. She stirs a pot with a wooden spoon while occasionally glancing at her phone on the counter.",
  "confidence": 0.91,
  "time_range": {
    "start_time": 45.2,
    "end_time": 48.6
  },
  "frames_analyzed": [
    {"frame_id": "frame_045", "timestamp": 45.2},
    {"frame_id": "frame_046", "timestamp": 46.4},
    {"frame_id": "frame_048", "timestamp": 48.6}
  ],
  "num_frames_analyzed": 3
}
```

**内部实现说明**（对LLM透明）：
```python
# 伪代码
def describe_visual(video_id, start_time, end_time, detail_level, focus):
    # 1. 根据时间范围和detail_level决定采样策略
    if detail_level == "brief":
        num_frames = 1  # 仅采样中间帧
    elif detail_level == "standard":
        num_frames = 3  # 采样首、中、尾
    else:  # detailed
        num_frames = 8  # 密集采样

    # 2. 调用sample_frames获取帧
    frames = sample_frames(
        video_id=video_id,
        time_range={start_time, end_time},
        num_frames=num_frames,
        resolution={width: 768, height: 432},  # 适中分辨率
        format="base64"
    )

    # 3. 调用Video LLM生成描述
    description = video_llm.caption(frames, focus=focus)

    return description
```

**设计理由**：
- LLM不需要关心采样细节，只需指定时间范围
- 不同detail_level自动调整采样密度
- 仍然返回采样信息（frames_analyzed），便于调试和追溯

---

#### API-6: `detect_objects`

**功能**：检测指定时间点画面中的对象

**输入参数**：
```json
{
  "video_id": "string",
  "timestamp": 45.2,           // 时间点（秒）
  "category": null,            // 可选：目标类别 "person" | "vehicle" | "animal" | ...
  "query": null,               // 可选：自然语言查询，如"person wearing red"
  "confidence_threshold": 0.3,
  "frame_source": null         // 可选：指定使用的frame_id（高级用法）
}
```

**输出**：
```json
{
  "timestamp": 45.2,
  "frame_info": {
    "frame_id": "frame_045",
    "frame_number": 1356
  },
  "objects": [
    {
      "object_id": "obj_tmp_001",    // 临时ID（仅在此帧有效）
      "category": "person",
      "bbox": [0.32, 0.15, 0.25, 0.70],  // 归一化坐标
      "bbox_pixels": [614, 162, 480, 756],
      "confidence": 0.94,
      "attributes": {
        "clothing_color": "red",
        "age_group": "adult",
        "gender": "female"
      }
    },
    {
      "object_id": "obj_tmp_002",
      "category": "phone",
      "bbox": [0.52, 0.35, 0.06, 0.10],
      "bbox_pixels": [998, 378, 115, 108],
      "confidence": 0.87
    }
  ]
}
```

**内部实现说明**：
```python
def detect_objects(video_id, timestamp, category, query, confidence_threshold):
    # 1. 获取该时间戳的帧
    frames = sample_frames(
        video_id=video_id,
        sample_method="specific",
        timestamps=[timestamp],
        resolution="original",  # 使用原始分辨率以提高检测精度
        format="base64"
    )
    frame = frames[0]

    # 2. 运行目标检测
    if query:
        # 开放词汇检测（Grounding DINO等）
        detections = open_vocab_detector.detect(frame, text_query=query)
    elif category:
        # 基于类别的检测
        detections = detector.detect(frame, category=category)
    else:
        # 通用检测
        detections = detector.detect(frame)

    # 3. 过滤低置信度结果
    detections = [d for d in detections if d.confidence >= confidence_threshold]

    return detections
```

---

#### API-7: `recognize_activity`

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
      "start_time": 45.2,
      "end_time": 48.1
    },
    {
      "label": "using_phone",
      "confidence": 0.76,
      "start_time": 46.5,
      "end_time": 48.6
    }
  ],
  "primary_activity": "cooking",
  "frames_analyzed": [
    {"timestamp": 45.2},
    {"timestamp": 46.4},
    {"timestamp": 47.6},
    {"timestamp": 48.6}
  ]
}
```

---

#### API-8: `get_scene_attributes`

**功能**：获取场景的环境属性

**输入参数**：
```json
{
  "video_id": "string",
  "timestamp": 45.2,
  "attributes": ["location", "lighting", "time_of_day"]  // 可选
}
```

**输出**：
```json
{
  "timestamp": 45.2,
  "location": "indoor_kitchen",
  "lighting": "artificial_bright",
  "time_of_day": "daytime",
  "weather": null,
  "camera_angle": "medium_shot",
  "camera_motion": "static"
}
```

---

#### API-9: `get_transcript`

**功能**：获取语音转文本结果

**输入参数**：
```json
{
  "video_id": "string",
  "time_range": {
    "start_time": 45.0,
    "end_time": 60.0
  },
  "include_speaker_info": false
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
      "speaker_id": "speaker_1",
      "confidence": 0.92
    }
  ]
}
```

---

#### API-10: `detect_audio_events`

**功能**：检测非语音音频事件

**输入参数**：
```json
{
  "video_id": "string",
  "time_range": {
    "start_time": 45.0,
    "end_time": 60.0
  },
  "event_types": null
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
      "confidence": 0.91
    }
  ]
}
```

---

### 3.3 Entity Trace API（实体追踪类）

（保持原设计，共4个API）

#### API-11: `register_entity`
#### API-12: `track_entity`
#### API-13: `query_entity_state`
#### API-14: `spatial_relation`

（详细规范与v1相同，此处省略）

---

### 3.4 Retrieval API（检索类）

（保持原设计，共2个API）

#### API-15: `temporal_search`
#### API-16: `find_similar_segments`

（详细规范与v1相同，此处省略）

---

### 3.5 Memory API（记忆类）

（保持原设计，共2个API）

#### API-17: `write_memory`
#### API-18: `read_memory`

（详细规范与v1相同，此处省略）

---

## 四、Video API 深度分析

### 4.1 为什么需要 sample_frames 和 crop_region？

**核心理由**：**将视频视为可访问的数据源，而非黑盒**

传统设计的问题：
```
❌ 旧设计：
LLM → describe_visual(45.2, 48.6) → 黑盒Video LLM → 描述文本
     （LLM看不到原始帧，完全依赖黑盒输出）
```

新设计的优势：
```
✅ 新设计：
LLM → sample_frames(45.2, 48.6) → 原始帧（base64图像）
    → LLM可以直接"看"帧（如果是GPT-4V/Claude）
    → 或者调用 describe_visual 获取语义描述
    → 或者调用 detect_objects 获取结构化信息
    （LLM有多种选择，更灵活）
```

**sample_frames 的价值**：

1. **支持多模态LLM直接观察**
   - GPT-4V、Claude 3.5可以直接"看"图像
   - 不需要通过中间的描述层

2. **支持自定义采样策略**
   - LLM可以根据任务需求调整采样密度
   - 关键时刻密集采样，非关键时刻稀疏采样

3. **支持外部专业模型**
   - 裁切后的图像可以送入专业模型（如医学影像分析、细粒度识别）

**crop_region 的价值**：

1. **支持局部区域分析**
   - 聚焦于感兴趣区域，避免背景干扰
   - "她手里拿的是什么" → 裁切手部区域 → 细粒度识别

2. **支持实体外观提取**
   - 注册实体时，提取实体外观作为"视觉签名"
   - 用于跨场景重识别

3. **支持批量处理**
   - 一次检测到5个对象 → 一次性裁切5个区域 → 批量分类

### 4.2 Video API 的两种使用模式

**模式1：直接使用（多模态LLM）**

```python
# GPT-4V的使用场景
LLM推理:
1. 调用 sample_frames(num_frames=10, format="base64")
2. LLM直接"看"10张图像（作为vision input）
3. LLM生成观察："I see a woman cooking in images 1-5, then..."
4. 基于观察规划下一步
```

优势：
- 充分利用多模态LLM的视觉能力
- 减少中间层损失
- LLM可以进行跨帧推理

**模式2：间接使用（通过Perception API）**

```python
# 传统LLM（如GPT-4 text-only）的使用场景
LLM推理:
1. 调用 describe_visual(45.2, 48.6)
   内部: sample_frames → Video LLM → 描述
2. LLM获得文本描述："A woman is cooking..."
3. 基于描述继续推理
```

优势：
- 适用于纯文本LLM
- Perception API封装了复杂性
- 更高效（不需要传输大量图像数据）

**两种模式的选择**：
- 如果LLM支持多模态 → 推荐模式1（更灵活）
- 如果LLM仅支持文本 → 使用模式2（更高效）
- 实际场景可能混合使用

### 4.3 Video API 的分辨率策略

不同API使用不同的分辨率：

| 用途 | 推荐分辨率 | 理由 |
|-----|-----------|------|
| 多模态LLM观察 | 640x360 - 768x432 | 平衡质量与数据量 |
| 对象检测 | 原始分辨率 | 需要高精度定位 |
| 视频描述 | 768x432 | 足够识别场景和对象 |
| 裁切后的对象识别 | 224x224 | 标准分类器输入 |
| 细粒度分析 | 1024x1024 | 需要看清细节 |

### 4.4 与其他API的协同

**sample_frames + describe_visual**：
```
sample_frames → 获得帧 → LLM"看"图像 → 快速判断
                       ↓（需要更详细描述时）
                       describe_visual → 专业Video LLM → 详细描述
```

**detect_objects + crop_region + 识别**：
```
detect_objects → 获得bbox → crop_region → 提取ROI → 专业识别模型
```

**sample_frames + 计数任务**：
```
sample_frames(num_frames=50) → LLM逐帧检查 → 统计出现次数
（适用于多模态LLM，避免调用大量API）
```

---

## 五、完整工作流示例

### 示例1：多模态LLM的工作流

**任务**："视频中穿红衣服的女性第一次拿起手机是什么时候？"

```
LLM（GPT-4V）推理过程：

Step 1: 了解视频结构
  → get_video_info()
  → 180秒，30fps，1080p

Step 2: 粗定位女性出现的片段
  → temporal_search(query="woman in red clothes", top_k=5)
  → 候选: seg_042(45.2-48.6s), seg_067(78.5-85.2s), ...

Step 3: 采样第一个候选片段的帧
  → sample_frames(
       time_range={45.2, 48.6},
       num_frames=8,
       resolution={768, 432},
       format="base64"
     )
  → 获得8张base64图像

Step 4: GPT-4V直接"看"这8张图像
  → GPT-4V observation: "我看到一个穿红色围裙的女性在厨房。
     在第1-3张图她在炒菜，第4张图她的手伸向右边，
     第5-8张图她手里拿着一个黑色的手机在看。"

Step 5: 确定精确时间点
  → 第5张图对应 timestamp=46.8s

Step 6: 注册实体并记录
  → register_entity(timestamp=46.8, description="woman in red")
  → write_memory(
       type="answer",
       content="Woman first picked up phone at 46.8s"
     )

答案: "46.8秒时"
```

---

### 示例2：细粒度对象识别

**任务**："视频45秒时，女性手里拿的是什么品牌的手机？"

```
LLM推理过程：

Step 1: 定位女性
  → detect_objects(timestamp=45.0, query="woman")
  → bbox: [0.32, 0.15, 0.25, 0.70]

Step 2: 采样高分辨率帧
  → sample_frames(
       time_range={45.0, 45.1},
       num_frames=1,
       resolution="original",  # 1920x1080
       format="base64"
     )

Step 3: 估计手部区域（基于人体姿态）
  → 女性bbox右下部分
  → hand_bbox = [0.52, 0.55, 0.10, 0.15]

Step 4: 裁切手部区域
  → crop_region(
       source={type: "timestamp", timestamp: 45.0},
       regions=[{bbox: hand_bbox, label: "hand"}],
       output_resolution={512, 512},
       padding=0.15  # 扩展15%包含更多上下文
     )
  → 获得手部高清裁切图像

Step 5: 多模态LLM识别手机品牌
  → GPT-4V看裁切图像
  → "这是一部iPhone，从背面的苹果logo和摄像头排列可以判断
     是iPhone 14 Pro"

答案: "iPhone 14 Pro"
```

---

### 示例3：批量场景分析

**任务**："总结视频中的主要场景"

```
LLM推理过程：

Step 1: 获取场景结构
  → get_temporal_structure(granularity="coarse")
  → 12个场景

Step 2: 为每个场景采样代表帧
  → 对每个scene:
      sample_frames(
        time_range={scene.start_time, scene.end_time},
        num_frames=1,  # 每个场景1帧
        sample_method="keyframe",
        resolution={640, 360},
        format="base64"
      )
  → 获得12张代表帧

Step 3: GPT-4V批量观察12张图像
  → "Scene 1 (0-15s): Kitchen, woman cooking
     Scene 2 (15-32s): Living room, woman on phone
     Scene 3 (32-45s): Kitchen again, setting table
     ..."

Step 4: 生成总结
  → write_memory(
       type="inference",
       content="Video shows a woman preparing dinner:
                cooking (scenes 1,3,7), setting table (3,8),
                answering phone calls (2,5), serving guests (10-12)"
     )
```

---

## 六、API数量与完备性分析

### 6.1 最终API列表（18个）

| ID | 类别 | API名称 | 功能 |
|----|------|---------|------|
| 1  | Video | get_video_info | 获取视频元信息 |
| 2  | Video | get_temporal_structure | 获取场景切分 |
| 3  | Video | sample_frames ⭐ | 采样视频帧 |
| 4  | Video | crop_region ⭐ | 裁切图像区域 |
| 5  | Perception | describe_visual | 生成视觉描述 |
| 6  | Perception | detect_objects | 对象检测 |
| 7  | Perception | recognize_activity | 动作识别 |
| 8  | Perception | get_scene_attributes | 场景属性 |
| 9  | Perception | get_transcript | 语音转文本 |
| 10 | Perception | detect_audio_events | 音频事件检测 |
| 11 | Entity Trace | register_entity | 注册实体 |
| 12 | Entity Trace | track_entity | 获取实体轨迹 |
| 13 | Entity Trace | query_entity_state | 查询实体状态 |
| 14 | Entity Trace | spatial_relation | 空间关系 |
| 15 | Retrieval | temporal_search | 时间段检索 |
| 16 | Retrieval | find_similar_segments | 相似片段查找 |
| 17 | Memory | write_memory | 写入记忆 |
| 18 | Memory | read_memory | 读取记忆 |

### 6.2 为什么是18个？

**从16个增加到18个**：
- 新增 `sample_frames`（Video API）
- 新增 `crop_region`（Video API）

**这两个API是否必要？**

✅ **sample_frames 是必要的**：
- 它是访问原始帧数据的唯一途径
- 支持多模态LLM直接观察视频
- Perception API可能在内部使用它
- 无法被其他API替代

✅ **crop_region 是必要的**：
- 它提供细粒度区域分析能力
- 支持"她手里拿的是什么"类细节问题
- 支持实体外观提取
- 虽然可以在Perception API内部实现，但作为独立API更灵活

**与最小性原则冲突吗？**

不冲突，因为：
1. 这两个API提供了**新的能力维度**（原始数据访问）
2. 它们是**原子的**（不可进一步拆解）
3. 它们是**正交的**（与其他API功能独立）

---

## 七、实现建议

### 7.1 Video API的实现

**sample_frames 实现**：
```python
def sample_frames(video_id, sample_method, time_range, num_frames, resolution, format):
    video = load_video(video_id)

    # 1. 确定采样时间点
    if sample_method == "uniform":
        timestamps = np.linspace(time_range.start, time_range.end, num_frames)
    elif sample_method == "keyframe":
        timestamps = extract_keyframe_timestamps(video, time_range)
    elif sample_method == "adaptive":
        timestamps = adaptive_sample(video, time_range, num_frames)

    # 2. 提取帧
    frames = []
    for ts in timestamps:
        frame = video.get_frame(ts)

        # 3. 调整分辨率
        if resolution:
            frame = resize(frame, resolution)

        # 4. 编码
        if format == "base64":
            image_data = encode_base64(frame)
        elif format == "url":
            image_url = save_and_get_url(frame)
        elif format == "frame_id":
            frame_id = save_frame(frame)

        frames.append({
            "frame_id": frame_id,
            "timestamp": ts,
            "image_data": image_data,
            ...
        })

    return frames
```

**crop_region 实现**：
```python
def crop_region(video_id, source, regions, output_resolution, padding, format):
    # 1. 获取源图像
    if source.type == "frame":
        image = load_frame(source.frame_id)
    elif source.type == "timestamp":
        frames = sample_frames(video_id, timestamps=[source.timestamp])
        image = frames[0]
    elif source.type == "image_data":
        image = decode_base64(source.image_data)

    # 2. 批量裁切
    cropped = []
    for region in regions:
        bbox = region.bbox  # 归一化坐标

        # 3. 转换为像素坐标
        h, w = image.shape[:2]
        x, y, rw, rh = bbox
        x_pix = int(x * w)
        y_pix = int(y * h)
        w_pix = int(rw * w)
        h_pix = int(rh * h)

        # 4. 应用padding
        if padding:
            pad = int(min(w_pix, h_pix) * padding)
            x_pix = max(0, x_pix - pad)
            y_pix = max(0, y_pix - pad)
            w_pix = min(w - x_pix, w_pix + 2*pad)
            h_pix = min(h - y_pix, h_pix + 2*pad)

        # 5. 裁切
        crop = image[y_pix:y_pix+h_pix, x_pix:x_pix+w_pix]

        # 6. 调整分辨率
        if output_resolution:
            crop = resize(crop, output_resolution)

        # 7. 编码
        if format == "base64":
            image_data = encode_base64(crop)

        cropped.append({
            "region_id": generate_id(),
            "label": region.label,
            "image_data": image_data,
            ...
        })

    return cropped
```

### 7.2 性能优化

**sample_frames 优化**：
- 使用FFmpeg高效解码指定帧
- 缓存常用分辨率的帧
- 批量解码连续帧（更快）

**crop_region 优化**：
- 批量裁切比多次单独调用快
- GPU加速resize操作
- 预计算常用bbox的裁切

---

## 八、总结

### 8.1 核心改进

**v2.0 相比 v1.0 的主要变化**：

1. ✅ **Video API 增强**（从2个→4个）
   - 新增 `sample_frames`：提供原始帧数据访问
   - 新增 `crop_region`：提供细粒度区域裁切

2. ✅ **更好的分层设计**
   - Video API：原始数据层（raw data）
   - Perception API：语义理解层（semantic understanding）
   - 两层分离，职责清晰

3. ✅ **支持多模态LLM**
   - GPT-4V/Claude可以直接"看"采样的帧
   - 不再完全依赖中间的描述层

4. ✅ **更灵活的使用方式**
   - 模式1：直接使用Video API（多模态LLM）
   - 模式2：通过Perception API（传统LLM）

### 8.2 最终API统计

- **总数**：18个原子操作
- **Video API**：4个（+2）
- **Perception API**：6个（不变）
- **Entity Trace API**：4个（不变）
- **Retrieval API**：2个（不变）
- **Memory API**：2个（不变）

### 8.3 设计原则验证

✅ **完备性**：覆盖视频理解的所有基础维度（原始数据+语义理解+实体+检索+记忆）
✅ **最小性**：每个操作都是必要的，18个是最小完备集
✅ **正交性**：操作之间功能独立，组合灵活
✅ **LLM友好**：接口清晰，支持多模态LLM
✅ **分层清晰**：Video API（数据层） + Perception API（理解层）

---

**文档版本**：v2.0
**最后更新**：2025-11-25
**主要变更**：增加sample_frames和crop_region，支持原始帧数据访问和多模态LLM
