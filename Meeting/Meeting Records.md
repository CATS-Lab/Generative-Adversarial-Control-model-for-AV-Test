# 2022-12-08

## Feng2021[NC]如何生成NDE和NADE？

### overview

数据：Feng2021(NC)并没有提供数据和代码，只有绘制图片的数据。

动机：常规测试采用NAD(naturalistic driving environment)测试，数据需求量大，速度慢。通过生成NADE(naturalistic and adversarial driving environment)场景用于测试AV，可以解决这些问题。

核心方法：

1. 生成NDE。从行车数据中计算出车辆在每个场景、状态下的行为特征的自然分布。通过MDP建模车辆决策过程。抽样决定MDP中的决策。

2. 以NDE为基础，在此之上调整有对抗性的少数车辆的行为。采用RL方法（value network价值网络），根据AV（被测试车辆）的行为调整BV的行为。由于AV的后续行为未知，因此在学习过程中采用surrogate models，利用先验知识。由于调整的车辆数量较少，根据importance sampling理论，生成的车辆行为不会与自然分布有太大差距，符合现实场景。

### NDE

生成场景有两步，1. 生成路面 2. 生成其他车辆的驾驶行为

利用CARLA+https://github.com/eleurent/highway-env 渲染出高速公路场景。

用MDP建模NDE中每辆BV的行为，以秒为单位确定BV的行为。根据BV当前状态下行为的自然分布，随机抽样出BV在每个时刻的行为。

在不同的场景和状态下车辆有不同自然分布。场景被人为分成6个，状态定义为场景下的数据特征（例如在car following下与前一辆车的距离）。得到的分布包括加速度的分布、换道时的速度分布等。

![image-20221230104950420](mdPics/image-20221230104950420.png)

### NADE

在重要事件改变某些关键车辆(POV, principal other vehicle)的分布。1. 找到pov  2. 决定如何修改分布。

识别pov：定义importance function。在每个时间点评估每辆BV的值。计算方式为criticality=exposure frequency * maneuver challenge. 即每个行为的发生概率*对AV的威胁度。值最大且大于阈值，则为POV。

exposure frequency为自然分布的概率，和前文一样获取。

maneuver challenge的值涉及AV和BV两者的行动和状态，采用强化学习方法计算。基于AV的已有知识或先验测试构建surrogate models（文中利用IDM 和 MOBIL models）。

针对car-following模式，通过MDP建模AV和BV的关系，state定义为速度、距离，action定义为BV的加速度。根据MDP构建决策树，学习未来发生碰撞的概率。

针对一般性场景，通过SM模型估计AV采取各种行为的概率，转化为car-following模式，计算期望值。

![image-20221230154440205](mdPics/image-20221230154440205.png)

确定POV之后，对POV车辆的行为从加权计算maneuver challenge的importance function中抽样，其他BV的行为从自然分布中抽样。流程如下：

![image-20221230174046120](mdPics/image-20221230174046120.png)

## AV test environments generation相关题材文章（ML算法，以GAN为主）

几种测试场景生成方法：

1. 从真实场景中过滤/启发式搜索挑战性场景（不能生成）
2. 从数据中得到分布，采样
3. 搜索挑战性场景参数（很难定义参数、目标，不够真实）
4. 基于规则插入（不够真实）
5. 学习算法生成（相关文章很少）

### 非ML相关

- `Guo2022[GEIT]` - Guo, Hongyan, et al. "Generation of a Scenario Library for Testing driver-automation Cooperation Safety under Cut-in Working Conditions." *Green Energy and Intelligent Transportation* (2022): 100004.

  - 本身方法没有生成场景，只是从自然驾驶数据中过滤得到重要场景。
  - review了几篇scenario generation相关的文章（包括）`Feng2021[NC]`。
  - 大部分这类文章称为scenario-based test，重点在于区别随机生成的场景（NDE）和危险场景（NADE），使用NADE进行测试，加快速度。

- `Karunakaran2020[NULL]` - Dhanoop Karunakaran, et al. "Efficient falsification approach for autonomous vehicle validation using a parameter optimisation technique based on Reinforcement Learning" arXiv.

  -  使用基于RL的Neural Architecture Search算法搜索具有对抗性的关键场景。其中搜索问题被转化为一个参数优化问题。场景的**重要参数**被提取出。
  -  review了用优化算法搜索对抗性场景以及用参数构建场景的文章。
  -  文章只进行了行人过马路场景的搜索，重要参数为汽车出发位置、行人速度、行人加速度、行人位置、天气。目标函数（判断是否为对抗性场景）定义为分段函数之类的。

- `Kim2019[ACM]` - Baekgyu Kim, et al. "Test Specification and Generation for Connected and Autonomous Vehicle in Virtual Environments." ACM Transactions on Cyber-Physical Systems(2019)

  - 前身工作“The SMT-based automatic road network generation in vehicle simulation environment”研究了如何formalize各种**弯道**，并自动生成道路。

  ![image-20230102094948203](mdPics/image-20230102094948203.png)

  - 提出在给定path和behavior下生成虚拟测试场景的方法，考虑3D坐标。
  - 特点是比前身工作做的更细，考虑信号灯、车、自行车、行人等细节要素。
  - 生成路径的方法称为SMT(Satisfiability Modulo Theories)-based path generation algorithm，动态要素生成方法称为model-checking-based object run generation。生成的动态要素比较死板，没考虑驾驶员行为等因素（可以理解为根据规则插入）。没有强调生成对抗性场景。

### ML生成场景

- `Baumann2021[IEEE]` - "Automatic Generation of Critical Test Cases for the Development of Highly Automated Driving Functions." IEEE VTC2021
  - 用Q-learning生成关键场景的参数。
  - 模拟了一个超车场景，确定了三个重要参数用来计算目标函数，用于确定场景的参数有几十个，没有在文中一一列出。

- `Wen2020[HCIS] `- "A scenario generation pipeline for autonomous vehicle simulators." Human-centric Computing and Information Sciences
  - 在生成场景过程中使用CNN。
  - 基于cnn的场景agent选择器选择agent生成场景。（没懂）

- `Tan2021[CVPR]` - "SceneGen: Learning to Generate Realistic Traffic Scenes." CVPR 2021
  - 使用一种neural autoregressive model生成交通场景。
  - 需要给定AV的初始状态和道路地图。算法将车辆等插入地图，模拟人类行为。
  - 学习交通场景的分布，然后进行采样。

- `Xiang2022[NULL]`

## SUMO生成道路

SUMO文档中`.edge.xml`文件中比较重要的特征

| Attribute Name |                          Value Type                          |                         Description                          |
| :------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|       id       |                         id (string)                          |             The id of the edge (must be unique)              |
|    **from**    |                      referenced node id                      | The name of a node within the nodes-file the edge shall start at |
|     **to**     |                      referenced node id                      | The name of a node within the nodes-file the edge shall end at |
|      type      |                      referenced type id                      | The name of a type within the [SUMO edge type file](../SUMO_edge_type_file.html) |
|  **numLanes**  |                             int                              |  The number of lanes of the edge; must be an integer value   |
|   **speed**    |                            float                             | The maximum speed allowed on the edge in m/s; must be a floating point number (see also "Using Edges' maximum Speed Definitions in km/h") |
|    priority    |                             int                              | The priority of the edge. Used for [#Right-of-way](#right-of-way)-computation |
|   **length**   |                            float                             |               The length of the edge in meter                |
|     shape      | List of positions; each position is encoded in x,y or x,y,z in meters (do not separate the numbers with a space!). | If the shape is given it should start and end with the positions of the from-node and to-node. Alternatively it can also start and end with the position where the edge leaves or enters the junction shape. This gives some control over the final junction shape. When using the option **--plain.extend-edge-shape** it is sufficient to supply inner geometry points and extend the shape with the starting and ending node positions automatically |
|   spreadType   |           enum ( "right", "center", "roadCenter")            | The description of how to compute lane geometry from edge geometry. See [SpreadType](#spreadtype) |
|     allow      |                   list of vehicle classes                    | List of permitted vehicle classes (see [access permissions](#road_access_permissions_allow_disallow)) |
|    disallow    |                   list of vehicle classes                    | List of forbidden vehicle classes (see [access permissions](#road_access_permissions_allow_disallow)) |
|   **width**    |                            float                             | lane width for all lanes of this edge in meters (used for visualization) |
|      name      |                            string                            |   street name (need not be unique, used for visualization)   |
|   endOffset    |                          float >= 0                          | Move the stop line back from the intersection by the given amount (effectively shortening the edge and locally enlarging the intersection) |
| sidewalkWidth  |                          float >= 0                          | Adds a sidewalk with the given width (defaults to -1 which adds nothing). |
| bikeLaneWidth  |                          float >= 0                          | Adds a bicycle lane with the given width (defaults to -1 which adds nothing). |
|    distance    |                            float                             | [Kilometrage](../Simulation/Railways.html#kilometrage_mileage_chainage) at the start of this edge. If the value is positive, kilometrage increases in driving direction; if the value is negative, kilometrage decreases. Kilometrage along the edge is computed as abs(*distance* + *offsetFromStart*). |

python代码生成`.node.xml`和`.edge.xml`文件

```
# 生成Node文件, 从左下角(0,0)开始生成的
nodes_num = 6
with open('example.nod.xml', 'w') as file:
    file.write('<nodes> \n\n')
    for i in range(nodes_num):
        for j in range(nodes_num):
            file.write('\t<node id="node%d" x="%d" y="%d" type="priority" /> \n' % (i * 6 + j, i * 100, j * 100))
    file.write('\n</nodes>')

# 生成edge文件
with open('example.edg.xml', 'w') as file:
    file.write('<edges> \n\n')
    for i in range(nodes_num):
        for j in range(nodes_num):
            k = 0
            if i > 0:
                file.write(
                    '\t<edge id="edge%d_%d" from="node%d" to="node%d" priority="75" numLanes="2" speed="40" /> \n' % (
                        i * 6 + j, k, i * 6 + j, (i - 1) * 6 + j))
                k = k + 1
            if i < 5:
                file.write(
                    '\t<edge id="edge%d_%d" from="node%d" to="node%d" priority="75" numLanes="2" speed="40" /> \n' % (
                        i * 6 + j, k, i * 6 + j, (i + 1) * 6 + j))
                k = k + 1
            if j > 0:
                file.write(
                    '\t<edge id="edge%d_%d" from="node%d" to="node%d" priority="75" numLanes="2" speed="40" /> \n' % (
                        i * 6 + j, k, i * 6 + j, i * 6 + j - 1))
                k = k + 1
            if j < 5:
                file.write(
                    '\t<edge id="edge%d_%d" from="node%d" to="node%d" priority="75" numLanes="2" speed="40" /> \n' % (
                        i * 6 + j, k, i * 6 + j, i * 6 + j + 1))
                k = k + 1
    file.write('\n</edges>')
```

之后在cmd运行 `netconvert` 命令生成 `network` 文件.

```
netconvert --node-files=example.nod.xml --edge-files=example.edg.xml --output-file=example.net.xml
```

# 2023-01-03

## Tan2021[CVPR]

网络设计中使用的参数：

对每一个actor $a_i$，用向量描述 1. 属性标签$(c_i)$，表示是汽车、行人还是自行车，2. 位置$(x_i, y_i)$，3. 所在的矩形位置（box），包括长宽和角度$(w_i, l_i, \theta_i)$，行人为圆点，4. 速度$(v_i)$

将所有BV的分布定义为序贯的条件分布乘积，用ConvLSTM学习这种序贯模型

## 其他文章

- `Tan2021[CVPR]` 

生成场景

- `Suo2021[CVPR]` - "TrafficSim: Learning to Simulate Realistic Multi-Agent Behaviors"

在给定场景下确定车辆行为

“Given a HD map M, traffic control C, and initial dynamic states of N traffic actors, our goal is to simulate their motion forward.”

- `Wang2021[CVPR]` - "AdvSim: Generating Safety-Critical Scenarios for Self-Driving Vehicles"

寻找对抗性场景

- `Zeng2020[ECCV]` - "DSDNet: Deep Structured Self-driving Network"

在SDV的视角预测其他车辆的行为，输出可行轨道