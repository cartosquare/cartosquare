---
title: 如何衡量机器学习的好坏 
date: 2017-11-24 19:31:20
categories:
  - 算法
tags:
  - 深度学习
  - 损失函数
  - SVMLoss
  - SoftMax
  - CrossEntropy
---

损失函数定义了机器学习的目标，是构建机器学习结构十分重要的一环。通常，损失函数包含数据损失(data loss)和正则化损失(regularization loss)，前者约束评分函数预测的结果要和真实结果尽可能相近，后者约束的是评分函数的参数要尽可能简单，避免过复杂的模型。对于不同的任务（分类、回归等），需要设计不同类型的损失函数，当然使用最多的损失函数就只有常见的几种，下面分别介绍。
<!-- more -->

## 数据损失
### 适用于分类的损失函数
对于一个N类的分类任务，对于每一个输入的样本，评分函数都会输出一个N维的向量，这个向量表明了该样本对应于某个类的得分（向量的第一个值表明该样本属于第一类的得分，第n个值表明该样本属于第n类的得分），得分最高的那个类别就是电脑预测的类别结果。

在训练的时候，我们还有每个样本真实的类别数据，通过比较电脑预测的结果和真实结果的差异，我们就能知道电脑预测的好坏，损失函数就是用来做比较差异这件事情的，所以损失函数会以电脑的预测结果和真实的类别结果为输入，输出一个实数值。

用数学语言总结一下，对于N类的分类任务，对于样本$x_i$，电脑预测的结果，回忆一下上一篇blog，就是评分函数输出的结果：
$$s\_j = f(x\_i,W)\_j$$
函数f会输出一个N维的得分向量，$f(x\_i,W)\_j$表示这个向量的第j个元素，即这个样本属于第j类的得分。假设这个样本的真实的类别是$y_i$，那么损失函数是所有样本的损失的平均：

$$L = \frac{1}{N}\sum\_{i=0}^N l\_i(f(x\_i,W),y\_i)$$

#### Multiclass Support Vector Machine loss
多类支持向量机损失函数是这样一个损失函数：它希望正确类别的得分比其他类别的得分高于某个固定的值$\Delta$，通常可以将$\Delta$设为1。于是，第i个样本的SVM损失值可以用下面的式子表示：
$$L\_i=\sum\_{j\neq{y\_i}} max(0, s\_j - s\_{y\_i} + \Delta)$$
其中$s\_j$是预测的第j类的得分，$y\_i$是样本$x\_i$的真实类别, $s\_{y\_i}$是在真实类别上的得分。注意到这里求和的不包括真实类别的那一项，而是将其它类别的得分与真实类别的得分相减，看是否大于固定值$\Delta$，如果大于的话就累加到loss里去，如果小于的话损失就置为0（说明其他类别的得分都小于真实类别的得分）。
下面是在caffe2中的实现代码，关于如何在caffe2中添加自定义层可以参考[这里](https://caffe2.ai/docs/custom-operators.html)。
```c++
template <>
bool SVMLossL1Op<float, CPUContext>::RunOnDevice() {
  // (1) 设置输入输出
  auto& X = Input(0);
  const float* Xdata = X.data<float>();

  auto& labels = Input(1);
  const int* labels_data = labels.data<int>();

  auto* Y = Output(0);
  const auto canonical_axis = X.canonical_axis_index(axis_);
  const int N = X.size_to_dim(canonical_axis);
  const int D = X.size_from_dim(canonical_axis);
  Y->Resize(N);
  float* Ydata = Y->mutable_data<float>();

  // （2）获取真实标签所对应的得分
  if (correct_class_score_.size() != N) {
    correct_class_score_.Resize(N);
  }
  float* correct_class_score_data = correct_class_score_.mutable_data<float>();
  math::Select<float, CPUContext>(N, D, Xdata, labels_data,
                                  correct_class_score_data, &context_);

  // (3) 计算每一个样本的svm损失
  // sum(max(0, result X - correct_class_score(X) + 1))
  for (int i = 0; i < N; ++i) {
    Ydata[i] = 0.0;
    for (int j = 0; j < D; ++j) {
      if (j != labels_data[i]) {
        // sum wrong class margin
        Ydata[i] += std::max<float>(
            0.0, Xdata[i * D + j] - correct_class_score_data[i] + 1);
      }
    }
  }

  return true;
}
```
* (1) 设置输入和输出。Loss函数有两个输入，一个是评分函数预测的每类的得分，假设共有D类，那么X应该是一个D维的向量，但是通常我们会设置一个batch的数据过网络，假设有N个数据一起过网络，那么X就是$N \times D$维。labels是N个数据真实的标签，是一个N维的变量。Loss函数只有一个输出，是一个N维的向量。

* (2）获取真实标签所对应的得分。这里对应的是$s\_{y\_i}$
* (3) 计算每一个样本的svm损失。

多类支持向量机损失函数有一些变种，比如，为了让损失函数便于求导，求和时可以加上平方：
$$L\_i=\sum\_{j\neq{y\_i}} max(0, s\_j - s\_{y\_i} + \Delta)^2$$


#### Cross-Entropy loss
在理解交叉熵损失函数之前，首先看一下Softmax操作。回想一下SVM loss，它的输入是一个N维的评分向量（N表示总类别数），这里的评分是在实数域上的，也就是说可以是任意的实数。而Softmax操作可以将这N个数归一化到[0,1]之间，并且使这N个数之和为1。这种方式实际上模拟了类别的概率分布，是我们大概可以看出样本属于不同类别的概率（但是实际上这些值并不是真实的概率，也就是说，他们之间的相对大小是有意义的，但是绝对值是无意义的）。交叉熵损失函数可以在Softmax的基础之上定义：
$$L\_i=-log(\frac{e^{f\_{y\_i}}}{\sum\_j{e^{f\_{y\_j}}}})$$
其中log里面的这部分：
$$\frac{e^{f\_{y\_i}}}{\sum\_j{e^{f\_{y\_j}}}}$$
就是Softmax操作。可见交叉熵就是对Softmax的结果做log然后取相反数。
交叉熵为什么这么定义，可以从两个角度来理解。
首先从信息论的角度来看，在信息论中，两个分布p和q的交叉熵由下式定义：
$$H(p,q) = -\sum\_{x}p(x)log(q(x))$$
回到交叉熵损失函数，假设p(x)是数据的真实分布，q(x)是电脑预测的分布，那么交叉熵损失函数就是希望这两个分布的交叉熵尽可能小。注意到，在交叉熵损失函数里并没有出现q(x)，那是因为对于样本$x\_i$,p(x) = [0, ..., 1, ..., 0]，实际上已经隐含在式子中了(只出现了正确类别的那项，因为只有这项p(x)为1）。
另外，p和q的交叉熵还可以表示为p的熵加上p、q之间的KL距离(Kullback-Leibler divergence)：
$$H(p,q) = H(p) + D\_{KL}(p||q)$$
因为 H(p)为0，那么交叉熵实际上等于p和q之间的KL距离，这个距离表示的是两个分布之间的距离。
总结一下，交叉熵损失函数希望的是实际分布p和预测分布q能够尽可能地接近。
从另一个角度，概率论的角度看，有：
$$P(y\_i|x\_i;W)=\frac{e^{f\_{y\_i}}}{\sum\_j{e^{f\_{y\_j}}}}$$
上式表示的是给定样本$x\_i$和参数集合W，这个样本是真实类别$y\_i$的概率。因此从概率的角度看，最小化正确类别的负的log似然，可以认为是在做极大似然估计（Maximum Likelihood Estimation, MLE）。
首先来看Softmax函数在caffe2中的实现(有删减):
```c++
// (1)
void SoftmaxCPU(
    CPUContext& context,
    const int N,
    const int D,
    const float* Xdata,
    float* Ydata,
    float* scale,
    const float* sum_multiplier,
    float* rowmax) {
  math::RowwiseMax<float, CPUContext>(N, D, Xdata, rowmax, &context);
  // Put the intermediate result X - max(X) into Y
  context.template Copy<float, CPUContext, CPUContext>(N * D, Xdata, Ydata);
  // (2) Subtract the max (for nomuerical reasons)
  math::Gemm<float, CPUContext>(
      CblasNoTrans,
      CblasNoTrans,
      N,
      D,
      1,
      -1,
      rowmax,
      sum_multiplier,
      1,
      Ydata,
      &context);
  // (3) Exponentiation
  math::Exp<float, CPUContext>(N * D, Ydata, Ydata, &context);
  math::Gemv<float, CPUContext>(
      CblasNoTrans, N, D, 1, Ydata, sum_multiplier, 0, scale, &context);
  // (4) Do division
  for (int i = 0; i < N; ++i) {
    for (int j = 0; j < D; ++j) {
      Ydata[i * D + j] /= scale[i];
    }
  }
}
```
* (1) Softmax 函数的输入包括一个$N \times D$维的评分矩阵Xdata,其中N为batch size，D为类别数。Ydata是softmax的计算结果。

* (2) 为了避免数值错误，可以先把每一个样本的评分都减去最大评分。其中 math::Gemm 是General matrix multiply 函数，可以计算两个矩阵相乘。这里是把Xdata减去每一行的最大值放到了Ydata里。

* (3) 计算$e^{f\_{y\_i}}$ 和${\sum\_j{e^{f\_{y\_j}}}}$

* (4) 计算$\frac{e^{f\_{y\_i}}}{\sum\_j{e^{f\_{y\_j}}}}$

有了Softmax的输出结果，交叉熵只需把真实标签对应下的值求log并取相反数即可：
```c++
for (int i = 0; i < N; ++i) {
    Ydata[i] = -log(std::max(Xdata[i * D + labelData[i]], kLOG_THRESHOLD()));
  }
```

### 适用于回归的损失函数
回归任务需要预测一个实数值，比如房价。此时通常用范数来定义损失函数，比如L1、L2范数等。
#### L2 Squared norm
L2平方范数描述的是预测值和实际值的L2范数距离,下面是对单个样本的计算公式：
$$L\_i=||f - y\_i||_2^2$$

L1范数则是把各个维度的差异累加起来：
$$L\_i=||f - y\_i||_1 = \sum\_j|f\_j - (y\_i)\_j|$$

相比于L1范数，L2范数更适合求导，但是L1范数却更鲁棒。
从优化方面看，L2范数会比Softmax、SVM loss难优化得多，因此要尽可能地将问题建模为分类问题求解。

### 其它损失函数
除了分类和回归，还有多种属性的分类问题，每个样本可能同时拥有多种属性。这时需要对每一种属性训练一个2类的svm loss。
另外，还有所谓的结构化预测问题(Structured prediction)，这里labels可以是图或者树的结构。
以后有时间再学习这方面的内容吧。

## 正则化损失
前面介绍的都是data loss，在实际的应用中，通常会再加上正则化损失：
$$\lambda R(W)$$

这里的函数R(W)可以是L2范数（和0向量的L2范数距离），称为L2 regularization
$$\frac{1}{2}\lambda W^2$$
这里加上的系数是为了求导方便。

也可以是L1范数，称为L1 regularization：
$$\lambda|W|$$

## 总结
目前用的最多的损失函数是带正则化的SVM loss和Cross-Entropy loss，实际上这两种loss是很类似的。前者的loss有一个max的截断，后者更加连续。可以说在不同的场景下有各自的优缺点，遇到实际问题可以都试一试比较一下。