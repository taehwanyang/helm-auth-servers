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
curl -u client:secret123 \
  -X POST http://auth.ythwork.com/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&scope=read"
```

---

### Resource Server 호출

```bash
curl -H "Authorization: Bearer <access_token>" \
  http://resource.ythwork.com/api/read
```

---

## 🔥 seccomp profile 생성
  - kubernetes의 Deployment에 seccomp 프로파일을 적용

## 먼저 RuntimeDefault 적용
  - Authorization Server Deployment에 RuntimeDefault를 적용해본다.
  - 그럼 OCI 런타임의 기본 seccomp 프로파일이 적용된다.

```yaml
        containers:
          # ...
          securityContext:
            allowPrivilegeEscalation: false
            seccompProfile:
              type: RuntimeDefault
```

### strace로 startup부터 app runtime의 syscall을 추적하기 위해 테스트용 Dockerfile 만들기
  - -ff
    - 스레드/프로세스별로 파일 분리
    - /tmp/trace.<pid> 형태로 생김

  - -s 256
    - 문자열 더 길게 기록
  - -tt
    - 타임스탬프 포함
  -o /tmp/trace
    - 로그 파일 prefix

```sh
# ...
RUN apt-get update \
    && apt-get install -y strace \
    && rm -rf /var/lib/apt/lists/*
# ...
ENTRYPOINT ["strace", "-ff", "-s", "256", "-tt", "-o", "/tmp/trace", "java", "-jar", "/app/app.jar"]
```

### Resource Server startup ~ app runtime syscall 파일 만들기

```sh
kubectl exec -it auth-test-resource-server-767b699969-dm7gl -n auth -- sh -c 'tar czf /tmp/trace.tar.gz /tmp/trace*'
kubectl cp auth/auth-test-resource-server-767b699969-dm7gl:/tmp/trace.tar.gz ./trace.tar.gz

tar xzf trace.tar.gz
cd tmp

cat trace* | awk '{print $2}' | sed 's/(.*//' | sort -u
```

### Spring Boot Application seccomp profile 완성본

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": [
    "SCMP_ARCH_AARCH64"
  ],
  "syscalls": [
    {
      "names": [
        "accept",
        "bind",
        "brk",
        "clock_getres",
        "clock_nanosleep",
        "close",
        "close_range",
        "connect",
        "epoll_create1",
        "epoll_ctl",
        "epoll_pwait",
        "eventfd2",
        "execve",
        "exit",
        "faccessat",
        "fchdir",
        "fcntl",
        "flock",
        "fstat",
        "fstatfs",
        "ftruncate",
        "futex",
        "getcwd",
        "getdents64",
        "geteuid",
        "getpid",
        "getrandom",
        "getrusage",
        "getsockname",
        "getsockopt",
        "gettid",
        "getuid",
        "ioctl",
        "listen",
        "lseek",
        "madvise",
        "mkdirat",
        "mmap",
        "mprotect",
        "munmap",
        "newfstatat",
        "openat",
        "openat2",
        "ppoll",
        "prctl",
        "pread64",
        "prlimit64",
        "read",
        "readlinkat",
        "recvfrom",
        "rseq",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sched_getaffinity",
        "sched_yield",
        "sendmmsg",
        "set_robust_list",
        "set_tid_address",
        "setsockopt",
        "shutdown",
        "socket",
        "socketpair",
        "statx",
        "sysinfo",
        "uname",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["clone"],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 2114060288,
          "op": "SCMP_CMP_MASKED_EQ"
        }
      ]
    },
    {
      "names": ["clone3"],
      "action": "SCMP_ACT_ERRNO",
      "errnoRet": 38
    }
  ]
}
```

## seccomp profile을 seccomp profile 폴더로 복사

```sh
cp profiles/springboot-java-v2.1.json /var/lib/kubelet/seccomp/profiles/springboot-java-v2.1.json
```

### Resource Server Deployment에 반영

```yaml
        containers:
          # ...
            securityContext:
              allowPrivilegeEscalation: false
              seccompProfile:
                type: Localhost
                localhostProfile: profiles/springboot-java-v2.1.json
```

