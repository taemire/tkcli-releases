# 7. 로깅 시스템

TKCLI는 자체 로그를 파일에 기록하며, TACHYON 운영 서비스의 로그 로테이션 상태를 확인하고 관리하는 기능을 제공합니다.

## 7.1 TKCLI 자체 로깅

### 로그 파일 위치 (우선순위)

1. `/usr/local/TACHYON/TTS40/logs/tkcli_YYYYMMDD.log` (기본)
2. `~/.tkcli/logs/tkcli_YYYYMMDD.log` (권한 없을 시)
3. `/tmp/tkcli_logs/tkcli_YYYYMMDD.log` (최종 대체)

### 로그 레벨 설정

`tkcli.ini` 파일에서 로그 레벨을 설정할 수 있습니다.

**설정 파일 위치:**
- TKCLI 실행 경로의 `tkcli.ini`

**설정 예시:**

```ini
basedir=/usr/local/TACHYON/TTS40
log_level=INFO
verbose=false
```

**로그 레벨:**
- `DEBUG`: 모든 로그 기록 (개발/디버깅용)
- `INFO`: 일반 정보 및 상태 변경 (기본값)
- `WARN`: 경고 메시지만
- `ERROR`: 오류만 기록

### 로그 확인

```bash
# 오늘 로그 확인
tail -f /usr/local/TACHYON/TTS40/logs/tkcli_$(date +%Y%m%d).log

# 최근 100줄
tail -100 /usr/local/TACHYON/TTS40/logs/tkcli_$(date +%Y%m%d).log

# 특정 날짜 로그
cat /usr/local/TACHYON/TTS40/logs/tkcli_20251207.log
```

### 로그 보관 정책

TKCLI 자체 로그는 **7일간 보관** 후 자동으로 삭제됩니다. 이 정책은 내부 `LogMaxAge` 상수로 관리됩니다.

---

## 7.2 서비스 로그 로테이션 관리 (logrotate 명령어)

TACHYON 운영 환경에서는 다양한 미들웨어와 서비스가 로그를 생성합니다. `tkcli logrotate` 명령어를 통해 모든 서비스의 로그 로테이션 상태를 한눈에 확인하고, 필요시 자동으로 설정할 수 있습니다.

### 빠른 사용법

```bash
# 전체 서비스 로그 로테이션 상태 확인
tkcli logrotate

# 특정 서비스만 확인
tkcli logrotate nginx

# 누락된 설정 자동 적용
tkcli logrotate --set
```

### 지원 서비스

| 서비스 유형 | 서비스 목록 | 로테이션 방식 |
| :--- | :--- | :--- |
| **미들웨어** | nginx, redis | 시스템 logrotate |
| **미들웨어** | opensearch, kafka, logstash, mariadb | 내부 관리 |
| **TACHYON** | Api, Auth, Manager, Stat, Batch, Report, Watchdog | Log4j2 |
| **관리 도구** | tkadmin | tkadmin.yml 설정 |
| **관리 도구** | tkcli | 자체 로테이션 |

### 상세 사용법

자세한 사용법과 출력 예시는 [서비스 관리 - 3.2.2 서비스 로그 로테이션](/03_commands/02_service_mgmt.md#322-서비스-로그-로테이션-상태-확인-및-설정-logrotate) 섹션을 참조하세요.


---

## 7.3 서비스별 로그 위치

| 서비스 | 로그 경로 |
| :--- | :--- |
| **TKCLI** | `$BASEDIR/logs/tkcli_YYYYMMDD.log` |
| **tkadmin** | `$BASEDIR/tkadmin/logs/` |
| **NGINX** | `$BASEDIR/nginx/logs/` |
| **Redis** | `$BASEDIR/redis/logs/` |
| **OpenSearch** | `$BASEDIR/opensearch/opensearch/logs/` |
| **Kafka** | `$BASEDIR/kafka/logs/` |
| **Java 서비스** | `$BASEDIR/logs/[ServiceName]/` |

> **참고**: `$BASEDIR`은 기본적으로 `/usr/local/TACHYON/TTS40`입니다.
