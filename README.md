[TOC]
> 实现了一个五子棋的训练 https://wandb.ai/opink/Muzero-15-15-5ziqi?workspace=user-opink


# from_alphazero_back_to_bandits
 从实现alphazero开始，回溯到Exploit-Explor问题
>强化学习：bandits和alphazero

## 1.“通用（最小化总遗憾的策略）过程比结果重要”

>问题：如果做（Exploit-Explore trade off）算力聚焦的时候，有一片区域在你的探索过程压根就探索不到的情况下

## 2.选择是一个技术活。专治选择困难症——bandits算法

什么策略才能最大化得到的奖励（最小化N步时的累计遗憾）？通过探索（explore）找到最佳分布的老虎机，并且以一种快速而高效的方法找到这个老虎机，且不断的利用（exploit）它获得最大化的奖励。

1. Thompson sampling算法：每个臂维护一个beta分布的参数，每次试验后，选中一个臂，拉一下，有收益则该臂的wins+=1，否则该臂的loss+=1.每次臂的选择方式是：用每个臂现有的beta分布产生一个随机数x，选择所有臂产生的随机数中最大的那个臂去拉。
2. UCB（Upper Confidence Bound）算法：平均值±变差 (V_i/N_i + (k*ln(N_total)/N_i)^0.5) k正态时选2。主要是抓住了随着试验进行，置信度区间的变动。UCB算法讲究的就是一个乐观，我尽可能的闲心这个ARM能做到它的上确界，正是这种乐观，UCB最后往往都能收敛到最优。
3. Epsilon-Greedy算法：每次以概率epsilon∈（0,1）做一件事，所有壁中随机选一个拉，否则，选择截止当前平均收益最大的那个臂拉一下。（epsilon的值可以控制对E-E trade off ，越接近0，越保守。这个算法也是我们人类大脑可以模拟的极限了）
4. 完全朴素（极大似然）：先平等的试验k次，k轮后，一直选择均值最大的那个臂，这个算法是人类在实际中最常采用的（问题在于k太大影响收益，k小导致严重的过拟合）。

>算法效果对比一目了然：UCB和Thonmpson采样算法显著优秀。至于你实际上要选哪一种bandits算法？（你可以选一种bandits算法来选bandits算法：）

## 3.用bandits算法解决推荐系统冷启动的简单思路

我想，屏幕前的你已经想到了，推荐系统冷启动可以用bandits算法来解决一部分。
>LinUCB算法：单纯的老虎机，他回报情况就是老虎机自己内部决定的，而在广告推荐领域，一个选择的回报，是由state_i（用户状态）和action_i一起决定的，如果我们能用feature来刻画state_i和action_i这一对儿（是不是到这儿才品出来有RL的味了。。），在选择之前通过feature预估每一个arm的期望回报及置信区间，那就合理多了。

利用UCB（最乐观选择，SCB是最悲观）来调整E-E trade off，节点探索的次数越多，置信区间越窄；每次选择置信度区间上界最大的节点max（feature_i + (k*ln(N)/1+Tn_i)^0.5）来平衡探索的次数和未探索的节点的。

原始alphaGo中利用一个V价值神经网络估计期望回报，对featrue_i我们可以加上MCTS，用加权分配，t为拉杆次数，开始时倾向于以预估的feature_i做指导，拉的越多越倾向使用MCTS给出的。max((1-r^(1/t))feature_i + r^(1/t)*UCT)


## 4. 总的来说，AlphaGo Zero分为两个部分，一部分是MCTS（蒙特卡洛树搜索），一部分是神经网络。

### 4.1 从原始的（ALphaGo的）开始
#### 4.1.1 MCTS部分
1. 蒙特卡洛：fastrollout
2. MiniMax Tree Search树搜索：UCT（UCB在树上的版本），select-[optional:expand（对某节点访问次数做个阈值）]-backup

合起来共四个步骤：select-expand-fastrollout-backup

#### 4.1.2 神经网络部分
1. 首先使用模仿学习从人类状态行动对中先监督学出2个神经网络（一个简单的弱鸡fastrollout策略网络P_f，一个复杂一些的策略网路SL Policy Network用于初始化RL Policy Network 用来作为tree policy之后计算PUCT),
2. 而后self-play，得到的对局结果用强化学习学习出2个神经网络RL Policy Network和 Value Network，
3. select时使用PUCT（Tree Policy with SL Policy），fastrollout结果与Value Network结果加权作为backup，

### 4.2 AlphaGo Zero
1. 只有一个网络，输出p和v；让mcts作为策略网络学习的老师，
2. self-play时使用MCTS得到的次数计算出一个概率分布π，根据π选择实际的行动，
3. 没有rollout过程，只有树搜索过程


### 4.3 AlphaZero
推广精髓，老虎机来啦！
> 让策略网络模仿到MCTS的探索效率（Exploit-Explore trade off）。


# 到Muzero，对比学习

看到Muzero和它的改进EfficientZero，发现efficientZero里第一个改进增加监督信号同对比学习里的BYOL、Sim-Siam太像了。
也确实。

## 5. MuZero ： 在state被embeding后的状态隐空间中MCTS
1. 只有在根节点处，muzero会排除规则不允许的行动。 而在之后的隐空间中mcts就没有限制了。
2. 正式走子时一样，选择mcts搜索后第一层最大的N
3. （人类使用图进行推理时，其实是在解码或“翻译”embeding到神经网络空间里的 算子）既然是在隐空间内plan，隐空间也可以再 投射（生成） 到graph的空间里
4. 这就打通了脑子了。投射谬误与沉没成本的联系，机制网络是投射谬误的结构。（要验证这句话，需要第一性。所谓"第一性"就是"若非"）
​


## 6.对比学习(从只要能定义正负例就可以自监督学习)
1. 那大脑其实也在做一样的事，增强现实（同时，也意味着，意识的虚假）
2. （对比学习的损失中，用align和uniform评价，有一部分是正比于在优化最大特征值别太离谱）
3. reward shaping 可以认为是（对比学习中的）代理任务，它不是直接的奖励，所谓进行shaping要有的“先验”可以是显然的命题来自监督用。
4. "这里学一点，那里学一点，那里在隐藏空间中相似可能有探索的价值。。。以为自己比别人多一个“自己的问题"最后看起来像是串起来了，其实只是？？？把ee交换图里在lattin诱导的那个g学出来了"
5. 我直觉的直觉中“理性只有一个功能，禁止（状态延续变化）"。hint：回到脆弱的模样。指的是不是加强探索（可我直觉上要的似乎是某种泛化性，从不同材料得到相同共同的)
6. 引用*所谓“基础好”，说白了是“能够把一切都很容易的变成自己的东西”；走到了这个良性循环之内，才能让人“学的越多、知道的越少”——知道的越少，那么把新东西纳入体系才更不费力。就好像你用“并行计算”轻易把map-reduce串进去一样。*
7. ​两个状态之间的动作（动作也可以embedding抽象成文字描述)，​词向量空间就可以利用计算相似度，评估一些（什么东西）

# 这些字的组织形式只是后来才理顺，打出来的，没啥用。重要的是之前过程中建立联系了的交换图、结构图、和待“若非”前的图的局部碎片。
> ./sim_pics/1~10

# TODO
1. Muzero中策略网络p的代理任务是学的像Bandits（mcts）策略π，那么策略网络的代理任务也可以是学的像因果图上的PC算法的结果。
2. 上述Muzero的机制网络实现是通过时序上的“1个动作诱导从一个状态到另一个状态的态射，把动作看做状态空间上的算子”。那么建立“从2个状态对诱导出一个动作”的生成网络，有什么本质的困难么？
