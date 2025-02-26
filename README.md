## 以 Windows 10 Docker Desktop 自架 n8n 為例

### Requirements
- 一組外部固定IP
    >提供 DNS A紀錄 setup 使用
- 防火牆配置
    >設定外部流量 Inbound 的 port及對應服務主機的 port    
- Docker Desktop
    >架設服務的平台

### 安裝 Docker Desktop
- [官網](https://docs.docker.com/desktop/setup/install/windows-install/) 下載應用程式(依個人電腦規格選取)
- 依下列步驟安裝
    - **開啟 .exe 檔案** \
    ![alt text](./src/01.%20點exe檔.png)
    - **系統詢問是否執行，點擊 Yes** \
    ![alt text](./src/02.%20UAC_Yes.png)
    - **配置設定維持預設即可，完成點選OK** \
    ![alt text](./src/03.%20Configuration.png)
    - **等待 Docker Desktop 解壓縮檔案** \
    ![alt text](./src/04.%20Unpacking%20files.png)
    - **出現如下畫面即安裝成功，完成點選Close** \
    ![alt text](./src/05.%20Installed%20Success.png)
- 開啟 Docker Desktop \
![alt text](./src/06.%20DockerDesktopIcon.png)
- 點擊 Accept \
![alt text](./src/07.%20Accept.png)
- 後續部分皆可 Skip 不影響操作及建置 \
![alt text](./src/08.%20Skip%20Login.png)
- 進到如下畫面即設定啟動完成 \
![alt text](./src/09.%20Started.png)

### 防火牆配置
- 以我家的設備網路走向為例
![alt text](./src/11.%20Network.png)

- 登入【無線分享器】設定頁面
    >192.168.50.1為內網設定IP，請依自家配置修改
    ![alt text](./src/12.%20WirelessConf.png)
- 找到【虛擬伺服器】的選項，新增以下防火牆規則
    >【服務名稱】可自行定義 \
    >【外部通訊埠】可自行更換 \
    >【內部通訊埠】則為443 \
    >【本地IP位址】則是服務的電腦IP
    ![alt text](./src/13.%20FireWallRule.png)

### 依 [n8n文件說明](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/) 操作
- 前面 3 項皆是 Linux 操作方式，Windows 可直接跳至 4. DNS setup
    >這邊請大家至自己申請的域名提供商設定即可
- 5.Create Docker Compose file
    >於 D:\n8n(路徑可自訂) 下建立 docker-compose.yml 檔案，複製官方所提供的程式碼
    ``` yml
    version: "3.7"

    services:
    traefik:
        image: "traefik"
        container_name: traefik
        restart: always
        command:
        - "--api=true"
        - "--api.insecure=true"
        - "--providers.docker=true"
        - "--providers.docker.exposedbydefault=false"
        - "--entrypoints.web.address=:80"
        - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
        - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
        - "--entrypoints.websecure.address=:443"
        - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
        - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
        - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
        ports:
        - "80:80"
        - "443:443"
        volumes:
        - traefik_data:/letsencrypt
        - /var/run/docker.sock:/var/run/docker.sock:ro

    n8n:
        image: docker.n8n.io/n8nio/n8n
        container_name: n8n
        restart: always
        ports:
        - "127.0.0.1:5678:5678"
        labels:
        - traefik.enable=true
        - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
        - traefik.http.routers.n8n.tls=true
        - traefik.http.routers.n8n.entrypoints=web,websecure
        - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
        - traefik.http.middlewares.n8n.headers.SSLRedirect=true
        - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
        - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
        - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
        - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
        - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
        - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
        - traefik.http.middlewares.n8n.headers.STSPreload=true
        - traefik.http.routers.n8n.middlewares=n8n@docker
        environment:
        - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
        - N8N_PORT=5678
        - N8N_PROTOCOL=https
        - NODE_ENV=production
        - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
        - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
        volumes:
        - n8n_data:/home/node/.n8n

    volumes:
    traefik_data:
        external: true
    n8n_data:
        external: true
    ```
- 6.Create <code>.env</code> file
    >於 D:\n8n(路徑可自訂) 下建立 .env 檔案，複製官方所提供的程式碼 \
    >需修改【DOMAIN_NAME】、【SUBDOMAIN】、【SSL_EMAIL】為自己的設定 \
    >備註：<font color="red">【GENERIC_TIMEZONE】</font>可依個人所屬地區做更改 - [參考文件](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
    ``` yml
    # The top level domain to serve from
    DOMAIN_NAME=example.com

    # The subdomain to serve from
    SUBDOMAIN=n8n

    # DOMAIN_NAME and SUBDOMAIN combined decide where n8n will be reachable from
    # above example would result in: https://n8n.example.com

    # Optional timezone to set which gets used by Cron-Node by default
    # If not set New York time will be used
    GENERIC_TIMEZONE=Europe/Berlin

    # The email address to use for the SSL certificate creation
    SSL_EMAIL=user@example.com
    ```
- 7.Create data folder
    >於 D:\n8n(路徑可自訂) 執行下列兩行指令(不含$符號)。<font color="red">P.S.官網文件是Linux的指令</font>
    ```docker
    $ docker volume create n8n_data
    $ docker volume create traefik_data
    ```
- 8.Start Docker Compose
    >於 D:\n8n(路徑可自訂) 執行下列指令(不含$符號)。<font color="red">P.S.官網文件是Linux的指令</font>
    - 啟動
        ```docker
        $ docker compose up -d
        ```
    - 停止
        ```docker
        $ docker compose stop
        ```
    >Demo範例
    ![alt text](./src/10.%20Build.png)
- 查看 Docker Desktop 的 Volumes / traefik_data SSL憑證是否申請成功
    >出現下圖箭頭所指的內容需要有金鑰才算成功，若不成功會顯示 null
    ![alt text](./src/15.%20SSL.png)
- 測試 https://n8n.example.com 可否正常連線，出現以下畫面則代表成功
    >https://n8n.example.com ==> 請替換為自己的域名
    ![alt text](./src/14.%20Finished%20Service.png)

### 可能顯示的錯誤訊息及配置問題
- 錯誤一 - traefik error
    ```PowerShell
    unable to generate a certificate for the domains [n8n.example.com]: error: one or more domains had a problem:\n[n8n.example.com] acme: error: 400 :: urn:ietf:params:acme:error:connection :: ${your public ip}: Timeout during connect (likely firewall problem)\n" ACME CA=https://acme-v02.api.letsencrypt.org/directory acmeCA=https://acme-v02.api.letsencrypt.org/directory domains=["n8n.example.com"] providerName=mytlschallenge.acme routerName=n8n@docker rule=Host(`n8n.example.com`)
    ```
    >解決方式: 確認 Router 上的防火牆設置是否配置正確 \
    >注意: <font color="red">防火牆規則順序</font>很重要，若其中一條符合了，後續的規則就不再匹配，小弟我就犯了這個錯誤，導致設定失敗...查問題查了快一周... \
    >下圖範例即是 3.4 條規則打架，當輸入 https://n8n.example.com 規則會在第3條匹配就成功，不過因為服務內的 5678 port 已經導向至 443 port，因而無法順利配發SSL憑證
    ![alt text](./src/16.%20Error%20FireWall%20Conf.jpg)
- 錯誤二 - 服務電腦 Docker Desktop 防火牆未開啟
    ![alt text](./src/17.%20Public%20Firewall.png)

### 參考文件
- [n8n 官方文件說明](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/)
- [安裝 Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/)
- [什麼是 Docker Compose?](https://docs.docker.com/compose/)
- [什麼是 Volumes?](https://docs.docker.com/engine/storage/volumes/)