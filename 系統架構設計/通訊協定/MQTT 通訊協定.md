# MQTT 通訊協定

![各系統訊息轉換示意圖](https://cacoo.com/diagrams/XIbHkJ4p7f6aNVdg-CB5DB.png)

## Node JS 範例

```js
const mqServer = {   // 來自 Web API: UDP
    "web_api": "https://example.com/api",
    "mqtt_host": "mqtt.example.com",
    "mqtt_port": 1883,
    "mqtt_ssl_port": 8883
}
const account = {   // 來自 Web API: create_device
    "device": "Sony XZ2",
    "loginid": "D2587",
    "password": "xxxxxxk6KWNbc3FxWWyyyyyy"
}
const mqtt_info = { // 來自 Web API: login
    "clientid": "1:A001-D2587(Sony XZ2)",
    "topic": {
        "pub": "to/",
        "sub": [ "from/#", "to/A001/#", "to/D2587/#" ]
    }
}

const client = require('mqtt').connect({
    connectTimeout: 10000,          // 10000ms
    clean: false,                   // not clean session
    reconnectPeriod: 1000,          // 1000ms
    keepalive: 60,                  // 60s
    host: mqServer.mqtt_host,
    port: mqServer.mqtt_ssl_port || mqServer.mqtt_port,
    protocol: mqServer.mqtt_ssl_port ? 'tls' : 'tcp',
    username: account.loginid,
    password: account.password,
    clientId: mqtt_info.clientid
})

client.subscribe(mqtt_info.topic.sub, {qos: 1}, (err, granted) => {
    if (err) {
        console.error('>>> Subscribe rror:', err)
    } else {
        console.log('>>> Subscribe granted:', granted)
    }
})

client.on('connect', () => {
    console.log('>>> Connected.')
}).on('message', (topic, message, packet) => {
    console.log(`>>> Message: [${topic}:${packet.qos}] ${message}`)
    if (packet.dup) console.log('>>> Packet:', packet)
})

// 查詢所有模組組態
const packet = { cmd: 1, version: 0 }
client.publish(mqtt_info.topic.pub+'$YS/'+account.loginid,
                JSON.stringify(packet), {qos: 1})
```

## 關鍵名詞/符號說明

MQTT 訊息 key | 其他符號 |  說明
:--:|---|---
`"s_id"` | `<sid>` `伺服器ID` `本地伺服器ID` | 伺服器短代碼 (4碼, 非 UDP 取得的本地伺服器ID)，在雲端連線時給定，本地連線時可用空值表示本地主機。
`"c_id"` | `<cid>` | 客端設備登入的帳號 `__loginid__` (5碼)。
`"m_id"` | `<mid>` `模組ID` | 模組代號 (3碼), 模組註冊時由系統指派。
`"timestamp"` | `timestamp` | 時間戳記，自 1970-01-01 (UTC) 起的毫秒數。

有時會在關鍵字前後加上 `*` 或 `_`，表示特別強調該名詞:
* `**`: 由模組開發者自行定義。
* `*`: 由用戶 (App) 自行定義。
* `__`: 由系統或模組指派，在生命週期不可變更，如: `伺服器ID`、`模組ID` 由系統指派，`設備ID`、`功能ID` 由模組開發者指派。
* `_`: 有固定值範圍。例如 `_version_` 表示版本數值，每個模組、設備的組態有自己的版號，每次異動時會漸增版號; 而 App 端可查詢組態儲存之，以後查詢時使用記錄的最後版號，以便得知是否有更新模組組態。


## 查詢伺服器下各模組/設備組態

方向 | MQTT 主題
:---:|----
App → 伺服器 | `to/$YS/<cid>`
<sup>註: MQTT 主題 為 [`<WebAPI>/login`](./Web%20API.md#app-登錄取得取得-mqtt-通訊時所需的資訊) 時所回應的 `topic.pub` + `"$YS/"` + `<cid>`</sup>

1. App 請求伺服器所有模組組態:

    ```js
    {
        "cmd": 1,
        "version": _version_
    }
    ```

    * `_version_` 為版本數值，若伺服器版本號碼大於請求方的值，則會送出該伺服器所有模組組態。現存的版本號號從 1 計起，每次組態有變更會加 1，因此請求方的 `_version_` 為零則表示要求伺服器重送出所有模組組態。

1. App 請求逐一比對指定伺服器/模組/設備版本之後的組態:

    ```js
    {
        "cmd": 1,
        "payload": "伺服器ID|模組ID|設備ID|_version_" // 或陣列
    }
    ```

    `伺服器ID`:
    * 僅在雲端連線時才有效，本地主機未到雲端系統註冊前 `s_id` 內容均為空值。
    * 本地連線時可給空值表示本地主機本身。


## 伺服器回應各模組/設備組態

方向 | MQTT 主題
:---:|----
伺服器 → App | `to/<cid>`

1. 伺服器回應新版本的模組/設備及其下的功能組態:

    ```js
    {
        "cmd": 101,
        "status": 0,
        "payload": {
            "s_id": "__伺服器ID__",
            "version": _version_,
            "name": "**伺服器名稱**",
            "modules": {                            // 模組物件
                "__模組ID__": {
                    "version": _version_,
                    "name": "**模組名稱**",
                    "devices": {                    // 設備物件
                        "__設備ID__": {
                            "version": _version_,
                            "name": "**設備名稱**",
                            "type": "**設備型號**",
                            "icon_id": "__設備項目id__",
                            "functions": {          // 功能物件
                                "__功能ID__": {
                                    "name": "**功能名稱**",
                                    "type": 1, // 輸入(可讀):1, 輸出(控制):2, 輸出入:3
                                    "value": "__值__", // 1, 2, 3, 100/n, n1~n2
                                    "icon_id": "__功能項目id__", // 若無可省略
                                    "attention": { // 可關注才有(可讀: type & 1 == 1)
                                        "message": "**通知訊息**",
                                        "icon": "**icon檔**",
                                        "sound": "**音效檔**"
                                    }
                                },
                                // 其他功能
                            }
                        },
                        // 其他設備
                    }
                },
                // 其他模組
            }
        }
    }
    ```

    * `devices` 的 `icon_id`: 見 [UI 定義 & 設計規範 - 智慧家庭: 設備項目定義](../../App/UI定義＆設計規範-智慧家庭.md#附錄一-設備項目定義)。

    * `functions` 的 `icon_id`: 見 [UI 定義 & 設計規範 - 智慧家庭: 功能項目定義](../../App/UI定義＆設計規範-智慧家庭.md#附錄二-功能項目定義)。

    * `functions` 的 `type`: 定義功能為可讀取(狀態)或可輸出(控制)
        |  值  |     說明         |
        |:---:|------------------|
        |   1  | 可讀取狀態       |
        |   2  | 可輸出控制       |
        |   3  | 可讀取/輸出      |
        | +16  | 情境 (輸出)       |
        | +32  | 智慧控制/排程 (輸出入) |
        | +64  | 可關注           |
        | +128 | 可製作圖表 (可讀取) |

        <sup>[註] Bit4: 可出現在情境中，Bit5: 可出現在智慧控制/排程中，Bit 6: 可關注，Bit 7: 可製作圖表</sup>

    * `functions` 的 `value`: 定義功能讀取或輸出的值域
        |   值  |    說明                          | 控制狀態值
        |:-----:|---------------------------------|---
        |   1   | 一次觸發 (pulse)                 | 1
        |   2   | 開/關 (on/off)                   | 0:關, 1:開, 2:切換
        |  100  | 調光 (0~100%), n=線性時間 (單位秒) | 0~100[,線性時間]
        | n1~n2 | 指定某段範圍值, 如: 温度          | 指定範圍值
        | 其他  | 通訊模組自行解釋
        | `null` | *** 僅在回應狀態表示該功能已失聯  |


1. 若無新的版本則返回 `modules` 為 `null`:
    ```js
    {
        "cmd": 101,
        "status": 0,
        "payload": {
            "s_id": "__伺服器ID__",
            "version": _version_,
            "name": "**伺服器名稱**",
            "modules": null
        }
    }
    ```


1. 若 `__伺服器ID__`、`__模組ID__` 或 `__設備ID__` 已不存在，則返 `_version_` 為 `0`:

    ```js
    {
        "cmd": 101,
        "status": 0,
        "payload": {
            "s_id": "__伺服器ID__",
            "version": 10,
            "name": "**伺服器名稱**",
            "modules": {
                "__模組ID__": {
                    "version": 0    // 此模組不存在
                },
                // 其他模組...
            }
        }
    }
    ```


## 模組/設備/功能狀態變更

1. 各通訊模組在設備/功能狀態有變更時，主動向本地伺服器、App 送出最新狀態:

    方向 | MQTT 主題
    :---:|----
    通訊模組 → 本地伺服器, App | `from/<mid>`

    ```js
    {
        "cmd": 2,
        "payload": "伺服器ID|模組ID|設備ID|功能ID|最新狀態值" // 或陣列
    }
    ```

    * `payload` 中的第一個欄位 `伺服器ID` 可以省略，通訊模組的訊息一定是送往本地主機。若要帶上 `伺服器ID` 時一定是 `本地伺服器ID`，否則此訊息會被丟棄。
    * 若同時有多組設備/功能狀態異動時， `payload` 可用陣列表示，例如:\
    `"payload": [ "|模組ID|設備ID|功能ID|最新狀態值", ... ]`

1. 本地伺服器收到狀態變更:

    於 `payload` 欄位的 `伺服器ID` 換置成 `本地伺服器ID`，增加時戳欄位 `_timestamp_`，送往雲端系統:

    方向 | MQTT 主題
    :---:|----
    本地伺服器 → 雲端伺服器 | `from/<sid>/<mid>`

    ```js
    {
        "cmd": 2,
        "payload": "伺服器ID|模組ID|設備ID|功能ID|最新狀態值", // 或陣列
        "timestamp": _timestamp_
    }
    ```

    * `_timestamp_` 為自 1970-01-01 (UTC) 起的毫秒數。對 App 而言，只要記住伺服器最後一次的時間戳記，在斷線重連時要求重送此時間後的更新狀態即可，見隨後 [查詢模組/設備/功能最新狀態](#查詢模組設備功能最新狀態)。


## 控制模組/設備/功能

由 App 或 `情境/智慧控制/排程` 觸發送往通訊模組:

方向 | MQTT 主題
:---:|----
App → 通訊模組 | `to/<mid>`

```js
{
    "cmd": 3,
    "payload": "伺服器ID|模組ID|設備ID|功能ID|**欲變更狀態值**" // 或陣列
}
```

* 通訊模組收到時 `伺服器ID`、`模組ID` 可不理會，每個通訊模組只會收到屬於自己的訊息。
* 若要同時控制多組設備/功能狀態時， `payload` 可用陣列表示，例如:\
`"payload": [ "伺服器ID|模組ID|設備ID|功能ID|**欲變更狀態值**", ... ]`


## 查詢模組/設備/功能最新狀態

1. App 啟動或斷線重連後要求各模組/設備/功能的最新狀態:

    方向 | MQTT 主題
    :---:|----
    App → 伺服器 | `to/$YS/<cid>`

    ```js
    {
        "cmd": 4,
        "payload": "伺服器ID|模組ID|設備ID|功能ID|_timestamp_" // 或陣列
    }
    ```

* `_timestamp_` 表示查詢這個時間點後有異動的狀態。
* `payload` 中的 `模組ID|設備ID|功能ID` 可以由右向左省略，表示要求某一項目以下的狀態:
    * `伺服器ID|模組ID|設備ID|功能ID|_timestamp_` 查單一功能狀態。
    * `伺服器ID|模組ID|設備ID|_timestamp_` 查此設備下所有功能狀態。
    * `伺服器ID|模組ID|_timestamp_` 查此模組下所有設備/功能狀態。
    * `伺服器ID|_timestamp_` 查此伺服器下所有模組/設備/功能狀態。
* 若同時要查詢多組設備/功能最新狀態時， `payload` 可用陣列表示，例如:\
`"payload": [ "伺服器ID|模組ID|設備ID|功能ID|_timestamp_", ... ]`


## 回應指定時間點後各模組/設備/功能最新狀態

伺服器收到查詢最新狀態，或通訊模組重新連線時主動送出:

方向 | MQTT 主題
:---:|----
伺服器 → App | `to/<cid>`

```js
{
    "cmd": 104,
    "status": 0,
    "payload": [ "伺服器ID|模組ID|設備ID|功能ID|狀態值|_timestamp_", ... ]
}
```

* `通訊模組 → 本地伺服器` 時，`伺服器ID` 為空值，且可省略 `_timestamp_` 欄位，本地伺服器在收到各設備/功能狀能值時會自動記錄此時的時間戳記。
* 如果自指定時間後都沒有異動，則返回 `payload` 為空陣列:
    ```js
    {
        "cmd": 104,
        "status": 0,
        "payload": []
    }
    ```


## 查詢系統模組情境/智慧控制/排程各功能內容

方向 | MQTT 主題
:---:|----
App → 系統模組 | `to/$00/<cid>`

App 請求系統模組各項目功能內容結構:

```js
{
    "cmd": 5,
    "payload": "伺服器ID|模組ID|設備ID|_version_"
}
```

* `伺服器ID`: 本地連線時可給空白。
* `模組ID`: 此處為 `$00`，`$` 加上兩個數字零。同 MQTT 主題的第二個欄位值。
* `設備ID`: 回應內容結構會根據各項目而有所不同
    設備ID | 項目名稱
    :---:|----
    SCENES | 情境
    WISDOMS | 智慧控制
    SCHEDULES | 排程
* `_version_` 為版本數值，若伺服器版本號碼大於請求方的值，則會送出該伺服器所有模組組態。現存的版本號號從 1 計起，每次組態變更會加 1，因此請求方的 `_version_` 為零則表示要求伺服器重送出指定模組/設備的內容結構。


## 系統模組回應查詢情境/智慧控制/排程各功能內容

方向 | MQTT 主題
:---:|----
系統模組 → App | `to/<cid>`

1. `SCENES` 情境:

    ```js
    {
        "cmd": 105,
        "status": 0,
        "payload": {
            "id": "SCENES",
            "version": _version_,
            "functions": {
                "情境ID": {
                    "name": "**情境名稱**",
                    "mode": 1,              // 重疊執行處理模式
                    "actions": [            // <action-list>
                        // ...
                    ]
                },
                // 其他情境...
            }
        }
    }
    ```

    * `mode`: 重疊執行處理模式 (同一情境之前的還在執行中)
        值 | 說明
        :---:|---
        0 | 取消都不執行
        1 | 舊的取消，新的開始執行
        2 | 舊的繼續，新的不予執行
        3 | 舊的繼續，新的開始執行
    * `actions` 欄位內容為 `<action-list>`，用下列 ABNF 語法表示之:
        ```bnf
        action-list = "[" *1concurrent *( action / action-list ) "]"
        concurrent  = "C"

        ; action = JSON 物件 { "id": item , "delay0": seconds }
        ; item = "模組ID|設備ID|功能ID|**控制狀態值**"
        ; seconds = 送出控制命令(item)前延遲秒數 (精確度 0.1 秒)

        ; 註: action-list 首個值為 "C" 表示同時(Concurrent)執行各個動作，否則為循序執行。
        ;     當所有動作結束後才算本 action-list 結束。
        ```

    範例:
    ```js
    {
        "cmd": 105,
        "status": 0,
        "payload": {
            "id": "SCENES",
            "version": 2,
            "functions": {
                "RDLightsOn": {
                    "name": "研發燈光開",
                    "mode": 1,
                    "actions": [ "C",
                        { "id": "dsc|amLight-1|PD002|1", "delay0": 0 },
                        { "id": "dsc|amLight-1|PD003|1", "delay0": 0 },
                        { "id": "", "delay0": 0.5 }
                    ]
                },
                "RDLightsOff": {
                    "name": "研發燈光關",
                    "mode": 1,
                    "actions": [
                        { "id": "dsc|amDimmer-0|001|0,50", "delay0": 0 },
                        { "id": "dsc|amLight-1|PD002|0", "delay0": 0.5 },
                        { "id": "dsc|amLight-1|PD003|0", "delay0": 0.5 }
                    ]
                }
            }
        }
    }
    ```

1. 若無新的版本則返回 `functions` 為 `null`:
    ```js
    {
        "cmd": 105,
        "status": 0,
        "payload": {
            "id": "SCENES",
            "version": _version_,
            "functions": null
        }
    }
    ```


## 異動系統模組情境/智慧控制/排程各功能內容

方向 | MQTT 主題
:---:|----
App → 系統模組 | `to/$00/<cid>`

App 請求系統模組各項目功能內容結構:

```js
{
    "cmd": 6,
    "id": "_id_",
    "action": "_action_",
    "payload": {
        // ... 視 id 值而不同
    }
}
```

* `_id_` = `SCENES`(情境)、`WISDOMS`(智慧控制)、`SCHEDULES`(排程)。
* `_action_` = `add`(增加)、`delete`(刪除)、`update`(更改)、`replace`(完全取代)。
* `payload` 內容因 `SCENES`(情境)、`WISDOMS`(智慧控制)、`SCHEDULES`(排程) 而不同，見以下說明。

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

    `_action_` = `delete`(刪除):
    ```js
    "payload": [ "情境ID", ... ]
    ```


## 系統模組回應異動情境/智慧控制/排程各功能內容

方向 | MQTT 主題
:---:|----
系統模組 → App | `to/<cid>`

App 請求系統模組各項目功能內容結構:

```js
{
    "cmd": 106,
    "status": 0,
    "payload": {
        "id": "_id_",               // SCENES, WISDOMS, SCHEDULES
        "action": "_action_",       // add, delete, update, replace
        "message": "**msg**"        // 當 "status" 不為零時表示錯誤訊息
    }
}
```

* `_id_` 及 `_action_` 是原異動封包帶入的欄位值。
* `**msg**` 可能為多筆錯誤訊息，會以斷行 (JSON 字串 "`\n`") 分隔每筆錯誤原因。


## 通訊模組註冊/異動/移除

1. 通訊模組第一次要先向本地伺服器註冊，送出組態結構，並取得模組身份驗證令牌。

    方向 | MQTT 主題
    :---:|----
    通訊模組 → 本地伺服器 | `to/$YS`

    ```js
    {
        "cmd": 20,                          // 通訊轉接模組 → 本地伺服器
        "m_id": "__模組ID__",
        "version": _version_,
        "name": "**自定模組名稱**",
        "devices": {                        // 設備物件
            "**自定設備ID**": {
                "version": _version_,
                "name": "**自定設備名稱**",
                "type": "**設備型號**",
                "icon_id": "__設備項目id__",
                "functions": {              // 功能物件
                    "**自定功能ID**": {
                        "name": "**自定功能名稱**",
                        "type": 1,      // 輸入(可讀):1, 輸出(控制):2, 輸出入:3
                        "value": "__值__", // 1, 2, 3, 100/n, n1~n2
                        "icon_id": "__功能項目id__", // 若無可省略
                        "attention": {  // 可關注才有 (可讀: type & 1 == 1)
                            "message": "**通知訊息**",
                            "sound": "**音效檔**",
                            "icon": "**圖示檔**"
                        }
                    },
                    // 第 2,3,... 功能
                }
            },
            // 第 2,3,... 設備
        }
    }
    ```

    * `devices` 結構內容請參考 [伺服器回應各模組/設備組態](#伺服器回應各模組設備組態)。
    * `**自定設備ID**`、`**自定功能ID**`: 不可包含 `|` (垂直線, ASCII Vertical Bar)，這兩個 ID UTF-8 字串總長度不可超出 112 個位元組 (bytes)，約可含 28~37 個 Unicode 字元。最好是以英數字、字下線 `_`、減號 `-` 表示為佳。長度也不宜太長，約 3~8 字元足以區分各設備及功能，不同一層的 ID 命名彼此並不會衝突。

    接收本地伺服器回應 JSON 格式:

    方向 | MQTT 主題
    :---:|----
    本地伺服器 → 通訊模組 | `to/<mid>`


    ```js
    {
        "cmd": 120,                         // 系統回覆註冊結果
        "status": 0,                        // 0:成功, 非零:錯誤
        "payload": "**msg**"                // 錯誤訊息
    }
    ```

    * 註冊失敗返回 payload 為錯誤訊息。
    * 若 `**自定模組ID**` 與已註冊模組有衝突時，則會返回 `__本地模組唯一ID__`。以後通訊則必須使用這個指定的模組 ID。

1. 當通訊模組卸下不再使用時請向本地伺服器註銷解除之，或本地伺服器定時發送[心跳訊息](#系統訊息)，若達一定時間模組沒回應視同失聯，再超過指定註銷時間則移除之。

    `通訊模組 → 本地伺服器`

    ```js
    {
        "cmd": 21,                          // 通訊轉接模組 → 本地伺服器
        "m_id": "__模組ID__"
    }
    ```

    接收本地伺服器回應 JSON 格式:

    `本地伺服器 → 通訊模組`

    ```js
    {
        "cmd": 121,                         // 本地伺服器 → 通訊轉接模組
        "status": 0,                        // 0:成功, 非零:錯誤
        "payload": "**msg**"                // 當 "status" 不為零時表示錯誤訊息
    }
    ```

1. 模組組態有異動時 (增刪改設備/功能的基本設定)，請將對應設備版本號碼加一，將整個組態結構送出通知伺服器及 App。App 自行決定收到後通知用戶是否立即重設專案或延後設定。

    `本地伺服器 → 通訊模組`

    ```js
    {
        "cmd": 22,                          // 通訊轉接模組 → 本地伺服器
        // ... 以下內容同 cmd 20
        "m_id": "__模組ID__",
        "version": _version_,
        "name": "**自定模組名稱**",
        "devices": {
            // ...
        }
    }
    ```


## 本地伺服器向雲端系統註冊/移除

1. 本地伺服器連線到雲端:
    1. 用戶先向雲端註冊帳號，取得伺服器身份驗證令牌，參考 [??]()。

    1. 身份驗證令牌連結到本地伺服器後，伺服器連線雲端主機註冊。

        `本地伺服器 → 雲端伺服器`

        ```js
        {
            "cmd": 25,                          // 本地伺服器 → 雲端伺服器
            "s_id": "**自定伺服器ID**"
        }
        ```

        接收雲端伺服器回應 JSON 格式:

        `雲端伺服器 → 本地伺服器`

        ```js
        {
            "cmd": 125,                         // 本地伺服器 → 通訊轉接模組
            "s_id": "**自定伺服器ID**",          // 註冊時送出的自自定伺服器ID
            "status": 0,                        // 0:成功, 非零:錯誤
            "payload": "**msg**"                // 錯誤訊息
        }

        // 註冊成功後解密 payload 為 JSON 字串如下:
        {
            "id": "__伺服器唯一ID__",
        }
        ```

        * 註冊失敗返回 payload 為錯誤訊息。
        * 若 `**自定伺服器ID**` 與已註冊伺服器有衝突時，則會返回 `__伺服器唯一ID__`。以後通訊則必須使用這個指定的伺服器 ID。

    1. 送出本地的各模組組態結構，參考 [回應伺服器下各模組/設備組態](#回應伺服器下各模組設備組態)。

    1. 同步用戶帳號及相關專案設定，參考 [本地伺服器和雲端系統同步資訊](#本地伺服器和雲端系統同步資訊)。


1. 解除註冊: 本地伺服器不再使用雲端服務，或與雲端主機斷線達一定時間後亦會被雲端主機視用解除註冊。日後要再使用雲端服務時，請重新註冊才可使用。

    `本地伺服器 → 雲端伺服器`

    ```js
    {
        "cmd": 26,                          // 本地伺服器 → 雲端伺服器
        "s_id": "__本地伺服器ID__"
    }
    ```

    接收本地伺服器回應 JSON 格式:

    `雲端伺服器 → 本地伺服器`

    ```js
    {
        "cmd": 126,                         // 雲端伺伺服器 → 本地伺服器
        "s_id": "__本地伺服器ID__",
        "status": 0,                        // 0:成功, 非零:錯誤
        "payload": "**msg**"                // 當 "status" 不為零時表示錯誤訊息
    }
    ```


## 系統訊息

1. 心跳及回應:

    方向 | MQTT 主題
    :---:|----
    伺服器 → 通訊模組, App | `to/<?id>/$YS`
    通訊模組, App → 伺服器 | `to/$YS/<?id>`

    ```js
    // 發送端
    {
        "cmd": 30,
        "timestamp": _timestamp_    // milliseconds since 01 January, 1970 UTC
    }
    ```

    方向 | MQTT 主題
    :---:|----
    通訊模組, App → 伺服器  | `to/$YS`
    伺服器 → 通訊模組, App | `to/<?id>`

    ```js
    // 接收端回應
    {
        "cmd": 130,
        "status": 0,
        "?_id": "**ID**",
        "payload": {
            "timestamp": _timestamp_    // 發送端送來的時戳原封不動送回
        }
    }
    ```

    * 在無通訊狀況時，伺服器每一段時間 (目前暫定一分鐘) 會向連線者送出心跳訊息。
    * 收到心跳訊息時，回應並帶上自己的 ID 回覆之:
        + 通訊轉接模組回應本地伺服器: `"m_id": "__通訊轉接模組ID__"`。
        + App (Client) 回應伺服器: `"c_id": "__loginid__"`。
        + 伺服器回應通訊模組或 App: `"s_id": "" 或 "__雲端伺服器配給本地伺服器ID__"`。
    * 回覆時將送來的時戳 `_timestamp_` 原封不動送回，這樣伺服器系統可以評估來回的延遲時間差。

1. 系統維護: 通知各通訊模組系統暫停作業，本地通訊模組執行關閉程序。而經由外部 MQTT 連線模組則會收到系統維護訊息後中斷連線，建議每 30 秒或一分鐘再嘗試連線，直到接通為止。

    方向 | MQTT 主題
    :---:|----
    伺服器 → 通訊模組 | `from/$YS`
    伺服器 → App | `from/$YS`

    ```js
    {
        "cmd": 31,
        "message": "**系統廣播訊息**"
    }
    ```


## 本地伺服器和雲端系統同步資訊

1. 用戶帳號狀態

    ```js
    {
        "cmd": 40,
        "s_id": "__伺服器ID__",
        "data": _base64_string_
    }
    ```

    * `data` 內容包含用戶帳號相關資訊。

1. 用戶專案的備份資訊

   App 送出備份資料:

    ```js
    {
        "cmd": 41,
        "s_id": "__伺服器ID__",
        "p_id": "__專案ID__",
        "data": _base64_string_
    }
    ```

    * 版本號碼以 `YYYYMMDD-HHMMSS` 表示。
    * 每一用戶各專案只保留最近 10 個版本。
    * `data` 內容由各 App 自定，本系統不管其內容，僅將之儲存供還原用。
    * 伺服器收到後加上欄位 `"version": "YYYYMMDD-HHMMSS"`，再同步到本地或雲端伺服器。

1. 用戶專案的還原資訊

    1. App 送出查詢專案目前有那些備份版本:

        ```js
        {
            "cmd": 42,
            "s_id": "__伺服器ID__",
            "p_id": "__專案ID__"
        }
        ```

        伺服器回應:

        ```js
        {
            "cmd": 142,
            "s_id": "__伺服器ID__",
            "p_id": "__專案ID__",
            "list": [ _version_, ... ]
        }
        ```

        返回版本清單由近到遠。

    1. App 送出還原專案備份版本:

        ```js
        {
            "cmd": 43,
            "s_id": "__伺服器ID__",
            "p_id": "__專案ID__"，
            "version": _version_            // 省略表示要求最後一次的版本
        }
        ```

        `version` 省略時表示要求最後一次的版本。

        伺服器回應:

        ```js
        {
            "cmd": 143,
            "s_id": "__伺服器ID__",
            "p_id": "__專案ID__",
            "version": _version_,
            "data": _base64_string_
        }
        ```

        若指定版本不存在時返回 `data` 為空字串。

