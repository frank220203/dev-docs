# 빅쿼리 slot 사용량 분석
## 프로젝트 시나리오
1. 평소 7000개의 slot을 예약
2. 약 400 개의 서비스 운영
2. 매일 raw/daily data 가공, 적재
3. 매주 월요일 weekly data 가공, 적재 (KST 기준)
4. 매달 1일 monthly data 가공, 적재 (KST 기준)
![img](./images/airflow-jobs.png)
5. 모든 스케줄은 작업 완료 후, GCP Buckets에 작업 완료 '.end' 파일 생성
5. 모든 스케줄은 GCP Composer를 통해 Airflow로 관리
6. 특정 날에 job이 몰리면 지연 발생
7. 지연이 발생할 경우 slot을 12000개까지 증설하여 지연 해소
8. 그런데 어느 순간부터 slot을 12000개까지 증설하지 않아도 지연된 job 해소 가능

## 개요
- 분석 날짜는 UTC 기준 / Airflow Schedule Time 기준 (raw : -1H, daily : -1D, weekly : 1W, monthly : -1M)
- raw : 당일 작업이 당일 00시부터 돌고, 당일 종료되어야 정상 (15시에 daily 작업이 돌기 때문에 최대 지연 허용 가능 시간 약 14시간)
- daily : 당일 작업이 당일 15시부터 돌고, 당일 종료되어야 정상 (평균 작업시간이 약 4시간. 당일 종료가 목표라면 최대 지연 허용 가능 시간 약 4시간)
- weekly : 일요일 23시부터 돌고, 익일 종료되어야 정상
- monthly : 당월 마지막 날 22시부터 돌고, 익월 1일에 종료되어야 정상

## 지연 이력 및 슬롯 증설 내역
- 2024년 09월 30일 (월) : raw, daily, weekly Job 맞물리며 지연 발생
    - 08시 weekly 작업 완료(09/22_23)
    - 08시 raw 작업 시작(09/29_23)
    - 23시 daily 작업 시작(09/29_15)
    - 23시 monthly 작업 시작(08/31_22)
- 2024년 10월 1일 (화) : **당일 daily 미수행(09/30_15)**
    - 16시 daily 작업 완료(09/29_15)
    - 18시 monthly 작업 완료(09/30_22)
    - 22시 raw 작업 완료(09/30_22)
    - 22시 raw 작업 시작(09/30_23)
- 2024년 10월 2일 (수) : **당일 daily 미수행(10/01_15)**
    - **03시 slot : 7000 -> 12000**
    - 09시 daily 작업 시작(09/30_15)
    - 13시 daily 작업 완료(09/30_15)
    - 18시 raw 작업 완료(10/01_22)
    - 18시 raw 작업 시작(10/01_23)
- 2024년 10월 3일 (목) : **daily Job 지연 해소**
    - 02시 daily 작업 시작(10/01_15)
    - 06시 daily 작업 완료(10/01_15)
    - 10시 raw 작업 완료(10/02_22)
    - 10시 raw 작업 시작(10/02_23)
    - 19시 daily 작업 시작(10/02_15)
    - 23시 daily 작업 완료(10/02_15)
    - **23시 slot : 12000 -> 7000**
- 2024년 10월 4일 (금) : **raw Job 지연 8시간**
    - 08시 raw 작업 완료(10/03_22)
- 2024년 10월 7일 (월) : raw, daily, weekly Job 맞물리며 지연 발생
    - **03시 slot : 7000 -> 12000**
    - 10시 weekly 작업 완료(09/29_23)
    - 15시 raw 작업 시작(10/06_23)
    - 23시 daily 작업 시작(10/06_15)
- 2024년 10월 8일 (화) : **daily Job 지연 해소**
    - 08시 daily 작업 완료(10/06_15)
    - 09시 raw 작업 완료(10/07_22)
    - 09시 raw 작업 시작(10/07_23)
    - 19시 daily 작업 시작(10/07_15)
    - 23시 daily 작업 완료(10/07_15)
    - **23시 slot : 12000 -> 7000**
- 2024년 10월 9일 (수) : **raw Job 지연 6시간**
    - 6시 raw 작업 완료(10/08_22)

## 분석 포인트
### 1. 전체 Job 비교 분석
```sql
 -- Composer 별 slot 사용량 및 job 개수 조회 Query
SELECT
FORMAT_TIMESTAMP('%Y%m%d', creation_time) as cyymmdd,
user_email,
SUM(total_slot_ms) as ttl_slot,
COUNT(job_id) as cnt_job
FROM
`bigdata-project-id`.`region-asia-northeast3`.INFORMATION_SCHEMA.JOBS
WHERE
 -- 당시 프로젝트는 1일치의 job log 당, 50G 조회 비용
 -- 비용 최소화를 위해 주요 날짜 선별 조회
creation_time BETWEEN '2024-09-30' AND '2024-11-07'
 -- 쿼리 실행이 완료된 경우
AND state = 'DONE'
AND user_email like "%composer-email%"
GROUP BY
cyymmdd,
user_email
ORDER BY cyymmdd, user_email
```
```sql
 -- 10-01 총 작업시간 Query
SELECT
FORMAT_TIMESTAMP('%Y%m%d', creation_time) as cyymmdd,
user_email,
SUM(TIMESTAMP_DIFF(end_time, creation_time, SECOND)) as duration_time
FROM
`bigdata-project-id`.`region-asia-northeast3`.INFORMATION_SCHEMA.JOBS
WHERE
 -- 10/01과 11/01 조회하여 비교
creation_time BETWEEN '2024-10-01' AND '2024-10-02'
 -- 쿼리 실행이 완료된 경우
AND state = 'DONE'
AND user_email like "%composer-email%"
GROUP BY
cyymmdd,
user_email
ORDER BY cyymmdd, user_email
```

| 비교 대상 | 09-30(월) | 10-01(화) | 10-31(수) | 11-01(금) |
|----------|-----------|-----------|-----------|-----------|
| 수행 Job | raw, daily, weekly, monthly | raw, daily, monthly | raw, daily, weekly(신규 서비스 추가로 catchup), monthly | raw, daily, monthly |
| slot 사용시간(ms) | 981,368,797,007 | 786,675,307,380 | 1,039,575,439,899 | 978,438,638,776 |
| raw Job 개수 | 170338 | 83496 | 210518 | 193327 |
|daily Job 개수 | 24156 | 22779 | 37289 | 15516 |
| weekly/monthly Job 개수 | 8667 | 5346 | 779 | 6601 |
| 총 Job 개수 | 203161 | **110612** | 248586 | **215444** |
| 총 작업시간(s) | 5,155,745 | **4,019,495** | 4,750,424 | **4,453,354** |

해석 : 10-01과 11-01의 Job 개수가 2배 차이 났지만, 작업시간이 2배의 차이를 보이고 있지 않다. -> **10-01의 Job은 하나 하나가 작업시간이 길었을 것이라 예상**

### 2. 3대 주요 서비스 비교
| 비교 대상 | 10-01(화) | 11-01(금) | 10-01(화) | 11-01(금) | 10-01(화) | 11-01(금) |
|----------|-----------|-----------|-----------|-----------|----------|-----------|
| 수행 Job | raw (Service1) || monthly (service2) || monthly (service3) ||
| 수행 Task | Task1 || Task2 || Task3 ||
| slot 사용시간(ms) | 3,105,566,508 | 2,800,556,134 | 1,202,071,746 | 1,143,163,451 | 185,720,339 | 167,981,952 |
| 처리 데이터(byte) | 4,323,671,548,043 | 4,475,100,030,539 | 3,476,638,883,313 | 3,699,231,479,309 | 860,729,278,179 | 955,879,372,595 |
| 총 작업시간(s) | 15,843 | 14,455 | **11,927** | **4,907** | **2,006** | **452** |

해석 : Service1은 작업시간이 큰 차이가 나지 않지만, 그 외 두 개의 Job은 작업시간이 2배 이상 차이가 발생했다. -> **10-01의 Job은 slot 경쟁으로 바로 실행되지 못 하고 대기 시간이 더 길었음을 어느정도 유추할 수 있다.**

### 3. 10월 1일 - 평균 작업시간이 더 길다 (raw 작업 제외)
```sql
 -- 10-01 작업시간 평균 Query
SELECT
FORMAT_TIMESTAMP('%Y%m%d', creation_time) as cyymmdd,
user_email,
AVG(TIMESTAMP_DIFF(end_time, creation_time, SECOND)) as duration_time
FROM
`bigdata-project-id`.`region-asia-northeast3`.INFORMATION_SCHEMA.JOBS
WHERE
 -- 10/01과 11/01 조회하여 비교
creation_time BETWEEN '2024-10-01' AND '2024-10-02'
 -- 쿼리 실행이 완료된 경우
AND state = 'DONE'
AND user_email like "%composer-email%"
GROUP BY
cyymmdd,
user_email
ORDER BY cyymmdd, user_email
```
| cyymmdd | user_email | duration_time(s) |
|---------|------------|----------|
| 20241001 | bigdata-composer-raw | 26.02394 |
| 20241001 | bigdata-composer-daily | **34.71118** |
| 20241001 | bigdata-composer-weekly-monthly | **238.1673** |
| 20241101 | bigdata-composer-raw | 18.58485 |
| 20241101 | bigdata-composer-daily | **6.002256** |
| 20241101 | bigdata-composer-weekly-monthly | **111.1998** |

### 4. 10월 1일 - 평균 작업시간 238(s) 보다 오래 걸린 job이 더 많다 (raw 작업 제외)
```sql
 -- 10-01 작업시간 평균 Query
SELECT
FORMAT_TIMESTAMP('%Y%m%d', creation_time) as cyymmdd,
user_email,
COUNT(TIMESTAMP_DIFF(end_time, creation_time, SECOND)) as count_up_238
FROM
`bigdata-project-id`.`region-asia-northeast3`.INFORMATION_SCHEMA.JOBS
WHERE
 -- 10/01과 11/01 조회하여 비교
creation_time BETWEEN '2024-10-01' AND '2024-10-02'
 -- 쿼리 실행이 완료된 경우
AND state = 'DONE'
AND user_email like "%composer-email%"
 -- 10/01 Weekly, Monthly 평균 작업 시간 이상의 Job 조회
AND TIMESTAMP_DIFF(end_time, start_time, SECOND) > 238
GROUP BY
cyymmdd,
user_email
ORDER BY cyymmdd, user_email
```
| cyymmdd | user_email | count_up_238 |
|---------|------------|----------|
| 20241001 | bigdata-composer-raw | 1223 |
| 20241001 | bigdata-composer-daily | **456** |
| 20241001 | bigdata-composer-weekly-monthly | **403** |
| 20241101 | bigdata-composer-raw | 2166 |
| 20241101 | bigdata-composer-daily | **46** |
| 20241101 | bigdata-composer-weekly-monthly | **282** |

## 변경점
### - 기존
1. weekly, monthly **개별** 서비스마다 GCP Bucket의 raw, daily 작업의 '.end' 파일 체크 후, job 수행 -> **동시 작업 증가로 slot 경쟁 발생**
```python
# Airflow - Weekly DAG Workflow (Python Code)

# 서비스 명 가져오기
service_name = params['service_name'].uppser()
# Sensor Operator에 넣어줄 리스트 선언
gcs_objects = []

# Month : DAG 실행 날짜의 ds 기반으로 전월 가져와서 path 생성
# monthly_date = str(kwargs['tomorrow_ds_nodash'])
# day_end = calender.monthrange(int(monthly_date[:4], int(monthly_date[4:6])))[1]

# 7일치 '.end' 파일 확인을 위한 path 생성
day_end = 7

for i in range(1, day_end+1):
    # macros.ds_add()로 i일씩 더하는 문자열 생성
    target_date = 'macros.ds_format(macros.ds_add(ds, days={}), "%Y-%m-%d", "%Y%m%d")'.format(i)
    # airflow jinja 탬플릿( {{...}} ) 형식의 문자열로 변환
    target_date = '{{ %s }}'%target_date
    # 체크해야 할 path 추가
    gcs_objects.append('success/{target_date}/{service_name}.end'.format(target_date=target_date, service_name=service_name))

# GoogleCloudStorageMultiObjectSensor : Airflow에서 Google Cloud Storage(GCS)의 특정 버킷에 여러 개의 객체가 존재하는지 확인하는 Sensor
check_end_file = GoogleCloudStorageMultiObjectSensor(
    task_id='check_end_file',
    bucket='bucketname',
    objects=gcs_objects,
    reschedule_interval=1200, # 20분 간격으로 확인
    timeout=86400,      # 24시간 초과하면 fail
    mode='reschedule', # 상태 변화 감지해도 한 번 더 확인
    soft_fail=False # task 실패시 DAG 실패
)
```
2. 같은 시간대에 raw, daily, weekly, monthly 작업이 slot 경쟁을 하며 수행되고 있음
```sql
 -- 시간별 작업 이력 Query
SELECT
FORMAT_TIMESTAMP('%Y%m%d-%H', creation_time) as cyymmddH,
job_id
FROM
`bigdata-project-id`.`region-asia-northeast3`.INFORMATION_SCHEMA.JOBS
WHERE
 -- 10/01과 11/01 조회하여 비교
creation_time BETWEEN '2024-10-01' AND '2024-10-02'
 -- 쿼리 실행이 완료된 경우
AND state = 'DONE'
 -- daily 작업이 완료된 후 weekly, monthly 작업이 수행되는 것을 확인하기 위해 3개만 조회
AND user_email IN ("bigdata-composer-email-daily.iam.gserviceaccount.com", "bigdata-composer-email-weekly.iam.gserviceaccount.com", "bigdata-composer-email-monthly.iam.gserviceaccount.com")
 -- 수 많은 작업 task중 실질적으로 빅쿼리 slot을 사용하는 airflow_ 명칭이 붙은 task 조회
AND job_id like "%airflow_%"
ORDER BY cyymmddH
```
| cyymmddH | job_id |
|---------|------------|
| 20241001-00 | airflow_month_v01_app1_2024_08_31T22_00_00_00_00_hashcode |
| **20241001-00** | airflow_**month**_v01_app2_2024_08_31T22_00_00_00_00_hashcode |
| **20241001-00** | airflow_**day**_v02_app3_2024_08_31T15_40_00_00_00_hashcode |
| 20241001-00 | airflow_day_v02_app4_2024_08_31T15_40_00_00_00_hashcode |
| **20241001-00** | airflow_**weekly**_v01_app1_2024_08_31T22_00_00_00_00_hashcode |
| 20241001-00 | airflow_weekly_v01_app2_2024_08_31T22_00_00_00_00_hashcode |
| 20241001-01 | airflow_month_v01_app3_2024_08_31T22_00_00_00_00_hashcode |
| **20241001-01** | airflow_**month**_v01_app4_2024_08_31T22_00_00_00_00_hashcode |
| **20241001-01** | airflow_**day**_v02_app1_2024_08_31T15_40_00_00_00_hashcode |
| 20241001-01 | airflow_day_v02_app2_2024_08_31T15_40_00_00_00_hashcode |
| **20241001-01** | airflow_**weekly**_v01_app4_2024_08_31T22_00_00_00_00_hashcode |
| 20241001-01 | airflow_weekly_v01_app9_2024_08_31T22_00_00_00_00_hashcode |

### - weekly/monthly Task 수행방식 변경
1. daily 작업 모니터링 DAG으로 GCP Bucket에 **모든** 서비스의 daily 작업 '.end' 파일 체크 후, **'daily.end'** 파일 생성
```python
# Airflow - Daily Monitoring DAG Workflow (Python Code)

def check_svc_done(**kwargs):
    # 서비스 리스트 가져오기
    svc_list = Variable.get("svc_list", deserialize_json=True)
    target_num = len(svc_list)
    # xcom에 target_num 값 저장
    return target_num

# PythonOperator : Airflow에서 파이썬 코드를 실행하는 Operator
check_svc_done = PythonOperator(
    task_id='check_svc_done',
    python_callable=check_svc_done
)

# BashSensor : Airflow에서 Bash 명령어를 실행하여 특정 조건이 만족될 때까지 대기하는 Sensor
check_fin_bash = BashSensor(
    task_id='check_fin_bash',
    bash_command="""
    #!/bin/sh

    # Google Cloud Storage(GCS)의 특정 버킷의 '.end' 파일 count
    fin_num=$(gsutil ls gs://success/{{ execution_date + macros.timedelta(days=(1))).strftime("%Y%m%d") }}/ | grep .end |wc -l)

    # check_svc_done task에서 반환한 서비스 개수랑 비교
    if [ "$fin_num" == {{ ti.xcom_pull(task_ids='check_svc_done') }} ]
    then
        exit 0
    else
        exit 1
    fi
    """,
    output_encoding='utf-8',
    poke_interval=30
)

# BashOperator : Airflow에서 Bash 명령어를 실행하는 Operator
check_fin_bash = BashOperator(
    task_id='create_daily_end',
    bash_command="""
    # daily.end 생성 후 Google Cloud Storage(GCS)의 특정 버킷의 'daily.end' 파일 복사
    touch daily.end && gsutil cp daily.end gs://success/{{ (execution_date + macros.timedelta(days=(1))).strftime("%Y%m%d") }}/daily.end
    """
)
```

2. weekly, monthly 작업은 GCP Bucket에 **모든** 서비스의 raw, daily 작업 '.end' 파일 체크 후, job 수행 -> **raw, daily 작업이 끝나고 weekly, monthly 작업이 시작되는 효과**
```python
# Airflow - Weekly/Monthly DAG Workflow (Python Code)

service_name = params['service_name'].uppser()
gcs_objects = []

# Month : DAG 실행 날짜의 ds 기반으로 전월 가져와서 path 생성
# monthly_date = str(kwargs['tomorrow_ds_nodash'])
# day_end = calender.monthrange(int(monthly_date[:4], int(monthly_date[4:6])))[1]
# for i in range(1, day_end+1):

# Week : 7일치 '.end' 파일 확인을 위한 path 생성
day_end = 7

for i in range(1, day_end+1):
    target_date = 'macros.ds_format(macros.ds_add(ds, days={}), "%Y-%m-%d", "%Y%m%d")'.format(i)
    target_date = '{{ %s }}'%target_date
    gcs_objects.append('success/{target_date}/{service_name}.end'.format(target_date=target_date, service_name=service_name))

    # 모든 daily 작업 완료 확인 로직 추가
    if i == day_end:
        gcs_objects.append('success/{target_date}/{service_name}.end'.format(target_date=target_date, service_name='daily'))

check_end_file = GoogleCloudStorageMultiObjectSensor(
    task_id='check_end_file',
    bucket='bucketname',
    objects=gcs_objects,
    reschedule_interval=1200,
    timeout=86400,
    mode='reschedule',
    soft_fail=False
)
```

| cyymmddH | job_id |
|---------|------------|
| 20241101-00 | airflow_day_v02_app4_2024_10_30T15_40_00_00_00_hashcode |
| 20241101-00 | airflow_day_v02_app2_2024_10_30T15_40_00_00_00_hashcode |
| **20241001-00** | airflow_**day**_v02_app3_2024_10_30T15_40_00_00_00_hashcode |
| **20241001-01** | airflow_**month**_v01_app4_2024_08_31T22_00_00_00_00_hashcode |
| 20241001-01 | airflow_month_v01_app1_2024_08_31T22_00_00_00_00_hashcode |
| 20241001-01 | airflow_month_v01_app3_2024_08_31T22_00_00_00_00_hashcode |

## 결론
### 1. raw, daily, weekly, monthly Job이 무작위로 수행되면서 slot 경쟁 -> 과도한 지연 발생
### 2. 24시간내에 작업이 완료되어야 하는 daily Job이 당일에 수행되지 않는 상황 발생 -> slot 12000 증설 주요 원인
### 3. weekly/monthly DAG의 Task 수행 방식 조정 (daily Job 완료 후 수행) -> 어느 정도 지연이 발생해도 7000 slot 유지 가능
### 4. 다만 여전히 weekly/monthly Job이 연달아, 혹은 동시 수행된다면 과도한 지연 발생으로 slot 증설 가능성 존재
### 5. 12-01(일), 12-02(월) weekly, monthly Job이 연달아 수행되는 case로 모니터링 후, 재검토