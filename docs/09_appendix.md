# 부록 A: 서비스 로깅 레벨 진단 기술 명세

본 부록은 `tkcli service loglevel` 명령어가 각 서비스의 로깅 레벨을 진단하기 위해 참조하는 설정 파일의 위치와 파싱 로직을 상세히 기술합니다. 시스템 엔지니어는 본 가이드를 참조하여 `tkcli`가 리포팅하는 정보의 근거를 파악하고, 필요시 수동으로 설정을 검증할 수 있습니다.

---

## 1. 개요

`tkcli`는 각 서비스의 설정 파일을 텍스트 기반으로 직접 파싱하거나 YAML 파서를 사용하여 로깅 관련 지시어를 추출합니다. 각 서비스는 설치 기반 디렉토리(`${BaseDir}`)를 기준으로 상대 경로를 탐색합니다.

## 2. 미들웨어 서비스 (Middleware)

| 서비스 | 참조 파일 경로 | 진단 방식 (Parsing Logic) |
| :--- | :--- | :--- |
| **NGINX** | `nginx/conf/nginx.conf` | **1. Error Log**: `error_log` 지시어의 마지막 인자를 추출하여 레벨(DEBUG, INFO 등)을 표시합니다.<br>**2. Access Log**: NGINX는 액세스 로그에 레벨 개념을 사용하지 않으므로, `access_log off;` 지시어 존재 여부만 확인하여 **ON/OFF** 상태로 리포팅합니다.<br>표시 형식: `레벨 (Access: ON/OFF)` |
| **Redis** | `redis/conf/redis.conf` 또는 `redis/redis.conf` | `loglevel` 지시어의 우측 값을 추출합니다.<br>표시 예: `loglevel notice` → **NOTICE** |
| **MariaDB** | `mariadb/my.cnf` | 파일 존재 여부만 확인하며, 기본값으로 **INFO (Default)**를 표시합니다.<br>*(MariaDB는 단일 레벨 설정보다 상세 로깅 설정을 주로 사용함)* |

## 3. TACHYON Java 서비스

TACHYON의 모든 Java 서비스(`api`, `auth`, `manager`, `stat`, `batch`, `report`, `watchdog`)는 다음의 우선순위에 따라 파일을 탐색합니다.

### 3.1 설정 파일 탐색 우선순위
1. `[svc]/conf/[svc].yml_dev` (YAML) - **최우선 순위 (Plaintext 원본)**
2. `[svc]/conf/[svc].yml` (YAML)
3. `[svc]/conf/logback-spring.xml` (XML)
4. `[svc]/conf/log4j2.xml` (XML)
5. `[svc]/conf/application.yml` (YAML)

### 3.2 파싱 기술 명세

#### YAML 파싱 (`.yml`, `.yml_dev`)
`gopkg.in/yaml.v3` 라이브러리를 사용하여 다음 계층의 값을 추출합니다:
- **경로**: `logging` > `level` > `root`
- **구조 예시**:
  ```yaml
  logging:
    level:
      root: INFO  # <-- 추출 대상
  ```

#### XML 파싱 (`.xml`)
문자열 패턴 매칭을 통해 `<root>` 태그의 `level` 속성을 추출합니다:
- **정규식/패턴**: `<root level="([^"]+)">`
- **구조 예시**:
  ```xml
  <root level="DEBUG">  <!-- <-- level 속성값 추출 -->
      <appender-ref ref="CONSOLE" />
  </root>
  ```

## 4. 관리 도구 (Management Tools)

| 서비스 | 참조 파일 경로 | 진단 방식 (Parsing Logic) |
| :--- | :--- | :--- |
| **tkadmin** | `${BaseDir}/tkadmin.yml` | YAML 파싱을 통해 `logging.level.root` 값을 추출합니다. |
| **tkcli** | `${BaseDir}/tkcli.ini` | 설정 파일 내 `log_level` 키값을 확인합니다. (미설정 시 `INFO`) |

---

## 5. 진단 상태(Status) 설명

| 상태 | 의미 | 대응 방법 |
| :--- | :--- | :--- |
| **OK** | 설정 파일을 정상적으로 찾고 파싱함 | - |
| **CONFIG NOT FOUND** | 지정된 경로에 설정 파일이 존재하지 않음 | 해당 서비스의 설치 경로를 확인하십시오. |
| **INVALID** | 파일은 발견했으나 로깅 레벨 지시어 부재 | 설정 파일 내에 `root` 레벨 설정이 누락되었는지 확인하십시오. |
| **-** | 진단 불가능 | 특정 서비스를 지정하여 상세 진단을 수행하십시오. |

> **참고**: `tkcli`는 Java 서비스의 경우 `yml_dev` 파일을 수정하고 이를 `yml`로 복사하여 반영하는 관리 워크플로우를 권장합니다.

---

## 6. 시스템 환경 설정 진단 및 구성 명세 (env)

`tkcli env` 명령어는 TACHYON 운영에 필수적인 커널 및 시스템 환경 변수를 진단하고 최적화합니다.

### 6.1 SELinux

| 구분 | 진단/참조 로직 |
| :--- | :--- |
| **현재 실행 모드** | `getenforce` 명령어를 우선 실행하며, 부재 시 `sestatus` 결과에서 `Current mode`를 추출합니다. |
| **영구 설정 모드** | `/etc/selinux/config` 파일의 `SELINUX=` 지시어 값을 직접 파싱합니다. |
| **변경 프로세스** | `--set` 실행 시 `setenforce`를 통한 즉시 반영과 `config` 수정을 통한 영구 반영을 동시에 수행합니다. (Disabled 설정 시 재부팅 안내) |

### 6.2 시스템 자원 제한 (Ulimit)

| 구분 | 진단/참조 로직 |
| :--- | :--- |
| **커널 전체 제한** | `/etc/security/limits.conf` 파일을 파싱하여 도메인별 `nofile` 설정을 수집합니다. |
| **서비스별 제한** | `systemctl show -p LimitNOFILE --value [SVC]` 명령을 통해 systemd 유닛에 구동 시 할당된 실제 값을 조회합니다. |
| **TACHYON 블록** | `--set` 기능을 통한 설정 시, 기존 파일 훼손을 방지하기 위해 `# --- TACHYON LIMITS START ---` 마커를 사용한 전용 블록을 관리합니다. |

### 6.3 방화벽 (Firewall)

- **활성 상태 감지**: `systemctl is-active`를 통해 `firewalld`와 `iptables` 서비스의 동작 여부를 동시에 체크합니다.
- **포트 목록 추출**: `firewalld`가 활성 상태일 때 `firewall-cmd --list-ports` 및 `--list-services` 결과를 파싱하여 제공합니다.
- **자동 허용 로직**: `--set` 인자 생략 시 TACHYON 관리 포트(443)와 현재 운영 중인 SSH 포트를 자동 탐지하여 개방합니다.

### 6.4 SSH 설정 및 연동

- **포트 감지**: `/etc/ssh/sshd_config` 파일에서 주석처리 되지 않은 `Port` 지시어를 탐색합니다. (지시어 부재 시 22번 표준 포트로 간주)
- **변경 및 연동 워크플로우**:
    1. `/etc/ssh/sshd_config` 내 Port 번호 수정
    2. `firewall-cmd --add-port=[NEW_PORT]/tcp`를 통해 신규 포트 방화벽 영구 허용 (`--permanent`)
    3. `firewall-cmd --reload`를 실행하여 방화벽 설정 즉시 반영
    4. `systemctl restart sshd`를 실행하여 설정 완료

---

## 7. JVM 메모리 설정 및 자동 최적화 기술 명세 (service jvm)

`tkcli service jvm` 명령어는 JVM 기반 서비스의 힙 메모리를 진단하고 최적화하기 위해 다음 기술 명세를 따릅니다.

### 7.1 서비스별 설정 파일 및 파싱 로직

| 서비스 | 설정 파일 경로 | 추출/변경 로직 |
| :--- | :--- | :--- |
| **OpenSearch** | `opensearch/opensearch/config/jvm.options` | `-Xms`, `-Xmx`로 시작하는 라인을 직접 파싱합니다. |
| **Kafka** | `kafka/bin/kafka-server-start.sh` | `KAFKA_HEAP_OPTS` 환경 변수 할당 라인을 정규식으로 추출합니다. |
| **Logstash** | `logstash/logstash-kafka-os/config/jvm.options` | `-Xms`, `-Xmx` 라인을 직접 파싱합니다. |
| **TACHYON 서비스** | `[svc]/bin/tachyon-[svc].sh` | 실행 스크립트 내 `java` 명령 인자 중 `-Xms`, `-Xmx` 패턴을 추출합니다. |
| **Watchdog** | `bin/tachyon-watchdog.sh` | 동일한 쉘 스크립트 정규식 파싱 방식을 사용합니다. |

### 7.2 자동 최적화 (auto) 알고리즘 상세

`--set auto` 옵션 실행 시, 다음 가중치 모델을 기반으로 시스템 총 메모리를 배분합니다.

1. **최적화 공식**: `할당량 = (전체메모리 - 기본 OS 점유분) * 서비스별 가중치`
2. **서비스별 가중치(Weight)**:
    - `opensearch`: 20%
    - `kafka`: 10%
    - `api`: 5%
    - `manager`, `auth`, `stat`, `batch`, `report`: 각 3~5%
    - `watchdog`: 2% (최소 128MB 보장)
3. **제한 정책(Guardrails)**:
    - **최대 상한**: 단일 서비스 힙 크기는 **16GB**를 초과하지 않도록 캡핑(Capping)합니다.
    - **최소 하한**: 서비스 가시성 확보를 위해 최소 **256MB~1GB** (서비스별 상이) 하한선을 적용합니다.
    - **에이전트 수량 가중치**: `--agents` 값이 클수록 API 및 Manager 서비스의 비중을 동적으로 상향 조정합니다.

### 7.3 안전 관리 프로세스

- **백업 정책**: 설정 변경 전 `${ConfigFile}.bak` 파일을 생성하며, 이미 백업이 존재하는 경우 덮어쓰지 않고 타임스탬프를 활용하여 이력을 보호합니다.
- **리소스 경고**: 총 JVM 할당량이 물리 메모리의 **80%**를 초과할 경우, 운영 안정성을 위해 경고 메시지를 출력하고 수동 검토를 권장합니다.

---

## 8. OS 메모리 관리 및 점유 방식 기술 명세 (Memory Management)

`tkcli info` 명령어가 리포팅하는 메모리 사용량이 사용자가 체감하는 수치보다 높게 확인되는 현상에 대한 기술적 근거를 설명합니다.

### 8.1 Linux vs Windows 메모리 관리 기조 차이

| 항목 | Linux (운영 환경) | Windows (비교 환경) |
| :--- | :--- | :--- |
| **핵심 철학** | "사용되지 않는 메모리는 낭비다 (Free RAM is wasted RAM)" | 사용자 가시성 중심의 리소스 유연 배분 |
| **캐시 활용** | 디스크 I/O 속도 향상을 위해 남은 모든 메모리를 **Page Cache**로 전용 | Superfetch 등을 통한 선독처리를 수행하나 리포팅 시에는 '여유'로 간주하는 경향 |
| **사용량 계산** | `Used = Total - Free` (Free는 순수 물리 빈 공간) | `Used = Total - Available` (즉시 사용 가능한 캐시 포함 여유분) |

### 8.2 페이지 캐시(Page Cache)와 과점유 현황

1. **과점유 현상 (Memory Occupancy)**:
    - Linux 커널은 파일을 읽거나 쓸 때 해당 데이터를 물리 메모리에 보관(Page Cache)합니다. 이는 이후 동일 파일 접근 시 느린 디스크 대신 빠른 메모리에서 즉시 응답하기 위함입니다.
    - TACHYON 서비스(OpenSearch, MariaDB 등)는 대량의 데이터를 지속적으로 처리하므로 시간이 지날수록 캐시 점유율이 높아져 `Free` 메모리가 0에 가깝게 표시될 수 있습니다.

2. **사용자 가시성**:
    - `tkcli info`는 운영체제 커널의 물리적 점유 상태를 직접 리포팅하므로, 이러한 캐시 영역을 모두 포함하여 '사용 중'으로 표시합니다.

3. **실질 가용성 (Available Memory)**:
    - 이 캐시 영역은 애플리케이션이 실제 메모리 할당을 요청하면 OS가 즉시 비워줄 수 있는 **삭제 가능한(Reclaimable)** 공간입니다. 따라서 점유율 수치가 높더라도 시스템의 실질적인 작업 수행 능력에는 지장을 주지 않습니다.

### 8.3 결론 및 판정 기준

`tkcli info`에서 확인되는 높은 메모리 점유율은 대부분 **I/O 성능 향상을 위한 Linux 커널의 정상적인 최적화 결과**입니다. 시스템 리소스가 실제로 부족한지 여부는 점유율 수치보다는 다음 지표를 기준으로 판단하십시오.

- **스왑(Swap) 발생 여부**: 물리 메모리가 부족하여 디스크 압축/교환이 빈번하게 일어나는지 확인.
- **OOM Killer 로그**: 커널 로그(`dmesg`)에 메모리 고갈로 인한 프로세스 강제 종료 이력이 있는지 확인.

> **TIP**: Linux 환경에서 실질적으로 '사용 가능한' 양을 알고 싶다면 유휴 공간(Free)이 아닌 가용 공간(Available) 지표를 참조해야 합니다. (tkcli는 향후 버전에서 가용 지표를 포함하도록 개선 예정입니다.)

---

## 9. 로그 분석 시스템 기술 명세 (analyze)

본 부록은 `tkcli analyze` 명령어가 MariaDB 및 OpenSearch에서 로그 데이터를 분석하기 위해 사용하는 내부 동작 원리와 쿼리 구조를 상세히 기술합니다. 시스템 엔지니어는 본 가이드를 참조하여 `tkcli`의 분석 결과를 검증하거나, CLI 도구 없이 직접 수동 분석을 수행할 수 있습니다.

---

### 9.1 개요 및 데이터 소스

`tkcli analyze` 명령어는 TACHYON 시스템의 두 가지 데이터 소스를 쿼리합니다:

| 데이터 소스 | 용도 | 연결 정보 소스 |
| :--- | :--- | :--- |
| **OpenSearch** | 실시간 로그 집계, 이상 탐지 | `app_info.properties` → `opensearch_ip`, `opensearch_port`, `opensearch_id`, `opensearch_pw` |
| **MariaDB** | 로그 테이블 증감 추세 분석 | `app_info.properties` → `db_ip`, `db_port`, `db_user`, `db_password` |

### 9.2 OpenSearch 인덱스 명명 규칙

TACHYON 시스템은 매체제어 이벤트 로그를 **월별 인덱스**로 분리하여 OpenSearch에 저장합니다.

**인덱스 이름 형식:**
```
{SPCODE}_{YEAR}_{MONTH}
```

**예시:**
- 기관 코드(`SPCODE`): `00133fkfkg`
- 2026년 1월: `00133fkfkg_2026_01`
- 2025년 12월: `00133fkfkg_2025_12`

**SPCODE 확인 방법:**
```bash
# app_info.properties 파일에서 확인
grep "spcode" /usr/local/TACHYON/TTS40/conf/app_info.properties_dev

# 또는 데이터베이스 이름 확인 (동일한 값)
tkcli info
```

**인덱스 목록 조회 (수동):**
```bash
# OpenSearch API로 직접 조회
curl -s "http://localhost:9200/_cat/indices?v" | grep -E "00133fkfkg_"
```

---

### 9.3 OpenSearch 필드 매핑 (Field Mapping)

매체제어 로그 인덱스의 주요 필드 구조입니다:

| 필드명 | 타입 | 설명 | 예시 값 |
| :--- | :--- | :--- | :--- |
| `server_dt` | `keyword` | 서버 수신 일자 (YYYY-MM-DD) | `2026-01-15` |
| `client_dt` | `keyword` | 클라이언트 이벤트 발생 일자 | `2026-01-15` |
| `puid` | `keyword` | 에이전트 고유 식별자 (UUID) | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| `process_nm` | `keyword`/`text` | 실행 프로세스 경로 | `C:\Windows\Explorer.EXE` |
| `file_nm` | `keyword`/`text` | 접근 대상 파일/경로 | `E:\Documents\report.xlsx` |
| `event_type` | `keyword` | 이벤트 유형 코드 | `READ`, `WRITE`, `COPY` |

> **참고**: `.keyword` 서픽스가 붙은 필드는 정확 일치 검색 및 집계에 사용됩니다. 집계 쿼리에서는 반드시 `process_nm.keyword` 형태로 사용하십시오.

---

### 9.4 media-top: 프로세스+경로별 Top N 분석

**목적**: 어떤 프로세스가 어떤 경로에 가장 많이 접근했는지 파악

#### 9.4.1 OpenSearch DSL 쿼리

`tkcli analyze media-top --month 2026-01 --limit 20` 실행 시 내부적으로 다음 DSL을 사용합니다:

```json
{
  "size": 0,
  "query": {
    "range": {
      "server_dt": {
        "gte": "2026-01-01",
        "lt": "2026-02-01"
      }
    }
  },
  "aggs": {
    "by_process_path": {
      "composite": {
        "size": 20,
        "sources": [
          { "process": { "terms": { "field": "process_nm.keyword" } } },
          { "path": { "terms": { "field": "file_nm.keyword" } } }
        ]
      }
    }
  }
}
```

#### 9.4.2 수동 실행 방법

```bash
# OpenSearch에서 직접 쿼리 실행
curl -X POST "http://localhost:9200/00133fkfkg_2026_01/_search" \
  -H "Content-Type: application/json" \
  -u "admin:admin" \
  -d '{
    "size": 0,
    "query": {
      "range": {
        "server_dt": { "gte": "2026-01-01", "lt": "2026-02-01" }
      }
    },
    "aggs": {
      "by_process_path": {
        "composite": {
          "size": 10,
          "sources": [
            { "process": { "terms": { "field": "process_nm.keyword" } } },
            { "path": { "terms": { "field": "file_nm.keyword" } } }
          ]
        }
      }
    }
  }' | jq '.aggregations.by_process_path.buckets'
```

#### 9.4.3 응답 구조 및 해석

```json
{
  "buckets": [
    {
      "key": {
        "process": "C:\\Windows\\Explorer.EXE",
        "path": ""
      },
      "doc_count": 42349
    },
    {
      "key": {
        "process": "C:\\Windows\\system32\\svchost.exe",
        "path": ""
      },
      "doc_count": 5966
    }
  ]
}
```

- **`key.process`**: 실행 프로세스 경로
- **`key.path`**: 접근 대상 파일/경로 (빈 문자열은 경로 미지정 이벤트)
- **`doc_count`**: 해당 조합의 발생 건수

---

### 9.5 agent-anomaly: 이상 에이전트 탐지

**목적**: 평균 대비 과도하게 로깅하는 에이전트 식별

#### 9.5.1 OpenSearch DSL 쿼리

```json
{
  "size": 0,
  "query": {
    "range": {
      "server_dt": {
        "gte": "2026-01-01",
        "lt": "2026-02-01"
      }
    }
  },
  "aggs": {
    "by_agent": {
      "terms": {
        "field": "puid.keyword",
        "size": 10000
      }
    },
    "agent_stats": {
      "extended_stats_bucket": {
        "buckets_path": "by_agent._count"
      }
    }
  }
}
```

#### 9.5.2 핵심 집계 설명

| 집계 유형 | 필드 | 설명 |
| :--- | :--- | :--- |
| `terms` | `by_agent` | 에이전트(PUID)별 로그 건수 집계 |
| `extended_stats_bucket` | `agent_stats` | 버킷 통계 계산 (평균, 표준편차 등) |

#### 9.5.3 이상 탐지 알고리즘

`tkcli`는 다음 로직으로 이상 에이전트를 판정합니다:

```
이상 여부 = (에이전트 로그 건수 / 전체 평균) >= 임계치(기본 5.0)
```

**판정 기준:**
- **🔴 심각**: 평균 대비 10배 이상 (Multiplier ≥ 10.0)
- **⚠️ 주의**: 평균 대비 5~10배 (Multiplier ≥ 5.0)

#### 9.5.4 수동 분석 절차

```bash
# 1. 에이전트별 통계 조회
curl -X POST "http://localhost:9200/00133fkfkg_2026_01/_search" \
  -H "Content-Type: application/json" \
  -u "admin:admin" \
  -d '{
    "size": 0,
    "aggs": {
      "by_agent": {
        "terms": { "field": "puid.keyword", "size": 10000 }
      },
      "agent_stats": {
        "extended_stats_bucket": { "buckets_path": "by_agent._count" }
      }
    }
  }' > /tmp/agent_stats.json

# 2. 평균값 확인
jq '.aggregations.agent_stats.avg' /tmp/agent_stats.json

# 3. 표준편차 확인
jq '.aggregations.agent_stats.std_deviation' /tmp/agent_stats.json

# 4. 상위 이상 에이전트 식별 (평균의 5배 이상 필터링)
AVG=$(jq '.aggregations.agent_stats.avg' /tmp/agent_stats.json)
THRESHOLD=$(echo "$AVG * 5" | bc)
jq --argjson th $THRESHOLD \
  '.aggregations.by_agent.buckets | map(select(.doc_count >= $th)) | sort_by(-.doc_count) | .[0:10]' \
  /tmp/agent_stats.json
```

---

### 9.6 process-top: 프로세스별 로그 집계

**목적**: 가장 많은 로그를 발생시키는 프로세스 순위 파악

#### 9.6.1 OpenSearch DSL 쿼리

```json
{
  "size": 0,
  "query": {
    "range": {
      "server_dt": {
        "gte": "2026-01-01",
        "lt": "2026-02-01"
      }
    }
  },
  "aggs": {
    "by_process": {
      "terms": {
        "field": "process_nm.keyword",
        "size": 20,
        "order": { "_count": "desc" }
      }
    }
  }
}
```

#### 9.6.2 수동 실행 (OpenSearch Dashboards)

OpenSearch Dashboards에서 다음 시각화를 생성할 수 있습니다:

1. **Visualize** → **Lens** 또는 **Vertical Bar Chart**
2. **Index Pattern**: `00133fkfkg_*`
3. **Metrics**: Count
4. **Buckets**: Terms, `process_nm.keyword`, Top 20

---

### 9.7 recommend: 데이터 기반 분석 추천

**목적**: MariaDB 로그 테이블의 증가율을 분석하여 심층 분석이 필요한 항목 자동 추천

#### 9.7.1 MariaDB 쿼리 구조

**현재 월 및 전월 건수 조회:**
```sql
-- 2026년 1월 건수
SELECT COUNT(*) 
FROM TPU_ISTAT_MEDIA_EVENT_LOG 
WHERE server_dt >= '2026-01-01' AND server_dt < '2026-02-01';

-- 2025년 12월 건수 (전월)
SELECT COUNT(*) 
FROM TPU_ISTAT_MEDIA_EVENT_LOG 
WHERE server_dt >= '2025-12-01' AND server_dt < '2026-01-01';
```

#### 9.7.2 분석 대상 테이블

| 테이블명 | 분석 유형 | 권장 명령어 |
| :--- | :--- | :--- |
| `TPU_ISTAT_MEDIA_EVENT_LOG` | 매체제어 로그 | `tkcli analyze media-top` |
| `TPU_ISTAT_EVENT_LOG` | 이벤트 로그 | `tkcli analyze process-top` |
| `TPU_ISTAT_SYS_LOG` | 시스템 로그 | `tkcli analyze process-top` |

#### 9.7.3 증가율 계산 공식

```
증가율(%) = (현재 월 건수 - 전월 건수) / 전월 건수 × 100
```

#### 9.7.4 우선순위 판정 기준

| 우선순위 | 조건 | 표시 아이콘 | 권장 조치 |
| :--- | :--- | :--- | :--- |
| **1 (폭증)** | 증가율 ≥ 100% | 🔴 | 즉시 분석 필요 |
| **2 (급증)** | 50% ≤ 증가율 < 100% | ⚠️ | 원인 분석 권장 |
| **3 (정상)** | 증가율 < 50% | ✅ | 정상 범위 |

#### 9.7.5 수동 분석 스크립트

```bash
#!/bin/bash
# recommend 분석 수동 수행

DBNAME="00133FKFKG"
MONTH="2026-01"
PREV_MONTH="2025-12"

# 매체제어 로그 증가율 분석
CURRENT=$(mysql -u tachyon -p'password' $DBNAME -N -e \
  "SELECT COUNT(*) FROM TPU_ISTAT_MEDIA_EVENT_LOG 
   WHERE server_dt >= '${MONTH}-01' AND server_dt < DATE_ADD('${MONTH}-01', INTERVAL 1 MONTH);")

PREVIOUS=$(mysql -u tachyon -p'password' $DBNAME -N -e \
  "SELECT COUNT(*) FROM TPU_ISTAT_MEDIA_EVENT_LOG 
   WHERE server_dt >= '${PREV_MONTH}-01' AND server_dt < '${MONTH}-01';")

if [ "$PREVIOUS" -gt 0 ]; then
  RATE=$(echo "scale=1; ($CURRENT - $PREVIOUS) * 100 / $PREVIOUS" | bc)
  echo "매체제어 로그: 현재=${CURRENT}, 전월=${PREVIOUS}, 증가율=${RATE}%"
  
  if (( $(echo "$RATE >= 100" | bc -l) )); then
    echo "🔴 폭증 - 즉시 분석 필요: tkcli analyze media-top --month $MONTH"
  elif (( $(echo "$RATE >= 50" | bc -l) )); then
    echo "⚠️ 급증 - 원인 분석 권장: tkcli analyze media-top --month $MONTH"
  else
    echo "✅ 정상 범위"
  fi
fi
```

---

### 9.8 db-trend: DB 데이터 증가 추세 분석

**목적**: LOG 테이블의 기간별(1주, 1개월, 1분기, 1년) 누적 증가 추세 파악

#### 9.8.1 MariaDB 쿼리 구조

**기간별 누적 건수 조회:**
```sql
-- 1주 전 시점까지의 누적 건수 (Start Total)
SELECT COUNT(*) FROM TPU_ISTAT_MEDIA_EVENT_LOG 
WHERE server_dt < DATE_SUB(CURDATE(), INTERVAL 7 DAY);

-- 현재까지의 누적 건수 (End Total)
SELECT COUNT(*) FROM TPU_ISTAT_MEDIA_EVENT_LOG 
WHERE server_dt < DATE_ADD(CURDATE(), INTERVAL 1 DAY);
```

#### 9.8.2 증가율 계산

```
기간 증가 건수 = End Total - Start Total
증가율(%) = (End Total - Start Total) / Start Total × 100
```

#### 9.8.3 추세 판정 기준

| 추세 | 증가율 조건 | 표시 |
| :--- | :--- | :--- |
| **폭증 (위험)** | ≥ 50% | 🔴 |
| **급증 (주의)** | 20% ~ 50% | 🟠 |
| **완만 (정상)** | 5% ~ 20% | ↗ |
| **유지 (안정)** | 0% ~ 5% | → |
| **감소** | < 0% | ↘ |

#### 9.8.4 수동 분석 쿼리 예시

```sql
-- 지난 1주간 증가 추세
SET @start_total = (SELECT COUNT(*) FROM TPU_ISTAT_MEDIA_EVENT_LOG 
                    WHERE server_dt < DATE_SUB(CURDATE(), INTERVAL 7 DAY));
SET @end_total = (SELECT COUNT(*) FROM TPU_ISTAT_MEDIA_EVENT_LOG);
SET @increase = @end_total - @start_total;
SET @rate = IF(@start_total > 0, (@increase / @start_total) * 100, 100);

SELECT '최근 1주' AS period, 
       @start_total AS start_cnt, 
       @end_total AS end_cnt, 
       @increase AS increase,
       CONCAT(ROUND(@rate, 1), '%') AS growth_rate,
       CASE 
         WHEN @rate >= 50 THEN '🔴 폭증 (위험)'
         WHEN @rate >= 20 THEN '🟠 급증 (주의)'
         WHEN @rate >= 5 THEN '↗ 완만 (정상)'
         WHEN @rate > 0 THEN '→ 유지 (안정)'
         ELSE '↘ 감소'
       END AS trend;
```

---

### 9.9 OpenSearch 연결 설정 확인

`tkcli analyze`가 사용하는 OpenSearch 연결 정보를 확인하는 방법입니다:

#### 9.9.1 설정 파일 위치

```bash
# 기본 설정 파일 (암호화된 상태일 수 있음)
cat /usr/local/TACHYON/TTS40/conf/app_info.properties

# 평문 원본 (운영 환경에 따라 다름)
cat /usr/local/TACHYON/TTS40/conf/app_info.properties_dev*
```

#### 9.9.2 주요 설정 항목

```properties
opensearch_ip=127.0.0.1
opensearch_port=9200
opensearch_id=admin
opensearch_pw=admin
```

#### 9.9.3 tkcli.ini 개별 설정 (우선순위 높음)

`tkcli.ini` 파일에서 OpenSearch URL을 별도로 지정할 수 있습니다:

```ini
[opensearch]
url=http://localhost:9200
```

---

### 9.10 분석 결과 파일 저장 형식

`--output` 옵션 사용 시 다음 형식으로 저장됩니다:

#### 9.10.1 CSV 형식 (`.csv`)

```csv
RANK,PROCESS,PATH,COUNT
1,C:\Windows\Explorer.EXE,,42349
2,C:\Windows\system32\svchost.exe,,5966
3,C:\Program Files\Windows Defender\MsMpEng.exe,E:\,2611
```

- **인코딩**: UTF-8 with BOM (Excel 호환)
- **구분자**: 콤마(`,`)
- **줄바꿈**: CRLF

#### 9.10.2 TXT 형식 (`.txt`)

테이블 형태의 텍스트 파일로 저장됩니다.

---

### 9.11 트러블슈팅

#### 9.11.1 OpenSearch 연결 실패

```
OpenSearch 호출 실패: dial tcp 127.0.0.1:9200: connect: connection refused
```

**원인 및 조치:**
1. OpenSearch 서비스 확인: `systemctl status opensearch`
2. 포트 리스닝 확인: `ss -tlnp | grep 9200`
3. 방화벽 확인: `firewall-cmd --list-ports`

#### 9.11.2 인덱스 없음 오류

```
해당 기간에 데이터가 없습니다.
```

**원인 및 조치:**
1. 인덱스 존재 확인: `curl "http://localhost:9200/_cat/indices?v" | grep {spcode}`
2. 날짜 범위 확인: 지정한 월에 데이터가 있는지 검증
3. SPCODE 확인: 인덱스명의 기관 코드가 올바른지 확인

#### 9.11.3 MariaDB 연결 실패

```
DB 연결 실패: dial tcp 127.0.0.1:3306: connect: connection refused
```

**원인 및 조치:**
1. MariaDB 서비스 확인: `systemctl status mariadb`
2. 접속 정보 확인: `app_info.properties`의 `db_ip`, `db_port`, `db_user`, `db_password`

---

### 9.12 고급 분석 쿼리 레퍼런스

#### 9.12.1 특정 프로세스 상세 분석

```bash
# Explorer.EXE의 접근 경로 상세 분석
curl -X POST "http://localhost:9200/00133fkfkg_2026_01/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "query": {
      "bool": {
        "must": [
          { "range": { "server_dt": { "gte": "2026-01-01", "lt": "2026-02-01" } } },
          { "match": { "process_nm": "Explorer.EXE" } }
        ]
      }
    },
    "aggs": {
      "by_path": {
        "terms": { "field": "file_nm.keyword", "size": 50 }
      }
    }
  }' | jq '.aggregations.by_path.buckets'
```

#### 9.12.2 특정 에이전트 이벤트 상세 조회

```bash
# 이상 에이전트의 최근 이벤트 100건 조회
curl -X POST "http://localhost:9200/00133fkfkg_2026_01/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 100,
    "query": {
      "bool": {
        "must": [
          { "term": { "puid.keyword": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" } }
        ]
      }
    },
    "sort": [{ "server_dt": "desc" }],
    "_source": ["server_dt", "process_nm", "file_nm", "event_type"]
  }' | jq '.hits.hits[]._source'
```

#### 9.12.3 일별 로그 발생량 추이

```bash
# 일별 로그 건수 집계
curl -X POST "http://localhost:9200/00133fkfkg_2026_01/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "by_day": {
        "terms": {
          "field": "server_dt",
          "size": 31,
          "order": { "_key": "asc" }
        }
      }
    }
  }' | jq '.aggregations.by_day.buckets'
```

---

> **참고**: 본 문서에 기재된 쿼리는 TACHYON 4.0 시스템의 기본 스키마를 기준으로 작성되었습니다. 실제 운영 환경에서 필드명이나 인덱스 구조가 다를 수 있으므로, 스키마 확인 후 사용하십시오.
