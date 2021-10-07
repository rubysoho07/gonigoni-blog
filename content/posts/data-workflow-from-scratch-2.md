---
title: "데이터 분석 워크플로우를 처음부터 만들어 보기 (2)"
date: 2021-04-26T21:20:19+09:00
tags: [Airflow, Big Data]
draft: false
categories: [Data Engineering]
comments: true
---

2월에 올렸던 [데이터 분석 워크플로우를 처음부터 만들어 보기]({{< ref "data-workflow-from-scratch.md" >}})에 이어서 작성하는 두번째 글입니다. 

그동안 시도해 봤던 것들은 다음과 같습니다. 

* S3 버킷에 원격으로 로그 올리도록 설정하기
* Airflow 2.x 버전에서 KubernetesExecutor를 사용하는 데, DAG이나 Task를 수동으로 실행할 때 에러가 발생하는 이유는?
* DAG에서 DB 이용하기: DB와 관련된 Operator 이용하기, Hooks 이용하기
* S3에서 파일을 가져와서 분석하기: S3Hook

테스트 한 환경은 Airflow 2.0.1, 2.0.2 버전입니다.

전체 내용은 [GitHub 저장소](https://github.com/rubysoho07/data-workflow-from-scratch)에서 확인하실 수 있습니다. 

# S3 버킷에 원격으로 로그를 올리도록 설정하기

지난 글에서 시스템 구성으로 KubernetesExecutor를 이용한다고 말씀드렸습니다. KubernetesExecutor를 이용하는 경우, Task가 끝나면 Worker Pod이 사라지면서 로그를 못 찾는 경우도 있기 때문에 로그를 S3 버킷에 저장해 보았습니다. 

먼저 S3에 대한 connection을 설정해 봅니다. `s3-bucket-name`은 변경하지 않아도 됩니다.

```python
import json

from airflow.models.connection import Connection

c = Connection(
    conn_id='test_s3', 
    conn_type='aws', 
    login='my_access_key', 
    password='secret_access_key'
)

print(c.get_uri())
# 결과: 'aws://my_access_key:secret_access_key@'
```

이 값을 기준으로 `AIRFLOW_CONN_(CONNECTION_ID_NAME)` 이라는 환경변수를 설정합니다. 이렇게 하면 `connection_id_name`이라는 Connection이 설정됩니다. 

다만, 이 환경 변수로 지정한 Connection은 Airflow UI에 표시되지 않음에 유의합니다. ([출처](http://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html#storing-a-connection-in-environment-variables))

아니면 Airflow 웹서버에서 Connection Type을 AWS로 설정하고, 각각의 항목을 채워줍니다. 이렇게 하면 Airflow UI에서도 Connection 정보를 볼 수 있습니다.

S3이나 AWS Connection 설정은 [다음 문서](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/connections/aws.html)를 참고해 주세요. 

그리고 `airflow.cfg` 파일 중 다음 내용을 수정합니다. `connection_id_name`은 앞에서 설정한 Connection 이름으로, `your-bucket-name` 부분은 로그를 저장할 버킷 이름으로 바꾸어 줍니다.

```
[logging]
remote_logging = True
remote_log_conn_id = connection_id_name
remote_base_log_folder = s3://your-bucket-name
```

또는 다음과 같은 환경 변수로 설정을 변경할 수 있습니다. 

* AIRFLOW__LOGGING__REMOTE_LOGGING
* AIRFLOW__LOGGING__REMOTE_LOG_CONN_ID
* AIRFLOW__LOGGING__REMOTE_BASE_LOG_FOLDER

# Airflow 2.x 버전에서 KubernetesExecutor를 사용하는데, DAG이나 Task를 수동으로 실행할 때 에러가 발생하는 이유는?

(2.0.2 버전에서 해결되었습니다.)

KubernetesExecutor를 이용하는데, `airflow tasks run` 명령으로 Task를 수동으로 실행하려고 시도했습니다. 

그런데 `Could not get scheduler_job_id` 오류가 나면서 Task 실행이 안 되는 현상이 있었는데요. 혹시 뭔가 잘못된 게 있어서 확인해 보니, GitHub에 다음과 같은 이슈가 올라와 있었네요.

* [#13805: Could not scheduler_job_id](https://github.com/apache/airflow/issues/13805)

대략 내용을 살펴보면 다음과 같습니다.

* 1.10.14 버전에서 2.0.0 버전으로 업데이트했는데 이런 오류가 발생했다. 
* KubernetesExecutor를 Executor로 지정한 경우에만 발생한다. 
* Backfill을 수행하는 경우에도 동일한 오류가 발생했다. 
* Scheduler Job에 대해서는 job_id가 지정되는데, Backfill 하는 경우에는 지정하지 않는 이슈였다. 

이 이슈는 다음 Pull Request로 해결되었고, 2.0.2 버전에 반영되어 있습니다. 

* [#14160: Unable to trigger backfill or manual jobs with Kubernetes executor](https://github.com/apache/airflow/pull/14160)

# DAG에서 DB 이용하기

보통 많은 데이터들은 DB에 저장되어 있는 경우가 많습니다. 그러면 DB에 연결해서 분석 작업을 하려면 어떻게 해야 할까요?

시작하기 전에, Airflow에서 외부 DB(MySQL, PostgreSQL, ...)에 연결하기 위해 Connection ID를 적절히 생성해 줍니다. 

Connection ID는 CLI로 생성하는 방법이나 웹서버에서 생성하는 방법이 있습니다. 

자세한 내용은 [Airflow의 문서](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html)를 참고하여 생성합니다. 

그리고 이 Connection ID를 이용해서 DB에 연결해 봅시다. 

## DB와 관련된 Operator 이용하기

Airflow의 공식 문서에 따르면, Operator는 'Task가 실제로 수행하는 일'을 작성하는 것입니다. 

Operator는 워크플로우 내 하나의 작업을 의미합니다. 항상 그렇지는 않지만 Operator는 다른 operator와 리소스를 공유하지 않는다고 합니다. (atomic이라는 말로 표현하네요. 적은 양의 데이터를 공유할 때는 XComs를 사용한다고 합니다.)

DAG은 Operator가 올바른 순서로 실행하도록 합니다. 하지만 의존성에 관계 없이 operator는 일반적으로 독립적으로 실행됩니다.

Airflow는 기본적으로 BashOperator, PythonOperator, EmailOperator 등의 built-in operator를 지원하는데요. 필요한 경우, 원하는 Provider 패키지를 설치하면 다른 Operator도 쓸 수 있습니다. 예를 들어 PostgreSQL Provider를 설치하면 PostgresOperator를 사용할 수 있습니다. 

아래는 여러 Operator에 대한 문서입니다. 

* [PythonOperator](https://airflow.apache.org/docs/apache-airflow/stable/howto/operator/python.html)
* [MySQL Operator](https://airflow.apache.org/docs/apache-airflow-providers-mysql/stable/operators.html)
* [How to Guide for PostgresOperator](https://airflow.apache.org/docs/apache-airflow-providers-postgres/stable/operators/postgres_operator_howto_guide.html)
* [KubernetesPodOperator](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/operators.html)
* [Amazon AWS Operators](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/operators/index.html)

그러면 PostgreSQL에 대한 Connection을 등록했다고 가정하고, 이를 이용해서 쿼리를 수행하는 Task를 만들어 보겠습니다. 이 예제에서는 PostgresOperator를 이용합니다. 

```python
from airflow.providers.postgres.operators.postgres import PostgresOperator

task_1 = PostgresOperator(
    task_id='run_query',
    postgres_conn_id='postgres_test',
    sql="SELECT * FROM table_name LIMIT 10;",
    dag=dag
)
```

이 Task는 `postgres_test` Connection 정보를 이용해서 DB에 연결한 뒤, `sql`에 지정한 SQL 문을 실행합니다. 

## Hooks 이용해서 쿼리하고 결과를 가공하기

DB에서 쿼리만 수행하는 경우라면 위의 예제가 충분하지만, 중간에 쿼리 결과를 가공해야 할 때가 있습니다. 이 때 Hook을 이용하면 DB에 연결할 수 있고, 여러 기능을 이용할 수 있습니다. 

Hooks는 Hive, S3, MySQL, Postgres, HDFS, Pig와 같은 외부 플랫폼과 데이터베이스에 대한 인터페이스입니다. 

자세한 설명은 [다음 문서](https://airflow.apache.org/docs/apache-airflow/stable/concepts.html#hooks)를 참고하세요. 

아래 설명은 링크한 문서의 내용을 한글로 옮겨서 작성했습니다. 

* Hook은 공용 인터페이스를 구현하며, Operator에 대한 building block으로 동작함
* `airflow.models.connection.Connection` 모델을 호스트 이름과 인증 정보를 얻는데 사용
* Hook은 파이프라인 바깥의 메타데이터 DB에 있는 인증 코드와 정보를 가지고 있음
* Hook은 Python 스크립트, Airflow PythonOperator, 그리고 iPython이나 Jupyter Notebook과 같은 대화형 환경에서 사용하기에 유용함

추가로 다음 문서를 같이 읽어보면 좋을 것 같네요. 

* [Hooks의 리스트](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/hooks/index.html)
* [airflow.hooks.dbapi](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/hooks/dbapi/index.html): DB와 관련된 Hooks 기본 구조

Connection이 등록되어 있다고 가정하고, Hook을 이용해서 DB에서 쿼리하는 예제를 만들어보면 다음과 같습니다. 

```python
from airflow.operators.python_operator import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook

def task_test_query():
    hook = PostgresHook(postgres_conn_id='postgres_test')
    # hook = PostgresHook('postgres_test')

    rows = hook.get_records("SELECT * FROM table_name LIMIT 10;")

    for row in rows:
        print(row)

task_2 = PythonOperator(
    task_id='run_query_with_python',
    python_callable=task_test_query,
    dag=dag
)
```

`postgres_test`에 지정된 Connection을 이용해서 DB에 연결하고, 쿼리를 수행한 결과를 `rows` 변수에 저장합니다. 그 다음으로는 쿼리 수행 결과를 돌면서 row를 출력합니다. 

# S3에서 파일을 가져와서 분석하기: S3Hook

먼저 AWS의 Connection ID를 설정합니다. 설정 방법은 [이 문서](http://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/connections/aws.html)를 참고하세요.

아래 예제는 특정한 버킷에서 Object 목록을 가져와서 출력하는 작업을 수행합니다. DAG 설정은 생략하였습니다. 

```python
from airflow.operators.python_operator import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

def task_s3_log_load():
    hook = S3Hook(aws_conn_id='aws_default')

    # Get list of objects on a bucket
    keys = hook.list_keys('bucket_name')

    for key in keys:
        print(key)

        obj = hook.get_key(key, 'bucket_name')

        print(obj.bucket_name, obj.key)


task_1 = PythonOperator(
    task_id='s3_analysis',
    python_callable=task_s3_log_load,
    dag=dag
)
```

# 마무리하며

데이터 분석 워크플로우를 처음부터 만들어 보는 이야기를 2회에 걸쳐서 진행했습니다. 개인적인 느낌을 이야기 해 보면, 원하는 기능을 구현할 때 문서를 참고해야 하는데요. 문서로는 충분하지 않아서 소스 레벨이나 GitHub에 올라온 이슈를 확인해야 하는 경우가 종종 있었습니다. 특히 기본적인 개념들이 간단하게 설명되어 있어서, 상세한 기능을 살펴보려면 Python API Reference 문서까지 찾아봐야 하는 경우가 있었습니다. 

예를 들면, BashOperator에 대한 설명은 다음 문서에 간단하게 설명되어 있습니다. 

* [Using Operators / BashOperator](http://airflow.apache.org/docs/apache-airflow/stable/howto/operator/bash.html)

하지만 Bashoperator에서 제공하는 상세한 기능들을 확인하려면 Python API Reference를 찾아야 합니다. 

* [Python API Reference / airflow.operators / airflow.operators.bash](http://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/operators/bash/index.html#airflow.operators.bash.BashOperator)

그리고 Operator들이 원래 지원하는 기능들은 BashOperator의 base class인 BaseOperator 클래스에 대한 설명을 찾아야 합니다. 

* [Python API Reference / airflow.models](http://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/index.html#airflow.models.BaseOperator)

만약에 프로덕션 레벨에서 이러한 기능들을 쓰게 된다면, 처음에는 이런 상황 때문에 삽질을 할 가능성이 있을 것 같아요. 

그리고 최신 버전인 2.x 버전과 1.x 버전 간 차이가 있기 때문에, 이 부분도 고려해야 합니다. 예를 들면 로깅 관련 설정들이 1.10.x 버전에서는 `[core]` 섹션에 있는데, 2.x 버전에서는 `[logging]` 섹션에 있습니다. 

하지만 Airflow 자체는 정말 좋은 툴이라고 생각합니다. DAG에서 Task를 설정하고, Task 간 실행 순서를 설정하는 부분을 쉽게 구현할 수 있는 것은 장점이라 생각합니다. 또한 지원하는 Operator들도 많아서, 여러가지 환경에 대응해서 쉽게 Task와 DAG을 구성할 수 있는 것도 장점입니다.

이후에도 가끔씩 Airflow를 사용해 보면서 이것저것 해 볼 예정입니다. 하지만 다음 달에는 개발과 관련 없는 주제로 글을 써 보려고 합니다. 

읽어주셔서 감사합니다. 