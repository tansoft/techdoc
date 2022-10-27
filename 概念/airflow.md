# 介绍

* airflow 是一个编排、调度和监控workflow的平台。
* 由Airbnb开源，现在在Apache Software Foundation 孵化。
* airflow 将workflow编排为由tasks组成的DAGs(有向无环图)，调度器在一组workers上按照指定的依赖关系执行tasks。
* airflow 提供了丰富的命令行工具和简单易用的用户界面以便用户查看和操作，并且airflow提供了监控和报警系统。

# 概念

* DAGs：即有向无环图(Directed Acyclic Graph)，将所有需要运行的tasks按照依赖关系组织起来，描述的是所有tasks执行的顺序。
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

# 安装

```bash
export AIRFLOW_HOME=~/airflow
# install from pypi using pip
pip install apache-airflow
# initialize the database
airflow initdb
# start the web server, default port is 8080
airflow webserver -p 8080
# start the scheduler
airflow scheduler
# visit localhost:8080 in the browser and enable the example dag in the home page
```

