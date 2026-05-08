# Eureka Server - Service Discovery

이 리포지토리는 MSA(Microservices Architecture) 환경에서 각 마이크로서비스의 위치(IP, Port)를 동적으로 관리하고 라우팅하기 위한 **Service Discovery(Eureka Server)** 프로젝트입니다.

운영 환경에서의 단일 장애점(SPOF, Single Point of Failure)을 방지하기 위해 2개의 인스턴스가 Peer-to-Peer로 구성된 **고가용성(High Availability, HA)** 아키텍처를 채택하고 있습니다.

## 1. 기술 스택 (Tech Stack)

- **Framework**: Spring Boot, Spring Cloud Netflix Eureka
- **Infrastructure**: Google Compute Engine (GCE), Google Artifact Registry (AR)
- **Containerization**: Docker, Docker Compose
- **CI/CD**: GitHub Actions
- **Security**: GCP Workload Identity Federation (WIF), Identity-Aware Proxy (IAP)

## 2. 주요 아키텍처 및 설정

### 2.1. 고가용성 (HA) 및 Peer 복제
- `eureka-server-1`과 `eureka-server-2` 두 개의 프로필을 사용하여 상호 등록(Peer Awareness)을 수행합니다.
- 한 대의 서버가 다운되더라도 다른 서버가 서비스 레지스트리 정보를 유지하여 시스템 전체의 가용성을 보장합니다.

### 2.2. 자기 보호 모드 (Self-Preservation)
- 네트워크 순단 현상으로 인해 마이크로서비스들이 정상임에도 불구하고 Heartbeat를 보내지 못할 경우, 레지스트리에서 인스턴스를 일괄 삭제하는 것을 방지하기 위해 `enable-self-preservation: true`가 적용되어 있습니다.

## 3. 배포 관련 파일 

- `application.yaml`: 유레카 서버의 핵심 설정 및 Peer 노드(defaultZone) 정보가 정의되어 있습니다.
- `docker-compose.yaml`: 로컬 개발 및 테스트를 위한 컨테이너 실행 환경입니다.
- `docker-compose.prod.yaml`: 운영 환경 배포 시 사용되는 컨테이너 실행 환경입니다.
- `.env.template`: 로컬 테스트 시 필요한 환경 변수 템플릿입니다.
- `deploy.yaml`: GitHub Actions 기반의 CI/CD 자동 배포 파이프라인 스크립트입니다.
- `ar-image-retention-policy.json`: Artifact Registry의 스토리지 비용 최적화를 위해 최근 3개의 이미지만 보관하도록 하는 수명 주기 정책입니다.

## 4. 로컬 개발 환경 실행 가이드

로컬 환경에서 두 대의 Peer 인스턴스를 띄워 서비스 등록 및 복제 상태를 테스트할 수 있습니다.

### 4.1. 환경 변수 설정
프로젝트 루트 디렉토리에 `.env.template` 파일을 복사하여 `.env` 파일을 생성합니다. (로컬 실행 시에는 더미 값을 입력해도 무방합니다.)
```bash
cp .env.template .env
```

### 4.2. 컨테이너 실행

아래 명령어를 통해 `pgsg-network` 브릿지 네트워크로 묶인 두 대의 유레카 서버를 백그라운드에서 실행합니다.

```bash
docker compose up --build -d
```

### 4.3. 접속 확인

브라우저를 통해 각 노드의 Eureka Dashboard에 접속하여 "DS Replicas" 항목에 서로가 등록되어 있는지 확인합니다.

- **Node 1**: [http://localhost:8761](https://www.google.com/search?q=http://localhost:8761)
- **Node 2**: [http://localhost:8762](https://www.google.com/search?q=http://localhost:8762)

## 5. 운영 환경 배포 (CI/CD)

배포는 GitHub Actions(`deploy/deploy.yaml`)를 통해 100% 자동화되어 있으며, 롤링 업데이트(Rolling Update)와 자동 롤백 메커니즘을 포함합니다.

### 5.1. 배포 트리거

- `main` 브랜치에 코드가 `push` 될 때 자동으로 실행됩니다. 
- GitHub 저장소의 `Actions` 탭에서 `workflow_dispatch`를 통해 수동 배포가 가능하며, 필요시 AR 리텐션 정책을 강제 적용할 수 있습니다.

### 5.2. 무중단 배포 메커니즘

1. **Build & Push**: 소스 코드를 빌드하여 GCP Artifact Registry에 이미지를 푸시합니다. (빌드 속도 최적화를 위해 `gha` 캐시 사용)
2. **Sequential Deploy**: `max-parallel: 1` 설정에 따라 `eureka-server-1`의 배포 및 헬스체크가 완전히 성공한 이후에 `eureka-server-2`의 배포가 시작됩니다.
3. **Health Check**: Actuator의 `/actuator/health` 엔드포인트를 호출하여 `status: UP`을 확인합니다.

### 5.3. 자동 롤백 전략

- 배포 후 헬스체크에 실패하면, 파이프라인은 즉시 중단되지 않고 이전에 검증된 가장 최근의 안정적인 이미지(`stable` 태그)를 Pull 받아 해당 인스턴스를 원상 복구합니다. 
- 두 노드가 모두 성공적으로 배포되고, 상호 Peer 복제 검증(`verify-peer-replication`)까지 통과해야만 새로운 이미지에 `stable` 태그가 부여됩니다.

## 6. 서버 유지보수 및 접속

운영 서버(GCE)는 외부로 SSH 포트가 개방되어 있지 않으며, IAP(Identity-Aware Proxy)를 통해서만 안전하게 접근할 수 있습니다.

```bash
# GCP CLI를 이용한 IAP 터널링 SSH 접속 명령어 예시
gcloud compute ssh eureka-server-1 \
    --project="[GCP_PROJECT_ID]" \
    --zone="[GCP_ZONE]" \
    --tunnel-through-iap

```
