# Docker Registry 실무 적용

---

## 📌 목차
1. [Pain Point & Solution](#1-pain-point--solution)
2. [Docker Registry 아키텍처 (Hosted, Proxy, Group)](#2-docker-registry-아키텍처)
3. [TLS/SSL 적용: Nginx vs Keytool (Direct SSL) 상세 분석](#3-tlsssl-적용-nginx-vs-keytool-direct-ssl-상세-분석)
4. [사설 인증서(Self-Signed Certificate) 메커니즘 및 Trust Chain](#4-사설-인증서self-signed-certificate-메커니즘-및-trust-chain)
5. [멀티 Registry Proxy & Group 구성 메커니즘 (GHCR 프록시 연동)](#5-멀티-registry-proxy--group-구성-메커니즘-ghcr-프록시-연동)
6. [포트 분리를 통한 독립 Registry 노출 및 인증 격리 전략](#6-포트-분리를-통한-독립-registry-노출-및-인증-격리-전략)

---

## 1. Pain Point & Solution

- **Pain Point:** 
  - 사내 전용 저장소의 부재 : 팀마다 이미지를 저장할 단일 접점이 없어 빌드된 이미지가 파편화되고 관리 통제가 불가능
  - 보안 및 인증 통제의 어려움 : Docker 데몬은 Self-Signed 사설 인증서 사용 시 SSL 에러를 발생
  - 외부 레지스트리 다운로드 제한 : 외부 저장소 이용 시 Rate Limit(다운로드 횟수 제한)에 걸리거나, 외부 망 장애 시 CI/CD 파이프라인이 즉시 중단되는 위험에 노출

- **Solution:** 
  - Nexus3 기반의 단일 접점 구축, Keytool을 활용한 Direct SSL(HTTPS) 암호화 통신 적용, 외부 이미지를 자동 캐싱하는 Proxy 저장소와 이를 하나로 통합하는 Group 레지스트리 구성.

---

## 2. Docker Registry 아키텍처

```mermaid
graph TD
    subgraph Client ["개발자 PC / CI Server"]
        DKC("Docker Client")
    end

    subgraph NexusGroup ["Nexus Docker Group (URL:Port 묶음)"]
        GPR_1("docker-group<br/>(포트 매핑 예: 18082)")
    end

    subgraph NexusMembers ["Nexus 저장소 멤버 (우선순위 순서)"]
        HPR_1("1순위: docker-hosted<br/>(사내 빌드 이미지 저장)")
    end

    subgraph NexusProxies ["Nexus 외부 Proxy"]
        PPR_1("2순위: dockerhub-proxy<br/>(Docker Hub)")
        PPR_2("3순위: ghcr-proxy<br/>(GHCR - Public 연동)")
        PPR_3("4순위: quay-proxy<br/>(Red Hat Quay)")
    end

    subgraph ExtReg ["외부 컨테이너 레지스트리"]
        EXT_D("Docker Hub")
        EXT_G("ghcr.io (GitHub)")
        EXT_Q("quay.io (Red Hat)")
    end

    DKC -. Pull 요청: 18082 .-> GPR_1
    GPR_1 ==> HPR_1
    GPR_1 ==> PPR_1
    GPR_1 ==> PPR_2
    GPR_1 ==> PPR_3

    PPR_1 -- 캐시 요청 --> EXT_D
    EXT_D == 이미지 제공 ==> PPR_1
    
    PPR_2 -- 캐시 요청 --> EXT_G
    EXT_G == 이미지 제공 ==> PPR_2

    PPR_3 -- 캐시 요청 --> EXT_Q
    EXT_Q == 이미지 제공 ==> PPR_3

    DKC -- Push 요청 (독립 포트 주입 필수: 18083) --> HPR_1

    classDef client fill:#e0f2fe,stroke:#2563eb,stroke-width:2px;
    classDef nexus fill:#f1f5f9,stroke:#0f172a,stroke-width:2px;
    classDef members fill:#fff,stroke:#1e293b,stroke-width:1.5px;
    classDef ext fill:#fee2e2,stroke:#ef4444,stroke-width:2px;

    class Client,DKC client;
    class NexusGroup,NexusMembers,NexusProxies nexus;
    class GPR_1,HPR_1,PPR_1,PPR_2,PPR_3 members;
    class ExtReg,EXT_D,EXT_G,EXT_Q ext;
```

### 아키텍처 핵심 설명
- **Hosted Repository:** 사내에서 직접 생성하고 빌드한 커스텀 Docker 이미지를 저장 (Read/Write 권한 분리 대상).
- **Proxy Repository:** 외부 Docker Registry(Docker Hub, GHCR, Quay)의 캐시 저장소 역할 (Read Only).
- **Group Repository:** 여러 Hosted 및 Proxy 저장소를 하나의 URL/포트로 묶어서 제공하는 가상 레지스트리. Member List 우선순위에 따라 순차 탐색.
  - ⚠️ **주의:** Docker API v2 Specification 및 Nexus OSS 규격상 Group Repository는 Push 요청을 수신할 수 없으며(Read-Only), Push 시 반드시 Hosted Repository 전용 포트로 직접 요청해야 합니다.

---

## 3. TLS/SSL 적용: Nginx vs Keytool (Direct SSL) 상세 분석

```mermaid
graph LR
    subgraph Client ["Docker Client (개발자 PC)"]
        DKC("DKC")
    end

    subgraph SSL_Nginx ["방법 1: Nginx SSL Termination (일반적인 방식)"]
        NGX("Nginx Reverse Proxy")
        NEX_N("Nexus Instance<br/>(Embedded Jetty)")
    end

    subgraph SSL_Keytool ["방법 2: Keytool 기반 Direct SSL (Zero Trust 보안)"]
        NEX_K("Nexus Instance<br/>(Embedded Jetty)")
    end

    DKC -- "(A) HTTPS 443" --> NGX
    NGX -- "(B) HTTP 8081 (평문)" --> NEX_N

    DKC -- "(C) HTTPS 18082 (In-Transit Encrypted)" --> NEX_K

    classDef client fill:#e0f2fe,stroke:#2563eb,stroke-width:1px;
    classDef nginx fill:#f8fafc,stroke:#0f172a,stroke-width:2px,stroke-dasharray: 5 5;
    classDef keytool fill:#f1f5f9,stroke:#1e3a8a,stroke-width:2px;
    classDef comp fill:#fff,stroke:#1e3a8a,stroke-width:2px;

    class Client,DKC client;
    class SSL_Nginx,NGX nginx;
    class NEX_N,NEX_K comp;
    class SSL_Keytool keytool;
```

### 이론 심화: 보안 정책별 인증서 적용 비교
1. **Nginx SSL Termination:**
   - Client~Proxy 구간만 SSL 암호화. Nginx와 Nexus 간 내부 통신(HTTP 8081)은 평문 전달.
   - **장점:** 중앙 집중식 인증서 관리 및 갱신 편의성.
2. **Keytool 기반 Direct SSL:**
   - Zero Trust 네트워크 보안 규정 준수. Reverse Proxy 내부 구간까지 포함한 **End-to-End In-Transit Encryption** 구현.
   - **사전 필수 작업:** Nexus 메뉴 `Security -> Realms`에서 **`Docker Bearer Token Realm`** 활성화 필수.

---

## 3-1. 호스트 터미널 작업 (Direct SSL 설정)
```bash
#Nexus 3 컨테이너 구동
sudo docker run -d \
  --name nexus \
  --restart always \
  -p 8081:8081 \
  -p 8443:8443 \
  -p 18082:18082 \
  -p 18083:18083 \
  -p 18084:18084 \
  -p 18085:18085 \
  -v /nexus-data:/nexus-data \
  sonatype/nexus3:latest

# 1. 호스트의 /nexus-data 디렉터리에 SSL 폴더 생성
sudo mkdir -p /nexus-data/etc/ssl

# 2. Keytool 명령어를 docker exec로 단발성 실행 (키스토어 파일은 /nexus-data에 바로 저장)
docker exec -it nexus keytool -genkeypair \
  -keystore /nexus-data/etc/ssl/keystore.jks \
  -storepass password \
  -keypass password \
  -alias nexus \
  -keyalg RSA \
  -keysize 2048 \
  -validity 3650 \
  -deststoretype JKS \
  -dname "CN=192.168.41.206, OU=IT, O=MyCompany, L=Seoul, ST=Seoul, C=KR" \
  -ext "SAN=ip:192.168.41.206"

# 3. /nexus-data/etc/nexus.properties SSL 설정 작성
sudo bash -c "cat << 'EOF' > /nexus-data/etc/nexus.properties
nexus-args=\${jetty.etc}/jetty.xml,\${jetty.etc}/jetty-http.xml,\${jetty.etc}/jetty-requestlog.xml,\${jetty.etc}/jetty-https.xml
application-port=8081
application-port-ssl=8443

ssl.etc=/nexus-data/etc/ssl
jetty.sslContext.keyStorePath=/nexus-data/etc/ssl/keystore.jks
jetty.sslContext.keyStorePassword=password
jetty.sslContext.keyManagerPassword=password
EOF"

# 4. 권한 정제 및 Nexus 컨테이너 재시작
sudo chown -R 200:200 /nexus-data/etc

docker restart nexus
```

---

## 3-2. 브라우저 UI 작업 (`https://192.168.41.206:8443`)
1. **Realm 활성화:** `Security -> Realms` ➔ **`Docker Bearer Token Realm`** Active 목록으로 이동 후 `Save`
2. **저장소 5개 생성:**

| Repository Name | Recipe | 세부 설정 옵션값 |
| :--- | :--- | :--- |
| **`docker-hosted`** | `docker (hosted)` | HTTPS Port: **`18083`** / Enable V1 API: **체크** / Deployment policy: **Allow redeploy** |
| **`ghcr-proxy`** | `docker (proxy)` | HTTPS Port: **`18084`** / Remote Storage: `https://ghcr.io/` / Docker Index: **`Use proxy registry (specified above)`** |
| **`quay-proxy`** | `docker (proxy)` | HTTPS Port: **`18085`** / Remote Storage: `https://quay.io/` / Docker Index: **`Use proxy registry (specified above)`** |
| **`dockerhub-proxy`**| `docker (proxy)` | **포트 미할당 (체크 해제)** / Remote Storage: `https://registry-1.docker.io` / Docker Index: **`Use Docker Hub Index`** |
| **`docker-group`** | `docker (group)` | HTTPS Port: **`18082`** / Members 순서: `hosted` ➔ `dockerhub` ➔ `ghcr` ➔ `quay` |

```bash
docker login 192.168.41.206:18083 -u admin -p [비밀번호]
```
---

## 4. 사설 인증서(Self-Signed Certificate) 메커니즘 및 Trust Chain

### 사설 인증서 에러 발생 원리
Docker Daemon은 Go 언어의 TLS 표준 라이브러리를 사용하며, 기본적으로 호스트 OS의 Root CA Store에 등록된 CA만 승인합니다. Self-Signed 인증서 사용 시 Trust Chain 검증 실패 에러가 발생합니다.

> `Error response from daemon: Get "[https://192.168.41.206:18082/v2/](https://192.168.41.206:18082/v2/)": x509: certificate signed by unknown authority`

### 해결책 메커니즘 비교
1. **`insecure-registries` 지정 (비권장/테스트용):**
   - TLS 검증 절차 무시. 보안 정책 무력화.
2. **`certs.d` 경로 CA 인증서 주입 (운영 권장):**
   - Docker Daemon이 특정 레지스트리(Domain:Port 또는 IP:Port)와 통신할 때 신뢰할 사설 CA/Leaf 인증서를 추가 등록.
   - **핵심 특징:** `/etc/docker/certs.d/` 경로는 온디맨드로 로드되므로 **데몬 재시작이 불필요**.
   - **주의사항:** Keytool에서 추출 시 반드시 **`-rfc` 옵션(PEM 포맷)**을 부여해야 함.

---

## 4-1. 인증서 주입 & `docker login` 성공 확인
```bash
# 1. JKS 키스토어에서 PEM(-rfc) 포맷 인증서 추출
docker exec -it nexus keytool -exportcert \
  -keystore /nexus-data/etc/ssl/keystore.jks \
  -alias nexus \
  -file /nexus-data/etc/ssl/nexus.crt \
  -rfc \
  -storepass password

# 2. Docker Client PC의 certs.d 경로로 복사 (데몬 재시작 불필요)
sudo mkdir -p /etc/docker/certs.d/192.168.41.206:18082
sudo mkdir -p /etc/docker/certs.d/192.168.41.206:18083
sudo mkdir -p /etc/docker/certs.d/192.168.41.206:18084
sudo mkdir -p /etc/docker/certs.d/192.168.41.206:18085

sudo cp /nexus-data/etc/ssl/nexus.crt /etc/docker/certs.d/192.168.41.206:18082/ca.crt
sudo cp /nexus-data/etc/ssl/nexus.crt /etc/docker/certs.d/192.168.41.206:18083/ca.crt
sudo cp /nexus-data/etc/ssl/nexus.crt /etc/docker/certs.d/192.168.41.206:18084/ca.crt
sudo cp /nexus-data/etc/ssl/nexus.crt /etc/docker/certs.d/192.168.41.206:18085/ca.crt

# 3. 인증서 주입 후 로그인 재시도 (성공 검증)
docker login 192.168.41.206:18083 -u admin -p [비밀번호]
docker login 192.168.41.206:18082 -u admin -p [비밀번호]
```

---

### 5. 멀티 Registry Proxy & Group 구성 메커니즘 (GHCR / Quay 연동)

- **Docker Hub:** Remote URL: `https://registry-1.docker.io` (Docker Index: `Use Docker Hub Index`)
- **GHCR (GitHub Container Registry):** Remote URL: `https://ghcr.io/` (Docker Index: `Use proxy registry`)
- **Quay:** Remote URL: `https://quay.io/` (Docker Index: `Use proxy registry`)

#### 🔑 GHCR PAT(Personal Access Token) 연동 전략
- **Public 이미지:** 별도 인증(PAT) 설정 없이 Remote URL 지정만으로 즉시 캐싱 가능.
- **Private 패키지 및 Rate Limit 대응:** 
  - 조직 내 Private GHCR 패키지를 프록시하거나, 트래픽 폭주 시 GitHub IP 제한(Rate Limit)을 방지하려면 **`read:packages` 권한의 PAT**를 발급하여 Nexus HTTP/HTTPS Authentication 메뉴에 등록 (Authenticated Proxy로 동작).

#### 🔀 Group 라우팅 및 이미지 이름 충돌(Naming Collision) 방지
- 여러 프록시와 Hosted 저장소를 `docker-group` 하나로 묶을 때, 서로 다른 레지스트리에 동일한 이름(예: `ubuntu`, `busybox`)의 이미지가 존재하면 **Member List 순서**에 따라 의도치 않은 저장소의 이미지가 끌려올 수 있음.
- **해결책:** Nexus의 **`Routing Rules`** 기능을 활용해 특정 딜리버리 경로(예: `github/*`, `quay/*`)별로 프록시 매핑을 강제 제어하여 이미지 이름 충돌을 방지함.

---

## 6. 포트 분리를 통한 독립 Registry 노출 및 인증 격리 전략

| Repository Name | Type | Assigned Port | 목적 및 보안 정책 | 인증 요구사항 |
| :--- | :--- | :--- | :--- | :--- |
| **docker-group** | Group | HTTPS 18082 | 개발자 공통 이미지 **Pull 전용** | 18082 개별 로그인 또는 Anonymous 접근 허용 |
| **docker-hosted** | Hosted | HTTPS 18083 | CI/CD 파이프라인 전용 **Push/Pull 창구** | 18083 계정 인증 필수 |
| **ghcr-proxy** | Proxy | HTTPS 18084 | 외부 GHCR 전용 독립 통로 | 18084 개별 로그인 필요 |
| **quay-proxy** | Proxy | HTTPS 18085 | 외부 Quay 전용 독립 통로 | 18085 개별 로그인 필요 |

> 💡 **핵심 메커니즘:** Docker Client는 `IP:포트번호` 조합을 서로 다른 독립된 레지스트리 서버로 인식합니다. 따라서 `18083` 포트로 `docker login`을 완료했더라도, `18082` 포트 접근 시 해당 인증 정보가 공유되지 않습니다. 포트별로 로그인을 각각 진행하거나 Nexus에서 Anonymous 읽기 권한을 허용해야 합니다.

---

## 6-1. 호스트 터미널 검증 스크립트
```bash
# 1. Group 창구(18082)를 통한 Docker Hub 이미지 Pull & 캐싱 검증
docker pull 192.168.41.206:18082/ubuntu:latest

# 2. GHCR 전용 포트(18084) 또는 Group 포트(18082)를 통한 GHCR 이미지 Pull & 캐싱 검증
docker pull 192.168.41.206:18084/stefanprodan/podinfo:latest
docker pull 192.168.41.206:18082/stefanprodan/podinfo:latest

# 3. Quay 전용 포트(18085) 또는 Group 포트(18082)를 통한 Quay 이미지 Pull & 캐싱 검증
docker pull 192.168.41.206:18085/quay/busybox:latest
docker pull 192.168.41.206:18082/quay/busybox:latest

# 4. 캐싱된 외부 이미지를 사내 Hosted 저장소 전용 포트(18083)로 Tagging 후 Push 업로드
docker tag 192.168.41.206:18084/stefanprodan/podinfo:latest 192.168.41.206:18083/my-company/podinfo:v1
docker push 192.168.41.206:18083/my-company/podinfo:v1

# 5. Hosted 저장소에 저장된 사내 이미지 Group 창구(18082)를 통해 최종 Pull 확인
docker pull 192.168.41.206:18082/my-company/podinfo:v1
```

---