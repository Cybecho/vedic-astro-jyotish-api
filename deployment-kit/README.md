# Jyotish API - Ready to Use Deployment

## 새로운 시스템에서 실행 방법

### 1. Docker 설치 (Ubuntu)
```bash
sudo apt update
sudo apt install docker.io docker-compose
sudo usermod -aG docker $USER
# 로그아웃 후 다시 로그인
```

### 2. 이미지 로드
```bash
# tar 파일을 이 폴더에 복사한 후
docker load -i jyotish-ready.tar
```

### 3. 실행
```bash
docker-compose -f docker-compose.ready.yml up -d
```

### 4. 확인
```bash
curl http://localhost:9393/api/ping
```

## 파일 목록
- `jyotish-ready.tar`: 빌드된 Docker 이미지
- `docker-compose.ready.yml`: 실행용 compose 파일
- `README.md`: 이 파일

## API 문서
- Swagger UI: http://localhost:9393/api/doc
- Health Check: http://localhost:9393/api/ping