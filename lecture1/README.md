# Nexus Repository 실무 초급: 모듈 1. 기본 구성 및 스토리지 설계

## 1. 개요 및 배경 (Overview)

### 1-1. Pain Point (현장의 문제점)

* **초기 인프라 구축 및 스토리지 설계의 막막함:** H2를 써야 할지 PostgreSQL을 써야 할지, 스토리지는 로컬(NFS)로 둬야 할지 S3를 써야 할지 등 사내 인프라 환경에 맞는 초기 아키텍처 결정이 어렵습니다. 한 번 잘못 설계하면 추후 마이그레이션 시 큰 비용이 발생합니다.
* **SSL/HTTPS 적용 및 프록시 설정의 험난함:** Nexus 본체에 직접 SSL을 적용하려다 Java Keytool(Keystore) 설정의 복잡함에 부딪힙니다. 또한 Nginx 프록시를 둘 때도 대용량 파일 업로드 중단 현상을 겪게 됩니다.

### 1-2. Solution (해결 방안)

* **명확한 아키텍처 판단 기준 제시:** 인프라 규모에 따른 DB 선택(H2 vs PostgreSQL) 및 스토리지(File vs S3) 설계 기준을 명확한 수치와 함께 정리해 드립니다.
* **Nginx 리버스 프록시 SSL 연동 데모:** Nexus 본체 대신 앞단에 Nginx를 배치하여 가장 효율적이고 안전하게 SSL을 구성하는 베스트 프랙티스를 시연합니다.
* **맞춤형 스토리지(NFS/S3) 실무 구성:** 파일 기반 스토리지와 클라우드 S3 객체 스토리지의 올바른 연동 방법을 직접 보여드립니다.

---

## 2. 시스템 요구사항 및 아키텍처 이론

### 2-1. Nexus 데이터 구조의 이해

Nexus Repository는 내부 데이터를 두 가지 형태로 분리하여 관리합니다.

* **Database:** 사용자 및 권한 관리, 저장소 설정, 컴포넌트 메타데이터, 검색 인덱스 등 내부 설정과 텍스트 기반 데이터를 관리합니다.
* **Blob Store:** `.jar`, `.npm`, `Docker Image Layer` 등 실제 대용량 바이너리 파일이 물리적으로 저장되는 공간입니다.

### 2-2. 공식 시스템 요구사항 및 주요 제약 (System Requirements)

현업 운영 시 장애를 예방하기 위해 반드시 지켜야 하는 필수 권장 사항입니다.

1. **Java 환경:** Nexus 3.78.0 이상 버전부터는 자체적으로 **OpenJDK Java 21**을 내장하여 제공하므로 별도의 호스트 Java 설치에 대한 의존성이 줄었습니다.
2. **메모리(RAM) 할당 원칙:** 가용 메모리의 **3분의 2를 Nexus(JVM)에 할당**하고, 나머지 3분의 1은 OS 프로세스 및 시스템 버퍼용으로 남겨두어야 합니다.
3. **디스크 임계값 (4GB Watermark):** 스토리지는 최소 4GB 이상의 여유 공간을 항상 유지해야 합니다. 만약 여유 공간이 **4GB 미만으로 떨어지면, Nexus 데이터베이스는 데이터 손상을 막기 위해 즉시 'Read-Only(읽기 전용)' 모드로 강제 전환**되어 모든 업로드가 차단됩니다.
4. **File Handle (파일 디스크립터) 제한 상향:** Nexus는 대량의 HTTP 연결과 파일 I/O를 처리하므로 OS의 기본 File Handle 수량을 크게 초과하여 사용합니다. 설정이 상향되지 않으면 데이터 유실(Data Loss)이 발생할 수 있습니다. (Linux 기준 통상 `65536` 이상 권장)

### 2-3. 규모별 Nexus 권장 사양

운영하려는 시스템의 일일/시간당 트래픽 요구량에 맞춰 인프라 스펙을 선택해야 합니다.

| Profile Size | Profile Description | DB | CPU | RAM | Local Blob Storage |
| --- | --- | --- | --- | --- | --- |
| **Small** | 시간당 2만 건 / 일일 20만 건 | Embedded H2 | 2 Core | 8 GB | 20 GB |
| **Medium** | 시간당 10만 건 / 일일 100만 건 | External PostgreSQL | 4 Core | 8 GB | 200 GB |
| **Large** | 시간당 100만 건 / 일일 1,000만 건 | External PostgreSQL (HA) | 노드당 4 Core | 노드당 16 GB | 200 GB 이상 |
| **Very Large** | 시간당 200만 건 / 일일 2,000만 건 | External PostgreSQL (HA) | 노드당 8 Core | 노드당 32 GB | 10 TB 이상 |

---




## 3. 데이터베이스 아키텍처 (H2 vs PostgreSQL) 및 배포 실습

Nexus Repository는 과거 OrientDB(현재 지원 종료 수순)를 거쳐, 현재는 **내장형 H2**와 **외부 PostgreSQL** 두 가지 DB 옵션을 제공합니다. 인프라 규모에 따라 명확한 선택 기준을 가져야 합니다.

### H2 vs PostgreSQL 요약 및 선택 기준

| 구분 | Embedded H2 (기본 내장 DB) | External PostgreSQL (엔터프라이즈 권장 DB) |
| --- | --- | --- |
| **적합한 환경** | 개발, PoC, 단일 팀 (소규모) | Mission-critical, CI/CD 연동, K8s 환경 |
| **처리 한계** | 일일 최대 20만 요청 / 10만 컴포넌트 | 제한 없음 (DB 서버 리소스에 비례) |
| **가용성 (HA)** | 단일 노드 전용 (HA 불가) | Active-Standby (2-Node) 구성 지원 |
| **데이터 안정성** | 비정상 종료 시 DB 파일 손상 위험 높음 | 트랜잭션 보장 및 백업/복구 안정성 높음 |
| **컨테이너 운영** | **공식적인 상용 컨테이너 배포 미지원** | Docker, K8s 등 클라우드 네이티브 환경 지원 |
| **특이 사항** | 설치가 간편함 (별도 DB 구성 불필요) | DB Owner 권한 필수 (관련 에러 발생 시 `pg_trgm` 모듈 추가 필요) |

---

### [실습] Docker Compose 기반 배포

앞서 살펴본 두 가지 DB 아키텍처를 바탕으로, 각각의 환경을 구축해 봅니다.

#### [실습 3-1] H2 DB 기반 Nexus 배포 [docker-compose-h2.yml](docker-compose-h2.yml)

모든 데이터와 메타데이터가 `nexus-data` 볼륨 내부의 파일 기반 DB(H2)에 저장되는 가장 단순한 형태입니다.

**1. docker-compose-h2.yml 작성**

```yaml
services:
  nexus-h2:
    image: "sonatype/nexus3:3.94.1"
    container_name: "nexus-h2"
    restart: "unless-stopped"
    ports:
      - "8081:8081"
    volumes:
      - "./nexus-h2-data:/nexus-data"


```

**2. 데이터 저장 디렉토리 생성 및 권한 부여**
컨테이너 내부의 Nexus 프로세스는 `nexus` 계정(UID: 200, GID: 200)으로 동작하므로, 호스트 디렉터리의 권한을 반드시 맞춰주어야 합니다.

```bash
mkdir -p ./nexus-h2-data
sudo chown -R 200:200 ./nexus-h2-data


```

**3. 컨테이너 실행 및 접속 검증**

```bash
# 실행 명령어
docker compose -f docker-compose-h2.yml up -d

# 초기 비밀번호 확인
docker exec -it nexus-h2 cat /nexus-data/admin.password

# UI 접속확인
http://localhost:8081

```

---

#### [실습 3-2] PostgreSQL 연동 Nexus 배포 [docker-compose-pg.yml](docker-compose-pg.yml)

상용 환경 구성을 가정하여, 외부 PostgreSQL 컨테이너와 연동하는 실습입니다.

**1. docker-compose-pg.yml 작성**

* `NEXUS_DATASTORE_ENABLED=true`: 외부 DB 사용 활성화
* `NEXUS_DATASTORE_NEXUS_JDBCURL`: PostgreSQL 접속 URL 지정

```yaml
services:
  nexus-db:
    image: "bitnami/postgresql:latest"
    container_name: "nexus-postgres"
    restart: "unless-stopped"
    environment:
      - POSTGRESQL_USERNAME=nexus
      - POSTGRESQL_PASSWORD=nexus123!
      - POSTGRESQL_DATABASE=nexus
      - POSTGRESQL_POSTGRES_PASSWORD=admin123!
    volumes:
      - "./nexus-psql-data:/bitnami/postgresql"
    networks:
      - "nexus_pg_net"

  nexus-pg:
    image: "sonatype/nexus3:3.94.1"
    container_name: "nexus-pg"
    restart: "unless-stopped"
    ports:
      - "8082:8081"
    environment:
      - NEXUS_DATASTORE_ENABLED=true
      - NEXUS_DATASTORE_NEXUS_JDBCURL=jdbc:postgresql://nexus-db:5432/nexus
      - NEXUS_DATASTORE_NEXUS_USERNAME=nexus
      - NEXUS_DATASTORE_NEXUS_PASSWORD=nexus123!
      - NEXUS_DATASTORE_NEXUS_ADVANCED=maximumPoolSize=200
    volumes:
      - "./nexus-pg-data:/nexus-data"
    depends_on:
      - "nexus-db"
    networks:
      - "nexus_pg_net"

networks:
  nexus_pg_net:
    driver: "bridge"


```

**2. 데이터 저장 디렉토리 생성 및 권한 부여**
Nexus 데이터 폴더는 `200:200`, Bitnami PostgreSQL 폴더는 `1001:1001` 권한이 필요합니다.

```bash
mkdir -p ./nexus-pg-data ./nexus-psql-data
sudo chown -R 200:200 ./nexus-pg-data
sudo chown -R 1001:1001 ./nexus-psql-data


```

**3. 컨테이너 실행 및 접속 검증**
테스트된 PostgreSQL 18.4 버전의 경우 `pg_trgm` (Trigram) 확장 모듈을 직접 설치하지 않아도 문제없이 작동하므로 기본 설정으로 진행합니다. 단, 사용 중 관련 에러가 발생하는 환경이라면 DB 접속 후 `CREATE EXTENSION IF NOT EXISTS pg_trgm;` 구문을 실행하여 모듈을 추가해야 합니다.

```bash
# 1) DB 컨테이너 구동
docker compose -f docker-compose-pg.yml up -d nexus-db

# 2) Nexus 컨테이너 구동
docker compose -f docker-compose-pg.yml up -d nexus-pg

# 3) 초기 비밀번호 확인
docker exec -it nexus-pg cat /nexus-data/admin.password

# 4) UI 접속확인
http://localhost:8082

```

---

## 4. Nginx 리버스 프록시 및 SSL/TLS 연동

### 리버스 프록시 도입의 핵심 이점 (Why Reverse Proxy?)

Nexus Repository를 운영할 때 JVM(Nexus 본체)에 직접 SSL 인증서를 적용하기보다는, 앞단에 Nginx와 같은 리버스 프록시를 두는 아키텍처가 일반적으로 권장됩니다.

* **SSL/TLS Offloading:** CPU 연산량이 많은 SSL 암복호화 처리를 Nginx가 전담하여 Nexus 본체의 부하를 줄입니다.
* **보안 및 포트 표준화:** Nexus 본체를 외부에 직접 노출하지 않고, Nginx 레벨에서 보안 정책을 적용할 수 있습니다.
* **스트리밍 최적화:** Nginx 설정을 통해 수 GB에 달하는 Docker 이미지나 아티팩트를 메모리 병목 없이 Nexus로 직접 스트리밍 업로드할 수 있습니다.

---

### [실습] Nginx 구성 및 SSL 인증서 연동 [docker-compose-nginx.yml](docker-compose-nginx.yml)

#### 1. docker-compose-nginx.yml 작성

```yaml
services:
  nginx:
    image: "nginx:alpine"
    container_name: "nexus-nginx"
    restart: "unless-stopped"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "./nginx/conf.d:/etc/nginx/conf.d"
      - "./nginx/ssl:/etc/nginx/ssl"
    networks:
      - "nexus_pg_net"

networks:
  nexus_pg_net:
    external: true
    # 주의: 앞선 컨테이너를 실행한 디렉토리명에 따라 네트워크 접두사가 달라집니다. (예: 폴더명이 nexus-oss-lecture인 경우)
    name: "nexus-oss-lecture_nexus_pg_net" 


```

#### 2. 디렉토리 생성 및 자체 서명 SSL 인증서 발급

테스트 목적으로 `mkcert`를 사용합니다. `mkcert`는 사용 시 로컬 머신에 Root CA를 자동으로 생성하고 등록해 주기 때문에 로컬 환경 접속 시에는 인증 문제 경고가 발생하지 않습니다. 그러나 다른 PC나 외부 환경에서 접속할 경우에는 인증서를 신뢰할 수 없어 보안 경고가 발생합니다.

```bash
mkdir -p ./nginx/ssl ./nginx/conf.d

# mkcert를 이용한 SSL 인증서 생성 (localhost 및 127.0.0.1용)
mkcert -cert-file ./nginx/ssl/nexus.pem -key-file ./nginx/ssl/nexus-key.pem localhost 127.0.0.1


```

#### 3. conf 파일 작성 [./nginx/conf.d/nginx.conf](./nginx/conf.d/nginx.conf)

> 핵심 설정 디렉티브 분석
> * `proxy_buffering off;` : 대용량 바이너리 스트리밍 시 Nginx 디스크 버퍼를 거치지 않고 Nexus로 즉시 전달하여 I/O 병목 및 업로드 지연 방지.
> * `client_max_body_size 0;` : Nginx의 기본 업로드 제한(1MB)을 해제하여 Docker 레이어 업로드 시 발생하는 `413 Too Large` 에러 방지.
> * `proxy_set_header X-Forwarded-Proto https;` : Nexus 내부 리다이렉션 링크가 HTTP로 깨지는 현상 예방.
> 
> 

```nginx
server {
    listen 80;
    server_name localhost;
    
    # HTTP 접속 시 HTTPS로 강제 리다이렉트
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    # 발급한 mkcert 인증서 매핑
    ssl_certificate     /etc/nginx/ssl/nexus.pem;
    ssl_certificate_key /etc/nginx/ssl/nexus-key.pem;

    # 대용량 아티팩트/Docker 이미지 전송 타임아웃 및 버퍼링 최적화
    proxy_send_timeout 120;
    proxy_read_timeout 300;
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_pass http://nexus-pg:8081/;
        proxy_pass_header Server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        
        # Nexus 대용량 파일 업로드 제한 해제 (0 = 무제한)
        client_max_body_size 0;
    }
}


```

#### 4. 컨테이너 실행 및 접속 검증

```bash
# nginx 컨테이너 구동
docker compose -f docker-compose-nginx.yml up -d


```

1. 웹 브라우저를 열고 `http://localhost`로 접속합니다.
2. `https://localhost`로 자동 리다이렉트 되는지 확인하고, SSL 인증서가 정상 적용되어 표시되는지 확인합니다.

---

## 5. Blob Store 스토리지 구성 및 설계

### 5-1. Blob Store 개요 및 내부 메커니즘

Blob Store는 메타데이터(DB 저장)를 제외한 실제 바이너리 아티팩트(`.jar`, `.npm`, `Docker Image Layer` 등) 파일이 물리적으로 저장되는 공간(Binary Large Object)입니다.

#### 1. Repository와 Blob Store의 관계

리포지토리는 단일 Blob Store 혹은 Group Blob Store에 1:N으로 연결됩니다.
여러 리포지토리가 동일한 Blob Store를 공유할 수 있으며, 최적의 성능을 위해 지연 시간(Latency)이 가장 낮은 위치에 할당하는 것이 원칙입니다.

#### 2. 파일 난독화 및 임의 수정 금지

리포지토리에 파일을 추가하면 Nexus는 파일 이름 충돌과 OS 파일 시스템 제약을 방지하기 위해 파일명을 난독화(Hash 형태 등)하여 저장합니다.

**경고:** Blob Store에 저장된 파일은 Nexus 애플리케이션의 통제를 받습니다. 디렉토리에 접근해 파일을 수동으로 삭제, 이동, 수정할 경우 DB 메타데이터와 불일치가 발생하여 데이터 손상(Corrupted)이 발생할 수 있습니다.

---

### 5-2. 스토리지 설계 전략 및 제약 사항

#### 1. 기본 저장소(`default`) 분리 권장

Nexus를 처음 설치하면 `$data-dir` 내부에 `default`라는 이름의 파일 시스템 Blob Store가 생성됩니다. 운영 안정성과 백업 효율성을 높이려면, 이 기본 경로 외부에 별도의 Blob Store를 생성하여 사용하는 것이 권장됩니다.

#### 2. 과도한 Blob Store 분할의 위험성

용도별(Docker, Maven 등)로 분리하는 것은 좋으나, 너무 잘게 쪼개어 수십 개의 Blob Store를 생성하면 성능 저하가 발생합니다.
Nexus의 Cleanup(오래된 파일 삭제) 및 Indexing Task는 Blob Store 단위로 직렬화(Serialize)되어 개별 실행되므로, 개수가 많아질수록 서버 부하가 기하급수적으로 증가합니다.

#### 3. 스토리지 타입 비교: File vs S3 (Object Storage)

인프라 환경과 예산에 따라 적절한 스토리지 타입을 선택해야 합니다.

| 구분 | File-Based Blob Store (Local Disk / NFSv4) | Cloud Object Store (AWS S3) |
| --- | --- | --- |
| **저장 아키텍처** | POSIX 표준 파일 시스템 구조 | REST API 기반 객체 스토리지 |
| **성능 (I/O 속도)** | **최상 (Low-Latency / High IOPS)** - 네트워크 홉(Hop)이 적어 빌드 속도가 매우 빠름 | **다소 느림** - API 호출 및 네트워크 Latency가 발생하여 File 방식 대비 최대 3배 느릴 수 있음 |
| **확장성 및 관리** | **한계 존재** - 디스크 파티션(LVM) 확장이나 NFS 쿼터 관리가 지속적으로 필요함 | **무제한 스케일아웃** - 용량 걱정 없이 무한히 확장 가능하며 관리 오버헤드가 없음 |
| **비용 효율성** | 고성능 SSD/NVMe 사용 시 비용이 높음 | 1GB당 저장 비용이 저렴함 |
| **권장 환경** | 온프레미스 인프라, **빠른 CI/CD 파이프라인 속도가 최우선인 환경** | AWS EC2/EKS 클라우드 환경, 대규모 백업 및 아카이빙 저장소 |
| **주의사항** | NFSv3는 지원하지 않음. **반드시 NFSv4 이상 권장** | 온프레미스 환경에서 클라우드 S3를 연동하면 네트워크 병목으로 인한 **성능 저하**가 발생함. (동일 리전 내 EC2 등에서만 사용 권장) |

---

### [실습] 5-3. NFS 연동 Blob Store

* NFS 서버의 디렉터리를 Docker 호스트에 마운트한 후, Nexus 컨테이너 내부 경로로 바인딩 볼륨 매핑합니다.
* Nexus 컨테이너 내부 프로세스는 `nexus` 계정(UID: 200, GID: 200)으로 동작합니다.
* NFS 공유 디렉터리의 소유권이 `200:200`으로 맞춰지지 않으면 디렉터리 생성 및 파일 쓰기 시 `Permission denied` 에러가 발생하며 Blob Store 생성이 실패합니다.

#### 1. NFS 스토리지 서버 측 설정 (NFS Server)

NFS 서버에서 디렉터리를 생성하고 권한 및 접근 제어(`/etc/exports`)를 설정합니다.

```bash
# 1) 디렉터리 생성 및 Nexus 전용 UID/GID 권한 부여
sudo mkdir -p /srv/nfs/nexus-blobs
sudo chown -R 200:200 /srv/nfs/nexus-blobs

# 2) /etc/exports 파일에 NFS 공유 설정 추가
echo "/srv/nfs/nexus-blobs *(rw,sync,no_subtree_check,all_squash,insecure,anonuid=200,anongid=200)" | sudo tee /etc/exports

# 3) NFS 서비스 재적용
sudo exportfs -r


```

#### 2. Nexus Docker 호스트 측 마운트 (NFS Client)

Nexus가 실행 중인 서버(Docker 호스트)에서 NFS 공유 경로를 마운트합니다.

```bash
# 1) 로컬 마운트 포인트 디렉터리 생성
mkdir -p ./nexus-nfs-blobs

# 2) NFS v4.1 마운트 실행 (성능 최적화 옵션, 192.168.41.150 = NFS 서버 IP)
sudo mount -t nfs -o defaults,noatime,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,vers=4.1 192.168.41.150:/srv/nfs/nexus-blobs $PWD/nexus-nfs-blobs

# 3) 마운트 및 소유권 확인 (200:200 표기 확인)
ls -ld ./nexus-nfs-blobs


```

#### 3. Compose 파일 볼륨 추가 (`docker-compose-pg.yml`)

NFS 마운트 경로를 Nexus 컨테이너 내부로 전달하도록 `volumes` 설정을 추가합니다.

```yaml
services:
  nexus-pg:
    volumes:
      # 아래 내용 추가
      - "./nexus-nfs-blobs:/nexus-data/blobs/nfs-blobstore"


```

적용 후 컨테이너를 재시작합니다.

```bash
docker compose -f docker-compose-pg.yml up -d nexus-pg


```

#### 4. Nexus Admin UI에서 NFS Blob Store 생성

1. Nexus 관리자 UI 접속 (`https://localhost`)
2. 좌측 메뉴 **Settings (톱니바퀴 아이콘)** 클릭
3. 좌측 메뉴 **Repository** ➡️ **Blob Stores** 클릭
4. **Create blob store** 버튼 클릭 후 항목 입력:

* **Type:** `File` 선택
* **Name:** `nfs-file-blobstore`
* **Path:** `/nexus-data/blobs/nfs-blobstore` *(컨테이너 내부 볼륨 바인딩 경로 입력)*

5. **Save** 클릭하여 생성

#### 5. 컨테이너 내부 마운트 경로 확인

```bash
docker exec nexus-pg df -h /nexus-data
docker exec nexus-pg df -h /nexus-data/blobs/nfs-blobstore


```

---

### [실습] 5-4. AWS S3 연동 Blob Store

#### 1. AWS UI에서 S3 버킷 생성

* AWS Console에서 S3 버킷을 생성하고 필요한 IAM 권한(Access Key)을 미리 준비합니다.

#### 2. Nexus Admin UI에서 S3 Blob Store 생성

1. Nexus 관리자 UI 접속 (`https://localhost`)
2. 좌측 메뉴 **Settings (톱니바퀴 아이콘)** 클릭
3. 좌측 메뉴 **Repository** ➡️ **Blob Stores** 클릭
4. **Create blob store** 버튼 클릭 후 항목 입력:

* **Type:** `S3` 선택
* **Name:** `S3-blobstore`
* **Region:** `생성 시 선택한 Region 선택`
* **Bucket:** `S3 Bucket Name 입력`
* **Access Key ID:** `IAM에서 생성한 Access Key ID 입력`
* **Secret Access Key:** `IAM에서 생성한 Secret Access Key 입력`

5. **Save** 클릭하여 생성
