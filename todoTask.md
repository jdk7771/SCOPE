# SCOPE 改进任务梳理：从 Potential / Snapshot 决策到热力图导航

## 1. 当前目标

师姐提到的任务可以理解为：改进当前系统中“评分到输出 action”这一段决策链路。

当前 baseline 的决策方式是：

```text
检测物体 + 更新 TSDF / Scene
-> 聚类生成 Snapshot
-> 检测 Frontier
-> VLM / GPT 给 Frontier potential 打分
-> 主 VLM 在 Snapshot / Frontier 中选一个
-> planner 根据选择生成导航目标点
-> agent_step 沿 pathfinder 走一步
```

希望改成更紧凑的空间决策方式：

```text
构建 goal-independent potential heatmap
-> 在 top-down map 上标注 frontier / 已探索轨迹 / 当前位姿 / key objects
-> 输入 map + frontier evidence + object evidence + goal
-> VLM 输出 OBJECT / FRONTIER / NAV_TARGET
-> planner 校验目标点并输出下一步动作
```

重点不是直接替换所有模块，而是先把主决策输入从大量 Snapshot/Frontier 多图列表，改成一张空间地图加少量证据图。

---

## 2. Baseline 现在到底怎么筛选

### 2.1 物体如何保存

代码中检测到的物体会先存进 `scene.objects`。

每个 object 主要包含：

- `class_name`：YOLOWorld 预测的类别名
- `bbox`：3D bbox
- `pcd`：点云
- `clip_ft`：CLIP 特征
- `conf`：置信度
- `image`：该 object 所属的 snapshot 图像

然后 `Scene.update_snapshots()` 会把空间上相近的物体聚成 `scene.snapshots`。每个 snapshot 对应一张全图，里面有若干个 object。

所以“20 chair 有一张图”更准确地说是：

```text
object 20 属于某个 snapshot image；
VLM 查询时，再根据 object bbox 从 snapshot image 里裁出 object crop。
```

### 2.2 Prefiltering 的真实逻辑

baseline 的 prefiltering 不是 CLIP 检索，也不是逐实例匹配，而是 GPT 类别筛选。

流程：

```text
1. 收集当前所有 snapshot crop 的类别名，得到 seen_classes
2. 把 question + seen_classes 发给 GPT
3. GPT 返回 top-k 相关类别 selected_classes
4. 保留包含 selected_classes 中任意类别的 Snapshot
5. 对保留下来的 Snapshot，只保留类别命中的 object crop
6. 最终 VLM 在过滤后的 Snapshot / crop / Frontier 中做选择
```

也就是说，它筛的是 category，不是 instance。

例如目标是“红色椅子”：

```text
seen_classes = [chair, table, sofa, cabinet, ...]
GPT prefiltering 返回: chair, sofa, ...
```

那么所有包含 `chair` 的 snapshot 都可能被保留。至于哪一把 chair 是红色的，baseline 不提前排序，而是交给最终 VLM 看 crop / snapshot 图判断。

### 2.3 如果有很多同类物体怎么办

如果场景中有 20 个 chair：

- prefiltering 只会选择 `chair` 这个类别；
- 所有包含 chair 的 snapshot 都可能留下；
- 所有相关 chair crop 都可能进入最终 VLM prompt；
- 最终 VLM 输出 `Snapshot i, Object j`；
- 代码再通过 mapping 找回真实 object id。

所以 baseline 的核心是：

```text
GPT 类别粗筛
+ VLM 多图精判
+ self_refine 二次确认
```

它没有一个显式的“20 个 chair 与 goal 逐个匹配排序”的算法。

### 2.4 颜色、材质、图像目标怎么处理

代码中没有可靠地结构化保存“红色 / 绿色 / 木质 / 条纹”等属性。

这些外观信息主要靠最终 VLM 从图片里看：

- Snapshot full image
- object crop
- image goal reference

CLIP 特征虽然存在，但当前主决策链路没有用它来做 goal-object 检索。

---

## 3. 当前 baseline 的问题

### 3.1 GPT / VLM 调用次数多

当前每一步可能包含：

```text
1. Frontier potential estimation
2. prefiltering
3. 主 VLM 决策
4. self_refine
```

场景后期 snapshot 和 object crop 很多，prompt 会变得很长，所以 baseline 才需要 prefiltering。

### 3.2 Snapshot 输入太多

当前主决策会输入：

- 多个 Snapshot full image
- 每个 Snapshot 下多个 object crop
- 多个 Frontier image
- egocentric view
- question / image goal
- Frontier potential score

如果探索时间长，Snapshot 数量会变多，图像输入很容易膨胀。

### 3.3 PotentialGraph 中的 goal_relevance 跨子任务会污染

当前 Frontier potential 包含：

- `semantic_richness`
- `explorability`
- `goal_relevance`
- `potential_score`

其中 `goal_relevance` 是任务相关的。如果一个 episode 中有多个 subtask，而 PotentialGraph 不重置，那么上一个任务的 goal relevance 可能会影响下一个任务。

更合理的是：

```text
PotentialGraph 只保存任务无关探索价值；
当前 subtask 的 goal relevance 交给最终 VLM 临时判断，不写回全局图。
```

### 3.4 纯热力图无法表达物体外观

热力图适合表达：

- 哪些区域未探索
- 哪些 frontier 更值得探索
- 哪些地方走过
- 当前 agent 在哪里

但它不适合单独表达：

- 红色椅子
- 绿色杯子
- 和 reference image 一样的物体
- “床旁边的柜子”这类上下文关系

所以不能让热力图完全替代 object crop / snapshot evidence。

---

## 4. 推荐的新方案

### 4.1 总体思路

用一张 top-down map 作为统一空间索引，再给 VLM 少量证据图。

```text
VLM input:
1. top-down map
2. frontier evidence images
3. key object evidence images
4. current egocentric view
5. question / image goal

VLM output:
OBJECT Oi
或 FRONTIER Fi
或 NAV_TARGET x, y
```

建议第一阶段优先输出 `OBJECT Oi` 或 `FRONTIER Fi`，不要一开始就让 VLM 直接输出坐标。离散候选更容易校验，也更容易接现有 planner。

### 4.2 Top-down map 中应该包含什么

一张给 VLM 的 map 建议包含：

- 可通行区域
- 障碍物 / occupied 区域
- 未探索区域
- goal-independent potential heatmap
- Frontier 编号：`F0, F1, F2, ...`
- Frontier 方向箭头
- 当前 agent 位置和朝向
- 历史轨迹
- key object 编号：`O0, O1, O2, ...`

注意：不要把所有 object 类别名直接写在图上。文字太多会乱，VLM 读小字也不稳定。

更合理的是：

```text
地图上只标 O0 / O1 / O2
prompt 或右侧 legend 写：
O0: chair
O1: sofa
O2: cabinet
```

### 4.3 Frontier evidence

Frontier 需要同时在 map 和图片中出现。

```text
Map:
F0 / F1 / F2 标在对应 frontier.position 附近

Evidence:
F0 image
F1 image
F2 image
```

原因是：如果 potential 是 goal-independent，那么 VLM 必须看 Frontier RGB 图，才能判断这个 frontier 是否和当前 goal 相关。

例如：

- 找 microwave，frontier 图像像厨房方向，则更相关；
- 找 bed，frontier 图像像卧室方向，则更相关；
- 如果只给 heatmap，VLM 无法判断目标语义相关性。

### 4.4 Object evidence

已见物体不能全放图上，也不能全发给 VLM。应该选 top-k key objects。

这里的 `object evidence` 指的是：为了让 VLM 判断“目标是否已经在已探索区域出现过”，给每个关键物体提供的视觉证据。它不是 map 上的文字标签，而是和 map 上 `Oi` 编号对应的图像证据。

每个 key object 在 map 上标 `Oi`，并配一张 evidence：

```text
O0 crop: chair
O1 crop: sofa
O2 crop: cabinet
```

对于需要上下文的问题，可以额外给对应 snapshot full image。

建议：

- object crop 用来判断颜色、材质、形状；
- snapshot full image 用来判断上下文关系；
- 默认只给 crop；
- description / image goal 或关系型描述时，再补 snapshot full image。

如果没有 object evidence，VLM 只能看到地图上的 `O0: chair` 这类粗标签，无法可靠判断“红色 chair”“和参考图一致的 chair”“床旁边的柜子”。所以 object evidence 是处理“目标之前已经出现过”的关键。

---

## 5. 关键物体如何筛选

### 5.1 Baseline-compatible 筛选

第一版可以沿用当前 prefiltering：

```text
question + seen_classes -> GPT -> selected_classes
```

然后只标注和输入这些类别的 object。

进一步限制数量：

- 每个类别最多保留 2-3 个 object；
- 总 object evidence 不超过 8-10 个；
- 优先级可以按检测置信度、距离当前 agent、是否最近被观测到排序。

优点：改动小，和当前 baseline 对齐。

缺点：如果有很多同类物体，仍然无法区分具体实例，只能靠最终 VLM 看图。

### 5.2 更强的 CLIP 检索

更好的版本可以加入 CLIP retrieval：

```text
文本 goal:
question / class / description -> text embedding
object crop -> image embedding
cosine similarity -> top-k object

图像 goal:
goal image -> image embedding
object crop -> image embedding
cosine similarity -> top-k object
```

这样可以先把已见物体候选压到 top-k，再交给 VLM 做最终判断。

这比“把所有 chair 都给 VLM 看”更合理，但实现量比类别筛选更大。

---

## 6. Heatmap 如何构建

### 6.1 2D 高斯中心点

如果 heatmap 表达 Frontier potential，那么 2D Gaussian 的中心点可以放在 frontier 的代表点上。

代码里每个 Frontier 有：

```python
frontier.position
```

它是 voxel grid 里的中心/代表位置，可以作为高斯中心。

基本形式：

```text
heatmap += score_i * Gaussian(distance(pixel, frontier_i.position), sigma)
```

### 6.2 单点高斯 vs region 高斯

两种方式：

```text
单点高斯：
以 frontier.position 为中心扩散。
优点：清楚，容易编号，适合 VLM。

region 高斯：
先在 frontier.region 上赋值，再做 gaussian_filter。
优点：更能表达边界形状。
缺点：更糊，编号和解释可能不如单点清楚。
```

MVP 建议先用单点高斯。

### 6.3 Potential 应该和 goal 无关

为了跨 subtask 复用，PotentialGraph 不建议保存 goal relevance。

可以保存：

- semantic richness
- explorability
- frontier quality
- visited penalty

当前 goal 是否相关，由最终 VLM 根据 question + frontier evidence 临时判断。

### 6.4 可以增加每个 subtask 临时 goal-related heatmap

如果担心 goal-independent heatmap 不够表达任务相关性，可以额外加一层“当前 subtask 临时 heatmap”。

推荐拆成两层：

```text
长期层：goal-independent potential
- semantic richness
- explorability
- frontier quality
- visited penalty
- 跨 subtask 保留

临时层：goal-related potential
- 当前 question / image goal 与 frontier 的相关性
- 每次切换 subtask 重新计算
- 不写回长期 PotentialGraph
```

这样可以同时解决两个问题：

- 长期地图不会被上一个任务的 goal relevance 污染；
- 当前任务仍然可以有 goal-aware 的探索偏好。

实现上可以把最终给 VLM 的 map 画成：

```text
base map
+ goal-independent heatmap
+ optional goal-related frontier markers / heatmap
+ frontier IDs
+ object IDs
```

如果要做 goal-related 层，建议不要每个 frontier 单独调用一次 VLM。更合理的是把所有 frontier 编号图和 frontier RGB 图一次性给 VLM，让它输出每个 frontier 的 goal relevance 或直接选择最相关 frontier。

---

## 7. VLM 输出与 planner 对接

### 7.1 推荐输出格式

第一阶段建议 VLM 输出：

```text
CHOICE: OBJECT O3
REASON: ...
```

或：

```text
CHOICE: FRONTIER F1
REASON: ...
```

不要一开始就强制输出坐标，因为：

- VLM 读图坐标容易错；
- 坐标可能落在障碍物或未知区域；
- 离散候选更容易 parse 和校验。

### 7.2 planner 如何处理

如果输出 `OBJECT Oi`：

```text
Oi -> object_id -> object bbox center
-> 找合适观察点
-> planner / pathfinder 走过去
```

如果输出 `FRONTIER Fi`：

```text
Fi -> frontier.position
-> 找 frontier 附近可通行点
-> planner / pathfinder 走过去
```

如果后续需要输出 `NAV_TARGET x, y`：

```text
parse voxel coordinate
-> 检查是否越界
-> 检查是否 occupied
-> 检查是否在 island / navigable
-> 如果非法，投影到最近可通行点
-> agent_step 输出下一步动作
```

---

## 8. 分阶段实现计划

### Stage 1：复现并整理 baseline 输入

目标：弄清当前每一步到底给 VLM 多少图。

需要统计：

- snapshot 数量
- object crop 数量
- frontier 数量
- prefilter 前后 snapshot / crop 数量
- 每步 VLM 调用次数

### Stage 2：做 VLM 用 top-down map

新增一个 map renderer，输出一张图片：

- 可通行 / 障碍 / 未探索
- frontier 编号
- 当前 agent
- 历史轨迹
- goal-independent heatmap
- key object 编号

这一阶段可以先不改决策，只保存图看效果。

### Stage 3：改主决策输入

新增 heatmap-based VLM prompt：

```text
top-down map
frontier images with IDs
key object crops with IDs
current egocentric view
question / image goal
```

VLM 输出：

```text
CHOICE: OBJECT Oi
```

或：

```text
CHOICE: FRONTIER Fi
```

然后接现有 planner。

### Stage 4：减少 VLM 调用

逐步去掉：

- per-frontier goal-related potential estimation
- prefiltering
- self_refine

可能保留：

- goal-independent frontier quality estimation
- 或一次性多 frontier 打分
- 或规则/CLIP 替代打分

---

## 9. 风险与消融实验

### 9.1 主要风险

1. Object evidence 设计不好会影响已见目标判断。

如果只给 top-down map，不给 object crop / snapshot evidence，那么 VLM 很难判断颜色、材质、形状、参考图匹配等信息。这类任务 baseline 反而可能更强。

2. Frontier relevance 需要额外处理。

如果 heatmap 完全 goal-independent，那么它只告诉 VLM“哪里值得探索”，不告诉它“哪里和当前目标相关”。因此必须给 frontier RGB 图，或者增加每个 subtask 临时 goal-related heatmap。

3. VLM 读图稳定性可能成为瓶颈。

地图如果太乱、文字太小、颜色太杂，VLM 会看错编号。所以 map 设计要克制：编号少、legend 清楚、颜色固定、不要堆类别名。

4. 坐标输出不一定稳定。

如果直接让 VLM 输出 `NAV_TARGET x, y`，可能出现坐标落在障碍物、未知区域或地图外的问题。第一阶段更建议输出 `OBJECT Oi` 或 `FRONTIER Fi`。

### 9.2 建议消融实验

可以设计三组对比：

```text
A. Baseline 原版
   Snapshot full image + object crop + frontier image + potential score

B. Heatmap + frontier evidence，不给 object evidence
   用来测试空间地图和 frontier evidence 对探索有没有帮助

C. Heatmap + frontier evidence + top-k object evidence
   用来测试是否能同时保留已见目标判断能力

D. Heatmap + frontier evidence + top-k object evidence + 临时 goal-related heatmap
   用来测试每个 subtask 重算 goal relevance 是否有提升
```

这里 B 方案中的 `object evidence` 是指 key object 的 crop / snapshot evidence。B 不给 object evidence，意味着 map 上可以不标物体，或者只标很粗的 object label，但不给对应图像证据。因此 B 更适合测试“探索 frontier 的能力”，不适合作为最终完整方案。

预期：

```text
B 可能探索效率更好，但已见目标判断可能变差；
C 是最有希望超过 baseline 的主方案；
D 如果 goal-related 层做得稳定，可能进一步提升目标相关探索。
```

---

## 10. 还需要和师姐确认的问题

1. 三个方向预测图到底是什么？

```text
是“当前三个方向看到的场景”？
还是“如果下一步往三个方向走，预测未来会出现的场景”？
```

这会影响是否更新 TSDF / Scene.objects。

如果只是当前 extra views，通常可以用于 VLM 判断，但不一定应该写入长期 memory，因为预测或临时观察可能不稳定。

2. Potential 是否完全 goal-independent？

建议：长期 PotentialGraph 中不要存 goal relevance。

3. VLM 最终输出什么？

推荐第一阶段输出：

```text
OBJECT Oi / FRONTIER Fi
```

后续再尝试：

```text
NAV_TARGET x, y
```

4. key object 的筛选方式用哪种？

可选：

- 沿用 GPT 类别 prefiltering；
- 加 CLIP retrieval；
- 类别筛选 + 每类 top-n；
- 只保留最近 / 高置信度物体。

5. Object evidence 要不要包含 snapshot full image？

建议默认只给 crop；如果任务需要上下文，再给对应 snapshot full image。

---

## 11. 当前推荐结论

最稳妥的改法不是“纯热力图直接输出导航点”，而是：

```text
goal-independent potential heatmap 表示探索价值；
frontier 编号 + frontier RGB 图表示未探索方向证据；
key object 编号 + 少量 crop/snapshot 表示已见目标候选；
VLM 在统一地图索引上选择 OBJECT 或 FRONTIER；
planner 负责把选择变成可执行动作。
```

这样既能减少原来大量 Snapshot/crop 输入，又不会丢掉颜色、材质、image goal 这类必须看图才能判断的信息。
