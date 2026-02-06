## 3.5 환경 구성 (env)


| 명령어 | 기능 | 주요 플래그 (옵션) |
|:---|:---|:---|
| `(기본)` | 전체 환경 설정 상태 조회 | `--set` : 각 항목의 권장값 자동 적용 |
| `selinux` | SELinux 설정 | `[mode]` : enforcing/permissive/disabled<br/>`--set` : 설정 적용 |
| `ulimit` | 시스템 리미트 설정 | `[value]` : Open Files 한도값 (예: 65535)<br/>`--set` : 설정 적용 |
| `firewall` | 방화벽 포트 설정 | `[port]` : 개방할 포트<br/>`--set` : 설정 적용 (SSH, 443 등 기본값 포함) |
| `ssh` | SSH 포트 설정 | `[port]` : 변경할 포트 번호<br/>`--set` : 서비스 재시작 및 방화벽 연동 |
| `web` | 웹 포트 설정 | `[port]` : 변경할 포트 번호<br/>`--set` : 서비스 재시작 및 방화벽 연동 |
| `db` | DB 포트 설정 | `[port]` : 변경할 포트 번호<br/>`--set` : 설정 적용<br/>`--yes` : 서비스 재시작 자동 승인 |
| `locale` | 시스템 Locale 설정 | `[lang]` : 언어코드 (ko/en)<br/>`--set` : 설정 적용 |


Tachyon 솔루션 구동에 최적화된 OS 환경(SELinux, Ulimit, Firewall, SSH)을 진단하고 설정합니다.
`tkcli env [subcommand]` 형식을 사용하여 각 항목별로 상세 제어가 가능하며, 인자 없이 `tkcli env` 실행 시 모든 항목에 대한 요약 점검을 수행합니다.

`--set` (또는 `-s`) 옵션은 설정을 실제로 적용할 때 사용하며, 인자 생략 시 각 환경에 맞는 **TACHYON 권장값**을 자동으로 선택합니다.

---

### 3.5.1 SELinux 설정 (selinux)

SELinux의 현재 실행 모드(런타임)와 영구 설정을 확인하고 최적화합니다.

**사용법:**
```bash
# 상태 확인
tkcli env selinux

# 권장값(disabled)으로 자동 설정
tkcli env selinux --set

# 특정 모드로 설정
tkcli env selinux disabled --set
tkcli env selinux permissive --set
```

**특징:**
- 이미 대상 모드로 설정되어 있는 경우 중복 작업을 생략합니다.
- `disabled` 모드 설정 시 영구 설정을 변경하며, 재부팅 전까지 즉시 효과를 위해 런타임에서 `permissive`를 적용합니다.

### 출력 예시

```
2025-12-24 21:03:14 [INFO] ### SELinux 구성 ###
--------------------------------------------
 항목                  상태                
--------------------------------------------
 런타임                Disabled            
 영구 설정             disabled            
--------------------------------------------
2025-12-24 21:03:14 [SUCC] 환경 확인 완료.
```

---

### 3.5.2 리미트 설정 (ulimit)

시스템 전체(`/etc/security/limits.conf`) 및 각 서비스별 `LimitNOFILE` 설정 상태를 진단합니다.

**사용법:**
```bash
# 리미트 상태 점검
tkcli env ulimit

# 권장값(65535)으로 시스템 리미트 자동 설정
tkcli env ulimit --set

# 특정 값으로 시스템 리미트 설정
tkcli env ulimit 99999 --set
```

**주의**: 시스템 리미트 설정 적용을 위해 **로그아웃 후 다시 로그인**하거나 시스템 재시작이 필요합니다.

### 출력 예시

```
2025-12-24 21:03:22 [INFO] ### 리미트(ulimit) 구성 및 서비스 진단 ###

 [ 시스템 전체 설정 (/etc/security/limits.conf) ]
---------------------------------------------------------------
 대상 (Domain)         유형        항목             값        
---------------------------------------------------------------
 *                     soft        nofile            65535     
 *                     hard        nofile            65535     
 root                  soft        nofile            65535     
 root                  hard        nofile            65535     
---------------------------------------------------------------

 [ 서비스별 개별 설정 (systemd LimitNOFILE) ]
-----------------------------------------------------------------------
 서비스명                        리미트 (NOFILE)       상태           
-----------------------------------------------------------------------
 mariadb                          32768                  OK             
 redis                            524288                 OK             
 ...
-----------------------------------------------------------------------
2025-12-24 21:03:22 [SUCC] 환경 확인 완료.
```

---

### 3.5.3 방화벽 설정 (firewall)

시스템 방화벽(`firewalld`) 상태를 확인하고 TACHYON 구동에 필요한 포트를 개방합니다.

**사용법:**
```bash
# 방화벽 상태 및 오픈 포트 진단
tkcli env firewall

# TACHYON 기본 포트(SSH, 443) 자동 개방
tkcli env firewall --set

# 특정 포트 추가 개방
tkcli env firewall 8080 --set
```

**설정 로직:**
- **인자 생략**: 현재 시스템의 SSH 포트를 자동 감지하여 443 포트와 함께 개방합니다.
- **포트 지정**: 입력한 포트를 `tcp` 프로토콜로 개방합니다.

### 출력 예시

```
2025-12-24 21:03:35 [INFO] ### 방화벽(Firewall) 구성 진단 ###

 [ 방화벽 상태 ]
--------------------------------------------
 서비스/존             상태                
--------------------------------------------
 firewalld             비활성 (Inactive)   
 iptables              비활성 (Inactive)   
--------------------------------------------
2025-12-24 21:03:35 [SUCC] 환경 확인 완료.
```

---

### 3.5.4 SSH 포트 설정 (ssh)

OpenSSH 서비스의 포트 번호를 점검하고 변경합니다.

**사용법:**
```bash
# 현재 SSH 포트 확인
tkcli env ssh

# SSH 포트를 2022로 변경
tkcli env ssh 2022 --set

# SSH 포트를 기본값(22)으로 원복
tkcli env ssh --set
```

**⚠️ 자동 구성 안내:**
- **방화벽 연동**: `--set`을 통해 포트 변경 시, 변경된 포트가 **방화벽에서 자동으로 허용**됩니다. (접속 차단 방지)
- **서비스 자동 재시작**: 설정 변경 후 `sshd` 서비스가 자동으로 재시작되어 즉시 적용됩니다.

### 출력 예시

```
2025-12-24 21:03:46 [INFO] ### SSH (OpenSSH) 구성 진단 ###

 [ ### SSH (OpenSSH) 구성 진단 ### ]
--------------------------------------------
 항목                  SSH 포트            
--------------------------------------------
 OpenSSH                22                  
--------------------------------------------

 '--set <포트>'를 사용하여 SSH 포트를 변경하세요
2025-12-24 21:03:46 [SUCC] 환경 확인 완료.
```

---

### 3.5.5 웹 포트 설정 (web)

TACHYON 웹 서비스(Nginx HTTPS)의 리스닝 포트를 점검하고 변경합니다.

**사용법:**
```bash
# 현재 웹 포트 확인
tkcli env web

# HTTPS 포트를 8443으로 변경
tkcli env web 8443 --set

# 기본값(443)으로 원복
tkcli env web --set
```

**v0.4.20+ 확장 옵션:**

| 옵션 | 설명 |
|:---|:---|
| `--mode unified/separated` | 통합/분리 모드 전환 |
| `--console <port>` | 분리 모드 시 관리콘솔 포트 (기본: 8443) |
| `--backend <port>` | 분리 모드 시 백엔드 API 포트 (기본: 443) |
| `--dry-run` | 변경 사항 미리보기 (실제 적용 안 함) |
| `--restore` | 백업에서 nginx.conf 복원 |

**확장 사용 예시:**
```bash
# 분리 모드로 전환
tkcli env web --mode separated --set

# 분리 모드 + 포트 지정
tkcli env web --mode separated --console 9443 --backend 10443 --set

# 미리보기 (변경 없이 확인만)
tkcli env web --mode separated --dry-run

# 백업에서 복원
tkcli env web --restore
```

**동작 상세:**
- **Nginx 설정 변경**: `/etc/nginx/nginx.conf` 또는 `conf.d/*.conf` 내의 `listen ... ssl` 지시어를 찾아 포트를 변경합니다.
- **방화벽 연동**: 변경된 포트를 방화벽에서 즉시 허용합니다.
- **서비스 제어**: 변경 사항 적용을 위해 Nginx 서비스를 자동으로 재시작합니다.
- **자동 백업**: 설정 변경 전 nginx.conf를 자동으로 백업합니다 (.bak).
- **복원 기능**: `--restore` 옵션으로 마지막 백업에서 설정을 복원할 수 있습니다.

---

### 3.5.6 DB 포트 설정 (db)

MariaDB 데이터베이스의 접속 포트를 변경하고, 이를 참조하는 모든 TACHYON 서비스 및 관리 도구의 설정을 일괄 동기화합니다.

**사용법:**
```bash
# 현재 DB 포트 확인
tkcli env db

# MariaDB 포트를 13306으로 변경 (자동 승인)
tkcli env db 13306 --set --yes

# 기본값(3306)으로 원복
tkcli env db 3306 --set
```

**⚠️ 자동화 범위 및 영향도:**
이 명령어는 단순한 포트 변경 이상의 광범위한 시스템 동기화 작업을 수행합니다.

1.  **시스템 포트 변경**: `/etc/my.cnf.d/server.cnf`의 MariaDB 포트를 변경합니다.
2.  **서비스 참조 일괄 갱신**:
    - `TACHYON_HOME` 내의 **모든 서비스 설정 파일**(`*.yml_dev`)을 검색하여 `jdbc:mariadb://` 연결 문자열의 포트를 치환합니다.
    - **전역 설정(`app_info.properties`)**의 `db_port` 항목을 함께 갱신하여 시스템 전체의 일관성을 유지합니다.
3.  **방화벽 및 재시작**:
    - 변경된 포트를 방화벽에 등록합니다.
    - MariaDB, API, Manager, Watchdog, **tkadmin** 등 영향을 받는 **모든 서비스를 순차적으로 재시작**합니다.
    - 운영 중 서비스 중단(Downtime)이 발생하므로, `--yes` 옵션이 없으면 사용자 승인을 요청합니다.

**⚠️ 주의사항 (Nginx Stream)**:
만약 Nginx가 DB 포트(3306 등)를 프록시하기 위해 `stream` 블록(`nginx.conf` 하단)을 사용 중이라면, 이 부분의 포트는 기본적으로 **자동으로 변경되지 않습니다**.

**v0.4.20+ Nginx Stream 자동 동기화:**

| 옵션 | 설명 |
|:---|:---|
| `--sync-stream` | nginx stream upstream 포트 자동 업데이트 |
| `--dry-run` | 변경 사항 미리보기 (실제 적용 안 함) |

```bash
# DB 포트 변경 + nginx stream 자동 동기화
tkcli env db 3307 --set --sync-stream

# 미리보기 (변경 없이 확인만)
tkcli env db 3307 --dry-run
```

`--sync-stream` 옵션을 사용하면 nginx.conf의 `upstream tachyon_mariadb` 블록 포트가 함께 변경됩니다.

**출력 예시:**
```
2026-01-08 16:30:00 [INFO] Setting Database port to 13306...
2026-01-08 16:30:00 [INFO] Updating SERVICE configurations (TACHYON_HOME)...
2026-01-08 16:30:01 [INFO]   Updated: api.yml_dev
2026-01-08 16:30:01 [INFO]   Updated: manager.yml_dev
...
2026-01-08 16:30:02 [INFO] Updating nginx stream upstream 'tachyon_mariadb' to port 13306...
2026-01-08 16:30:02 [SUCC] Updated nginx stream upstream for MariaDB
2026-01-08 16:30:05 [INFO] Restarting TACHYON services...
2026-01-08 16:30:10 [SUCC] Database port updated to 13306
```

### 3.5.7 DB 포트 감지 및 연결 문제 해결

`tkcli`는 MariaDB 연결 시 다음 순서로 포트를 자동으로 감지합니다:

1.  **MariaDB 설정 파일**: `my.cnf` 또는 `server.cnf`의 `[mysqld]` 섹션에 명시된 `port` 값.
2.  **Nginx Stream 설정**: `tkcli env db --sync-stream`으로 동기화된 Nginx `upstream tachyon_mariadb` 포트.
3.  **기본값**: `3306`.

**문제 상황: DB 포트를 변경했는데 `tkcli`가 접속하지 못하는 경우**

1.  **증상**: `dial tcp 127.0.0.1:3306: connect: connection refused` 에러 발생.
2.  **원인**: MariaDB 설정 파일에 포트가 명시되어 있지 않고, Nginx 설정과도 동기화되지 않아 기본값(3306)을 시도했기 때문입니다.
3.  **해결 방법**:
    *   **방법 A (권장)**: `tkcli env db <PORT> --sync-stream` 명령을 실행하여 Nginx 설정을 업데이트하면, `tkcli`가 이를 감지하여 해당 포트로 접속합니다.
    *   **방법 B**: MariaDB 설정 파일(`server.cnf` 등)의 `[mysqld]` 섹션에 `port=<PORT>`를 명시합니다.

> **Tip**: `tkcli env db` 명령은 포트 변경과 동시에 Nginx Stream 설정을 동기화하므로, 안전하게 포트를 변경할 수 있는 가장 좋은 방법입니다.


---

### 환경 구성 권장 값 요약

| 항목 | 권장 값 | 명령어 예시 |
| :--- | :--- | :--- |
| **SELinux** | `disabled` | `tkcli env selinux --set` |
| **Ulimit** | `65535` | `tkcli env ulimit --set` |
| **Firewall** | `ssh, 443, 3306` | `tkcli env firewall --set` |
| **SSH Port** | `22` | `tkcli env ssh --set` |
| **Web Port** | `443` | `tkcli env web --set` |
| **DB Port** | `3306` | `tkcli env db --set` |
| **Locale** | `ko_KR.UTF-8` | `tkcli env locale ko --set` |

---

### 3.5.8 Locale 설정 (locale)

시스템 Locale(언어 및 문자셋) 설정을 확인하고 변경합니다.

**배경:**
RockyLinux 9 Non-GUI(Minimal) 모드로 설치 시 `LANG=C` 또는 `LANG=en_US`로 기본 설정되어 제품 콘솔에서 한글이 깨질 수 있습니다. `ko_KR.UTF-8` 또는 `en_US.UTF-8`로 변경하여 해결합니다.

**사용법:**
```bash
# 현재 locale 상태 확인
tkcli env locale

# 한글 locale로 설정
tkcli env locale ko --set

# 영어 locale로 설정
tkcli env locale en --set
```

**특징:**
- 현재 설정이 이미 `*.UTF-8`이면 변경하지 않고 Skip 합니다.
- 언어팩(`glibc-langpack-ko`)이 미설치된 경우 수동 설치 안내를 출력합니다.
- 폐쇄망 환경 지원: 자동 패키지 설치 없이 경고만 출력합니다.

**출력 예시:**

```
2026-01-21 22:52:34 [INFO] ### Locale (언어/문자셋) 구성 ###
-------------------------------------------------
 항목                  상태                     
-------------------------------------------------
 시스템 Locale         LANG=ko_KR.UTF-8         
 VC Keymap             kr                       
 한글 언어팩           ✓ 설치됨                
-------------------------------------------------
2026-01-21 22:52:34 [SUCC] 환경 확인 완료.
```

**언어팩 미설치 시:**
```
2026-01-21 22:00:00 [WARN] 언어팩(glibc-langpack-ko)이 설치되어 있지 않습니다.
  ℹ ISO 파일이나 로컬 레포지토리를 통해 설치하세요:
    rpm -ivh glibc-langpack-ko-*.rpm
```


---

### 3.5.9 Nginx ACL 설정 (web acl)

Nginx 웹 서버의 접근 제어 목록(ACL)을 관리하여 특정 IP만 관리콘솔에 접근하도록 제한합니다.
IP 주소(IPv4) 또는 CIDR 대역(예: `192.168.1.0/24`)을 지원합니다.

> ⚠️ **영향 범위**: ACL은 **관리콘솔(프론트엔드)** 접근만 제어합니다. 백엔드 API 서비스(443 포트)에는 영향을 주지 않으므로, API 호출이나 서비스 연동에는 지장이 없습니다.

**사용법:**

```bash
# 1. ACL 정책 확인
tkcli env web acl list

# 2. 관리자 IP 추가 (필수)
tkcli env web acl add 203.0.113.10
tkcli env web acl add 192.168.100.0/24

# 3. ACL 기능 활성화 (모든 접속 차단, 허용된 IP만 접속 가능)
# 주의: 반드시 관리자 IP를 먼저 추가(add)한 후 활성화(enable)해야 합니다.
tkcli env web acl enable

# [기타]
# 특정 IP 허용 삭제
tkcli env web acl remove 203.0.113.10

# ACL 기능 비활성화 (모든 IP 접속 허용)
tkcli env web acl disable

# 설정 테스트 (문법 검사 및 적용 상태 확인)
tkcli env web acl test
```

**주의사항:**
- ACL 기능을 활성화(`enable`)하면 기본적으로 **모든 접속이 차단(deny all)**되며, `add` 명령으로 추가한 IP만 접속이 허용됩니다.
- 로컬호스트(`127.0.0.1`)는 내부 통신을 위해 자동으로 허용 규칙에 포함되지 않으므로 필요 시 명시적으로 추가하거나 `location` 구조에 따라 예외 처리됩니다(현재 구현에서는 명시적 추가 권장).
- 설정 변경(`add`, `remove`, `enable`, `disable`) 후에는 반드시 Nginx를 재시작해야 적용됩니다.
  - `tkcli service restart nginx`

**설정 파일 동작:**
- 기본적으로 `/app/nginx/conf/nginx.conf` (또는 설치 경로) 파일의 `http` 또는 `location` 블록 내에 `allow`/`deny` 지시어를 주입합니다.
- `--config` 옵션으로 대상 설정 파일 경로를 직접 지정할 수 있습니다.
