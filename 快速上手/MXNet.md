# 名词解释

## NDArray

```python
from mxnet import nd  # nd=ndarray
```

## 创建行向量
```python
x = nd.arange(12)
x
# [ 0.  1.  2.  3.  4.  5.  6.  7.  8.  9. 10. 11.]
```

## 获取实例形状
```python
x.shape
# (12,)
```

## 元素总数
```python
x.size
# 12
```

## 更改形状

一维数组改为（3，4）

```python
X = x.reshape((3, 4))
# 也可以：X = x.reshape((-1, 4))
# 或 X = x.reshape((3, -1))
```

输出：

```python
[[ 0.  1.  2.  3.]
 [ 4.  5.  6.  7.]
 [ 8.  9. 10. 11.]]
```

## 创建张量

```python
#各元素为0
nd.zeros((2, 3, 4))
#各元素为1
nd.ones((3, 4))
```

输出：

```python
[[[0. 0. 0. 0.]
 [0. 0. 0. 0.]
 [0. 0. 0. 0.]]

[[0. 0. 0. 0.]
 [0. 0. 0. 0.]
 [0. 0. 0. 0.]]]
————————————————————
[[1. 1. 1. 1.]
 [1. 1. 1. 1.]
 [1. 1. 1. 1.]]
```

## 通过列表指定元素值

```python
Y = nd.array([[2, 1, 4, 3], [1, 2, 3, 4], [4, 3, 2, 1]])
```

输出：

```python
[[2. 1. 4. 3.]
 [1. 2. 3. 4.]
 [4. 3. 2. 1.]]
```

## 随机填充

均值为0、标准差为1的正态分布

```python
nd.random.normal(0, 1, shape=(3, 4))
```

输出：

```python
[[ 2.2122064   0.7740038   1.0434405   1.1839255 ]
 [ 1.8917114  -1.2347414  -1.771029   -0.45138445]
 [ 0.57938355 -1.856082   -1.9768796  -0.20801921]]
```

## 运算

```python
# X = [[ 0.  1.  2.  3.]
#      [ 4.  5.  6.  7.]
#      [ 8.  9. 10. 11.]]
#
# Y = [[ 2.  1.  4.  3.]
#      [ 1.  2.  3.  4.]
#      [ 4.  3.  2.  1.]]
```

### 加

```python
X + Y
# [[ 2.  2.  6.  6.]
#  [ 5.  7.  9. 11.]
#  [12. 12. 12. 12.]]
```

### 乘

```python
X * Y
# [[ 0.  1.  8.  9.]
#  [ 4. 10. 18. 28.]
#  [32. 27. 20. 11.]]
```

### 除

```python
X / Y
# [[ 0.    1.    0.5   1.  ]
#  [ 4.    2.5   2.    1.75]
#  [ 2.    3.    5.   11.  ]]
```

### 指数运算

```python
Y.exp()
# [[ 7.389056   2.7182817 54.59815   20.085537 ]
#  [ 2.7182817  7.389056  20.085537  54.59815  ]
#  [54.59815   20.085537   7.389056   2.7182817]]
```

### 转置矩阵

```python
Y.T
# Y =   [[2. 1. 4. 3.]
#        [1. 2. 3. 4.]
#        [4. 3. 2. 1.]]
#
# Y.T = [[2. 1. 4.]
#        [1. 2. 3.]
#        [4. 3. 2.]
#        [3. 4. 1.]]
```

### 矩阵乘法

X与Y的转置做矩阵乘法

```python
nd.dot(X, Y.T)
# [[ 18.  20.  10.]
#  [ 58.  60.  50.]
#  [ 98. 100.  90.]]
```

### 连结（concatenate）

```python
#行连结，即：
# X
# Y
nd.concat(X, Y, dim=0)
# [[ 0.  1.  2.  3.]
#  [ 4.  5.  6.  7.]
#  [ 8.  9. 10. 11.]
#  [ 2.  1.  4.  3.]
#  [ 1.  2.  3.  4.]
#  [ 4.  3.  2.  1.]]

#列连结，即：X Y
nd.concat(X, Y, dim=1)
# [[ 0.  1.  2.  3.  2.  1.  4.  3.]
#  [ 4.  5.  6.  7.  1.  2.  3.  4.]
#  [ 8.  9. 10. 11.  4.  3.  2.  1.]]
```

### 大小、相等判断

```python
#对应位置相等赋值1，否则0
X == Y
# [[0. 1. 0. 1.]
#  [0. 0. 0. 0.]
#  [0. 0. 0. 0.]]
#
# 还可以比较 X >= Y, X != Y 等
```

### 求和

```python
X.sum()
# [66.]
```

### 转换标量 asscalar

```python
X.norm().asscalar()
# 22.494442
```

## 广播(复制)机制

```python
A = nd.arange(3).reshape((3, 1))
B = nd.arange(2).reshape((1, 2))
A, B
A + B
# [[0.]
#  [1.]     +  [[0. 1.]]     ==>
#  [2.]]
# 
#
# [[0. 0.]     [[0. 1.]       [[0. 1.]
#  [1. 1.]   +  [0. 1.]   =    [1. 2.]
#  [2. 2.]]     [0. 1.]]       [2. 3.]]

```

## 索引

```python
X[1:3]
# [[ 4.  5.  6.  7.]
#  [ 8.  9. 10. 11.]]
X[1, 2] = 9
# [[ 0.  1.  2.  3.]
#  [ 4.  5.  9.  7.]
#  [ 8.  9. 10. 11.]]
X[0:2, :] = 10
# [[10. 10. 10. 10.]
#  [10. 10. 10. 10.]
#  [12. 12. 12. 12.]]
```

## 复制结构并赋值0

```python
Z = Y.zeros_like()
```

## 减少内存开销

```python
#会生成临时结果再赋值
X = X + Y
#下面操作不会生成临时结果
X[:] = X + Y
X += Y
```

## 和NumPy转换

```python
# NumPy to NDArray
import numpy as np

P = np.ones((2, 3))
D = nd.array(P)
D

# NDArray to NumPy
D.asnumpy()

```

## 查找模块里的函数和类

MXNet网站 http://mxnet.apache.org/

```python
from mxnet import nd

print(dir(nd.random))
# ['NDArray', '_Null', '__all__', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', '_internal', '_random_helper', 'current_context', 'exponential', 'exponential_like', 'gamma', 'gamma_like', 'generalized_negative_binomial', 'generalized_negative_binomial_like', 'multinomial', 'negative_binomial', 'negative_binomial_like', 'normal', 'normal_like', 'numeric_types', 'poisson', 'poisson_like', 'randint', 'randn', 'shuffle', 'uniform', 'uniform_like']
# 均匀分布采样（uniform）
# 正态分布采样（normal）
# 泊松分布采样（poisson）

#函数的详细帮助
help(nd.ones_like)

#Jupyter记事本里
nd.random.uniform?   #独立窗口显示帮助内容
nd.random.uniform??  #独立窗口显示函数代码
```

# 自动求梯度

对函数 y = 2x⊤x 求关于列向量x的梯度

```python
from mxnet import autograd, nd

x = nd.arange(4).reshape((4, 1))
x.attach_grad()  # 申请存储梯度所需要的内存
#用record记录求梯度相关计算
with autograd.record():
	y = 2 * nd.dot(x.T, x)
#由于x形状为(4,1)，y是一个标量。调用backward自动求梯度
#如果y不是标量，会先对y中元素求和再求梯度
y.backward()
#梯度为4x，验证：
assert (x.grad - 4 * x).norm().asscalar() == 0
x.grad
# [[ 0.]
 [ 4.]
 [ 8.]
 [12.]]
```

调用 record 之后，autograd 会把运行模式从预测模式转为训练模式，可通过 is_training 判断

```python
print(autograd.is_training())
# False
with autograd.record():
    print(autograd.is_training())
# True
```

# 线性回归两种实现

## 从零实现

```python
%matplotlib inline
from IPython import display
from matplotlib import pyplot as plt
from mxnet import autograd, nd
import random

# 生成实验用数据集
# 2个特征，1000个样本，真实权重w = [2, -3.4]^T，偏差b = 4.2， 随机噪声服从均值为0，标准差0.01正态分布。
num_inputs = 2
num_examples = 1000
true_w = [2, -3.4]
true_b = 4.2
features = nd.random.normal(scale=1, shape=(num_examples, num_inputs))
labels = true_w[0] * features[:, 0] + true_w[1] * features[:, 1] + true_b
labels += nd.random.normal(scale=0.01, shape=labels.shape)


#观察 第二个特征features[:, 1] 和 标签 labels的散点图
def use_svg_display():
    # 用矢量图显示
    display.set_matplotlib_formats('svg')

def set_figsize(figsize=(3.5, 2.5)):
    use_svg_display()
    # 设置图的尺寸
    plt.rcParams['figure.figsize'] = figsize

set_figsize()
plt.scatter(features[:, 1].asnumpy(), labels.asnumpy(), 1);  # 加分号只显示图
```

## 简洁实现