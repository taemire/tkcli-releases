# 5. 전역 옵션

모든 명령어에서 사용 가능한 전역 옵션입니다.

## 5.1 기본 디렉토리 설정 (`--basedir`, `-b`)

기본 설치 경로가 아닌 다른 경로를 사용할 경우 설정합니다.  
설정값은 `tkcli.ini` 파일에 영구적으로 저장됩니다.

```bash
# 경로 변경 및 저장
tkcli -b /custom/path info

# 또는
tkcli --basedir=/custom/path info
```

## 5.2 디버그 모드 (`--debug`, `-d`)

디버깅을 위한 상세 출력을 활성화합니다.

```bash
tkcli -d service check
tkcli --debug info
```

## 5.3 설정 제거 (`--uninstall`, `-u`)

TKCLI가 설치한 자동 완성 스크립트와 설정 파일을 제거합니다.

```bash
tkcli --uninstall
# 또는
tkcli -u
```

**제거 대상:**
- Bash completion 스크립트 (`/etc/bash_completion.d/tkcli` 등)
- Completion 체크 플래그 파일 (`~/.tkcli_completion_checked`)
- 레거시 `.bashrc` 설정 블록

## 5.4 버전 정보 (`--version`, `-v`)

간단한 버전 정보를 출력합니다.

```bash
tkcli -v
# 또는
tkcli --version
# 출력: tkcli version 0.3.19
```

## 5.5 쉘 모드 진입 (`--shell`, `-s`)

인프라 관리 및 테스트를 위해 인터랙티브 쉘 모드로 명시적으로 진입합니다.

```bash
tkcli -s
# 또는
tkcli --shell
```
