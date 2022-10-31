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

* Scheduler：

  * SchedulerJob
  * Executor
  * DagFileProcessor

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

## Docker 方式

```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.1.2/docker-compose.yaml'
# compose会引用这几个目录，可以放置自己的内容
make ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)\nAIRFLOW_GID=0" > .env
docker-compose -f docker-compose.yaml up

# 自定义镜像 Dockfile
FROM apache/airflow:2.2.1
COPY new_requirements.txt .
RUN /usr/local/bin/python -m pip install --upgrade pip
RUN pip install -r new_requirements.txt
COPY ./dags dags
COPY ./plugins plugins
COPY ./airflow_docker.cfg airflow.cfg
#修改docker-compose.yaml再部署
```

## 自启动运行

### systemd

* 在 scripts/systemd 目录中复制配置文件到 /usr/lib/systemd/system
* 在 /etc/sysconfig/airflow 获取环境配置，请确保在此文件中指定SCHEDULER_RUNS变量。 也可以定义其他变量，例如AIRFLOW_HOME或AIRFLOW_CONFIG

### upstart

* 可以在scripts/upstart目录中找到示例脚本，可能需要调整start on和stop on节。
* scripts/upstart/README中列出了一些可能的选项，根据需要修改.conf文件并复制到/etc/init目录。
* 如果您使用其他用户/组，请在*.conf文件中更改setuid和setgid
* 可以使用initctl手动启动，停止，查看状态

```bash
initctl airflow-webserver status
```

# 配置

## 切换到MySQL

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

## 访问授权

### 默认授权

需求文件中使用`password`子包。

```bash
# airflow.cfg
[webserver]
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth

$ cd ~/airflow
$ python
Python 2.7.9 ( default, Feb 10 2015 , 03 :28:08 )
>>> import airflow
>>> from airflow import models, settings
>>> from airflow.contrib.auth.backends.password_auth import PasswordUser
>>> user = PasswordUser ( models.User ())
>>> user.username = 'new_user_name'
>>> user.email = 'new_user_email@example.com'
>>> user.password = 'set_the_password'
>>> session = settings.Session ()
>>> session.add ( user )
>>> session.commit ()
>>> session.close ()
>>> exit ()
```

### LDAP

如果您使用的是Active Directory，并且没有明确指定用户所在的OU，则需要将`search_scope`更改为“SUBTREE”。

有效的search_scope选项可以在 [ldap3文档中](http://ldap3.readthedocs.org/searches.html%3Fhighlight%3Dsearch_scope) 找到。

```bash
[ webserver ]
authenticate = True
auth_backend = airflow.contrib.auth.backends.ldap_auth

[ ldap ]
# set a connection without encryption: uri = ldap://<your.ldap.server>:<port>
uri = ldaps://<your.ldap.server>:<port>
user_filter = objectClass = *
# in case of Active Directory you would use: user_name_attr = sAMAccountName
user_name_attr = uid
# group_member_attr should be set accordingly with *_filter
# eg :
#     group_member_attr = groupMembership
#     superuser_filter = groupMembership=CN=airflow-super-users...
group_member_attr = memberOf
superuser_filter = memberOf = CN = airflow-super-users,OU = Groups,OU = RWC,OU = US,OU = NORAM,DC = example,DC = com
data_profiler_filter = memberOf = CN = airflow-data-profilers,OU = Groups,OU = RWC,OU = US,OU = NORAM,DC = example,DC = com
bind_user = cn = Manager,dc = example,dc = com
bind_password = insecure
basedn = dc = example,dc = com
cacert = /etc/ca/ldap_ca.crt
# Set search_scope to one of them:  BASE, LEVEL , SUBTREE
# Set search_scope to SUBTREE if using Active Directory, and not specifying an Organizational Unit
search_scope = LEVEL
```

### Kerberos

### OAuth



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

# 使用

## 目录组成

* dags：存放DAG目录
* plugins：存放插件目录，包括 python 模块，钩子，操作符，传感器，宏，执行器和Web 视图。
* airflow.cfg：全局配置

## DAG 格式

```python
from datetime import datetime, timedelta
import airflow
from airflow import DAG
# 常用 Operator
from airflow.operators.python import PythonOperator
from airflow.operators.dummy_operator import DummyOperator

# Airflow 默认参数
args = {
  'owner': 'someone',
  'depends_on_past': False, # 是否要之前日期的任务执行完才运行执行当前日期任务，指定start_date的运行除外
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
  schedule_interval = '0 5 * * *', # 也可以写成crontab形式，调度器会在每天结束后（次日凌晨），开始当天的执行
  template_searchpath='scripts', # 查找相关文件的路径，如 .sql 文件
  default_args=args,
  catchup=False, # DAG执行后，如果 catchup 为 True，则调度器开始对 start_date 到现在时间，按 schedule_interval 声明的间隔周期逐一执行。
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

# 流程

## 触发规则

所有的operators都有一个 trigger_rule 参数，定义任务被触发规则：

- all_success: 所有父任务被成功执行（默认）
- all_failed：所有父任务处于failed 或者 upstream_failed 状态
- all_done：所有的父任务都完成执行（例如有些skip）
- one_failed：当至少有一位父任务失败后会触发，不会等待所有父任务完成
- one_success：当至少有一位父任务成功后会触发，并不等待所有的父任务完成
- dummy：依赖关系仅用于显示，随意触发

设置与depends_on_past（boolean）一起使用，当设置为True时，如果任务的上一个schedule计划未成功，则会阻止任务被触发。

## 感应器 Sensor

上游数据的监控感应器，例如监控FTP新数据或S3

### S3KeySensor

```python
import airflow.providers.amazon.aws.sensors.s3_key

s3_sensor = S3KeySensor(
  task_id = 'new_s3_file',
  bucket_key = 'airflow/xxx/{{ds_nodash}}/*.csv',
  wildcard_match = True, #是否通配符匹配
  bucket_name = 'xxx',
  aws_conn_id = 'aws_s3',
  timeout = 18*60*60,
  poke_interval = 30, #每30秒检查一次
  dag=dag
)
```

### ExternalTaskSensor

```python
# 等待外部单个任务
child_task1 = ExternalTaskSensor(
  task_id="child_task1",
  external_dag_id=parent_dag.dag_id,
  external_task_id=parent_task.task_id,
  timeout=600,
  allowed_states=['success'],
  failed_states=['failed', 'skipped'],
  mode="reschedule",
)

# 等待指定的一组任务完成
child_task2 = ExternalTaskSensor(
  task_id="child_task2",
  external_dag_id=parent_dag.dag_id,
  external_task_group_id='parent_dag_task_group_id',
  timeout=600,
  allowed_states=['success'],
  failed_states=['failed', 'skipped'],
  mode="reschedule",
)
```

### FileSensor

```python
waitfile_task = FileSensor(task_id="waiting_file",
	fs_conn_id="fs_default",
  filepath="/home/airflow/airflow_files/data.csv",
  poke_interval=5
)
```

## 作业依赖 Dependencies

* 可以通过菜单 Browse -> DAG Dependencies 查看
* 触发器：TriggerDagRunOperator

```python
next_dag = TriggerDagRunOperator(
  trigger_dag_id = 'download_price',
  task_id = 'dowload_next',
  execution_date = "{{ds}}",
  wait_for_completion = False
)
# 该Task运行到这个地方，会触发调用download_price
```

## 分支

### BranchPythonOperator

```python
# 注意不能使用 depends_on_past = True，因为下游会给出Skip状态
def branch_func(**kwargs):
  ti = kwargs['ti']
  xcom_value = int(ti.xcom_pull(task_ids='start_task'))
  if xcom_value >= 5:
    return 'continue_task'
  else:
    return 'stop_task'

start_op = BashOperator(
	task_id='start_task',
  bash_command="echo 5",
  xcom_push=True,
  dag=dag)

branch_op = BranchPythonOperator(
  task_id='branch_task',
  provide_context=True,
  python_callable=branch_func,
  dag=dag)

continue_op = DummyOperator(task_id='continue_task', dag=dag)
stop_op = DummyOperator(task_id='stop_task', dag=dag)

start_op >> branch_op >> [continue_op, stop_op]
```

## 特殊流程

### LatestOnlyOperator

历史回填的任务会跳过，只有当天最新正常的任务才会执行。

```python
import datetime as dt
from airflow.models import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators.latest_only_operator import LatestOnlyOperator
from airflow.utils.trigger_rule import TriggerRule

dag = DAG(
    dag_id='latest_only_with_trigger',
    schedule_interval=dt.timedelta(hours=4),
    start_date=dt.datetime(2016, 9, 20),
)

latest_only = LatestOnlyOperator(task_id='latest_only', dag=dag)

task1 = DummyOperator(task_id='task1', dag=dag)
task1.set_upstream(latest_only)
# = task1 << latest_only
# = latest_only.set_downstream(task1)
# = latest_only >> task1

task2 = DummyOperator(task_id='task2', dag=dag)

task3 = DummyOperator(task_id='task3', dag=dag)
task3.set_upstream([task1, task2])

task4 = DummyOperator(task_id='task4', dag=dag,
                      trigger_rule=TriggerRule.ALL_DONE)
task4.set_upstream([task1, task2])
```

* latest_only任务除了最后一次才会运行。
* task1 位于 latest_only 下游，会跳过除最新版本之外的所有运行。
* task2 正常运行。
* task3 由于默认 trigger_rule 为 all_success ，从task1接收时跳过，从task2接收时执行。
* task4 由于其 trigger_rule 为 all_done，两种情况都会触发。

![image](images/latest_only_with_trigger-7217379.png)

## 复制

### 动态生成DAG

用于代码雷同，但是业务逻辑不同时，批量生成DAG

```python
with DAG("loop_example") as dag:
  first = DummyOperator(task_id="first")
  last = DummyOperator(task_id="last")
  # 并行执行四个不同的branch
  options = ["branch_a", "branch_b", "branch_c", "branch_d"]
  for option in options:
    t = DummyOperator(task_id=option)
    first >> t >> last
```

```python
def generate_dag(job_type):
  with DAG(
    dag_id = f'download_{job_type}_price',
  ) as dag:
    download_task = PythonOperator(
      task_id = f'download_{job_type}_prices',
      python_callable = download_prices,
    )
    email_task = EmailOperator(
      task_id = 'send_email',
      to = 'xxx@xxx.com',
      subject = 'job {job_type} is downloaded - {{ds}}',
      html_content = """<h3>test</h3> {{ ds_nodash }}<br/>{{ dag }}""",
      dag = dag
    )
    download_task >> email_task
    return dag
job_types = ['type1','type2']
for job_type in job_types:
  globals()["Dynamic_DAG"+job_type] = generate_dag(job_type)
  
```

### SubDAG 重复多个DAG

![https://airflow.apache.org/_images/subdag_before.png](images/ff08647dba87b73d409e1a5b05326243_490x243.jpg)

多个task任务合并到一个SubDAG中：

![https://airflow.apache.org/_images/subdag_after.png](images/1513a94effe885d258965d133574b060_550x63.jpg)

```python
from airflow.operators.dummy_operator import DummyOperator

# Dag is returned by a factory method
def sub_dag ( parent_dag_name , child_dag_name , start_date , schedule_interval ):
  dag = DAG (
    ' %s . %s ' % ( parent_dag_name , child_dag_name ),
    schedule_interval = schedule_interval ,
    start_date = start_date ,
  )
  dummy_operator = DummyOperator (
    task_id = 'dummy_task' ,
    dag = dag ,
  )
  return dag

main_dag = DAG (
  dag_id = 'parent_dag' ,
  schedule_interval = timedelta ( hours = 1 ),
  start_date = datetime ( 2016 , 1 , 1 )
)

sub_dag = SubDagOperator (
  subdag = sub_dag('parent_dag', 'child_dag', main_dag.start_date,
		main_dag.schedule_interval),
  task_id = 'child_dag',
  dag = main_dag,
)
```

# 变量

## 参数传递 Variable

在程序中使用变量

```python
from airflow.models import Variable

foo = Variable.get("foo")
foo_json = Variable.get("foo", deserialize_json=True)
```

## 宏 Macros

在脚本文件中，支持jinja模版引擎的宏替换。参数是否支持需要查看文档是否有 templated 字眼。http://airflow.incubator.apache.org/code.html#macros

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
{{ macros.ds_add(ds, 7) }}
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

可以通过```user_defined_macros```传递用户变量：

```bash
# dict(foo='bar')
{{ foo }}
```

可以通过```user_defined_filters```传递过滤器：http://jinja.pocoo.org/docs/dev/api/#writing-filters

```bash
# dict(hello=lambda name:'Hello %s' % name)
{{ 'World' | hello }}
```

## 数据传递 XCom

在前后task之间传递结果，通过界面Admin->XComs，可以查看前后Task调用时传递的数据

```python
# 很多Operators 会自动push结果到xcom
 # 例如 python 中前一个函数 download_prices 直接返回变量
  return tickers
 # 后一个函数通过以下方式获取
  tickers = context['ti'].xcom_pull(task_ids='download_prices')

# 在 templates 里直接使用
SELECT * FROM {{ task_instance.xcom_pull(task_ids='foo', key='table_name') }}
```

# 配置

## 连接配置 Connection

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

## 资源池 Pools

* 资源和任务分离，做连接池复用，限制资源的并行度

* 通过Menu->Admin->Pools建立

* 在Operator中可以指定pool

```bash
job = BashOperator(
	pool='data_pool'
)
```

## 队列

* Worker 可以监听多个队列，可以启动时指定队列：```airflow worker -q spark_queue```
* 使用 CeleryExecutor 时，可以指定发送任务到 Celery 队列。配置在 `airflow.cfg` 的 `celery -> default_queue`

# 操作

## 提供商 Provider

60多个，不同的开源社区分别维护，如Amazon提供的各种Hook和Operator

## Hook



## 操作类 Operator

https://airflow-doc-zh.readthedocs.io/zh_CN/latest/31/

### BashOperator

```python
from airflow.operators.bash import BashOperator

t1 = BashOperator(task_id='b1', bash_command='date', dag=dag)
t2 = BashOperator(task_id='b2', bash_command='sleep 1', retries = 3, dag=dag)

templated_command = """
	{% for i in range(5) %}
		echo "{{ ds }}"
		echo "{{ macros.ds_add(ds, 7) }}"
		echo "{{ params.my_param }}"
		echo "{{ dag_run.conf['conf'] }}"
	{% endfor %}
"""

bash_task = BashOperator(
  task_id = 'bash_task',
  bash_command = templated_command, # or "echo value: {{ ds }}"
  params = {'my_param': 'value'},
  env = {'EXECUTION_DATE': "{{ ds }}"}, # 也可以对应 jinja 模版
  dag = dag,
)
t2.set_upstream(t1)
bash_task.set_upstream(t2)

# 等于
dag.set_dependency('t1', 't2')
dag.set_dependency('t2', 'bash_task')
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

### S3ListOperator

```python
list_s3_file = S3ListOperator(
  task_id = 'list_s3_files',
  bucket = 'xxx',
  prefix = 'airflow/xxx',
  delimiter = '/',
  aws_conn_id = 'aws_s3',
)
```

### HiveOperator

```python
hive_task = HiveOperator(
  task_id="hive_task",
  hql="LOAD DATA INPATH '/tmp/data_cleaned.csv' INTO TABLE tweets PARTITION(df='2018-10-01')"
)
```

# 分布式执行

## CeleryExecutor

* 需要在airflow.cfg配置队列服务，RabbitMQ，Redis等。http://docs.celeryproject.org/en/latest/getting-started/brokers/index.html

* Worker中需要：
  * 安装airflow，CLI需要在可执行路径中
  * 相关支持项，例如HiveOperator，需要安装hive CLI；MySqlOperator 需要PYTHONPATH提供所需的Python库
  * Worker中DAGS_FOLDER，需要自行把DAG同步，常见使用S3或Chef，Puppet，Ansible等使用git同步。
  * Worker中运行：```airflow worker``` 启动
  * 确保在 celery_broker_transport_options 中设置超过最长运行任务的ETA的可见性超时
  * 任务可以并消耗资源，确保 Worker 有足够的资源来运行 worker_concurrency 任务
* 可以使用 ```airflow flower``` 或 ```celery flower``` 启动Web UI观察。

## DaskExecutor

* 支持单机器和远程集群运行。Dask不支持队列。 https://distributed.readthedocs.io/

* Dask Worker必须能够导入Airflow和您需要的任何依赖项。

```bash
DASK_HOST = 127.0.0.1
DASK_PORT = 8786
dask-scheduler --host $DASK_HOST --port $DASK_PORT
# 编辑airflow.cfg以将执行程序设置为DaskExecutor并在[dask]部分中提供Dask Scheduler地址

#worker
dask-worker $DASK_HOST : $DASK_PORT
```

## MesosExecutor

* https://www.kancloud.cn/luponu/airflow-doc-zh/889668
* 可以直接在 mesos slave上运行Airflow任务，要求安装和配置Airflow。
* 在安装了Airflow的docker容器内运行，该容器在mesos slave上运行。

# 高级设置

## 集群策略

在设置文件中，可以设置集群全局policy，参考：airflow_settings.py

```python
def policy( task ):
  # 凡是 HivePartition 操作放置到 sensor_queue
  if task.__class__.__name__ == 'HivePartitionSensor' :
    task.queue = "sensor_queue"
  # 运行超时强制为 48 小时内
  if task.timeout > timedelta( hours = 48 ):
    task.timeout = timedelta( hours = 48 )
```

## 文档注释

在Web界面中，DAG 在 Graph View 中，任务 在 Task Details 中，可以用 doc_md 配置显示内容

```python
dag = DAG( 'my_dag', default_args = default_args )
dag.doc_md = __doc__
t = BashOperator( "foo", dag = dag )
t.doc_md = """ \
#Title"
Here's a [url](www.airbnb.com)
"""
```

## DAG程序包

可以在Dags目录里放置zip包，Airflow只会扫描zip包中根目录的文件。

模块依赖常用做法，制作成zip包：

```bash
virtualenv zip_dag
source zip_dag/bin/activate

mkdir zip_dag_contents
cd zip_dag_contents

pip install --install-option = "--install-lib= $PWD " my_useful_package
cp xxx/sth_dag.py .

zip -r zip_dag.zip *
```

## 日期时区

* WebUI总是显示UTC。
* 程序中的时区，可以在 airflow.cfg中 [core] 下设置 default_timezone，默认为 utc，可改为其他 IANA 时区，例如Europe/Amsterdam，但是建议使用UTC。
* DAG中使用这个时区值来处理datetime声明的时间，如果设置了时区，start_date = datetime(2017,1,1) 是指当地时间。
* Airflow 在模板中返回时区敏感的 datetime，但不会将它们转换为本地时间，因此它们仍然是 UTC 时区。 由 DAG 来处理转换。

```python
from airflow.utils import timezone

# 当前时间建议使用utc
now = timezone.utcnow()
a_date = timezone.datetime( 2017, 1, 1 )

# 判断是否有时区
timezone.is_aware()

# 判断是否UTC
timezone.is_naive()
```

```python
# 使用时区
import pendulum

# 建议使用明确说明时区的datetime来声明时间
local_tz = pendulum.timezone( "Europe/Amsterdam" )
default_args = dict(
  start_date = datetime( 2016, 1, 1, tzinfo = local_tz ),
  owner = 'Airflow'
)
dag = DAG ( 'my_tz_dag', default_args = default_args )
print( dag.timezone ) # <Timezone [Europe/Amsterdam]>
```

```python
# 转换时间到时区

local_tz = pendulum.timezone( "Europe/Amsterdam" )
local_tz.convert( execution_date )
```

## SLA

Operator 中有sla( datetime.timedelta )字段，说明任务允许执行时间，例如作业一小时没有运行完发送报警邮件。



# 常用命令

https://www.kancloud.cn/luponu/airflow-doc-zh/889675

```bash
# 创建用户
airflow users create --username user --firstname f --lastname l --role Admin --email e@mail.com
# 附加角色
airflow users add-role -u user --role op
# 手工测试
airflow test dag_id task_id date
# 信息
airflow info
airflow list_dags
airflow list_tasks tutorial
airflow list_tasks tutorial --tree
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
airflow backfill test -s 2013-01-01 -e 2013-02-20
```

# 最佳实践

* 推荐配置存 Connection
* 不建议用Variable，因为需要数据库连接存取metadata的
* DAG定义文件中，在外层不要有资源昂贵的操作，例如数据库读取，API调用。
* 有些 时候可以直接修改metadata数据库，批量配置例如用户，权限等。
