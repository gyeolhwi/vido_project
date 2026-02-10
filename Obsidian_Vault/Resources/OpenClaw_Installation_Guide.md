# OpenClaw 설치 가이드

## 1. WSL 및 Ubuntu 설치
1.  **WSL 설치**: 터미널에서 `wsl --install -d Ubuntu` 명령어를 사용하여 WSL과 Ubuntu를 한 번에 설치합니다.
2.  **Microsoft Store에서 Ubuntu 설치**: 이미 WSL이 설치되어 있다면, Microsoft Store에서 Ubuntu를 검색하여 설치할 수 있습니다.

## 2. 필수 패키지 설치

### 2.1. 시스템 업데이트 및 업그레이드
```bash
sudo apt update && sudo apt upgrade -y
```

### 2.2. Python 및 Git 설치
```bash
sudo apt install -y python3-pip python3-venv git
sudo apt install git -y  # Git이 설치되지 않은 경우
```

### 2.3. Node.js 및 npm 설치
```bash
sudo apt install nodejs npm -y
```
-   **버전 확인**:
    ```bash
    node --version
    npm --version
    ```

## 3. Node.js 버전 불일치 문제 해결

Node.js 설치 후 버전 불일치 문제가 발생할 경우, 다음 단계를 따릅니다.

### 3.1. 기존 Node.js 및 npm 제거
```bash
sudo apt remove nodejs npm -y
```

### 3.2. NodeSource 저장소 추가 (LTS 버전)
Node.js의 안정적인 LTS 버전을 설치하기 위해 NodeSource 저장소를 추가합니다.
```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
```

### 3.3. Node.js 및 npm 재설치
```bash
sudo apt install nodejs npm -y
```