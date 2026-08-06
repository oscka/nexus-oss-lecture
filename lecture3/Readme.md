# Docker Registry 실무 적용 및 시연 가이드

> **[Pain Point]** 컨테이너 전용 사내 Registry 부재 및 Docker TLS/인증 통신 설정의 어려움  
> **[Solution]** Sonatype Nexus3 기반의 Hosted / Proxy / Group 저장소 구축 및 포트 분리, SAN 인증서 적용을 통한 안전한 Docker 이미지 Push/Pull 라이브 시연

---

## 1. 개요 및 배경 (Overview)

### 1.1 Pain Point (현장의 문제점)
1. **컨테이너 전용 사내 Registry 부재**
   - 팀 간 Docker 이미지를 안전하게 공유 및 저장할 중앙 레포지토리가 없어 개발 생산성이 저하됩니다.
2. **보안 및 인증 설정의 어려움**
   - Docker Client는 기본적으로 HTTPS(TLS) 통신을 강제하며, SAN(Subject Alternative Name)이 미포함된 자체 서명 인증서는 연결을 거부합니다.
3. **외부 망 트래픽 및 보안 리스크**
   - Docker Hub에서 매번 동일한 이미지를 반복 수신하여 대역폭이 낭비되고 외부 망 장애 시 빌드/배포가 중단됩니다.

### 1.2 Solution (해결 방안)
- **데이터 보존성 확보**: Sonatype Nexus3를 Docker 컨테이너로 실행하고 VM Host와 Volume을 직결하여 컨테이너 재시작 시에도 데이터를 영구 보존합니다.
- **TLS 통신 테스트 환경 구축 간소화**: 실무 환경의 복잡한 OpenSSL/CA 발급 절차 대신, 시연 및 POC(개념 검증) 목적으로 keytool 단일 명령어를 활용해 SAN이 주입된 JKS 인증서를 신속히 생성·적용하고 Docker Client 신뢰 등록을 완료합니다.
- **저장소 삼총사 구축**: Docker Hosted / Proxy / Group 저장소를 구성하여 사내 이미지 Push, 외부 이미지 캐싱(Caching), 통합 Pull을 한 번에 해결합니다.

---

## 2. 핵심 개념 이해 (Core Concepts)

### 2.1 Nexus Docker 저장소 3가지 유형

| 저장소 종류 (Type) | 역할 및 설명 | 주요 용도 | 접근 포트 예시 |
| :--- | :--- | :--- | :--- |
| **Docker Hosted** | 사내에서 직접 빌드한 고유 컨테이너 이미지를 저장하는 Private 저장소 | 이미지 Push (업로드) 및 사내 자산 관리 | `8082` |
| **Docker Proxy** | Docker Hub 등 외부 Public Registry를 **캐싱(Caching)**하는 저장소 | 외부 이미지 다운로드 속도 향상 및 망분리 대응 | *(내부 연결)* |
| **Docker Group** | Hosted와 Proxy를 하나의 단일 엔드포인트로 묶어주는 저장소 | 사내/외부 구분 없이 통합 Pull (다운로드) | `8083` |

---

## 3. 실습 1: VM Host $\leftrightarrow$ Docker Volume 디렉토리 준비

VM Host 디렉토리를 Docker Volume으로 연결하여, 컨테이너가 재시작되거나 삭제되어도 Nexus 데이터 및 JKS 인증서가 영구 보존되도록 사전 디렉토리를 구축합니다.

```bash
# 1. VM Host에 Nexus 데이터 및 SSL 저장 디렉토리 생성
mkdir -p /opt/nexus-data/etc/ssl

# 2. SSL 키스토어 작업용 디렉토리 생성
mkdir -p /root/nexus-ssl

# 3. 소유권 부여 (Nexus 내부 계정 UID 200번 권한 지정)
chown -R 200:200 /opt/nexus-data
```

---

## 4. 실습 2: Keytool 기반 SAN 인증서 생성 및 Volume 배치

OpenSSL 변환 과정(PKCS12 변환 등) 없이, `keytool` 단일 명령어로 SAN(`-ext "SAN=ip:192.168.41.209"`)이 포함된 JKS 키스토어를 직접 만든 후 Docker Client용 인증서(`nexus.crt`)만 추출합니다.

### Step 1: Keytool 단일 명령어로 JKS 및 CRT 생성

```bash
# 1. 작업 디렉토리 이동
cd /root/nexus-ssl

# 2. SAN 정보가 포함된 JKS 키스토어 단번에 생성 (비밀번호: password)
keytool -genkeypair -keystore keystore.jks   -storepass password -keypass password   -alias nexus -keyalg RSA -keysize 2048 -validity 3650   -dname "CN=192.168.41.209, OU=IT, O=MyCompany, L=Seoul, ST=Seoul, C=KR"   -ext "SAN=ip:192.168.41.209"

# 3. Docker Client 신뢰 등록에 필요한 공개키 인증서(nexus.crt) 추출
keytool -exportcert -keystore keystore.jks   -storepass password -alias nexus -rfc -file nexus.crt
```

### Step 2: Volume 마운트 경로로 복사 및 권한 설정

```bash
# 1. Volume 경로로 JKS 및 CRT 강제 복사
cp -f /root/nexus-ssl/keystore.jks /opt/nexus-data/etc/ssl/keystore.jks
cp -f /root/nexus-ssl/nexus.crt /opt/nexus-data/etc/ssl/nexus.crt

# 2. 소유권(200:200) 및 읽기 권한 설정
chown -R 200:200 /opt/nexus-data/etc/ssl
chmod 644 /opt/nexus-data/etc/ssl/keystore.jks
```

### Step 3: nexus.properties 사전 설정 (HTTPS 및 JKS 경로 명시)

`/opt/nexus-data/etc/nexus.properties` 파일에 SSL 포트 및 비밀번호 설정을 명시합니다.

```bash
# 한 줄 명령어 실행
printf "nexus-args=\${jetty.etc}/jetty.xml,\${jetty.etc}/jetty-https.xml,\${jetty.etc}/jetty-requestlog.xml
application-port-ssl=8443
ssl.etc=/nexus-data/etc/ssl
jetty.sslContext.keyStorePath=/nexus-data/etc/ssl/keystore.jks
jetty.sslContext.keyStorePassword=password
jetty.sslContext.keyManagerPassword=password
" > /opt/nexus-data/etc/nexus.properties

# 권한 재설정
chown -R 200:200 /opt/nexus-data/etc
```

---

## 5. 실습 3: Nexus Docker 컨테이너 실행 및 HTTPS 접속 검증

```bash
# 1. 기존 컨테이너가 있다면 중지 및 삭제
docker stop nexus 2>/dev/null && docker rm nexus 2>/dev/null

# 2. Nexus 실행 (Volume 연동 및 포트 매핑)
docker run -d   --name nexus   -p 8081:8081   -p 8443:8443   -p 8082:8082   -p 8083:8083   -v /opt/nexus-data:/nexus-data   --restart always   sonatype/nexus3:latest

# 3. 실시간 로그 확인 (Started Sonatype Nexus Repository Manager 출력 시 기동 완료)
docker logs -f nexus
```

> **포트 역할 정리**
> - **8081**: Nexus HTTP Web UI
> - **8443**: Nexus HTTPS (JKS 적용) Web UI / Admin
> - **8082**: Docker Hosted 저장소 (Push 전용)
> - **8083**: Docker Group 저장소 (Pull 전용)

### 초기 비밀번호 확인 및 로그인
```bash
cat /opt/nexus-data/admin.password
```
- 브라우저에서 `https://192.168.41.209:8443` 접속 후 `admin` 계정 로그인 및 비밀번호 변경.

---

## 6. 실습 4: Nexus Web UI 저장소 생성 및 Realm 설정

### Step 1: Docker Bearer Token Realm 활성화 (필수)
1. `https://192.168.41.209:8443` 접속 및 로그인.
2. 상단 톱니바퀴 (**Server Admin**) $\rightarrow$ **Security** $\rightarrow$ **Realms** 이동.
3. **Docker Bearer Token Realm**을 `Active` 컬럼으로 이동 후 **Save**.

### Step 2: Docker 저장소 3종 생성 (**Repositories** $\rightarrow$ **Create repository**)

#### A. docker-hosted (Push 전용)
- **Recipe**: `docker (hosted)`
- **Name**: `docker-hosted`
- **HTTPS**: Check $\rightarrow$ Port: `8082` 입력
- **Enable Docker V1 API**: Check
- **Deployment policy**: `Allow redeploy`

#### B. docker-proxy (캐싱 전용)
- **Recipe**: `docker (proxy)`
- **Name**: `docker-proxy`
- **HTTP/HTTPS**: 체크 안 함 *(외부 직접 노출 불필요)*
- **Remote storage**: `https://registry-1.docker.io` (Docker Hub)
- **Docker Index**: `Use Docker Hub`

#### C. docker-group (통합 Pull 전용)
- **Recipe**: `docker (group)`
- **Name**: `docker-group`
- **HTTPS**: Check $\rightarrow$ Port: `8083` 입력
- **Group Member**: `docker-hosted` (상단), `docker-proxy` (하단) 순서로 Selected 영역에 배치.

---

## 7. 실습 5: Docker Client SSL 신뢰 등록 및 실무 시연

### Step 1: Docker Client 인증서 등록 (`certs.d`)

Docker Daemon이 자체 서명(Self-Signed) JKS 인증서를 보안 경고 없이 신뢰하도록 각 포트 디렉토리에 `ca.crt`를 등록합니다.

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

### Step 2: [시연 1] Hosted 저장소 로그인 및 이미지 Push (사내 이미지 업로드)

```bash
# 1. Hosted 저장소(8082 포트) 로그인
docker login 192.168.41.209:8082
# Username: admin / Password: <설정한 비밀번호>

# 2. 테스트용 이미지 다운로드 및 태그 생성
docker pull alpine:latest
docker tag alpine:latest 192.168.41.209:8082/my-service:v1.0

# 3. Nexus Hosted 저장소로 Push
docker push 192.168.41.209:8082/my-service:v1.0
```

---

### Step 3: [시연 2] Group 저장소에서 사내 이미지 Pull (내부 다운로드)

```bash
# 1. Group 저장소(8083 포트) 로그인
docker login 192.168.41.209:8083
# Username: admin / Password: <설정한 비밀번호>

# 2. 기존 로컬 이미지 삭제 후 테스트
docker rmi 192.168.41.209:8082/my-service:v1.0

# 3. Group 포트(8083)를 통한 사내 이미지 Pull
docker pull 192.168.41.209:8083/my-service:v1.0
```

---

### Step 4: [시연 3] Proxy 저장소를 통한 외부 이미지 Caching Pull

```bash
# 1. 로컬에 없는 외부 이미지(nginx)를 Group 포트(8083)로 요청
docker pull 192.168.41.209:8083/nginx:latest
```

> **시연 검증 포인트**
> 1. **Nexus Web UI** $\rightarrow$ **Repositories** $\rightarrow$ **docker-proxy** $\rightarrow$ **Browse** 이동 시 `nginx` 이미지가 자동 Caching된 것을 확인합니다.
> 2. 이후 동일한 외부 이미지 요청은 외부 인터넷 연결 유무와 상관없이 Nexus 내부 네트워크를 통해 즉시 수신 가능합니다.