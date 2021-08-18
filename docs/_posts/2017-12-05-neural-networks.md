---
title: 神经网络
date: 2017-12-05 21:11:39
tags:
  - 机器学习
  - 全连接层
  - 激活函数
  - 正规化
---
神经网络通常由输入层（数据层）、全连接层、激活层、正规化层、以及损失层组成。其中的全连接层，激活层和正规化层又称为隐藏层。下面分别介绍。

<!-- more -->

## 全连接层（Full Connected Layer）

### 输出计算
设$M \times K$维矩阵$X$是上一层的输出，其中K是每一个样本的特征数, M 为 batch size；$M \times N$维向量Y是全连接层的输出, 其中N是全连接层的神经元个数，那么全连接层的参数$W$是一个$N \times K$维的矩阵, 基b是一个N维的矩阵，并且有：
$$
Y = XW^T + b \qquad (1)
$$
注意可能和其他地方的形式不太一样，其他地方可能写为:
$$ Y = WX + b$$
只要对两边进行转置，并替换Y、b、X的行列顺序即和上面是等价的，式（1）这种方式实际上是caffe2进行计算的方式：
```c++
template <
      typename T_X,
      typename T_W,
      typename T_B,
      typename T_Y,
      typename MATH>
  bool DoRunWithType() {
    const auto& X = Input(0);
    const auto& W = Input(1);
    const auto& b = Input(2);
    auto* Y = Output(0);
    CAFFE_ENFORCE(b.ndim() == 1, b.ndim());
    // batch size
    const auto canonical_axis = X.canonical_axis_index(axis_);
    const auto M = X.size_to_dim(canonical_axis);
    const auto K = X.size_from_dim(canonical_axis);
    const auto canonical_axis_w = W.canonical_axis_index(axis_w_);
    const int N = TransposeWeight ? W.size_to_dim(canonical_axis_w)
                                  : W.size_from_dim(canonical_axis_w);

    // Error checking
    CAFFE_ENFORCE(M == X.size() / K, dimErrorString());
    CAFFE_ENFORCE(K == W.size() / N, dimErrorString());
    CAFFE_ENFORCE(N == b.dim32(0), dimErrorString());
    CAFFE_ENFORCE(N == b.size(), dimErrorString());

    Y_shape_cache_ = X.dims();
    // This is an invariant of canonical_axis, so we can DCHECK.
    DCHECK_LE(canonical_axis + 1, Y_shape_cache_.size());
    Y_shape_cache_.resize(canonical_axis + 1);
    Y_shape_cache_[canonical_axis] = N;
    Y->Resize(Y_shape_cache_);
    CAFFE_ENFORCE(M * N == Y->size(), dimErrorString());

    if (X.size() == 0) {
      // skip the rest of the computation if X is empty
      Y->template mutable_data<T_Y>();
      return true;
    }

    // default to FLOAT as math.h does.
    TensorProto::DataType math_type = TensorProto_DataType_FLOAT;
    if (fp16_type<MATH>()) {
      math_type = TensorProto_DataType_FLOAT16;
    }

    // 计算XW^T
    math::Gemm<T_X, Context, Engine>(
        CblasNoTrans,
        TransposeWeight ? CblasTrans : CblasNoTrans,
        M,
        N,
        K,
        1,
        X.template data<T_X>(),
        W.template data<T_W>(),
        0,
        Y->template mutable_data<T_Y>(),
        &context_,
        math_type);
    // 加上基向量
    if (bias_multiplier_.size() != M) {
      // If the helper bias multiplier is not M, reshape and fill it with one.
      bias_multiplier_.Resize(M);
      math::Set<T_B, Context>(
          M,
          convert::To<float, T_B>(1),
          bias_multiplier_.template mutable_data<T_B>(),
          &context_);
    }
    math::Gemm<T_B, Context, Engine>(
        CblasNoTrans,
        CblasNoTrans,
        M,
        N,
        1,
        1,
        bias_multiplier_.template data<T_B>(),
        b.template data<T_B>(),
        1,
        Y->template mutable_data<T_Y>(),
        &context_,
        math_type);
    return true;
  }
```
这里用了下面这个函数：
```c++
template <>
void Gemm<float, CPUContext>(
    const CBLAS_TRANSPOSE TransA,
    const CBLAS_TRANSPOSE TransB,
    const int M,
    const int N,
    const int K,
    const float alpha,
    const float* A,
    const float* B,
    const float beta,
    float* C,
    CPUContext* context,
    TensorProto::DataType math_type);
```

这个函数实现了下面这个操作：
$$C = alpha \cdot A \times B + beta \cdot C$$
这里A是$M \times K$矩阵，B是$K \times N$矩阵，C是$M \times N$矩阵。alpha, beta 都是标量。实际上，把 alpha 设为1，beta 设为0就是两个矩阵相乘。 

### 导数计算
为了求dW，我们可以设式（1）中的W的维数为$K * N$，那么有：
$$Y = XW + b \qquad (2)$$
其实，转置只是把元素的顺序改变了，所以有$dX=(d(X^T))^T$
对于式（2），有：
$$
\frac{dY}{dW} = X \\\\
\frac{dY}{dX} = W \\\\
\frac{dY}{db} = 1 \\\\
$$
caffe2 中的代码如下：
```c++
 template <
      typename T_X,
      typename T_W,
      typename T_DY,
      typename T_B,
      typename T_DX,
      typename T_DW,
      typename T_DB,
      typename MATH>
  bool DoRunWithType() {
    const auto& X = Input(0);
    const auto& W = Input(1);
    const auto& dY = Input(2);
    // batch size
    const auto canonical_axis = X.canonical_axis_index(axis_);
    const int M = X.size_to_dim(canonical_axis);
    const int K = X.size_from_dim(canonical_axis);
    const auto canonical_axis_w = W.canonical_axis_index(axis_w_);
    const int N = W.size_to_dim(canonical_axis_w);
    CAFFE_ENFORCE(M * K == X.size());
    CAFFE_ENFORCE(K * N == W.size());

    auto* dW = Output(0);
    auto* db = Output(1);
    dW->ResizeLike(W);
    db->Resize(N);

    if (X.size() == 0) {
      // generate a zero blob for db and dW when X is empty
      // skipped
      //...
      return true;
    }

    // default to FLOAT as math.h does.
    TensorProto::DataType math_type = TensorProto_DataType_FLOAT;
    if (fp16_type<MATH>()) {
      math_type = TensorProto_DataType_FLOAT16;
    }

    // Compute dW
    math::Gemm<T_DY, Context, Engine>(
        CblasTrans,
        CblasNoTrans,
        N,
        K,
        M,
        1,
        dY.template data<T_DY>(),
        X.template data<T_X>(),
        0,
        dW->template mutable_data<T_DW>(),
        &context_,
        math_type);
    if (bias_multiplier_.size() != M) {
      // If the helper bias multiplier is not M, reshape and fill it
      // with one.
      bias_multiplier_.Resize(M);
      math::Set<T_B, Context>(
          M,
          convert::To<float, T_B>(1),
          bias_multiplier_.template mutable_data<T_B>(),
          &context_);
    }
    // Compute dB
    math::Gemv<T_DY, Context>(
        CblasTrans,
        M,
        N,
        1,
        dY.template data<T_DY>(),
        bias_multiplier_.template data<T_B>(),
        0,
        db->template mutable_data<T_DB>(),
        &context_);

    // Compute dX
    if (OutputSize() == 3) {
      auto* dX = Output(2);
      dX->ResizeLike(X);
      math::Gemm<T_DX, Context, Engine>(
          CblasNoTrans,
          CblasNoTrans,
          M,
          K,
          N,
          1,
          dY.template data<T_DY>(),
          W.template data<T_W>(),
          0,
          dX->template mutable_data<T_DX>(),
          &context_,
          math_type);
    }
    return true;
  }
```
上面的dY是后向自动微分时上一层传进来的导数，需要与这一层的导数相乘再传到下一层。
这里还用到了下面这个函数：
```c++
template <>
void Gemv<float, CPUContext>(
    const CBLAS_TRANSPOSE TransA,
    const int M,
    const int N,
    const float alpha,
    const float* A,
    const float* x,
    const float beta,
    float* y,
    CPUContext* context,
    TensorProto::DataType math_type);
```
它实现的是下面的操作：
$$
y = alpha \cdot Ax + beta \cdot y
$$
其中A是$M \times N$矩阵，x是N维向量，y是M维向量。alpha 和 beta 都是标量，实际上设置 alpha 为1，beta为0，就是矩阵和向量相乘。

## 激活函数

常见的激活函数有Sigmoid、Tanh、ReLU以及Leaky ReLU函数。

### Sigmoid
Sigmoid 函数具有下面的数学形式：
$$\sigma(x) = \frac{1}{1 + e^{-x}} \qquad (3)$$
它可以把实数输入映射到0和1之间。在神经网络早期，Sigmoid函数曾被广泛应用，但是有如下的缺点：
* 梯度杀手：当Sigmoid的输出位于长尾部分时，梯度几乎为0，无法进行有效学习。
* 非0中心：Sigmoid的输出值不是以0位中心的（都大于0），这也导致了梯度计算的不稳定，学习不容易收敛。
caffe2实现：
```c++
struct SigmoidCPUFunctor {
  template <typename T>
  inline void
  operator()(const int n, const T* x, T* y, CPUContext* /*device_context*/) {
    ConstEigenVectorArrayMap<T> xM(x, n);
    EigenVectorArrayMap<T>(y, n) = 1. / (1. + (-xM).exp());
  }
};
```
这里的输入x是一个n维向量。
根据导出的除法法则，可以推导出Sigmoid函数的导数表达式：
$$
\\begin{align}
\frac{d\sigma}{dx} & =  \frac{0 \cdot (1 + e^{-x}) - (-1) \cdot e^{-x} \cdot 1}{(1 + e^{-x})^2} \\\\
& = \frac{1}{1 + e^{-x}} \cdot \frac{e^{y\_i}}{1 + e^{-x}} \\\\
& =  \frac{1}{1 + e^{-x}} \cdot (1 - \frac{1}{1 + e^{-x}}) \\\\
& = \\sigma (x)  \cdot (1 - \sigma (x) ) \qquad (4)
\\end{align}
$$
caffe2中的实现：
```c++
struct SigmoidGradientCPUFunctor {
  template <typename T>
  inline void Run(
      const int n,
      const T* y,
      const T* dy,
      T* dx,
      CPUContext* /*device_context*/) {
    ConstEigenVectorArrayMap<T> yM(y, n), dyM(dy, n);
    EigenVectorArrayMap<T>(dx, n) = dyM * yM * (1. - yM);
  }
};
```

### Tanh 
Tanh函数对Sigmoid函数进行了放缩平移变换：
$$
\\begin{align}
tanh(x) & = 2\sigma (2x) - 1 \\\\
& = \frac{2}{1 + e^{-2x}} - 1 \\\\
& =  \frac{1 - e^{-2x}}{1 + e^{-2x}} \\\\
& =  \frac{e^x - e^{-x}}{e^x + e^{-x}} \qquad (4)
\\end{align}
$$
因此它的值域在-1到1之间。由此可见，Tanh比Sigmoid要更好，因此在实际使用时，能用Tanh就不应该用Sigmoid。
caffe2中的实现：
```c++
struct TanhCPUFunctor {
  template <typename T>
  inline void
  operator()(const int n, const T* x, T* y, CPUContext* /*device_context*/) {
#ifdef CAFFE2_USE_ACCELERATE
    vvtanhf(y, x, &n);
#else
    ConstEigenVectorArrayMap<T> x_arr(x, n);
    EigenVectorMap<T>(y, n) = 1 - 2 * ((x_arr * 2).exp() + 1).inverse();
#endif
  }
};
```
Tanh的导数为：
$$
\\begin{align}
\frac{dtanh}{dx} & = \frac{(e^x + e^{-x}) \cdot (e^x + e^{-x}) - (e^x - e^{-x}) \cdot (e^x - e^{-x})}{(e^x + e^{-x})^2} \\\\
& = \frac{(e^x + e^{-x})^2 - (e^x - e^{-x})^2}{(e^x + e^{-x})^2} \\\\
& = 1 - (\frac{e^x - e^{-x}}{e^x + e^{-x}})^2 \\\\
& = 1 - (tanh(x))^2
\\end{align}
$$
caffe2中的实现：
```c++
struct TanhGradientCPUFunctor {
  template <typename T>
  inline void Run(
      const int n,
      const T* y,
      const T* dy,
      T* dx,
      CPUContext* /*device_context*/) {
    ConstEigenVectorArrayMap<T> dy_arr(dy, n);
    ConstEigenVectorArrayMap<T> y_arr(y, n);
    EigenVectorMap<T>(dx, n) = dy_arr * (1 - y_arr * y_arr);
  }
};
```

### ReLU
ReLU 函数的形式如下：
$$f(x) = max(0, x) \qquad (5)$$
ReLU 函数的好处是：
* 通非线性的Tanh和Sigmoid相比，线性的ReLU函数能够极大地加速梯度下降算法的收敛
* 计算十分简单
坏处是：
* 如果learning rate没有设置合理，会有很多神经元不被激活

ReLU函数的导数是：
$$
\frac{df}{dn} =
\\begin{cases}
0,  & if\, x \lt 0 \\\\
1, & if\, x \ge 0
\\end{cases}
$$

实现代码：
```c++
template <>
bool ReluOp<float, CPUContext>::RunOnDevice() {
  auto& X = Input(0);
  auto* Y = Output(0);
  Y->ResizeLike(X);

#ifdef CAFFE2_USE_ACCELERATE
  const float zero = 0.0f;
  vDSP_vthres(X.data<float>(), 1, &zero, Y->mutable_data<float>(), 1, X.size());
#else
  EigenVectorMap<float>(Y->mutable_data<float>(), X.size()) =
      ConstEigenVectorMap<float>(X.data<float>(), X.size()).cwiseMax(0.f);
#endif
  /* Naive implementation
  const float* Xdata = X.data<float>();
  float* Ydata = Y->mutable_data<float>();
  for (int i = 0; i < X.size(); ++i) {
    Ydata[i] = std::max(Xdata[i], 0.f);
  }
  */
  return true;
}

template <>
bool ReluGradientOp<float, CPUContext>::RunOnDevice() {
  auto& Y = Input(0);
  auto& dY = Input(1);
  auto* dX = Output(0);
  CAFFE_ENFORCE_EQ(dY.size(), Y.size());
  dX->ResizeLike(Y);

  const float* Ydata = Y.data<float>();
  const float* dYdata = dY.data<float>();
  float* dXdata = dX->mutable_data<float>();
  // TODO: proper vectorization with Eigen
  EigenVectorArrayMap<float> dXvec(dXdata, dX->size());
  ConstEigenVectorArrayMap<float> Yvec(Ydata, Y.size());
  ConstEigenVectorArrayMap<float> dYvec(dYdata, dY.size());
  dXvec = dYvec * Yvec.cwiseSign();
  /* Previous implementation
  for (int i = 0; i < Y.size(); ++i) {
    dXdata[i] = Ydata[i] > 0 ? dYdata[i] : 0;
  }
  */
  return true;
}
```

### Leaky ReLU
Leaky ReLU 修复了ReLU会杀死神经元的缺点：当输入小于0时，不直接置0，而是乘以一个很小的数$\alpha$(比如为0.01)：
$$
f(n) =
\\begin{cases}
\\alpha x,  & if\, x \lt 0 \\\\
x, & if\, x \ge 0
\\end{cases}
$$
这里的$\alpha$也可以作为每个神经元的一个参数进行优化，就像PReLU里那样。
Leaky ReLU函数的导数是：
$$
\frac{df}{dn} =
\\begin{cases}
\\alpha,  & if\, x \lt 0 \\\\
1, & if\, x \ge 0
\\end{cases}
$$
实现代码：
```c++
template <>
bool LeakyReluOp<float, CPUContext>::RunOnDevice() {
  const auto& X = Input(0);
  auto* Y = Output(0);
  Y->ResizeLike(X);
  ConstEigenVectorMap<float> Xvec(X.template data<float>(), X.size());
  EigenVectorMap<float> Yvec(Y->template mutable_data<float>(), Y->size());
  Yvec = Xvec.cwiseMax(0.f) + Xvec.cwiseMin(0.f) * alpha_;
  return true;
}

template <>
bool LeakyReluGradientOp<float, CPUContext>::RunOnDevice() {
  const auto& Y = Input(0);
  const auto& dY = Input(1);
  auto* dX = Output(0);
  dX->ResizeLike(Y);
  CAFFE_ENFORCE_EQ(Y.size(), dY.size());
  ConstEigenVectorMap<float> Yvec(Y.template data<float>(), Y.size());
  ConstEigenVectorMap<float> dYvec(dY.template data<float>(), dY.size());
  EigenVectorMap<float> dXvec(dX->template mutable_data<float>(), dX->size());
  Eigen::VectorXf gtZero = (Yvec.array() >= 0.0f).cast<float>();
  dXvec = dYvec.array() * gtZero.array() -
      dYvec.array() * (gtZero.array() - 1.0f) * alpha_;
  return true;
}
```
## 正则化层
