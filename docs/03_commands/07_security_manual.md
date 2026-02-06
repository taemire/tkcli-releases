# 보안 취약점 수동 점검 매뉴얼

> **✅ 자동 진단 안내 (v0.4.24+)**
>
> **v0.4.24** 버전부터 KISA 2026 주요정보통신기반시설 기술적 취약점 **전체 항목(85개)에 대한 자동 진단 및 조치**가 구현되었습니다.
> 따라서 본 매뉴얼의 대부분 항목은 `tkcli analyze security` 명령어로 자동 처리 가능하며, `tkcli fix security`를 통해 원클릭 조치가 가능합니다.
>
> 자세한 구현 명세는 [KISA 2026 보안 취약점 구현 명세서](../../SECURITY_IMPLEMENTATION_SPEC_2026.md)를 참조하십시오.
> 본 문서는 레거시 환경이나 자동 진단이 불가능한 특수 상황을 위한 참고용으로 유지됩니다.

`tkcli analyze security` 실행 결과, 드물게 '수동 점검 필요(MANUAL)' 상태로 표시되거나 교차 검증이 필요한 경우 본 가이드를 참조하십시오.

> **📌 참고**: 이 문서는 KISA 주요정보통신기반시설 기술적 취약점 분석 평가 상세 가이드를 기반으로 작성되었습니다.

---

## 1. OS 보안 점검 (U-01 ~ U-73)

### U-01. Root 계정 원격 접속 제한

| 항목 | 내용 |
|:---|:---|
| **분류** | 계정 관리 |
| **위험도** | 상 |
| **점검 대상** | SSH, Telnet 등 원격 접속 서비스 |

**점검 방법**
```bash
# SSH 설정 확인
grep -i "PermitRootLogin" /etc/ssh/sshd_config

# Telnet 설정 확인 (사용 시)
cat /etc/securetty
```

**양호 기준**
- `PermitRootLogin no` 설정

**조치 방법**
```bash
# SSH 설정 변경
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

### U-02. 패스워드 복잡성 설정

| 항목 | 내용 |
|:---|:---|
| **분류** | 계정 관리 |
| **위험도** | 상 |
| **점검 대상** | PAM 설정 |

**점검 방법**
```bash
# PAM 복잡성 설정 확인
cat /etc/pam.d/system-auth | grep pam_pwquality
cat /etc/security/pwquality.conf
```

**양호 기준**
- 영문, 숫자, 특수문자 조합
- 최소 8자 이상

**조치 방법**
```bash
# /etc/security/pwquality.conf 설정
minlen = 8
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
```

---

### U-03. 계정 잠금 임계값 설정

| 항목 | 내용 |
|:---|:---|
| **분류** | 계정 관리 |
| **위험도** | 상 |
| **점검 대상** | PAM faillock 설정 |

**점검 방법**
```bash
grep -E "deny=|faillock" /etc/pam.d/system-auth
grep -E "deny=|faillock" /etc/pam.d/password-auth
```

**양호 기준**
- 5회 이하 로그인 실패 시 계정 잠금

**조치 방법**
```bash
# faillock 설정 예시
auth required pam_faillock.so preauth deny=5 unlock_time=600
auth [default=die] pam_faillock.so authfail deny=5 unlock_time=600
```

---

### U-04. 패스워드 파일 보호

| 항목 | 내용 |
|:---|:---|
| **분류** | 계정 관리 |
| **위험도** | 상 |
| **점검 대상** | /etc/passwd, /etc/shadow |

**점검 방법**
```bash
# Shadow 파일 사용 여부 확인
awk -F: '$2 != "x" && $2 != "!" && $2 != "*" {print $1}' /etc/passwd
```

**양호 기준**
- 모든 계정의 패스워드가 /etc/shadow에 암호화되어 저장

---

### U-05. Root 이외의 UID 0 금지

| 항목 | 내용 |
|:---|:---|
| **분류** | 계정 관리 |
| **위험도** | 상 |
| **점검 대상** | /etc/passwd |

**점검 방법**
```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

**양호 기준**
- root 외 UID 0을 가진 계정이 없음

---

### U-06. root 홈/패스 권한

| 항목 | 내용 |
|:---|:---|
| **분류** | 파일 권한 |
| **위험도** | 상 |
| **점검 대상** | su 명령어 권한 |

**점검 방법**
```bash
ls -l /usr/bin/su
cat /etc/pam.d/su | grep pam_wheel
```

**양호 기준**
- su 명령어가 wheel 그룹으로 제한됨

---

### U-07 ~ U-09. 패스워드 정책 점검

| 항목 | 점검 대상 | 권장 설정 |
|:---|:---|:---|
| U-07 | 최소 길이 | 8자 이상 (`PASS_MIN_LEN 8`) |
| U-08 | 최대 사용기간 | 90일 이하 (`PASS_MAX_DAYS 90`) |
| U-09 | 최소 사용기간 | 1일 이상 (`PASS_MIN_DAYS 1`) |

**점검 방법**
```bash
cat /etc/login.defs | grep -E "PASS_MIN_LEN|PASS_MAX_DAYS|PASS_MIN_DAYS"
```

---

### U-10 ~ U-14. 계정 관리 점검

| ID | 점검 항목 | 점검 명령어 |
|:---|:---|:---|
| U-10 | 불필요한 계정 | `cat /etc/passwd \| grep -E "lp:\|uucp:\|nuucp:"` |
| U-11 | 관리자 그룹 | `grep "^root" /etc/group` |
| U-12 | 소유자 없는 그룹 | `find / -nogroup -print 2>/dev/null` |
| U-13 | 동일 UID | `awk -F: '{print $3}' /etc/passwd \| sort \| uniq -d` |
| U-14 | 사용자 Shell | `grep -E "^nobody\|^bin\|^daemon" /etc/passwd` |

---

### U-15. Session Timeout 설정

**점검 방법**
```bash
echo $TMOUT
cat /etc/profile | grep TMOUT
```

**양호 기준**
- TMOUT=600 이하 설정

**조치 방법**
```bash
echo "export TMOUT=600" >> /etc/profile
source /etc/profile
```

---

### U-16 ~ U-27. 파일/디렉터리 권한 점검

| ID | 점검 대상 | 권장 권한 | 점검 명령어 |
|:---|:---|:---|:---|
| U-16 | PATH 환경변수 | '.' 미포함 | `echo $PATH` |
| U-17 | 소유자 없는 파일 | 없음 | `find / -nouser -print 2>/dev/null` |
| U-18 | /etc/passwd | 644 (root) | `ls -l /etc/passwd` |
| U-19 | /etc/shadow | 400 (root) | `ls -l /etc/shadow` |
| U-20 | /etc/hosts | 600 (root) | `ls -l /etc/hosts` |
| U-21 | /etc/xinetd.conf | 600 (root) | `ls -l /etc/xinetd.conf` |
| U-22 | /etc/rsyslog.conf | 644 (root) | `ls -l /etc/rsyslog.conf` |
| U-23 | /etc/services | 644 (root) | `ls -l /etc/services` |
| U-24 | SUID/SGID | 불필요 파일 제거 | `find / -perm -4000 -print 2>/dev/null` |
| U-25 | 사용자 환경파일 | 적정 권한 | `ls -la ~/.bashrc ~/.profile` |
| U-26 | World Writable | 없음 | `find / -perm -2 -type f 2>/dev/null` |
| U-27 | /dev 디바이스 | 정상 파일만 | `find /dev -type f 2>/dev/null` |

---

### U-28 ~ U-35. 서비스 관리 점검 (1)

| ID | 점검 항목 | 점검 명령어 |
|:---|:---|:---|
| U-28 | .rhosts 파일 | `find / -name ".rhosts" 2>/dev/null` |
| U-29 | TCP Wrapper | `cat /etc/hosts.allow /etc/hosts.deny` |
| U-30 | hosts.lpd | `ls -l /etc/hosts.lpd` |
| U-31 | NIS 서비스 | `systemctl status ypserv ypbind` |
| U-32 | UMASK 설정 | `grep UMASK /etc/login.defs` |
| U-33 | 홈 디렉터리 권한 | `ls -ld /home/*` |
| U-34 | 홈 디렉터리 존재 | `awk -F: '{print $6}' /etc/passwd \| xargs ls -d 2>&1` |
| U-35 | 숨겨진 파일 | `find / -name ".*" -type f 2>/dev/null` |

---

### U-36 ~ U-51. 서비스 관리 점검 (2)

| ID | 점검 항목 | 양호 기준 | 점검 명령어 |
|:---|:---|:---|:---|
| U-36 | Finger 서비스 | 비활성화 | `systemctl status finger` |
| U-37 | Anonymous FTP | 비활성화 | `grep anonymous_enable /etc/vsftpd/vsftpd.conf` |
| U-38 | r 계열 서비스 | 비활성화 | `systemctl status rsh rlogin rexec` |
| U-39 | Cron 파일 권한 | 600 (root) | `ls -l /etc/cron.allow /etc/cron.deny` |
| U-40 | DoS 취약 서비스 | 비활성화 | `grep -E "echo\|discard\|daytime" /etc/xinetd.d/*` |
| U-41 | NFS 서비스 | 필요시만 활성화 | `systemctl status nfs` |
| U-42 | NFS 접근 통제 | 적정 설정 | `cat /etc/exports` |
| U-43 | automountd | 비활성화 | `systemctl status autofs` |
| U-44 | RPC 서비스 | 비활성화 | `rpcinfo -p localhost` |
| U-45 | NIS, NIS+ | 비활성화 | `systemctl status ypserv` |
| U-46 | tftp, talk | 비활성화 | `systemctl status tftp talk` |
| U-47 | Sendmail 버전 | 최신 버전 | `sendmail -d0.1 -bv root 2>&1 \| head -1` |
| U-48 | 스팸 릴레이 | 제한 설정 | `postconf \| grep smtpd_relay_restrictions` |
| U-49 | Sendmail 실행권한 | 제한 | `ls -l /usr/sbin/sendmail` |
| U-50 | DNS 버전 | 최신 버전 | `named -v` |
| U-51 | Zone Transfer | 제한 | `grep allow-transfer /etc/named.conf` |

---

### U-52 ~ U-70. 서비스 관리 점검 (3)

| ID | 점검 항목 | 양호 기준 | 점검 명령어 |
|:---|:---|:---|:---|
| U-52 | Apache 디렉터리 리스팅 | Indexes 제거 | `grep -r "Indexes" /etc/httpd/` |
| U-53 | Apache 프로세스 권한 | 비root 구동 | `ps -ef \| grep httpd` |
| U-54 | Apache 상위 디렉터리 | AllowOverride None | `grep AllowOverride /etc/httpd/conf/httpd.conf` |
| U-55 | Apache 불필요 파일 | 매뉴얼 제거 | `ls /var/www/html/manual/` |
| U-56 | Apache 링크 사용금지 | FollowSymLinks 제거 | `grep FollowSymLinks /etc/httpd/conf/httpd.conf` |
| U-57 | Apache 파일 업로드 | LimitRequestBody 설정 | `grep LimitRequestBody /etc/httpd/conf/httpd.conf` |
| U-58 | Apache 영역 분리 | 별도 디렉터리 | `grep DocumentRoot /etc/httpd/conf/httpd.conf` |
| U-59 | SSH 원격 접속 | 활성화 | `systemctl status sshd` |
| U-60 | FTP 서비스 | 필요시만 | `systemctl status vsftpd` |
| U-61 | FTP 계정 Shell | 제한 | `grep -E "ftp:\|ftpuser:" /etc/passwd` |
| U-62 | Ftpusers 파일 권한 | 640 (root) | `ls -l /etc/vsftpd/ftpusers` |
| U-63 | Ftpusers 설정 | root 포함 | `cat /etc/vsftpd/ftpusers` |
| U-64 | at 파일 권한 | 640 (root) | `ls -l /etc/at.allow /etc/at.deny` |
| U-65 | SNMP 서비스 | 필요시만 | `systemctl status snmpd` |
| U-66 | SNMP Community | public 금지 | `grep community /etc/snmp/snmpd.conf` |
| U-67 | 로그온 배너 | 설정 | `cat /etc/issue /etc/issue.net` |
| U-68 | NFS 설정파일 권한 | 644 (root) | `ls -l /etc/exports` |
| U-69 | expn, vrfy 제한 | 비활성화 | `postconf \| grep disable_vrfy_command` |
| U-70 | Apache 정보 숨김 | ServerTokens Prod | `grep ServerTokens /etc/httpd/conf/httpd.conf` |

---

### U-71 ~ U-73. 패치 및 로그 관리

| ID | 점검 항목 | 점검 방법 |
|:---|:---|:---|
| U-71 | 최신 보안 패치 | `yum check-update --security` 또는 `apt list --upgradable` |
| U-72 | 로그 정기 검토 | 로그 검토 정책 및 이력 확인 (문서/프로세스) |
| U-73 | 시스템 로깅 설정 | `cat /etc/rsyslog.conf` 정책 확인 |

---

## 2. WEB 보안 점검 (N-01 ~ N-06)

Nginx 웹 서버 보안 설정 점검 항목입니다.

### Nginx 설정 파일 위치

```bash
# 공통 경로
/etc/nginx/nginx.conf
/etc/nginx/conf.d/*.conf

# Tachyon TTS 환경
/usr/local/TACHYON/TTS40/nginx/conf/nginx.conf
```

---

### N-01. 버전 정보 노출 제한

| 항목 | 내용 |
|:---|:---|
| **분류** | 정보 노출 |
| **위험도** | 중 |

**점검 방법**
```bash
grep -i "server_tokens" /etc/nginx/nginx.conf
curl -I http://localhost 2>&1 | grep Server
```

**양호 기준**
- `server_tokens off;` 설정

**조치 방법**
```nginx
# nginx.conf의 http 블록 내
http {
    server_tokens off;
}
```

---

### N-02. HTTP 메서드 제한

| 항목 | 내용 |
|:---|:---|
| **분류** | 접근 제어 |
| **위험도** | 중 |

**점검 방법**
```bash
grep -i "limit_except" /etc/nginx/nginx.conf
curl -X OPTIONS http://localhost -I
```

**양호 기준**
- 불필요한 메서드(PUT, DELETE, TRACE 등) 차단

**조치 방법**
```nginx
location / {
    limit_except GET POST {
        deny all;
    }
}
```

---

### N-03. 디렉터리 리스팅 제한

| 항목 | 내용 |
|:---|:---|
| **분류** | 접근 제어 |
| **위험도** | 상 |

**점검 방법**
```bash
grep -i "autoindex" /etc/nginx/nginx.conf
```

**양호 기준**
- `autoindex off;` 또는 설정 없음 (기본값 off)

**조치 방법**
```nginx
# autoindex on; 설정 제거 또는 off로 변경
autoindex off;
```

---

### N-04. 파일 업로드 제한

| 항목 | 내용 |
|:---|:---|
| **분류** | 설정 관리 |
| **위험도** | 중 |

**점검 방법**
```bash
grep -i "client_max_body_size" /etc/nginx/nginx.conf
```

**양호 기준**
- 적정 크기로 제한 (예: 10M)

**조치 방법**
```nginx
http {
    client_max_body_size 10M;
}
```

---

### N-05. 숨겨진 파일 접근 제한

| 항목 | 내용 |
|:---|:---|
| **분류** | 접근 제어 |
| **위험도** | 중 |

**점검 방법**
```bash
grep -E "location.*\\\\." /etc/nginx/nginx.conf
curl http://localhost/.htaccess
```

**양호 기준**
- `.`으로 시작하는 파일 접근 차단

**조치 방법**
```nginx
location ~ /\. {
    deny all;
}
```

---

### N-06. 로그 설정 확인

| 항목 | 내용 |
|:---|:---|
| **분류** | 로그 관리 |
| **위험도** | 중 |

**점검 방법**
```bash
grep -E "access_log|error_log" /etc/nginx/nginx.conf
ls -l /var/log/nginx/
```

**양호 기준**
- access_log 및 error_log 모두 활성화

**조치 방법**
```nginx
http {
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;
}
```

---

## 3. DB 보안 점검 (D-01 ~ D-06)

MariaDB/MySQL 데이터베이스 보안 설정 점검 항목입니다.

### MariaDB 설정 파일 위치

```bash
# 공통 경로
/etc/my.cnf
/etc/mysql/my.cnf
/etc/my.cnf.d/*.cnf

# Tachyon TTS 환경
/usr/local/TACHYON/TTS40/mariadb/my.cnf
```

---

### D-01. 기본 계정 관리

| 항목 | 내용 |
|:---|:---|
| **분류** | 계정 관리 |
| **위험도** | 상 |

**점검 방법**
```sql
-- 익명 계정 확인
SELECT User, Host FROM mysql.user WHERE User='';

-- 테스트 데이터베이스 확인
SHOW DATABASES LIKE 'test%';
```

**양호 기준**
- 익명 계정 및 테스트 데이터베이스 없음

**조치 방법**
```sql
-- 익명 계정 삭제
DROP USER ''@'localhost';
DROP USER ''@'%';

-- 테스트 데이터베이스 삭제
DROP DATABASE IF EXISTS test;

-- 또는 mysql_secure_installation 실행
```

---

### D-02. Root 원격 접속 제한

| 항목 | 내용 |
|:---|:---|
| **분류** | 계정 관리 |
| **위험도** | 상 |

**점검 방법**
```sql
SELECT User, Host FROM mysql.user WHERE User='root';
```

**양호 기준**
- root 계정의 Host가 'localhost' 또는 '127.0.0.1'만 허용

**조치 방법**
```sql
-- 원격 root 접속 제거
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
FLUSH PRIVILEGES;
```

---

### D-03. 설정 파일 권한

| 항목 | 내용 |
|:---|:---|
| **분류** | 설정 관리 |
| **위험도** | 중 |

**점검 방법**
```bash
ls -l /etc/my.cnf
ls -l /var/lib/mysql/
```

**양호 기준**
- 설정 파일: 640 이하 (root/mysql 소유)
- 데이터 디렉터리: 750 (mysql 소유)

**조치 방법**
```bash
chmod 640 /etc/my.cnf
chown root:mysql /etc/my.cnf
chmod 750 /var/lib/mysql
chown mysql:mysql /var/lib/mysql
```

---

### D-04. 패스워드 복잡성

| 항목 | 내용 |
|:---|:---|
| **분류** | 패스워드 |
| **위험도** | 중 |

**점검 방법**
```sql
SHOW VARIABLES LIKE 'validate_password%';
-- 또는
SHOW VARIABLES LIKE 'simple_password_check%';
```

**양호 기준**
- 패스워드 검증 플러그인 활성화

**조치 방법**
```sql
-- MariaDB
INSTALL PLUGIN simple_password_check SONAME 'simple_password_check';

-- MySQL
INSTALL COMPONENT 'file://component_validate_password';
SET GLOBAL validate_password.policy = MEDIUM;
```

---

### D-05. 네트워크 바인딩 설정

| 항목 | 내용 |
|:---|:---|
| **분류** | 네트워크 |
| **위험도** | 상 |

**점검 방법**
```bash
grep bind-address /etc/my.cnf /etc/my.cnf.d/*
netstat -tlnp | grep 3306
```

**양호 기준**
- `bind-address = 127.0.0.1` 또는 특정 IP로 제한

**조치 방법**
```ini
# my.cnf [mysqld] 섹션
[mysqld]
bind-address = 127.0.0.1
```

---

### D-06. 감사 로그 설정

| 항목 | 내용 |
|:---|:---|
| **분류** | 로그 관리 |
| **위험도** | 중 |

**점검 방법**
```bash
grep -E "log_error|server_audit" /etc/my.cnf
ls -l /var/log/mysql/
```

**양호 기준**
- 에러 로그 활성화
- 감사 플러그인 사용 권장

**조치 방법**
```ini
# my.cnf [mysqld] 섹션
[mysqld]
log_error = /var/log/mysql/error.log

# 감사 플러그인 (선택)
plugin-load = server_audit=server_audit.so
server_audit_logging = ON
server_audit_file_path = /var/log/mysql/audit.log
```

---

## 4. 참고 자료

- [KISA 주요정보통신기반시설 기술적 취약점 분석·평가 상세 가이드](https://www.kisa.or.kr)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [Nginx Security Guide](https://nginx.org/en/docs/)
- [MariaDB Security Best Practices](https://mariadb.com/kb/en/security/)
