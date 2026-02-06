## 3.2 서비스 관리 (service)


| 명령어 | 기능 | 주요 플래그 (옵션) |
|:---|:---|:---|
| `check` | 전체 서비스 상태 점검 | 없음 |
| `start` | 서비스 시작 | `[service]` : 서비스명 (생략 시 all) |
| `stop` | 서비스 중지 | `[service]` : 서비스명 (생략 시 all) |
| `restart` | 서비스 재시작 | `[service]` : 서비스명 (생략 시 all) |
| `jvm` | JVM 메모리 최적화 | `--set` : 설정값 (auto, S, M, L, 4G 등)<br/>`--dry-run` : 시뮬레이션<br/>`--no-restart` : 재시작 건너뛰기 |
| `loglevel` | 로그 레벨 조회/설정 | `[service]` : 대상 서비스<br/>`--set` : 레벨 설정 (INFO, DEBUG 등) |
| `logrotate` | 로그 로테이션 관리 | `--set` : 권장 설정 자동 적용 |


TACHYON 및 미들웨어 서비스의 상태를 확인하고 제어합니다. 각 서브커맨드(`check`, `start`, `stop`, `restart`, `loglevel`, `jvm`, `logrotate`)는 독립적인 도움말(`-h`)을 지원합니다.

### 사용법

```bash
tkcli service [action] [service_name] [flags]
```

### 사용 가능한 액션

- `check`: 모든 서비스의 상태 확인
- `start <service>`: 서비스 시작
- `stop <service>`: 서비스 중지
- `restart <service>`: 서비스 재시작
- `jvm`: JVM 힙 메모리 설정 관리 및 자동 최적화

### 서비스 목록

**미들웨어 서비스:**
- `mariadb` - MariaDB 데이터베이스
- `redis` - Redis 캐시
- `nginx` - NGINX 웹 서버
- `zookeeper` - Zookeeper
- `kafka` - Kafka 메시지 브로커
- `opensearch` - OpenSearch 검색 엔진
- `opensearch-dashboards` - OpenSearch 대시보드
- `logstash-kafka-os` - Logstash

**TACHYON 서비스:**
- `TACHYON-Api1` - API 서버
- `TACHYON-Auth1` - 인증 서버
- `TACHYON-Manager1` - 관리 서버
- `TACHYON-Stat1` - 통계 서버
- `TACHYON-Report1` - 리포트 서버
- `TACHYON-Batch1` - 배치 서버
- `TACHYON-Watchdog1` - 감시 서버

**특수 옵션:**
- `all` - 모든 서비스 (순차적 시작/중지)

### 예제

```bash
# 모든 서비스 상태 확인
tkcli service check

# 특정 서비스 시작
tkcli service start mariadb
tkcli service start TACHYON-Api1

# 모든 서비스 시작 (순차적 실행 워크플로우)
tkcli service start all

/* 실행 순서 상세 */

1. Level 1: 기본 인프라 (Infra)
   - zookeeper
   - redis
   - mariadb
   - opensearch
   - opensearch-dashboards
   (3초 대기)

2. Level 2: 메시징 (Kafka)
   - kafka
   (5초 대기 & 연결 상태 검증)

3. Level 3: 로그 수집 및 웹 서버
   - logstash-kafka-os
   - nginx

4. Level 4: TACHYON API
   - TACHYON-Api* (발견된 모든 인스턴스: Api1, Api2...)
   (60초 대기 - API 초기화)

5. Level 5: TACHYON 서비스 로직
   - TACHYON-Auth*
   - TACHYON-Manager*
   - TACHYON-Stat*
   - TACHYON-Report*
   - TACHYON-Batch*
   (10초 대기)

6. Level 6: 감시 (Watchdog)
   - TACHYON-Watchdog*


# 특정 서비스 중지
tkcli service stop nginx

# 모든 서비스 중지 (역순 워크플로우)
tkcli service stop all
(Watchdog -> Service Logic -> API -> Logs/Web -> Kafka -> Infra 순서로 안전하게 종료)
```

### 출력 예시 (service check)

```
2025-12-24 21:02:46 [INFO] ### TACHYON 서비스 상태 점검 ###

--- 서비스 리소스 사용량 ---
-------------------------------------------------------------------------
 서비스명                        상태        CPU(%)      메모리         
-------------------------------------------------------------------------
 mariadb                         Active      0.1         324.78 MB      
 redis                           Active      0.1         512.0 KB       
 zookeeper                       Active      0.1         123.01 MB      
 kafka                           Active      1.0         1.36 GB        
 opensearch                      Active      0.0         4.62 GB        
 TACHYON-Api1                    Active      0.0         771.18 MB      
 TACHYON-Batch1                  Active      0.2         1.08 GB        
 TACHYON-Manager1                Active      0.0         1.21 GB        
 ...
-------------------------------------------------------------------------

--- JVM 힙 메모리 설정 ---
---------------------------------------------
 서비스           초기(-Xms)    최대(-Xmx)  
---------------------------------------------
 opensearch       4g            4g          
 kafka            2G            2G          
 ...
---------------------------------------------

  ✓ 모든 JVM 서비스의 메모리 설정이 확인되었습니다.

--- 로그 로테이션 상태 요약 ---
  ✓ 모든 서비스의 로그 로테이션 설정이 올바릅니다.

2025-12-24 21:02:46 [SUCC] 서비스 상태 확인 완료.
```

---

### 3.2.1 서비스 로깅 레벨 확인 및 설정 (loglevel)

운영 중인 각 서비스의 로깅 레벨(DEBUG, INFO, WARN, ERROR 등)을 확인하거나 변경합니다.
Java 기반의 TACHYON 서비스와 주요 미들웨어(Nginx, Redis 등)의 설정 파일을 분석하여 현재 적용된 레벨을 표시하며, `--set` 옵션을 통해 즉시 변경이 가능합니다.

**사용법:**
```bash
# 모든 서비스의 로깅 레벨 확인
tkcli service loglevel

# 특정 서비스의 로깅 레벨 확인
tkcli service loglevel api

# 특정 서비스의 로깅 레벨 변경
tkcli service loglevel api --set DEBUG
tkcli service loglevel auth --set INFO
tkcli service loglevel batch --set WARN
```

**지원되는 레벨:**
- `DEBUG`, `INFO`, `WARN`, `ERROR`, `TRACE`, `OFF`

**진단 대상 파일 및 세부 항목:**
- **TACHYON 서비스**: `*.yml_dev` (평문 원본), `logback-spring.xml`, `log4j2.xml`
  - `logging.level.*` 설정을 기준으로 시스템 로깅 레벨을 표시합니다.
- **NGINX**: `nginx/conf/nginx.conf`
  - **Error Log**: `error_log` 지시어에 설정된 레벨(DEBUG, INFO, ERROR 등)을 추출합니다.
  - **Access Log**: NGINX 특성상 레벨 없이 기록 여부만 제어하므로, `access_log off;` 설정 여부를 파싱하여 **ON/OFF**로 표시합니다.
- **Redis**: `redis.conf` (`loglevel` 지시어)
  - Redis 표준 레벨(DEBUG, VERBOSE, NOTICE, WARNING)을 추출합니다.

**로깅 레벨 변경 로직 (Java 서비스 - v0.4.22 개선):**

| 단계 | 동작 | 설명 |
|:---:|:---|:---|
| 1 | `*.yml_dev` 파일 수정 | `logging.level.*` 아래 모든 키 일괄 변경 (root, com.tachyon 등) |
| 2 | `*.yml` 파일 복사 | Spring Boot가 사용하는 실제 설정 파일에 반영 |
| 3 | `*_ORIGIN` 파일 동기화 | 기본 설정 파일도 함께 업데이트 (재설치 후에도 설정 유지) |
| 4 | 서비스 재시작 | `systemctl restart TACHYON-[Svc]1.service` |

> **참고**: `logging.level.root` 키가 없는 서비스(batch, watchdog 등)도 자동으로 키가 생성됩니다.

---

### 3.2.2 서비스 로그 로테이션 상태 확인 및 설정 (logrotate)

TACHYON 및 미들웨어 서비스의 로그 로테이션 설정 상태를 확인하고, 누락된 설정을 자동으로 구성합니다.

**사용법:**
```bash
# 전체 서비스 로그 로테이션 상태 확인
tkcli service logrotate

# 특정 서비스 로그 로테이션 상태 확인
tkcli service logrotate nginx

# 누락된 서비스에 대해 로그 로테이션 자동 설정
tkcli service logrotate --set

# 특정 서비스 로그 로테이션 설정
tkcli service logrotate redis --set
```

**지원 서비스:**
- **미들웨어**: `nginx`, `redis`, `opensearch` (메인), `opensearch-gc` (GC 로그), `kafka`, `logstash`, `mariadb`
- **TACHYON**: `api`, `auth`, `manager`, `stat`, `batch`, `report`, `watchdog` (Java/Log4j2 기반)
- **관리 도구**: `tkadmin`, `tkcli`

**로테이션 유형:**
- `logrotate`: 시스템 logrotate 데몬 사용 (`/etc/logrotate.d/`)
- `log4j2`: Java Log4j2 내부 롤링 정책 사용
- `internal`: 서비스 자체 내장 로테이션 메커니즘 사용

---

### 3.2.3 서비스 JVM 메모리 최적화 및 관리 (jvm)

TACHYON 및 관련 미들웨어의 JVM 힙 메모리(-Xms, -Xmx) 설정을 조회, 변경하거나 시스템 사양에 맞춰 자동 최적화합니다.

**사용법:**
```bash
# 전체 JVM 서비스 메모리 설정 및 시스템 대비 할당량 점검
tkcli service jvm

# 특정 서비스의 메모리 설정 변경
tkcli service jvm api --set 2G

# 공식 권장 프리셋 적용 (XS, S, M, L)
tkcli service jvm --set S

# 시스템 사양 및 에이전트 수 기반 자동 최적화 (권장)
tkcli service jvm --set auto

# 에이전트 수를 명시하여 자동 최적화 수행
tkcli service jvm --set auto --agents 5000

# 실제 반영 없이 변경될 결과만 미리보기 (Dry-run)
tkcli service jvm --set auto --dry-run

# 설정 변경 후 서비스 자동 재시작 건너뛰기
tkcli service jvm --set auto --no-restart
```

**자동 최적화 알고리즘 (auto):**
- 서버의 **물리 메모리(RAM)** 크기와 관리 중인 **에이전트 수(Scale)**를 분석하여 서비스별 최적 비중을 계산합니다.
- OpenSearch(20%), Kafka(10%), API(5%) 등 각 컴포넌트의 중요도와 실제 부하 분담률을 반영합니다.
- 서비스별 최소 Heap(256MB~1GB) 보장 및 최대 상한(16GB) 정책이 적용되어 시스템 안정성을 확보합니다.

**프리셋 유형:**
- `XS` (Extra Small): 8GB 이하 저사양 테스트 환경용 (초경량 설정)
- `S` (Small): 16GB 권장 사양용
- `M` (Medium): 32GB 권장 사양용
- `L` (Large): 64GB 이상 고사양용

**안전 장치:**
- 모든 설정 변경 전 원본 파일에 대한 **백업(`.bak`)**을 자동 생성합니다.
- **리소스 경고**: 총 JVM 할당량이 물리 메모리의 **80%**를 초과할 경우, 운영 안정성을 위해 경고 메시지를 출력하고 수동 검토를 권장합니다.
- 서비스 재시작 시 `tkadmin`에 사전 보고하여 Watchdog 오탐을 방지합니다.
