由于目前的深度学习框架已经足够简单，足够好用了，使用者唯一的负担几乎就是搞清楚一个模块的输入输出，但这往往会很让人困扰，本文讲解`nn.RNN` 和 `nn.LSTM` 的输入输出。

## nn.RNN

### 输入

我们先来看官方文档:

- **input_size** – 输入变量的特征数
- **hidden_size** – 隐藏状态的特征
- **num_layers** - 层数
- **batch_first** - 如果此为`true` , 那么输入输出的变量的维度会被规定为(batch, seq, feature)， 这很重要，对于我自己而言，我一般会要求`batch_first`. 

```python
input_dim = 5    # 每个输入元素的特征维度,如每个单词用长度为 5 的特征向量表示
hidden_dim = 6   # 隐状态的特征维度,如每个单词在隐藏层中用长度为 6 的特征向量表示
rnn_layers = 4   # 循环层数
rnn = nn.RNN(input_size=input_dim, hidden_size=hidden_dim, num_layers=rnn_layers, batch_first=True)
```

这段代码展示我们创建循环层的过程，

此时，我们的输入变量的维度应该为`(batch_size,seq,input_dim)`, 否则不合法。

当我们需要用循环层去做计算时，我们需要输入两个变量`input, h_0`, 其中`h_0` 是隐藏状态的初始值。

`h_0` 的维度应该为 `(rnn_layers,batch_size,hidden_dim)`

```python
batch_size = 2
sequence_length = 3    # 输入的序列长度，如 i love you 的序列长度为 3,每个单词用长度为 feature_num 的特征向量表示
input = torch.randn(batch_size, sequence_length, embed_dim)
h0 = torch.randn(rnn_layers, batch_size, hidden_dim)
```

### 输出

```python
output, hn = rnn(input, h0)
```

### output

`output` 表示每个样本在每个时间的最后一层隐藏状态，维度为`(batch_size,seq,hidden_dim)`

### hn

`hn` 表示所有样本在最后一个时间的每一次层的隐藏状态，维度为 `(rnn_layers,batch_szie,hidden_dim)`

因此我们有`output[:,-1,:] = hn[-1,:,:]`.

## nn.LSTM

- **input_size** -  输入变量特征的维数
- **hidden_size** - 隐藏状态`h`的特征维数
- **num_layers** - 层数

与`nn.RNN` 差不多

### 输入

- **input**: 维度 `(batch_size,seq,input_dim)`
- **(h_0, c_0)**` : `h0` 和 `c0` 组成的元组，它们的维度都一样 (rnn_layers,batch_size,hidden_dim)`

### 输出

- **output** ： 每个样本在每个时间点的最后一个隐藏状态，维度`(batch_size,seq,hidden_dim)`
- **h_n** :每个样本在最后一个时间的每一层的隐藏状态， 维度`(rnn_layers,batch_size,hidden_dim)`
- **c_n**： 每个样本在最后一个时间的每一次层的细胞状态，维度`(rnn_layers,batch_size,hidden_dim)`