# 🚀 MeetU 프로젝트 EC2 양방향 자동 배포 및 연결 가이드 (도메인: meet-u-career.com)

## 🧱 전체 구조 요약

- **프론트엔드**: Next.js + Nginx → EC2 (A) → `https://meetu-career.com`
    
- **백엔드**: Spring Boot + FastAPI → EC2 (B) → `https://api.meetu-career.com`
    
- 배포: GitHub Actions (`deploy.yml`) 자동화 사용 예정
    
- 연결: 프론트에서 백엔드 API 호출 시 `NEXT_PUBLIC_API_URL=https://api.meetu-career.com` 사용
    

---

## ✅ 프론트엔드 EC2 (A) 작업 내용

### 📁 1. 코드 클론 및 구성

```bash
mkdir meet-u-frontend
cd meet-u-frontend
git clone https://github.com/your-org/meet-u-frontend.git .
```

### ⚙️ 2. .env.local 설정

```env
NEXT_PUBLIC_API_URL=https://api.meetu-career.com
```

> GitHub Actions로 배포할 경우, 이 값은 Secret 또는 템플릿 주입으로 전달 필요

### 🐳 3. Dockerfile 예시

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install && npm run build
CMD ["npm", "run", "start"]
```

### 🌐 4. Nginx 설정 (예시)

```nginx
server {
  listen 80;
  server_name meet-u-career.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
}
```

### 🔐 5. HTTPS 적용 (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d meet-u-career.com
```

---

## ✅ 백엔드 EC2 (B) 작업 내용

### 📁 1. 코드 클론 및 디렉토리 준비

```bash
mkdir meet-u-backend
cd meet-u-backend
git clone https://github.com/your-org/meet-u-career-backend.git .
```

### ⚙️ 2. .env 파일 작성 (Spring Boot 환경 변수)

```env
DB_HOST=...
DB_PORT=...
DB_NAME=...
DB_USER=...
DB_PASSWORD=...
JPA_HIBERNATE_DDL_AUTO=update
```

### 🐘 3. Java 설치 + Gradle 빌드 준비

```bash
sudo apt update && sudo apt install openjdk-17-jdk -y
chmod +x gradlew
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
export PATH=$JAVA_HOME/bin:$PATH
./gradlew clean build -x test --no-daemon
```

### 🐳 4. Dockerfile 예시 (Multi-stage)

```dockerfile
FROM gradle:7.2.0-jdk as builder
WORKDIR /app
COPY . .
RUN chmod +x gradlew && ./gradlew clean build -x test --no-daemon

FROM openjdk:17
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 🌐 5. 백엔드 CORS 설정 (Spring Boot)

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
            .allowedOrigins("https://meetu-career.com")
            .allowedMethods("*")
            .allowCredentials(true);
}
```

### 🐍 6. FastAPI CORS 설정

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://meetu-career.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 🔐 7. HTTPS 적용 (선택)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d api.meetu-career.com
```

---

## 🔁 GitHub Actions 연동 시 체크리스트

|항목|프론트|백엔드|
|---|---|---|
|`.env` 외부 주입|✅ 필요|✅ 필요|
|SSH 키 등록|✅|✅|
|Actions deploy.yml 작성|✅|✅|
|프록시 포트, 보안 그룹 오픈|프론트: 80/443|백엔드: 8080 or 80|
|도메인 매핑|`meetu-career.com`|`api.meetu-career.com`|

---

## ✅ 예상 API 연동 흐름 예시

프론트 코드 (Next.js):

```ts
const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/jobposting/list`);
```

실제 요청:

```
GET https://api.meetu-career.com/api/jobposting/list
```

---

## 🏁 정리

- 프론트와 백엔드는 배포 방식은 별도지만, 연결은 도메인 주소로 해결
    
- 가장 중요한 건 "CORS 허용 + API 주소 맞추기 + 도메인 매핑"
    
- GitHub Actions 설정만 완료되면 EC2 직접 접속 없이 완전 자동화 가능!