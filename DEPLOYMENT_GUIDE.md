# Jyotish API Docker 배포 완전 가이드

## 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [현재 빌드 방식의 문제점](#현재-빌드-방식의-문제점)
3. [Swiss Ephemeris 빌드가 필요한 이유](#swiss-ephemeris-빌드가-필요한-이유)
4. [해결방안](#해결방안)
5. [단계별 실행 가이드](#단계별-실행-가이드)
6. [다른 시스템으로 이미지 전송](#다른-시스템으로-이미지-전송)
7. [개선된 Dockerfile](#개선된-dockerfile)

## 프로젝트 개요

**Jyotish API**는 베다 점성술(Vedic Astrology) 계산을 위한 REST API입니다.

### 기술 스택
- **Framework**: Symfony 5.4 (PHP 7.4)
- **컨테이너**: Docker + Nginx
- **천체계산**: Swiss Ephemeris (C 라이브러리)
- **문서화**: Swagger/OpenAPI

### 주요 기능
- Varga Charts (D1~D60 분할 차트)
- 행성 위치 계산 (9개 행성)
- Panchanga 계산 (인도 달력 시스템)
- Dasha 계산 (행성 주기)
- Yoga 분석 (점성술적 조합)

## 현재 빌드 방식의 문제점

현재 README에서 제시하는 설치 방법:

```bash
# 1. 저장소 클론
git clone https://github.com/teal33t/jyotish-api.git
cd jyotish-api

# 2. Docker 컨테이너 빌드 및 시작
docker-compose up --build -d

# 3. 컨테이너 내부에서 Swiss Ephemeris 컴파일 (추가 작업 필요!)
docker-compose exec jyotish_api bash -c "cd swetest/src && make clean && make && chmod 777 swetest/src && chmod +x swetest/src/swetest"

# 4. API 확인
curl http://localhost:9393/api/ping
```

### 문제점
- **매번 수동 빌드 필요**: 새로운 시스템마다 3단계 추가 명령어 실행
- **시간 소요**: C 라이브러리 컴파일 시간
- **복잡성**: 사용자가 Docker 내부 명령어 실행 필요
- **실패 가능성**: 빌드 도구나 환경 문제로 실패 위험
- **플랫폼별 차이**: macOS에서는 정상 작동하지만 Linux에서 권한 문제 발생 가능

## Swiss Ephemeris 빌드가 필요한 이유

### Swiss Ephemeris란?
- **C로 작성된 천체 계산 라이브러리**
- 스위스 취리히 대학에서 개발
- 고정밀 천체 위치 계산 (행성, 달, 소행성 등)
- 점성술 소프트웨어의 표준 라이브러리

### 왜 컨테이너 내부에서 빌드하는가?
1. **플랫폼 의존성**: 각 시스템/아키텍처에 맞는 네이티브 컴파일 필요
2. **바이너리 호환성**: x86_64, ARM, 다른 리눅스 배포판별 차이
3. **최적화**: 타겟 시스템에 맞는 컴파일러 최적화
4. **라이브러리 의존성**: 시스템별 C 라이브러리 차이

### 빌드 과정 분석

```bash
# Makefile 주요 내용
CFLAGS = -g -O9 -Wall  	# Linux gcc 최적화
CC=cc

SWEOBJ = swedate.o swehouse.o swejpl.o swemmoon.o swemplan.o swepcalc.o sweph.o swepdate.o swephlib.o swecl.o swehel.o

# 정적 라이브러리 생성
libswe.a: $(SWEOBJ)
	ar r libswe.a $(SWEOBJ)

# 실행 파일 생성
swetest: swetest.o libswe.a
	$(CC) $(OP) -o swetest swetest.o -L. -lswe -lm
```

**생성되는 파일들:**
- `libswe.a`: 정적 라이브러리 (아카이브)
- `swetest`: 실행 파일 (PHP에서 호출)
- `*.o`: 오브젝트 파일들

## 해결방안

### 방안 1: 현재 빌드된 상태를 이미지로 저장 (즉시 해결)
한 번 빌드 완료 후 해당 상태를 Docker 이미지로 저장하여 재사용

### 방안 2: Dockerfile 개선 (장기적 해결)
빌드 과정을 Dockerfile에 포함시켜 자동화

### 방안 3: Multi-stage Build (프로덕션용)
빌드 환경과 실행 환경을 분리하여 최종 이미지 크기 최소화

## 단계별 실행 가이드

### Step 1: 현재 상태 확인

```bash
# 현재 실행 중인 컨테이너 확인
docker ps

# 컨테이너 이름이 jyotish_api인지 확인
# 없다면 먼저 빌드:
# docker-compose up --build -d
# docker-compose exec jyotish_api bash -c "cd swetest/src && make clean && make && chmod 777 swetest/src && chmod +x swetest/src/swetest"
```

### Step 2: 빌드된 컨테이너를 이미지로 저장

```bash
# 현재 컨테이너를 이미지로 커밋
docker commit jyotish_api jyotish-api:ready-to-use

# 생성된 이미지 확인
docker images | grep jyotish

# 예상 출력:
# jyotish-api    ready-to-use    abc123def456    2 minutes ago    1.2GB
```

### Step 3: 기존 컨테이너 정리 (선택사항)

```bash
# 기존 컨테이너 중지 및 제거
docker-compose down

# 또는 개별 중지
docker stop jyotish_api
docker rm jyotish_api
```

### Step 4: 새 이미지로 실행 테스트

```bash
# 저장된 이미지로 직접 실행
docker run -d -p 9393:9393 --name jyotish_api_test jyotish-api:ready-to-use

# Linux 환경에서 권한 문제 사전 해결 (macOS에서는 불필요)
docker exec jyotish_api_test bash -c \
"mkdir -p /var/www/api/var/{cache/dev,log} \
&& chown -R www-data:www-data /var/www/api/var \
&& chmod -R 777 /var/www/api/var"

# API 동작 확인
curl http://localhost:9393/api/ping

# 예상 응답:
# {"pong":"success"}

# 테스트 컨테이너 제거
docker stop jyotish_api_test
docker rm jyotish_api_test
```

### Step 5: 간편한 docker-compose 파일 생성

현재 디렉토리에 `docker-compose.ready.yml` 파일이 이미 생성되어 있습니다:

```yaml
services:
  jyotish_api:
    image: jyotish-api:ready-to-use
    container_name: jyotish_api
    ports:
      - "9393:9393"
    restart: unless-stopped
```

실행:
```bash
docker-compose -f docker-compose.ready.yml up -d
```

## 다른 시스템으로 이미지 전송

### 방법 1: 파일로 저장/복원 (추천)

#### 현재 시스템에서:
```bash
# 이미지를 tar 파일로 저장
docker save jyotish-api:ready-to-use -o jyotish-api-ready.tar

# 파일 크기 확인 (약 1-2GB 예상)
ls -lh jyotish-api-ready.tar

# 압축하여 크기 줄이기 (선택사항)
gzip jyotish-api-ready.tar
# 결과: jyotish-api-ready.tar.gz (약 400-600MB)
```

#### 다른 Ubuntu 시스템에서:

1. **Docker 설치**:
```bash
sudo apt update
sudo apt install docker.io docker-compose
sudo usermod -aG docker $USER
# 로그아웃 후 다시 로그인 또는 newgrp docker
```

2. **이미지 파일 복사**:
```bash
# USB, scp, 또는 다른 방법으로 tar 파일 복사
# 예: scp user@source-server:/path/jyotish-api-ready.tar ./
```

3. **이미지 로드 및 실행**:
```bash
# 압축된 경우
gunzip jyotish-api-ready.tar.gz

# 이미지 로드
docker load -i jyotish-api-ready.tar

# 즉시 실행
docker run -d -p 9393:9393 --name jyotish_api jyotish-api:ready-to-use

# 또는 docker-compose 사용
# docker-compose.ready.yml 파일도 함께 복사한 후:
docker-compose -f docker-compose.ready.yml up -d
```

4. **동작 확인**:
```bash
curl http://localhost:9393/api/ping
```

5. **/var 디렉토리 권한 문제 해결** (Linux 환경에서 중요):

macOS에서는 문제가 없었지만, Linux 환경에서는 `/var/www/api/var` 하위 디렉토리 권한 문제로 인해 API가 제대로 작동하지 않을 수 있습니다.

**발생 가능한 오류:**
```
Error: Unable to write to /var/www/api/var/cache/dev
```

**해결 방법:**
```bash
# 필요한 디렉토리 생성 및 권한 설정
docker exec jyotish_api bash -c \
"mkdir -p /var/www/api/var/{cache/dev,log} \
&& chown -R www-data:www-data /var/www/api/var \
&& chmod -R 777 /var/www/api/var"

# API 재확인
curl http://localhost:9393/api/ping
```

**권한 문제가 해결되는 것들:**
- Symfony 캐시 디렉토리 접근 문제
- 로그 파일 쓰기 권한 문제
- 웹서버(www-data) 사용자 권한 문제

### 방법 2: Docker Hub 업로드 (공개/비공개)

#### 업로드:
```bash
# Docker Hub 로그인
docker login

# 태그 추가 (yourusername을 실제 Docker Hub 사용자명으로 변경)
docker tag jyotish-api:ready-to-use yourusername/jyotish-api:ready-to-use

# 업로드
docker push yourusername/jyotish-api:ready-to-use
```

#### 다운로드 및 사용:
```bash
# 다른 시스템에서 다운로드
docker pull yourusername/jyotish-api:ready-to-use

# 실행
docker run -d -p 9393:9393 --name jyotish_api yourusername/jyotish-api:ready-to-use
```

### 방법 3: 프라이빗 레지스트리

#### 레지스트리 서버 설정:
```bash
# 프라이빗 레지스트리 실행
docker run -d -p 5000:5000 --name registry registry:2
```

#### 이미지 업로드:
```bash
docker tag jyotish-api:ready-to-use localhost:5000/jyotish-api:ready-to-use
docker push localhost:5000/jyotish-api:ready-to-use
```

#### 다른 시스템에서 사용:
```bash
docker pull your-registry-server:5000/jyotish-api:ready-to-use
docker run -d -p 9393:9393 --name jyotish_api your-registry-server:5000/jyotish-api:ready-to-use
```

## 개선된 Dockerfile

현재 디렉토리에 `api/Dockerfile.improved`가 생성되어 있습니다. 이 파일은 Swiss Ephemeris 빌드를 자동화합니다:

### 주요 개선사항:
```dockerfile
# Build Swiss Ephemeris library during Docker build
RUN cd swetest/src && \
    make clean && \
    make && \
    chmod 777 /var/www/api/swetest/src && \
    chmod +x /var/www/api/swetest/src/swetest
```

### 사용법:
```bash
# 개선된 Dockerfile로 빌드
docker-compose -f docker-compose.improved.yml up --build -d

# 한 번에 모든 빌드가 완료됨 (수동 작업 불필요)
```

## 배포 키트 생성

완전한 배포를 위해 다음 파일들을 준비:

```bash
# 배포용 디렉토리 생성
mkdir -p deployment-kit

# 필요한 파일들 복사
cp docker-compose.ready.yml deployment-kit/
cp deployment-kit/README.md deployment-kit/
docker save jyotish-api:ready-to-use -o deployment-kit/jyotish-api-ready.tar
```

### 배포 키트 내용:
- `jyotish-api-ready.tar`: 완전히 빌드된 Docker 이미지
- `docker-compose.ready.yml`: 실행용 compose 파일
- `README.md`: 새 시스템에서의 설치 가이드

## API 사용법

### 주요 엔드포인트:

1. **Health Check**:
```bash
curl http://localhost:9393/api/ping
```

2. **차트 계산**:
```bash
curl "http://localhost:9393/api/calculate?latitude=28.6139&longitude=77.209&year=2023&month=12&day=25&hour=12&min=0&sec=0&time_zone=%2B05%3A30"
```

3. **현재 시간 차트**:
```bash
curl "http://localhost:9393/api/now?latitude=35.7219&longitude=51.3347&time_zone=%2B03%3A30"
```

4. **API 문서**:
- Swagger UI: http://localhost:9393/api/doc

## 트러블슈팅

### 일반적인 문제들:

1. **Linux 환경에서 /var 디렉토리 권한 문제** (가장 흔한 문제):

**현상**: macOS에서는 정상 작동하지만 Linux에서 API 호출 시 오류 발생
```
Error: Unable to write to /var/www/api/var/cache/dev
```

**원인**: Symfony의 캐시 및 로그 디렉토리에 대한 웹서버 권한 부족

**해결책**:
```bash
# 컨테이너 실행 후 권한 설정
docker exec jyotish_api bash -c \
"mkdir -p /var/www/api/var/{cache/dev,log} \
&& chown -R www-data:www-data /var/www/api/var \
&& chmod -R 777 /var/www/api/var"

# 또는 컨테이너 재시작 후 바로 실행
docker restart jyotish_api
docker exec jyotish_api bash -c "chown -R www-data:www-data /var/www/api/var && chmod -R 777 /var/www/api/var"
```

**이 명령이 해결하는 문제들**:
- `mkdir -p`: 존재하지 않는 캐시/로그 디렉토리 생성
- `chown -R www-data:www-data`: 웹서버 사용자 소유권 설정
- `chmod -R 777`: 모든 사용자에게 읽기/쓰기/실행 권한 부여

2. **포트 충돌**:
```bash
# 다른 포트 사용
docker run -d -p 8080:9393 --name jyotish_api jyotish-api:ready-to-use
```

3. **권한 문제** (Docker 자체):
```bash
sudo usermod -aG docker $USER
newgrp docker
```

4. **이미지 로드 실패**:
```bash
# 파일 무결성 확인
ls -la jyotish-api-ready.tar
# 다시 저장 시도
docker save jyotish-api:ready-to-use -o jyotish-api-ready-new.tar
```

5. **메모리 부족**:
```bash
# 시스템 리소스 확인
docker system df
docker system prune  # 불필요한 이미지/컨테이너 정리
```

## 결론

이 가이드를 통해:
- **기존 방식**: 3단계 수동 작업 필요
- **개선된 방식**: 1단계로 즉시 실행 가능
- **이식성**: 어떤 시스템에서든 동일하게 작동
- **안정성**: 빌드 실패 위험 제거

새로운 시스템에서는 단 몇 개의 명령어로 Jyotish API를 실행할 수 있습니다.