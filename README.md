> 一开始想用按键精灵写，但是安卓模拟器一直跑不起来跳一跳。后来偶然看到 Github 上 [wangshub/wechat_jump_game](https://github.com/wangshub/wechat_jump_game) ，刚好Python比较熟悉，就仔细看了下。从Issues里找到了安卓运行的方法。  
> 跑起来之后就研究了一下代码，还是蛮有启发的。但因为之前构思按键精灵有过一个思路，比现有的这个方法简化不少。于是改了一下代码。这里记录一下思路。  
> PS: 本代码目前只处理了安卓，但核心算法部分和iOS是通用的。

## 0. 已有思路
![](http://upload-images.jianshu.io/upload_images/1980018-d297600443cb445c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

- 找到棋子的x,y
- 找到目标平台中心点的x,y
- 计算两点像素距离（橙色）
- 按压时间 = 距离 × 系数

> 缺点：这个方法得到的结果是直线的像素距离，于是不同屏幕尺寸下，感官上相同的跳跃距离，像素距离确是不同的，这就需要手动设置不同系数（一大堆的congif）。  
> 思考：能不能更直观地设置跳跃所需要的时间？

---

## 1. UI自适应规则
>实测 4:3/16:9/2:1 三类屏幕比例，得到如下结果。

#### 游戏内UI
- 游戏主画面按宽度适配，垂直居中。
- 其他界面元素四角定位，比如关闭右上角，分数左上角。
- 这里是一个重要信息，**游戏主画面是根据宽度缩放的**。
![](http://upload-images.jianshu.io/upload_images/1980018-cfb438666a02b16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 再来一局
- 核心UI是16:9，然后fit到窗口。
- 这样就能找准按钮的位置了。
![](http://upload-images.jianshu.io/upload_images/1980018-e8d6080d2319dc86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

## 2. 简化方案
![](http://upload-images.jianshu.io/upload_images/1980018-35a966aa2d702031.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 从绝对距离到比例关系
- 根据我们上一节提到的宽度适配原则。
- 假设两个方块离得非常远，刚好斜着隔开整个屏幕，跳跃时间假设是1秒。
- 那么如果把目标平台移近一半，那跳跃需要时间是不是就是0.5秒呢？
- 也就是说 **把跳跃距离，从像素的绝对距离，转化为相对的比例关系。**

![](http://upload-images.jianshu.io/upload_images/1980018-673619527ba5744c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 从斜边到邻边
- 另外，因为游戏视角固定。
- 所以斜边的比例关系，可以用 X 轴的水平的比例关系替代。
- 那么 **跳跃距离** 就转化为：**起点到目标的水平距离** 和 **屏幕宽度** 的比例关系。
- 不再需要获取 Y 轴。
- **按压时间 = 跳全屏宽度需要的时间 × (起点和目标的横向距离 / 屏幕宽度)**

```
distanceX = abs(board_x-piece_x) #起点到目标的水平距离
shortEdge = min(im.size) #屏幕宽度
jumpPercent = distanceX/shortEdge #跳跃百分比
jumpFullWidth = 1700 #跳过整个宽度 需要按压的毫秒数
press_time = round(jumpFullWidth*jumpPercent) #按压时长
```

> 优点：省略了所有 Y 轴的取值，并且自适应所有设备。（实测 4:3/16:9/2:1都成功。原方案的**所有**配置文件都不需要了。）

---

## 3. 复杂情况 - 潜在的可能
![method_bais.png](http://upload-images.jianshu.io/upload_images/1980018-27074b9ad1146506.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 上一次从另一个方向跳过来且未落在中心
- 可能下次起跳位置需要修正（紫色）
- 不过误差并不大，除非当前格子很大。
- 加上还有模拟器卡顿或者其他细微的影响，所以这点偏移基本忽略，目前没有看到方案适配这个。

> 之所以不必纠结误差，因为即使每次偏移中心10%，不考虑平台大小，也要连续累积10次才会掉落。  
> 也就是棋子要往一个方向跳10次，概率`(1/2)^(10-1)=1/512`，其实很低的。  
> 实际过程中，模拟器卡顿掉帧对我来说影响更大。  

---

## 4. PC+模拟器简单教程（模拟器避坑）
- 我用的夜神模拟器。需要在【多开管理】里新建一个安卓5.1才可以运行小游戏。默认的安卓4.4.2运行会闪退。
- 需要额外安装 [**adb**](https://adb.clockworkmod.com/)，然后把安装好的 `adb.exe`，复制覆盖掉`~/nox/bin`下面的`nox_adb.exe`和`adb.exe`。
- 环境配置添加`~/nox/bin`
- 从多开管理中重启模拟器。
- 安装依赖 `pip install -r requirements.txt`
- 到模拟器打开跳一跳，进入游戏界面。
- 运行脚本 `python wechat_jump_auto_slim` 就会自动玩了。
- 运行的话只需要这单个文件就可以了

### 目前最高跑到过 7000+

---

## 5. Git位置
<https://github.com/Erimus-Koo/wechat_jump_game>  
还搞不懂Github怎么用，不晓得怎么提交回去，就先这样吧。

---

## 6. 更新
- 更新为内存IO，不再写入根目录文件，减少磁盘IO。（其实是因为读写中途经常和onedrive起冲突）
- 更新了随机时间，扩大了点击的随机范围。但是发现腾讯很可能搞了服务器黑名单(详见[跳一跳外挂屏蔽规则](https://www.jianshu.com/p/4a49a6e2b88f))，就先凉着吧。
- 从抓包数据看，这个版本已经绕过作弊检测了。[# 958](https://github.com/wangshub/wechat_jump_game/issues/958) 2018-01-09 [# 1053](https://github.com/wangshub/wechat_jump_game/issues/1053) 实测防ban成功。
- 又把readme给细化了一遍。目前花费的时间，写代码 < 写说明 < Pull Request ，也是醉了。


`微信 小程序 跳一跳 python 外挂`
