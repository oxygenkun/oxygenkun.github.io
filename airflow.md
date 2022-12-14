---
title: airlfow
theme: white
---
# 
<!-- .slide: data-background="./image/front.png" -->

<img src="./image/airflow_wordmark_1.png" height="200" />


**大数据领域非常热门的开源任务编排框架**

以code形式构建和执行调度工作流

[https://airflow.ecdataway.com/](https://airflow.ecdataway.com/ "https://airflow.ecdataway.com/")

---

## 主要场景：

*   data pipeline：ETL

*   data analysis

*   data preparing for machine learning

----

#### 业务上遇到的问题

- 放在服务器的cron脚本是否正常在运行？

- 有复杂先依赖关系的脚本，怎么保证先后顺序？

- 脚本分散在各个服务器，难以管理。

----

### 对比服务器配置cron：

* 可在网页直接观察每日任务执行情况

![](image/image_0N1YIA8liK.png)

----

### 对比服务器配置cron：

* 可以网页查看运行日志

![](image/log.png)

----

### 对比服务器配置cron：

* 可以做复杂的调度依赖管理

![](<image/dags1.png>)

----

### 对比服务器配置cron：

* 可以启停重跑任务

![](<image/log.png>)

[演示](https://airflow.ecdataway.com/dags/tutorial/grid)

----

#### 业务上的主要实践

场景：对京东提供全量数据

保证ELT当中的**E**xtract和**L**oad部分稳定运行。

- Data Pipeline 托管大部分（80%）的数据调度任务

- DQC (data quality check) 托管数据质量检测任务

- Data Backup 托管每日Clikchouse数据备份任务以及MySQL主从备份状态监控

---

##  架构与核心概念

- **有向无环图 DAG** （ Directed Acrylic Graphs）

- **DAG** 细分成不同的tasks任务

![](./image/edge_label_example.png)

----

### 架构

![](./image/arch-diag-basic.png)


----

### 定义tasks

airflow 提供丰富的class

- Operators：执行模块

    `BashOperator`, `PythonOperator`, `MysqlOperator`, `SparkOperator` ...

- Sensor：特殊的Operator，只有监控触发机制

    `FileSensor`, `ExternalTaskSensor` ...

<img src="./image/edge_label_example.png" height="100" />

----

#### 定义控制流

airflow提供的python语法述task之间依赖关系


```python
task_a = ... #可以是一个sensor监控一个条件是否存在
task_b = ... #operator1，从DB1存到DB2
task_c = ... #operator2，从DB3存到DB2
task_d = ... #operator3，DB2合并数据到目标表

task_a >> [task_b, task_c] >> task_d
```

![](./image/basic-dag.png)

----

[example\_complex - Graph - Airflow (ecdataway.com)](https://airflow.ecdataway.com/dags/example_complex/graph)

----

#### 定义调度间隔

- cron表达式语法

    ```
    "0 1 * * *"
    ```

- 预设

    ```
    "@once", 
    "@hourly" 
    ...
    ```
----

#### 调度启停

定义DAG的时候要考虑到幂等性和原子性

![](./image/cut2.png)

----

#### 模板导入宏变量

通过jinja模板引擎传入宏变量

```python
templated_command = dedent(
    """
{% for i in range(5) %}
    echo "{{ ds }}"
    echo "{{ macros.ds_add(ds, 7)}}"
{% endfor %}
"""
)

t3 = BashOperator(
    task_id='templated',
    depends_on_past=False,
    bash_command=templated_command,
)
```

---

#  airflow dag 编写

```python [1-8|9-38|33-43|45-58|59-61|62-75]
from datetime import datetime, timedelta
from textwrap import dedent
import pendulum
# The DAG object; we'll need this to instantiate a DAG
from airflow import DAG

# Operators; we need this to operate!
from airflow.operators.bash import BashOperator
with DAG(
    'tutorial',
    description='A simple tutorial DAG',
    schedule_interval="0 1 * * *",
    start_date=pendulum.datetime(2022, 9, 22, tz='Asia/Shanghai'),
    catchup=False,
    tags=['example'],
    # 传入给task的默认参数
    default_args={
        'depends_on_past': False,
        'email': ['airflow@example.com'],
        'email_on_failure': False,
        'email_on_retry': False,
        'retries': 1,
        'retry_delay': timedelta(minutes=5),
        # 'wait_for_downstream': False,
        # 'execution_timeout': timedelta(seconds=300),
        # 'on_failure_callback': some_function,
        # 'on_success_callback': some_other_function,
        # 'on_retry_callback': another_function,
        # 'trigger_rule': 'all_success'
    },
) as dag:

    t1 = BashOperator(
        task_id='print_date',
        bash_command='date',
    )

    t2 = BashOperator(
        task_id='sleep',
        depends_on_past=False,
        bash_command='sleep 5',
        retries=3,
    )
    
    templated_command = dedent(
        """
    {% for i in range(5) %}
        echo "{{ ds }}"
        echo "{{ macros.ds_add(ds, 7)}}"
    {% endfor %}
    """
    )

    t3 = BashOperator(
        task_id='templated',
        depends_on_past=False,
        bash_command=templated_command,
    )
    
    t1 >> [t2, t3]
    
    t1.doc_md = dedent(
        """\
    #### Task Documentation
    You can document your task using the attributes `doc_md` (markdown),
    `doc` (plain text), `doc_rst`, `doc_json`, `doc_yaml` which gets
    rendered in the UI's Task Instance Details page.
    ![img](http://montcs.bloomu.edu/~bobmon/Semesters/2012-01/491/import%20soul.png)
    **Image Credit:** Randall Munroe, [XKCD](https://xkcd.com/license.html)
    """
    )
    dag.doc_md = __doc__  # providing that you have a docstring at the beginning of the DAG; OR
    dag.doc_md = """
    This is a documentation placed anywhere
    """ # otherwise, type it like this

```

----

[查看DAG](https://airflow.ecdataway.com/dags/tutorial_dag/grid)

---

# 一些细节

----

### 渐进式整合Python脚本

| 不耦合 | 部分耦合 | 完全耦合 |
| -------| ----- | ---- |
| `BashOperator` 调用Python脚本 | `PythonOperator` 调用Python函数 | TaskFlow API 装饰器封装 Python函数 |

----

## 不同风格的Python Operator

1. 基础 API（1.0版本）
2. taskflow API（2.0版本）

----

基础PythonOperator定义

```python
from airflow.operators.python import PythonOperator

def insert_category_mysql(month):
    pass

task_category = PythonOperator(
        task_id='task_category',
        python_callable=insert_category_mysql,
        op_kwargs={
            'month': '{{data_interval_start.in_timezone("Asia/Shanghai") | ds}}'
        }
    )
```
----

taskflow定义

```python
from airflow.decorators import task

@task
def task_category(data_interval_start=None):
    month = data_interval_start.in_timezone("Asia/Shanghai").to_date_string()
    pass

task_category = task_category()
```

----

### 最佳实践

将业务逻辑和调度定义逻辑分开

在调度逻辑中使用业务函数

- `dag_some_task.py` : 调度脚本。定义airflow相关的DAG、Tasks等。

- `some_task.py` : 业务逻辑脚本。将业务封装成不同的函数。

---

# DAGs仓库

[Airflow需要的DAGs仓库](https://git.sh.nint.com/yang.jingming/airflow-dags)

```
- src/
  - utils/
  - dags/
    - yjm/
      - dags_1.py
      ...     
    - yxp/
```

> 服务器自动同步仓库代码

----

# toolkit仓库

解决不同Python仓库都要使用的公司工具类

在各个仓库不一致的问题

> 同一个函数，有的仓库会直接抛出异常，另一个仓库居然打印log后直接执行。

```
- config
 ...
- lib
 - PyMysqlPool.py
 - MyCkClient.py
 ...
```

[nint_tookkit仓库地址](https://git.sh.nint.com/yang.jingming/nint_toolkit)

----

安装

```bash
pip install \
git+ssh://git@git.sh.nint.com/yang.jingming/nint_toolkit.git@master
```

使用

```python
from nint_toolkit.lib.MyCkClient import MyCkClient
```

---

谢谢

Q&A
