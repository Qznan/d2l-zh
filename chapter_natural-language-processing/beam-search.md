# 束搜索

上一节介绍了如何训练输入和输出均为不定长序列的编码器—解码器。本节我们介绍如何使用编码器—解码器来预测不定长的序列。

上一节里已经提到，在准备训练数据集时，我们通常会在样本的输入序列和输出序列后面分别附上一个特殊符号“&lt;eos&gt;”表示序列的终止。我们在接下来的讨论中也将沿用上一节的全部数学符号。为了便于讨论，假设解码器的输出是一段文本序列。设输出文本词典$\mathcal{Y}$（包含特殊符号“&lt;eos&gt;”）的大小为$\left|\mathcal{Y}\right|$，输出序列的最大长度为$T'$。所有可能的输出序列一共有$\mathcal{O}(\left|\mathcal{Y}\right|^{T'})$种。

进一步的，假设以“&lt;eos&gt;”结尾的序列才是可用序列，且“&lt;eos&gt;”不能在序列首位，在此限制下，根据“&lt;eos&gt;”可能出现的位置，最终所有可能的输出序列会减少为$\left|\mathcal{Y}\right|^{1}+\left|\mathcal{Y}\right|^{2}+...+\left|\mathcal{Y}\right|^{T'-1}=\frac{\left|\mathcal{Y}\right|^{T'}-\left|\mathcal{Y}\right|}{\left|\mathcal{Y}\right|-1}$。可见，即使有限制条件，可能的序列仍接近$\mathcal{O}(\left|\mathcal{Y}\right|^{T'})$种。


## 贪婪搜索

让我们先来看一个简单的解决方案：贪婪搜索（greedy search）。对于输出序列任一时间步$t'$，我们从$|\mathcal{Y}|$个词中搜索出条件概率最大的词

$$y_{t'} = \operatorname*{argmax}_{y \in \mathcal{Y}} P(y \mid y_1, \ldots, y_{t'-1}, \boldsymbol{c})$$

作为输出。一旦搜索出“&lt;eos&gt;”符号，或者输出序列长度已经达到了最大长度$T'$，便完成输出。

我们在描述解码器时提到，基于输入序列生成输出序列的条件概率是$\prod_{t'=1}^{T'} P(y_{t'} \mid y_1, \ldots, y_{t'-1}, \boldsymbol{c})$。我们将该条件概率最大的输出序列称为最优输出序列。而贪婪搜索的主要问题是不能保证得到最优输出序列。

下面来看一个例子。假设输出词典里面有“A”“B”“C”和“&lt;eos&gt;”这4个词。图10.9中每个时间步下的4个数字分别代表了该时间步生成“A”“B”“C”和“&lt;eos&gt;”这4个词的条件概率。在每个时间步，贪婪搜索选取条件概率最大的词。因此，图10.9中将生成输出序列“A”“B”“C”“&lt;eos&gt;”。该输出序列的条件概率是$0.5\times0.4\times0.4\times0.6 = 0.048$。


![在每个时间步，贪婪搜索选取条件概率最大的词](../img/s2s_prob1.svg)


接下来，观察图10.10演示的例子。与图10.9中不同，图10.10在时间步2中选取了条件概率第二大的词“C”。由于时间步3所基于的时间步1和2的输出子序列由图10.9中的“A”“B”变为了图10.10中的“A”“C”，图10.10中时间步3生成各个词的条件概率发生了变化。我们选取条件概率最大的词“B”。此时时间步4所基于的前3个时间步的输出子序列为“A”“C”“B”，与图10.9中的“A”“B”“C”不同。因此，图10.10中时间步4生成各个词的条件概率也与图10.9中的不同。我们发现，此时的输出序列“A”“C”“B”“&lt;eos&gt;”的条件概率是$0.5\times0.3\times0.6\times0.6=0.054$，大于贪婪搜索得到的输出序列的条件概率。因此，贪婪搜索得到的输出序列“A”“B”“C”“&lt;eos&gt;”并非最优输出序列。

![在时间步2选取条件概率第二大的词“C”](../img/s2s_prob2.svg)

## 穷举搜索

如果目标是得到最优输出序列，我们可以考虑穷举搜索（exhaustive search）：穷举所有可能的输出序列，输出条件概率最大的序列。

虽然穷举搜索可以得到最优输出序列，但它的计算开销$\mathcal{O}(\left|\mathcal{Y}\right|^{T'})$很容易过大。例如，当$|\mathcal{Y}|=10000$且$T'=10$时，我们将评估$10000^{10} = 10^{40}$个序列：这几乎不可能完成。而贪婪搜索的计算开销是$\mathcal{O}(\left|\mathcal{Y}\right|T')$，通常显著小于穷举搜索的计算开销。例如，当$|\mathcal{Y}|=10000$且$T'=10$时，我们只需评估$10000\times10=10^5$个序列。


## 束搜索

束搜索（beam search）是对贪婪搜索的一个改进算法。它有一个束宽（beam size）超参数。我们将它设为$k$。在时间步1时，选取当前时间步条件概率最大的$k$个词，分别组成$k$个候选输出序列的首词。在之后的每个时间步，基于上个时间步的$k$个候选输出序列，从$k\left|\mathcal{Y}\right|$个可能的输出序列中选取条件概率最大的$k$个，作为该时间步的候选输出序列。最终，我们从各个时间步的候选输出序列中筛选出包含特殊符号“&lt;eos&gt;”的序列，并将它们中所有特殊符号“&lt;eos&gt;”后面的子序列舍弃，得到最终候选输出序列的集合。


![束搜索的过程。束宽为2，输出序列最大长度为3。候选输出序列有$A$、$C$、$AB$、$CE$、$ABD$和$CED$](../img/beam_search.svg)

图10.11通过一个例子演示了束搜索的过程。假设输出序列的词典中只包含5个元素，即 $\mathcal{Y} = \{A, B, C, D, E\}$ ，且其中一个为特殊符号“&lt;eos&gt;”。设束搜索的束宽等于2，输出序列最大长度为3。在输出序列的时间步1时，假设条件概率 $P(y_1 \mid \boldsymbol{c})$ 最大的2个词为 $A$ 和 $C$ 。我们在时间步2时将对所有的 $y_2 \in \mathcal{Y}$ 都分别计算 $P(y_2 \mid A, \boldsymbol{c})$ 和 $P(y_2 \mid C, \boldsymbol{c})$ ，并重新累乘前面时间步的条件概率来计算截至当前时间步的条件概率，即 $P(A,y_2 \mid \boldsymbol{c})=P(A \mid \boldsymbol{c})P(y_2 \mid A, \boldsymbol{c})$ 和 $P(C, y_2 \mid \boldsymbol{c})=P(C \mid \boldsymbol{c})P(y_2 \mid C, \boldsymbol{c})$ ，以此选出10个条件概率中最大的2个，假设为 $P(A, B \mid \boldsymbol{c})$ 和 $P(C, E \mid \boldsymbol{c})$ 。那么，我们在时间步3时将对所有的 $y_3 \in \mathcal{Y}$ 都分别计算 $P(y_3 \mid A, B, \boldsymbol{c})$ 和 $P(y_3 \mid C, E, \boldsymbol{c})$ ，并同样累乘得到 $P(A, B, y_3 \mid \boldsymbol{c})$ 和 $P(C, E, y_3 \mid \boldsymbol{c})$ 后，选出10个条件概率中最大的2个，假设为 $P(A, B, D \mid \boldsymbol{c})$ 和 $P(C, E, D \mid \boldsymbol{c})$ 。如此迭代下去直到限定的最大时间步数，并输出最终2个里面具有“&lt;eos&gt;”的有效序列及其对应的条件概率 $P(y_{1:n} \mid \boldsymbol{c})$ ，其中 $y_{n+1}$ 为“&lt;eos&gt;”。

这里值得注意两点：

1. 集束搜索过程中，还以上文为例，假设时间步1保留了 $A$ 和 $C$ ,时间步2却可能最终会保留成 $A,B$ 和 $A,D$ ,即第一步为 $C$ 的可能序列将不被考虑了。显然这将发生在 $P(A, B \mid \boldsymbol{c})$ 和 $P(A, D \mid \boldsymbol{c})$ 均大于任意以 $C$ 为第一步的可能组合即 $P(C, y_2 \in \mathcal{Y} \mid \boldsymbol{c})$ 的时候。集束搜索相当于每一步均做了一定的裁剪，并且只有到最后一步，才能确定保留序列的具体形式。

2. 对“&lt;eos&gt;”的处理。假设某个时间步保留的条件概率最大的 $k$ 个序列里，某个或某几个序列遇到了“&lt;eos&gt;”，设个数为 $n$ ，此时一般有两种处理方式：
* 将这n个序列连同对应的条件概率先提取出来，此时保留的序列变为 $k-n$ 个，可以找到原先比较时条件概率第 $k+1$ 至 $k+n-1$ 大的序列补上。而在工程上，一般事先直接保留 $2k$ 个候选序列，再从中抽出遇到“&lt;eos&gt;”的序列后，再保留前 $k$ 个。因为这样能考虑到万一前 $k$ 个序列均遇到“&lt;eos&gt;”的极端情况。该方法的结束条件为达到最大的时间步数，并且最终得到的序列个数显然可能大于 $k$ 个。
* 不动这 $n$ 个序列，但是需要通过一定的操作保证已经遇到“&lt;eos&gt;”的序列在后续步骤中不管计算后面的任何字符，其条件概率不改变，即 $P(y_{1:n}, eos \mid \boldsymbol{c})=P(y_{1:n}, eos,... \mid \boldsymbol{c})$ ， $“...”$ 表示任何字符序列。如此才能保证后续的概率比较中比较的是有效序列 $P(y_{1:n}, eos \mid \boldsymbol{c})$ 的真正概率。

在最终候选输出序列的集合中，我们取以下分数最高的序列作为输出序列：

$$ \frac{1}{L^\alpha} \log P(y_1, \ldots, y_{L}) = \frac{1}{L^\alpha} \sum_{t'=1}^L \log P(y_{t'} \mid y_1, \ldots, y_{t'-1}, \boldsymbol{c}),$$

其中 $L$ 为最终候选序列长度， $\alpha$ 一般可选为0.75。首先，直觉上对概率进行累乘得越多，则概率越小。若取对数作为分数，则是对负数进行累加，累加得越多，分数值同样越小(分数最高为0，对应概率为1)。所以若不以 $L$ 为分母对长度进行归一化，则分数最高的序列总是偏向于取短序列。而若 $\alpha<1$ 其作用是对短序列进行鼓励，惩罚长序列。通俗理解是长序列分数归一化程度会比短序列归一化程度弱，即长序列分数绝对值相对更大，所以分数会更小。若 $\alpha>1$ 则反之。最后分析可知，束搜索的计算开销为 $\mathcal{O}(k\left|\mathcal{Y}\right|T')$ 。这介于贪婪搜索和穷举搜索的计算开销之间。此外，贪婪搜索可看作是束宽为1的束搜索。束搜索通过灵活的束宽 $k$ 来权衡计算开销和搜索质量。

## 小结

* 预测不定长序列的方法包括贪婪搜索、穷举搜索和束搜索。
* 束搜索通过灵活的束宽来权衡计算开销和搜索质量。


## 练习

* 穷举搜索可否看作特殊束宽的束搜索？为什么？
* 在[“循环神经网络的从零开始实现”](../chapter_recurrent-neural-networks/rnn-scratch.md)一节中，我们使用语言模型创作歌词。它的输出属于哪种搜索？你能改进它吗？




## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/6817)

![](../img/qr_beam-search.svg)
