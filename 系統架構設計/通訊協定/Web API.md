# Web API

## App 尋找本地伺服器

若為私有 IP，則先以 UDP Port 9998 廣播尋找本地伺服器，若有回應則連接到指定伺服器，否則連到預設雲端伺服器 (https://oisp.smart-ehome.com/)。

1. 尋找本地伺服器 UDP 資料: `REQ SmartEHome [TAB] <port> [TAB] <client_key>`
    * `<port>` 為程式接收回應的 udp 埠, 零表示使用發送封包時的 port。
    * `<client_key>` 為至少 16 字元隨機值 (本次臨時 key)，以 base64 編碼。

    ```php
    $client_keystr = "01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16";
    $client_key = hex2bin(str_replace(' ', '', $client_keystr));
    $client_key_b64 = base64_encode($client_key);
    echo "REQ SmartEHome\t0\t{$client_key_b64}";
    # output: "REQ SmartEHome[↹]0[↹]AQIDBAUGBwgJEBESExQVFg=="
    # 以 [↹] 表示 Tab 字元
    ```

1. 本地伺服器回應 UDP `<port>`: `SmartEHome [TAB] <y> [TAB] <z>`
    * `<y>`: iv[16]+加密後的資料，轉為 base64 編碼。
    * `<z>`: `<y>` 的 `HMAC-SHA1` 驗證值，轉為 base64 編碼。 HMAC-SHA1 使用的鍵值為 `<client_key>`。
    * 若驗證正確則執行解密: `<text> =  AES-CTR 解碼(<x>, <y.data>, <y.iv>)`。
    * 解密出的 `<text>` 為一 JSON 字串:
        ```js
        {
            "s_id": "AMMA-2F",
            "ip": "192.168.1.11",
            "web_port": 8000,
            "mqtt_port": 1883
        }
        ```
    * php 逐步示範：
        ```php
        // <client key> = Hex(01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16)
        $client_key = hex2bin(str_replace(' ', '', '01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16'));

        // 收到: "SmartEHome [↹] Aj8NxaPiTAj8NxaPi+SNPYXPN03/xH3xLt5HGibmPga+P7lwGyHFo3jOGFojQtu32vw72ss0XIKa6k829IXf1hwb+na+fdY/dBKNMHA7aevJvs7uY6UqQUejcBNm [↹] jOejRETyOw/YyHkwo5utOARx+U0="
        $y = "TAj8NxaPi+SNPYXPN03/xH3xLt5HGibmPga+P7lwGyHFo3jOGFojQtu32vw72ss0XIKa6k829IXf1hwb+na+fdY/dBKNMHA7aevJvs7uY6UqQUejcBNm";
        $z = "jOejRETyOw/YyHkwo5utOARx+U0=";

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
        // 輸出: {"s_id":"AMMA-2F","ip":"192.168.1.11","web_port":8000,"mqtt_port":1883}
        ```
    * 參見 [客端 UDP 完整範例](#客端-udp-完整範例)。


## App 第一次要先取得本地伺服器的授權帳號及密碼

url = `http://<本地伺服器IP>:<WebPort>/admin/newuser`

1. App 送出 `POST` 資料如下:

    * 第 2 個起用戶必須向 admin 取得授權碼

    ```
    s_id=<本地伺服器ID>
    code=<向系統管理者取得授權碼>
    ```

1. 網頁伺服器回覆:

    ```js
    // 錯誤
    {
        "s_id": "<伺服器ID>",
        "status": 1,                        // 0:成功, 1:需admin授權碼, 2:其他錯誤
        "payload": "錯誤訊息"
    }

    // 正確
    {
        "s_id": "<伺服器ID>",
        "status": 0,                     	// 0:成功, 非零:錯誤
        "payload": {
            "user": "<用戶ID>",
            "password": "<用戶密碼>"       	// 0:成功, 非零:錯誤
        }
    }
    ```

    * 若收到 `status` = 1，請向系統管理者 (第一個註冊者) 取得新用戶授權碼後再試一次。
    * 每次新用戶授權碼使用後會失效，因此系統管理者要為每個新用戶重新取得一次授權碼。連續取得新用戶授權碼時，只有最後一個有效。


## 系統管理者取得新用戶的授權碼

url = `http://<本地伺服器IP>:<WebPort>/admin/getcode`

1. App (系統管理者) 送出 `POST` 資料如下:

    ```
    s_id=<本地伺服器ID>
    user=<系統管理者帳號>
    password=<系統管理者密碼>
    ```

1. 網頁伺服器回覆:

    ```js
    // 錯誤
    {
        "s_id": "<伺服器ID>",
        "status": 1,                        // 0:成功, 非零:錯誤
        "payload": "錯誤訊息"
    }

    // 正確
    {
        "s_id": "<伺服器ID>",
        "status": 0,                     	// 0:成功, 非零:錯誤
        "payload": "<新用戶授權碼>"
        }
    }
    ```


## 網站連線主機名稱:

1. 雲端伺服器一定 SSL 連線: `<WebHost> = "https://oisp.smart-ehome.com"`。
1. 本地伺服器無加密連線: `<WebHost> = "http://<本地伺服器IP>:<WebPort>"`。


## App 登錄取得身份驗證令牌

url = `<WebHost>/api/login`

`<身份驗證令牌>` 用於 MQTT 通訊協定。

1. App 送出 `POST` 資料如下:

    ```
    s_id=<本地伺服器ID>
    user=<用戶ID>
    password=<用戶密碼>
    ```

1. 網頁伺服器回覆:

    ```js
    {
        "s_id": "<伺服器ID>",
        "status": 0,                        // 0:成功, 非零:錯誤
        "payload": "<身份驗證令牌>"           // 身份驗證令牌 或 錯誤訊息
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
