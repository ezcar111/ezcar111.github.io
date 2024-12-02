---
layout: post
read_time: true
show_date: true
title:  Textom Crawler Process update
date:   2024-06-02 13:00:00 -0600
description: 텍스톰 수집 과부화 문제를 해결하고 효율성을 높이기 위한 Master-Slave 구조 업데이트
img: posts/20240602/textom_logo.png
tags: [crawling, web service, distributed system, performance, optimization]
author: Huiseong Kwon
github: ezcar111/textom_cube
mathjax: yes
---

## 텍스톰 크롤러의 구조와 기능 업데이트 🚀

최근 텍스톰24가 업데이트됨에 따라 사용자가 급증하여, 웹사이트 수집 요청이 대규모로 증가하였습니다. 이로 인해 **수집 속도 저하**와 **웹 차단 문제**를 마주하였습니다. 이러한 문제를 해결하기 위해 기존 단일 프로세스 구조를 **Master-Slave 분산 처리 구조**로 변경하였습니다. gRPC통신을 통해 이번 글에서는 새로운 구조와 이를 통해 얻어진 성능 개선 내용을 공유합니다.

---

### 문제점 🛑
1. **수집 속도 저하**  
   - 단일 프로세스에서 요청을 처리하던 기존 방식은 대규모 데이터 수집시 병목 현상 발생
   - 수집요청이 DB에 입력될 때 해당 요청을 처리해야 할 서버가 배정됨
   - 현재 수집서버들의 상태는 고려되지 않고 수집서버들의 IP가 순차적으로 배정
   - 유휴한 수집서버가 있음에도 불구하고 데이터 수집이 진행되지 않은 상황이 발생하여 데이터 수집 지연
   
2. **웹사이트 차단 문제**  
   - 동일한 IP에서의 반복적인 요청으로 인해 여러 웹사이트에서 차단되는 사례가 발생

3. **서버관리의 문제**
   - 단일 프로세스의 서버 여러대를 동시에 관리하는 구조가 마련되지 않아 각서버를 따로 관리하는 현상이 발생
<center><img src='../assets/img/posts/20240602/20240602_Before_Configuration_Diagram_by_Crawler.png'></center>

---

### 해결 방안 💡
#### 1. **Master-Slave 구조 (TO-BE 구조) 도입**
- **Master**:
  - 수집 작업 요청을 분배하고 전체 작업 진행 상태를 관리
  - 각 슬레이브의 상태를 실시간 모니터링하여 작업을 동적으로 할당
  
- **Slave**:
  - 30개 이상의 분산된 서버에서 크롤링 작업을 병렬로 수행
  - 독립적인 IP를 사용하여 각 요청을 처리, 차단 위험을 최소화
<center><img src='../assets/img/posts/20240602/20240602_Before_After_Crawler_Network_flow.png'></center>

#### 2. **분산 시스템을 위한 주요 기술 스택**
- **gRPC**:
  - Master와 Slave 간의 작업 요청 및 결과 수집에 사용
  - gRPC 플러그인를 기반으로 경량화된 통신 프로토콜을 구현하여 작업의 실시간 처리를 지원
- **git**:
  - Master코드와 Slave코드를 효율적으로 관리하기 위해 git lab 가용
  - 수정사항 및 히스토리 관리 체계 생성

---

### 구현 방법 🛠️

#### 1. Master통신 프로세스
- **loop 구조 생성, 작업 상태 모니터링 및 동적 작업 분배**
  - Master는 반복적으로 작업 요청을 처리할 수 있는 loop 구조를 생성
  - DB를 지속적으로 확인하며, 새로운 작업 발생시 Slave에게 동적으로 작업을 분배

``` python
class ComManager(Com):

   def __init__(self):
      super().__init__()
      self.decorator = ManagerEarthlingDecorator()

   def serve(self):
      compose = self.monitor.get_compose()['manager']
      port = compose['port']
      self.decorator.serve(port)
      server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
      EarthlingProtocol_pb2_grpc.add_EarthlingServicer_to_server(Earthling(), server)
      server.add_insecure_port(f'[::]:{port}')
      server.start()
      server.wait_for_termination()

   def loop(self):
      while True:
         result = select_wait_task()
         # log.debug(result)
         for message in result:
               channel = message['channel']
               task_no = message['idx']
               kwd_idx = message['k_idx']
               search_kind = message['search_kind']
               data_type_names = message['data_type_names']
               assistant = self.monitor.get_compose()['assistant_N']

               for ass in assistant:
                  addr, port = ass['address'], ass['port']
                  idle_count = self.decorator.getIdleWorkerCount(addr, port)
                  time.sleep(0.02)  # 트래픽 제한을 위한 딜레이

                  if idle_count > 0:
                     data_exec = 'N'
                     # data_type_names = [ 'web_data', 'news_data', 'facebook_data' ]
                     for data_type_name in data_type_names:
                        data_type_key_name = f"{data_type_name}_data"
                        data_exec = message[data_type_key_name]
                        if data_exec == 'Y':
                           data_desc = { "channel": channel, "data_type_name": data_type_name, "state": data_exec }
                           update_state_to_start(task_no, data_type_name, channel)
                           result = self.decorator.notifyTaskToAss(addr, port, task_no, json.dumps(data_desc))
                           result_message = json.loads(result.message)
                           is_success = result_message["is_success"]
                           err_message = result_message["err_message"]

                           if not is_success:
                                 update_state_to_wait(task_no, data_type_name, channel)

                           collect_info = json.dumps({
                                 "req_time": str(datetime.now()).split('.')[0],
                                 "kwd_idx": kwd_idx,
                                 "cct_idx": task_no,
                                 "chan": channel,
                                 "cate": data_type_name,
                                 "wkr_addr": addr,
                                 "wkr_port": port,
                           })
                           
                           logass.info(collect_info)
                           log.debug(f"[{kwd_idx}] Request task-{task_no} ({channel}: {data_type_name}) to Assistant Domain: {addr}:{port}")
                           break
                        break
         time.sleep(0.15)  # 이 딜레이는 반드시 필요한 딜레이
def run():  
    mng = ComManager()
    p = Process(target=mng.loop, args=())
    p.start()

    compose = mng.monitor.get_compose()['manager']
    addr, port = compose['address'], compose['port']
    log.debug(f"Started Manager Server From {addr}:{port}")
    mng.serve()

if __name__ == "__main__":
   run()
```

- **DAO(Data Access Object)생성**
  - **데이터베이스 접근 캡슐화** : 수집사이트(Naver, Daum, Google 등) 각각의 DAO를 별도 생성하여 데이터베이스 쿼리와 연관된 모든 작업을 수행
  - **작업 대기(Task Queue) 관리**: `select_wait_task` 메서드를 통해 특정 조건에 부합하는 대기 작업(수집할 키워드와 관련 데이터)을 데이터베이스에서 조회하여 반환합니다. 이를 통해 작업 상태를 효율적으로 모니터링하고 관리합니다.
  - **작업 상태 업데이트**: `update_state_to_finish` 메서드는 수집 작업 완료 후 데이터를 업데이트하며, 작업 상태와 키워드 상태를 변경하여 프로세스의 일관성을 유지합니다.
  

``` python
import os, sys, time
import yaml, random, pickle
from .earthling_db_pool import exec
from service.Logging import log

# channel : 수집사이트(Naver, Daum, Google, Youtube 등)를 의미함
def get_dao(channel = ''):
    dao_list = []
    with open("earthling/handler/dao/settings.yaml") as f:
        dao_settings = yaml.load(f, Loader=yaml.FullLoader)    
        
        if channel != '':
            channel_setting = dao_settings.get(channel)
            dao_file_path = channel_setting.get("dao_file_path")
            with open(dao_file_path, 'rb') as file:
                dao = pickle.load(file)
            return dao

        for channel in list(dao_settings.keys()):
            dao = None
            channel_setting = dao_settings.get(channel)
            dao_file_path = channel_setting.get("dao_file_path")
            with open(dao_file_path, 'rb') as file:
                dao = pickle.load(file)
                dao_list.append(dao)
                
    return dao_list

def select_wait_task():
    daos = get_dao()
    random.shuffle(daos)
    # log.debug(daos)
    result = []
    for dao in daos:
        q_result = dao.select_wait_task()
        if "list" in str(type(q_result)):
            result = result + q_result
        time.sleep(0.2)
    return result

def update_state_to_wait(no, data_type_name, channel):
    dao = get_dao(channel)
    dao.update_state_to_wait(no, data_type_name)

def update_state_to_start(no, data_type_name, channel):
    dao = get_dao(channel)
    dao.update_state_to_start(no, data_type_name)

def update_state_to_finish(no, data_type_name, channel):
    dao = get_dao(channel)
    dao.update_state_to_finish(no, data_type_name)

def get_collection_cond(no, channel):
    dao = get_dao(channel)
    return dao.get_collection_cond(no)

def update_state_S_to_Y(channel):    
    dao = get_dao(channel)
    return dao.update_state_S_to_Y()

```


``` python
import os, sys
sys.path.append(os.path.dirname(os.path.abspath(os.path.dirname(__file__))))

import yaml
from handler.dao.BaseDAO import BaseDAO
from service.Logging import log
import random

class NaverDAO(BaseDAO):

   def __init__(self):
      super().__init__("naver")

   def select_wait_task(self):
      test_index_query = self.get_test_index_query()
      host_ip = self.get_host_ip()

      query = """SELECT * FROM naver_table WHERE (web_data  = 'Y' OR blog_data = 'Y' 
         OR news_data = 'Y' OR cafe_data = 'Y'
         OR kin_data  = 'Y' OR academic_data = 'Y') LIMIT 100 
      """
      result = self.exec(query)
      for item in result:
         item["channel"] = self.channel
         item["data_type_names"] = self.data_type_names
      return result
   def update_state_to_finish(self, no, data_type_name):
      self.update_state(no, data_type_name, 'N')
      query = f"SELECT keyword_list_idx, web_data, blog_data, news_data, cafe_data, kin_data, academic_data FROM scraw_naver_datainfo WHERE idx={no}"
      result = self.exec(query)
      if len(result) > 0:
         k_idx = result[0]["keyword_list_idx"]
         crawl_state = [
            result[0]["web_data"],
            result[0]["blog_data"],
            result[0]["news_data"],
            result[0]["cafe_data"],
            result[0]["kin_data"],
            result[0]["academic_data"]
         ]

         if ('Y' not in crawl_state) and ('S' not in crawl_state):
            query = f"UPDATE keyword_list SET naver_status = 'merge' WHERE idx = {k_idx}"
            self.exec(query)
```

#### 2. Slave 동작 프로세스
- **피클(Cache 또는 Serialize) 관리**
  - 수집 사이트별 크롤러와 관련된 메타데이터, 상태 정보, 설정을 직렬화(Serialize)하여 .pickle 파일로 저장
  - 피클은 캐싱처럼 동작하며, Slave가 실행될 때 초기화 데이터를 로드하거나 이전 상태를 복구하는 데 활용


``` python
from earthling.handler.dao.NaverDAO import NaverDAO
from earthling.handler.dao.GoogleDAO import GoogleDAO
# DAO 클래스 임포트 생략

dao_class = {
    "naver": NaverDAO(),
    "google": GoogleDAO(),
    # 매핑 생략
}

from application.naver.NaverBase import NaverBase
from application.google.GoogleBase import GoogleBase
# 데이터 클래스 임포트 생략

data_class = {
    "naver": {
        "base": NaverBase(),
        # 매핑 생략
    },
    "google": {
        "base": GoogleBase(),
        # 매핑 생략
    },
    # 채널 매핑 생략
}

if __name__ == "__main__":
    with open("earthling/handler/dao/settings.yaml") as f:
        dao_settings = yaml.load(f, Loader=yaml.FullLoader)

        for channel in list(dao_settings.keys()):
            dao_file_path = dao_settings[channel].get("dao_file_path")
            dao = dao_class.get(channel)
            if dao:
                with open(dao_file_path, 'wb') as file:
                    log.debug(dao_file_path)
                    pickle.dump(dao, file)

    with open("application/settings.yaml") as f:
        app_settings = yaml.load(f, Loader=yaml.FullLoader)

        for channel in list(app_settings.keys()):
            if "dict" not in str(type(app_settings[channel])):
                continue

            serialized_file_path = app_settings[channel].get("serialized_file_path")
            data_types = app_settings[channel].get("data_types")

            save_file_path = f"{serialized_file_path}/base.pickle"
            with open(save_file_path, 'wb') as ff:
                log.debug(save_file_path)
                target_data_class = data_class.get(channel).get("base")
                pickle.dump(target_data_class, ff)

            for data_type in data_types:
                target_data_class = data_class.get(channel).get(data_type)
                if target_data_class:
                    save_file_path = f"{serialized_file_path}/{data_type}.pickle"
                    with open(save_file_path, 'wb') as ff:
                        log.debug(save_file_path)
                        pickle.dump(target_data_class, ff)

```

- **gRPC 기반 통신 및 작업 처리**
  - Slave는 Master로부터 gRPC를 통해 작업을 수신하고, 크롤링 작업을 수행하며 결과를 Master에 전달
  - 작업 요청과 결과 전달은 실시간으로 이루어지며, 유휴 상태(Idle Count)를 관리하여 작업 효율성을 극대화


``` python
class ComAssistant(Com):
    def __init__(self, decorator):
        super().__init__()
        self.decorator = decorator
        self.procs = []

    def serve(self):
        compose = self.monitor.get_compose()
        host_addr = compose['host']['address']
        assistant = compose['assistant']
        for ass in assistant:
            if ass['address'] == host_addr:
                port = str(ass['port'])
                self.decorator.serve(port)
                break

    def loop(self):
        while True:
            try:
                idle_count = 0
                workers = WorkerPool.getInstance().workers
                for worker in workers:
                    is_working = True if worker.is_working.value > 0 else False
                    if not is_working: 
                        is_working = WorkerPool.getInstance().pop_work()
                    if not is_working: 
                        idle_count = idle_count + 1
                self.decorator.set_idle_count(idle_count)
            except Exception as err:
                log.debug(err)
                log.debug("Can't connect remote...")
                time.sleep(1)
            time.sleep(3)

def action(task):
    log.debug("Undefined Action")

def run(action):
    compose = Monitor().get_compose()
    mng_addr, mng_port = compose['manager']['address'], compose['manager']['port']
    host_addr, host_port = compose['host']['address'], compose['host']['port']
    
    task_queue = Queue()
    WorkerPool.getInstance(action).set_task_queue(task_queue)
    shared_idle_count = Value('i', 0)
    earthling = proto.AssistantEarthling(shared_idle_count, WorkerPool.getInstance())
    decorator = proto.AssistantEarthlingDecorator(earthling)
    ass = ComAssistant(decorator)

    try:
        echoed = ass.decorator.echo(mng_addr, mng_port, str(host_port))
        log.debug(f"Manager로부터 받은 echo 메시지: {echoed}")
    except Exception as err:
        log.debug(err)
        log.debug("Manager 서버에 연결할 수 없습니다.")

    p = Process(target=ass.loop, args=())
    p.start()

    log.debug(f"Started Assistant Server From {host_addr}:{host_port}")
    ass.serve()
```
#### 3. 데이터 저장 및 관리
- 크롤링된 데이터는 `ElasticsearchConnector`와 `MySQLPoolConnector`를 사용하여 Elasticsearch와 MySQL에 저장.
- 데이터 접근 계층은 DAO 패턴(`handler/dao`)을 통해 구현하여 관리 용이성 향상.

---

### 성능 개선 결과 🌟
1. **처리 속도 300% 증가**  
   - 기존 방식에서는 평균 100건(13개 수집사이트의 3개월 데이터 하루제한 1000건, 0 ~ 1,184,000 건)의 요청 처리에 약 10분이 소요되었으나, **Master-Slave** 구조 도입 후 8분 내로 단축. (직접적은 크롤링 속도는 변화가 없지만 ip차단 최소화로 sleep 시간 단축 + 작업분배시간 최소화)
   - gRPC를 활용한 병렬 처리로 작업 지연 없이 효율적 처리 가능.

2. **차단 사례 약 70% 감소**  
   - IP 분산 처리로 인해 동일한 도메인에 대한 연속 요청이 차단될 가능성 대폭 감소.

3. **확장성 강화**  
   - Slave 서버를 추가하는 것만으로 전체 처리량을 손쉽게 확장 가능.
   - 현재 35대의 Slave 서버를 운영 중이며, 필요 시 무중단으로 서버를 추가 배치 가능.

---

### 차후 계획 및 업데이트 예정 📝
- **작업 로드 밸런싱 강화**:
  - Slave의 처리 능력에 따라 작업을 자동으로 재분배하는 알고리즘 도입.
- **모니터링 대시보드 개발**:
  - 작업 현황, 크롤링 성공률 및 에러 상태를 시각화하여 실시간 모니터링 제공.
- **VPN Proxy를 활용한 SNS IP차단 대책 수립**:
  - VPN과 Proxy를 결합하여 Slave 각각이 고유의 IP를 통해 크롤링 요청 필요.

---


### 결론 🎯
Master-Slave 구조 도입을 통해 텍스톰 크롤러는 대규모 데이터를 빠르고 안정적으로 수집할 수 있게 되었습니다. 앞으로도 지속적인 업데이트를 통해 더 나은 성능과 확장성을 제공할 예정입니다.

---

### 태그
- **#Crawling**  
- **#Distributed System**  
- **#Performance Optimization**  
- **#WebService**

---

### 참고 링크
- [GitHub Repository](https://github.com/ezcar111/textom_cube)

---