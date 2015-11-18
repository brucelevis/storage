#  服务器场景管理
[TOC]
## 需求分析
	网络游戏服务器中游戏对象管理一般以地图场景为单位，简单的方案就是以链表或者数组形式存储在地图对象中。常用操作有根据对象编号获得对象本身，获取某一坐标点周围某个范围内的特定对象，这种需求称为AOI。AOI(Area Of Interest)在MMOPRG游戏服务器上是不可或缺的技术，广义上，AOI系统支持任何游戏世界中的物体个 体对一定半径范围内发生的事件进行处理；但MMOPRG上绝大多数需求只是对半径范围内发生的物体离开/进入事件进行处理。你进入一个 游戏场景时，如果你能看到其他玩家，那背后AOI系统就正在运作。显然,AOI实现方案的好坏直接决定了服务器能够承载的同时在线人数上限，也 决定了策划对游戏玩法的发挥程度。回合制和即时制的MMOPRG通常选用不同的AOI方案。为了高效的AOI，那么底层得有高效的场景对象管理发放。
## 常见场景管理方法
### 数组/链表
	所有对象存放于数组或链表中，每次使用时遍历全部对象，如果对象数目比较少，这种方法简单高效，但是无法适应大量对象的场景。这种方式几乎不会出什么bug，可以在开发时使用此种方式验证逻辑的正确性。
### 四叉树
	此种方式内存占用少，理论上是O(n),当然这棵树深度会被限制，比如深度为5，那么树的节点最大数目也会被确定。但是对象信息是被存储在叶节点上的，那么非叶节点是被浪费的，查找某个坐标对应的叶节点最大也可能会递归到某个叶节点，递归次数取决于树的深度。见下图，来源于网络。

![图片来源于网络](https://raw.githubusercontent.com/doublefox1981/storage/master/quadtree.jpg)
### 网格/空间切割
	俗称网格法，最为朴素的方法。实现方法：如某个地图长宽为M*N，用K*K的网格保存对象，将坐标为x，y的对象保存在网格[x%k,y%k]里。实际编码过程中，可以要求K为2的幂次方，可以方便的用位运算取代取模。这种方式如果实现查找x，y周围某个范围(dist)的对象，可以通过优化为[x-dist,y-dist,x+dist,y+dist]正方形框R。通过R可以计算出跟R重叠的网格，如果某网格跟R完全重叠，那么该网格内的所有对象均满足条件，不用进行距离判断，如果某网格与R部分重叠，则遍历该网格所有对象进行距离判断。实际应用中根据应用场景，可以调整K的大小，以实现cpu和内存的最优使用。为了节省内存，如某个网格没有对象存在，那么可以不会该网格分配内存，采用lazycreate方式创建，每次如有某个对象从网格remove，则检测该网格对象数量，如为0，则删除该网格。
图1
![图1](https://raw.githubusercontent.com/doublefox1981/storage/master/p2.png)
图2
![图2](https://raw.githubusercontent.com/doublefox1981/storage/master/p1.png)

	如上两图，图2是根据x，y，dist裁剪出来的网格。其中8,9是跟R完全重叠，2,3,4,5,7,10,12,13,14,15部分重叠，8,9不用计算距离。这是一个极端的例子，选择合适的K值后，不会裁剪出这么多的网格。

## 代码示例
```lua
--是否是2的幂次方
local function is_power_of_2(n)
	return (n&(n-1)==0)
end
```
```lua
--计算矩形框r1是否全部包含r2，{[1]left,[2]top,[3]right,[4]bottom}
local function include(r1,r2)
	return r2[1]>=r1[1] and r2[2]>=r1[2] and r2[3]<=r1[3] and r2[4]<=r1[4]
end
```
```lua
local SUB_SCENE_WIDTH = 16
local Scene = class(function(self,w,h)
	self.subwidth = w // SUB_SCENE_WIDTH
	self.subheight = w // SUB_SCENE_WIDTH 
	self.width = w
	self.height = h
	self.child = {}    -- 存储所有非空的网格
	self.chddata = {}  -- 网格对象数量管理

-- 计算x，y坐标所在的网格索引
function Scene:CalcIndex(x,y)
	if not self:CheckPos(x,y) then
		return 0
	end
	local subx = x // SUB_SCENE_WIDTH 
	local suby = y // SUB_SCENE_WIDTH 
	return suby * self.subwidth + subx + 1
end

-- 进入地图
function Scene:Insert(obj,x,y)
end

-- 离开地图
function Scene:Remove(obj,x,y)
end

-- 更新obj的位置
function Scene:Update(obj,ox,oy,nx,ny)
end

-- 查询x，y，dist内的obj
function Scene:Query(x,y,dist,obj)
end
```

## 基本流程
	

 1. 服务器启动后，根据地图的长和宽实例化Scene
 2. 玩家进入地图后，调用Insert加入到对应网格中
 3. 玩家移动后，调用Update修改所在网格，以及存储的坐标
 4. 利用Query可以查询周围obj，根据需求决定是否做进一步的AOI处理

	