# 神经网络加速技巧

## 摘要
在谷歌的这篇论文中介绍了一种weight folding的变换技巧，能够避免在前向时BN层的计算，从而达到有效的加速效果。下面是证明过程

## 符号约定
- conv() 卷积运算
- I 卷积输入四维张量，大小为(Batchsize,C_in,H,W)
- X 卷积输出四维张量，大小为(Batchsize,C_out,H,W)
- O Bn层输出的四维张量，大小为(Batchsize,C_out,H,W)
- weight 卷积核四维张量，大小为(C_in,C_out,K,K)
- gamma 公式中的Gamma，有时也表示做weight,大小(C_out)
- beta  公式中的Beta，有时也表示做bias,大小(C_out)
- mean() 表示采用EMA的方式取均值运算，运算结果与bn中运算一致为(C_out)维向量
- dev() 表示采用EMA的方式取标准差运算，运算结果与bn中运算一致为(C_out)维向量
- sigma 表示训练完成后固定下来的标准差，(C_out)维向量
- mu 表示训练完成之后固定下来的均值，(C_out)维向量
- *,+,\ 都是对应元素的张量运算，若张量维数不匹配则采用类似broadcast机制利用相同元素填充

## 问题描述

对与conv->batchnorm2d组成的一个block在正向时的计算等效与一个新的conv运算。更为具体的，新的conv运算的参数表达式为:
$$
new\_weight=gamma*weight/sigma \tag{1}
$$
$$
new\_bias=beta-gamma*mu/sigma \tag{2}
$$
原本的正向传播的计算过程为:
$$
X=conv(I,weight) \tag{3}
$$
$$
O=gamma*(X-mean(X))/var(X)+beta \tag{4}
$$

*注意在$mean(X)$是C_out维的向量，而$X$是4维张量，其维数不匹配。因此$X-mean(X)$需要进行broadcast，即将$mean(X)$中与$X$匹配的那一维对齐，在其余维采用填充操作。其余的运算也采用类似的操作*

## 证明过程
将公式(3)带入公式(4)中得到下式：

$$
O=gamma*(conv(I,weight)-mean(conv(I,weight)))/var(conv(I,weight))+beta
\tag{4}
$$
当正向传播时，running_mean和running_var不再更新只作为常数参与运算。因此$mean(conv(I,weight))$,$var(conv(I,weight))$可以看做常量用$sigma,mu$表示即：
$$
O=gamma*(conv(I,weight)-mu)/sigma+beta
$$

此时$sigma,miu,gamma,weight,beta$都是常量，因此整个式子可以写作：
$$
O=gamma*(conv(I,weight)-mu)/sigma+beta\\
=(gamma*conv(I,weight)-gamma*mu)/sigma+beta\\
=gamma*conv(I,weight)/sigma+beta-gamma*mu/sigma\\
=conv(I,gamma*weight)/sigma+beta-gamma*mu/sigma\\
=conv(I,new\_weight)/sigma+new\_bias
$$
可能有人会对$gamma$乘进卷积运算中有些疑问，可以将所有运算在C_out维上展开，取其中的每个元素进行运算。则$gamma$、$sigma$等都变为了标量数乘，这样可能会比较好理解。
