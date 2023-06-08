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
