# Web API

## App 尋找本地伺服器

若為私有 IP，則先以 UDP Port 9998 廣播尋找本地伺服器，若有回應則連接到指定伺服器，否則連到預設雲端伺服器 (`<inet.web_api>`: https://oisp.smart-ehome.com/api)。

1. 尋找本地伺服器 UDP 資料: `REQ SmartEHome [TAB] <port> [TAB] <client_key>`
    * `<port>` 為程式接收回應的 udp 埠, 零表示使用發送封包時的 port。
    * `<client_key>` 為至少 16 字元隨機值 (本次臨時 key)，以 base64 編碼。

    ```php
    $client_keystr = "01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16";
    $client_key = hex2bin(str_replace(' ', '', $client_keystr));
    $client_key_b64 = base64_encode($client_key);
    echo "REQ SmartEHome\t0\t{$client_key_b64}";
    # output: "REQ SmartEHome[↹]0[↹]AQIDBAUGBwgJEBESExQVFg=="
    # 上行輸出以 [↹] 表示 Tab 字元
    ```

1. 本地伺服器回應 UDP `<port>`: `SmartEHome [TAB] <y> [TAB] <z>`
    * `<y>`: iv[16]+加密後的資料，轉為 base64 編碼。
    * `<z>`: `<y>` 的 `HMAC-SHA1` 驗證值，轉為 base64 編碼。 HMAC-SHA1 使用的鍵值為 `<client_key>`。
    * 若驗證正確則執行解密: `<text> =  AES-CTR 解碼(<client_key>, <y.data>, <y.iv>)`。
    * 解密出的 `<text>` 為 JSON 字串，例如:
        ```json
        {
            "server": "AMMA-2F",
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
    * 以下所提到 `<本地伺服器ID>` 、 `<local.web_api>` 、 `<inet.web_api>` 是指此處 JSON 物件中 `server` 、 `local.web_api` 、 `inet.web_api` 的值，以本範例而言其值如下:
        * `<本地伺服器ID>`: `"AMMA-2F"`
        * `<local.web_api>`: `"https://dev.smart-ehome.com/api"`
        * `<inet.web_api>`: `"https://oisp.smart-ehome.com/api"`
    * php 逐步示範：
        ```php
        // <client key> = Hex(01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16)
        $client_key = hex2bin(str_replace(' ', '', '01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16'));

        // 收到: "SmartEHome [↹] $y [↹] $z"
        $y = "1ZW3kd3Ih4jXUKl+sMo4xRp0NP6hHPM5Fgrz3R98x/5o3l+wKyaVNWrJ7fx82/0Vy0QFM3GVnRwIxdsXF/KK0SZmd/e/gSNzH53cz/CLPlB8bqNa3nwfAy2wPoYZz8hcpfsGCCGUzvXu7ah+RObJ8mWxIbMaWfpFpl+fDEn0RptJy66VjZuTijfVRWF0rYZm11XtKfMdfnO2yIj+fFSkAalEQqKRKsraIK39tK1/DHVmHduKqRhdmPu9Y8T7YgHk5ACEZcFvpn+u4GLmk1jeh8gNPsAJuCyy6oeujvk7d1OVL/9xW1DNMSFi6XJiYhAgGs9CCJScuP0bH/AG+qPMpA1kjN2wjY4z04Ns51kf9hhi97BmODEGtPtvm+FyP579f30KNL6Zv0obf/34vhmPPWsne8MTxCTRcG3lzU+gipL3/TogDWn6j7JhmjeDoI7YTGU2xp1QTJr5sgILJmUfrupf/oPpbyclz7252ToFaH2fl3rVDap2dx5rtZBhANJDJiJj6nG+5+cIpw3sJHOa";
        $z = "PCX8xaGWcYtBCbqswye7eUiXpgk=";

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


## App 第一次要先取得本地伺服器的授權帳號及密碼

url = `<local.web_api>/newuser`

1. App 送出 `POST` 資料如下:

    * 第 2 個起用戶必須向 admin 取得授權碼 (authcode)

    ```
    server=<本地伺服器ID>
    authcode=<向系統管理者取得授權碼>
    ```

1. 網頁伺服器回覆:

    ```js
    // 正確
    {
        "server": "<伺服器ID>",
        "status": 0,                     	// 0:成功
        "payload": {
            "username": "<用戶ID>",
            "password": "<用戶密碼>"
        }
    }

    // 錯誤
    {
        "server": "<伺服器ID>",
        "status": 1,                        // 1:需admin授權碼, 2:其他錯誤。請參見 <payload> 一欄錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```

    * 若收到 `status` = 1，請向系統管理者 (第一個註冊者) 取得新用戶授權碼後再試一次。
    * 每次新用戶授權碼使用後會失效，因此系統管理者要為每個新用戶重新取得一次授權碼。每次取得的用戶授權碼有效期間只有 4 個小時，超過時間後無法註冊為新用戶，請再重新取得授權碼註冊。


## 系統管理者取得新用戶的授權碼

url = `<local.web_api>/getauthcode`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    username=<系統管理者帳號>
    password=<系統管理者密碼>
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
        "status": 1,                        // 非零:錯誤, 請參見 <payload> 一欄錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```


## 網站連線主機名稱

1. 雲端伺服器連線: `<WebAPI> = <inet.web_api>`。
1. 本地伺服器連線: `<WebAPI> = <local.web_api>`。


## App 登錄取得身份驗證令牌

url = [`<WebHost>`](#網站連線主機名稱)`/login`

取得 MQTT 通訊時所需的資訊。

1. App 送出 `POST` 資料如下:

    ```
    server=<本地伺服器ID>
    username=<用戶ID>
    password=<用戶密碼>
    ```

1. 網頁伺服器回覆:

    ```js
    {
        "server": "<伺服器ID>",
        "status": 0,                        // 0:成功, 非零:錯誤
        "payload": {                        // MQTT 用物件 或 錯誤訊息
            "router": "<routing key>",
            "queue": "<Queue Name>",
            "token": "<身份驗證令牌>"
        }
    }
    ```


## 客端 UDP 完整範例

* node js:

    ```js
    const crypto = require('crypto');

    var port = 9998;

    // var client_keystr = '01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16';
    // var client_key = Buffer.from(client_keystr.replace(/ /g, ''), 'hex');
    var client_key = crypto.randomBytes(16);

    var msg = 'REQ SmartEHome\t0\t' + client_key.toString('base64');
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
        if (fields[0] !== 'SmartEHome' || fields.length != 3) return;

        // SmartEHome [TAB] <encrypted> [TAB] <hmac>
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
    $msg = "REQ SmartEHome\t0\t{$client_key_b64}";
    echo ">>> send: $msg\n";

    $sock = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
    socket_set_option($sock, SOL_SOCKET, SO_BROADCAST, 1);
    socket_sendto($sock, $msg, strlen($msg), 0, '255.255.255.255', $port);

    $hmac_key = $client_key;  // key for HMAC-SHA1
    while (true) {
        if (socket_recv($sock, $reply, 8192, MSG_WAITALL) === FALSE) {
            $errorcode = socket_last_error();
            $errormsg = socket_strerror($errorcode);
            die("Could not receive data: [$errorcode] $errormsg \n");
        }

        echo ">>> recv: $reply\n";
        $fields = explode("\t", $reply);
        if ($fields[0] !== "SmartEHome" || count($fields) != 3) continue;

        // SmartEHome [TAB] <encrypted> [TAB] <hmac>
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
