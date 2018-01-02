## Web API

1. Apps 可連雲端伺服器或本地伺服器:
    * 若為私有 IP，則先以 UDP Port 9999 廣播尋找本地伺服器，若有回應則連接到指定伺服器，否則連到預設雲端伺服器。
    * 尋找本地伺服器 UDP 資料: `REQ SmartEHome: <x>`， `<x>` 為至少16字元隨機字串，以 base64 編碼。
    * 本地伺服器回應 UDP Port 9999: `SmartEHome [TAB] <y> [TAB] <z>`

        1. <server key> 由條碼掃描本地伺服器網頁而來。
        2. `<y>` 和 `<z>` 皆為 base64 編碼。
        3. 驗證 `<y>` 是否和 `HMAC-SHA1(<server key>, <x> base64 解碼) 再 base64 編碼` 相等，若不等則表示本訊息不正確，捨棄不理會。
        4. 若驗證正確則執行解密: `<msg> =  AES-CTR 解碼(<server key>, <z> base64 解碼)`

            ```js
            <msg> = {
                "s_id": "<本地伺服器ID>",
                "ip": "<本地伺服器IP>",
                "web_port": <WebPort>,
                "mqtt_port": <MqttPort>
            }
            ```

        * AES CTR 演算法參考 [AES-JS](https://github.com/ricmoo/aes-js)。
        <br>

1. 網站連線主機:
    * 雲端伺服器一定 SSL 連線: `<WebHost> = "https://server:port"`。
    * 本地伺服器無加密連線: `<WebHost> = "http://<本地伺服器IP>:<WebPort>"`。

1. Apps 登錄取得身份驗證令牌: url = `<WebHost>/api/login`
    * Apps 送出 `POST` 資料如下:

        ```
        s_id=本地伺服器ID
        user=用戶ID
        password=AES-CTR 加密(<server key>, <用戶密碼>) 再 base64 編碼
        ```

    * Web 回覆:

        ```js
        {
            "s_id": "<伺服器ID>",
            "status": 0,                        // 0:成功, 非零:錯誤
            "payload": "**msg**"                // 加密資料 或 錯誤訊息
        }

        // 登錄成功後解密 AES-CTR 解碼(<server key>, "**msg**" base64 解碼)` 為 JSON 如下:
        {
            "token": "__身份驗證令牌__"         // 送往 Server 要帶上此值
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
                                    "type": 2,              // 只可輸出(控制)
                                    "value": "1",           // 一次觸發: puls
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
                                    "value": "2",           // 開關: on, off
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
                                    "value": "2",           // 開關: on, off
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

