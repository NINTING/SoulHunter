
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [技术实现](#技术实现)
  - [怪物转向](#怪物转向)
  - [智能体的移动](#智能体的移动)
    - [操控行为](#操控行为)
      - [__Seek(targetPos)__ 返回一个智能体到达目标位置的力](#__seektargetpos__-返回一个智能体到达目标位置的力)
      - [__Flee(targetPos)__ 产生智能体离开的力](#__fleetargetpos__-产生智能体离开的力)
      - [__Arrive(targetPos,Deceleration)__ 操控智能体缓慢减速至目标位置或附近](#__arrivetargetposdeceleration__-操控智能体缓慢减速至目标位置或附近)
      - [__persuit__ 追逐](#__persuit__-追逐)
      - [__Evade__ 逃避](#__evade__-逃避)
      - [__Wander__ 智能体在环境中徘徊](#__wander__-智能体在环境中徘徊)
      - [__Obstacle Avoidance__ 躲避障碍物](#__obstacle-avoidance__-躲避障碍物)
      - [__Wall Avoidence__ 避开墙](#__wall-avoidence__-避开墙)
      - [__Interpose__ 插入](#__interpose__-插入)
      - [__Hide__ 隐藏](#__hide__-隐藏)
      - [__Path Following__ 路径跟随](#__path-following__-路径跟随)
      - [__Offset Pursuit__ 保持偏移的追逐](#__offset-pursuit__-保持偏移的追逐)
      - [__Separation__ 群组行为 分离](#__separation__-群组行为-分离)
      - [__Alignment__ 群组行为 队列](#__alignment__-群组行为-队列)
      - [__Cohesion__  群组行为 聚集](#__cohesion__-群组行为-聚集)
      - [__Flocking__ 群组行为 群集](#__flocking__-群组行为-群集)
  - [组合操控](#组合操控)
  - [空间划分](#空间划分)

<!-- /code_chunk_output -->


# 技术实现

## 怪物转向

> 参考链接 <https://www.cnblogs.com/miloyip/archive/2013/04/19/3029852.html>

坐标系:x,z为水平平面,y轴垂直xz平面
朝向向量 lookv, 目标向量 targetv

$cos(\theta) = lookv * targetv$
$\theta = acos(lookv * targetv)$
其中$\theta$为前向向量和目标下向量的弧度

由于余弦永远为非负值，因此无法判断旋转方向

需要使用向量叉乘
$ OriV = cross(lookv,targetv)$
$OriV.y > 0,则为顺时针$ 
$OriV.y < 0,则为逆时针$
$OriV.y = 0,则在正前方$

手势法则：
左手握拳，大拇指朝上，四指呈顺时针，两向量叉乘垂直平面分量>0
左手握拳，大拇指朝下，四指呈逆时针，两向量叉乘垂直平面分量<0


## 智能体的移动

> 一个智能体的运动过程可以分为3个部分
行动选择：选择计划目标
操控：计算相关数据，产生操控力，决定移动方向，以及如何移动
移动：不同的物体移动方式的不同

__移动实例模型__

- BaseGameEntity 游戏实体基类
- MovingEntity ：BaseGameEntity  可移动实体
    - vHead vector2D 实体的朝向
    - vSide vector2D 垂直于vHead
    *vHead,vSide定义了局部坐标系*
    - dMass double
    - dMaxForce 最大操控力
    - dMaxTurnRate 最大旋转速率
- Vehicle : MovingEntity 具体对象
    - GameWorld 智能体所处环境的所有对象和数据
    - SteeringBehaviors 操控行为类
    - Update()
    更新函数，计算此时所有操控行为产生的合力，更新位置，速度，朝向，判断是否超出活动范围

### 操控行为

#### __Seek(targetPos)__ 返回一个智能体到达目标位置的力
预期速度(智能体直接从当前位置出发所需的速度)
因此$SeekV = DesireV - CurrV$
![seek](C:\Users\YT\Desktop\soulhunter\SoulHunter\image\Seek.PNG)

####  __Flee(targetPos)__ 产生智能体离开的力
与seek计算类似，预期速度朝着目标相反方向

#### __Arrive(targetPos,Deceleration)__ 操控智能体缓慢减速至目标位置或附近

Enum Deceleration{slow,normal,fast}

```lua
function Arrive(targetPos,deceleration)
    vToTarget = taretPos - Vehicle.Pos
    dist = vToTarget:Length();
    if dist > 0 then
        cdDecelerationTweaker = 0.3
        dSpeed = dist / deceleration cdDecelerationTweaker
        dSpeed = math.min(dSpeed,Vehicle->MaxSpeed)
        v2DesiredVelocity = vToTarget * dSpeed / dSpeed
        return   v2DesiredVelocity - Vehicle->Velocity()
    end
    return vector2(0,0)
end
```

####  __persuit__ 追逐
当智能体需要追逐一个目标时会预测他的未来位置，不断调整缩短距离
有一个特殊的条件，及逃避者面朝智能体，则智能体直接向逃避者移动，判断条件为逃避者的朝向的反方向与智能体朝向向量角度不超过20

若还需要计算智能体转向时间，则可以计算智能体朝向与逃避者的朝向的点积+转向率

####  __Evade__ 逃避
除了不用预测位置，evade与persuit类似


####  __Wander__ 智能体在环境中徘徊

在智能体的前方设置一个圆，目标被限制在圆上,智能体移向目标。每帧给目标一个随机的位移，随着时间的推移，目标沿着圆周移来移去。
圆周的尺寸，圆周到智能体的距离，每帧的位移大小都可以影响智能体的移动和转弯的幅度

```lua
dWanderRadius   圆周的尺寸
dWanderDistance 到圆周的距离
dWanderJitter   每帧的位移

function Wander()
    -- 添加小位移在目标位置
    vWanderTarget = vWanderTarget + Vector2(RandomClamped() * dWanderJitter,RandomClamped() * dWanderJitter)
    -- 重新投影到圆周上
    vWanderTarget.Normalize()
    vWanderTarget = vWanderTarget * dWanderRadius
    -- 移动圆周
    vTargetLocal = vWanderTarget + vector2(dWanderDistance,0)
    vTargetWorld = Vehicle.PointToWorld(vTargetLocal)
    return vTargetWorld - Vehicle.Pos 
end 
```

####  __Obstacle Avoidance__ 躲避障碍物
    - 计算最近相交点
        1. 构建智能体检测盒，宽度等于包围半径，长度与智能体的速度成正比
        2. 只考虑在检测盒内的障碍物，迭代世界范围内的障碍物，标记检测盒内的障碍物
        3. 将标记的障碍物转为到局部坐标空间，抛弃在智能体后方的障碍物
        4. 测试障碍物检测盒重叠，若障碍物距离智能体朝向向量的长度 < 障碍物包围半径 + 检测盒半径，则相交
        5. 找出最近相交点

    - 计算操控力
        - 侧向力
        障碍物包围半径 - 障碍物与智能体的水平距离产生侧向操控力，
        - 制动力
        智能体的反向向量，正比障碍物的距离

####  __Wall Avoidence__ 避开墙
智能体前向突出三根触须，检测是否与墙相交
操控力方向为墙的法线方向，正比于中间触须前端穿透墙的距离

####  __Interpose__ 插入
将一个智能体插入两个智能体中间

####  __Hide__ 隐藏
找到一个位置使得障碍物在敌人和ai之间

####  __Path Following__ 路径跟随
智能体沿着给定的一系列路点移动
当ai到达最后一个节点时，使用arrive，否则使用seek
智能体的速度，pathfollow完全依赖游戏环境

####  __Offset Pursuit__ 保持偏移的追逐

偏移总是发生在'领头'的局部空间，因此我们需要得到世界空间的偏移。然后如同pursuit那样，预测领头的未来位置，使用arrive移动到目的位置

####  __Separation__ 群组行为 分离
分离产生操控力，操控智能体远离附近的智能体，计算附近邻居到智能体的向量的合力，力的大小反比与智能体到邻居的距离

####  __Alignment__ 群组行为 队列
使得智能体与其周围的邻居保持相同方向的朝向
计算周围智能体的总体朝向，将其平均，得到期望朝向。
期望朝向-当前朝向 = 操控力，随着时间推移最终所有智能体都会朝向期望朝向

####  __Cohesion__  群组行为 聚集

聚集使得智能体移向邻居，计算所有邻居的位置总和，求其平均值便是他的质心，使智能体朝质心移动

####  __Flocking__ 群组行为 群集
群集行为中的个体知道自身，随机的聚集在一起。
通过分离，队列，聚集三种行为组合构成，当一个智能体离群体太远时，智能体会自动的wander，保证整个群体始终运动


## 组合操控
一个最终的合力由多个操控力合成，由于有最大操控力的限制，因此需要对各个计算的操控力进行截止

__加权截断总和__
每个操控行为乘上权值相加然后截断计算合力
这个方法可能会导致在一些场景中无法很好的应用，例如相互的冲突力。

__优先级的加权截断__
给操控力的各种行为加上优先级，先处理优先级高的行为

__确保无重叠__

当两个 智能体交叠时，separation 无法很好的起作用，因此需要一个专门的算法防止这种现象
可以检查智能体的包围半径是否与其他智能体有重叠，如果有，则朝向相交的相反方向移动

## 空间划分

__单元分割技术__
将空间分割成一个个的单元格，当一个实体在该单元格内，则该实体加入单元格的链表中
如此计算时，只需要计算周围单元格有哪些，在计算周围单元格内的实体是否在检测范围内加入邻居表中
class Cell      单元格的基本实体对象
class CellSpacePartition    单元格
