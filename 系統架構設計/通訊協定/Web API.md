# Web API

[[TOC]]

## App 尋找本地服務器

若為私有 IP，則先以 UDP Port 9998 廣播尋找本地服務器，若有回應則連接到指定服務器，否則連到預設雲端服務器 (`<inet.web_api>`: `https://oisp.smart-ehome.com/api`)。

1. 尋找本地服務器 UDP 資料: `REQ SmartHOME [TAB] <port> [TAB] <client_key>`
    * `<port>` 為程式接收回應的 udp 埠, 零表示使用發送封包時的 port。
    * `<client_key>` 為 16 字元隨機值 (本次臨時 key)，以 base64 編碼。

    ```php
    $client_keystr = "01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16";
    $client_key = hex2bin(str_replace(' ', '', $client_keystr));
    $client_key_b64 = base64_encode($client_key);
    echo "REQ SmartHOME\t0\t{$client_key_b64}";
    # output: "REQ SmartHOME[↹]0[↹]AQIDBAUGBwgJEBESExQVFg=="
    # 上行輸出以 [↹] 表示 Tab 字元
    ```

1. 本地服務器回應 UDP `<port>`: `SmartHOME [TAB] <y> [TAB] <z>`
    * `<y>`: iv[16]+加密後的資料，轉為 base64 編碼。
    * `<z>`: `<y>` 的 `HMAC-SHA1` 驗證值，轉為 base64 編碼。 HMAC-SHA1 使用的鍵值為 `<client_key>`。
    * 若驗證正確則執行解密: `<text> =  AES-CTR 解碼(<client_key>, <y.data>, <y.iv>)`。
    * 解密出的 `<text>` 為 JSON 字串，例如:
        ```json
        {
            "server": "d2045940-5e38-11e8-bea1-77a4b0d356bf",
            "name": "AMMA-2F",
            "local": {
                "web_api": "https://dev.smart-ehome.com/api",
                "mqtt_host": "dev.smart-ehome.com",
                "mqtt_port": 1883,
                "mqtt_ssl_port": 8883
            },
            "inet": {
                "web_api": "https://oisp.smart-ehome.com/api",
                "mqtt_host": "oisp.smart-ehome.com",
                "mqtt_port": 1883,
                "mqtt_ssl_port": 8883
            }
        }
        ```
    * 以下所提到 <b id="json"></b>`<本地服務器ID>`、`<local.web_api>`、`<inet.web_api>` 是指此處 JSON 物件中 `server`、`local.web_api`、`inet.web_api` 的值，以本範例而言其值如下:
        * `<本地服務器ID>`: `d2045940-5e38-11e8-bea1-77a4b0d356bf`
        * `<local.web_api>`: `https://dev.smart-ehome.com/api`
        * `<inet.web_api>`: `https://oisp.smart-ehome.com/api`
    * php 逐步示範：
        ```php
        // <client key> = Hex(01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16)
        $client_key = hex2bin(str_replace(' ', '', '01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16'));

        // 收到: "SmartHOME [↹] $y [↹] $z"
        $y = "E7zQdgsvbcqSEkoHtxqiIQTQXN8S32fvXn6LLUj4zRBF92cnT1v4Fe5lKOGrjXgBi6BSoG+izgHVLJXJBFBbujXvrK6n3OfHIFQ9cDVvW+QmkFRrvxWeg7VyoFWTyVE2tO8AeO5ON08bfD9VypdGdRatQgCrXfHj3Ny8ajiYQXQ5SQvSfhDRxHrLUtr65dmX4CHgsuvqTnjmfO5K78DJbcPe+bPaBFNi3o0BZTbwbigU5APDOT9FQrYw8HrwY3313gaY1S//S/cGzZLwHWSyA/v7JIohnercDrbolBjOe17/g4h99SiTYcXtbs84fc9c2TZQ5jvxp8XL2ya4PW+D0jKjXEwWV84+WNc9JJDmXdr4Ab9QDtVWtcO9NJQ53Ej5Aov9LBV3JXlhPrTIrQ3YhOCfJXNxfkGaJiSatX4AkLum9QPtLR+ySx9F70zlO9WV5xeOTX1ZDcLq+M1d0AqRbNUpx1CUkjrmwQGtpxJFlLNepfZTSg0r/4C+7D90OcB8kgdVwiTdAuzus9VotHJTexXVYGeTD+1ItAbM7PxBK41tfFVZmK2yaAMc6mcNh8/A8jAEzVpjOzH9LyGIbI5arnHAfQ==";
        $z = "LrppVYNbY7agCf7K5ll1YgWtzgc=";

        // 檢驗 HMAC-SHA1 是否相符
        $hmac_key = $client_key;
        if (base64_encode(hash_hmac("sha1", $y, $hmac_key, true)) !== $z) {
            echo "*** Unmatch HMAC-SHA1";
            // return false;
        }

        // 解密
        $encrypted = base64_decode($y);
        $iv = substr($encrypted, 0, 16);
        $encryptedText = substr($encrypted, 16);
        #$text = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $client_key, $encryptedText, 'ctr', $iv);
        $text = openssl_decrypt($encryptedText, 'aes-128-ctr', $client_key, OPENSSL_RAW_DATA, $iv);
        echo "$text\n";
        // 輸出: 如同前述的 JSON 值
        ```
    * 參見 [客端 UDP 完整範例](#客端-udp-完整範例)。


## 網站連線主機名稱

1. 雲端服務器連線: `<WebAPI>` = [`<inet.web_api>`](#json)。
1. 本地服務器連線: `<WebAPI>` = [`<local.web_api>`](#json)。


## Web API 一律採用 POST 方法

1. 為了安全起見 Web API 一律採用 POST 方法，若網站有對外開放時請使用 SSL 加密，避免帳戶資料的外洩。
1. `POST` 資料內容可選擇下列兩種表示法，請於 HTTP 標題指定 `Content-Type`:\
    以 [App 設備第一次要先向本地服務器註冊 `create_device`](#app-設備第一次要先向本地服務器註冊-create_device) 範例說明:
    ```
    server=<本地服務器ID>
    userid=<用戶第三方認證ID>
    name=<用戶名稱>
    device=<設備名稱>
    authcode=<授權碼>
    lang=zh-TW
    ```
    實際內容為:
    ```
    server=d2045940-5e38-11e8-bea1-77a4b0d356bf
    userid=AX1234567890
    name=Sam Wang
    device=iPhone 7
    authcode=1G78F95W8F
    lang=zh-TW
    ```
    * JSON: `Content-Type: application/json`
        ```json
        {
            "server": "d2045940-5e38-11e8-bea1-77a4b0d356bf",
            "userid": "AX1234567890",
            "name": "Sam Wang",
            "device": "iPhone 7",
            "authcode": "1G78F95W8F",
            "lang": "zh-TW"
        }
        ```
    * Form Urlencoded: `Content-Type: application/x-www-form-urlencoded`
        ```
        server=d2045940-5e38-11e8-bea1-77a4b0d356bf&userid=AX1234567890&name=Sam%20Wang&device=iPhone%207&authcode=1G78F95W8F&lang=zh-TW
        ```
1. 服務器回覆一律為 JSON 格式 (`Content-Type: application/json; charset=utf-8`)\
    以下 JSON 格式中的註解僅為說明用，實際返回不包含這些註解:
    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                     	// 0:成功
        "payload": _視各功能返回不同型態的_物件_陣列_或_字串_
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


## 推播令牌 token 設定

1. 推播令牌: 為 FCM 推播時設備的授權令牌。App 必須自行向 FCM (Google 雲端訊息推播) 註冊主題推播 `/topics/<本地服務器ID>` 才可收到系統推播訊息。此令牌為必要時本系統對指定設備推播個人訊息。
2. 可在任一 Web API 中指定 `token` 推播令牌欄位，若和資料庫帳號中不同時會自動更新:
    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    token=<推播令牌>
    ```
3. 若 `<推播令牌>` 為單一 `?`，則表示清除資料庫中的 `token` 欄位，此後本系統將無法針對這個設備推播個人訊息。但此設備仍會收到主題推播，除非刪除應用程式或應用程式向 FCM 解除本服務器的主題推播。


## App 用戶及設備註冊

### App 設備第一次要先向本地服務器註冊 (create_device)

URI = [`<local.web_api>`](#json)`/create_device`

1. App 送出 `POST` 資料如下:

    第二個起的用戶第一次授權必須向 [系統管理者取得新用戶的授權碼 `get_authcode`](#系統管理者取得新用戶的授權碼-get_authcode)，第一個註冊的用戶即為系統管理者 (此時不檢查 `authcode`)。

    ```
    server=<本地服務器ID>
    userid=<用戶第三方認證ID>
    name=<用戶名稱>
    device=<設備名稱>
    os=<作業系統>
    authcode=<授權碼>
    lang=zh-TW
    ```

    * [`<本地服務器ID>`](#json): 由 UDP 取得。
    * <b id="userid"></b>`<用戶第三方認證ID>`: 指 APP 在 Facebook 或 Google 取得的第三方認證識別 ID。由於本系統信任 APP 所指定的 `<用戶第三方認證ID>`，因此請 APP 設計者妥善保管此一資訊，最好能以加密方式保存之。
    * `<用戶名稱>`: 為取自第三方認證帶過來的名稱，若無可省略此欄，本系統會以 `<用戶第三方認證ID>` 取代。
    * `<設備名稱>`: 請 App 自行定義，如: iPhone、iPad、Sony XZ ...，以供後續調用區別。若 `<設備名稱>` 已存在，則會延用舊有登入帳號，重新分配一個密碼，舊的密碼立即作廢。
    * `<作業系統>`: 此欄位可省略，若有時請指定為 `android`、`iOS` 或其他。
    * `<授權碼>`: 此欄位在新設備註冊前需向系統管理者取得之。若未帶此授權碼，除了第一個註冊的設備外，其餘會回報錯誤要求取得授權碼，之後再請再重試一次。
    * [`lang`](#訊息語系-lang): 返回錯誤訊息使用語系。

1. 網頁服務器回覆:

    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                     	// 0:成功
        "payload": {
            "device": "<設備名稱>",
            "loginid": "<登入帳號>",
            "password": "<登入密碼>"
        }
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 若收到 `status` = `2`，請向系統管理者 (第一個註冊者) 取得新授權碼後再試一次。參見 [特殊的錯誤碼處理原則](#網頁服務器回覆-status-錯誤處理)。
    * 每次授權碼使用後會失效，因此系統管理者要為每個新設備取得授權碼。每次取得的授權碼有效期間只有 4 個小時，超過時間後將無法註冊新設備，請重新取得授權碼後再註冊設備。


### 用戶取得已註冊設備 (get_devices)

URI = [`<local.web_api>`](#json)`/get_devices`

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    userid=<用戶第三方認證ID>
    ```
    或
    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    ```

    * [`<本地服務器ID>`](#json): 由 UDP 取得。
    * [`<用戶第三方認證ID>`](#userid): 指用戶在 Facebook 或 Google 取得的第三方認證識別 ID。
    * `loginid`、`password`: 必需是已註冊用戶設備帳號，以驗證用戶的合法性。通常就是 App 安裝所在的設備，每次註冊後必須將帳號、密碼儲存起來。
    * 可使用 `<用戶第三方認證ID>` 或 `loginid、password` 其中一種方式來確認用戶身份。
    * 當 App 設備更換或不確定以前是否已註冊過某一設備型號時，可用此 API 取得用戶已註冊清單，將已廢棄不用的設備刪除。

1. 網頁服務器回覆:

    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                     	// 0:成功
        "payload": {
            "userid": "<用戶第三方認證ID>",
            "name": "<用戶名稱>",
            "devices": ["<設備名稱1>", "<設備名稱2>", ...]
        }
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


### 用戶刪除已註冊設備 (delete_device)

URI = [`<local.web_api>`](#json)`/delete_device`

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    target_userid=<用戶第三方認證ID>
    target_device=<設備名稱>
    ```

    * [`<本地服務器ID>`](#json): 由 UDP 取得。
    * `loginid`、`password`: 必需是已註冊用戶設備帳號，以驗證用戶的合法性。通常就是 App 安裝所在的設備，每次註冊後必須將帳號、密碼儲存起來。
    * `target_userid`、`target_device`: 驗證登入用戶合法性後會將指定用戶的設備刪除之。若未指定 `target_userid`、`target_device` 則視同刪除本身。
    * 若不是系統管理者時，當用戶所有註冊設備都已清空時，本系統會自動移除此用戶資訊。若是系統管理者名下已無註冊設備，在[重新註冊一個新的設備](#app-設備第一次要先向本地服務器註冊-create_device)時並不需要授權碼驗證。

1. 網頁服務器回覆:

    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                        // 0:成功
        "payload": "<完成訊息>"              // 已刪除用戶、還有 n 個設備 ...
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


### 系統管理者取得新用戶的授權碼 (get_authcode)

URI = [`<local.web_api>`](#json)`/get_authcode`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    ```

1. 網頁服務器回覆:

    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                     	// 0:成功
        "payload": "<新用戶授權碼>"
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


### 取得註冊用戶名單 (get_reg_users)

URI = [`<local.web_api>`](#json)`/get_reg_users`

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    ```

    * [`<本地服務器ID>`](#json): 由 UDP 取得。
    * `loginid`、`password`: 必需是已註冊用戶設備帳號，以驗證用戶的合法性。

1. 網頁服務器回覆:

    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                     	// 0:成功
        "payload": [
            {
                "userid": "<用戶1第三方認證ID>",
                "name": "<用戶1名稱>",
                "is_admin": 1
            },
            {
                "userid": "<用戶2第三方認證ID>",
                "name": "<用戶2名稱>",
                "is_admin": 0
            }
            // 其他註冊用戶...
        ]
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 若為系統管理者，返回的 `<payload>` 為系統現行用戶名單 (陣列)。其中欄位 `is_admin` 表示該用戶是否為系統管理者。
    * 非系統管理者只會返回自己的資料 (陣列中只有一筆)。


### 變更用戶名稱或系統管理者身份 (update_reg_user)

URI = [`<local.web_api>`](#json)`/update_reg_user`

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    target_userid=<用戶第三方認證ID>
    target_name=<用戶名稱>
    is_admin=<0,1>
    ```

    * [`<本地服務器ID>`](#json): 由 UDP 取得。
    * `loginid`、`password`: 必需是已註冊用戶設備帳號，以驗證用戶的合法性。
    * `target_userid`、`is_admin`: 設定用戶 `target_userid` 管理者身份 (`is_admin`)。
    * `target_name`: 不為空白時才更新用戶名稱，此欄位可省略。
    * 僅系統管理者才可變更其他用戶 `is_admin` 身份。
    * 非系統管理者只能更改自己的用戶名稱。

1. 網頁服務器回覆:

    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                     	// 0:成功
        "payload": "<成功訊息>"
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 確認系統管理者身份無誤後，返回的 `<payload>` 為系統現行用戶名單 (陣列)。其中欄位 `is_admin` 表示該用戶是否為系統管理者。
    * 本系統允許多個系統管理者。


### 刪除註冊用戶 (delete_reg_user)

URI = [`<local.web_api>`](#json)`/delete_reg_user`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    target_userid=<用戶第三方認證ID>
    ```

    * [`<本地服務器ID>`](#json): 由 UDP 取得。
    * [`<用戶第三方認證ID>`](#userid): 指用戶在 Facebook 或 Google 取得的第三方認證識別 ID。
    * `loginid`、`password`: 必需是已註冊用戶設備帳號，以驗證用戶的合法性。
    * `target_userid`: 刪除用戶 `target_userid` 及其下所有的註冊設備、登入資訊等。

1. 網頁服務器回覆:

    ```js
    // 正確
    {
        "server": "<服務器ID>",
        "status": 0,                     	// 0:成功
        "payload": "<完成訊息>"
    }

    // 錯誤
    {
        "server": "<服務器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 用戶被刪除後，必須重新註冊設備後方可繼續使用本系統。


## App 登錄取得取得 MQTT 通訊時所需的資訊 (login)

URI = [`<WebAPI>`](#網站連線主機名稱)`/login`

取得 MQTT 通訊時所需的資訊。

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    lang=zh-TW
    ```

    * [`lang`](#訊息語系-lang): 登入成功時，本系統會將您的語系存到對應的登入帳號。以後在 MQTT 通訊中若發生錯誤時，會以此刻指定的語系送出相關訊息。

1. 網頁服務器回覆:

    ```js
    {
        "server": "<服務器ID>",
        "status": 0,                        // 0:成功, 非零:錯誤
        "payload": {                        // MQTT 用物件 或 錯誤訊息(字串)
            "clientid": "<client_id>",
            "topic": {
                "pub": "to/",
                "sub": ["from/#", "to/<uid>", "to/<cid>"]
            }
        }
    }
    ```

    * `<client_id>`、`topic.pub`、`topic.sub` 於 MQTT 連線登入時使用，參見 [MQTT 通訊協定 - Node JS 範例](./MQTT%20通訊協定.md#node-js-範例)。


## MQTT 系統功能轉接

### 查詢系統中目前組態資訊 (mq_confreq)

URI = [`<WebAPI>`](#網站連線主機名稱)`/mq_confreq`

本功能為 [MQTT 通訊協定 - 查詢服務器下各模組設備組態 (cmd=1)](./MQTT%20通訊協定.md#查詢服務器下各模組設備組態-cmd1) 中原有功能，改用 Web API 轉接。

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    lang=zh-TW
    ```

1. 網頁服務器回覆:

    ```js
    {
        "server": "<服務器ID>",
        "status": 0,                // 0:成功, 非零:錯誤
        "payload": {                // 所有模組設備組態 或 錯誤訊息(字串)
            // 詳見 MQTT 通訊協定手冊 - 服務器回應各模組/設備組態
        }
    }
    ```

    * 成功時 `payload` 內容詳見 [MQTT 通訊協定 - 服務器回應各模組/設備組態 (cmd=101)](./MQTT%20通訊協定.md#服務器回應各模組設備組態-cmd101)。


### 查詢系統中所有模組/設備/功能最新狀態 (mq_statusreq)

URI = [`<WebAPI>`](#網站連線主機名稱)`/mq_statusreq`

本功能為 [MQTT 通訊協定 - 查詢模組/設備/功能最新狀態 (cmd=4)](./MQTT%20通訊協定.md#查詢模組設備功能最新狀態-cmd4) 中原有功能，改用 Web API 轉接。

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    lang=zh-TW
    ```

1. 網頁服務器回覆:

    ```js
    {
        "server": "<服務器ID>",
        "status": 0,                // 0:成功, 非零:錯誤
        "payload": {                // 所有模組/設備/功能最新狀態 或 錯誤訊息(字串)
            // 詳見 MQTT 通訊協定手冊 - 查詢模組/設備/功能最新狀態
        }
    }
    ```

    * 成功時 `payload` 內容詳見 [MQTT 通訊協定 - 回應指定時間點後各模組/設備/功能最新狀態 (cmd=104)](./MQTT%20通訊協定.md#回應指定時間點後各模組設備功能最新狀態-cmd104)。


### 送出控制模組/設備/功能最新狀態 (mq_ctlstatus)

URI = [`<WebAPI>`](#網站連線主機名稱)`/mq_ctlstatus`

本功能為 [MQTT 通訊協定 - 控制模組/設備/功能 (cmd=3)](./MQTT%20通訊協定.md#控制模組設備功能-cmd3) 中原有功能，改用 Web API 轉接。

App 送出 `POST` 資料如下:

JSON: `Content-Type: application/json`
```js
{
    "server": "<本地服務器ID>"，
    "loginid": "<登入帳號>"，
    "password": "<登入密碼>"，
    "payload": _payload_
}
```

* `_payload_`: 參見 [MQTT 通訊協定 - 控制模組/設備/功能 (cmd=3)](./MQTT%20通訊協定.md#控制模組設備功能-cmd3)。

範例: 送出 "測試 - 入侵警報" 推播訊息，參考 [MQTT 通訊協定 - PUSHES 推播](./MQTT%20通訊協定.md#pushes) 的 `message` 欄位及應用範例

```js
{
    "server": "<本地服務器ID>"，
    "loginid": "<登入帳號>"，
    "password": "<登入密碼>"，
    "payload": "|$00|PUSHES|2|測試 - %m"    // 若只用原有推播設定訊息時, 狀態值給 "1" 即可
}
```


### 查詢系統模組情境/智慧控制/排程/推播各功能內容 (mq_sdsqryreq)

URI = [`<WebAPI>`](#網站連線主機名稱)`/mq_sdsqryreq`

本功能為 [MQTT 通訊協定 - 查詢系統模組情境/智慧控制/排程/推播各功能內容 (cmd=5)](./MQTT%20通訊協定.md#查詢系統模組情境智慧控制排程推播各功能內容-cmd5) 中原有功能，改用 Web API 轉接。

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    did=<設備ID>
    version=<_version_>
    lang=zh-TW
    ```

    * `設備ID`: 回應內容結構會根據各項目而有所不同
        設備ID | 項目名稱
        :---:|----
        SCENES | 情境
        WISDOMS | 智慧控制
        SCHEDULES | 排程
        PUSHES | 推播
    * `_version_` 為版本數值，省略 (視同零)。若服務器版本號碼大於請求方的值，則會送出該服務器所有模組組態。現存的版本號號從 1 計起，每次組態變更會加 1，因此請求方的 `_version_` 為零則表示要求服務器重送出指定系統模組設備的內容結構。


1. 網頁服務器回覆:

    ```js
    {
        "server": "<服務器ID>",
        "status": 0,            // 0:成功, 非零:錯誤
        "payload": {            // 情境/智慧控制/排程/推播各功能內容 或 錯誤訊息(字串)
            // 詳見 MQTT 通訊協定手冊 - 系統模組回應查詢情境/智慧控制/排程/推播各功能內容
        }
    }
    ```

    * 成功時 `payload` 內容詳見 [MQTT 通訊協定 - 系統模組回應查詢情境/智慧控制/排程/推播各功能內容 (cmd=105)](./MQTT%20通訊協定.md#系統模組回應查詢情境智慧控制排程推播各功能內容-cmd105)。


### 異動系統模組情境/智慧控制/排程/推播各功能內容 (mq_sdsupdreq)

URI = [`<WebAPI>`](#網站連線主機名稱)`/mq_sdsupdreq`

本功能為 [MQTT 通訊協定 - 異動系統模組情境/智慧控制/排程/推播各功能內容 (cmd=6)](./MQTT%20通訊協定.md#異動系統模組情境智慧控制排程推播各功能內容-cmd6) 中原有功能，改用 Web API 轉接。

1. App 送出 `POST` 資料如下:

    JSON: `Content-Type: application/json`
    ```js
    {
        "server": "<本地服務器ID>"，
        "loginid": "<登入帳號>"，
        "password": "<登入密碼>"，
        "did": "_id_",
        "action": "_action_",
        "payload": _payload_
    }
    ```

    * `_id_` = `SCENES`(情境)、`WISDOMS`(智慧控制)、`SCHEDULES`(排程)、`PUSHES`(推播)。
    * `_action_` = `add`(增加)、`delete`(刪除)、`update`(更改)、`replace`(完全取代)。
    * `_payload_` 內容因 `SCENES`(情境)、`WISDOMS`(智慧控制)、`SCHEDULES`(排程)、`PUSHES`(推播) 而不同，如下所述:

    1. 情境: `_id_` = `SCENES`

        `_action_` = `add`(增加)、`update`(更改)、`replace`(完全取代):
        ```js
        "payload": {
            "情境ID": {
                "name": "**情境名稱**",
                "mode": 1,              // 重疊執行處理模式
                "actions": [            // <action-list>
                    // ...
                ]
            },
            // 其他情境...
        }
        ```

        * 詳細說明參見 [MQTT 通訊協定 - 異動系統模組情境/智慧控制/排程/推播各功能內容 - SCENES 情境](./MQTT%20通訊協定.md#scenes)。(此處的 `payload` 就是對應 MQTT 訊息中的 `functions`)

        `_action_` = `delete`(刪除):
        ```js
        "payload": [ "情境ID", ... ]
        ```

    1. 智慧控制: `_id_` = `WISDOMS`

        `_action_` = `add`(增加)、`update`(更新)、`replace`(完全取代):
        ```js
        "payload": {
            "智慧控制ID": {
                "active": 1,
                "name": "**智慧控制名稱**",
                "states": [ S1, S2, ..., Sn ]   // 狀態機
            },
            // 其他智慧控制...
        }
        ```

        * `states` 狀態機為一陣列，其內每個元素 `Si`{i=1~n} 為一群移轉物件陣列 `Ti`:
        ```js
        Si = [ T1, T2, ..., Tm ]    // 每一狀態含有數個移轉物件 Tj
        Tj = {  // Tj{j=1~m} 移轉物件 (Transition)
                "expression": expression, // 條件運算式
                "actions": [ ... ], // <action-list>
                "error": "*錯誤訊息*", // 若 `expression` 成立要送出的錯誤訊息
                "next": 0, // -1: 結束(智慧控制), 0: 不移轉(繼續下個), 1~n: 移轉到 S1~Sn
                "interval": 0 // 當 next=0 時, 表示 expression 成立後下次執行的間隔秒數
        }
        ```
        * 詳細說明參見 [MQTT 通訊協定 - 異動系統模組情境/智慧控制/排程/推播各功能內容 - WISDOMS 智慧控制](./MQTT%20通訊協定.md#wisdoms)。(此處的 `payload` 就是對應 MQTT 訊息中的 `functions`)

        `_action_` = `delete`(刪除):
        ```js
        "payload": [ "智慧控制ID", ... ]
        ```

    1. 排程: `_id_` = `SCHEDULES`

        `_action_` = `add`(增加)、`update`(更改)、`replace`(完全取代):
        ```js
        "payload": {
            "排程ID": {
                "active": 1,            // 是否啟用
                "name": "**排程名稱**",
                "timer": {
                    "start_time": "",   // 起始時間("YYYY-MM-DD HH:MM:SS.ZZZ"): 時分秒可省略
                    "end_time": "",     // 結束時間點("YYYY-MM-DD HH:MM:SS.ZZZ")
                    "holiday": 0,       // 1:例假日執行, 2:工作日執行, 其他值:不理會
                    "weeks": [],        // 0~6: 週日~週六
                    "months": [],       // 1~12 月
                    "days": [],         // 1~31 日, 負值表本月倒數天數之日
                    "hours": [],        // 0~23 時
                    "minutes": []       // 0~59 分, 至少要指定一個值
                },
                "actions": [            // <action-list>
                    // ...
                ]
            },
            // 其他排程...
        }
        ```

        * 詳細說明參見 [MQTT 通訊協定 - 異動系統模組情境/智慧控制/排程/推播各功能內容 - SCHEDULES 排程](./MQTT%20通訊協定.md#schedules)。(此處的 `payload` 就是對應 MQTT 訊息中的 `functions`)

        `_action_` = `delete`(刪除):
        ```js
        "payload": [ "排程ID", ... ]
        ```

    1. 推播: `_id_` = `PUSHES`

        `_action_` = `add`(增加)、`update`(更改)、`replace`(完全取代):
        ```js
        "payload": {
            "推播ID": {
                "name": "*推播名稱*",
                "comment": "*註解*",    // 註解: 本欄位為選項，僅供使用者自行運用
                "message": "*訊息*",    // 訊息內容
                "sound": "*音效*",      // 音效 ID
                "icon": "*圖標*",       // 圖標 ID
                "data": {               // 系統通知給 App 的額外資訊
                    "space": "*空間*",
                    "page": "*頁面*"
                    // 依 App 需求增加...
                }
            },
            // 其他推播...
        }
        ```

        * 詳細說明參見 [MQTT 通訊協定 - 異動系統模組情境/智慧控制/排程/推播各功能內容 - PUSHES 推播](./MQTT%20通訊協定.md#pushes)。(此處的 `payload` 就是對應 MQTT 訊息中的 `functions`)

        `_action_` = `delete`(刪除):
        ```js
        "payload": [ "推播ID", ... ]
        ```

1. 網頁服務器回覆:

    ```js
    {
        "server": "<服務器ID>",
        "status": 0,                // 0:成功, 非零:錯誤
        "payload": "**msg**"
    }
    ```

    * `payload` 內容詳見 [MQTT 通訊協定 - 系統模組回應異動情境/智慧控制/排程/推播各功能內容 (cmd=106)](./MQTT%20通訊協定.md#系統模組回應異動情境智慧控制排程推播各功能內容-cmd106)。


## 取得推播歷史訊息

URI = [`<WebAPI>`](#網站連線主機名稱)`/messages`

取得 MQTT 系統推播的歷史訊息。

1. App 送出 `POST` 資料如下:

    ```
    server=<本地服務器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    page=1
    page_size=100
    ```

    * `page`: 讀取第幾頁，從 1 計起。訊息由近到遠排序。
    * `page_size`: 每頁多少筆訊息，本欄位為選項，預設為每頁 100 筆。

1. 網頁服務器回覆:

    ```js
    {
        "server": "<服務器ID>",
        "status": 0,                // 0:成功, 非零:錯誤
        "payload": {
            "page": 1,
            "page_size": 100,
            "total_count": 350,
            "messages": [
                { "seqno": 350, "time": "2019-01-17 12:21:09", "body": "前門進出通知" },
                { "seqno": 349, "time": "2019-01-17 10:26:20", "body": "測試 - 警鈴" },
                //...
                { "seqno": 251, "time": "2019-01-08 20:31:34", "body": "入侵警報 - 前門被打開" }
            ]
        }
    }
    ```

    * `payload`: 當 `status` = 0 時，返回的為歷史訊息物件，否則為錯誤訊息。
    * `page`: 頁次, 1~N， N = Ceiling(`total_count`/`page_size`)。
    * `page_size`: 每頁多少筆訊息。
    * `total_count`: 資料庫中總共有多少筆訊息。
    * `messages`: 為歷史訊息陣列，每一訊息包含 `seqno`(序號)、`time`(時間戳記)、`body`(訊息內容)。


## 訊息語系 (lang)

* 訊息語系為選項，可省略。
* 此處是指 Web API 或 MQTT 中若遇到錯誤時返回的訊息語系，並不包含各模組或用戶自行設定的場所、設備、裝置的名稱。
* 語系採用 [Web browser language identification codes](https://www.metamodpro.com/browser-language-codes) 標準設定，以 2~5 碼表示 (不分大小寫):
    1. 依順找不到適用的語系時，預設為 `en` 英語語系。
    1. 台灣使用正體 (繁體) 中文語系代碼為 `zh-TW`，可簡寫為 `tw`。
    1. 中國大陸使用簡體中文語系代碼為 `zh-CN`，可簡寫為 `cn`。
* Web API 可於下列位置指定返回錯誤訊息的語系: (依優先順序高到低)
    1. 在 Post 中指定: `lang=zh-TW`。
    1. 在 URI 查詢字串 (query string) 指定，如: `https://hostname/api/create_device?lang=zh-TW`。
    1. HTTP 標題中的 "Accept-Language" 指定的順序。通常瀏覽器會有預設語系，例如 Google Chrome 可能帶的如下:\
    `Accept-Language: zh-TW,zh;q=0.9,zh-CN;q=0.8,en-US;q=0.7,en;q=0.6` \
    表示語系的順序為: `zh-TW` → `zh` → `zh-CN` → `en-US` → `en`。
* 目前本系統預設語系有 `en` (`en-??`)、`tw` (`zh-TW`)、`cn` (`zh-CN`)。


## 網頁服務器回覆 status 錯誤處理

* `status = 1` 表示一般性錯誤，使用者請依訊息做對應處理即可，程式不需特別處理。除非在 API 中另有說明。
* `status = 2` 請依 API 中說明處理，例如 [App 設備第一次要先向本地服務器註冊](#app-設備第一次要先向本地服務器註冊-create_device)。
* `status = 3` 登錄失敗，帳號/密碼錯誤，或帳號已被移除，[請重新註冊設備](#app-設備第一次要先向本地服務器註冊-create_device)。
* `status >= 10` 表示進入系統資料庫前發生錯誤，無法進行用戶、帳號授權及登入等處理:

    status | 訊息 | 處理措施
    :---:|:---:|---
    10 | 無效的服務器 ID | 請確認您所連接是正確的服務器, 或是請重新用 [UDP 尋找本地服務器](#app-尋找本地服務器)
    11 | 未指派命令 | 請檢查 URI "[`<WebAPI>`](#網站連線主機名稱)/`<命令>`" 格式是否正確, 此錯誤表示您未指定 `<命令>`
    12 | 無效的命令 | 請檢查 URI "[`<WebAPI>`](#網站連線主機名稱)/`<命令>`" 中的 `<命令>` 是否正確,<br>如: App 登錄 [`https://dev.smart-ehome.com/api/login`](#app-登錄取得取得-mqtt-通訊時所需的資訊-login)


## 客端 UDP 完整範例

* node js:

    ```js
    const crypto = require('crypto');

    var port = 9998;

    // var client_keystr = '01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16';
    // var client_key = Buffer.from(client_keystr.replace(/ /g, ''), 'hex');
    var client_key = crypto.randomBytes(16);

    var msg = 'REQ SmartHOME\t0\t' + client_key.toString('base64');
    console.log(`>>> send: ${msg}`);

    var dgram = require('dgram');
    var sock = dgram.createSocket('udp4');
    sock.bind(() => {
        sock.setBroadcast(true);
    });
    sock.send(msg, 0, msg.length, port, '255.255.255.255');

    var hmac_key = client_key;
    sock.on('message', (message, rinfo) => {
        var reply = message.toString();
        console.log(`>>> recv: ${reply}`);
        var fields = reply.split('\t');
        if (fields[0] !== 'SmartHOME' || fields.length != 3) return;

        // SmartHOME [TAB] <encrypted> [TAB] <hmac>
        var tag = crypto.createHmac('sha1', hmac_key).update(fields[1]).digest();
        if (tag.toString('base64') !== fields[2]) {
            console.log('*** Unmatch HMAC-SHA1:');
            console.log('hmac_key = Buffer.from(\''+hmac_key.toString('hex')+'\', \'hex\');');
            console.log(`   etext = \'${fields[1]}\'`);
            console.log(`    hmac = \'${fields[2]}\'`);
            return;
        }

        var encrypted = Buffer.from(fields[1], 'base64');
        var iv = encrypted.slice(0, 16);
        var encryptedBytes = encrypted.slice(16);

        var aesCtr = crypto.createDecipheriv('aes-128-ctr', client_key, iv);
        var decryptedBytes = aesCtr.update(encryptedBytes);

        var text = Buffer.from(decryptedBytes).toString();
        console.log(`>>> text: ${text}`);
        sock.close();
    });
    ```

* PHP:

    ```php
    <?php

    function nosp($s) {
        return str_replace(' ', '', $s);
    }

    $port = 9998;

    // $client_keystr = "01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16";
    // $client_key = hex2bin(nosp($client_keystr));
    $client_key = openssl_random_pseudo_bytes(16);

    $client_key_b64 = base64_encode($client_key);
    $msg = "REQ SmartHOME\t0\t{$client_key_b64}";
    echo ">>> send: $msg\n";

    $sock = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
    socket_set_option($sock, SOL_SOCKET, SO_BROADCAST, 1);
    socket_sendto($sock, $msg, strlen($msg), 0, '255.255.255.255', $port);

    $hmac_key = $client_key;  // key for HMAC-SHA1
    while (true) {
        if (socket_recv($sock, $reply, 8192, MSG_WAITALL) === false) {
            $errorcode = socket_last_error();
            $errormsg = socket_strerror($errorcode);
            die("Could not receive data: [$errorcode] $errormsg \n");
        }

        echo ">>> recv: $reply\n";
        $fields = explode("\t", $reply);
        if ($fields[0] !== "SmartHOME" || count($fields) != 3) continue;

        // SmartHOME [TAB] <encrypted> [TAB] <hmac>
        if (base64_encode(hash_hmac("sha1", $fields[1], $hmac_key, true)) !== $fields[2]) {
            echo "*** Unmatch HMAC-SHA1:\n";
            echo "\$hmac_key  = hex2bin(\"".bin2hex($hmac_key)."\");\n";
            echo "    \$etext = \"{$fields[1]}\";\n";
            echo "    \$hmac  = \"{$fields[2]}\";\n";
            continue;
        }

        $encrypted = base64_decode($fields[1]);
        $iv = substr($encrypted, 0, 16);
        $encryptedText = substr($encrypted, 16);
        #$text = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $client_key, $encryptedText, 'ctr', $iv);
        $text = openssl_decrypt($encryptedText, 'aes-128-ctr', $client_key, OPENSSL_RAW_DATA, $iv);
        echo ">>> text: $text\n";
        break;
    }

    socket_close($sock);
    ```
