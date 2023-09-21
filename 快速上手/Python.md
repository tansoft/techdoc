## 2 vs 3
* 大多数生产应用程序都在使用 Python 2.7
* 在新的 Python 应用中使用 Python 3

## 最佳实践
> https://learnku.com/docs/python-guide/2018/translation-instructions/3247

## pipenv Python项目依赖管理工具

```
pip install pipenv
cd myproject
pipenv install requests
pipenv run python main.py
pipenv shell
```

## virtualenv 独立运行的虚拟环境
```
pip install virtualenv
cd myproject_path
virtualenv myproject
virtualenv -p /usr/bin/python2.7 myproject # 指定2.7环境
#开始使用环境
source myproject/bin/activate
pip install requests
deactivate
```

> 删除环境只需要删除对应文件夹

## 保持环境一致性
```
pip freeze > requirements.txt   # 保存当前安装环境
pip install -r requirements.txt  # 安装相关环境
```

## 代码结构
* 如果是单一文件,直接在根目录./sample.py
* 如果是模块包,在根目录./sample/__init__.py
* LICENSE
* setup.py
* requirements.txt
* ./docs/
* ./tests/
* Makefile
```
init:
    pip install -r requirements.txt

test:
    py.test tests

.PHONY: init test
```
* 特别在django程序模板执行中,应指定当前目录,以减少目录嵌套
```
django-admin.py startproject samplesite .
```

* import最佳实践
```
#最好
import modu
modu.sqrt(1)
#次之
from modu import sqrt
sqrt(1)
#最差
from modu import *
sqrt(1)
```

* 包系统
```
# pack/modu.py 使用(注意__init__.py会加载)
import pack.modu
#导入太深的包使用
import very.deep.module as mod
```

* 上下文管理器

如果封装的逻辑量很大，则类的方法可能会更好。 而对于处理简单操作的情况，函数方法可能会更好。

> CustomOpen 首先被实例化，然后调用它的 __enter__ 方法，而且 __enter__ 的返回值在 as f 语句中被赋给 f 。 当 with 块中的内容执行完后，会调用 __exit__ 方法。

```
class CustomOpen(object):
    def __init__(self, filename):
        self.file = open(filename)

    def __enter__(self):
        return self.file

    def __exit__(self, ctx_type, ctx_value, ctx_traceback):
        self.file.close()

with CustomOpen('file') as f:
    contents = f.read()
```

> custom_open 函数一直运行到 yield 语句。 然后它将控制权返回给 with 语句，然后在 as f 部分将 yield 的 f 赋值给 f。 finally 确保不论 with 中是否发生异常， close() 都会被调用。

```
from contextlib import contextmanager

@contextmanager
def custom_open(filename):
    f = open(filename)
    try:
        yield f
    finally:
        f.close()

with custom_open('file') as f:
    contents = f.read()
```

* 可变和不可变类型

从 0 到 19 创建一个连续的字符串（例如「012..1819」）

```
# 差
nums = ""
for n in range(20):
    nums += str(n)   # slow and inefficient
print nums

# 好
nums = []
for n in range(20):
    nums.append(str(n))
print "".join(nums)

# 较好
nums = [str(n) for n in range(20)]
print "".join(nums)

# 最好
nums = map(str, range(20))
print "".join(nums)
```

字符串连接有时使用 join() 并不总是最好的选择。比如当用预先确定数量的字符串创建一个新的字符串时，使用加法操作符确实更快，但在上文提到的情况 下或添加到已存在字符串的情况下，使用 join() 是更好的选择。

```
foo = 'foo'
bar = 'bar'
foobar = foo + bar  # This is good

foo += 'ooo'  # This is bad, instead you should do:

foo = ''.join([foo, 'ooo'])
```

除了 str.join() 和 +，您也可以使用 % 格式运算符来连接确定数量的字符串，但 PEP 3101 建议使用 str.format() 替代 % 操作符。

```
foo = 'foo'
bar = 'bar'

foobar = '%s%s' % (foo, bar) # It is OK
foobar = '{0}{1}'.format(foo, bar) # It is better
foobar = '{foo}{bar}'.format(foo=foo, bar=bar) # It is best
```

## Pandas/DataFrame & numpy

```python
import numpy as np
import pandas as pd
```

### 增删改查

#### 创建DataFrame

```python
#生成0-15顺序数，把一维数组转换为4*4数组
data = pd.DataFrame(np.arange(16).reshape(4,4), index=list('abcd'), columns=list('ABCD'))
    A   B   C   D
a   0   1   2   3
b   4   5   6   7
c   8   9  10  11
d  12  13  14  15

#指定列名构造新DF
data = pd.DataFrame(columns=['timestamp','aa','bb'])

#通过字典创建DF
dict = {'user':['zhangsan','lisi','wangwu'],'age':['20','24','45'], 'gender':['male', 'male', 'male']}
data = pd.DataFrame(dict)

# 创建随机df
data = pd.DataFrame({
   'A': pd.date_range(start='2016-01-01',periods=N,freq='D'),#freq设置步长，默认D表示日
   'x': np.linspace(0,stop=N-1,num=N),#数列，起始点，结尾点，元素个数
   'y': np.random.rand(N),
   'C': np.random.choice(['Low','Medium','High'],N).tolist(),
   'D': np.random.normal(100, 10, size=(N)).tolist()
})

# 增加行
frames = [df1, df2, df3]
df = pd.concat(frames)

# 可以先新建一个df，再append进去，append方法会被淘汰
new=pd.DataFrame({'time':time, 'aa':8, 'bb':6}, index=[len(data)]) 
data = data.append(new)

# 增加列
data['count'] = 1

```

#### 删除操作

```python
# 删除行，括号中为行索引
data = data.drop(0)

# 删除列，输入列名
data = data.drop('precip_total', axis=1)

# gc回收，循环处理大量数据时建议主动回收
import gc
del data
gc.collect()

```

#### 统计信息

```python
# 行数
len(data)

# 列数
print data.columns.size

# 行名
data.index

# 列名
data.columns

#设置显示行、列数，宽度
pd.set_option('max_columns',500)
pd.set_option('max_row',500)
pd.set_option('max_colwidth',100)

#打印DateFrame信息
print(df.info())
<class 'pandas.core.frame.DataFrame'>
Index: 3 entries, 2014-11-17 to 2014-11-19
Data columns (total 3 columns):
 #   Column   Non-Null Count  Dtype  
---  ------   --------------  -----  
 0   ts_code  3 non-null      object 
 1   open     3 non-null      float64
 2   high     3 non-null      float64
dtypes: float64(2), object(1)
memory usage: 96.0+ bytes
None

#打印列信息 mean：平均值 std：标准差
print(data['age'].describe())
count     8.000000
mean     20.500000
std       3.964125
min      11.000000
25%      21.000000
50%      21.000000
75%      23.000000
max      23.000000
Name: age, dtype: float64

# 分位数统计
# q = 0.5 中位数
# interpolation = "linear" "lower" "higher" "midpoint" "nearest"
df['Height'].quantile(q = [0.25, 0.5, 0.75])
0.25    4.25
0.50    6.00
0.75    7.75
Name: Height, dtype: float64

#列值出现次数统计
# ascending=True 进行升序排序，False 降序
# dropna=True 忽略nan值统计
# normalize=True 归一化，统计总和为1.0
# bins=6 把数据进行分档
print(data['name'].value_counts())
也可以用
print(data.groupby('name').size())
sravan    3
abc       4
Name: name, dtype: int64

print(data['age'].value_counts(bins=4))
(20.0, 23.0]      7
(10.987, 14.0]    1
(17.0, 20.0]      0
(14.0, 17.0]      0
Name: age, dtype: int64

#列某值出现次数
print(data['name'].value_counts()['sravan'])
3

# 跨所有列进行统计（结果很奇怪）
print(data.groupby('name').count())

# 所有列所有值汇总
data.apply(pd.value_counts)

```

#### 定位操作

```python

#取索引为a的行
data.loc['a']
#取第一行
data.iloc[0]
A   0
B   1
C   2
D   3

# 取前两行
data[0:2]

# 取第二行
data.ix[1]
data.iloc[1]

#取'A','B'列所有行
data.loc[:, ['A', 'B']]
#取1，2列所有行
data.iloc[:, [0, 1]]
    A   B
a   0   1
b   4   5
c   8   9
d  12  13

#多行，多列
data.loc[['a', 'b'], ['A', 'B']]
data.iloc[[0, 1], [0, 1]]

#提取A列为0行
data.loc[data['A'] == 0]
#多筛选条件
data.loc[(data['A'] == 0) & (data['C'] == 2)]
#其他写法
data[data['A'] == 0]
data[data['A'].isin([0])]
data[(data['A'] == 0) & (data['C'] == 2)]
data[(data['A'].isin([0])) & (data['C'].isin([2]))]
   A  B  C  D
a  0  1  2  3

# x列大于5所有行
data[data.x>5]

# 定位数据类型不一样的行
data[data['colA'].map(type) == datatime.datatime]

# 筛选出该列不为空的值
data[data['column'].notnull()]

# 找出有空值的列
data[data['column'].isnull()]
或
data.isnull().any()

# 找出有空值的行
data[data.isnull().T.any()]

```

#### 修改操作

##### 结构修改

```python
# 变换数据类型
data["RNC编号"].astype("Int64")

# 更换列顺序，直接设置新顺序即可
data = data[['age', 'gender', 'name']]

# 更换列名
data.columns = ['time', 'demand', longitude', 'latitude']
或
data.rename(columns={'Longitude':'longitude', 'Latitude':'latitude'}, inplace=True)

# 调整索引从1开始
data.index = range(1, len(data)+1)

# 列下移
# 表示将这一列整体向下移动一行, 索引0的位置补Null，-2则表示往上移两行
data['columnA'].shift(1)

# 判断并删除重复行
# 可以用来判断重复行，两行除索引外完全重复（即对所有列判断）
data.duplicated()

# 判断某列的重复行
data.duplicated(['Feeder'])
或
dup_row = data.duplicated(subset=["地市", "名称", "告警源", "最近发生时间"])
data["duplicated"] = dup_row
或
data[data["duplicated"]==True]

# 删除重复行
data = data.drop_duplicates()
data.drop_duplicates(['Feeder'])

# 替换列中A值到B
replace = dict{'A':'B'}
data['col1'] = data.col1.replace(replace)
```

##### 值修改

```python
# 填充空值为0.0，还可以填充 pd.NA 代替 np.nan，它代表空整数、空布尔、空字符
# 如果是现有空值，可以用<NA>做为字符串来找
data['col1'] = data['col1'].fillna(0.0)
or
data['col1'].fillna(value = 0.0, inplace=True)

# 更改某值
data.loc[8825,'wx_phrase'] = 'T-Storm'

# 将ratio列中不为零的行的angle字段对应值置为1
branch.loc[branch.ratio !=0, "angle"] = 1

# 列值运算
df['col2'] = df['col1'].map(lambda x: x**2)

# 多列运算
# axis=1 每行处理，axis=0 每列处理
df['col3'] = df.apply(lambda x: x['col1'] + 2 * x['col2'], axis=1)

# 差值
# 求每两行内各字段的差值
data.diff()
# 求第二列差值，补空填0
data.diff()[2].fillna(0)

# 平均值
# 对每一列的数据求平均值
data.mean()
# 求每一行平均值
data.mean(1)
# 最大/最小/求和
data.max() data.min() data.sum()
# 中位数
data.median()
# 极值对应索引
data.idxmax() data.idxmin()
# 平均绝对误差（mean absolute deviation），表示各个变量值之间差异程度
data.mad()，求，对的数值之一
# 方差
data.var()
# 标准差
data.std()
# 累加
data.cumsum()

# 判断列中是否含有exc列表内容
exc = ['6','10']
data[data['uid'].isin(exc)]
data[data.uid.isin(exc)]
# 判断不在没有isnotin，只能用apply，性能差
data.uid[data.apply(lambda x:x.uid not in exc, axis=1)]

# One-hot 编码，可多列同时
pd.get_dummies(df, columns=["diw", "dim"])

```

##### 高级操作

###### 采样

```python
DataFrame.sample(n=None, frac=None, replace=False, weights=None, random_state=None, axis=None)
# n是要抽取的行数
# frac是抽取的比例（0-1之间，frac=0.8，就是抽取其中80%）
# replace：是否为有放回抽样，replace=True时为有放回抽样。replace=False(默认为False)是无放回的采样，当采样数n大于样本数且没有设置replace=True时，会出现异常
# weights：指定样本抽中的概率，默认等概论抽样
# random_state：指定抽样的随机种子，可以使得每次抽样的种子一样，每次抽样结果一样
```

###### 遍历

```python
# itertuples 比 iterrows 快
# 使用 df.items() 返回每列对应数据
# 使用 df.iterrows() 返回每行对应数据
# 使用 df.itertuples() 返回每行对应的数据，namedtuples对象
# 使用 df.applymap() 处理每个元素

print(df)
print('iteritems---->>')
for idx,day in df.items():
    print(idx)
    print(day)
print('iterrows---->>')
for idx,day in df.iterrows():
    print(idx)
    print(day)
print('itertuples---->>')
for day in df.itertuples():
    print(day)

             ts_code      open      high
trade_date                              
2014-11-17  sh000001  2506.864  2508.767
2014-11-18  sh000001  2474.182  2477.052
2014-11-19  sh000001  2452.150  2461.491

iteritems---->>

ts_code
trade_date
2014-11-17    sh000001
2014-11-18    sh000001
2014-11-19    sh000001
Name: ts_code, dtype: object

open
trade_date
2014-11-17    2506.864
2014-11-18    2474.182
2014-11-19    2452.150
Name: open, dtype: float64

high
trade_date
2014-11-17    2508.767
2014-11-18    2477.052
2014-11-19    2461.491
Name: high, dtype: float64

iterrows---->>

2014-11-17
ts_code    sh000001
open       2506.864
high       2508.767
Name: 2014-11-17, dtype: object

2014-11-18
ts_code    sh000001
open       2474.182
high       2477.052
Name: 2014-11-18, dtype: object

2014-11-19
ts_code    sh000001
open        2452.15
high       2461.491
Name: 2014-11-19, dtype: object

itertuples---->>

Pandas(Index=datetime.date(2014, 11, 17), ts_code='sh000001', open=2506.864, high=2508.767)
Pandas(Index=datetime.date(2014, 11, 18), ts_code='sh000001', open=2474.182, high=2477.052)
Pandas(Index=datetime.date(2014, 11, 19), ts_code='sh000001', open=2452.15, high=2461.491)

# applymap
# 得出字符串中含有w的项
bool_array = df.applymap(lambda x:"w" in x)
# 只保存有w的项，其它项变为nan
out_array = df[bool_array]
print(out_array)

```

###### 分组聚合等

```python
#分组
df['col3'] = df.groupby('col1')['col2'].transform(lambda x: (x.sum() - x) / x.count())
or
sumcount = df.groupby('col1')['col2'].transform(lambda x: x.sum() + x.count())
df['col1'].map(sumcount)

#聚合
# 生成col1_mean, col1_sum与col2_count列
df['col2'] = df.groupby('col1').agg({'col1':{'col1_mean': mean, 'col1_sum‘’: sum}, 'col2': {'col2_count': count}})

# 滚动窗口均值
# 7天移动平均
# window 窗口大小
# axis=0 0:列 1:行
# min_periods 需要有值的观测点的最小数量，决定显示状态，=1表示每个观测点都有值
# center 不是往前取，前后各取
# win_type 可指定值的权重
stock['volsum7'] = stock['vol'].rolling(window=7).mean()
# 过去7天平均值（不包括当前数据）
stock['pre_volsum7'] = stock['vol'].shift(1).rolling(window=7).mean()

# 合并
df1 = pd.merge(df1, df2, left_on='trade_date', right_on='trade_date1')

# 去重
res3 = df["fengxiang"].unique()
print(res3)
['东北风' '北风' '西北风' '西南风' '南风' '东南风' '东风' '西风']

# 排序
df["aqi"].sort_values(ascending=True, inplace=False)
df.sort_values(by="aqi", ascending=True, inplace=False)

# 多列排序
df.sort_values(by=["aqiLevel", "bWendu"], ascending=[True, False])

# 协方差
# 正数，代表两组数据“同向”发展,数值越大，“同向”程度越高
# 负数代表两组数据“反向”发展,数值越大，“反向”程度越高
# 0 完全没有关系
# 字段两两比较结果
data.cov()
      年龄     身高     得分
年龄   2.5   11.5  -25.0
身高  11.5   55.7 -115.0
得分 -25.0 -115.0  250.0
# 查看年龄和身高的关系
data['年龄'].cov(data['身高'])
11.5

# 相关系数
# 1 高度相似，-1 反向相似，0 没有关系
# 字段两两比较结果
data.corr()
          年龄        身高        得分
年龄  1.000000  0.974541 -1.000000
身高  0.974541  1.000000 -0.974541
得分 -1.000000 -0.974541  1.000000
# 查看年龄和身高的关系
data['年龄'].corr(data['身高'])
0.9745412767961823
# 空气质量和温差的相关系数
res8 = df["aqi"].corr(df["白天温度"]-df["晚上温度"])

# 指数加权移动平均（EWMA）
# 常用的时间序列预测方法，适用于平稳或具有趋势的数据
#  halflife：半衰期，表示权重下降到原值一半所需时间间隔。该值越小，对历史数据影响越大。默认值为None，表示使用com参数或手动制定span或alpha参数。
#  com：衰减系数，表示相邻两个时间点距离。例如，若com=0.5，则相邻两个时间点距离为2，如果com=0.3，则相邻两个时间点距离为3。默认值为None。
#  span：时间跨度，表示跨越时间范围。设置span，halflife和com参数将被忽略。如果设置了window参数，则span将自动计算为2* window + 1。默认值为None。
#  alpha：平滑指数削弱因子，即给定时间点的权值分配。0到1之间数字。较大值意味着给过去更大权重，较小值则趋向于让预测更平稳。默认值为None。
#  min_periods：计算EWMA值需要的时间点数。默认值为1，在输入的数据点数量不足时，将使用具有缺失值的输出数据点（NaN）进行补偿。
#  adjust：是否应用修正因子，在开始时减少偏差。在时间序列中，前几个观测点对于计算正在发生的过程的均值或变化率并不具有相同重要性。参数为
True（默认值），则是根据实际样本数量n和传递给函数的decay估算出一个带修正因素的EWMA。否则忽略修正因素，会导致最初几个值比平滑后的值更偏离原始值。
#  ignore_na：是否在计算过程中包含缺失值。默认值为False。
ewma_data = data.ewm(alpha=0.5).mean()

```

###### 时间转换

```python

# 时间转换为字符串
df['date_str'] = df['datetime'].apply(lambda x:x.strftime('%Y-%m-%d'))

# 字符串转换为时间
df['datetime'] = pd.to_datetime(df['date_str'])
或
df['datetime'] = df['date_str'].apply(lambda x:pd.to_datetime(x))

# 时间索引，用于画图
import mplfinance as mpf
df = df.set_index(['datetime'])
mpf.plot(daily, type='candle', mav=(3,6,9), volume=True)

# 时间操作
pd.Timedelta('1 D')
pd.Timestamp('2019-01-01 00:00:00')
time_yesterday = datetime.datetime.now() - datetime.timedelta(days= 1)
print(time_yesterday.strftime('%Y-%m-%d'))

# 星期处理
# 0:周一 5:周六 6:周日
tmp["week_of_day"] = tmp["datetime"].dt.dayofweek
# Sunday
tmp["week_day_name"] = tmp["datetime"].dt.day_name()

```

## mplfinance

```python
import mplfinance as mpf
buypoint = []
sthdata = []
daily = daily.set_index(['trade_date'])
for date,day in daily.iterrows():
    buypoint.append(2.3 if xxx else np.nan)
    sthdata.append(-4.5)

# macd https://baike.baidu.com/item/MACD%E6%8C%87%E6%A0%87/6271283?fromtitle=MACD&fromid=3334786
exp12 = daily['close'].ewm(span=12, adjust=False).mean()
exp26 = daily['close'].ewm(span=26, adjust=False).mean()
dif = exp12 - exp26
dea = dif.ewm(span=9, adjust=False).mean()
histogram = (dif - dea) * 2

apds = [
        #成交量上叠加曲线
        mpf.make_addplot(bardata, panel=1, secondary_y=True),
        #macd
        mpf.make_addplot(histogram, type='bar', width=0.7, panel=2, color='dimgray', alpha=1, secondary_y=False),
        mpf.make_addplot(dif.to_numpy(), panel=2, color='fuchsia', secondary_y=True),
        mpf.make_addplot(dea.to_numpy(), panel=2, color='yellow', secondary_y=True),
        #自定义指标
        mpf.make_addplot(avgdata7,panel=3,linestyle='dotted',ylabel='mydata(green)',color='g'),
        mpf.make_addplot(avgdata30,panel=3,linestyle='dotted',secondary_y=True,ylabel='mydata',color='b'),
    ]
# 主图叠加买卖标记
if xxx:
    apds.append(mpf.make_addplot(buypoint,type='scatter',markersize=50,marker='^'))

#画图
mpf.plot(daily, type='hollow_and_filled', volume=True, addplot=apds, style='starsandstripes', axtitle='title')

more_points = [('2016-05-02',207),('2016-05-06',204),('2016-05-10',208.5),('2016-05-19',203.5),
       ('2016-05-25',209.5), ('2016-06-08',212),('2016-06-16',207.5)]
seq_of_seq_repeat_point_in_between=[
    [('2016-05-02',207),('2016-05-06',204)],
    [('2016-05-06',204),('2016-05-10',208.5),('2016-05-19',203.5),('2016-05-25',209.5)],
    [('2016-05-25',209.5),('2016-06-08',212),('2016-06-16',207.5)]
    ]
dates = ['2016-05-02',
    '2016-05-06',
    '2016-05-10',
    '2016-05-19',
    '2016-05-25',
    '2016-06-08',
    '2016-06-16']
datepairs = [(d1,d2) for d1,d2 in zip(dates,dates[1:])]
'''
# figscale=1.25 主图缩放
mpf.plot(daily, type='hollow_and_filled', volume=True, addplot=apds, style='starsandstripes', axtitle=‘title’
         # 横线
         ,hlines=dict(hlines=[3.4,5.6],colors=['g','r'],linestyle='-.')
         # 竖线
         ,vlines=dict(vlines=['2013-11-06','2014-11-15','2015-11-25'],linewidths=(1,2,3))
         # 带x坐标的连线
         ,alines=more_points
         #,alines=dict(alines=seq_of_seq_repeat_point_in_between,colors=['b','r','c','k','g'])
         #,alines=dict(alines=seq_of_seq_repeat_point_in_between,colors=['b','r','c'],linewidths=10,alpha=0.35)
         # 使用open close等数据的连线
         ,tlines=datepairs
         #,tlines=dict(tlines=datepairs,tline_use='high')
         #,tlines=dict(tlines=datepairs,tline_use=['open','close'])
         #,tlines=[dict(tlines=datepairs,tline_use='high',colors='g'),
         #   dict(tlines=datepairs,tline_use='low',colors='b'),
         #   dict(tlines=datepairs,tline_use=['open','close'],colors='r')
         #]
         # 保存图
         #,savefig='testsave.png'
         #,panel_ratios=(4,1) #每个panel的缩放比例，例如主图4，附图1
         # 指定主图和副图位置
         #,main_panel=2,volume_panel=0
         )

```

## Streamlit

```python
# 安装，有90多个包，建议venv
python3 -m venv .
source ./venv/bin/activate
pip install streamlit

# demo
streamlit hello

# 运行程序
streamlit run st-demo.py

# Markdown
st.markdown('Streamlit Demo')
# 设置网页标题
st.title('一个傻瓜式构建可视化 web的 Python 神器 -- streamlit')
# 展示一级标题
st.header('1. 安装')
# 展示二级标题
st.subheader('1.1 生成 Markdown 文档')
st.text('和安装其他包一样，安装 streamlit 非常简单，一条命令即可')
# 代码
code1 = '''pip3 install streamlit'''
st.code(code1, language='bash')
# 公式
st.latex()
# 小字体文本
st.caption()

# Table
df = pd.DataFrame(
    np.random.randn(10, 5),
    columns=('第%d列' % (i+1) for i in range(5))
)
st.table(df)

# 可排序表格
df = pd.DataFrame(
    np.random.randn(10, 5),
    columns=('第%d列' % (i+1) for i in range(5))
)
# highlight_null：空值高亮
# highlight_min：最小值高亮
# highlight_max：最大值高亮
# highlight_between：某区间内的值高亮
# highlight_quantile：分位数
st.dataframe(df.style.highlight_max(axis=0))

# 监控组件
col1, col2, col3 = st.columns(3)
# 三栏显示，前面是主值，后面是升跌幅
col1.metric("Temperature", "70 °F", "1.2 °F")
col2.metric("Wind", "9 mph", "-8%")
col3.metric("Humidity", "86%", "4%")

# 图表
# 折线图
chart_data = pd.DataFrame(
    np.random.randn(20, 3),
    columns=['a', 'b', 'c'])
st.line_chart(chart_data)
# 面积图
chart_data = pd.DataFrame(
    np.random.randn(20, 3),
    columns = ['a', 'b', 'c'])
st.area_chart(chart_data)
# 柱状图
chart_data = pd.DataFrame(
    np.random.randn(50, 3),
    columns = ["a", "b", "c"])
st.bar_chart(chart_data)
# 地图
df = pd.DataFrame(
    np.random.randn(1000, 2) / [50, 50] + [37.76, -122.4],
    columns=['lat', 'lon']
)
st.map(df)

# 外部图表支持
# matplotlib.pyplot
st.pyplot
# Bokeh
st.bokeh_chart
# Altair
st.altair_chart
# vega-lite
st.vega_lite_chart
# Plotly
st.plotly_chart
# PyDeck
st.pydeck_chart
# Graphviz
st.graphviz_chart

# 操作组件
button：按钮
download_button：文件下载
file_uploader：文件上传
checkbox：复选框
radio：单选框
selectbox：下拉单选框
multiselect：下拉多选框
slider：滑动条
select_slider：选择条
text_input：文本输入框
text_area：文本展示框
number_input：数字输入框，支持加减按钮
date_input：日期选择框
time_input：时间选择框
color_picker：颜色选择器

# 多媒体
# 都支持 numpy_array,bytes,file,url
st.image
st.audio
st.video

# 状态组件
# 进度条，如游戏加载进度
for i in range(101):
    st.progress(i)
    do_something_show()
# 等待提示
with st.spinner("Please wait...")
    do_something_slow()
# 页面底部飘气球，表示祝贺
do_something()
st.balloons()
# 显示信息
st.error("err")
st.warning("warning")
st.info("info")
st.success("success")
st.exception("exc")

# 布局
# 侧边栏
st.sidebar
# 多列
st.columns
# 隐藏信息，点击后可展开展示详细内容，如：展示更多
st.expander
# 包含多组件的容器
st.container
# 包含单组件的容器
st.empty

# 流程控制
# 让 Streamlit 应用停止而不向下执行，如：验证码通过后，再向下运行展示后续内容。
st.stop
# 表单，Streamlit 在某个组件有交互后就会重新执行页面程序，而有时候需要等一组组件都完成交互后再刷新（如：登录填用户名和密码），这时候就需要将这些组件添加到 form 中
st.form
# 在 form 中使用，提交表单。
st.form_submit_button

# 缓存装饰器 st.cache ，避免反复加载
DATE_COLUMN = 'date/time'
DATA_URL = ('https://s3-us-west-2.amazonaws.com/'
            'streamlit-demo-data/uber-raw-data-sep14.csv.gz')
@st.cache
def load_data(nrows):
    data = pd.read_csv(DATA_URL, nrows=nrows)
    lowercase = lambda x: str(x).lower()
    data.rename(lowercase, axis='columns', inplace=True)
    data[DATE_COLUMN] = pd.to_datetime(data[DATE_COLUMN])
    return data


```

# 常用代码

```python
#增加上级目录为可搜索目录
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

#命令行处理
import argparse
parser = argparse.ArgumentParser('''程序名称''')
parser.add_argument('-d', '--date', default = today.strftime('%Y%m%d'), metavar = '发布数据的日期')
parser.add_argument('-s', '--min_size', default = 1, metavar = '如果数据量小于`min-size`, 退出')
parser.add_argument('-S', '--max_size', default = 1000, metavar = '如果数据量大于`max-size`, 退出')
parser.add_argument('-u', '--unit', default = 'm', metavar = '大小单位，默认为m，可以指定k、m, 可以为浮点数')
args = parser.parse_args()
args.min_size
>>> 5

```
