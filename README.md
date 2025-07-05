# ImmortalWrt-Openlist-Deployment
openlist在immortalwrt/openwrt上部署及子域名访问

本文为个人在immortalwrt安装openlist过程中与Gemini的对话总结。
个人通过此步骤已经实现外网子域名访问openlist。

-----

### ImmortalWrt 上部署 Openlist 及 HTTPS 訪問全流程總結

本指南將引導您在 ImmortalWrt (OpenWrt) 路由器上部署 Openlist，並配置 Nginx 實現 `https://openlist.xxxxx.top` 訪問，同時確保 `uhttpd` 監聽 80/8443 端口，並配置 IPv6 DDNS Go 域名解析和 Let's Encrypt 證書的自動申請與續期。

#### 前置準備：

1.  **SSH 訪問路由器：** 確保您可以使用 SSH 客戶端連接到您的 ImmortalWrt 路由器。
2.  **網絡連通性：** 路由器需要能夠訪問互聯網（Let's Encrypt 服務器、阿里云 DNS API）。
3.  **域名：** 擁有 `xxxxx.top` 域名的管理權限，並確保其 DNS 服務商為阿里云。
4.  **阿里云 AccessKey：** 準備好具有 `AliyunDNSFullAccess` 權限的阿里云 AccessKey ID 和 AccessKey Secret。
5.  **Openlist 可執行文件：** 根據路由器的 CPU 架構，準備好 Openlist 的可執行文件，官方github：https://github.com/OpenListTeam/OpenList-OpenWRT ，並上傳到路由器（例如 `/opt/openlist/`）。

-----

#### 步驟一：安裝和配置 Openlist

1.  **上傳 Openlist 可執行文件：**
    將下載的 Openlist 可執行文件上傳到路由器的指定目錄，例如 `/opt/openlist/`。給予執行權限。
    ```bash
    mkdir -p /opt/openlist
    # scp /path/to/openlist_binary root@your_router_ip:/opt/openlist/openlist_binary
    chmod +x /opt/openlist/openlist_binary
    ```
2.  **啟動 Openlist (初始測試)：**
    手動啟動 Openlist，並確認它在 5244 端口監聽。
    ```bash
    /opt/openlist/openlist_binary & # 在後台運行
    netstat -tulnp | grep 5244
    ```
3.  **配置 Openlist (關閉強制 HTTPS)：**
    登錄 Openlist 的 Web 界面 (`http://<路由器內網IP>:5244`)，進入 **「Web 協議」** 設置。
      * **啟用 SSL：** 勾選。
      * **強制 HTTPS：** **取消勾選。** (這一步是關鍵，讓 Openlist 自身在 5244 端口提供 HTTP 服務)
      * **SSL 證書/密鑰：** 這裡可以暫時不填寫，或者填寫您希望 Openlist 自身使用的證書路徑（與 Nginx 無關的證書）。
      * 保存設置。

-----

#### 步驟二：配置 IPv6 DDNS-Go 域名解析

這部分假設您使用 `ddns-go` 進行動態 DNS 解析。

1.  **安裝 `ddns-go` (如果尚未安裝)。**
    根據您的路由器架構下載 `ddns-go` 可執行文件，並上傳到路由器。
    ```bash
    mkdir -p /opt/ddns-go
    # scp /path/to/ddns-go_binary root@your_router_ip:/opt/ddns-go/ddns-go_binary
    chmod +x /opt/ddns-go/ddns-go_binary
    ```
2.  **啟動 `ddns-go` 並配置：**
    手動啟動 `ddns-go` (`/opt/ddns-go/ddns-go_binary -c /opt/ddns-go/config.yaml &`) 或配置為服務。
    登錄 `ddns-go` 的 Web 界面 (`http://<路由器內網IP>:9876`)。
      * **域名：** 配置 `openlist.xxxxx.top` 和 `*.xxxxx.top`。
      * **類型：** 選擇 `AAAA` (用於 IPv6)。
      * **DNS 服務商：** 選擇 `Aliyun`，填寫您的 AccessKey ID 和 AccessKey Secret。
      * 保存並啟用。
      * **確保 `openlist.xxxxx.top` 和 `*.xxxxx.top` 的 AAAA 記錄能正確解析到您的路由器公網 IPv6 地址。**

-----

#### 步驟三：安裝和配置 Nginx

1.  **安裝 Nginx 和 `cifs-utils` (如果需要 Windows 共享掛載)：**

    ```bash
    opkg update
    opkg install nginx-ssl kmod-fs-cifs cifs-utils
    ```

      * `nginx-ssl` 提供了帶 SSL 支持的 Nginx。
      * `kmod-fs-cifs` 和 `cifs-utils` 用於掛載 Windows 共享，如果不需要則跳過。

2.  **配置 `uhttpd` 監聽 80 和 8443 端口：**
    讓 Nginx 獨佔 443 端口，`uhttpd` 負責 LuCI。

    ```bash
    /etc/init.d/nginx stop   # 確保 Nginx 停止，避免衝突
    /etc/init.d/uhttpd stop  # 確保 uhttpd 停止

    uci set uhttpd.main.listen_https='8443' # LuCI HTTPS 端口
    uci commit uhttpd
    /etc/init.d/uhttpd restart
    ```

    **驗證：** 確保 `uhttpd` 成功監聽 80 和 8443，且 443 端口空閒。

    ```bash
    netstat -tulnp | grep 8443
    netstat -tulnp | grep 80
    netstat -tulnp | grep 443
    ```

3.  **配置 Nginx 主配置文件 (`/etc/nginx/nginx.conf`)：**
    確保 Nginx 主配置文件包含必要的全局設置和 `conf.d` 目錄。

    ```bash
    vi /etc/nginx/nginx.conf
    # 確保以下內容存在：
    # error_log  /tmp/nginx_error.log debug; # 開啟 debug 日誌方便排查
    worker_processes  1;
    events { worker_connections  1024; }
    http {
        include       mime.types;
        default_type  application/octet-stream;
        sendfile        on;
        keepalive_timeout  65;
        gzip  on; # 建議啟用 Gzip 壓縮
        include /etc/nginx/conf.d/*.conf; # 關鍵行，包含子配置
    }
    ```

4.  **配置 Nginx 代理 Openlist (`/etc/nginx/conf.d/openlist.conf`)：**
    Nginx 只需要監聽 443 端口，並代理到 Openlist 的 HTTP 端口。

    ```bash
    vi /etc/nginx/conf.d/openlist.conf
    ```

    ```nginx
    # /etc/nginx/conf.d/openlist.conf

    server {
        listen 443 ssl; # 只監聽 443 端口
        listen [::]:443 ssl; # IPv6 也監聽 443

        server_name openlist.xxxxx.top; # 匹配您的域名

        # SSL 憑證路徑，將在證書申請步驟中獲得和安裝
        ssl_certificate /etc/nginx/ssl/xxxxx.top.crt;
        ssl_certificate_key /etc/nginx/ssl/xxxxx.top.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256:EECDH+CHACHA20:EECDH+AESGCM:EECDH+AES256:EECDH+AES128:RSA+AES:!aNULL:!MD5:!RC4:!DES:!3DES:!DSS';
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always; # 強制瀏覽器使用 HTTPS

        ssl_stapling on; # OCSP Stapling (如果憑證支持)
        ssl_stapling_verify on;
        resolver 8.8.8.8 8.8.4.4 valid=300s; # 設置 DNS 解析器
        resolver_timeout 5s;

        location / {
            proxy_pass http://localhost:5244; # 代理到 Openlist 的 HTTP 端口
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https; # 告訴後端應用，請求是 HTTPS
            proxy_set_header X-Forwarded-Host $http_host;

            proxy_set_header Upgrade $http_upgrade; # WebSocket 支持
            proxy_set_header Connection "upgrade";

            proxy_buffering on; # sub_filter 需要此功能
            # 解決混合內容問題：將後端返回的 HTTP 鏈接替換為 HTTPS
            sub_filter 'http://openlist.xxxxx.top/' 'https://openlist.xxxxx.top/';
            sub_filter 'http://localhost:5244/' 'https://openlist.xxxxx.top/'; # 根據 Openlist 生成的鏈接模式調整
            sub_filter_once off;
            sub_filter_types text/html text/css application/javascript application/json application/xml;
        }
    }
    ```

    保存並退出。

-----

#### 步驟四：申請和自動續期 Let's Encrypt 泛域名憑證 (acme.sh)

1.  **安裝 `acme.sh`：**

    ```bash
    rm -rf /root/.acme.sh/ # 如果之前安裝過，先刪除舊的
    curl https://get.acme.sh | sh
    ```

2.  **配置阿里云 DNS API 憑證：**

    ```bash
    vi /root/.acme.sh/account.conf
    # 添加以下兩行，確保沒有空格或中文：
    Ali_Key="您的阿里云AccessKeyID"
    Ali_Secret="您的阿里云AccessKeySecret"
    ```

    保存並退出。

3.  **設置默認 CA 為 Let's Encrypt：**

    ```bash
    /root/.acme.sh/acme.sh --set-default-ca --server letsencrypt
    ```

4.  **申請泛域名憑證：**

    ```bash
    /root/.acme.sh/acme.sh --issue -d xxxxx.top -d '*.xxxxx.top' --dns dns_ali --debug 2 > /tmp/acme_debug.log 2>&1
    ```

    **等待命令執行完成。** 檢查 `/tmp/acme_debug.log` 確保憑證申請成功且沒有 `InvalidAccessKeyId` 或 `InvalidDomainName.Format` 錯誤。日誌末尾應顯示 `Success` 和憑證文件的路徑 (例如 `fullchain.cer` 在 `xxxxx.top_ecc` 目錄下)。

5.  **安裝憑證到 Nginx 目錄並配置自動重載：**

    ```bash
    mkdir -p /etc/nginx/ssl # 確保目錄存在
    /root/.acme.sh/acme.sh --install-cert -d xxxxx.top \
    --key-file       /etc/nginx/ssl/xxxxx.top.key \
    --fullchain-file /root/.acme.sh/xxxxx.top_ecc/fullchain.cer \ # 根據實際生成路徑調整
    --reloadcmd     "/etc/init.d/nginx reload"
    ```

    **這一步是實現自動化續期的關鍵。** `acme.sh` 會自動設置 cron job。

6.  **設置憑證文件權限：**

    ```bash
    chmod 600 /etc/nginx/ssl/xxxxx.top.key
    chmod 644 /etc/nginx/ssl/xxxxx.top.crt
    ```

7.  **手動重啟 Nginx (以確保憑證被加載)：**

    ```bash
    /etc/init.d/nginx restart
    ```

    **驗證：** 檢查 `netstat -tulnp | grep 443` 確保 Nginx 正在監聽。

-----

#### 步驟五：配置內網 DNS 解析 (Dnsmasq)

1.  **編輯 `dnsmasq` 配置：**
    ```bash
    vi /etc/config/dhcp
    ```
2.  在 `config dnsmasq` 部分添加一行（或新增 `config domain` 節）：
    ```uci
    config dnsmasq
        # ...
        list 'address' '/openlist.xxxxx.top/192.168.10.1' # 替換為您的路由器內網 IP
    ```
    保存並退出。
3.  **重啟 `dnsmasq` 服務：**
    ```bash
    /etc/init.d/dnsmasq restart
    ```
4.  **在內網客戶端刷新 DNS 緩存：** (例如 Windows 上運行 `ipconfig /flushdns`)

-----

#### 步驟六：防火牆配置

確保路由器的防火牆允許外部訪問 Openlist (通過 Nginx 的 443 端口)。

1.  **通過 LuCI 界面：** **網絡** -\> **防火牆** -\> **通信規則**，添加一個規則，允許來自 WAN 區域的 TCP 流量到路由器上的 443 端口。
2.  **通過命令行：**
    ```bash
    uci add firewall rule
    uci set firewall.@rule[-1].name='Allow-HTTPS-Openlist'
    uci set firewall.@rule[-1].src='wan'
    uci set firewall.@rule[-1].dest_port='443'
    uci set firewall.@rule[-1].proto='tcp'
    uci set firewall.@rule[-1].target='ACCEPT'
    uci commit firewall
    /etc/init.d/firewall reload
    ```

-----

#### 最終測試：

1.  **內網訪問：**
      * `https://openlist.xxxxx.top`：應通過 Nginx 訪問，顯示安全連接。
      * `http://192.168.10.1:5244`：應直接訪問 Openlist 的 HTTP 服務。
2.  **外網訪問：**
      * `https://openlist.xxxxx.top`：應通過 Nginx 訪問，顯示安全連接。

-----

