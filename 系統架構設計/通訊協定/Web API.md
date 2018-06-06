# Web API

## App 尋找本地伺服器

若為私有 IP，則先以 UDP Port 9998 廣播尋找本地伺服器，若有回應則連接到指定伺服器，否則連到預設雲端伺服器 (`<inet.web_api>`: `https://oisp.smart-ehome.com/api`)。

1. 尋找本地伺服器 UDP 資料: `REQ SmartHOME [TAB] <port> [TAB] <client_key>`
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

1. 本地伺服器回應 UDP `<port>`: `SmartHOME [TAB] <y> [TAB] <z>`
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
    * 以下所提到 <span id="json"></span>`<本地伺服器ID>`、`<local.web_api>`、`<inet.web_api>` 是指此處 JSON 物件中 `server`、`local.web_api`、`inet.web_api` 的值，以本範例而言其值如下:
        * `<本地伺服器ID>`: `d2045940-5e38-11e8-bea1-77a4b0d356bf`
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
        $text = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $client_key, $encryptedText, 'ctr', $iv);
        echo "$text\n";
        // 輸出: 如同前述的 JSON 值
        ```
    * 參見 [客端 UDP 完整範例](#客端-udp-完整範例)。


## App 設備第一次要先向本地伺服器註冊

url = [`<local.web_api>`](#json)`/create_account`

1. App 送出 `POST` 資料如下:

    第二個起的用戶第一次授權必須[向系統管理者取得授權碼](#系統管理者取得新用戶的授權碼) (authcode)，第一個註授權的用戶即為系統管理者。

    ```
    server=<本地伺服器ID>
    userid=<用戶第三方認證ID>
    username=<用戶名稱>
    device=<設備名稱>
    authcode=<授權碼>
    lang=zh-TW
    ```

    * [`<本地伺服器ID>`](#json): 由 UDP 取得。
    * <span id="userid"></span>`<用戶第三方認證ID>`: 指 APP 在 Facebook 或 Google 取得的第三方認證識別 ID。由於本系統信任 APP 所指定的 `<用戶第三方認證ID>`，因此請 APP 設計者妥善保管此一資訊，最好能以加密方式保存之。
    * `<用戶名稱>`: 為取自第三方認證帶過來的名稱，若無可省略此欄，本系統會以 `<用戶第三方認證ID>` 取代。
    * `<設備名稱>`: 請 App 自行定義，如: iPhone、iPad、Sony XZ ...，以供後續調用區別。若 `<設備名稱>` 已存在，則會延用舊有登入帳號，重新分配一個密碼，舊的密碼立即作廢。
    * `<授權碼>`: 此欄位在新設備註冊前需向系統管理者取得之。若未帶此授權碼，除了第一個註冊的設備外，其餘會回報錯誤要求取得授權碼，之後再請再重試一次。
    * [`lang`](#訊息語系-lang): 返回錯誤訊息使用語系。

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
        "status": 0,                     	// 0:成功
        "payload": {
            "device": "<設備名稱>",
            "loginid": "<登入帳號>",
            "password": "<登入密碼>"
        }
    }

    // 錯誤
    {
        "server": "<伺服器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 若收到 `status` = `2`，請向系統管理者 (第一個註冊者) 取得新授權碼後再試一次。參見 [特殊的錯誤碼處理原則](#網頁伺服器回覆-status-錯誤處理)。
    * 每次授權碼使用後會失效，因此系統管理者要為每個新設備取得授權碼。每次取得的授權碼有效期間只有 4 個小時，超過時間後將無法註冊新設備，請重新取得授權碼後再註冊設備。


### 用戶取得已註冊設備

url = [`<local.web_api>`](#json)`/get_accounts`

1. App 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    userid=<用戶第三方認證ID>
    ```

    * [`<本地伺服器ID>`](#json): 由 UDP 取得。
    * [`<用戶第三方認證ID>`](#userid): 指用戶在 Facebook 或 Google 取得的第三方認證識別 ID。
    * 當 App 設備更換或不確定以前是否已註冊過某一設備型號時，可用此 API 取得用戶已註冊清單，再決定是否為新註冊設備。

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
        "status": 0,                     	// 0:成功
        "payload": {
            "userid": "<用戶第三方認證ID>",
            "username": "<用戶名稱>",
            "devices": ["<設備名稱1>", "<設備名稱2>", ...]
        }
    }

    // 錯誤
    {
        "server": "<伺服器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


### 用戶刪除已註冊設備

url = [`<local.web_api>`](#json)`/delete_account`

1. App 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    userid=<用戶第三方認證ID>
    loginid=<登入帳號>
    password=<登入密碼>
    device=<設備名稱>
    ```

    * [`<本地伺服器ID>`](#json): 由 UDP 取得。
    * [`<用戶第三方認證ID>`](#userid): 指用戶在 Facebook 或 Google 取得的第三方認證識別 ID。
    * `loginid`、`password`: 必需要屬於用戶 `userid` 其中之一的設備帳號，以驗證用戶的合法性。通常就是 App 安裝所在的設備，每次註冊後必須將帳號、密碼儲存起來。
    * `device`: 驗證用戶合法性後會將指定的設備刪除之，請注意這裡的是指要刪除的設備，不是帳號 `loginid` 關聯的設備，小心不要誤刪除自己本身。
    * 若不是系統管理者時，當用戶所有註冊設備都已清空時，本系統會自動移除此用戶資訊。若是系統管理者名下已無註冊設備，在[重新註冊一個新的設備](#app-設備第一次要先向本地伺服器註冊)時並不需要授權碼驗證。

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
        "status": 0,                        // 0:成功
        "payload": "<完成訊息>"              // 已刪除用戶、還有 n 個設備 ...
    }

    // 錯誤
    {
        "server": "<伺服器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


## 系統管理者取得新用戶的授權碼

url = [`<local.web_api>`](#json)`/get_authcode`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    ```

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
        "status": 0,                     	// 0:成功
        "payload": "<新用戶授權碼>"
    }

    // 錯誤
    {
        "server": "<伺服器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


##

### 取得註冊用戶名單

url = [`<local.web_api>`](#json)`/get_reg_users`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    userid=<用戶第三方認證ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    ```

    * [`<本地伺服器ID>`](#json): 由 UDP 取得。
    * [`<用戶第三方認證ID>`](#userid): 指用戶在 Facebook 或 Google 取得的第三方認證識別 ID。
    * `loginid`、`password`: 需驗證的登入帳號、密碼。

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
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
        "server": "<伺服器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 確認系統管理者身份無誤後，返回的 `<payload>` 為系統現行用戶名單 (陣列)。其中欄位 `is_admin` 表示該用戶是否為系統管理者。


### 變更系統管理者身份

url = [`<local.web_api>`](#json)`/update_reg_user`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    userid=<用戶第三方認證ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    target_userid=<用戶第三方認證ID>
    target_name=<用戶名稱>
    is_admin=<0,1>
    ```

    * [`<本地伺服器ID>`](#json): 由 UDP 取得。
    * [`<用戶第三方認證ID>`](#userid): 指用戶在 Facebook 或 Google 取得的第三方認證識別 ID。
    * `loginid`、`password`: 需驗證的登入帳號、密碼。
    * `target_userid`、`is_admin`: 設定用戶 `target_userid` 管理者身份 (`is_admin`)。
    * `target_name`: 不為空白時才更新用戶名稱，此欄位可省略。

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
        "status": 0,                     	// 0:成功
        "payload": "<成功訊息>"
    }

    // 錯誤
    {
        "server": "<伺服器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 確認系統管理者身份無誤後，返回的 `<payload>` 為系統現行用戶名單 (陣列)。其中欄位 `is_admin` 表示該用戶是否為系統管理者。
    * 本系統允許多個系統管理者。


### 刪除註冊用戶

url = [`<local.web_api>`](#json)`/delete_reg_user`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    userid=<用戶第三方認證ID>
    loginid=<系統管理者登入帳號>
    password=<系統管理者登入密碼>
    target_userid=<用戶第三方認證ID>
    ```

    * [`<本地伺服器ID>`](#json): 由 UDP 取得。
    * [`<用戶第三方認證ID>`](#userid): 指用戶在 Facebook 或 Google 取得的第三方認證識別 ID。
    * `loginid`、`password`: 需驗證的登入帳號、密碼。
    * `target_userid`: 刪除用戶 `target_userid` 及其下所有的註冊設備、登入資訊等。

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
        "status": 0,                     	// 0:成功
        "payload": "<完成訊息>"
    }

    // 錯誤
    {
        "server": "<伺服器ID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 用戶被刪除後，必須重新註冊設備後方可繼續使用本系統。


## App 登錄取得身份驗證令牌

url = [`<WebAPI>`](#網站連線主機名稱)`/login`

取得 MQTT 通訊時所需的資訊。

1. App 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    loginid=<登入帳號>
    password=<登入密碼>
    lang=zh-TW
    ```

    * [`lang`](#訊息語系-lang): 登入成功時，本系統會將您的語系存到對應的登入帳號。以後在 MQTT 通訊中若發生錯誤時，會以此刻指定的語系送出相關訊息。

1. 網頁伺服器回覆:

    ```js
    {
        "server": "<伺服器ID>",
        "status": 0,                        // 0:成功, 非零:錯誤
        "payload": {                        // MQTT 用物件 或 錯誤訊息(字串)
            "router": "<routing key>",
            "clientid": "<cliet id>",
            "token": "<身份驗證令牌>"
        }
    }
    ```

##

### 網站連線主機名稱

1. 雲端伺服器連線: `<WebAPI>` = [`<inet.web_api>`](#json)。
1. 本地伺服器連線: `<WebAPI>` = [`<local.web_api>`](#json)。


### 訊息語系 (lang)

* 此處是指 Web API 或 MQTT 中若遇到錯誤時返回的訊息語系，並不包含各模組或用戶自行設定的場所、設備、裝置的名稱。
* 語系採用 [Web browser language identification codes](https://www.metamodpro.com/browser-language-codes) 標準設定，以 2~5 碼表示 (不分大小寫):
    1. 依順找不到適用的語系時，預設為 `en` 英語語系。
    1. 台灣使用正體 (繁體) 中文語系代碼為 `zh-TW`，可簡寫為 `tw`。
    1. 中國大陸使用簡體中文語系代碼為 `zh-CN`，可簡寫為 `cn`。
* Web API 可於下列位置指定返回錯誤訊息的語系: (依優先順序高到低)
    1. 在 Post 中指定: `lang=zh-TW`。
    1. 在 URL 查詢字串 (query string) 指定，如: `https://hostname/api/create_account?lang=zh-TW`。
    1. HTTP 標題中的 "Accept-Language" 指定的順序。通常瀏覽器會有預設語系，例如 Google Chrome 可能帶的如下:\
    `Accept-Language: zh-TW,zh;q=0.9,zh-CN;q=0.8,en-US;q=0.7,en;q=0.6` \
    表示語系的順序為: `zh-TW` → `zh` → `zh-CN` → `en-US` → `en。`
* 目前本系統預設語系有 `en` (`en-??`)`、tw` (`zh-TW`)`、cn` (`zh-CN`)。


### 網頁伺服器回覆 status 錯誤處理

* `status = 1` 表示一般性錯誤，使用者請依訊息做對應處理即可，程式不需特別處理。除非在 API 中另有說明。

* `status = 2` 請依 API 中說明處理，例如 [App 設備第一次要先向本地伺服器註冊](#app-設備第一次要先向本地伺服器註冊)。

* `status >= 10` 表示進入系統資料庫前發生錯誤，無法進行用戶、帳號授權及登入等處理:

    | status | 訊息 | 處理措施 |
    |:---:|:---:|---|
    | 10 | 無效的伺服器 ID | 請確認您所連接是正確的伺服器, 或是請重新用 [UDP 尋找本地伺服器](#app-尋找本地伺服器) |
    | 11 | 未指派命令 | 請檢查 url "[`<WebAPI>`](#網站連線主機名稱)/`<命令>`" 格式是否正確, 此錯誤表示您未指定 `<命令>` |
    | 12 | 無效的命令 | 請檢查 url "[`<WebAPI>`](#網站連線主機名稱)/`<命令>`" 中的 `<命令>` 是否正確,<br>如: App 登錄 [`https://dev.smart-ehome.com/api/login`](#app-登錄取得身份驗證令牌) |


### 客端 UDP 完整範例

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
        $text = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $client_key, $encryptedText, 'ctr', $iv);
        echo ">>> text: $text\n";
        break;
    }

    socket_close($sock);
    ```
