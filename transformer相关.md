来自苏神博客：https://spaces.ac.cn/archives/8747
### 梯度消失
1. 它指的是（主要是在模型的初始阶段）越靠近输入的层梯度越小，趋于零甚至等于零，而我们主要用的是基于梯度的优化器，所以梯度消失意味着我们没有很好的信号去调整优化前面的层。
2. 换句话说，前面的层也许几乎没有得到更新，一直保持随机初始化的状态；只有比较靠近输出的层才更新得比较好，但这些层的输入是前面没有更新好的层的输出，所以输入质量可能会很糟糕（因为经过了一个近乎随机的变换），因此哪怕后面的层更新好了，总体效果也不好。

### Transformer如何解决梯度消失
1.残差链接，
2. LN不能解决梯度消失问题，反倒是导致梯度消失的原因之一。LN用的post norm
3. Warmup
4. 初始标准差是0.02也是为了缓解梯度消失

### Warmup如何在post norm上缓解梯度消失问题
1. Warmup是在训练开始阶段，将学习率从0缓增到指定大小，而不是一开始从指定大小训练。如果不进行Wamrup，那么模型一开始就快速地学习，由于梯度消失，模型对越靠后的层越敏感，也就是越靠后的层学习得越快，然后后面的层是以前面的层的输出为输入的，前面的层根本就没学好，所以后面的层虽然学得快，但却是建立在糟糕的输入基础上的。
2. 很快地，后面的层以糟糕的输入为基础到达了一个糟糕的局部最优点，此时它的学习开始放缓（因为已经到达了它认为的最优点附近），同时反向传播给前面层的梯度信号进一步变弱，这就导致了前面的层的梯度变得不准。但我们说过，Adam的更新量是常数量级的，梯度不准，但更新量依然是数量级，意味着可能就是一个常数量级的随机噪声了，于是学习方向开始不合理，前面的输出开始崩盘，导致后面的层也一并崩盘。
3. 所以，如果Post Norm结构的模型不进行Wamrup，我们能观察到的现象往往是：loss快速收敛到一个常数附近，然后再训练一段时间，loss开始发散，直至NAN。如果进行Wamrup，那么留给模型足够多的时间进行“预热”，在这个过程中，主要是抑制了后面的层的学习速度，并且给了前面的层更多的优化时间，以促进每个层的同步优化。
这里的讨论前提是梯度消失，如果是Pre Norm之类的结果，没有明显的梯度消失现象，那么不加Warmup往往也可以成功训练。

### 既然LN不能解决梯度消失问题，为什么不去掉
1. 它稳定了前向传播的数值，并且保持了每个模块的一致性。比如BERT base，我们可以在最后一层接一个Dense来分类，也可以取第6层接一个Dense来分类；但如果你是Pre Norm的话，取出中间层之后，你需要自己接一个LN然后再接Dense，否则越靠后的层方差越大，不利于优化。
2. 梯度消失也不全是“坏处”，对于Finetune阶段来说，它反而是好处。在Finetune的时候，我们通常希望优先调整靠近输出层的参数，不要过度调整靠近输入层的参数，以免严重破坏预训练效果。而梯度消失意味着越靠近输入层，其结果对最终输出的影响越弱，这正好是Finetune时所希望的。所以，预训练好的Post Norm模型，往往比Pre Norm模型有更好的Finetune效果

### 初始标准差是0.02如何缓解梯度消失
1.，如果x的方差是1，F(x)的方差是σ2，那么初始化阶段，Norm操作就相当于除以√‾‾‾‾‾‾‾1+σ2。如果σ比较小，那么残差中的“直路”权重就越接近于1，那么模型初始阶段就越接近一个恒等函数，就越不容易梯度消失。简单地将初始化标注差设小一点，就可以使得σ变小一点，从而在保持Post Norm的同时缓解一下梯度消失，何乐而不为？那能不能设置得更小甚至全零？一般来说初始化过小会丧失多样性，缩小了模型的试错空间，也会带来负面效果。

### 为什么MLM要多加Dense？
1. 还是跟BERT用了0.02的标准差初始化直接相关。刚才我们说了，这个初始化是偏小的，如果我们不额外加Dense就乘上Embedding预测概率分布，那么得到的分布就过于均匀了（Softmax之前，每个logit都接近于0），于是模型就想着要把数值放大。现在模型有两个选择：第一，放大Embedding层的数值，但是Embedding层的更新是稀疏的，一个个放大太麻烦；第二，就是放大输入，我们知道BERT编码器最后一层是LN，LN最后有个初始化为1的gamma参数，直接将那个参数放大就好。
2. 模型优化使用的是梯度下降，我们知道它会选择最快的路径，显然是第二个选择更快，所以模型会优先走第二条路。这就导致了一个现象：最后一个LN层的gamma值会偏大。如果预测MLM概率分布之前不加一个Dense+LN，那么BERT编码器的最后一层的LN的gamma值会偏大，导致最后一层的方差会比其他层的明显大，显然不够优雅；而多加了一个Dense+LN后，偏大的gamma就转移到了新增的LN上去了，而编码器的每一层则保持了一致性。

