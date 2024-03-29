# 应用在栅格网络中利用矩阵对齐消除Dijkstra序列化瓶颈的最短路径算法
STAR

## Situation：
情景：传统Dijkstra以及Dijkstra的诸多变体，例如双向的Dijkstra、多线程的Dihkstra，以及Fibonacci Heap的Dijkstra并不能在大规模地图中或者实时更新的地图中取得可接受的算法效率。
当地图规模达到1000\*1000的规格的时候，传统Dijkstra计算最短路径的耗时达到了将近1分钟，这还是地图保持不变类似于迷宫的情况下，
当地图变成实时更新的时候，例如在交通中的道路导航，飞行器在天空中根据气象条件需要的实时更改导航，效率就会变得更差，当实时更新的地图规模达到512 \* 512 \* 12的时候，传统Dijkstra的耗时就达到了惊人的6.5小时，这样的算法效率显然很难接受。

## Task：
任务：而该课题的任务就是在给定天气气象数据预测情况的条件下，和在给定的起点和终点之间，为无人飞行器找到一条最安全的通行路径，当然无人飞行器携带的燃料是有限的，所以飞行时间是有限制的。

## Action：
行为：对于大规模的应用场景，我们首先想到的肯定是并发，但不幸的是传统Dijkstra难以并发，困难在于传统Dijkstra的过程是严格序列化的，只有在确定上一个点的选择之后，才可以进行下一个点的探索，而这过程中一直需要维持一个有序的序列，来找寻下一个点。这就是dijkstra难以并发的原因也是效率不尽如人意的原因。然后我们发现在一个层次图中是不需要维持这个有序的待选序列的，而所有的有向图都可以在扩充一维之后变成具有层次的层次图，添加的该维可能是花销维或者时间维，这个层次图层间不能通行，而是把原图中的邻接关系映射到层与层之间，然后逐层进行探索，这就提供了并发的基础。
利用栅格网络的特性，我们可以用矩阵来对栅格进行建模，用一个探索矩阵来探索地图，用一个标记矩阵来标记当前网格是从哪个网格传播过来的，以便到达目标点之后的回溯。探索矩阵除了出发点被初始化为0，其余点均被初始化为正无穷，代表未探索过。
我们利用pytorch来利用GPU的并行计算能力来进行矩阵的运算。每一次探索，将矩阵按照4个方向分别进行对齐，空出来的部分用无穷来填充，然后和可传递矩阵进行运算，然后对4个方向的运算结果进行降维操作（也就是最小化操作），在导航矩阵中记录最小值的移动方向。探索到给定目标节点之后，用导航矩阵的记录来进行回溯。
首先我们在带障碍的单位花费迷宫问题中进行了算法的验证。很多现实问题都可以被建模成带障碍的迷宫寻路问题，所以这个场景很有代表意义。在单位花费的迷宫问题中，每一步的花销都是1，所以添加花销维，将原始maze映射成0-1可通行矩阵，0为障碍，1为可通行。按照算法，进行层间传播，将探索矩阵按照四个可通行方向进行对齐，然后与传递矩阵进行相加，得到cost为t的目标点，直到访问到目标节点。所以整个算法运算的次数为T*4。
然后我们回到之前提到的无人飞行器导航问题，该问题与迷宫问题不一样的是要求的并不是最短路径而是最安全的路径，所以我们针对该问题进行了具体的改进，按照数据集中提供的气象条件，对每一个网格进行坠毁概率的预测，那么此时整个算法的目标就变成了找到具有最小坠毁概率期望的路径。因为由原始气象数据转化过来的坠毁概率矩阵本身就具备了时间维，所以我们就不必扩展，我们初始化探索矩阵除了出发网格为0其余网格全1，代表未探索的网格坠毁概率为1，因为此时呆在原地是有意义的，所以对5个方向进行探索，针对前一层中5个可能的父节点，算法通过平移前一层坠毁概率矩阵来对齐两层，每次计算一个方向上传播而来的坠毁概率，5个方向对应的坠毁概率存在一个临时的张量中，然后让temp按照第0维降维，就得到了当前层的最小坠毁概率，temp在降维中得到的坐标标记了当前层的最小坠毁概率是由前一层的哪个方向传播而来，探索最大飞行时间T之后，然后按照导航矩阵从目标节点进行回溯，找到最安全的路径。

## Result：

结果：
在无人机的导航问题中，MAD算法运算时间仅为12秒，而传统dijkstra则花费了6.5个小时，表现是碾压级别的，这证明了我们的算法是可靠可行的。下面我们用较为简单的迷宫问题来分析算法的时间复杂度。
实验结果表明算法的时间复杂度为线性的。而在迷宫中，当迷宫问题扩展到500*500的时候，MAD的时间花销就远低于dijkstra，而且我们还做了另一个实验来验证MAD实际上是Dijkstra的下界。首先Dijkstra的下届也就是最好情况，就是地图的可通行的路径就是最短的路径。因为GPU和CPU的运算速度还是有很大差距的，所以我们在保持地图规格不变的同时，扩大了障碍之间的间距，使得可通行的网格变多，以此来弥补两个运算器之间的差异，随着间距的变大，MAD的运行时间逐步往下移，趋于Dijkstra的下界。


简介：
该课题是从阿里云天池大赛‘未来可期’中衍生得来的，这个比赛由英国皇家气象局提供了的一部分地区的5天18个小时的气象数据（风速，降雨）之类的值，整个地区被划分成网格状548*421，每个地区都有固定的横纵以及时间坐标和当时的气象数据，提供了10个预测模型预测出的数据，然后在数据预处理方面需要根据这10个官方给出的预测模型的预测值来给出最终预测，这里我们是利用了xgboost模型来确定最终预测，然后就是寻路算法的设计了。

一开始是想到很常用的A-star算法，但经过分析和尝试之后，我们发现A-star一次只能搜寻一个起点到一个终点，在这里，我们是从一个城市起飞，然后飞到10个目的地，也就是说我们需要重复10次，这样效率比较低，并且非常致命的一点，整个寻路问题可以看作是在有障碍的地图中寻路，A-star的启发式搜索出来的路线会距离障碍区非常近，所以非常不安全。所以放弃了这个想法，然后基于Dijkstra的一次运行可以搜索多个终点的最优路径的优点，我们选择尝试Dijkstra，我们首先尝试了cpu版本的Dijkstra，发现耗时非常长，一天的路径搜索耗时就是6个小时，这是不能接受的，我们就想到去并发Dijkstra，发现因为其搜索序列的固定性，只有搜完前一个之后才能搜索后一个，所以难以并发，只有用每一个线程去模拟每一个节点的传播才可以实现并发，基于这个思想，我们提出了扩展的层次网络，在层次的网络结构中，就不需要保持搜索队列的有序性，可以并发进行搜索。

利用栅格地图的结构性特征，我们可以用矩阵来对其进行建模，然后针对其固定的传播方向（上下左右和原地不动）来进行地图的探索。用一个探索矩阵来探索地图，用一个标记矩阵来标记当前网格是从哪个网格传播过来的，以便到达目标点之后的回溯。实际上这里我们又对数据进行了一次处理，将网格中的气象数据经过sigmod函数的转换转化成0到1之间该地点的坠毁概率，探索矩阵除了出发点被初始化为0，其余点均被初始化为1，代表未探索过。针对前一层中5个可能的父节点，算法通过平移前一层坠毁概率矩阵来对齐两层，每次计算一个方向上传播而来的坠毁概率，5个方向对应的坠毁概率存在一个临时的张量中，然后让temp按照第0维降维，就得到了当前层的最小坠毁概率，temp在降维中得到的坐标标记了当前层的最小坠毁概率是由前一层的哪个方向传播而来，探索最大飞行时间T之后，然后按照导航矩阵从目标节点进行回溯，找到最安全的路径。

然后在此基础上，因为学术论文的要求，我又将该特定应用场景抽象化，设计了Matrix Alignment Dijkstra算法应用在带障碍的迷宫搜索问题，带障碍的迷宫搜索问题是一个平面2D场景，所以我们在2D的基础上通过添加cost维的方法将地图扩展成3D的，但实质上每层的可传播矩阵都是固定一样的，并且迷宫的移动花销是单位距离，所以在探索时我们应该将探索矩阵和该层的传播矩阵进行相加，在探索到目标节点后，整个算法停止，可以理论分析得到，整个算法一共运行了4*T次，所以时间复杂度维O(t),实验结果是在小规模的迷宫地图上，是比不过cpu版本的dijkstra的，因为GPU一次运算时间消耗是要比CPU高的，但在地图规模达到一定规模之后，实验显示500\*500的规模时，MAD算法就已经完全碾压Dijkstra的。所以在一些实时的任务中，例如实时的交通导航之类的任务，我觉得这个算法的实用性还是很强的。
