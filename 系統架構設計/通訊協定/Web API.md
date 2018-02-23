## Web API

1. Apps 可連雲端伺服器或本地伺服器:
    * 若為私有 IP，則先以 UDP Port 9999 廣播尋找本地伺服器，若有回應則連接到指定伺服器，否則連到預設雲端伺服器。
    * 尋找本地伺服器 UDP 資料: `REQ SmartEHome [TAB] <port> [TAB] <x>`
        1. `<port>` 為程式接收回應的 udp 埠, 零表示使用發送封包時的 port。
        2. `<x>` 為至少 16 字元隨機值 (本次臨時 key)，以 base64 編碼。

        ```php
        $client_keystr = "01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16";
        $client_key = hex2bin(str_replace(' ', '', $client_keystr));
        $client_key_b64 = base64_encode($client_key);
        echo "REQ SmartEHome\t0\t{$client_key_b64}";
        # output: "REQ SmartEHome[↹]0[↹]AQIDBAUGBwgJEBESExQVFg=="
        # 以 [↹] 表示 Tab 字元
        ```

    * 本地伺服器回應 UDP `<port>`: `SmartEHome [TAB] <y> [TAB] <z>`
        1. `<server key>` 由條碼掃描本地伺服器網頁而來。本例假設為 Hex(`FF 88 10 CA 5E 2F 86 00 7F 66 67 46 C3 4B 0F DA`)。
        2. `<y>`: iv[16]+加密後的資料，轉為 base64 編碼。
        3. `<z>`: `<y>` 的 `HMAC-SHA1` 驗證值，轉為 base64 編碼。 HMAC-SHA1 使用的鍵值為 `<server key> + <client key>`。
        4. 若驗證正確則執行解密: `<text> =  AES-CTR 解碼(<server key>, <y.data>, <y.iv>)`

        ```php
        // <server key> = Hex(FF 88 10 CA 5E 2F 86 00 7F 66 67 46 C3 4B 0F DA)
        // <client key> = Hex(01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16)
        $server_key = hex2bin(str_replace(' ', '', "FF 88 10 CA 5E 2F 86 00 7F 66 67 46 C3 4B 0F DA"));
        $client_key = hex2bin(str_replace(' ', '', "01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16"));

        // 收到: "SmartEHome [↹] Qw0/HcSzMVxKcqPjkKBbcHK/M1b0eVa3sA6JH4NVaHRSynHR6WHFqyPwcI1af62AHe00vSFRpwPvM2hVBfHHcpVxY14W1f5whIhHlRehGvZkRZPR+7Fr [↹] /k1rBNeTHHuMb6x5s4bZDShdfPU="
        $y = "Qw0/HcSzMVxKcqPjkKBbcHK/M1b0eVa3sA6JH4NVaHRSynHR6WHFqyPwcI1af62AHe00vSFRpwPvM2hVBfHHcpVxY14W1f5whIhHlRehGvZkRZPR+7Fr";
        $z = "/k1rBNeTHHuMb6x5s4bZDShdfPU=";

        // 檢驗 HMAC-SHA1 是否相符
        $hmac_key = $server_key . $client_key;
        if (base64_encode(hash_hmac("sha1", $y, $hmac_key, true)) !== $z) {
            echo "*** Unmatch HMAC-SHA1";
            return false;
        }

        // 解密
        $encrypted = base64_decode($y);
        $iv = substr($encrypted, 0, 16);
        $encryptedText = substr($encrypted, 16);
        $text = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $server_key, $encryptedText, 'ctr', $iv);
        echo "$text\n";
        // 輸出: {"s_id":"AMMA-2F","ip":"192.168.1.11","web_port":8000,"mqtt_port":1883}
        ```

        * AES CTR 演算法參考 [AES-JS](https://github.com/ricmoo/aes-js)。

        * 完整 node js 客端範例:

            ```js
            const aesjs = require('aes-js');
            const crypto = require('crypto');

            var port = 9999;
            var server_keystr = 'FF 88 10 CA 5E 2F 86 00 7F 66 67 46 C3 4B 0F DA';
            var server_key = aesjs.utils.hex.toBytes(server_keystr.replace(/ /g, ''));

            // var client_keystr = '01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16';
            // var client_key = aesjs.utils.hex.toBytes(client_keystr.replace(/ /g, ''));
            var client_key = crypto.randomBytes(16);

            var msg = "REQ SmartEHome\t0\t" + client_key.toString('base64');
            console.log(`>>> send: ${msg}`);

            var dgram = require('dgram');
            var sock = dgram.createSocket('udp4');
            sock.bind(() => {
                sock.setBroadcast(true);
            });
            sock.send(msg, 0, msg.length, port, "255.255.255.255");

            var hmac_key = Buffer.concat([Buffer.from(server_key), Buffer.from(client_key)]);
            sock.on('message', (message, rinfo) => {
                var reply = message.toString();
                console.log(`>>> recv: ${reply}`);
                var fields = reply.split('\t');
                if (fields[0] !== "SmartEHome" || fields.length != 3) return;

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
                var iv = encrypted.slice(0, 16);;
                var encryptedText = encrypted.slice(16);

                var aesCtr = new aesjs.ModeOfOperation.ctr(server_key, iv);
                var decryptedBytes = aesCtr.decrypt(encryptedText);
                // var text = aesjs.utils.utf8.fromBytes(decryptedBytes);
                var text = Buffer.from(decryptedBytes).toString();
                console.log(`>>> text: ${text}`);
                sock.close();
            });
            ```

        * 完整 php 客端範例:

            ```php
            <?php

            function nosp($s) {
                return str_replace(' ', '', $s);
            }

            $port = 9999;
            $server_key = hex2bin(nosp("FF 88 10 CA 5E 2F 86 00 7F 66 67 46 C3 4B 0F DA"));

            // $client_keystr = "01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16";
            // $client_key = hex2bin(nosp($client_keystr));
            $client_key = openssl_random_pseudo_bytes(16);

            $client_key_b64 = base64_encode($client_key);
            $msg = "REQ SmartEHome\t0\t{$client_key_b64}";
            echo ">>> send: $msg\n";

            $sock = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
            socket_set_option($sock, SOL_SOCKET, SO_BROADCAST, 1);
            socket_sendto($sock, $msg, strlen($msg), 0, '255.255.255.255', $port);

            $hmac_key = $server_key . $client_key;  // key for HMAC-SHA1
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
                $text = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $server_key, $encryptedText, 'ctr', $iv);
                echo ">>> text: $text\n";
                break;
            }

            socket_close($sock);
            ```

1. App 第一次要先取得本地伺服器的授權帳號: url = `http://<本地伺服器IP>:<WebPort>/admin/newuser`
    * Apps 送出 `POST` 資料如下:<br>第2個起用戶必須向 admin 取得授權碼
        ```
        s_id=<本地伺服器ID>
        authorized_code=<從 admin 取得授權碼>
        ```

    * Web 回覆:

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

1. 網站連線主機:
    * 雲端伺服器一定 SSL 連線: `<WebHost> = "https://server:port"`。
    * 本地伺服器無加密連線: `<WebHost> = "http://<本地伺服器IP>:<WebPort>"`。
    <br>

1. Apps 登錄取得身份驗證令牌: url = `<WebHost>/api/login`
    * Apps 送出 `POST` 資料如下:

        ```
        s_id=<本地伺服器ID>
        user=<用戶ID>
        password=<用戶密碼>
        ```

    * Web 回覆:

        ```js
        {
            "s_id": "<伺服器ID>",
            "status": 0,                        // 0:成功, 非零:錯誤
            "payload": "__身份驗證令牌__"       // 身份驗證令牌 或 錯誤訊息
        }
        ```

1. Apps 向伺服器查詢各組態內容: url = `<WebHost>/api/get-list`
    * Apps 送出查詢 `POST <WebHost>/api/get-list`:

        ```
        token=__身份驗證令牌__
        ```

    * Web 回覆 JSON 物件，會以所有啟用中的通訊模組組態:

        ```js
        // 錯誤，例如: 無效的 `__身份驗證令牌__`
        {
            "s_id": "<伺服器ID>",
            "status": 1,                        // 非零:錯誤
            "payload": "**msg**"                // 錯誤訊息
        }

        // 成功
        {
            "s_id": "<伺服器ID>",
            "status": 0,                        // 0:成功, 非零:錯誤
            "payload:" {            // 啟用中的通訊模組
                "__模組ID__": {
                    "name": "**模組名稱**",
                    "devices": {                        // 設備物件
                        "**設備ID**": {
                            "name": "**設備名稱**",
                            "functions": {              // 功能物件
                                "**功能ID**": {
                                    "name": "**功能名稱**",
                                    "type": 1,              // 輸入(讀取狀態):1, 輸出(控制):2, 輸出入:3
                                    "value": "__Value__",   // 1, 2, 3, 100/n, n1~n2
                                    "ui": "__UI__",         // UI 元件
                                    "last_status": ""       // 最後狀態值
                                },
                                // 其他功能...
                            }
                        },
                        // 其他設備...
                    }
                },

                // 其他通訊模組...

                // 系統模組: 情境、智慧控制、排程
                "$SYS": {
                    "name": "**系統服務**",
                    "devices": {
                        "SCENES": {                     // 情境
                            "name": "**情境**",
                            "functions": {
                                "**情境ID**": {
                                    "name": "**情境名稱**",
                                    "type": 3,              // 只可輸出(控制)
                                    "value": "2",           // 1:觸發/執行中，0:取消/未執行
                                    "ui": "01300"           // UI 元件
                                },
                                // 其他情境...
                            }
                        },
                        "WISDOMS": {                    // 智慧控制
                            "name": "**智慧控制**",
                            "functions": {
                                "**智慧控制ID**": {
                                    "name": "**智慧控制名稱**",
                                    "type": 3,              // 讀取狀態及輸出控制
                                    "value": "2",           // 1:啟用，0:停用
                                    "ui": "01400"           // UI 元件
                                },
                                // 其他智慧控制...
                            }
                        },
                        "SCHEDULES": {                  // 排程
                            "name": "**排程**",
                            "functions": {
                                "**排程ID**": {
                                    "name": "**排程名稱**",
                                    "type": 3,              // 讀取狀態及輸出控制
                                    "value": "2",           // 1:啟用，0:停用
                                    "ui": "01500"           // UI 元件
                                },
                                // 其他排程...
                            }
                        }
                    }
                }
            }
        }
        ```
