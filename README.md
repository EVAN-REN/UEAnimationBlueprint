# UEAnimationBlueprint.
Replicate the Lyra animation blueprint.

1.
动画蓝图，ue5中一般将动画接口蓝图，动画实现蓝图，动画配置蓝图
动画接口蓝图为主要动画蓝图，动画实现蓝图有着参数和方法，动画配置蓝图配置一些资产，他继承自动画实现蓝图

transition rule share存储转换规则
使用BlueprintThreadSafeUpdateAnimation，这个中可以update各种数据，是在work thread中更新的，不会影响game thread的速度，在这个里面执行所有的update方法
在每个update方法中，里面其他类中的数据都是通过property access来获取的，是线程安全的，然后来set，update各种数据
在character蓝图开始是调用链接动画层

2.
跑步停止
动画主蓝图：
BlueprintThreadSafeUpdateAnimation链接各种update的function，这些function会update一些变量
创建移动跳跃的状态机，变量会用在状态机改变的条件中
类implement动画接口蓝图
状态机的状态链接接口动画蓝图的接口layer

动画接口蓝图：
创建多个接口layer

动画实现蓝图：
类implement动画接口蓝图
实现接口layer，接口layer上可以直接设置sequence
也可以在update等绑定function，在sequence动态绑定anim，在function使用SetSequenceWithInertialBlending或者SetSequenced和sequence来平滑过渡动画

动画配置蓝图：
在类中配置sequence的anim动画

character蓝图：
使用animation blueprint绑定主蓝图
在event graph中使用link anim class layers来绑定配置蓝图

3.
跳跃条件：与上类似

4.
距离匹配（调整走一步的距离和速度的关系）
动画实现蓝图start里：
将sequence player改为sequence evaluator，并改为sync group同步动画，里面多一个动态绑定explicit time
在relevant函数中改为convert to sequence evaluator，绑定setsequence，加上set explicit time来精确控制动画播放
然后再update函数中使用Advance Time by Distance Matching，允许动画的播放进度根据 角色实际移动的距离 进行推进
Advance Time by Distance Matching需要用到distance curve，可以添加为变量，然后在动画中添加一个同名的curve

动画实现蓝图cycle里：
将sequence player改为follower并加入sync group，update里面加入set playrate to match speed（调整动画播放速率，让动画整体匹配角色速度）

动画实现蓝图stop里：类似start，但在relevant里加入Distance Match to Target（调整动画时间点，让关键帧（停步、落地等）对齐目标位置），绑定曲线
在update函数中绑定advance time

动画实现蓝图fall land里：类似start，但是sequence直接绑定，relevant中设置explicit time
在update中设置Distance Match to Target

5.
步幅适配（调整步幅与播放速率）
动画实现蓝图start里：
在sequence evaluator后面调用stride warping(用于调整角色腿部的步幅长度)，模式改为graph驱动，最小阈值速度为0，
Pelvis Bone 控制角色的骨盆（Pelvis），用于调整角色的整体高度，以适应步幅变化和不同地形（例如斜坡、楼梯）
IK Foot Root Bone 是所有 IK 脚部骨骼的根节点，用于控制角色脚步的位置，确保步幅变化时角色的脚不会漂移
Foot Definitions 定义角色的 左右脚骨骼，用于让 Stride Warping 识别哪些骨骼需要调整步幅

设置stride warping所需的alpha值（Alpha 是一个 0 到 1 之间的数值（通常），0.0 → 完全使用 A 值（不影响原始状态），1.0 → 完全使用 B 值（完全应用目标状态））
然后在relevant函数中设置alpha为0
在update函数中，设置变化的alpha值，并根据alpha值插值设置最低的playrate


动画实现蓝图cycle里：同上
设置stride warping所需的alpha值
在update函数中，根据是否撞到墙来插值地设置变化的alpha值

给动画添加sync maker设置左右脚，这样在相同的同步组中就可以同步

6.
方向适配（适配不同方向）
创建一个枚举Enumeration存储基本方向
创建一个结构体存储四种方向的动画

主动画蓝图中：
在状态机最上层加一个Inertialization（用于在动画切换时平滑过渡，减少动画切换时的突然跳变和生硬感）
在速度计算function中计算出当前速度角朝向角度和速度角度的差值和当前速度值，用function设置这两种角度的枚举方向
创建一个获取方向的function，传入角度，盲区（消除角度180到-180突变和0附近的精度不准问题），方向，是否使用方向，来判断当前枚举方向（设置为pure和线程安全）

动画实现蓝图中：
在start的relevant中将动态动画设置为根据枚举角度使用的结构体动画（用带yaw偏移的）
在start中添加orientation warping（Orientation Warping 让角色在不旋转整个身体的情况下，对齐运动方向，上半身旋转，下半身不动，上下左右之间的过渡）进行方向适配，设置为graph，加入各种spine和ik骨骼，角度设置为带偏移的方向角度
在cycle中添加orientation warping同上，角度设置为不带偏移的方向角度（因为方向已经一致，不需要带偏移，刚启动可能不一致）
在cycle的update中将动态动画设置为根据枚举角度使用的结构体动画（这里用的是不带yaw偏移的）
在stop中的relevant中将动态动画设置为根据枚举角度使用的结构体动画（用带yaw偏移的）

动画配置蓝图中：
重新配置所有的结构体动画

character 蓝图中：
打开character蓝图的use controller rotation yaw，取消orient rotation to movement

7.
回转运动pivot
主动画蓝图中：
在加速度计算的function中加入pivot direction，设置此值为此值与当前加速度的插值，计算pivot方向的角度，然后算出pivot的方向

图层中加入pivot相关图层，pivot里面链接对应layer，并在绑定的relevant函数中设置初始pivot方向，在绑定的update函数中更新pivot时间
pivot到cycle通过状态通知和pivot时间加函数（判断pivot方向与速度方向是否垂直， 检测角色是否需要打破当前 Pivot（急转向），然后决定是否进入一个新的动画状态（比如改变运动方向））

动画实现蓝图中：
创建新的状态机，设置状态为复制和start一样的状态，修改stride warping的alpha值和动画绑定的relevant和update方法
在relevant函数中设置pivot开始加速度，设置set sequence，设置explicit time，设置stride warping的alpha值，设置pivot停止时间，设置主蓝图中的pivot时间
在update函数中设置累积的explicit time，存储评估器的动画，（判断此动画与当前加速度方向动画是否一样，不一样设置为当前加速度动画，并设置pivot开始加速度为当前加速度）在短时间内切换pivot运动动画；然后设置距离匹配（如果加速度和速度相反，角色还在靠近回转运动的点，进行距离匹配）（使用距离匹配是因为我们要尽快结束掉当前的动画序列，以便开始执行新匹配到的新动画序列）用到predict ground movement pivot location（根据当前的移动速度、地面坡度、角色的物理状态等，预测角色在未来某个时间点会在哪个位置 转向 或 枢轴点（Pivot） 发生变化。）然后用distance match；如果加速度和速度不相反，角色要加速远离回转的点，设置时间，提前到动画其余部分（更新stride warping的alpha和应用advance time by distance match）

再加一个状态（为了实现左右交替时，状态可以直接从回转到回转）
过度条件满足加速度和速度方向相反并且角色没有垂直于初始枢轴方向移动并且当前加速度相对 起始枢轴加速度 发生了方向变化

动画配置蓝图中：
配置所有pivot动画

8.
原地旋转
主动画蓝图中：
在rotation function中更新yaw值

创建一个yaw offset的模式（混合（在cycle中平和的混合，使用朝向控制器），保持（保留开始时动画的原始偏移），累加（空闲期间抵消角色旋转））

再创建一个保持更新的函数根据角色的当前状态处理更新yaw的偏移（累加模式中将当前root yaw offset减去每个delta偏移的yaw，并限制在一个范围内，防止过度偏转；混合模式用Float Spring Interp弹簧插播，让root yaw offset随时间逐步趋近0，同时具备物理惯性；最后改为混合模式）

在图层中最外面加入一个rotate root bone使根骨骼根据yaw值旋转
在idle状态机的update函数中，如果正在blend out出idle状态，那么将yaw过度值设为0，如果不是则将root yaw offset mode改为累加状态，并保证如果idle动画的yaw offset过大时，更新角色的root yaw offset
在start状态机的update函数中，如果不在blend out出idle状态时，将root yaw offset mode改为保持状态

动画实现蓝图中：
在idle state中，添加两个状态机（原地旋转和转向恢复）和idle循环调用
在原地旋转的output的relevant中，获取恢复或修正方向，在原地旋转的evaluator中绑定一个relevant来设置旋转方向动画时间为0，explicit time为0， 再绑定一个update，在里面根据修正方向设置对应动画，然后更新一下旋转动画时间，再根据旋转动画时间设置explicit time
再转向恢复里调用sequence player，绑定刚开始position为旋转动画时间，在sequence player的relevant函数中设置和原地旋转update函数中动画绑定部分相同，只不过set sequence用的是sequence player
设置idle到原地旋转，旋转恢复到原地旋转的条件，是当root yaw offset绝对值大于50； 原地旋转到旋转恢复是当旋转曲线值接近0，旋转恢复到idle是自动

动画配置蓝图中：
配置所有旋转动画

9.
Aim offset：
在所有aimoffset相关动画设置一下base pose aimation为最其中基本那个动画
创建aim offset来设置一下这些动画

主动画蓝图中：
在set root yaw offset的function设置aim yaw为root yaw offset的-1，为了保持武器瞄准和相机一致，并创建update aiming data的function来更新pitch
在动画层最外面直接链接aiming接口，并绑定一下参数

在动画实现蓝图中
在aiming的接口中，接收pitch和yaw两个参数，并链接aim offset，给aim offset再连接上base pose和用于动画配置中调整的blend space

动画配置蓝图中：
配置所有aim offset的blend space

10.
脚步IK控制绑定Control Rig：
主动画蓝图中：
在动画层最外面加入一个bool型的control rig，创建一个control rig class（cr_foot来设置脚步控制器）

cr_foot中：
rig hierachy中导入mesh，并且在骨骼中加入各个部分的control

收集姿势到control上：通过Get Transform - Bone获取左右脚骨骼和spine/臀骨控制器的当前变换，再通过Set Transform - Bone设置左右脚ik骨骼和胸部/身体控制器的变换，然后用Parent Constraint将控制器 (Control) 绑定到骨骼 (Bone)绑定左右脚

追踪左右脚：创建一个foot trace function，input加入三个变量（Rig Element Key用于标识角色骨骼、控制器、曲线等元素的唯一键。它用于访问、修改和操作Control Rig 里的元素）/当前命中的法线方向，球体的半径）output加入三个变量（目标偏移的z值，击中位置的法线方向，bool是否在root中）整体function检查了脚前方control处上下50范围内是否有物体，并返回会碰撞物体的z轴和朝向
然后通过这个函数追踪左右脚

然后根据左右脚要调整的参数来调整左右脚：根据左右脚偏移的最小值来调整pelvis offset，用pelvis offset和pelvis的transform来调整pelvis control，然后同样把偏移量放到ik foot root

然后处理脚的偏移：创建一个函数，input加入五个变量（Rig Element Key，当前的z偏移，目标的z偏移，当前击中的法线，是否在根中，输出新的z偏移）函数中先通过pelvis偏移处理目标偏移，然后弹簧插值处理当前值到目标偏移（作为function的z偏移），通过目标偏移处理ik rig的relative transform，得到的结果去设置offset transform，得到的结果去设置control的bone的rotate（用aim math计算 让骨骼朝向某个目标的旋转角度）

根据这个function调整左右脚的偏移

把ik运用到脚上：设置pelvis的bone，再用basic ik设置脚的ik
