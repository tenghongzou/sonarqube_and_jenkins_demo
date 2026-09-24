# sonarqube_and_jenkins_demo

以 Docker Compose 建立 **Jenkins（CI）+ SonarQube（程式碼品質分析）** 的開發環境，並由 Nginx 依網域反向代理。

> ⚠️ 本專案為**開發環境**設定。正式環境部署時請更換所有密碼與 Jenkins agent secret。

## 架構

```
         瀏覽器
            │ :80                              :50000（外部 agent 用）
            ▼                                     │
        ┌────────┐  sonarqube.pervive.cc   ┌───────────┐
        │ nginx  │ ──────────────────────▶ │ sonarqube │ ──▶ postgres
        │        │  jenkins.pervive.cc     │   :9000   │
        └────────┘ ───────────┐            └───────────┘
                              ▼
                        ┌──────────┐  inbound  ┌───────────────┐
                        │ jenkins  │ ◀──────── │ jenkins-agent │
                        │  :8080   │           │               │
                        └──────────┘           └───────────────┘
                              └──── /var/run/docker.sock ────┘
                                  （共用主機 Docker）
              全部服務位於同一個 network：sonarnet
```

| 服務 | Image | 說明 |
|---|---|---|
| `sonarqube` | `sonarqube:26.9.0.129388-community` | 靜態程式碼分析 |
| `db` | `postgres:16.15` | SonarQube 資料庫 |
| `nginx` | `nginx:1.30.5-alpine` | 唯一對外的 80 port，依網域轉發 |
| `jenkins` | `jenkins/jenkins:2.568.3-lts-jdk21` + docker CLI | CI controller |
| `jenkins-agent` | `jenkins/inbound-agent:3391.va_37fa_a_305d6d-3-jdk21` + docker CLI | 執行 build 的 agent |

## 目錄結構

```
.
├── docker-compose.yml        # 服務定義
├── .env.example              # 環境變數範本（複製成 .env 使用）
├── jenkins/Dockerfile        # Jenkins controller，加裝 docker CLI
├── jenkins-agent/Dockerfile  # Jenkins agent，加裝 docker CLI
└── nginx/
    ├── nginx.conf            # Nginx 主設定
    ├── conf.d/               # HTTP 反向代理（jenkins、sonarqube）
    └── services/             # 預留給 stream（TCP）轉發設定
```

## 事前準備

### 1. 設定 `vm.max_map_count`

SonarQube 內建的 Elasticsearch 需要 `vm.max_map_count` 至少 `262144`。

**Linux**

```shell
# 一次性
sudo sysctl -w vm.max_map_count=262144

# 永久：在 /etc/sysctl.conf 加入下行後執行 sudo sysctl -p
vm.max_map_count=262144
```

**Windows（Docker Desktop + WSL2）**：每次開機後都要重新設定，在 PowerShell 執行：

```shell
wsl -d docker-desktop
sysctl -w vm.max_map_count=262144
```

**macOS（Docker Desktop）**：預設通常已是 262144，可用下列指令確認：

```shell
docker run --rm alpine sysctl vm.max_map_count
```

### 2. 設定網域

本機開發時，在 `/etc/hosts`（Windows：`C:\Windows\System32\drivers\etc\hosts`）加入：

```
127.0.0.1 sonarqube.pervive.cc
127.0.0.1 jenkins.pervive.cc
```

### 3. 建立 `.env`

```shell
cp .env.example .env
```

| 變數 | 說明 |
|---|---|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` | SonarQube 資料庫帳密。只在 postgres volume **首次初始化**時生效；已有資料時請沿用原帳密，或進 DB 以 `ALTER USER` 修改 |
| `JENKINS_AGENT_NAME` | Jenkins 節點名稱，需與 Jenkins 上建立的節點同名 |
| `JENKINS_AGENT_SECRET` | 節點的 secret，Jenkins 啟動後才能取得（見下方步驟） |
| `DOCKER_GID` | 主機 `docker.sock` 的群組 GID。Linux：`getent group docker \| cut -d: -f3`；Docker Desktop：`0` |

`.env` 已列入 `.gitignore`，請勿 commit。

## 啟動

`jenkins-agent` 需要 Jenkins 產生的 secret，第一次啟動請依序進行：

```shell
# 1. 先啟動 agent 以外的服務
docker compose up -d --build sonarqube db nginx jenkins
```

2. 取得 Jenkins 初始管理員密碼，開啟 http://jenkins.pervive.cc 完成安裝精靈：

   ```shell
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```

3. 到「管理 Jenkins → Nodes → New Node」建立與 `JENKINS_AGENT_NAME` 同名的節點：
   - Remote root directory：`/home/jenkins/agent`
   - Launch method：`Launch agent by connecting it to the controller`

4. 在節點頁面複製 secret，填入 `.env` 的 `JENKINS_AGENT_SECRET`，再啟動 agent：

   ```shell
   docker compose up -d --build jenkins-agent
   ```

5. 開啟 http://sonarqube.pervive.cc，預設帳密 `admin` / `admin`，首次登入會要求修改密碼。

之後啟動只要執行：

```shell
docker compose up -d
```

## 串接 Jenkins 與 SonarQube

1. **SonarQube 產生 token**：「My Account → Security → Generate Tokens」，類型選 Global Analysis Token。
2. **Jenkins 安裝 plugin**：「管理 Jenkins → Plugins」安裝 `SonarQube Scanner`。
3. **Jenkins 設定 SonarQube server**：
   - 「管理 Jenkins → Credentials」新增 `Secret text`，內容為上一步的 token。
   - 「管理 Jenkins → System → SonarQube servers」新增，名稱例如 `sonarqube`，URL 填 `http://sonarqube:9000`（容器間直接連線，不經 nginx）。
4. **Jenkins 設定 Scanner**：「管理 Jenkins → Tools → SonarQube Scanner installations」新增，名稱例如 `SonarScanner`，並勾選自動安裝。
5. **SonarQube 設定 Webhook**（讓 pipeline 等待 Quality Gate 結果）：「Administration → Configuration → Webhooks」新增，URL 填 `http://jenkins:8080/sonarqube-webhook/`。

`Jenkinsfile` 範例：

```groovy
pipeline {
    agent { label 'main-agent' }

    stages {
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'   // 對應第 4 步設定的名稱
                    withSonarQubeEnv('sonarqube') {         // 對應第 3 步設定的名稱
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=my-project"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
```

## 常用指令

```shell
docker compose ps                    # 查看服務狀態
docker compose logs -f sonarqube     # 追蹤單一服務 log
docker compose restart nginx         # 修改 nginx 設定後重新載入
docker compose down                  # 停止並移除容器（保留 volume 資料）
docker compose down -v               # ⚠️ 連同所有 volume 資料一起刪除
```

## 升級版本

1. 修改 `docker-compose.yml` 中的 image tag；Jenkins 與 agent 的版本則修改對應 `Dockerfile` 的 `ARG`，並同步更新 compose 中的 `image` 名稱。
2. 執行 `docker compose up -d --build`。
3. SonarQube 升級後，第一次啟動需到 http://sonarqube.pervive.cc/setup 執行資料庫升級。

> PostgreSQL 跨大版本（例如 16 → 18）無法直接換 image，需先 `pg_dump` 匯出再匯入新版本。

## 注意事項

- Jenkins 與 agent 掛載了主機的 `/var/run/docker.sock`，pipeline 可以直接使用 `docker` 指令，但這等同擁有主機 root 權限，僅適合開發或受信任環境。
- Nginx 使用 Docker 內建 DNS 在請求時才解析服務名稱，因此某個服務尚未啟動時，Nginx 仍可正常啟動，只有該網域會回應 502。

## 疑難排解

| 狀況 | 可能原因 |
|---|---|
| SonarQube 不斷重啟，log 出現 `max virtual memory areas vm.max_map_count [65530] is too low` | 未設定 `vm.max_map_count`，見「事前準備」 |
| `docker compose up` 顯示 `required variable ... is missing a value` | `.env` 未建立或缺少該變數 |
| jenkins-agent 無法連線，log 出現 `401` 或 `Unauthorized` | `JENKINS_AGENT_NAME` 與 Jenkins 節點名稱不一致，或 secret 錯誤 |
| pipeline 執行 `docker` 出現 `permission denied ... docker.sock` | `DOCKER_GID` 與主機 docker.sock 的群組不一致 |
| 開啟網址出現 502 | 對應服務尚未啟動完成，SonarQube 首次啟動約需 1～2 分鐘 |
