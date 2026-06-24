我大致把代码框架看了一下，用gemma 跑了一下 成功率还可以

gemma3:27b       a418f5838eaf    17 GB

 Total success_by_snapshot results: 23.74, len: 278
 Total spl_by_snapshot results: 18.68, len: 278
 Total success_by_distance results: 45.32, len: 278
 Total spl_by_distance results: 33.43, len: 278
 Total success_by_task results for image: 40.91, len: 88
 Total success_by_task results for object: 54.55, len: 99
 Total success_by_task results for description: 39.56, len: 91
 Total spl_by_task results for image: 30.68, len: 88
 Total spl_by_task results for object: 37.19, len: 99
 Total spl_by_task results for description: 32.01, len: 91
 Average number of filtered snapshots: 4.798561151079137
 Average number of total snapshots: 13.989208633093526
 Average number of total frames: 63.2589928057554


SCOPE: 需要四次调用GPT 
        1.Frontier potential estimation  
        2.Prefiltering一步最多 1 次。用 GPT 根据 question + 已见类别列表筛 top-k 类别。  
        3.主 VLM 决策  
        4.Self-refine 看一下决策准不准 

想法：
      输入给大模型的图：  可通行/未知区域 + frontier 编号 + 历史轨迹 + 当前位姿 + 热力图 （goal-independent potential（这个还是需要提前VLM评价一下））

    另给：
      Frontier 0 图片
      Frontier 1 图片

    VLM 一次性看：
      goal + 热力图 + frontier 图片列表 + 当前视角

    输出：
      NAV_TARGET: x, y或 Frontier i -> 底层规划 再转成导航点


      问题就是图中没办法标出所有的物体，就算把物体用文字标出来，但没有语义信息，一块把每个物体的图片输入，但是VLM可以容忍这么大的输入吗？


      关于子任务切换之后是什么都不改变的  对于 potential score 是调用VLM进行的，SEMANTIC_RICHNESS  EXPLORABILITY  GOAL_RELEVANCE  是不是可以遗忘这个GOAL_RELEVANCE 
 
question ：
      这个scope就是每次看三个方向，预测的话前边预测的是分别往三个方向预测之后直
