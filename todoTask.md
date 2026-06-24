## 改进方向：用热力图替代多次 VLM 调用，直接输出导航点

到底要不要更新，还要不要看？因为预测的不一定是准确的

代码里所有检测到的物体会存在 scene.objects，再聚成 scene.snapshots。VLM 决策时会看到：
Snapshot 全图
Snapshot 里每个 object crop
object 的类别名
Frontier 图像和 potential score
当前视角     全部的snapshot？有些过分了太多了吧 如何选择一出现的目标点



任务无关热力图：只表达 semantic_richness + explorability + visited penalty，可以跨子任务复用
不要把当前的问题  永久写进 PotentialGraph 可以作为一个分层 
可以筛选关键物体进行

底图：可通行/障碍/未知区域
热力图：探索潜力，半透明覆盖
Frontier：用编号圆点或箭头单独标出来
历史轨迹：细线
当前 agent：明显箭头
物体：只画筛选后的关键物体

图中直接标物体位置是必要的，因为它告诉 VLM：目标如果已经见过，大概在哪。但不要把所有物体文字全塞图上，会乱，而且 VLM 读小字不稳定。



PotentialGraph:
  只存任务无关探索价值

Heatmap:
  可通行区域 + 未知区域 + frontier 编号 + 潜力热力图 + 轨迹 + agent + top-k object markers

VLM input:
  heatmap
  current egocentric view
  optional top-k object crops / reference image
  question

VLM output:
  NAV_TARGET: voxel_x, voxel_y

Planner:
  校验目标点
  找最近可通行点
  pathfinder 输出下一步动作




  一张 top-down 图：
  可通行/未知区域 + frontier 编号 + 历史轨迹 + 当前位姿 + goal-independent potential

另给：
  Frontier 0 图片
  Frontier 1 图片
  ...

VLM 一次性看：
  goal + 热力图 + frontier 图片列表 + 当前视角

输出：
  NAV_TARGET: x, y
  或 Frontier i -> 再转成导航点



已见物体：不要全放图上，也不要全喂 VLM。先用类别/CLIP 检索 top-k，再给 VLM。



GPT先筛选top20的物体 包含物体的snapshot都保留 然后用VLM再详细看




输入给 VLM：
1. 一张 top-down map
   - frontier 编号 F0/F1/F2
   - 当前 agent 位置和朝向
   - 历史轨迹
   - goal-independent potential heatmap
   - key object 编号 O0/O1/O2

2. Frontier evidence
   - F0 对应的 frontier RGB 图
   - F1 对应的 frontier RGB 图
   - ...

3. Object evidence
   - O0 对应的 object crop 或 snapshot crop
   - O1 对应的 object crop 或 snapshot crop
   - ...

4. Question / goal image

输出：
- 如果目标已经见过：OBJECT Oi
- 如果需要探索：FRONTIER Fi 或 NAV_TARGET x,y



用 goal-independent potential heatmap 表示探索价值；用 frontier 编号和 frontier 图像让 VLM 判断任务相关性；用 key object 编号和少量 crop/snapshot 表示已发现目标候选。VLM 不再在大量 Snapshot/Frontier 原始列表里选，而是在统一地图索引上选择 OBJECT 或 FRONTIER。


1.Frontier potential estimation  2.Prefiltering一步最多 1 次。用 GPT 根据 question + 已见类别列表筛 top-k 类别。  3.主 VLM 决策  4.Self-refine 不是每步必有


预测的是 如果下一步往这三个方向走 出现的场景 还是就是当下三个方向的场景？这个决定如何更新c场景图


