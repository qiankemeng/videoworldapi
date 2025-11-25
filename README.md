

---

# 一、项目概述（Project Overview）

## 1.1 项目愿景

在长视频理解任务中，不再把视频当作“一次性输入给 MLLM 的帧序列”，
而是当作一个可以通过 **视频世界 API（Video World API）** 进行交互的环境：

* LLM 本身**不直接看像素**，
* 视频首先被各种视觉小模型处理为可检索、可查询、可记忆的“视频世界”；
* LLM 通过 **原子视频操作（world API） + 由 LLM 自动诱导的高层工具库**
  在这个世界里“移动、观察、检索、记忆、推理”。

我们采用一个严格 **train-free** 的设定：

* 底层 LLM（如 GPT-4o）始终是冻结的黑盒，不进行任何任务微调；
* 所有能力提升来自：

  * 视频 world API 的设计；
  * Phase 1 中利用 LLM 自动构建的层次化工具库；
  * Phase 2 中利用 OpenAI `tools` 的多步工具调用与记忆机制。

---

# 二、研究问题与目标（Problem & Goals）

## 2.1 核心研究问题

在统一的视频 world API 下，我们希望回答：

1. **如何设计一套 LLM 友好、可扩展的视频原子操作库（world API）？**

   * 使 LLM 能通过这些操作有效访问长视频的时空信息与记忆结构。

2. **如何在世界 API 之上，通过 train-free 的 LLM 自动诱导高层工具库？**

   * 这些高层工具应是参数化、可复用的“推理子过程”，而不是按题写死的 pipeline。

3. **在长视频 QA / 推理任务上，层次化工具（原子+高层）的引入，
   相比只有原子工具、per-question pipeline、端到端 MLLM，
   在性能、效率、任务域 OOD 适应上带来什么优势？**

## 2.2 目标

1. 设计一套**通用的视频 world API**：

   * 覆盖时间导航、局部感知、实体与轨迹、语义检索、记忆读写等关键能力；
   * 与底层视觉模型解耦，方便替换实现。

2. 构建一个 **两阶段框架**：

   * Phase 1：train-free LLM 工具诱导（Tool Induction）；
   * Phase 2：基于 OpenAI `tools` 的工具驱动长视频推理。

3. 在多个任务域 / 数据集上进行系统实验，验证：

   * 层次化工具库 vs 纯原子工具；
   * 领域内（in-domain）与任务域 OOD；
   * 与端到端 MLLM 的对比。

---

# 三、整体框架（System Overview）

系统大致分为三层、两个阶段：

## 3.1 三层结构

1. **世界层（Video World + World API）**

   * 输入：原始长视频；
   * 预处理：场景切分、关键帧/clip 抽样、视频特征、检测/跟踪、ASR、embedding 索引；
   * 对 LLM 暴露为一组统一的 **原子世界操作**（world API）。

2. **工具层（Tool Layer）**

   * 原子工具：直接映射到 world API；
   * 高层工具：由 LLM 在 Phase 1 自动诱导产生，是对原子工具的 workflow 封装。

3. **控制层（Controller LLM）**

   * 接受问题和当前记忆状态；
   * 在工具列表（原子+高层）上进行多步 function calling；
   * 驱动整个推理过程。

## 3.2 两个阶段

1. **Phase 1 – 工具诱导（Training-free Tool Induction）**

   * 输入：world API 规范 + 任务域描述（问题类型、若干示例）；
   * 由 LLM 输出一组高层工具定义（JSON/DSL），
   * 通过“工具编译器”转为可执行函数 + OpenAI `tools` schema。

2. **Phase 2 – 工具驱动长视频推理（Tool-based Video Reasoning）**

   * 使用冻结的工具库（原子+高层），
   * Controller LLM 通过 `tools` 与视频世界和记忆交互，
   * 实现长视频 QA / 推理任务。

---

# 四、视频 World API：原子操作库设计（重点）

这一节是“原子操作库”的详细设计，你后面可以直接改造成表格 / schema。

## 4.1 设计原则

1. **面向“动作”，不面向“模型名”**

   * 工具描述的是“在视频世界中能做什么”，
   * 而不是暴露底层模型细节（哪个 detector，哪个 backbone）。

2. **统一时空与实体表示**

   * 核心参数尽量统一：

     * `video_id`
     * `timestamp` / `time_range = {start_time, end_time}`
     * `entity_id` / `entity_hint`
     * `text_query`
     * `level ∈ {frame, segment, event}`

3. **原子操作粒度适中**

   * 既不要“像素级极细粒度”，也不要“一整条任务 pipeline”；
   * 适合作为高层工具的“积木”。

4. **接口规范 LLM 友好**

   * `name` 简短、语义清晰；
   * `description` 专注效果而非实现；
   * `parameters` 数量可控（2–6 个为宜）；
   * 提供参数类型和含义。

5. **区分有状态 vs 无状态**

   * 区分“只查询不改变世界”（stateless）与“会修改记忆/状态”（stateful）。

---

## 4.2 原子操作分类与详细列表

我按 6 大类给出推荐的原子操作集合。你可以后续按需要增删。

---

### 4.2.1 时间导航与场景结构（Temporal & Scene Navigation）

#### 1）`list_scenes`

* **功能**：返回视频的粗粒度场景切分与简要描述。
* **输入**：

  * `video_id: string`
* **输出**：

  * `scenes: List[{scene_id, start_time, end_time, brief_caption}]`
* **用途**：

  * 作为高层理解的入口，让 LLM 有粗略的“章节感”。

---

#### 2）`get_segment`

* **功能**：根据时间范围获取对应 segment 的元信息（ID、场景 ID 等）。
* **输入**：

  * `video_id: string`
  * `start_time: float`
  * `end_time: float`
* **输出**：

  * `segment_id: string`
  * `scene_id: string`
  * `duration: float`
* **用途**：

  * 将显式时间范围转换为 segment ID，便于后续检索/记忆。

---

#### 3）`search_segments_by_text`

* **功能**：在整部视频中，根据文本描述粗检索若干候选时间段。
* **输入**：

  * `video_id: string`
  * `query: string`  // 如“红衣男子出现的片段”
  * `top_k: int`
* **输出**：

  * `candidates: List[{segment_id, start_time, end_time, score}]`
* **用途**：

  * 全局粗定位，常作为高层工具的第一步。

---

### 4.2.2 局部感知（Local Perception）

#### 4）`describe_segment`

* **功能**：对短时间段进行自然语言描述。
* **输入**：

  * `video_id: string`
  * `start_time: float`
  * `end_time: float`
* **输出**：

  * `caption: string`
* **用途**：

  * 从视觉世界中拉回文本证据，用于细节理解与记忆写入。

---

#### 5）`detect_objects`

* **功能**：在指定时间点或小片段内检测对象。
* **输入**：

  * `video_id: string`
  * `timestamp: float`
  * `category: string`   // 如 "person", "car"
  * `text_query?: string` // 可选，如“穿红衣服的男子”
* **输出**：

  * `detections: List[{bbox, category, confidence, entity_hint}]`
* **用途**：

  * 实现对象存在性判断、局部实体识别，高层工具用来做出现检测等。

---

#### 6）`recognize_actions`

* **功能**：识别时间段中主要动作/事件标签。
* **输入**：

  * `video_id: string`
  * `start_time: float`
  * `end_time: float`
* **输出**：

  * `actions: List[{label, confidence}]`
* **用途**：

  * 动作类问题、事件计数的基础。

---

### 4.2.3 实体与轨迹操作（Entity & Trajectory）

#### 7）`register_entity`

* **功能**：基于当前检测结果或 hint，为感兴趣的对象分配全局 `entity_id`。
* **输入**：

  * `video_id: string`
  * `timestamp: float`
  * `entity_hint: string` // “穿红衣服的男子”
* **输出**：

  * `entity_id: string`
* **用途**：

  * 建立任务相关的实体引用，方便跨时间访问。

---

#### 8）`get_entity_trajectory`

* **功能**：查询实体在视频中的出现时间段及轨迹。
* **输入**：

  * `video_id: string`
  * `entity_id: string`
* **输出**：

  * `trajectory: List[{start_time, end_time, scene_id, path_repr}]`
* **用途**：

  * 实现“该人物何时何地出现”、“跟踪人物行为”等高层操作。

---

#### 9）`query_entity_state`

* **功能**：查询某个时刻/时间段内实体的状态或属性。
* **输入**：

  * `video_id: string`
  * `entity_id: string`
  * `time: float`
  * `query: string` // 如“他在做什么”、“他拿着什么”
* **输出**：

  * `answer: string`
* **用途**：

  * 支撑实体状态类问题和复杂事件链分析。

---

### 4.2.4 语义检索与相似性（Semantic Retrieval）

#### 10）`search_segments_by_semantics`

* **功能**：在指定时间范围内，检索语义上与文本相似的片段。
* **输入**：

  * `video_id: string`
  * `query: string`
  * `time_range?: {start_time, end_time}` // 可选
  * `top_k: int`
* **输出**：

  * `candidates: List[{segment_id, start_time, end_time, score}]`
* **用途**：

  * 全局或局部语义检索，是很多高级工具的基础。

---

#### 11）`search_similar_to_example`

* **功能**：以一个已知片段为例，在全视频中检索视觉/语义相似片段。
* **输入**：

  * `video_id: string`
  * `example_segment: {start_time, end_time}`
  * `top_k: int`
* **输出**：

  * `similar_segments: List[{segment_id, start_time, end_time, score}]`
* **用途**：

  * 实现“类似事件重复发生次数”、“同一类动作反复”等分析。

---

### 4.2.5 层次记忆操作（Hierarchical Memory Operations）

#### 12）`write_memory`

* **功能**：将某个时间段/事件的摘要或结构化信息写入记忆池。
* **输入**：

  * `video_id: string`
  * `level: string` // "frame" / "segment" / "event"
  * `time_range: {start_time, end_time}`
  * `content: string` // 由 LLM 或工具生成的摘要/说明
* **输出**：

  * `memory_id: string`
* **用途**：

  * 构建帧级、片段级、事件级的层次记忆，支撑长程推理。

---

#### 13）`read_memory`

* **功能**：在记忆中基于查询检索相关条目。
* **输入**：

  * `video_id: string`
  * `level: string`
  * `query: string`
  * `top_k: int`
* **输出**：

  * `memories: List[{memory_id, time_range, content}]`
* **用途**：

  * 在长程推理中快速访问历史关键事件/信息。

---

#### 14）`merge_events`

* **功能**：将多个事件级记忆合并成更高层水平的事件/剧情摘要。
* **输入**：

  * `video_id: string`
  * `event_ids: List[string]`
* **输出**：

  * `merged_event: {memory_id, time_range, content}`
* **用途**：

  * 多段聚合、剧情总结等高层任务。

---

### 4.2.6 Meta 操作（Meta & Utility APIs）

#### 15）`list_entities`

* **功能**：列出当前已注册的实体。
* **输入**：

  * `video_id: string`
* **输出**：

  * `entities: List[{entity_id, hint}]`

---

#### 16）`get_video_metadata`

* **功能**：获取视频的基础元信息（总时长、分辨率、帧率等）。
* **输入**：

  * `video_id: string`
* **输出**：

  * `duration: float`
  * `frame_rate: float`
  * `resolution: {width, height}`

---

以上 16 个原子操作构成了第一版的视频 world API。
在实现时，你可以根据现成的视觉模型把这些操作映射到后端 pipeline。

---

# 五、Phase 1：工具诱导（Training-free Tool Induction）

## 5.1 问题定义

给定：

* 视频 world API 规范（上一节的原子操作库）；
* 某个任务域的描述：

  * 任务类型分类（出现/计数/顺序/因果/总结…）；
  * 少量典型问题+参考解题思路（文字）。

目标：

> 利用 LLM 生成一组**参数化、高复用性**的高层工具定义，
> 每个高层工具是对若干原子操作的组合，封装一个“推理子模板”。

## 5.2 工具定义格式（示例）

使用 JSON 形式描述高层工具，例如：

```json
{
  "tool_name": "locate_first_appearance",
  "description": "根据人物描述，定位该人物在视频中首次出现的时间段。",
  "inputs": {
    "video_id": "string",
    "person_query": "string"
  },
  "outputs": {
    "start_time": "number",
    "end_time": "number",
    "evidence": "string"
  },
  "steps": [
    {
      "type": "call_world_api",
      "api": "search_segments_by_text",
      "params": {
        "video_id": "{{video_id}}",
        "query": "{{person_query}}",
        "top_k": 20
      },
      "save_as": "candidates"
    },
    {
      "type": "call_world_api",
      "api": "detect_objects",
      "loop_over": "candidates",
      "params": {
        "video_id": "{{video_id}}",
        "timestamp": "{{item.center_time}}",
        "category": "person",
        "text_query": "{{person_query}}"
      },
      "save_as": "detections"
    },
    {
      "type": "aggregate",
      "operation": "pick_earliest_detection",
      "inputs": "detections",
      "save_as": "first_hit"
    }
  ]
}
```

## 5.3 执行流程

1. 用 prompt 给 LLM 提供：

   * world API 列表（简化版）；
   * 任务域描述（问题类型、示例）；
   * 希望产生的工具数量、类型建议（如：定位类工具、计数类工具…）。

2. LLM 输出若干工具定义 JSON。

3. 工具编译器：

   * 检查 JSON 合法性；
   * 将 `steps` 编译为 Python/后端可执行逻辑；
   * 自动生成对应的 OpenAI `tools` schema。

4. 对每个任务域只运行一次工具诱导，得到：

   * `ToolLib_A`、`ToolLib_B`…
   * 在对应任务域内的实验中**保持冻结**。

---

# 六、Phase 2：工具驱动长视频推理（Tool-based Reasoning）

## 6.1 Controller 视角

对 Controller LLM 而言，每个问题时它能看到：

* 用户问题 Q；
* 当前视频 id；
* 部分已有记忆摘要（可选）；
* `tools` 列表：

  * 原子 world API；
  * Phase 1 生成的高层工具。

使用 OpenAI function calling 机制：

* 按步骤规划：

  * 调某个高层工具 / 原子工具 → 获取中间结果；
  * 写入/读取记忆；
  * 继续调用，直到回答问题。

## 6.2 示例推理轨迹（简述）

问题：

> “穿红衣服的男子第一次出现时在做什么？”

可能的执行轨迹：

1. Controller 调用 `locate_first_appearance(video_id, "a man in red clothes")`；
2. 高层工具内部通过 `search_segments_by_text` + `detect_objects` 获取第一次出现的时间段；
3. 返回 `start_time, end_time`；
4. Controller 再调用 `describe_segment(video_id, start_time, end_time)` 获取细节描述；
5. 将结果写入记忆 `write_memory(level="event", time_range, content)`；
6. 最终输出答案。

---

# 七、实验与 OOD 设计（简要）

这里只列结构，后面你可以细化为章节。

1. **数据 & 任务域划分**

   * 按**任务类型**分域：

     * A：定位+计数
     * B：顺序+因果
     * C：总结/多段推理
   * 也可扩展成不同视频内容数据集（视频域）。

2. **系统设置**

   * Flat-Atomic：只给原子 world API；
   * Hier-LLM-Tools（Ours）：原子 + 高层工具；
   * Per-question Pipeline baseline；
   * 端到端 MLLM baseline。

3. **指标**

   * 任务性能（accuracy/F1）；
   * 工具调用步数、token 消耗；
   * 高层工具复用度、调用比例；
   * 任务域 OOD：例如 `ToolLib_A` 用在 B 域 vs `ToolLib_B` 用在 B 域。

---

# 八、项目实施计划（Milestones）

可以简单规划几步：

1. **M1 – 世界 API 原型实现**

   * 选定一批长视频数据（或 synthetic sandbox）；
   * 实现 `list_scenes / describe_segment / search_segments_by_text` 等核心原子操作。

2. **M2 – 简化版工具诱导 + 手动调试**

   * 在一个单一任务域 A 上，跑 Phase 1，得到初版 `ToolLib_A`；
   * 手动检查几种工具的合理性。

3. **M3 – 完整工具库 + Phase 2 推理**

   * 构建若干典型高层工具；
   * 完成 Controller + memory 实现，在小规模数据上跑通 Q→tool-calls→answer pipeline。

4. **M4 – 扩展到多任务域 + OOD 实验**

   * 加入任务域 B、C，构建 `ToolLib_B / ToolLib_C`；
   * 做 in-domain 和 cross-domain 对比。

5. **M5 – 论文撰写与可视化**

   * 整理 world API 规范表、工具示例、case study。

---