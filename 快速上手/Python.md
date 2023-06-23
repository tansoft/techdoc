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

## Pandas/numpy/DataFrame

```python
import numpy as np
import pandas as pd
```

### 创建DataFrame

```python
#生成0-15顺序数，把一维数组转换为4*4数组
data = pd.DataFrame(np.arange(16).reshape(4,4), index=list('abcd'), columns=list('ABCD'))
    A   B   C   D
a   0   1   2   3
b   4   5   6   7
c   8   9  10  11
d  12  13  14  15
#指定列名
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
```

### 定位操作

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

# 行数
len(data)
# 列数
print data.columns.size
# 列名
data.columns
# 行名
data.index

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

# 更改某值
data.loc[8825,'wx_phrase'] = 'T-Storm'

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

# 填充空值为0.0，还可以填充 pd.NA 代替 np.nan，它代表空整数、空布尔、空字符
# 如果是现有的空值，可以用<NA>做为字符串来找到它
data['precip_hrly'] = data['precip_hrly'].fillna(0.0)

# 替换列中A值到B
replace = dict{'A':'B'}
data['wx_phrase'] = data.wx_phrase.replace(replace)

# 变换数据类型
data["RNC编号"].astype("Int64")

# 将ratio列中不为零的行的angle字段对应值置为1
branch.loc[branch.ratio !=0, "angle"] = 1

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

# 增加行，可以先新建一个df，再append进去
new=pd.DataFrame({'time':time, 'aa':8, 'bb':6}, index=[len(data)]) 
data = data.append(new)

# 增加列
data['count'] = 1

# 删除行，括号中为行索引
data = data.drop(0)

# 删除列，输入列名
data = data.drop('precip_total', axis=1)

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

```
### 操作

```python
# 采样
DataFrame.sample(n=None, frac=None, replace=False, weights=None, random_state=None, axis=None)
# n是要抽取的行数
# frac是抽取的比例（0-1之间，frac=0.8，就是抽取其中80%）
# replace：是否为有放回抽样，replace=True时为有放回抽样。replace=False(默认为False)是无放回的采样，当采样数n大于样本数且没有设置replace=True时，会出现异常
# weights：指定样本抽中的概率，默认等概论抽样
# random_state：指定抽样的随机种子，可以使得每次抽样的种子一样，每次抽样结果一样

# 遍历
# itertuples 比 iterrows 快
# 使用 df.items() 返回每列对应数据
# 使用 df.iterrows() 返回每行对应数据
# 使用 df.itertuples() 返回每行对应的数据，namedtuples对象

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


#全列操作，对全列每行执行some_function，axis=1为每行执行，axis=0为每列执行
df.apply(lambda row: some_function(row), axis=1)

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
# x列各值出现次数
data['x'].value_counts()
# 对每一列数据进行统计，包括计数，均值，std，各个分位数
data.describe()

# 判断列PULocationID中是否含有exc列表内容
data[data['PULocationID'].isin(['6','10'])]
data[data.PULocationID.isin(exc)]
# 判断不在没有isnotin，只能用apply，性能很差
data.PULocationID[data.apply(lambda x:x.PULocationID not in exc, axis=1)]

# One-hot 编码，可多列同时
pd.get_dummies(df, columns=["diw", "dim"])

# gc回收，循环处理大量数据时建议主动回收
import gc
del data
gc.collect()

```

### 时间转换

```python
# 字符串转时间
time = pd.to_datetime('2019-01-01 00:51:00')

# 列转换为时间Timestamp类型
data['valid_time_gmt'] = pd.to_datetime(data['valid_time_gmt'])
或
data['valid_time_gmt'] = data['valid_time_gmt'].apply(lambda x:pd.to_datetime(x))

# 时间操作
pd.Timedelta('1 D')
pd.Timestamp('2019-01-01 00:00:00')

# 星期处理
# 0:周一 5:周六 6:周日
tmp["week_of_day"] = tmp["datetime"].dt.dayofweek
# Sunday
tmp["week_day_name"] = tmp["datetime"].dt.day_name()

```

### 打印

```python
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
