# from_alphazero_back_to_bandits
 从实现alphazero开始，回溯到Exploit-Explor问题
>强化学习：bandits和alphazero

## 1.“通用（最小化总遗憾的策略）过程比结果重要”

>问题：如果做（Exploit-Explore trade off）算力聚焦的时候，有一片区域在你的探索过程压根就探索不到的情况下

## 2.选择是一个技术活。专治选择困难症——bandits算法

什么策略才能最大化得到的奖励（最小化N步时的累计遗憾）？通过探索（explore）找到最佳分布的老虎机，并且以一种快速而高效的方法找到这个老虎机，且不断的利用（exploit）它获得最大化的奖励。

1. Thompson sampling算法：每个臂维护一个beta分布的参数，每次试验后，选中一个臂，拉一下，有收益则该臂的wins+=1，否则该臂的loss+=1.每次臂的选择方式是：用每个臂现有的beta分布产生一个随机数x，选择所有臂产生的随机数中最大的那个臂去拉。
2. UCB（Upper Confidence Bound）算法：平均值±变差 (V_i/N_i + (k*ln(N_total)/N_i)^0.5) k正态时选2.
3. Epsilon-Greedy算法：每次以概率epsilon∈（0,1）做一件事，所有壁中随机选一个拉，否则，选择截止当前平均收益最大的那个臂拉一下。（epsilon的值可以控制对E-E trade off ，越接近0，越保守）
4. 完全朴素（极大似然）：先试验一些次，每个臂都有均值后，一直选择均值最大的那个臂，这个算法是人类在实际中最常采用的。

>算法效果对比一目了然：UCB和Thonmpson采样算法显著优秀。至于你实际上要选哪一种bandits算法？你可以选一种bandits算法来选bandits算法。

## 3.用bandits算法解决推荐系统冷启动的简单思路

我想，屏幕前的你已经想到了，推荐系统冷启动可以用bandits算法来解决一部分。
>LinUCB算法：单纯的老虎机，他回报情况就是老虎机自己内部决定的，而在广告推荐领域，一个选择的回报，是由state_i（用户状态）和action_i一起决定的，如果我们能用feature来刻画state_i和action_i这一对儿（是不是到这儿才品出来有RL的味了。。），在选择之前通过feature预估每一个arm的期望回报及置信区间，那就合理多了。

利用UCB（最乐观选择，SCB是最悲观）来调整E-E trade off，节点探索的次数越多，置信区间越窄；每次选择置信度区间上界最大的节点max（feature_i + (k*ln(N)/1+Tn_i)^0.5）来平衡探索的次数和未探索的节点的。

原始alphaGo中利用一个V价值神经网络估计期望回报，对featrue_i我们可以加上MCTS，用加权分配，t为拉杆次数，开始时倾向于以预估的feature_i做指导，拉的越多越倾向使用MCTS给出的。max((1-r^(1/t))feature_i + r^(1/t)*UCT)


## 4. 总的来说，AlphaGo Zero分为两个部分，一部分是MCTS（蒙特卡洛树搜索），一部分是神经网络。

我们是要抛弃人类棋谱的，学会如何下棋完全是通过自对弈来完成。

过程是这样，首先生成棋谱，然后将棋谱作为输入训练神经网络，训练好的神经网络用来预测落子和胜率。

MCTS就是用来自对弈生成棋谱的，
>a. 每次模拟通过选择具有最大行动价值Q的边加上取决于所存储的先验概率P和该边的访问计数N（每次访问都被增加一次）的上限置信区间U来遍历树。
b. 展开叶子节点，通过神经网络来评估局面s；向量P的值存储在叶子结点扩展的边上。
c. 更新行动价值Q等于在该行动下的子树中的所有评估值V的均值。
d. 一旦MCTS搜索完成，返回局面s下的落子概率π，与N^(1/τ)成正比，其中N是从根状态每次移动的访问计数， τ是控制温度的参数。

按照论文所述，每次MCTS使用1600次模拟。过程是这样的，现在AI从白板一块开始自己跟自己下棋，只知道规则，不知道套路，那只好乱下。每下一步棋，都要通过MCTS模拟1600次上图中的a~c，从而得出我这次要怎么走子。
来说说a~c，MCTS本质上是我们来维护一棵树，这棵树的每个节点保存了每一个局面（situation）该如何走子（action）的信息。这些信息是，N(s, a)是访问次数，W(s, a)是总行动价值，Q(s, a)是平均行动价值，P(s, a)是被选择的概率。

##### 4.1.1 a.Select
每次模拟的过程都一样，从父节点的局面开始，选择一个走子。比如开局的时候，所有合法的走子都是可能的选择，那么我该选哪个走子呢？这就是select要做的事情。MCTS选择Q(s, a) + U(s, a)最大的那个action。

- U(s, a) = c_puct × 概率P(s, a) × np.sqrt(父节点访问次数N) / ( 1 + 某子节点action的访问次数N(s, a) )
- feature_i = Q(s_t,a)   ：把当前的局面作为输入传给神经网络，神经网络会返回给我们一个action向量p和当前胜率v。

用论文中的话说，c_puct是一个决定探索水平的常数；这种搜索控制策略最初倾向于具有高先验概率和低访问次数的行为，但是渐近地倾向于具有高行动价值的行为。
计算过后，我就知道当前局面下，哪个action的Q+U值最大，那这个action走子之后的局面就是第二次模拟的当前局面。比如开局，Q+U最大的是当头炮，然后我就Select当头炮这个action，再下一次Select就从当头炮的这个棋局选择下一个走子。

##### 4.1.2 b.Expand
现在开始第二次模拟了，假如之前的action是当头炮，我们要接着这个局面选择action，但是这个局面是个叶子节点。就是说当头炮之后可以选择哪些action不知道，这样就需要expand了，通过expand得到一系列可能的action节点。这样实际上就是在扩展这棵树，从只有根节点开始，一点一点的扩展。

##### 4.1.3 c.Evaluate（fast rollout）
我们使用虚拟损失来确保每个线程评估不同的节点。
因为我们是多线程同时在做MCTS，由于Select算法都一样，都是选择Q+U最大节点，所以很有可能所有的线程最终选择的是同一个节点，这就尴尬了。我们的目的是尽可能在树上搜索出各种不同的着法，最终选择一步好棋，怎么办呢？论文中已经给出了办法，“我们使用虚拟损失来确保每个线程评估不同的节点。”就是说，通过Select选出某节点后，人为增大这个节点的访问次数N，并减少节点的总行动价值W，因为平均行动价值Q = W / N，这样分子减少，分母增加，就减少了Q值，这样递归进行的时候，此节点的Q+U不是最大，避免被选中，让其他的线程尝试选择别的节点进行树搜索。这个人为增加和减少的量就是虚拟损失virtual loss。

##### 4.1.4 c.Backup
现在MCTS的过程越来越清晰了，Select选择节点，选择后，对当前节点使用虚拟损失，通过递归继续Select，直到分出胜负或Expand节点，得到返回值value。现在就可以使用value进行Backup了，但首先要还原W和N，之前N增加了虚拟损失，这次要减回去，之前减少了虚拟损失的W也要加回来。然后开始做Backup.


#### d. Play
按照上述过程执行a~c，论文中是每步棋执行1600次模拟，那就是1600次的a~c，这个MCTS的过程就是模拟自我对弈的过程。模拟结束后，基本上能覆盖大多数的棋局和着法，每步棋该怎么下，下完以后胜率是多少，得到什么样的局面都能在树上找到。然后从树上选择当前局面应该下哪一步棋，这就是步骤d.play:"在搜索结束时，AlphaGo Zero在根节点s0选择一个走子a，与其访问计数幂指数成正比.

在随后的时间步重新使用搜索树：与所走子的动作对应的子节点成为新的根节点；保留这个节点下面的子树所有的统计信息，而树的其余部分被丢弃。

当模拟结束后，对于当前局面（就是树的根节点）的所有子节点就是每一步对应的action节点，选择哪一个action呢？按照论文所说是通过访问计数N来确定的。这个好理解吧？实现上也容易，当前节点的所有节点是可以获得的，每个子节点的信息N都可以获得，然后从多个action中选一个

#### NerosNetwork
上面说过，通过MCTS算出该下哪一步棋。然后接着再经过1600次模拟算出下一步棋，如此循环直到分出胜负，这样一整盘棋就下完了，这就是一次完整的自对弈过程，那么MCTS就相当于人在大脑中思考。我们把每步棋的局面s_t 、算出的action概率向量 π_t 和胜率z_t （就是返回值value）保存下来，作为棋谱数据训练神经网络。

训练目标是最小化预测胜率v和自我对弈的胜率z之间的误差，并使神经网络走子概率p与搜索概率π的相似度最大化。简单点说就是让神经网络的预测跟MCTS的搜索结果尽量接近。

MCTS和神经网络你中有我、我中有你，如此反复迭代，网络预测的更准确，MCTS的结果更强大。在AlphaGo Zero中，自我对弈是由以前所有迭代中最好的玩家生成的。每次训练迭代之后，与最好玩家对弈测量新玩家的能力；如果以55%的优势获胜，那么它将取代最好的玩家，而自我对弈将由这个新玩家产生。

# 到Muzero，对比学习

看到Muzero和它的改进EfficientZero，发现同对比学习里的BYOL、Sim-Siam太像了。
也确实

## 5. MuZero ： 在state被embeding后的状态隐空间中MCTS
1. 只有在根节点处，muzero会排除规则不允许的行动。 而在之后的隐空间中mcts就没有限制了（“重要的是错误。”这意味着可以进行自监督的对比学习了）
2. 正式走子时一样，选择mcts搜索后第一层最大的N
3. （人类使用图进行推理时，其实是在解码或“翻译”embeding到神经网络空间里的 算子）既然是在隐空间内plan，隐空间也可以再 投射（生成） 到graph的空间里
4. 这就打通了脑子了


## 6.对比学习(只要能定义正负例就可以自监督学习)
1. 那大脑其实也在做一样的事，增强现实（同时，也意味着，意识的虚假）
2. （对比学习的损失中，用align和uniform评价，有一部分是正比于在优化最大特征值别太离谱）
3. 错误提供信息，或者说，错误和规则等价
4. "这里学一点，那里学一点，那里在隐藏空间中相似可能有探索的价值。。。以为自己比别人多一个“自己的问题"最后看起来像是穿起来了，其实只是？？？把ee交换图里在lattin诱导的那个g学出来了"
