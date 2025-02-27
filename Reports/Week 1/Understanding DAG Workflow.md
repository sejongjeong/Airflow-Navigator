### DAG Parsing Optimizations

https://www.youtube.com/watch?v=W0X7NEj7mRk&embeds_referring_euri=https%3A%2F%2Flilys.ai%2F&embeds_referring_origin=https%3A%2F%2Flilys.ai&source_ve_path=OTY3MTQ

## Summary
DAG parsing optimizations in Apache Airflow, focusing on improving the speed and efficiency of DAGprocessing. The central theme revolves around identifying and mitigating performance bottlenecks during DAG parsingto enhance Airflow's overall performance. The speaker explains how isolating DAG processes leads to repeated imports, causing delays, and introduces a solution to pre-import common Airflow modules in the main thread to reduce parsing time. Furthermore, the video addresses the issue of slow variable lookups, particularly with external services like AWS Secret Manager, and presents an optional caching mechanism to alleviate this problem, despite it being a discouraged practice. Ultimately, the speaker emphasizes that anyone can contribute to improving open-source projects like Airflow by investigating performance issues and implementing optimizations, even without being an expert.
Key Term
DAG parsing: This refers to the process of reading, analyzing, and understanding the structure and components...
1. 🛠️ DAG Parsing Optimization Challenges
The speaker discusses optimizing how Airflow parses DAGs in Core Airflow. 
A performance issue was noted where having 300 DAGs in a single file performed well, but separate files for each DAGcaused high CPU usage and scheduler overwhelm. 
The speaker initially lacked knowledge of Airflow's internal workings, which led to deeper investigation and discovery of optimization strategies. 
Using Breeze for local testing, the speaker confirmed the problem, observing high CPU usage with increasing numbers of DAGs. 

2. ⏱️ Measuring DAG Parsing Performance
The speaker measured the time it takes to parse every single DAG, revealing a log feature available when a specific configuration is activated. 
All DAGs were found to take approximately 300 milliseconds to parse, despite being parsed by two processes every 30 seconds. 
With over 200 DAGs, parsing within the allotted 30 seconds is impossible, leading to overlaps and high CPU usage. 
The speaker utilized the Airflow config page for parameter insights by searching for relevant terms like "path." 

3. ⚙️ DAG Parsing Solutions and Optimizations
Airflow struggles with file parsing speed, prompting the need for solutions to mitigate performance issues. 
Increasing the parsing interval can provide more time for Airflow to process all DAGs. 
Utilizing more processes can enhance parsing capabilities but requires a robust server setup to manage the load. 
Running the DAG processor separately from the scheduler alleviates performance issues, enabling the scheduler to function optimally. 
Proper configuration involves adjusting parameters to prevent the automatic start of the DAG processor, which must then be initiated manually. 

4. ⚙️ DAG Parsing Performance Insights
Airflow's DAG parsing involves executing the Python file, which can affect speed significantly. 
Custom code can be added to time the execution of each line to identify performance bottlenecks, particularly those taking over 300 milliseconds. 
Although variables are often considered a performance issue, they were not significantly slowing down the process in this instance. 
The primary performance hit was due to imports, which took over 200 milliseconds of the total 300 milliseconds, revealing inefficiencies in how Python handles them. 
Initial confusion about the speed of Python and Airflow led to further investigation into the parsing and import efficiency processes. 

5. 🚀 DAG Parsing Optimization Techniques
In Airflow, DAGs consist of user code, and parsing them requires isolation to prevent crashes in the main processes. 
A forked process is used to safely parse the DAG, which only retains the needed imports in cache before being destroyed. 
Recognizing that repeated imports slow down performance, optimizing solutions include pre-importing frequently used Airflow libraries. 
Initial tests showed a 22% speed improvement from pre-importing, which later increased to 59% after updating outdated import paths. 
Performance gains ranged from 25% to 70% with diverse DAGexamples, heavily contingent on the complexity and type of imports. 
Despite optimizations, issues arise with variables and external service calls, leading to slowdowns, especially when using AWS Secrets Manager. 
Implementing a cache for variables was considered as a solution, but challenges like cache invalidation and inconsistency among processes were noted. 
The community supported the optional cache, utilizing Python's Multiprocessing Manager for shared dictionary management, though configurations can lead to unexpected behavior. 
Despite its potential for speed, the experimental feature should be employed cautiously, recognizing that significant improvements depend on the DAG's variable usage. 
Users can achieve substantial impact by understanding and investigating optimization opportunities, even if they’re not experts in the field. 
5.1. DAG Parsing in Airflow
In Airflow, when parsing a DAG (Directed Acyclic Graph), user code is isolated to prevent crashes in Airflow. 
A separate process is created for DAG parsing, which operates on a copy of the main process's memory. 
This process caches imports for potential reuse but discards all memory after the DAG parsing is complete. 
As a result, imports are never processed in the main thread, leading to longer processing times. 
5.2. DAG Parsing Optimizations Insights
Software optimization involves identifying repetitive tasks and finding efficient ways to perform them only once. 
One proposed solution was to preprocess imports in the main thread before forking, resulting in significant performance gains. 
Pre-importing common Airflow dependencies led to a 22% speed improvement in initial tests, though results were initially unexpected. 
Updating obsolete import paths significantly enhanced performance, achieving up to 59% faster execution time when passing DAGs. 
Testing with a diverse set of example DAGs revealed a performance improvement range from 25% to 70%, depending on DAGcomplexity and import dependencies. 
5.3. Optimizing DAG Performance with Caching
The use of variables in DAGs significantly slows performance, particularly when using distant databases or AWS Secret Manager, resulting in around 800 milliseconds per DAG. 
Despite documentation advising against variable usage, users still utilize them, leading to the consensus that a caching solution could help mitigate performance issues. 
Caching presents challenges, such as cache invalidation and inconsistent states across different processes, but it remains an efficient method for saving time and reducing costs associated with API calls. 
Discussions in the community supported the implementation of an optional cache, acknowledging it could inadvertently encourage bad practices. 
The new caching system is designed to be authentication-enabled by default. 
5.4. DAG Parsing with Python Multiprocessing
DAGs utilize Python Multiprocessing Manager, allowing processes to share a dictionary, but they write to their own memory rather than shared memory. 
The Multiprocessing Managermanages a dictionary by spawning a separate process, making dictionary calls appear seamless to the user. 
Changes in configuration can lead to inconsistencies, particularly with multiple schedulers, potentially causing the DAGto behave unpredictably. 
Writing a DAGwith extensive variable calls can yield significant performance improvements, although metrics should be approached cautiously. 
The feature is still experimental, and users should implement it with caution, especially regarding connections treated as secrets. 
5.5. Impact Through Investigation
Anyone can have an impact without being an expert by digging down into the code and metrics. 
Understanding what is happening in the system can lead to achieving cool outcomes. 
Time and motivation are essential for successful investigation and contribution. 
Not everyone can dedicate time like paid contributors can, making this challenge prevalent. 
With determination, it is possible to achieve significant results.

## Script
I'm going to talk about some of the work I did on optimizing the way Airflow parses DAGs in Core Airflow. I'm also going to quickly talk about how users can not make mistakes that make the DAG parsing longer, but this is not the core of the talk, of the talk about how I did faster. So just a quick slide about me. 

Okay. I'm French, as you could have guessed from my accent, and I'm Raphael Vandon. My pronouns are he and him, and I'm working at AWS. 

The logo should be right there. It's here on my screen. I don't know why it doesn't display here. 

I'm working at AWS since one year, and my job is to contribute to Airflow. full time, so I'm really grateful to Amazon to pay people to do that. So the story I'm gonna tell, it starts with an issue, a ticket that was assigned to me, and the ticket was titled, "Steadier Performance Issue." And someone was asking the question, like they have a DAG, they have like 300 DAGs. 

If they create them in one file, the performances are okay. If they create 300 DAGs from 300 files, like one file per DAG, then the scheduler is overwhelmed and the CPU is super high, they cannot do anything with it. And they're asking, why is it the case? It's a legit question, because in the end it's the same, like. 

DAGs are the same, and you could think that maybe doing loop and rolling, not creating the things in the loop, but doing the loop for the computer would make it faster, not the case. If you know how Airflow works internally, you might have guessed why it's the case. At the time, I was quite new on Airflow, so I didn't have a good grasp of what was happening inside, so I didn't really know what was happening, so I just took this ticket and started investigating. 

And thankfully, because if I had known, maybe I wouldn't have dug as deep, and maybe I wouldn't have discovered how to improve this stuff. So what I Started doing, I don't have the pictures. OK, OK, cool. 

I don't know what's happening between the screen and my screen. OK, so what I did is, since I didn't know much, I started from the beginning. The beginning is just trying to reproduce the problem and trying to look at it really from the outside. 

So I used Breeze, which is a way to run Airflow locally using Docker. You just type one command and everything is running, and you can put your DAGs in there and see what's happening. It runs in Docker, so I was just looking at Docker, looking at the CPU usage of the container, and I tried to just vary the number of DAGs to. 

Confirm there was indeed the fact that there were many DAGs that was creating the problems. It was not a threshold effect, for example. So I read like 200 DAGs. 

You can see that the CPU is pretty high. 100 files, you can see it's starting to breathe a bit, and 25 files, everything is fine. We have plenty of CPU to use to do other stuff.
So once I had this, I started digging and reading the code to understand what was happening. So the passing time, I could measure the passing time for every single DAG. There was actually a log. 

If you activate this configuration setting, the DAG parser is going to print the. time it takes to parse every single DAG. So it's very verbose, but very helpful when you debug. 

And I could see all the DAGs are pretty much the same, and I could see that it was taking 300 milliseconds per file. Now the DAGs, by default, are parsed every 30 seconds, and there are two processes doing that. So if you do the math, it means that if you have more than 200 DAGs, you cannot go through all of them in 30 seconds on two processes. 

You're going to have to, that's going to be some overlap, and the scheduler is going to fall behind, and of course the CPU is going to be high all the time. By the way, to find those Things, so I was reading the code, but I also went to the Airflow config page on the Airflow website, and I just did a search for "path," and I was looking at all the parameters, and this was a good way to see what was the parameters that were operating in this.
So now I was starting to understand what was happening. It was just that Airflow couldn't pass the files fast enough. And so even with this, if you don't know much, you can already propose solutions, not solutions to fix the problem, but solutions to mitigate it. 

We can act on the configuration we just found. One of the solutions is to just increase the parsing. Interval. 

If Airflow cannot go through all the DAGs in 30 seconds, just give it more time. Another solution is to have more processes, so if you give it more processes, you'll have more resources to parse up the DAGs. Of course, if you do that, there are drawbacks, like the first drawback in the fine processing interval is that your changes are going to take longer to appear. 

The drawback of having more processes is that you need a beefier server. If you just give it like 10 processes on a server with two cores, it's not going to do you too much. And then another solution is to run the DAG processor separately. 

The initial problem was the scheduler has a performance issue. If you do that, then the DAG processor has a performance issue, but the scheduler doesn't have any performance issues anymore, it can do its scheduler things. So the way you do that, when you set this parameter, it tells the scheduler that it should not start the DAG processor, and then you have to start it manually by running this copy.
So these are solutions that I can already give to the person who opened the ticket to tell them this is how you can get usable Airflow today, and now I can start looking at how I'm going to fix the problem. How can I Make it faster? So when Airflow is parsing a DAG, what we mean by parsing is actually it's executing the file. It's just running the Python code. 

So what I can do is I can just go in there and add some of my custom code to time all the lines, and see what's taking so long, like what's taking 300 milliseconds to run this Python file. So if you take a look at it, maybe you have an idea what is going on. Can you take a guess at what is slow in this file? So yes, some people say the variables. 

So indeed the variables are something that is not recommended, and it can be slow. But here, in this case, it was not that slow. It was not taking very long. 

What was taking long is, I tricked you. It's not on the screen. There was the import. 

So the imports are pretty basic. It's just Airflow, like import task and sensor variable. There is nothing crazy going on here, but when I was timing it, like out of the 300 milliseconds, we were spending more than 200 milliseconds on the imports. 

So I thought, well, this is very weird. How can Python run, because I was new to Airflow, I was new to Python as well. So I was like, how can Python run stuff efficiently if it takes so long to pass the imports every single time? So I read a bit about what was Happening.
What is happening is that in Airflow, we... Okay. What is happening is that in Airflow, when you parse a DAG, a DAG is a user code, so you don't want user code messing with your Airflow code. 

They can be anything in the DAG, you don't want to be able to crash Airflow. So what we do is to isolate that, the process is started, just to parse the DAG. It works on a copy of the memory of the main process, and when it's done, it's just destroyed. 

So what's happening is that you create the process, you do all the imports, they are stored, they are put in the cache, like we usually do in Python, so that we can reuse. them later, and then when we're done with the DAG, we just trash the process with the memory, and you lose all the work you did on the imports. So because of this, the imports are never processed in the main thread. 

They are never in the main memory, and so we have to process them every time, which is taking a long time. Software optimization is often identifying what is repeated and then finding a way to do it only once. We found what was repeated. 

Now we just have to find a way to do it only once. So the solutions that we're thinking about was, first, maybe you could think about communicating between the processes. But if we do that, it's pretty complex. 

There are many independent processes that can be created and destroyed. It's really hard to do, so didn't investigate this one. Maybe we could also stop using one process per DAG, but then we lose the isolation that we really need, that we cannot do without, so really not a good option. 

And then the last option was to do the import in the main thread before forking. We cannot do all the imports because some of the imports are going to be dependencies of the users that are going to conflict with what we are importing in Airflow, and also if you just do all The imports, it's user code again, but there is a set of imports that we know the users are going to import very often, and also we know what they are going to do is the imports of Airflow. Everyone is going to import operators, sensors, annotations, tags, stuff like this, so this we can preprocess for them, and it should be a good gain for many DAGs. 

I did a bug with this. What I did is I just parsed the file, I just did the tokenization, so I don't run the Python or anything. It's pretty fast, you just read the file and you separate it into tokens. 

I didn't write the code myself, I just used the Unibox library. That does that, and then I just look at the imports, and if it starts with Airflow, I import that. Using that, I did some metrics, tried to look at some benchmarks, and I could see that it was 22% faster if I was pre-importing the Airflow imports, which was nice, but I was a bit surprised because when I looked at the file first, it was taking a really long time, so I was expecting something better. 

So there was a catch. I timed the file again after doing that, and if we look at the code one more time, there is an import at the top. This one, it's actually an obsolete import, it's from like Airflow.censors. 

Now they have been moved to Airflow.providers.aws.something, and so there is some code in there that is like intercepting the import and importing something else instead, and this prevents this from being cached. So if I replace this with the actual import, like the new one, what I can see is that if I run the benchmark again, I can see a 59 gain on the time it takes to pass the DAGs, which is a lot more impressive, and that was very satisfying to me. Now, I was just running this on the files that I had given from a user, so just one type of DAG is not super representative of what users would do in the field. 

So what I did, Is, I took some example DAGs, which are just examples that people write when they write operators, and so I could get a more diverse set of DAGs and more representative performance improvements. And we get a wide range of improvements, from 25% faster to 70% faster, which is... the range is pretty wide, but overall it's very good. 

It's going to depend, of course, on how complex your DAG is, and it's also going to depend on how many imports you have, for example, that are not Airflow important that we cannot preprocess. So this is very nice, but then I was investigating a bit more on the issue, about this. usage of variables. 

When I told you that this was not very long, actually you were right, because I was running in Breeze, and when you run in Breeze you have everything that's local, like the DB is local as well, so it's pretty fast. But as soon as you have a DB that's distant, or even worse if you run like AWS Secret Manager, which is going to do network calls to AWS every time you do a secret lookup, it gets a lot slower. Like we got to 800 milliseconds per DAG as soon as I enabled AWS Secret Manager. 

And if like with my previous optimization, I could get it down to like 600 milliseconds maybe, which is still Very bad. So I was thinking, like, what should I do with it? And I talked a bit with the community, and the answer was just, don't do it. Don't use the variables, and then your DAG is faster. 

So it's a bad practice. The documentation is pretty clear about it. They say that you shouldn't do it, but the thing is that users do it anyway. 

So we cannot go to the user and hit them with a stick until they stop doing it, even if it's written. So what should we do about it? What we should do is, even though it's not recommended, we can do whatever we can to make it as fast as possible and not as bad as it is right now. So what I thought about was, Let's add a cache that solves everything, right? No, it doesn't. 

There are many problems with caching. Cache invalidation, for example, is a big problem. When do you decide that you need to refresh your cache? Also, for example, different processes can have a different state in their cache, and then the behavior might be inconsistent. 

It also delays the time it takes for updates to occur, but also, it's still very efficient. It's efficient at saving time, it's efficient at saving money as well, because whenever you query an external service, you might be charged for the API calls that you do. It's the case for the secret manager on AWS, but also for GCP, for Azure, like all three cloud providers charge per API call, so it can be some pretty expensive cloud deals. 

So adding a cache was a good solution, and there was a lot of discussion in the community on whether we actually wanted to do that, because it was kind of encouraging a bad practice, and also because of all the catches I just talked about, but the conclusion was yes. So we now have an optional cache and variables. It's auth by default. 

Mark was talking about it this morning in his talk. This is the configuration to enable it, and just on the Technical side, it uses Python Multiprocessing Manager. So since the DAGs are in different processes, we cannot, like, they can share a dictionary that's here before, but if they write to it, they're going to write in their own memory, not in shared memory. 

So the Multiprocessing Manager is something that's in core Python. It's super cool. It spawns a different process that's only going to manage a dictionary, and then every call you make to the dictionary is transparently sent to this process, and the process sends back, like, if you say, "Give me this key," it's going to send back the key. 

So it's a nice way to do it. There are some caveats, like what I just talked about before, especially if you use configure like variables to decide, for example, how many tasks there's going to be in your DAG. You might have some bad surprises, because when you change the configuration, and if you have two schedulers, for example, they won't have the same status at the same time, and your DAG might be flipping back and forth during the time the two caches are not in the same state. 

And I'm not going to show you any metrics, any benchmarks, because I can make the benchmarks as good as I want. You can just write a DAG that has like one million. Variable calls, and it's going to improve by 99%, but it can help a lot if your DAG has variable calls. 

It's also including connections, because connections are actually secrets, so whenever you do AWS connection ID, for example, it's actually a query on the secret, it's only during parsing, so not when the DAG is actually running on the worker, because then we went to fresh data. And also it's experimental, so it's a feature that you should use with care. The conclusion of my talk is that you don't have to be an expert to have an impact on something. 

You just, by digging down and looking at the code. And the metrics and trying to understand what is happening, you can do something cool. You just need time and motivation to investigate. 

I know that it's not easy for everyone, especially the time components, because I'm paid to contribute to Airflow, so I can do that, but not everyone can do that on their personal time. But yeah, if you have time and motivation, you can achieve stuff, and that's it..

# Airflow DAG 파싱 및 처리 과정

## 1. 소개

Apache Airflow는 워크플로우 관리 플랫폼으로, DAG(Directed Acyclic Graph)를 통해 작업 흐름을 정의하고 관리합니다. 이 발표는 Airflow의 내부 동작, 특히 DAG 파싱과 처리 과정의 아키텍처를 중점적으로 설명합니다.

### 주요 내용:
- DAG 파싱 프로세스
- DAG 프로세싱
- 데이터 저장 구조
- 메타데이터 데이터베이스와의 관계
- 파싱 최적화 기법

---

## 2. DAG 파싱 프로세스

### 2.1 개요

DAG 파싱은 Python 파일로 작성된 워크플로우 정의를 Airflow가 읽고, 유효성을 검사하며, 필요한 메타데이터를 데이터베이스에 저장하는 과정입니다.

### 2.2 파싱 주요 단계

1. **디렉토리 스캔**: `dags_folder` 디렉토리 주기적 확인
2. **파이썬 파일 로드**: `*.py` 확장자 파일 메모리 로드
3. **DAG 객체 파싱**: 유효한 DAG 객체 검색 및 검증
4. **Task/Operator 검증**: 작업 간 연결 관계 및 의존성 확인

### 2.3 파싱 컴포넌트

#### DagFileProcessorManager
```python
class DagFileProcessorManager(LoggingMixin):
    def run(self):
        while True:
            self.prepare_file_queue()
            self._start_new_processes()
            self._collect_results()
```

- 여러 DAG 파일을 병렬로 처리
- 파일 큐 준비, 프로세스 생성, 결과 수집 담당

#### DagFileProcessorProcess
```python
def _parse_file(msg: DagFileParseRequest, log):
    bag = DagBag(dag_folder=msg.file, safe_mode=True)
    serialized_dags, errors = _serialize_dags(bag, log)
    return DagFileParsingResult(
        fileloc=msg.file,
        serialized_dags=serialized_dags,
        import_errors=errors,
        warnings=[],
    )
```

- 독립적인 프로세스에서 개별 DAG 파일 처리
- `DagBag` 객체를 통해 DAG 로드 및 직렬화

### 2.4 파싱 병목 현상

- **문제점**: 많은 개별 DAG 파일이 있을 경우 CPU 사용량 급증
- **기본 설정**: 30초마다 2개 프로세스로 파싱, 파일당 약 300ms 소요
- **성능 한계**: 200개 이상 DAG 파일 시 30초 내 모든 파일 파싱 불가

---

## 3. DAG 프로세싱

### 3.1 개요

파싱된 DAG는 Scheduler에 의해 처리되며, 실행 시점과 태스크 의존성을 분석하여 Executor에 작업을 전달합니다.

### 3.2 프로세싱 단계

1. **DAG 객체 수집**: DagBag에서 파싱된 DAG 객체 가져오기
2. **스케줄링 결정**:
   - `start_date`, `end_date`, `schedule_interval` 평가
   - 실행 필요성 결정
3. **Task 상태 결정**: TaskInstance 상태 조회
4. **Executor에 작업 전달**: 다양한 executor 지원 (Local, Celery, Kubernetes 등)

### 3.3 코드 예시

```python
# airflow/scheduler/scheduler_job.py
class SchedulerJob:
    def _process_dags(self, dagbag):
        for dag in dagbag.dags.values():
            dag_runs = self.create_dag_runs(dag)
            for dag_run in dag_runs:
                self._process_task_instances(dag_run)
```

---

## 4. DAG 저장 구조

### 4.1 데이터베이스 개요

Airflow는 DAG 및 태스크의 메타데이터를 관계형 데이터베이스에 저장합니다:
- 기본 지원: MySQL, PostgreSQL, SQLite
- 메타데이터 관리: 실행 상태, 이력, 구성 정보

### 4.2 주요 테이블

| 테이블명 | 설명 |
|----------|------|
| `dag` | DAG 메타데이터 저장 |
| `serialized_dag` | 직렬화된 DAG 구조 저장 |
| `dag_run` | 실행된 DAG 인스턴스 정보 |
| `task_instance` | 각 태스크의 실행 상태 관리 |
| `import_error` | 파싱 중 발생한 오류 기록 |
| `log` | 태스크 실행 로그 |

### 4.3 DAG 직렬화 및 저장

```python
for dag in bag.dags.values():
    serialized_dag = SerializedDAG.to_dict(dag)
    session.add(SerializedDagModel(dag_id=dag.dag_id, data=serialized_dag))
```

- DAG 객체는 JSON 형식으로 직렬화
- `serialized_dag` 테이블에 저장
- 스케줄러와 웹 UI는 이 데이터 활용

---

## 5. 메타데이터 데이터베이스 관계

### 5.1 데이터 흐름

1. DAG 정의 → DAG 파싱
2. 파싱된 정보 → 데이터베이스 테이블 저장
3. Scheduler → 실행 정보 조회
4. Task 큐 등록 → Executor 실행
5. Task 상태 업데이트 → 데이터베이스 갱신

### 5.2 데이터베이스 상호작용

```python
def update_dag_parsing_results_in_db(bundle_name, dags, session):
    for dag in dags:
        SerializedDagModel.write_dag(dag, bundle_name=bundle_name, session=session)
    session.commit()
```

- 파싱 결과, 실행 상태, 로그 정보 등 지속적 업데이트
- 웹 UI는 이 정보를 통해 현재 상태 표시

---

## 6. DAG 파싱 최적화 기법

### 6.1 성능 문제 분석

- **분리된 프로세스**: 사용자 코드 안전 실행을 위한 별도 프로세스 생성
- **반복 임포트**: 같은 모듈 반복 임포트로 시간 소모
- **프로세스 격리**: 파싱 후 프로세스 종료로 메모리 캐시 손실
- **측정 결과**: 파싱 시간의 2/3 이상이 임포트에 사용

### 6.2 최적화 기법 1: Pre-Importing

```python
# 공통 모듈 미리 import
import airflow
import airflow.operators
```

- **개념**: 자주 사용하는 모듈 미리 임포트로 시간 단축
- **성능 개선**: 25%~70% 속도 향상 효과
- **구현**: 토큰화하여 Airflow 관련 임포트 식별 및 사전 로드

### 6.3 최적화 기법 2: 캐싱 메커니즘

```python
from multiprocessing import Manager

manager = Manager()
shared_cache = manager.dict()

def get_variable(key):
    if key in shared_cache:
        return shared_cache[key]
    else:
        value = fetch_from_external_service(key)
        shared_cache[key] = value
        return value
```

- **문제점**: 변수 조회, 외부 서비스 호출 시 지연 발생(~800ms)
- **구현**: Multiprocessing Manager를 사용한 공유 캐시
- **고려사항**: 캐시 무효화, 프로세스 간 일관성, 외부 API 호출 감소

### 6.4 설정 최적화 옵션

- **파싱 간격 증가**: `dag_file_processor_timeout`
- **프로세스 수 증가**: `max_threads`
- **DAG 프로세서 분리**: `[scheduler] parse_dags = False`
- **캐싱 기능 활성화**: 기본적으로 인증 사용

---

## 7. 결론 및 권장 사항

### 7.1 주요 개선 전략

- 모듈 사전 임포트를 통한 파싱 속도 개선
- 변수 및 시크릿 관리를 위한 캐싱 메커니즘
- 설정 매개변수 최적화

### 7.2 실무 적용 권장사항

- DAG 파일 수와 복잡성 관리
- 변수 사용 최소화 또는 캐싱 활용
- 환경에 맞는 설정 매개변수 조정
- 정기적인 성능 모니터링

### 7.3 성과 측정

- 파싱 시간: 최대 70% 감소
- CPU 사용량: 뚜렷한 감소
- 외부 API 호출: 캐싱으로 감소

---

## 8. 참고 자료

- Airflow 공식 문서: [https://airflow.apache.org/docs/](https://airflow.apache.org/docs/)
- Airflow 소스 코드: [https://github.com/apache/airflow](https://github.com/apache/airflow)
- DAG 파싱 최적화 발표: [YouTube](https://www.youtube.com/watch?v=W0X7NEj7mRk)
- Airflow Database Schema 문서









