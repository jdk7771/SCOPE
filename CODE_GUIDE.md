# SCOPE 代码阅读指南

## 这个项目做什么

SCOPE 是一个**具身导航**系统。智能体在一个 3D 室内场景里，收到一个问题（比如"找到冰箱"或"找到那个红色的沙发"），然后自己探索、找到目标。

核心思路：**感知（检测物体）→ 建图（场景图谱 + 障碍物地图）→ 决策（VLM 选去哪）→ 行动（走一步）→ 循环**

---

## 代码入口

只有一个入口文件：**`run_goatbench_evaluation.py`**

整个 `main()` 函数（第 35 行）就是一个三层循环：

```
for 每个场景:
    for 每个问题(episode):
        for 每一步(step):
            1. 观察周围 → 更新场景图谱 + TSDF地图
            2. 层次聚类 → 生成 Snapshot（物体快照）
            3. 检测 Frontier（未探索边界）
            4. GPT 评估每个 Frontier 的潜力 → 更新 Potential Graph
            5. VLM 决策：选哪个 Snapshot 或 Frontier
            6. 导航到目标
            7. 检查是否成功
```

---

## 核心的 5 个类，就是 5 个文件

按重要性排序：

### 1. `Scene` — `src/scene_goatbench.py`

**场景管理器，所有物体信息都在这。**

关键属性：
- `self.objects` — 所有检测到的物体 `{obj_id: {class_name, bbox, pcd, clip_ft, ...}}`
- `self.snapshots` — 物体聚类后的快照 `{image_path: SnapShot对象}`
- `self.frames` — 每一帧的检测结果

关键方法：
- `update_scene_graph()`（第 300 行）— **最核心的方法**。流程：YOLO检测 → SAM分割 → CLIP提取特征 → 深度图转3D点云 → 和已有物体做匹配（空间相似度+视觉相似度）→ 合并或新建物体
- `update_snapshots()`（第 724 行）— 用层次聚类把物体聚成 Snapshot
- `periodic_cleanup_objects()`（第 803 行）— 定期去噪、过滤、合并物体

### 2. `TSDFPlanner` — `src/tsdf_planner.py`

**导航规划器，管地图和移动。**

关键属性：
- `self.frontiers` — 当前所有 Frontier（未探索边界）列表
- `self.max_point` — VLM 选中的目标（Snapshot 或 Frontier）
- `self.target_point` — 对应的导航目标坐标

关键方法：
- `update_frontier_map()`（第 130 行）— 在 TSDF 地图上检测 Frontier，给每个 Frontier 拍照获取图像特征
- `set_next_navigation_point()`（第 431 行）— 根据 VLM 的选择，计算导航目标坐标
- `agent_step()`（第 569 行）— 向目标移动一步

### 3. `PotentialGraph` — `src/potential_graph.py`

**探索潜力评估。把空间网格化，每个格子有一个"值不值得去"的分数。**

关键属性：
- `self.nodes` — 所有网格节点 `{(i,j): PotentialNode}`

关键方法：
- `update_from_frontier()`（第 132 行）— 当一个 Frontier 被 GPT 评估后，把它的潜力分值传播到周围网格
- `get_potential_at_position()`（第 381 行）— 查某个位置的潜力分（供 VLM 决策参考）
- `mark_visited()`（第 328 行）— 标记已访问区域，降低其探索价值

这个模块的详细解读见后文「两条评分/决策流水线 → Pipeline B」。

### 4. `eval_utils_gpt_goatbench.py`

**和 OpenAI API 的所有交互都在这里，Prompt 也在这。**

四个关键函数：
- `get_step_info()`（第 113 行）→ 组装数据（Snapshot 图像、Frontier 图像等）
- `prefiltering()`（第 352 行）→ GPT 先筛选相关类别，过滤无关 Snapshot（不然上下文太长）
- `format_explore_prompt()`（第 202 行）→ 构造主 Prompt（系统提示 + Snapshot + Frontier + 输出格式）
- `explore_step()`（第 477 行）→ 完整的 VLM 交互循环（调用 API → 解析 → self-refine 验证 → 重试）

### 5. `query_vlm_goatbench.py`

**VLM 查询的中间层。** 把 Scene 里 snapshots/frontiers 的数据转成 `step_dict`，调 `explore_step()`，再把返回结果解析成具体的 Snapshot/Frontier 对象，返回给主循环。

---

## 两条评分/决策流水线

### Pipeline A：VLM 决策「该往哪走」

这是系统的**主决策链**：

```
Scene.snapshots + TSDFPlanner.frontiers + PotentialGraph 分数
    │
    ▼
组装 Prompt（Snapshot 图像 + Frontier 图像 + 潜力分 + 问题）
    │
    ▼
Prefiltering（GPT 选 top-k 相关类别，过滤无关 Snapshot）
    │
    ▼
GPT-4o 选择: "Snapshot 3, Object 2" 或 "Frontier 1"
    │
    ▼  如果是 Snapshot + 描述类任务
Self-Refine（GPT 二次验证这个 Snapshot 真的包含目标物体）
    │
    ▼
解析返回 → 确定导航目标 → 移动
```

调用链：`run_goatbench_evaluation.py:436` → `query_vlm_goatbench.py:12` → `eval_utils_gpt_goatbench.py:477` → `eval_utils_gpt_goatbench.py:49`

**怎么改这条链：**

| 想改什么 | 去哪改 |
|---------|--------|
| 换模型 | `eval_utils_gpt_goatbench.py` 第 50 行的 `model="gpt-4o-2024-11-20"` |
| 换 Prompt | 同文件 `format_explore_prompt()` 第 202 行 |
| 不用 VLM，自己写规则 | `query_vlm_goatbench.py` 的 `query_vlm_for_response()`，自己写选择逻辑替代 `explore_step()` 调用 |
| 关闭 Prefiltering | `cfg/eval_goatbench.yaml` 设 `prefiltering: false` |
| 关闭 Self-Refine | `cfg/eval_goatbench.yaml` 设 `use_self_refine: false` |

### Pipeline B：Frontier 潜力评分

这是决定 **"哪个方向值得探索"** 的链：

```
每个 Frontier 拍一张照片
    │
    ▼
potential_estimation_gpt_goal.py → GPT-4o 评估
    │
    ▼
GPT 返回三个维度 + 总分:
  - semantic_richness (语义丰富度: High/Medium/Low → 4.5/3.0/1.5)
  - explorability    (可探索性:   High/Medium/Low → 4.5/3.0/1.5)
  - goal_relevance   (目标相关性: High/Medium/Low → 4.5/3.0/1.5)
  - potential_score  (总潜力分: 1.0-5.0)
    │
    ▼
potential_graph.py → _parse_potential_text() 解析文本 → _update_node_scores() 加权传播到周围网格
    │
    ▼
最终 VLM 决策时，每个 Frontier 带着它的潜力分一起展示给 GPT-4o
```

调用链：`run_goatbench_evaluation.py:376` → `potential_estimation_gpt_goal.py:90` → `potential_graph.py:132` → `potential_graph.py:199`

**怎么改这条链：**

| 想改什么 | 去哪改 |
|---------|--------|
| 换评估模型 | `potential_estimation_gpt_goal.py` 第 106 行的 `model="gpt-4o-2024-11-20"` |
| 改 Prompt（让 GPT 输出不同维度） | `potential_estimation_gpt_goal.py` 的 `format_content()` 第 17 行 |
| 改解析逻辑（比如把 High/Medium/Low 映射成不同数值） | `potential_graph.py` 的 `_parse_potential_text()` 第 199 行 |
| 改三个维度的融合权重 | `potential_graph.py` 的 `_update_node_scores()` 第 318 行 |
| 不用 GPT 评估，用自己规则 | `potential_graph.py` 的 `update_from_frontier()` 里传 `potential_text=None` 就会用默认分 3.0。你可以在 `_parse_potential_text()` 里插入自己规则 |
| 改潜力衰减速度 | `potential_graph.py` `__init__` 参数 `decay_factor`（默认 0.95） |
| 改 Frontior 影响范围 | `potential_graph.py` `__init__` 参数 `influence_radius`（默认 2.0m） |

---

## 修改 API 接口只需要动一个文件

**`src/const.py`** 就两行：

```python
END_POINT = ""
OPENAI_KEY = ""
```

所有需要调 API 的代码都 `from src.const import *`。涉及两个文件：
- `src/eval_utils_gpt_goatbench.py` — 主 VLM 决策 + Prefiltering + Self-Refine
- `src/potential_estimation_gpt_goal.py` — Frontier 潜力评估

---

## 配置文件就是开关面板

### `cfg/eval_goatbench.yaml` — 控制几乎所有行为

| 参数 | 作用 | 默认值 |
|------|------|--------|
| `choose_every_step` | true = 每步都问 VLM；false = 只到目标后问 | true |
| `prefiltering` | 是否用 GPT 预筛选 Snapshot 类别 | true |
| `top_k_categories` | 预筛选保留几个类别 | 10 |
| `use_self_refine` | 描述类任务是否让 GPT 二次验证 | true |
| `egocentric_views` | Prompt 是否加自我中心视角 | true |
| `use_full_obj_list` | Snapshot 是否展示所有检测物体 | false |
| `success_distance` | 导航成功距离阈值（米） | 1.0 |
| `planner.max_dist_from_cur_phase_1` | 探索阶段每次走几步 | 1 |
| `planner.max_dist_from_cur_phase_2` | 接近目标阶段每次走几步 | 1 |
| `planner.final_observe_distance` | 最终观察目标的距离 | 1.5 |
| `scene_graph.obj_include_dist` | 物体纳入场景图的最大距离 | 3.5 |

### `cfg/concept_graph_default.yaml` — 控制物体检测/匹配参数

一般不需要改。主要是 IoU 阈值、去噪/合并/过滤间隔、DBSCAN 参数等。

---

## 目录结构速览

```
SCOPE/
├── cfg/
│   ├── concept_graph_default.yaml   # ConceptGraph 参数
│   └── eval_goatbench.yaml          # ★ 评测主配置
├── src/
│   ├── const.py                     # ★ API 密钥
│   ├── scene_goatbench.py           # ★ Scene：场景加载、物体检测、场景图
│   ├── tsdf_planner.py              # ★ TSDF规划器：障碍地图、Frontier、导航
│   ├── query_vlm_goatbench.py       # ★ 组装 VLM 请求、解析返回
│   ├── eval_utils_gpt_goatbench.py  # ★ GPT 调用、Prompt、Prefiltering、Self-Refine
│   ├── potential_graph.py           # ★ 潜在地图
│   ├── potential_estimation_gpt_goal.py # ★ GPT 评估 Frontier 潜力
│   ├── hierarchy_clustering.py      # 层次聚类
│   ├── logger_goatbench.py          # 日志与结果
│   ├── habitat.py / geom.py / utils.py  # 工具
│   └── conceptgraph/                # ConceptGraph 子模块
│       ├── slam/                    # 物体匹配、合并、去噪
│       └── utils/                   # CLIP、IoU 等
├── run_goatbench_evaluation.py      # ★★ 主入口
└── environment.yml
```

---

## 如何开始调试

1. 改 `src/const.py` 配好 API
2. 只跑 5% 数据试试：`python run_goatbench_evaluation.py -cf cfg/eval_goatbench.yaml --start_ratio 0.0 --end_ratio 0.05`
3. 想改 VLM 的行为 → 改 `eval_utils_gpt_goatbench.py` 的 Prompt
4. 想改潜力评估 → 改 `potential_graph.py` 或 `potential_estimation_gpt_goal.py`
5. 想换检测模型 → 改 `cfg/eval_goatbench.yaml` 的 `yolo_model_name` / `sam_model_name`
6. 想不用 GPT 做决策 → 改 `query_vlm_goatbench.py`
7. 结果在 `results/exp_eval_goatbench/` 下
