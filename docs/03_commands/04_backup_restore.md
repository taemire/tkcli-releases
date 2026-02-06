## 3.4 백업 및 복원 (backup/restore)


MariaDB 및 OpenSearch의 데이터를 백업하고 복원합니다. Cobra 네이티브 서브커맨드 구조로 구현되어 자동완성 및 도움말이 자연스럽게 지원됩니다.

### 사용법

```bash
# 백업
tkcli backup [mariadb|opensearch] [flags]

# 복원
tkcli restore [mariadb|opensearch] [args]
```

---

### 3.4.1 MariaDB 백업 (backup mariadb)

mariabackup을 사용하여 MariaDB 데이터베이스를 백업합니다.

**사용법:**
```bash
# 전체 백업 (기본)
tkcli backup mariadb

# 백업 유형 지정
tkcli backup mariadb --type full    # 전체 물리적 백업
tkcli backup mariadb --type incr    # 증분 백업
tkcli backup mariadb --type config  # 설정 데이터만 (로그 제외)
tkcli backup mariadb --type log     # 로그 포함 전체 백업

# 옵션 조합
tkcli backup mariadb --type full --retention 30 --compress
```

**플래그:**

| 플래그 | 축약 | 설명 | 기본값 |
| :--- | :--- | :--- | :--- |
| `--type` | `-t` | 백업 유형 (config, log, full, incr) | full |
| `--retention` | `-r` | 보관 일수 | 30 |
| `--compress` | `-c` | qpress 압축 사용 | true |
| `--incremental-basedir` | | 증분 백업 기준 디렉토리 | (자동 탐지) |

**출력 예시:**
```
📦 MariaDB 백업
[████████████░░░░░░░░] 60% | 백업실행   | 1m30s

✅ MariaDB Backup (full) completed successfully.
  Path: /opt/backup/mariadb/full_20260122_103000
  Size: 256.5 MB
  Duration: 2m30s
```

> 💡 **프로그레스바**: v0.4.23부터 TTY 환경에서 시각적 프로그레스바가 표시됩니다.
> 파이프(`|`) 또는 리다이렉트(`>`)로 출력하면 기존 텍스트 형식으로 표시됩니다.

---

### 3.4.2 OpenSearch 백업 (backup opensearch)

OpenSearch 스냅샷 저장소에 인덱스 스냅샷을 생성합니다.

**사용법:**
```bash
# 스냅샷 저장소 초기화 (최초 1회)
tkcli backup opensearch --init-repo

# 스냅샷 생성 (자동 이름)
tkcli backup opensearch

# 스냅샷 이름 지정
tkcli backup opensearch snap_20260116
```

**플래그:**

| 플래그 | 설명 |
| :--- | :--- |
| `--init-repo` | 스냅샷 저장소 초기화 (fs 타입) |

**출력 예시:**
```
2026-01-16 10:35:00 [INFO] OpenSearch 스냅샷을 생성합니다...
2026-01-16 10:35:00 [INFO] 스냅샷 이름: snap_20260116_103500
2026-01-16 10:35:10 [SUCC] 스냅샷이 성공적으로 생성되었습니다.
```

---

### 3.4.3 MariaDB 복원 (restore mariadb)

mariabackup 백업으로부터 MariaDB 데이터베이스를 복원합니다.

**사용법:**
```bash
# 전체 백업에서 복원
tkcli restore mariadb /backup/full_20260116

# 전체 + 증분 백업에서 복원
tkcli restore mariadb /backup/full_20260116 /backup/incr_20260117 /backup/incr_20260118
```

**출력 예시:**
```
2026-01-16 11:00:00 [INFO] MariaDB 복원을 시작합니다...
2026-01-16 11:00:00 [INFO] 전체 백업: /backup/full_20260116
2026-01-16 11:00:00 [WARN] MariaDB 서비스가 실행 중입니다. 복원 전 중지하세요.
2026-01-16 11:00:00 [INFO] 명령어: systemctl stop mariadb
```

> ⚠️ **주의**: MariaDB 복원 전 반드시 서비스를 중지해야 합니다.

---

### 3.4.4 OpenSearch 복원 (restore opensearch)

저장된 스냅샷으로부터 OpenSearch 인덱스를 복원합니다.

**사용법:**
```bash
tkcli restore opensearch snap_20260116
```

**출력 예시:**
```
2026-01-16 11:10:00 [INFO] OpenSearch 스냅샷 복원을 시작합니다...
2026-01-16 11:10:00 [INFO] 스냅샷: snap_20260116
2026-01-16 11:10:15 [SUCC] 스냅샷 복원이 완료되었습니다.
```
