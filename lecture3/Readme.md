# Docker Registry 실무 적용 및 Nexus3 레포지토리 구축 가이드

> **본 가이드는 사내 컨테이너 전용 레포지토리(Nexus3)의 구축부터 TLS(HTTPS) SAN 인증서 적용, Docker Hosted/Proxy/Group 저장소 포트 분리 및 실무 이미지 Push/Pull/Caching 시연까지 전 과정을 다룹니다.**

---

## 1. 개요 및 배경 (Overview)

### 1.1 Pain Point (현업의 주요 문제점)
1. **사내 컨테이너 이미지 중앙 관리 체계 부재**
   - 개발 팀 간 커스텀 Docker 이미지를 안전하고 체계적으로 공유할 표준 Private Registry가 부족함.
   - 외부 Public Registry 직접 사용으로 인한 버전 관리 및 보안 거버넌스 공백 발생.
2. **Docker Client SSL/TLS 통신 및 인증 설정 장벽**
   - Docker Daemon은 기본적으로 HTTPS 통신을 강제하며, SAN(Subject Alternative Name)이 포함되지 않거나 신뢰할 수 없는 사설 인증서는 접속을 차단함 (`x509: certificate relies on legacy Common Name field` 또는 `unknown authority` 에러).
3. **외부 망 트래픽 낭비 및 외부 장애 전파**
   - CI/CD 파이프라인 빌드 시 매번 Docker Hub에서 동일한 Base Image를 수신하여 네트워크 대역폭 낭비 및 Rate Limit(도커허브 요청 제한) 발생.
   - 외부망 연결 장애 시 사내 서비스 빌드/배포 전체 중단 risk.

### 1.2 Solution (해결 방안)
- **Sonatype Nexus Repository Manager 3 기반 저장소 이중화/통합**
  - **Docker Hosted**: 사내 빌드 이미지 저장 (Push 전용)
  - **Docker Proxy**: 외부 Docker Hub 이미지 캐싱 (Speedup 및 Rate Limit 회피)
  - **Docker Group**: Hosted + Proxy 단일 엔드포인트 제공 (통합 Pull)
- **Keytool 기반 SAN 사설 인증서 구성 및 Docker 신뢰 경로 등록**
  - 복잡한 CA 체계 대신 `keytool` 단일 명령으로 IP 기반 SAN 인증서를 생성하여 TLS 통신 구축.
  - Docker Client의 `/etc/docker/certs.d/<Host>:<Port>/ca.crt` 신뢰 경로 등록.
- **데이터 영구 보존성 확보**
  - VM Host Directory와 Docker Volume (`/nexus-data`) 직결로 컨테이너 재시작/재생성 시에도 데이터 보존.

---

## 2. Architecture & Port Map

```
+-----------------------------------------------------------------------------------+
| VM Host (IP: 192.168.41.209)                                                      |
|                                                                                   |
|   +---------------------------------------------------------------------------+   |
|   | Nexus3 Docker Container ( sonatype/nexus3:latest )                        |   |
|   |                                                                           |   |
|   |   [ Port 8081 ] ---> HTTP Web UI (Admin UI)                               |   |
|   |   [ Port 8443 ] ---> HTTPS Web UI (SAN Cert Applied)                      |   |
|   |                                                                           |   |
|   |   [ Port 8082 ] ---> Docker Hosted Repo (Push: 사내 이미지 저장)            |   |
|   |   [ Port 8083 ] ---> Docker Group Repo  (Pull: 사내 + Caching 외부 이미지)   |   |
|   |                       |-- Member 1: Docker Hosted Repo                    |   |
|   |                       +-- Member 2: Docker Proxy Repo (Docker Hub)        |   |
|   |                                                                           |   |
|   +---------------------------------------------------------------------------+   |
|                                                                                   |
|   Volume Mount: /opt/nexus-data <---> /nexus-data                                |
+-----------------------------------------------------------------------------------+
```

---

## 3. 실습 1: Host Directory 및 Volume 사전 작업

VM Host에서 Nexus 데이터 및 SSL 인증서가 저장될 영구 디렉토리를 생성하고, Nexus 내부 실행 계정(UID `200`, GID `200`) 권한을 부여합니다.

```bash
# 1. VM Host 내 Nexus 데이터 및 SSL 저장소 디렉토리 생성
mkdir -p /opt/nexus-data/etc/ssl

# 2. SSL 인증서 작업 전용 임시 디렉토리 생성
mkdir -p /root/nexus-ssl

# 3. Nexus 컨테이너 권한(UID:GID = 200:200) 매핑
chown -R 200:200 /opt/nexus-data
```

---

## 4. 실습 2: Keytool 기반 SAN 인증서 발급 및 TLS 환경 설정

Docker Daemon은 IP 접속 시 **SAN(Subject Alternative Name)** 항목에 해당 IP가 명시되어 있어야 인증서를 수용합니다.

### Step 1: SAN 주입 JKS 키스토어 생성 및 공개키 추출

```bash
# 1. SSL 작업 디렉토리 이동
cd /root/nexus-ssl

# 2. Keytool 단일 명령어로 SAN이 포함된 JKS 키스토어 생성 (비밀번호: password)
keytool -genkeypair -keystore keystore.jks   -storepass password -keypass password   -alias nexus -keyalg RSA -keysize 2048 -validity 3650   -dname "CN=192.168.41.209, OU=IT, O=MyCompany, L=Seoul, ST=Seoul, C=KR"   -ext "SAN=ip:192.168.41.209"

# 3. Docker Client 신뢰 등록용 PEM/CRT 공개키 인증서 추출
keytool -exportcert -keystore keystore.jks   -storepass password -alias nexus -rfc -file nexus.crt
```

### Step 2: Nexus Volume 경로 복사 및 권한 설정

```bash
# 1. 생성된 JKS 및 CRT 파일을 Volume 경로로 복사
cp -f /root/nexus-ssl/keystore.jks /opt/nexus-data/etc/ssl/keystore.jks
cp -f /root/nexus-ssl/nexus.crt /opt/nexus-data/etc/ssl/nexus.crt

# 2. 소유권(200:200) 및 읽기 권한 설정
chown -R 200:200 /opt/nexus-data/etc/ssl
chmod 644 /opt/nexus-data/etc/ssl/keystore.jks
chmod 644 /opt/nexus-data/etc/ssl/nexus.crt
```

### Step 3: nexus.properties 설정 (HTTPS 활성화)

`/opt/nexus-data/etc/nexus.properties` 파일에 HTTPS 포트 및 JKS 설정 정보를 정의합니다.

```bash
cat <<'EOF' > /opt/nexus-data/etc/nexus.properties
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-https.xml,${jetty.etc}/jetty-requestlog.xml
application-port-ssl=8443
ssl.etc=/nexus-data/etc/ssl
jetty.sslContext.keyStorePath=/nexus-data/etc/ssl/keystore.jks
jetty.sslContext.keyStorePassword=password
jetty.sslContext.keyManagerPassword=password
EOF

# 파일 권한 설정
chown -R 200:200 /opt/nexus-data/etc
```

---

## 5. 실습 3: Nexus Docker 컨테이너 구동 및 검증

```bash
# 1. 기존 nexus 컨테이너 정리
docker stop nexus 2>/dev/null && docker rm nexus 2>/dev/null

# 2. Nexus3 컨테이너 실행 (Volume 및 포트 바인딩)
docker run -d   --name nexus   -p 8081:8081   -p 8443:8443   -p 8082:8082   -p 8083:8083   -v /opt/nexus-data:/nexus-data   --restart always   sonatype/nexus3:latest

# 3. 실시간 구동 로그 확인 ("Started Sonatype Nexus Repository Manager" 확인)
docker logs -f nexus
```

### 초기 비밀번호 확인 및 접속
```bash
# 초기 admin 비밀번호 출력
cat /opt/nexus-data/admin.password
```
- 웹 브라우저 접속: `https://192.168.41.209:8443`
- 비밀번호 변경 및 익명 접근 설정 진행.

---

## 6. 실습 4: Nexus Web UI 저장소 구성 & Realm 활성화

### Step 1: Docker Bearer Token Realm 활성화 (필수)
1. `https://192.168.41.209:8443` 접속 후 관리자 로그인.
2. **Server Admin (톱니바퀴)** -> **Security** -> **Realms** 이동.
3. `Docker Bearer Token Realm` 항목을 **Active** 컬럼으로 이동 후 **Save**.

### Step 2: Docker 저장소 3종 생성 (**Repositories** -> **Create repository**)

#### A. `docker-hosted` (사내 이미지 업로드/Push 전용)
- **Recipe**: `docker (hosted)`
- **Name**: `docker-hosted`
- **HTTP/HTTPS**: `HTTPS` 체크 -> Port: `8082` 입력
- **Enable Docker V1 API**: 체크
- **Deployment policy**: `Allow redeploy`

#### B. `docker-proxy` (외부 Docker Hub 캐싱 전용)
- **Recipe**: `docker (proxy)`
- **Name**: `docker-proxy`
- **HTTP/HTTPS**: 체크 안 함 *(단독 외부에 포트 노출 불필요)*
- **Remote storage**: `https://registry-1.docker.io`
- **Docker Index**: `Use Docker Hub`

#### C. `docker-group` (통합 이미지 다운로드/Pull 전용)
- **Recipe**: `docker (group)`
- **Name**: `docker-group`
- **HTTP/HTTPS**: `HTTPS` 체크 -> Port: `8083` 입력
- **Group Members**: `docker-hosted` (상단 우선순위), `docker-proxy` (하단 순서) 배치

---

## 7. 실습 5: Docker Client SSL 신뢰 등록 및 라이브 시연

### Step 1: Docker Client 인증서 신뢰 등록 (`certs.d`)

```bash
# 1. 포트별 Docker certs.d 디렉토리 생성
mkdir -p /etc/docker/certs.d/192.168.41.209:8082          /etc/docker/certs.d/192.168.41.209:8083          /etc/docker/certs.d/192.168.41.209:8443

# 2. 추출한 SAN 인증서(nexus.crt)를 각 포트의 ca.crt로 복사
cp -f /root/nexus-ssl/nexus.crt /etc/docker/certs.d/192.168.41.209:8082/ca.crt
cp -f /root/nexus-ssl/nexus.crt /etc/docker/certs.d/192.168.41.209:8083/ca.crt
cp -f /root/nexus-ssl/nexus.crt /etc/docker/certs.d/192.168.41.209:8443/ca.crt

# 3. Docker 데몬 재시작
systemctl restart docker
```

---

### Step 2: [시연 1] Hosted 저장소 로그인 및 이미지 Push

```bash
# 1. Hosted 저장소(8082 포트) 로그인
docker login 192.168.41.209:8082
# ID: admin / PW: <변경한 비밀번호>

# 2. 테스트용 alpine 이미지 다운로드 및 태깅
docker pull alpine:latest
docker tag alpine:latest 192.168.41.209:8082/my-service:v1.0

# 3. Hosted 저장소로 이미지 Push
docker push 192.168.41.209:8082/my-service:v1.0
```

---

### Step 3: [시연 2] Group 저장소에서 사내 이미지 Pull

```bash
# 1. Group 저장소(8083 포트) 로그인
docker login 192.168.41.209:8083

# 2. 로컬 이미지 삭제 후 검증
docker rmi 192.168.41.209:8082/my-service:v1.0 alpine:latest

# 3. Group 포트(8083)를 통해 사내 이미지 Pull
docker pull 192.168.41.209:8083/my-service:v1.0
```

---

### Step 4: [시연 3] Proxy 저장소를 통한 외부 이미지 Caching Pull

```bash
# 1. 외부 이미지(nginx)를 Group 포트(8083)로 요청
docker pull 192.168.41.209:8083/nginx:latest
```

> **검증 포인트**:
> 1. Nexus Web UI (`https://192.168.41.209:8443`) -> **Browse** -> `docker-proxy` 이동 시 `nginx` 캐시 생성 확인.
> 2. 차후 동일 요청 시 외부 인터넷망 연결이 끊겨도 Nexus 내부 캐시를 통해 고속 다운로드 가능.

---