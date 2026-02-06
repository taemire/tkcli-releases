# 1. 설치 및 설정

## 1.1 설치 및 설정

`tkcli`는 단일 바이너리 형태로 배포됩니다. 시스템 환경에 맞는 바이너리를 다운로드하여 서버의 적절한 경로(예: `/usr/local/bin`)에 배치합니다.

### 실행 권한 부여

바이너리를 내려받은 후 반드시 실행 권한을 부여해야 합니다.

```bash
# 실행 파일에 권한 부여 (예: /usr/local/bin 에 위치한 경우)
chmod +x /usr/local/bin/tkcli
```

### 원클릭 환경 설정 (추천)

권한 부여가 완료되면 다음 명령어를 실행하여 현재 사용 중인 셸(Bash, Zsh 등)에 `tkcli` 통합 설정을 자동으로 반영합니다.

```bash
tkcli -i
```

이 명령어는 다음 작업을 수행합니다:
- **자동 완성 등록**: 명령어 입력 시 `Tab` 키를 통한 자동 완성 기능 활성화
- **셸 프로필 업데이트**: `~/.bashrc` 또는 `~/.zshrc`에 필요한 환경 정보 추가

### 자동 완성 자동 설치

별도의 초기 설정을 진행하지 않고 `tkcli`를 처음 실행하면 자동 설치를 묻지만, 다음과 같이 명시적으로 설정 제거 및 재설치를 수행할 수도 있습니다.

```console
[root@localhost ~]# tkcli --uninstall
🗑️  tkcli 설정을 제거합니다...

📝 Bash 자동 완성 스크립트를 제거합니다...
  ✓ /etc/bash_completion.d/tkcli 제거됨
  ✓ 자동 완성 확인 플래그 제거됨
  ✓ 자동 완성 스크립트가 성공적으로 제거되었습니다

2026-01-17 19:31:26[SUCC] /root/.bashrc에서 레거시 설정이 제거되었습니다.
[root@localhost ~]#
[root@localhost ~]# tkcli --install -y
2026-01-17 19:31:33[SUCC] 설정 완료! /root/.bashrc에 적용되었습니다. 'source /root/.bashrc'를 실행하거나 쉘을 재시작하십시오.
[root@localhost ~]#
```

## 1.2 구성 관리

`tkcli`는 실행 파일과 동일한 위치에 생성되는 `tkcli.ini` 파일을 통해 주요 설정을 관리합니다.

| 설정 키 | 설명 | 기본값 |
| :--- | :--- | :--- |
| **basedir** | TACHYON 설치 루트 경로 | `/usr/local/TACHYON/TTS40` |
| **admin_api_url** | tkadmin 연동 서버 주소 | `http://127.0.0.1:13700` |
| **log_level** | 로그 기록 수준 (DEBUG, INFO, WARN, ERROR) | `DEBUG` |

### 1.2.1 tkadmin 연동 상세 로직

`tkcli`가 관리자 서버(`tkadmin`)로 작업 리포트를 보낼 때 사용하는 URL은 자동으로 구성됩니다.

1.  **연동 URL 결정 우선순위**:
    *   **1순위**: `tkcli.ini` 내의 `admin_api_url` 설정값.
    *   **2순위 (자동 감지)**: `tkcli` 실행 폴더에 `tkadmin.yml`이 존재할 경우, 해당 파일의 `port` 설정을 읽어 `http://127.0.0.1:{port}`로 자동 구성.
    *   **3순위**: 기본값 `http://127.0.0.1:13700` 사용.

2.  **통신 엔드포인트**:
    *   실제 보고는 설정된 주소 뒤에 `/tkadmin/api/monitor/report` 경로가 자동으로 붙어 수행됩니다.
    *   예: `admin_api_url`이 `http://192.168.0.100:13700`인 경우, 실제 통신 대상은 `http://192.168.0.100:13700/tkadmin/api/monitor/report`가 됩니다.

3.  **적용 시점**:
    *   설정 변경 후 즉시 적용되며, 별도의 서비스 재시작은 필요하지 않습니다 (CLI 성격상 매 수행 시 설정을 로드함).
