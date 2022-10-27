# 介绍

* airflow 是一个编排、调度和监控workflow的平台。
  * 由Airbnb开源，现在在Apache Software Foundation 孵化。
  * airflow 将workflow编排为由tasks组成的DAGs(有向无环图)，调度器在一组workers上按照指定的依赖关系执行tasks。
  * airflow 提供了丰富的命令行工具和简单易用的用户界面以便用户查看和操作，并且airflow提供了监控和报警系统。
  * Airflow管道使用Jinja模版引擎，可以动态生成。

* crontab的不足：
  * 多任务之间依赖，任务消耗时间
  * 进度查看和执行日志，历史调度情况
  * 出错重试和报警

* 同类付费软件：Informatica，Talend，Control-M，Fivetran
* ETL 痛点：
  * 日益增加数据量和不均衡峰值
  * 快速排查任务失败原因，重试，监控，报警
  * 新工具配置规范，敏捷开发


# 概念

* DAGs：即有向无环图(Directed Acyclic Graph)，将所有需要运行的tasks按照依赖关系组织起来，描述的是所有tasks执行的顺序。

* DAG Run：是DAG的一个运行实例。

* Operators：airflow内置了很多operators，如：

  * BashOperator 执行一个bash 命令
  * PythonOperator 调用任意的Python 函数
  * EmailOperator 用于发送邮件
  * HTTPOperator 用于发送HTTP请求
  * SqlOperator 用于执行SQL命令
  * 用户可以自定义Operator，这给用户提供了极大的便利性。可以理解为用户需要的一个操作,是Airflow提供的类

* Tasks：Task 是 Operator的一个实例

  * Task Instance：由于Task会被重复调度，每次task的运行就是不同的task instance了。
  * Task instance 有自己的状态，包括"running", "success", "failed", "skipped", "up for retry"等。
  * Task Relationships：DAGs中的不同Tasks之间可以有依赖关系

* Executor：执行器，任务具体怎样运行

  * SequentialExecutor：单进程顺序执行任务，默认执行器，通常只用于测试。

  * LocalExecutor：多进程本地执行任务。

  * CeleryExecutor：分布式调度，生产常用。需要依赖三方队列redis或rabbitmq。任务由Master node的Celery executor发送到队列里，Worker node上的Celery worker接收任务进行。

    <img src="images/image-20221027140457865.png" alt="image-20221027140457865" style="zoom:50%;" />

  * DaskExecutor：动态任务调度，主要用于数据分析。

* DAG的属性：

  * Max Active Runs：最多多少个DAG实例同时运行（DAG Run）
  * Concurrency：单个DAG里可以并发运行多少个任务


# 安装

```bash
#如果考虑虚拟环境
python3 -m venv /path/to/new/env

# 安装指南
https://airflow.apache.org/docs/apache-airflow/stable/start/local.html

# 需要把变量添加到 ~/.bashrc
export AIRFLOW_HOME=~/airflow
# 如果是Mac上跑，需要增加以下变量
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES

# install from pypi using pip
pip install apache-airflow
# initialize the database
airflow initdb
# start the web server, default port is 8080
airflow webserver -p 8080 -D
# start the scheduler
airflow scheduler -D
# visit localhost:8080 in the browser and enable the example dag in the home page

```

# 配置

## 切换MySQL

```bash
# 注意，默认单机airflow是用本机sqlite存储，高可用需要考虑切换为MySQL或PG
docker run --name mysql_server -e MYSQL_ROOT_PASSWORD=pass -p 3306:3306 -d mysql:5.7
create database airflow character set utf8;
global variable explicit_defaults_for_timestamp needs to be on(1)
# 修改airflow.cfg
sql_alchemy_conn = mysql+mysqlconnector://user:pass@host:3306/airflow
# 安装插件
pip install mysql-connector-python==8.0.22
# 初始化
airflow db init
```

## 邮件配置

```bash
# airflow.cfg
[stmp]
smtp_host = xxx
xxx
# 使用 email operator 发送邮件
```

## 日志

```bash
[logging]
base_log_folder = 
remote_logging = True
remote_log_conn_id = aws_s3
remote_base_log_folder = s3://xxx/airflow/logs
xxx
```

## API

```bash
#开启远程调用API
auth_backend = airflow.api.auth.backend.basic_auth
#可以通过web管理界面测试
```



# 目录

* dags：存放DAG目录
* airflow.cfg：全局配置

# 使用

## DAG 格式

```python
from datetime import datetime, timedelta
import airflow
# 常用 Operator
from airflow.operators.python import PythonOperator
from airflow.operators.dummy_operator import DummyOperator

# Airflow 默认参数
args = {
  'owner': 'someone',
  'depends_on_past': False,
  'start_date': datetime(2019, 7, 26), #start_date会决定这个DAG从哪天开始生效
  'start_date': days_ago(2),# 也可以写成两天前

  #出错通知
  'email': ['airflow@example.com'],
  'email_on_failure': False,
  'email_on_retry': False,

  #出错重试次数和间隔
  'retries': 1,
  'retry_delay': timedelta(minutes=5),

  # 'queue': 'bash_queue',
  # 'pool': 'backfill',
  # 'priority_weight': 10,
  # 'end_date': datetime(2016, 1, 1),
}

#dag图
with DAG(
  'test_param_sql', # 也可以指定dag_id
  description = 'text',
  tags = ['tag1','tag2'],
  schedule_interval = timedelta(days=1), # schedule_interval是调度的频率
  schedule_interval = '0 5 * * *', # 也可以写成crontab形式
  template_searchpath='scripts', # 查找相关文件的路径，如 .sql 文件
  default_args=args,
  catchup=False, #表示任务如果赶不及是否需要重新跑，例如每天的数据，前两天没有跑，是否需要重跑
  max_active_runs=1, # 避免同时运行造成冲突，例如引起MySQL dead lock
) as dag:
  #各种Operator
  bash_task = BashOperator(
    task_id = 'bash_task',
    bash_command = "echo value: {{ dag_run.conf['conf']}}",
    dag = dag,
  )
  match_finish = DummyOperator(
    task_id='match_finish',
    dag=dag
  )

#Operator 之间连接
bash_task >> match_finish
```

## 带参数运行DAG

运行时指定参数

```bash
# access configuration in DAG use {{ dag_run.conf }}
# use in PythonOperator functions
def hello_function(**context):
	conf = context["dag_run"].conf.get("conf")
	# 判断空值读取
	conf = context["dag_run"].conf.get("conf", None) if ("dag_run" in context and context["dag_run"] is not nil)
example = PythonOperator(
  task_id='python_example',
  python_callable = hello_function,
  provide_context = True,
)
# use in BashOperator
bash_task = BashOperator(
  bash_command = "echo value: {{ dag_run.conf['conf']}}",
)

# trigger with cli
airflow dags trigger --conf '{"conf":"value"}' example_dag
# trigger with web ui in configure textarea
{"conf":"value"}
```

## 参数传递 Variable

在程序中使用变量

```python
from airflow.models import Variable

foo = Variable.get("foo")
foo_json = Variable.get("foo", deserialize_json=True)
```

## 数据传递 XCom

在前后task之间传递结果

```python
# 很多Operators 会自动push结果到xcom
 # 例如 python 中前一个函数 download_prices 直接返回变量
  return tickers
 # 后一个函数通过以下方式获取
  tickers = context['ti'].xcom_pull(task_ids='download_prices')

# 在 templates 里直接使用
SELECT * FROM {{ task_instance.xcom_pull(task_ids='foo', key='table_name') }}
```

## Macros

在脚本文件中，支持jinja模版引擎的宏替换。参数是否支持需要查看文档是否有 templated 字眼

```bash
{{ ds }}          当天的 YYYY-MM-DD 格式的日期
{{ ds_nodash }}   当天的 YYYYMMDD 格式的日期
同样支持前后天的变量和对应的 _nodash 变量：{{ prev_ds}} {{ next_ds }} {{ yesterday_ds }} {{ tomorrow_ds }}
{{ ts }}          2018-01-01T00:00:00+00:00
{{ ts_nodash }}   20180101T000000
{{ ts_nodash_with_tz }} 20180101T000000+0000
{{ execution_date }} 运行时间
{{ prev_execution_date }} 运行前后
{{ dag }}         DAG对象
{{ task }}        Task对象
{{ macros }}      对应魔术函数，如：macros.datetime，macros.timedelta，macros.dateutil，macros.time，macros.uuid，macros.random
{{ ti }} {{ task_instance }} Instance对象
{{ params }}      传递参数，airflow.cfg 里允许 dag_run_conf_overrides_params，通过 trigger_dag -c 传入
{{ var.value.my_var }} 变量
{{ var.json.my_var.path }} JSON变量
{{ task_instance_key_str }} 唯一字符串，{dag_id}__{task_id}__{ds_nodash}
{{ conf }}        airflow.cfg配置，对应airflow.configuration.conf
{{ run_id }}      当前DAG run_id
{{ dag_run }}     DagRun对象
{{ test_mode }}   是否客户端运行的测试模式
```

## Connection

通过connection进行数据库连接信息的配置，实现和代码分离。

### MySQL Connection

```python
from airflow.hooks.base_hook import BaseHook

conn = BaseHook.get_connection('mysql_default')
mydb = mysql.connector.connect(host=conn.host,
                               user=conn.login,
                               password=conn.password,
                               database=conn.schema,
                               port=conn.port)
mycursor = mydb.cursor()
for ...
	sql = """INSERT INTO tab(x,x,x) VALUES(%s, %s, %s)"""
	mycursor.executemany(sql, vals)
mydb.commit()
```

## Pools



## Operator

### BashOperator

```python
from airflow.operators.bash import BashOperator

bash_task = BashOperator(
  task_id = 'bash_task',
  bash_command = "echo value: {{ dag_run.conf['conf']}}",
  dag = dag,
  env = {'EXECUTION_DATE': "{{ ds }}"} # 对应 jinja 模版
)
```

### PythonOperator

```python
from airflow.operators.python import PythonOperator

def hello_function(**context):
  print('hello')

example = PythonOperator(
  task_id='python_example',
  python_callable = hello_function,
  provide_context = True,
)
```

### MySqlOperator

```python
from airflow.providers.mysql.operators.mysql import MySqlOperator

mysql_task = MySqlOperator(
  task_id = 'mysql_job',
  mysql_conn_id='mysql_default',
  sql='mysql_sql.sql',
  dag=dag,
)
```

### PostgresOperator

```python
from airflow.operators.postgres_operator import PostgresOperator

'''
param_sql.sql 文件中支持变量替换：
insert into test.param_sql_test
select * from test.dm_input_loan_info_d
where period = {{params.period}};
'''
test_param_sql = PostgresOperator(
  task_id='test_param_sql',
  postgres_conn_id='postgres_default',
  sql='param_sql.sql',
  dag=dag,
  params={'period': '201905'},
  pool='pricing_pool')
```

### EmailOperator

```python
from airflow.operators.email_operator import EmailOperator

email_task = EmailOperator(
  task_id = 'send_email',
  to = 'xx@xx.com',
  subject = 'some thing',
  html_content = """<h3>email test</h3>""",
  dag = dag,
)
```

# 常用命令

```bash
# 创建用户
airflow users create --username user --firstname f --lastname l --role Admin --email e@mail.com
# 附加角色
airflow users add-role -u user --role op
# 手工测试
airflow test dag_id task_id date
# 信息
airflow info
# 常用命令
airflow cheat-sheet
# 读取配置
airflow config get-value core dags_folder
# DAG
airflow dags list
airflow dags unpause dag_name
# 运行 trigger
airflow dags trigger dag_name
airflow dags list-runs
# 运行某个 Task
airflow tasks run dag_name task_name "2022-09-10 11:55:00"
# 重跑失败任务
airflow dags backfill -s 2022-09-10 -e 2022-09-12 --rerun-failed-tasks
```

