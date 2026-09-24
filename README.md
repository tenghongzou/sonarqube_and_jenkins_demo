# sonarqube_and_jenkins_demo

## 服務版本

| 服務 | Image |
|---|---|
| SonarQube | `sonarqube:26.9.0.129388-community` |
| PostgreSQL | `postgres:16.15` |
| Nginx | `nginx:1.30.5-alpine` |
| Jenkins | `jenkins/jenkins:2.568.3-lts-jdk21`（加裝 docker CLI，見 `jenkins/Dockerfile`） |
| Jenkins agent | `jenkins/inbound-agent:3391.va_37fa_a_305d6d-3-jdk21`（加裝 docker CLI，見 `jenkins-agent/Dockerfile`） |

## 啟動

```shell
cp .env.example .env   # 填入密碼、agent secret、DOCKER_GID
docker compose up -d --build
```

- `JENKINS_AGENT_SECRET`：Jenkins 啟動後，到「管理 Jenkins → Nodes」建立與 `JENKINS_AGENT_NAME` 同名的節點（啟動方式選 inbound agent），取得 secret 後填入 `.env`，再執行 `docker compose up -d jenkins-agent`。
- `POSTGRES_USER` / `POSTGRES_PASSWORD` 只在 postgres volume 首次初始化時生效；已有資料的 volume 請沿用原本的帳密，或進 DB 以 `ALTER USER` 修改。
- Jenkins 與 agent 掛載了主機的 `/var/run/docker.sock`，等同擁有主機 root 權限，僅適合 demo／受信任環境。

## docker-compose
sonarqube 中的 elasticsearch 需要 vm.max_map_count=262144

### linux 
#### 一次性

```shell
sysctl -w vm.max_map_count=262144
```

#### 永久修改
修改 /etc/sysctl.conf

```conf
vm.max_map_count=262144
```

### Windows 
Windows 使用 WSL2 來使用 Docker Desktop
所以每次開機後都要修改，使用 powershell
#### 一次性

```shell
wsl -d docker-desktop 
sysctl -w vm.max_map_count=262144
```
