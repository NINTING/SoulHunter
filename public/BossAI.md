

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

- Seek(targetPos) 返回一个智能体到达目标位置的力
预期速度(智能体直接从当前位置出发所需的速度)
因此$SeekV = DesireV - CurrV$
![seek](C:\Users\YT\Desktop\soulhunter\SoulHunter\image\Seek.PNG)

- Flee(targetPos) 产生智能体离开的力
与seek计算类似，预期速度朝着目标相反方向

- Arrive(targetPos,Deceleration) 操控智能体缓慢减速至目标位置或附近
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

