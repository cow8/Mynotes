# 论文笔记:Espresso

## 摘要
这篇文章的主要贡献是开源了一个基于C与CUDA的self-contained的神经网络量化的库，利用这个库能够构建出在前向时实际能够达到加速的程序。其中主要利用的技术是bit-packing、register-blocking以及基本的xnor,bit-count和bit-shift。在我实际测试中，能够达到X10以上的加速比。这篇文章主要按照一下的顺序介绍:
- 基础知识介绍
- 论文要点回顾
- 实验结果

由于原文提供的代码cmake文件有些问题，且部分测试代码并未公布。目前我还在整理代码，后续我会将测试代码即更新过的cmake文件分享到我的[项目文件夹](https://github.com/cow8/espresso)

## 基础知识介绍
### Cmake语法
- add_subdirectory():指定在指定的subdirectory中再继续找CmakeList.txt
- add_library():指定编译目标是编译出一个库
- find_package(CUDA):找某个包，例如CUDA
### GEMM运算的定义
对于矩阵运算:
> $C=alpha*A*B+beta*C$，其中alpha，beta是实数，A，B，C是矩阵

可以利用SGEMM( TRANSA,  TRANSB,  M,  N,  K,  alpha,  A, LDA, B, LDB, beta, C, LDC)完成相应的运算。这样其实可以通用地定义了矩阵的点积和加法等多种运算。推荐扩展阅读这篇[文章](https://petewarden.com/2015/04/20/why-gemm-is-at-the-heart-of-deep-learning/)
函数（其实并不是函数，而是一个规范）的参数解释如下:
- transa、transb:分别表示A，B是否需要转置
- M:A和矩阵C的行数. 
- N:B和C的列数. 
- K:A和B的列数.
- alpha,beta:实数系数.
- A,B,C:矩阵A,B，一般用 T **A表示，T属于{uint8,uint64,float}
- LDA,LDB,LDC:递增步长，及第一维坐标增加1，相应的地址增量是多少，用于区别行列主序


### CUDA中提供的高效率的函数/操作
- bit-count:
    ```
    __device__ ​ int __popcll ( unsigned long long int x )
        Count the number of bits that are set to 1 in a 64 bit integer.`
    ```
- 位运算
    - \>\>:算术右移
    - \>\>\>:逻辑右移
    - <<,<<<:也类似的是算术、逻辑左移
    - ^:xnor即同或运算
## 论文要点回顾
### 实现量化的流程
- **bit-packing**:将64个参数量化（即按符号转化为0/1）之后放入一个64位的整型，整型中的每一位表示一个参数。
- **基于bit-packing向量点积**
    - 可以证明两个个量化后向量的点积运算可以等价与异或运算与bitcount运算的组合。
    - 但需要满足向量的长度是64的倍数
    - 注意:向量运算完的结果还是利用浮点型存储
    - 量化GEMM:两个矩阵相乘可以看成对应的行向量与列向量相乘，再求和。即可以转化为之前定义的量化的向量积运算。
- 由此可以按照一般的方式定义dense、conv、pool等层级
### Hybrid DNNs:
- 论文分别提供了CPU、GPU、 GPUOPT的代码。
- CPU上的全精度的kernel
- GPU全精度的kernel
- GPU上的BDNN的kernel
### Ｍemory layout:
- **在内存中张量的存储形式，类似矩阵中对行主序和列主序的约定**
- 在L维（channel）上进行bit-packing，在L==1时，在N维上
- 这样的设计在lifting、UNROLL的时候会比较快
### input-binarizartion:
- **即对输入在bit-plane上展开，针对每个bit做运算，最终再合并**
- 以[0,255]的颜色为例，像素值若是127
- 则变成0111,1111
- 相乘的结果再根据位置做weight
- *实际上没有实现(参见src/kernel/pinputl.cu)*
### bit-packing带来的问题:
- 若不满足64位填不满的填-1可能造成很严重的误差比如
- 因此，**严格要求进行矩阵运算的张量A(D,M,N,L)中M\*N\*L是64的倍数**。由此对在此之上定义的层级、网络都有限制。但对一般的网络而言影响不大，因为一般都是64的倍数。

## 实验结果
### GEMM速度
- configuration :1060,8192*8192相乘,Batchsize=1,Average on 10
- 不包含设置时间，启动时间
- 0.8us pgemm 得到float数,9us得到量化数
- 16.6us sgemm得到float数
- 注意，当重复测试10次时速度较快，随重复测试次数增加速度减慢
### 在minist上inference的性能 
- configuration: 1060,cuda 8.0、mnist、batch_size=512、average on 10000 iters、mlp（无卷积）
- 不包含启动时间、不包含数据读取时间
- pytorch上的全精度相同结构的对照:173.22 us/input
- espresso:13 us/input
### 正确性验证
- 全精度全连接层（输入1\*2\*3\*1,权重2\*6,输出2\*1）
```
输入:
35.000000 23.000000 34.000000 -43.000000 -45.000000 62.000000 | 
113.000000 124.000000 121.000000 -7.000000 -21.000000 -46.000000 |
权重:
-108.000000 -109.000000 105.000000 | 
98.000000 -83.000000 -47.000000 |
输出:
-6110.000000 -9796.000000 |
正确
```
- 量化的全连接层（对输入的形状是有要求的！）
```
经测试（在test.cu中）
正确
```
















