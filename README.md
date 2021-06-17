# VFX Lesson
Unity visual Effect Graph的新手教程
Unity 使用的版本 2021.1.3f 或者更新版本 
visual effect graph 使用的11.0版本
 
##  笔记:

1. Audio Sample:
  需要FMOD的支持，因为读取数据的接口是FMOD提供的，我在wwise中没有查到具体接口能够采样出多个数据。
  另外，使用这个sample需要当前播放模式，并且能够声音能够播放。
  需要注意的是：
  默认采样的数据最后一个数据与第一个数据重复，如果不想处理需要在vfx中跳过（当然循环的时候刚好对了）
  音量大小与数据的大小有关联，当静音情况下，数据为0
2. Trigger GPU Event
  无论出现了几个Update Context，Trigger GPU Event都是最后触发，多个Trigger GPU Event都是没有前后顺序关系。
  Trigger GPU Event连接GPU Event，只能对接Initialize Context，这是不同于CPU Event的，可以链接Spawn Context
  CPU/GPU不能 连在相同的initialize context上
3.  Trigger over Time
	里面填的数量表示1秒内触发的次数，并不是当前触发的次数
4. Trigger over Distance
	由于粒子系统并不追求连续运动，没有办法通过当前位置，前一帧位置（可以保存多一点数据在Old position中），Unity在这里利用当前的速度与每帧的时间间隔来计算下一帧的距离，填的数字仍然是1 米间隔内触发多少次。
    注意，由于同期触发的initialize context是一次性处理，我尝试了好像没有办法在**速度很块**的情况下，将发射出来的粒子均匀的分布。
5.  使用Depth Buffer来做collide的方案，要注意跟实际的物理世界的变化，由于Depth buffer是从镜头角度观察投影的深度图，因此有些有“洞”的结构在某些视角出现“洞”消失的情况（虽然可以自定义一个camera来定义depth buffer的生成角度，也不一定能解决问题）
	我在使用过程中发现，depth buffer做碰撞时精度较低，如果需要效果特别清晰的 谨慎使用这个技术。
6. Game 窗口中要Enable Vsync，否则在全屏下由于帧率同步的问题会显得VFX运行时长的变化，从而效果不同
7. visual effect graph中sample中带的OutputEventHelper，从表现上来看是失效了，我怀疑是在Spawn Context里面设置的Attribute接口改成了Set Spawn xxx，导致数据读不出来（不排除我用错了）
8. output particle strip quad 这个renderer，可以指定 uv的连续方式 在tiling mode中选中custom 出现TexCoord，可以在里面设置，默认行为是v方向使用age over life，表现等同stretch，如果要参照其他的参数，可以使用custom模式自己设置Texcoord即可。
9. connect to target也是一种决定朝向的方式，注意虽然它也能近似表现出连线的效果，但是线一般是断的，重叠的，通常不建议使用这个来表现连线，请使用strip 这个类型的粒子。
10. prewarm的设置在vfx资源对象上，其中total time表示要预热的时间，step count用来表示此预热时间要分多少步完成，其中最重要的total time。


## 示例说明（简短）
### context
- Event：需要指定特定Event名字的结点，可以与Spawn Context连接，用来控制Spawn的Start/Stop
在这里使用了timeline 通过给vfx 发送Event Name来逐步启动特效，以及通过移动vfx的坐标来触发spawn Event中的Spawn Over Distance的设置。
- Initilization：展示了Initialize Context中可以获取Spawn中的Set Spawn Event XXX的属性，从而不同的Spawn 共用一个Initialize 模块的功能
- Spawn：每接收一次消息就触发一次，与粒子本身是否已经play无关。Spawn例子展示了不同的spawn设置的用法。
- Space：是指context上的L/W，其中L表示Local，设置为Local是，所产生的粒子在最终渲染时要收到VFX这个gameobject影响，这是一般的情况，
设置为Local的vfx可以摆放在scene的任意位置上，但是如果vfx对象发生transform变化，粒子表现为跟随vfx一起变化的特点
将Context设置为world，表示生成的粒子的位置不再受vfx对象的transform变化，如果希望制作的vfx本身能够被放在scene的某个位置上（受transform影响），在粒子更新过程中不在受vfx对象的影响。可以使用ChangeSpace结点来获取当前的vfx位置作为发射时的位置，其他状态下都是world空间即可 具体参照Space
- Output：展示了vfx的渲染模块，相同的粒子可以表现不同的渲染过程，每个渲染过程自己的特有的属性可以在output context中设置，与在Update/Initialize Context不同的地方时，在Output Context的部分，实现表现在Shader中，而不是计算中（有时候可以认为是等价的）
一般renderer关心的是粒子的朝向 粒子的大小 粒子的颜色 粒子的uv等信息，这些根据output类型不同变动的话，可以把计算放在output context中。
- Update：Update紧跟Initialize Context部分，可以用多个Update context串在一起，但是他们都会整合在一个计算单元内。Update表示每一个粒子每帧要进行的操作。常用的计算操作有：更新位置（Update Position）更新生命（Age Particle） 设置Alive（Reap Particle） 更新旋转角度（Update Rotation）
对于位置或者角度，常用的设计思路有两种：1、设置速度，加速度等，根据这些相关量由粒子系统自己计算位置。2、使用set position明确的设置当前的位置。
这两种思路有各自擅长的地方，直接设置位置，可以保证位置的准确性，能够维持粒子之间的外形，比如示例中的正弦波形图，另外一种可以产生复杂的连续的运动轨迹，比如涡流效果等。

### Attribute 属性
- SpawnEvent Attribute：这指的是在发射器的时候设置的属性，在Initialize Context继承后就能转化成粒子自身的属性，增加一种控制手段。
- Custom Attribute：这里指如果系统提供的属性不足用的情况下 可以自定义添加属性，一般流程是首先设置（通常发生在Initialize Context中）
然后更新，使用（get/set，一般发生在Update Context中）比如，需要不同粒子要制作跟uv流动相关的动画，可以设置一个custom attribute，然后在output中get。示例中表示的是，记录半径以及角度，在此基础上制作动画。
- LifeTime：这个例子用来表明 Age与Lifetime的区别与联系。

### Block
== Block表现为直接使用在不同Context中的结点（称之为模块更好）在block中，一般都最终跟设置了对应的属性（一般包括获取属性，计算，写入相关属性的过程）不用operator来连线表示，通常的情况是连线表示会结点更多，但是这里也有一些负面的问题，就是当执行未达到预期的时候，通常需要查看生成的代码才能知道对应的关系。以下这些示例就是用来说明block依赖哪些数据的。==

- Force：影响速度，最终影响到位置，示例展示的是虽然context是local的，但是Force可以是world这样的效果。最常用的Force有重力，turbulence等，turbulence表现相当的混乱，注意Force的效果是可以叠加使用的。

- Collision：影响速度，生命，位置，朝向等信息示例展示的是在collide之后粒子死亡触发了另外一个粒子的表现。
- Rotation：表现得是通过影响Angular speed影响最终得Angular的情况
- Size：用来表示size，scale，以及Pivoit相互的关联。

### operator
== Operator用来表示运算，出了一般的加减乘除之外，有一些更高级使用更常见的操作。== 
- Coordination：表示坐标系转换，坐标系转化类似颜色的转化，通常不同的坐标系都有自己善于表达的方面。这里注意vfx默认使用的是3D 笛卡尔坐标系（x  y z）来表示的，不同坐标系需要转换回来，否则每办法对接。
Coordiniation用两个粒子来表现一个是阿基米德螺线 一个是球坐标效果，只用来说明，具体使用的时候要看效果才能判断
- Noise：展示了不同noise的效果图，同时展示了当noise作用在位置时的效果。noise是程序化生成中重点部分，具体可以参照substance designer中generator里面noise目录下的内容。
Periodic：周期，这里表示利用时间这个连续变动的变量生成各种周期曲线的方法。周期曲线是循环动画的基础。
- Sample：采样，从数据集中采集数据，根据数据集的堆叠维度，对应的采样坐标的量也要变化，比如1D的需要一个参数，2D的需要2个参数，3D的需要3个参数。大致上采集的数据源有 纹理数据，模型数据，比如gradient，multi-position，audio等都是转换为纹理数据来采集的。
对于纹理数据有两种采集方式 Load/Sample，具体看sample.vfx
- Wave:表现了在vfx中实现了的常用的wave，这些wave本身也是一种周期波形。wave.vfx显示了Sawtooth sinewave等



