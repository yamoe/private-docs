# Airflow

## 설치

### multipass (ubuntu 22.04 LTS) 설치

```bash
# 생성
multipass launch -c 1 -d 10G -m 2G -n airflow

# 접속
multipass launch -c 1 -d 10G -m 2G -n airflow


## ubuntu UTC 설정
# 확인
date

# utc 설정
sudo timedatectl set-timezone 'UTC'

# 확인
date
```

### python 3.10 설치

3.10.6 기본 설치로 pip3 만 설치

```bash

# python 3 버전 확인
python3 -V

# python 로 3 버전 사용하도록 추가
sudo ln -sf /usr/bin/python3 /usr/bin/python

# pip3 설치
sudo apt-get update
sudo apt-get -y install python3-pip

# pip3 버전 확인(22.0.2)
pip -V
```

### airflow 설치

```bash
export AIRFLOW_HOME=~/airflow

AIRFLOW_VERSION=2.6.3
PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"

CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

# 설치
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# .bashrc PATH 추가
export PATH=$PATH:/home/ubuntu/.local/bin
```

### airflow standalone 실행 (SQLite 기본 사용)

```bash
# 실행
airflow standalone
```

web 접속: [http://loocalhost:8080](http://192.168.144.240:8080/)

airflow 실행 시 console log 의 접속 계정 확인.

```text
standalone | Airflow is ready
standalone | Login with username: admin  password: Nn4d2Yz6usYu8dFm
```

## MySQL 사용 (기본 SQLite)

### MySQL 8 설치

```bash
# mysql 8 설치
sudo apt-get -y install mysql-server-8.0
sudo systemctl status mysql

# secure 설정
sudo mysql_secure_installation

# root 암호 설정
sudo mysql
alter user 'root'@'localhost' identified with mysql_native_password by '123456789';
flush privileges;

# UTC 설정
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
default_time_zone='+00:00'

sudo systemctl restart mysql
mysql -u root -p123456789 -e "select now();"

```

### Airflow MySQL 설정

```bash
# mysql root 접속
mysql -u root -p123456789
```

```sql

-- 편의상 암호 규칙 체크 제거
UNINSTALL COMPONENT 'file://component_validate_password';

-- DB 생성
CREATE DATABASE airflow_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'airflow_user' IDENTIFIED BY '123456789';
GRANT ALL PRIVILEGES ON airflow_db.* TO 'airflow_user';
flush privileges;
show databases;
```

```bash
# database 주소 확인
airflow config get-value database sql_alchemy_conn

# 변경
# vi ~/airflow/airflow.cfg
sql_alchemy_conn = mysql+mysqldb://airflow_user:123456789@localhost:3306/airflow_db

# mysqlclient 설치
pip install cryptography psycopg2-binary boto3 botocore
sudo apt -y install libmysqlclient-dev
sudo apt -y install pkg-config
pip install mysqlclient

# db init
airflow db init

# table 생성 확인
mysql -u airflow_user -p123456789 airflow_db -e "show tables;"

# airflow 재시작
CTRL + C
airflow standalone
```

## 간단한 예제

### 테스트용 설정 변경 ~/airflow/airflow.cfg

```cfg
load_example = False
min_file_process_interval = 60
dag_dir_list_interval = 30
executor = LocalExecutor
parallelism = 64
dag_concurrency = 32
```


### ~/airflow/dags/sample.py

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.utils.dates import days_ago
from datetime import timedelta

default_args = {
        'owner': 'airflow',
        'catchup': False,
        'execution_timeout': timedelta(hours=6),
        'depends_on_past': False,
    }

dag = DAG(
    'sample',
    default_args = default_args,
    description = "sample description",
    schedule_interval = "@daily",
    start_date = days_ago(3),
    tags = ['daily'],
    max_active_runs=3,
    concurrency=1
)

sample_a = BashOperator(
    task_id='sample_a',
    bash_command='echo hello',
    dag=dag)

sample_b = BashOperator(
    task_id='sample_b',
    bash_command='echo hello',
    dag=dag)
    
sample_a << sample_b
```
