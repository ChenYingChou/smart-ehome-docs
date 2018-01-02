## Web API

1. Apps 可連雲端伺服器或本地伺服器:
    * 若為私有 IP，則先以 UDP Port 9999 廣播尋找本地伺服器，若有回應則連接到指定伺服器，否則連到預設雲端伺服器。
    * 尋找本地伺服器 UDP 資料: `REQ SmartEHome: <x>`， `<x>` 為至少16字元隨機字串，以 base64 編碼。
    * 本地伺服器回應 UDP Port 9999: `SmartEHome [TAB] <y> [TAB] <z>`

        1. <server key> 由條碼掃描本地伺服器網頁而來。
        2. `<y>` 和 `<z>` 皆為 base64 編碼。
        3. 驗證 `<y>` 是否和 `HMAC-SHA1(<server key>, <x> base64解碼) base64編碼` 相等，若不等則表示本訊息不正確，捨棄不理會。
        4. 若驗證正確則執行解密: `<msg> =  AES CTR(<server key>, <z> base64解碼)`

            ```js
            <msg> = {
                "ServerID": "<本地伺服器ID>",
                "IP": "<本地伺服器IP>",
                "WebPort": <WebPort>,
                "MqttPort": <MqttPort>
            }
            ```

2. 網站連線主機:
    * 雲端伺服器一定 SSL 連線: `<Web> = "https://server:port"`。
    * 本地伺服器無加密連線: `<Web> = "http://<本地伺服器IP>:<WebPort>"`。

3. Apps 登錄取得身份驗證令牌: url 為 `<Web>/api/login`
    * `POST` 送出資料如下:

        ```
        user=<user name>
        password=<password>
        ```

    * Web 回覆:

        ```js
        {
            "s_id": "<伺服器ID>",
            "status": 0,                        // 0:成功, 非零:錯誤
            "payload": "**msg**"                // 加密資料 或 錯誤訊息
        }
        ```


