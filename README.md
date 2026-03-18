# 📦 OAuth2 Authorization / Resource Server Helm Chart

이 Helm Chart는 Kubernetes 환경에서 다음 두 서비스를 배포합니다:

- 🔐 Authorization Server  
- 🔓 Resource Server  

Ingress를 통해 외부 도메인으로 접근 가능하며, 내부적으로는 Kubernetes DNS를 사용하여 통신합니다.

---

## 🏗 Architecture

```
Client
  ↓
Ingress (nginx)
  ↓
-------------------------
| auth.ythwork.com      |
| → Authorization Server |
-------------------------

-------------------------
| resource.ythwork.com  |
| → Resource Server      |
-------------------------

(Internal)
Resource Server → Authorization Server (JWK 조회)
```

---

## 🚀 Features

- OAuth2 Authorization Server (Token 발급)
- JWT 기반 인증 (Resource Server)
- Kubernetes Ingress 기반 외부 노출
- 내부 DNS 기반 서비스 간 통신
- Helm을 통한 환경별 설정 관리

---

## 📁 Chart Structure

```
.
├── templates/
│   ├── authorization-server-deployment.yaml
│   ├── authorization-server-service.yaml
│   ├── resource-server-deployment.yaml
│   ├── resource-server-service.yaml
│   └── ingress.yaml
├── values.yaml
└── Chart.yaml
```

---

## ⚙️ Prerequisites

- Kubernetes cluster (k3s, AKS, EKS 등)
- Helm 3+
- NGINX Ingress Controller
- 도메인 설정 (예: auth.ythwork.com, resource.ythwork.com)

---

## 🔧 도메인 설정

```bash
sudo vi /etc/hosts
```

```
127.0.0.1 auth.ythwork.com
127.0.0.1 resource.ythwork.com
```

---

## 📦 Installation

```bash
helm install auth-test . -n auth --create-namespace --set ingress.className=nginx
```

---

## 🔄 Upgrade

```bash
helm upgrade auth-test . -n auth
```

---

## ❌ Uninstall

```bash
helm uninstall auth-test
```

---

## 🔐 JWT Configuration

Resource Server는 다음과 같이 구성됩니다:

- issuer-uri → 외부 도메인 (`auth.ythwork.com`)
- jwk-set-uri → 내부 서비스 DNS

### Environment Variables

```bash
SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK_SET_URI=http://auth-test-authorization-server/oauth2/jwks
```

---

## 🌐 Ingress

- auth.ythwork.com → Authorization Server  
- resource.ythwork.com → Resource Server  

---

## 🧪 Testing

### Token 발급

```bash
curl -u client:secret \
  -X POST http://auth.ythwork.com/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=urn:ietf:params:oauth:grant-type:password&username=user&password=1234&scope=read"
```


---

### Resource Server 호출

```bash
curl -H "Authorization: Bearer <access_token>" \
  http://resource.ythwork.com/api/read
```

---

## 🔥 중요

Deployment에 replicas는 1이며 resource는 일부러 넣지 않았습니다. 
