# 8. 문제 해결

## 8.1 일반적인 문제

### 명령어를 찾을 수 없음

```bash
# PATH 확인
echo $PATH

# tkcli 위치 확인
which tkcli

# PATH에 추가 (임시)
export PATH=$PATH:/usr/local/bin

# PATH에 추가 (영구)
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

### 권한 오류

```bash
# 실행 권한 부여
chmod +x tkcli

# 소유권 변경
sudo chown $USER:$USER tkcli
```

### 데이터베이스 연결 실패

```bash
# MariaDB 상태 확인
tkcli service check | grep mariadb

# MariaDB 시작
tkcli service start mariadb

# 연결 테스트
mysql -u root -p
```

### 패키지 생성 시 버전 감지 실패

```bash
# MariaDB 버전 미지정 시 "최신 버전을 찾을 수 없습니다" 오류가 발생하는 경우

# 1. 인터넷 연결 확인
curl -s -o /dev/null -w "%{http_code}" https://downloads.mariadb.org
curl -s -o /dev/null -w "%{http_code}" https://archive.mariadb.org

# 2. 두 사이트 모두 차단된 경우 → 버전을 직접 지정
tkcli update package mariadb --version 10.11.16

# 3. v0.5.7 이상으로 업데이트 시 archive.mariadb.org 폴백 자동 지원
tkcli version
```

## 8.2 자동 완성이 작동하지 않음

```bash
# Completion 재설치
tkcli completion bash | sudo tee /etc/bash_completion.d/tkcli

# .bashrc 확인
grep tkcli ~/.bashrc
```

## 8.3 시스템 환경 문제

### SELinux 로 인한 서비스 오류

TACHYON 서비스가 정상적으로 시작되지 않거나 파일 쓰기 권한 오류가 발생하는 경우 SELinux 설정을 확인하십시오.

```bash
# SELinux 상태 확인
tkcli env selinux

# SELinux 를 Permissive 모드로 전환 (로그만 활성, 차단 해제)
tkcli env selinux --set permissive
```
