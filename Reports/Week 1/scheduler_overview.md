## Overview

## 1. Point Based 스케줄링

```python
dag = DAG(
    '1_point_based_dag',
    schedule="0 0 * * *",  # 매일 자정에 실행
    start_date=datetime(2023, 1, 1)
)
```

- 5개의 필드를 사용: 분(0-59), 시간(0-23), 일(1-31), 월(1-12), 요일(0-6)
- 특정 시점(point in time)에 실행되도록 정확히 지정 가능
- 복잡한 스케줄링 패턴 표현 가능
- 시간대(timezone) 설정을 통해 국제적 운영에 적합

## 2. Interval Based 스케줄링

- 두 실행시간 사이의 간격(interval)을 정의하여 주기적으로 실행

```python
dag = DAG(
    'interval_based_dag',
    schedule=timedelta(hours=6),  # 6시간마다 실행
    start_date=datetime(2023, 1, 1)
)
```

- interval을 직관적으로 설정 가능 (시간, 일, 주 단위)
- 첫 번째 실행은 start_date 이후 interval이 지난 시점
- 다음 실행은 항상 이전 실행의 예정 시간 + interval
- 실행이 밀려도 간격은 일정하게 유지됨(catch-up 실행)

## 3. Timetable 기반 스케줄링

- 보다 유연한 스케줄링 로직을 구현하기 위한 방식

```python
class CustomTimetable(Timetable):

#scheduling logic here

dag = DAG(
    'timetable_based_dag',
    timetable=CustomBusinessHoursTimetable(),
    start_date=datetime(2023, 1, 1)
)
```

- 복잡한 비즈니스 로직 기반 스케줄링 구현 가능
- 영업일만 실행, 공휴일 제외, 특정 이벤트 기반 실행 등 복잡한 패턴 지원
- 다음 실행 시간을 동적으로 계산 가능
- 커스텀 스케줄링 알고리즘을 구현하여 재사용 가능

### Timetable 클래스

Dag가 언제 실행되어야 하는지 결정하는 추상화된 인터페이스

- 실행 시간 계산: 이전 실행 정보를 바탕으로 다음 실행 시간을 계산합니다.
- 데이터 간격 정의: 각 DAG 실행이 처리해야 할 데이터의 시간 간격을 결정합니다.
- 스케줄링 추상화: 다양한 스케줄링 패턴을 일관된 인터페이스로 제공합니다.

```python
class Timetable(Protocol):
    @property
    def summary(self) -> str:


    def infer_manual_data_interval(self, run_after: DateTime) -> DataInterval:
        ...

    def next_dagrun_info(
        self,
        last_automated_data_interval: Optional[DataInterval],
        restriction: TimeRestriction,
    ) -> Optional[DagRunInfo]:
```

## 4. DAG Aware Scheduling

- DAG Aware 스케줄링은 다른 DAG의 실행 상태나 결과에 따라 DAG를 실행하는 방식
- TriggerDagRunOperator와 ExternalTaskSensor를 사용하여 구현합니다.

```python
wait_for_upstream = ExternalTaskSensor(
    task_id='wait_for_upstream',
    external_dag_id='upstream_dag',
    external_task_id='final_task',
    dag=dag
)

trigger_downstream = TriggerDagRunOperator(
    task_id='trigger_downstream',
    trigger_dag_id='downstream_dag',
    dag=dag
)
```

- DAG 간 의존성 정의 가능
- 한 DAG의 완료 후 다른 DAG를 실행하는 워크플로우 체인 구성
- 특정 태스크의 성공/실패에 따른 조건부 실행
- 복잡한 워크플로우 오케스트레이션 가능

## 5. Dataset Based Scheduling

데이터셋의 변경을 감지하여 DAG를 실행

```python
from airflow.datasets import Dataset

my_dataset = Dataset('s3://bucket/data.csv')

# 데이터셋을 생산하는 DAG
producer_dag = DAG(
    'producer_dag',
    schedule="0 0 * * *",
    start_date=datetime(2023, 1, 1),
    outlets=[my_dataset]
)

# 데이터셋에 의존하는 DAG
consumer_dag = DAG(
    'consumer_dag',
    schedule=[my_dataset],  # 데이터셋이 업데이트될 때 실행
    start_date=datetime(2023, 1, 1)
)
```

- 데이터 기반 워크플로우에 적합
- 실제 데이터 가용성에 따른 실행 보장
- 데이터 파이프라인 간 의존성 명확하게 표현
- 불필요한 실행 방지 및 리소스 효율성 향상

## 6. Event Based Scheduling

- 외부 이벤트(HTTP 요청, 메시지 큐 등)에 스케줄을 트리거하는 방식
- 외부 툴과 통합 용이
- 동적 파라미터 전달

```python
# REST API를 통한 트리거
curl -X POST 'http://airflow-webserver:8080/api/v1/dags/my_dag/dagRuns' \
  -H 'Content-Type: application/json' \
  --data '{"conf":{"key":"value"}}'
```

## 7. Conditional Scheduling

- DAG 또는 Task 레벨에서 조건부 로직을 적용

```python
def should_run(**context):
    # 실행 여부를 결정하는 로직
    return True  # or False

dag = DAG(
    'conditional_dag',
    schedule="0 0 * * *",
    start_date=datetime(2023, 1, 1)
)

with dag:
    BranchPythonOperator(
        task_id='branch_task',
        python_callable=should_run,
    )
```

- 동적 조건에 따른 실행 경로 결정
- ShortCircuitOperator로 조건부 실행 중단
- BranchPythonOperator로 조건부 분기 처리
- 불필요한 작업 스킵으로 리소스 절약

## 종합적 고려사항

## Reference

- [Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/index.html)
