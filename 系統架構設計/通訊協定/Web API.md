## Web API

1. Apps 可連雲端伺服器或本地伺服器:
    * 若為私有 IP，則先以 UDP Port 9999 廣播尋找本地伺服器，若有回應則連接到指定伺服器，否則連到預設雲端伺服器。
    * 尋找本地伺服器 UDP 資料: `**SmartEHome: <xxxx>**`， `<xxxx>` 表示 4 位數字 (含前置零)。
    * 本地伺服器回應 UDP Port 9999: `<yyyy>` 為 `<xxxx>` x 1234 取低 4 位數字 (含前置零)。
        ```
        SmartEHome: <yyyy>
        Server ID: <本地伺服器ID>
        <server ip> <port>
        ```

