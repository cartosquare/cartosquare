---
title: 机器如何学习？
date: 2017-11-26 20:13:39
tags:
  - 深度学习
  - 机器学习
  - 随机梯度下降
  - 求导
  - 链式法则
  - 自动微分
---

机器学习的目标是获得一组最优的参数，这些参数决定了评分函数，因此我们要找一个最优的评分函数。实际上通过引入损失函数，我们可以通过最小化损失函数来不断更新评分函数的参数，具体来说，机器学习可以通过一种迭代的技术，使得每一步的迭代之后，损失函数的输出都能减小，因此在有限步迭代之后，损失函数就能达到最小值点（当然这是一个局部最小值）。在每一次迭代之后，我们都会根据损失函数的输出更新评分函数的参数，这样就达到了学习的目的。

<!-- more -->
## 梯度下降(Gradient Decent)
梯度下降就是这样一种迭代技术。基于函数f的梯度负方向$-\Delta f$是这个函数下降最快的方向，Gradient Decent通过从某个点$w_0$出发，不断沿着梯度负方向前进一个小步长$\eta$来达到求函数f的局部最优解的目的：
$$w^*=argmin_wf(w) \qquad (1)$$

假设经过I步达到最优点$w^*$，那么寻找最优解的过程如下：

$$
w\_1 = w\_0 - \eta \Delta f(w\_0) \\\\
w\_2 = w\_1 - \eta \Delta f(w\_1) \\\\
\cdot\cdot\cdot \\\\
w^* = w\_{I-1} - \eta \Delta f(w\_{I-1})  \qquad (2)
$$

这里步长$\eta$的设置一般是动态的，如果设定为固定值，从点$w_0$到最优点的轨迹有点像Z字形，收敛的时间会很长，所以一般都会动态地减小步长值。

### 导数
为了进行迭代，我们必须要在每一步里计算当前点的梯度，也就是导数：
$$\frac{d}{dx}f = \lim\_{x\to x\_0}\frac {f(x) - f(x\_0)}{x - x\_0} \qquad (3)$$

考虑到实际上机器学习的函数十分复杂，首先它是评分函数和损失函数组成的复合函数：
$$F = Loss(Score(x)) \qquad (4)$$
其次，评分函数有可能十分复杂，比如，在深度学习里，评分函数是一系列网络层的复合，假设有N个网络层，每个网络的函数是$K_{i}$，那么评分函数就是这N个网络层的复合函数：

$$Score(x)=K\_{n}(K\_{n-1}(...K\_1(x))) \qquad (5)$$

#### 链式法则
因此一般的机器学习框架都会采用一种后向自动微分的方式求导数。这种方式不仅不需要使用微积分的分析方式求出导数的表达式，运算效率还特别高。其中的原理是运用了复合函数求导的链式法则：
$$\frac {d}{dx}f(y(x)) = \frac {df}{dy} \frac{dy}{dx} \qquad (6)$$

比如，要计算
$$y = sin(x^2)$$
在$x=1$处的导数，首先我们分成几步计算y的值：
$$
\\begin{align}
& x\_0 = 1 \\\\
& x\_1 = x\_0^2 = 1 \\\\
& x\_2 = sin(x\_1) = 0.8415 \\\\
& y = x\_2 = 0.8415
\\end{align}
$$
计算导数时，只需反过来对每一步求导即可：

$$
\\begin{align}
& \bar{v\_2} = \bar y = 1 \\\\ 
& \bar{v\_1} = \bar{v\_2}\frac{dv\_2}{dv\_1} = \bar{v\_2} \times cos(x\_1) = 0.8415 \\\\
& \bar{v\_0} = \bar{v\_1}\frac{dv\_1}{dv\_0} = \bar{v\_1} \times 2x\_0 = 1.0806 
\\end{align}
$$

对于这个例子，我们可以很快验证，函数f的导数是：
$$
\frac{dy}{dx} = cos(x^2)\cdot 2x
$$
代入x=1，得：
$$
cos(1)\times 2 = 1.0806
$$

因此，自动微分的好处在于我们不必显示地用微积分的方法求解导数表达式。在caffe2中，我们只需要对每一层的函数编码导数的表达式，整个神经网络表示的大函数的梯度则通过自动微分的方式汇集每一层的导数值得来。

下面来看看之前介绍的SVM loss和Softmax loss函数的梯度如何计算：

#### SVM Loss的梯度
样本i的SVM loss用下式表示：
$$L\_i=\sum\_{j\neq{y\_i}} max(0, s\_j - s\_{y\_i} + \Delta) \qquad (7)$$

由于caffe2框架采用自动微分的方式，所以对于样本i，我们只需要求导到$s_j$即可(即评分函数的输出），不需要一直往前求导，即可以认为$s\_j$就是函数的输入变量。显然当$i \neq j$时有：
$$
\frac{d{L\_i}}{d{s\_j}} = 
\\begin{cases}
0, & s\_j - s\_{y\_i} + \Delta \lt 0 \\\\
1, & s\_j - s\_{y\_i} + \Delta \gt 0 \qquad (8)\\\\
\\end{cases}
$$
当$i == j$时有，
$$\frac{d{L\_i}}{d{s\_j}} = \sum\_{j\neq{y\_i}}I(0, s\_j - s\_{y\_i} + \Delta) \qquad (9)$$
其中，
$$
I(0, s\_j - s\_{y\_i} + \Delta)= 
\\begin{cases}
0, & s\_j - s\_{y\_i} + \Delta \lt 0 \\\\
-1, & s\_j - s\_{y\_i} + \Delta \gt 0  \qquad (10)\\\\
\\end{cases}
$$

在caffe2中可以实现如下：
```c++
template <>
bool SVMLossL1GradientOp<float, CPUContext>::RunOnDevice() {
  // (1)
  auto& X = Input(0);  // predict scores
  const float* X_data = X.template data<float>();
  int N, D;
  const auto canonical_axis = X.canonical_axis_index(axis_);
  N = X.size_to_dim(canonical_axis);  // batch size
  D = X.size_from_dim(canonical_axis);

  // (2)
  auto& Y = Input(1);  // ground truth labels
  const int* Y_data = Y.template data<int>();
  // check label dimension
  if (Y.ndim() == canonical_axis) {
    CAFFE_ENFORCE_EQ(Y.size(), N);
  } else {
    CAFFE_ENFORCE_EQ(Y.size_to_dim(canonical_axis), N);
    CAFFE_ENFORCE_EQ(Y.size_from_dim(canonical_axis), 1);
  }

  // (3)
  auto& d_avg_loss = Input(2);  // avg_loss grad(gradient from top layer)
  const float* d_avg_loss_data = d_avg_loss.template data<float>();
  CAFFE_ENFORCE(d_avg_loss.ndim() == 1);
  CAFFE_ENFORCE(d_avg_loss.dim32(0) == N);

  //(4)
  auto* dX = Output(0);  // gradient r.s.t predict scores
  dX->ResizeLike(X);
  float* dX_data = dX->template mutable_data<float>();

  // (5)
  // calculate gradient for each sample
  for (int i = 0; i < N; ++i) {
    // for the class(i) that is not target class(j), the gradient is:
    // I(max(0, p_i - p_j + \Delta))
    // for class j, the gradient is:
    // \sum_i I(max(0, p_i - p_j + \Delta))
    int cnt = 0;
    float target_score = X_data[i * D + Y_data[i]];
    for (int j = 0; j < D; ++j) {
      if (j != Y_data[i]) {
        float loss = X_data[i * D + j] - target_score + 1;
        if (loss > 0) {
          dX_data[i * D + j] = 1 * d_avg_loss_data[i];
          ++cnt;
        } else {
          dX_data[i * D + j] = 0;
        }
      }
    }
    dX_data[i * D + Y_data[i]] = -1 * cnt * d_avg_loss_data[i];
  }

  return true;
}
```
这里有几点说明(下面的序号对于上面代码中注释的序号）：
* （1）第一个输入X和SVMLoss的输入是一样的，是神经网络上一层的输出，在这里主要是为了获取样本数量（batch size）N以及feature维度D
* （2） 第二个输入Y也和SVMLoss的第二个输入一样，是样本真实的标签
* （3）第三个输入d\_avg\_loss是下一层的导数传入，一半情况下SVMLoss下一层是一个AveragedLoss层，所以这个值是该层的导数输出
* （4）dX就是我们要求的导数，它的维度和上一层的输入X的维度是一样的
* （5）实际计算导数的过程，注意到是对每一个输入都进行了导数的计算

#### SoftMax Loss的梯度
SoftMax函数的形式如下：
$$S = \frac{e^{y\_i}}{\sum\_j{e^{y\_j}}} \qquad (11)$$
根据导数的除法法则，有：
$$
\\begin{align}
\frac{dS}{d{y\_i}} & =  \frac{e^{y\_i} \cdot \sum\_j{e^{y\_j}} - e^{y\_i} \cdot e^{y\_i}}{(\sum\_j{e^{y\_j}})^2} \\\\
& = \frac{e^{y\_i}}{\sum\_j{e^{y\_j}}} \cdot \frac{\sum\_j{e^{y\_j}} - e^{y\_i}}{\sum\_j{e^{y\_j}}} \\\\
& =  \frac{e^{y\_i}}{\sum\_j{e^{y\_j}}} \cdot (1 - \frac{e^{y\_i}}{\sum\_j{e^{y\_j}}}) \\\\
& = S \cdot (1 - S) \qquad (12)
\\end{align}
$$
可以看出求SoftMax在点x处的导数十分简单，只要知道在x处的函数值就行了。

下面是caffe2里关于SoftMax导数的实现：
```c++
template <>
bool SoftmaxGradientOp<float, CPUContext>::RunOnDevice() {
  // (1)
  auto& Y = Input(0);
  // (2)
  auto& dY = Input(1);
  // (3)
  auto* dX = Output(0);
  const auto canonical_axis = Y.canonical_axis_index(axis_);
  const int N = Y.size_to_dim(canonical_axis);
  const int D = Y.size_from_dim(canonical_axis);
  // First, get scales
  if (scale_.size() != N) {
    scale_.Resize(N);
  }
  if (sum_multiplier_.size() != D) {
    sum_multiplier_.Resize(D);
    math::Set<float, CPUContext>(D, 1.f, sum_multiplier_.mutable_data<float>(),
                                 &context_);
  }
  dX->ResizeLike(Y);
  const float* Ydata = Y.data<float>();
  const float* dYdata = dY.data<float>();
  float* dXdata = dX->mutable_data<float>();
  context_.Copy<float, CPUContext, CPUContext>(Y.size(), dYdata, dXdata);
  float* scaledata = scale_.mutable_data<float>();
  // (4)
  for (int i = 0; i < N; ++i) {
    math::Dot<float, CPUContext>(D, Ydata + i * D, dYdata + i * D,
                                 scaledata + i, &context_);
  }
  // (5)
  math::Gemm<float, CPUContext>(CblasNoTrans, CblasNoTrans, N, D, 1, -1,
                                scaledata, sum_multiplier_.data<float>(), 1,
                                dXdata, &context_);
  // (6)
  math::Mul<float, CPUContext>(Y.size(), dXdata, Ydata, dXdata,
                               &context_);
  return true;
}
```
几点说明：
* (1) 这个输入Y就是上面推导的S，即SoftMax函数的输出值，我们只需要这一个输入就可以计算导数dX
* (2) dY是下一层传进来的导数，这一层的导数算出来后要乘以dY
* (3) 需要计算的导数dX，维度和Y是一样的
* (4)(5)(6) 分别计算了$Y\cdot dY$, $dY - Y\cdot dY$以及($dY - Y\cdot dY)\cdot Y$，最后这个式子正是SoftMax的导数与上一层梯度的乘积

一般而言，Softmax的下一层是Cross-Entropy层：
$$L = -log(S\_j) \qquad (13)$$
其中j是样本i的真实类别。故其导数为：
$$\frac{dL}{dS} = -\frac{1}{S\_j} \qquad (14)$$
caffe2里的实现：
```c++
template <>
bool LabelCrossEntropyGradientOp<float, CPUContext>::RunOnDevice() {
  auto& X = Input(0);
  auto& label = Input(1);
  auto& dY = Input(2);
  auto* dX = Output(0);
  int N, D;
  if (X.ndim() > 1) {
    N = X.dim32(0);
    D = X.size_from_dim(1);
  } else {
    N = 1;
    D = X.dim32(0);
  }
  CAFFE_ENFORCE(
      (label.ndim() == 1) || (label.ndim() == 2 && label.dim32(1) == 1));
  CAFFE_ENFORCE_EQ(label.dim32(0), N);
  CAFFE_ENFORCE_EQ(dY.ndim(), 1);
  CAFFE_ENFORCE_EQ(dY.dim32(0), N);
  dX->ResizeLike(X);
  math::Set<float, CPUContext>(dX->size(), 0.f, dX->mutable_data<float>(),
                               &context_);
  const float* Xdata = X.data<float>();
  const float* dYdata = dY.data<float>();
  const int* labelData = label.data<int>();
  float* dXdata = dX->mutable_data<float>();
  for (int i = 0; i < N; ++i) {
    // (1)
    dXdata[i * D + labelData[i]] =
        - dYdata[i] / std::max(Xdata[i * D + labelData[i]], kLOG_THRESHOLD());
  }
  return true;
}
```
注意（1）处的实现即是计算式子-14，只不过是多乘以了下一层传递过来的导数。

### 梯度下降的变种
梯度下降算法中，可以以不同的频率更新权重，比如，可以所有的样本都计算一遍以后更新权重，所有样本训练一遍称为一个场景（epoch）；也可以分批次计算，在每批次结束之后再进行权重更新，这种方式称*Mini-batch Gradient Decent*；还可以每个样本计算一遍后马上进行权重更新，这种方式称为*Stochastic Gradient Decent(随机梯度下降)*。这三种方式，权重更新的频率越来越快，权重更新得快能够加快收敛的速度，但是也不是越快越好，因为有可能会跳离极小点两边摆动，因为权重更新快也表示了参与计算梯度的样本量减少，导致了算法的不稳定。
所以在通常的神经网络中，都会采用*Mini-batch Gradient Decent*这种方式。这里的mini-batch一般设置为2的指数，比如32，62，128，256等。