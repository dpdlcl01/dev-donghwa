


## EC2 서버 초기화 및 프론트엔드 배포 전 준비 작업 요약

### 1. **불필요한 기존 프로젝트 폴더 삭제**
```bash
cd ~             # 홈 디렉토리로 이동
ls -al           # 모든 폴더/파일 목록 확인 (숨김 파일 포함)
```
- 이전 프로젝트 `backend`, `frontend` 폴더 삭제 시도 중 `Permission denied` 발생
- 원인: `docker_backend/db` 하위 디렉토리에 root 권한 파일 존재
    
#### 해결:
```bash
sudo rm -rf backend frontend
```

---

### 2. **기존 Docker 컨테이너/이미지 정리**
#### (컨테이너가 남아있을 경우를 대비해 추가 조치 예정자용)
```bash
# 실행 중인 모든 컨테이너 중지 및 삭제
sudo docker stop $(docker ps -aq)
sudo docker rm $(docker ps -aq)

# 사용되지 않는 이미지, 볼륨, 네트워크 삭제
sudo docker system prune -a
sudo docker volume prune
```

---

### 3. **현재 디스크 공간 확인 및 정리 불필요 확인**
```bash
df -h
```
- 총 디스크: 29GB
- 사용 중: 약 4.7GB (→ 충분한 여유 있음)

```bash
sudo du -h --max-depth=1 / | sort -hr | head -n 10
```
- `/usr` (2.0G), `/var` (805M), `/root` (2.0G) 등 사용 중
- `/root/swapfile`이 2GB 차지 중 → **Swap 영역**

---

### 4. **현재 홈 디렉토리 완전 정리 상태 확인**
```bash
ls -al ~ tree -L 2
```
→ `0 directories, 0 files` → 클린 상태 완료


---

### **반드시 필요한 기본 패키지**

|패키지|용도|확인 명령|
|---|---|---|
|**Docker**|백엔드 컨테이너 실행|`docker --version`|
|**Docker Compose**|여러 서비스 (Spring, DB 등) 컨테이너 조합 실행|`docker-compose --version` 또는 `docker compose version`|
|**Git**|GitHub에서 코드 클론|`git --version`|
|**OpenJDK** (Spring Boot를 로컬에서 빌드하는 경우)|`./gradlew build` 필요 시|`java -version` (Docker 안에서만 빌드하면 필요 없음)|

---

### **보통 설치하는 추가 유틸 (선택적)**

|패키지|용도|
|---|---|
|**tree**|디렉토리 구조 확인|
|**curl / wget**|테스트 요청, 외부 파일 다운로드|
|**vim / nano**|서버 내 파일 수정|
|**ufw / fail2ban**|보안 설정 (포트 오픈, 접속 제한 등)|
|**htop**|리소스 상태 실시간 확인 (CPU/RAM)|

---

### **Docker 설치 여부 확인**
```bash
# 도커 버전 확인
docker --version
# 도커 컴포즈 버전 확인
docker-compose --version
```

### **Docker설치**
```bash
# 1. 필수 패키지 설치
sudo apt update
sudo apt install ca-certificates curl gnupg

# 2. Docker GPG key 및 저장소 추가
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Docker 및 Docker Compose 설치
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. 설치 후 버전 확인
docker --version
docker compose version
```

### **도커 권한 설정 (ubuntu 계정으로 쓸 때)**
```bash
sudo usermod -aG docker $USER
```
>이후엔 **재로그인 또는 `reboot`** 하면 `sudo` 없이도 `docker` 명령 사용 가능


---

### **클론용 폴더 따로 만들기**
```bash
mkdir meet-u-backend
cd meet-u-backend
git clone https://github.com/dpdlcl01/meet-u-career-backend.git .
```
>주의: `.`(점)을 붙이면 현재 폴더에 바로 클론


![[Pasted image 20250422101139.png]]

![[Pasted image 20250422101246.png]]

---

### **주요 구성 확인**

|파일/폴더|설명|
|---|---|
|`Dockerfile`|Spring Boot 앱을 컨테이너 이미지로 빌드하기 위한 설정|
|`docker-compose.yml`|Docker 컨테이너 실행 설정 (Spring Boot + DB 등)|
|`build.gradle` & `gradlew`|Gradle 빌드 도구 구성 (Java 앱 실행용)|
|`src/`|실제 백엔드 애플리케이션 코드|
|`.gitignore`, `.gitattributes`, `README.md`|버전관리 및 문서용|
|`database/`, `docs/`|DB 초기화 스크립트나 문서 (추정)|

---

### **1. Docker 컴포즈 실행**





![[Pasted image 20250422101517.png]]


---

### **`.env` 파일은 일반적으로 `.gitignore`에 포함**

|이유|설명|
|---|---|
|🔐 보안|`.env` 파일에는 DB 비밀번호, JWT 시크릿 같은 민감한 정보가 포함됨|
|🔁 환경별 다름|개발/운영/테스트 환경마다 `.env` 값이 달라야 함|
|🧹 깨끗한 Git|개인 설정이 공유되지 않도록 방지|

### **`.env`는 직접 서버에서 만들어야 하는 파일**
- `.env.example` 파일은 Git에 포함시켜 예시만 공유
- `.env`는 각 환경(개발, 운영 등)에서 따로 작성


```bash
nano .env
```




![[Pasted image 20250422102114.png]]

![[Pasted image 20250422102255.png]]

![[Pasted image 20250422102939.png]]

![[Pasted image 20250422103045.png]]

![[Pasted image 20250422105730.png]]



