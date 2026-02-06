# 2. 기본 사용법

## 2.1 명령어 형식

```bash
tkcli [command] [subcommand] [arguments] [flags]
```

> **Tip**: 대화형 쉘 모드로 진입하려면 `--shell` (또는 `-s`) 플래그를 사용하십시오. 자세한 내용은 [4. 대화형 쉘 모드](./04_shell.md)를 참고하세요.

## 2.2 도움말 확인

```bash
# 전체 도움말
tkcli --help

# 특정 명령어 도움말
tkcli service --help
tkcli size --help
```

## 2.3 버전 확인

```bash
# 간단한 버전 정보
tkcli --version

#상세 버전 정보 (미들웨어 버전 포함)
tkcli version
```

## 2.4 환경 설정 및 진단 (env)

SELinux, 시스템 리미트(`ulimit`) 등 시스템 환경 설정을 확인하고 최적화합니다.

```bash
# SELinux 상태 확인
tkcli env selinux

# 시스템 및 서비스 리미트 상태 진단
tkcli env ulimit

# 권장 설정 자동 적용
tkcli env ulimit --set
```
